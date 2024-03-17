---
title: 高性能图像缩放：Pillow-SIMD resize [2]
categories: Algorithm
date: 2024/3/17 14:35:00
mathjax: true
---

## 译者序

这篇文章是[The fastest production-ready image resize out there. Part 1. General optimizations](https://uploadcare.com/blog/the-fastest-production-ready-image-resize/)的DeepL机翻+人工复核。如有翻译问题可在评论区指出。其作者Alex Karpinsky系[pillow-simd](https://github.com/uploadcare/pillow-simd)的开发者。

上一篇的翻译[在这](https://zhuanlan.zhihu.com/p/687243531)。

## 前言

在此前的[介绍性文章](https://uploadcare.com/blog/the-fastest-image-resize/)（对应的[中译版](https://zhuanlan.zhihu.com/p/687243531)）中，我对图像缩放的挑战进行了全面总结。结果发现这个故事相当长，而且有点半生不熟：它没有包含一行代码。

不过，如果没有前面的总结，就很难谈及具体的优化方法。当然，我们可以将一些技术应用到手头的任何代码中。例如，缓存计算或减少分支。但我认为，如果没有对所处理问题的整体理解，就无法完成某些事情。这就是活人与编译器自动优化的区别。这也是为什么人工优化仍然起着至关重要的作用：**编译器处理代码，人类处理问题**。编译器无法判断一个数字是否具有足够的随机性，而人类却可以。

让我们回忆一下，我们正在讨论通过Pillow Python库中基于卷积的重采样来优化图像缩放速度。在这篇“通用优化”文章中，我将向大家介绍我几年前实施的更改。这不是一个逐字逐句的故事：我优化了叙述的优化顺序。我还在版本库中创建了一个单独的 2.6.2版本分支，这是我们的起点。

## 测试

如果你想拥有交互式的体验——不止于阅读，还想运行一些测试，这里有pillow-perf软件仓库。你可以自行安装并运行测试：

```shell
# Installing packages needed for compiling and testing

$ sudo apt-get install -y build-essential ccache python-dev libjpeg-dev
$ git clone -b opt/scalar https://github.com/uploadcare/pillow-simd.git
$ git clone --depth 10 https://github.com/python-pillow/pillow-perf.git
$ cd ./pillow-simd/

# Swithching to the commit everything starts with

$ git checkout bf1df9a

# Building and setting up Pillow

$ CC="ccache cc" python ./setup.py develop

# Finally, running the test

$ ../pillow-perf/testsuite/run.py scale -n 3
```

由于Pillow由许多模块组成，无法以增量方式编译，因此我们使用ccache工具加快重复编译的速度。

pillow-perf允许你测试许多不同的操作，但我们对规模特别感兴趣，其中`-n 3`设置了测试运行的次数。虽然代码的运行速度很慢，但我们还是使用较小的`-n`数来避免睡着。下面是初始性能的结果，来自[commit bf1df9a](https://github.com/uploadcare/pillow-simd/commit/bf1df9a04f0f92124a18789871f43391e9a00f01)：

```
Operation             Time         Bandwidth
---------------------------------------------
to 320x200 bil        0.08927 s    45.88 Mpx/s
to 320x200 bic        0.13073 s    31.33 Mpx/s
to 320x200 lzs        0.16436 s    24.92 Mpx/s
to 2048x1280 bil      0.40833 s    10.03 Mpx/s
to 2048x1280 bic      0.45507 s     9.00 Mpx/s
to 2048x1280 lzs      0.52855 s     7.75 Mpx/s
to 5478x3424 bil      1.49024 s     2.75 Mpx/s
to 5478x3424 bic      1.84503 s     2.22 Mpx/s
to 5478x3424 lzs      2.04901 s     2.00 Mpx/s
```

大家可能还记得，测试显示的是 2560×1600 RGB 图像的缩放。

上述结果与2.6版本的官方基准性能有所不同。原因有两个：

- 官方基准测试使用的是64位Ubuntu 16.04和GCC 5.3，而我在本文中使用的是32位Ubuntu 14.04和GCC 4.8。读完这篇文章，你就会明白我为什么这么做了。
- 在这一系列文章中，我首先提交的是与优化无直接关系但会影响性能的错误修复。

## 代码结构

我们将要研究的代码的重要部分位于[Antialias.c](https://github.com/uploadcare/pillow-simd/blob/f961886b527d2a6de83ab4da952f032730f12ff1/libImaging/Antialias.c)文件中，即`ImagingStretch`函数。该函数的代码可分为三部分：

```
// Prelude
if (imIn->xsize == imOut->xsize) {
  // Vertical resize
} else {
  // Horizontal resize
}
```

正如我在上一篇文章中强调的，基于卷积的大小调整可以分两次（two passes）进行：第一次是改变图像宽度，第二次是改变图像高度，反之亦然。

`ImagingStretch`函数在一次调用中处理其中一个维度。在这里，你可以看到每次调整尺寸操作都要调用函数两次。首先是前处理，根据输入参数执行一个或另一个操作。在函数内部，根据处理方向的调整，两次传递看起来都差不多。为简洁起见，下面是其中一个过程的代码：

```
for (yy = 0; yy < imOut->ysize; yy++) {
 // Counting the coefficients
 if (imIn->image8) {
   // A single 8-bit channel loop
 } else {
   switch (imIn->type) {
     case IMAGING_TYPE_UINT8:
       // Loop, multiple 8-bit channels
     case IMAGING_TYPE_INT32:
       // Loop, single 32-bit channel
     case IMAGING_TYPE_FLOAT32:
       // Loop, single float channel
   }
 }
}
```

我们可以看到Pillow支持的图片格式分支：单通道8位、灰色阴影；多通道8位、RGB、RGBA、LA、CMYK等；单通道32位和32位浮点运算。我们对多通道8位格式特别感兴趣，因为它是最常见的图像格式。

## 优化点1：有效利用缓存

虽然我说过这两段步骤看起来很相似，但还是有很大区别的。让我们先看看“垂直”的做法：

```
for (yy = 0; yy < imOut->ysize; yy++) {
  // Calculating coefficients
  for (xx = 0; xx < imOut->xsize*4; xx++) {
    // Convolution, column of pixels
    // Saving pixel to imOut->image8[yy][xx]
  }
}
```

以及“水平”的做法

```
for (xx = 0; xx < imOut->xsize; xx++) {
  // Calculating coefficients
  for (yy = 0; yy < imOut->ysize; yy++) {
    // Convolution, row of pixels
    // Saving pixel to imOut->image8[yy][xx]
  }
}
```

在垂直处理（vertical pass）过程中，我们会沿列方向遍历结果图像（resulting image），而在水平处理（horizontal pass）过程中，我们沿行遍历。垂直遍历对处理器高速缓存来说是一个严重的问题。嵌入式循环的每一步都会访问一行的起始元素，因此所请求的值与前一步访问的值相距甚远。

这在处理较小尺寸的卷积时是个问题。目前的处理器只能从内存中读取64字节的缓存行。这意味着，当我们卷积的像素小于16个时，从RAM中读取部分数据到缓存是一种浪费。但想象一下，我们将循环倒置。现在，每个卷积的像素都不在下一行，而是当前行的下一个像素。因此，大部分所需的像素都已经在缓存中了。

这种代码组织的第二个不利因素与较长的卷积行有关：在大幅降频的情况下。问题是，相邻卷积的原始像素会有很大的交集，如果这些数据还保留在缓存中，那就再好不过了。但当我们从上到下移动时，缓存中的旧卷积数据会逐渐被新卷积数据取代。因此，在嵌入式循环之后，当下一个外部步骤开始时，缓存中就没有上层行了，它们都被下层行所取代。因此，我们必须再次从 RAM 中读取这些上行数据。而当读取到较低行时，缓存中的所有数据都已被较高行取代。这样就形成了一个循环，缓存中根本没有需要的数据。

为什么会这样呢？在上面的伪代码中，我们可以看到两种情况下的第二行都代表卷积系数的计算。在垂直方向（vertical pass）上，系数只取决于结果图像中某一行的yy值，而在水平方向上，系数则取决于当前列中的xx值（也就是卷积系数取决于目标图像中的坐标[xx,yy]）。因此，我们不能简单地将两个循环对调，我们应该在针对xx的循环中计算系数。如果我们在内循环中计算系数，性能就会下降。特别是当我们使用Lanczos滤波器进行计算时——它使用了三角函数。

因此，我们不应该在每一步都计算系数，即使一列中对每个像素的系数都是相同的。不过，我们可以预先计算好所有列的系数，并在内循环中使用它们。下面来实践一下。

分配一行内存用于存放系数：

```
k = malloc(kmax * sizeof(float));
```

现在，我们实际需要一个由上面的数组组成的矩阵。我们可以通过分配一个平面内存片段并通过寻址模拟两个维度来简化：

```
kk = malloc(imOut->xsize * kmax * sizeof(float));
```

我们还需要存储xmin和xmax，它们也与xx相关。让我们为它们创建一个数组：

```
xbounds = malloc(imOut->xsize * 2 * sizeof(float));
```

此外，在循环内部，ww的值用于归一化，即$$ww = 1 / \sum{k}$$。我们根本不需要存储它。相反，我们可以对系数本身进行归一化处理，而不是对卷积结果进行归一化处理。因此，在计算出系数后，我们会再次遍历这些系数，用每个值除以系数总和。这样，所有系数之和就变成了$$1.0$$：

```
k = &kk[xx * kmax];

for (x = (int) xmin; x < (int) xmax; x++) {
  float w = filterp->filter((x - center + 0.5) * ss);
  k[x - (int) xmin] = w;
  ww = ww + w;
}

for (x = (int) xmin; x < (int) xmax; x++) {
  k[x - (int) xmin] /= ww;
}
```

最后，我们可以将流程“旋转90度”：

```
// Calculating the coefficients
for (yy = 0; yy < imOut->ysize; yy++) {
  for (xx = 0; xx < imOut->xsize; xx++) {
    k = &kk[xx * kmax];
    xmin = xbounds[xx * 2 + 0];
    xmax = xbounds[xx * 2 + 1];
    // Convolution, row of pixels
    // Saving pixel to imOut->image8[yy][xx]
  }
}
```

以下是我们得到的性能结果，[commit d35755c](https://github.com/uploadcare/pillow-simd/commit/d35755c5faf3ec8ea25ffb9b6249deaa16b3b2f5#diff-3f8dc1d8e4d89c526319ca16c9527bc9)：

```
Operation             Time         Bandwidth      Improvement
-------------------------------------------------------------
to 320x200 bil        0.04759 s    86.08 Mpx/s    87.6 %
to 320x200 bic        0.08970 s    45.66 Mpx/s    45.7 %
to 320x200 lzs        0.11604 s    35.30 Mpx/s    41.6 %
to 2048x1280 bil      0.24501 s    16.72 Mpx/s    66.7 %
to 2048x1280 bic      0.30398 s    13.47 Mpx/s    49.7 %
to 2048x1280 lzs      0.37300 s    10.98 Mpx/s    41.7 %
to 5478x3424 bil      1.06362 s     3.85 Mpx/s    40.1 %
to 5478x3424 bic      1.32330 s     3.10 Mpx/s    39.4 %
to 5478x3424 lzs      1.56232 s     2.62 Mpx/s    31.2 %
```

第四列显示了性能改进情况。

