---
title: torch.compile选型指南
categories: hpc
date: 2025/12/29 16:31:00
mathjax: true
---

# 前言

`torch.compile`是PyTorch 2.0的核心特性之一。

compile这个词源自古拉丁语com-pilāre。com-意为共同，-pilāre意为搜刮，收集。合起来的意思是把零散的东西地毯式地，不遗漏地搜刮，收集到一起。后面衍生出编纂、汇编的含义。可以作为类比的是C/C++中的汇编含义——编译器会无遗漏地搜集用户给定的高级语言代码，在不改变其语义的前提下，将高级语言转换为机器语言的形式。而`torch.compile`的功能则是无遗漏地收集Python代码，在不改变其原有语义的前提下，将其转换为一个更高效的形式。

目前`torch.compile`的实现方式是：首先将那些基于PyTorch的Python代码捕获为一张静态计算图，再针对这张图做一些优化以提高执行效率，最终转换为GPU上的机器语言，也就是kernel*。因为这个特性实在是太好上手了，而且效果还凑合，所以作者就以这篇关于`torch.compile`的文章的撰写作为对ai编译的一个入门。

*更更具体地说，`torch.compile`通过将静态计算图转换为一个或多个kernel，也可能进一步记录为CUDA Graph，通过减少kernel调度执行的overhead、减少一些不必要的global memory读写以及一些内存布局上的优化来提高执行效率。其中kernel调度执行问题一般指的是Python的执行效率，以及一些kernel发射的CPU overhead。

## 前置阅读

[Pytorch TORCH.COMPILE 使用指南](https://zhuanlan.zhihu.com/p/620163218)：是PyTorch官方文章[Introduction to `torch.compile`](https://docs.pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)的翻译。文章内容包含`torch.compile`的**基本使用方式**、**初步的工作原理**以及**和现有的TorchScript、FX Tracing等方案的对比**。

看完上面这篇入门的，对下面这篇文章的接受度可能更高一点。

[torch.compile 技术剖析：PyTorch 2.x 编译系统详解](https://zhuanlan.zhihu.com/p/1968068966934095005)：相较上篇文章额外讲解了非常非常多的细节。

# 流程初探

## 流程图

仅考虑正向推理，不考虑反向传播，整个`torch.compile`的流程大致由以下三步组成：

TorchDynamo捕获FX Graph（静态图）→TorchInductor将FX Graph转换为ATen IR→TorchInductor实施一系列计算图优化→生成Triton等IR→编译到kernel

Talk is cheap，下面我们来动手搭一个demo，近距离观察`torch.compile`内部各个环节的输入输出。

## 脚手架搭建

目前作者使用的脚手架是https://github.com/lumina37/tcompile

pyproject.toml如下：

```toml
[project]
name = "tcompile"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
authors = [{ name = "lumina37", email = "starry.qvq@gmail.com" }]
requires-python = ">=3.13,<3.14"
dependencies = ["torch", "torchvision", "triton;sys_platform!='win32'", "triton-windows;sys_platform=='win32'"]

[build-system]
requires = ["uv_build>=0.9.25,<0.10.0"]
build-backend = "uv_build"

[tool.uv.sources]
torch = [{ index = "pytorch-cu130", marker = "platform_system != 'Darwin'" }]
torchvision = [{ index = "pytorch-cu130", marker = "platform_system != 'Darwin'" }]

[[tool.uv.index]]
name = "pytorch-cu130"
url = "https://download.pytorch.org/whl/cu130"
explicit = true

[tool.ruff]
line-length = 120
target-version = "py313"
```

## 观察FX Graph

目前如果要观察TorchDynamo所捕获到的FX Graph，有两种方式：

1. 设置`TORCH_LOGS`环境变量为`graph`
2. 调用`torch._logging.set_logs(graph=True)`

二者效果一致。

在FX Graph捕获完毕后，TorchInductor将FX Graph转换为ATen IR前，用户依然有机会插入一些自定义的模板替换操作。不过这里不做深入，还是以把握默认行为为主。

## 观察ATen IR


