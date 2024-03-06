---
title: 论文笔记：Automated Robust Metric Calibration Algorithm for 3D Camera Systems
categories: 光场相机
date: 2024/3/2 14:44:00
mathjax: true
---

## 论文链接

[IEEE Explore](https://ieeexplore.ieee.org/document/7447858)

[官方PDF](https://raytrix.de/wp-content/uploads/software/Automated-Robust-Metric-Calibration-Algorithm-for.pdf)

## 写在前面

这篇文章是光场相机标定（Light Field Camera Calibration）的经典。对光场相机的基础知识介绍较为详尽，方便快速入门。所提出的方法也已经历过生产实践的检验。

## Abstract

开头是一段对光场相机应用的背景介绍，这里直接略过。

想实现深度估计就必须做相机标定（Camera Calibration）。本文提出了一个稳健的、自动化的相机标定技术。

## Introduction

第一段简要介绍了一下光场相机的发展历程。

第二段提到Raytrix以“虚深度”（Virtual Depth）的形式提供物到相机的距离。并且引用[10](https://www.researchgate.net/publication/258713151_Single_Lens_3D-Camera_with_Extended_Depth-of-Field)详细讲解了Raytrix相机。虚深度的概念会在后面图解，你可以暂时把它理解为一个变量与随相机型号变化的常量的比值。

第三四段简要介绍了先前提出的光场标定算法。

第五段，作者声明这个新提出的标定算法更精确地建模了MLA（Micro Lens Array，微透镜阵列），由此不仅可以更准确地度量三维空间，更可以减少深度图中的噪声。

倒数第二段提到，这篇期刊论文是先前一篇会议论文[14](https://ieeexplore.ieee.org/document/7151596)的改进版——作者新引入了主透镜偏斜、偏移情况下的建模。

最后一段介绍文章结构。

## Theoretical Foundations

这一大章都在讲理论基础。因为我也刚入门，所以就全篇翻译一下。

### Depth estimation

（这里我怀疑作者放错图了，下图应当是散焦式全光相机的成像图，而Plenoptic 2.0是聚焦式的，佐证可见[此处](https://www.researchgate.net/publication/241391634_Resolution_in_Plenoptic_Cameras/figures?lo=1)）

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-001.jpg" alt="散焦式全光相机成像示意图">

上图中穿过主透镜和MLA的光最终汇聚于TCP（Total Covering Plane）。传感器（Sensor）必须置于TCP前，这样各个ML在传感器上呈的像才是一个个小圆像。如果放在TCP上那就直接成点了。这些小圆像被称作MI（Microlens Image）。相邻的MI存在部分相似，这是因为它们都是对同一物体在不同角度的观察。我们可以进一步利用这些相似区域做深度估计。

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-002.jpg" alt="深度估计示意图" width="50%">

这一小节的终极目标就是算出$P_V$点“到相机的距离”$a$，也就是到MLA[主平面](https://zh.wikipedia.org/zh-cn/%E5%9F%BA%E9%BB%9E)的距离。

计算$a$之前要先计算$P_V$点的虚深度$v$。我们需要先知道物体上某一点在两个相邻MI中的位置，如上图分别记作$i_1$和$i_2$。这两个MI对应的ML（Micro Len，微透镜）的投影中心（投影到垂直光轴的虚线上）记为$c_1$和$c_2$。既然知道了传感器上对应物体上同一点的两条光路，我们就可以用几何模型计算出虚深度$v$。

上图中$b$是$P_V$和传感器到MLA主平面的距离，$D$是相邻ML的中心距离。虚深度的计算公式如下：

$$v = \frac{a}{b} = \frac{D}{D-(i_1-i_2)}$$

Raytrix相机还有一个多焦点（Multi-Focus）的特性，即在相机的MLA中存在三种不同焦距的ML。该特性使得相机能获得更大的景深（DoF，Depth of Field），但也会造成一种特殊的“偏差”（aberration），如下图所示，即测量出三种不同的焦平面。

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-003.jpg" alt="存在多种焦距的ML导致的测量偏差">

因为焦平面到MLA主平面的距离与虚深度线性相关，由此我们可以假定对不同的ML有不同的$b$（传感器到MLA主平面的距离）。并用$b_i$表示第$i$种ML对应的$b$。

### Projection model

下图粗略地演示了从MLA左侧的虚深度到主透镜右侧的真实深度的转换过程。该图同样展示了投影过程中各个空间的名称。

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-004.jpg" alt="从虚深度到真实深度的投影转换示意图">

空间Ⅰ中的点由“横向”（这里指的是垂直主光轴的方向，搞不懂为什么要叫lateral）的坐标再加上虚深度来表示。空间Ⅱ中的点是由空间Ⅰ投影而来的真实坐标，相当于去除了MLA的作用。空间Ⅲ的点为去变形（undistorted）之后，位于主透镜焦平面上的点。空间Ⅳ的点为对应的真实物体上的点，也反映了真实深度。这些点的$x$、$y$坐标轴与主光轴垂直，$z$坐标轴与主光轴平行。感觉原始概念有些难以理解，还是来看公式吧。

由于虚深度对应了空间Ⅰ中的坐标$(x,y,z)_Ⅰ$，因此我们可以利用公式$v = \frac{a}{b_i}$投影得到空间Ⅱ中的坐标$(x,y,z)_Ⅱ$：

$$x_Ⅱ = x_Ⅰ \cdot b_i$$
$$y_Ⅱ = y_Ⅰ \cdot b_i$$
$$z_Ⅱ = z_Ⅰ \cdot b_i$$

然后是从空间Ⅱ到空间Ⅲ的去变形步骤。这里的变形由主透镜的倾斜（Tilt）和偏移（Shift）导致。需要注意的是，主透镜倾斜和偏移产生的变形效果各自独立。作者这里使用了来自引用15（Decentering distortion of lenses，作者Brown，我没找到原文）的方法来解决偏移变形；而倾斜变形可以通过简单的旋转变换纠正。

为建模主透镜在倾斜和偏移状态下的3D位姿，以下定义一系列变量。$\theta_L$、$\sigma_L$用于表示主透镜的朝向角；$X_L$、$Y_L$用于表示主透镜中心相对传感器中心的坐标偏移；而主透镜与传感器之间的距离被定义为$Z_L$。这个$Z_L$不需要特意求取，因为我们后面的计算只会用到主透镜与TCP之间的距离$B_L$。变量的可视化如下图所示：

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-007.jpg" alt="变量可视化">

对于光场相机而言，主透镜的倾斜会导致最终成像的倾斜，如下图所示。这一效应被称为沙姆定律（Scheimpflug Principle）。关于沙姆定律的一些科普可以看这篇知乎文章《[移轴摄影、沙姆定律与射影几何](https://zhuanlan.zhihu.com/p/25030168)》。

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-006.jpg" alt="倾斜的主透镜">

以沙姆定律为基础，我们可以通过一次旋转来纠正倾斜变形。这次旋转的旋转轴应与主光轴垂直。

接下来我们使用引用15的方法来解决偏移变形。其作者Brown认为变形量由两个变形系数$k_1$、$k_2$定义。由空间Ⅱ到空间Ⅲ的$x$、$y$坐标转换公式如下：

$$x_Ⅲ = x_Ⅱ \cdot (1 + k_1 r^2 + k_2 r^4)$$
$$y_Ⅲ = y_Ⅱ \cdot (1 + k_1 r^2 + k_2 r^4)$$

其中$r$表示空间Ⅱ中的点到“变形中心”（distortion center，主光轴与传感器平面的交点）的欧氏距离（坐标平方和开方）。

然后我们需要做径向深度去变形（radial depth undistortion）。这部分的变形量由与$r$绑定的系数$d_1$、$d_2$，以及与虚深度$z_d$（不明白这里为什么不用$v_d$）绑定的系数$d_d$定义。其中虚深度对变形量的增益是线性的。这一步去变形对应的公式如下：

$$z_Ⅲ' = z_Ⅱ + (1 + z_d d_d) \cdot (d_1 r^2 + d_2 r^4)$$

注意$z_Ⅲ'$度量的是到MLA的距离，而空间Ⅲ中的距离应当从主透镜的主平面起算。为了获得“真正的”$z_Ⅲ$，首先计算主透镜与TCP之间的距离$B_L$：

$$B_L = \frac{T_L}{2} \left( 1 - \sqrt{1 - 4 \frac{f_L}{T_L}} \right)$$

其中$f_L$是主透镜的焦距，$T_L$为主透镜的对焦距离。这里各种距离的概念有点多，为方便读者理解我把上面的那张成像示意图再贴一遍。

<img src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-001.jpg" alt="散焦式全光相机成像示意图">

现在，只需要把从MLA起算的深度$z_Ⅲ'$与主透镜到TCP的距离$B_L$加起来，再减去MLA到TCP的距离就能得到从主透镜起算的深度，亦即空间Ⅲ中的深度$z_Ⅲ$了。而根据光场相机的设计理论，TCP到传感器的距离恰好等于传感器与MLA之间的间距$b_i$。于是我们可以写出$z_Ⅲ$的表达式：

$$z_Ⅲ = z_Ⅲ' + B_L - 2b_i$$

最后使用薄透镜理论

$$\frac{1}{f_L} = \frac{1}{G_L} + \frac{1}{B_L}$$

将$z_Ⅲ$投影到物方的空间Ⅳ中即可。

## Implementation

这章主要讲光场相机标定算法的实现，以及标定参照物的设计。对于传统相机的标定，棋盘格是常用的参照物。但对于光场相机，当棋盘格的边界与[对极线](https://zh.wikipedia.org/wiki/%E5%AF%B9%E6%9E%81%E5%87%A0%E4%BD%95)（epipolar lines）平行时将无法估计深度。换用圆形图案的参照物可以解决这一问题。

我们使用MSER算法来从图像中提取圆形图案特征。随后使用引用18（Design and test of a calibration method for the calculation of metrical range values for 3D light field cameras，一篇硕士学位论文，找不到链接）中给出的算法将特征点对齐到网格并将相邻特征点的像素间距关联到真实距离。

在这之后，我们获得了两组点集，第一组是从拍摄图像中提取的，与虚深度关联的目标点集（target points）；第二组是对齐到间距已知的矩形网格的模型点集（model points）。并且两组点之间存在一一对应的关系。利用这一关系我们便能完成对相机内参（intrinsic parameters，也称固有参数）的优化。

利用上一章中提到的方法，使用一组初始内参将目标点集投影到空间Ⅳ中。如果内参取值恰当，那么投影后的目标点应当与其实际所在的位置重合。而这一实际位置可由$z_Ⅳ=0$平面上的模型点集经过一仿射变换表示。为计算归一化的损失，不妨对目标点集施加一个仿射逆变换，使其变换到$z_Ⅳ=0$平面上，再利用变换后的目标点与模型点之间的距离来计算归一化损失。

（本节的最后一段未翻译，留待后续阅读理解）