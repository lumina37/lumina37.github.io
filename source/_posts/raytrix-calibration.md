---
title: 论文笔记：Automated Robust Metric Calibration Algorithm for 3D Camera Systems
categories: 光场相机
date: 2024/3/2 14:44:00
mathjax: true
---

## 论文链接

[IEEE-Explore](https://ieeexplore.ieee.org/document/7447858)

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

<img alt="散焦式全光相机成像示意图" src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-001.jpg">

上图中穿过主透镜和MLA的光最终汇聚于TCP（Total Covering Plane）。传感器（Sensor）必须置于TCP前，这样各个ML在传感器上呈的像才是一个个小圆像。如果放在TCP上那就直接成点了。这些小圆像被称作MI（Microlens Image）。相邻的MI存在部分相似，这是因为它们都是对同一物体在不同角度的观察。我们可以进一步利用这些相似区域做深度估计。

<img alt="深度估计示意图" src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-002.jpg" width="50%">

这一小节的终极目标就是算出$P_V$点“到相机的距离”$a$，也就是到MLA[主平面](https://zh.wikipedia.org/zh-cn/%E5%9F%BA%E9%BB%9E)的距离。

计算$a$之前要先计算$P_V$点的虚深度$v$。我们需要先知道物体上某一点在两个相邻MI中的位置，如上图分别记作$i_1$和$i_2$。这两个MI对应的ML（Micro Len，微透镜）的投影中心（投影到垂直光轴的虚线上）记为$c_1$和$c_2$。既然知道了传感器上对应物体上同一点的两条光路，我们就可以用几何模型计算出虚深度$v$。

上图中$b$是$P_V$和传感器到MLA主平面的距离，$D$是相邻ML的中心距离。虚深度的计算公式如下：

$$v = \frac{a}{b} = \frac{D}{D-(i_1-i_2)}$$

Raytrix相机还有一个多焦点（Multi-Focus）的特性，即在相机的MLA中存在三种不同焦距的ML。该特性使得相机能获得更大的景深（DoF，Depth of Field），但也会造成一种特殊的“偏差”（aberration），如下图所示，即测量出三种不同的焦平面。

<img alt="存在多种焦距的ML导致的测量偏差" src="https://cdn.jsdelivr.net/gh/Starry-OvO/picx-images-hosting@master/2403_raytrix-calibration/fig-003.jpg">

因为焦平面到MLA主平面的距离与虚深度线性相关，由此我们可以假定对不同的ML有不同的$b$（传感器到MLA主平面的距离）。并用$b_i$表示第$i$种ML对应的$b$。

### Projection model

下图粗略地演示了从MLA左侧的虚深度到主透镜右侧的真实深度的转换过程。
