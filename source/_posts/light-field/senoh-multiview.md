---
title: 对Senoh的聚焦式光场图像多视角转换相关工作的解读
categories: LightField
date: 2024/5/17 17:56:00
mathjax: true
---

## 前言

本文主要解读Takanori Senoh在聚焦式光场图像的多视角转换方向上做的一系列工作。我们假定阅读者已经掌握了一定的背景知识。下面直接切入正题。

## m57272 - Conversion of Single-focused Plenoptic 2.0 Lenslet Image to Multiview Image

[文章链接](https://dms.mpeg.expert/doc_end_user/current_document.php?id=79464)

### Introduction

文章首先点明主旨：本文引入了一个新的Lenslet to Multiview转换软件。然后介绍了一些背景，列举了一些创新点。最后是一大堆输入参数的简写，下文中我们会将它们全部替换为人类可读的形式。

### Lenslet Image to Multi-view Conversion

整个转换过程可以被拆分为五小步

1. 读标定数据并确定lenslet结构。
2. 亮度校正。
3. 估计patch尺寸。
4. 根据输入参数平滑patch尺寸图。
5. 提取patch再拼合为multi-view图像。

### Lenslet Image Parameter

本小节对应步骤：读标定数据并确定lenslet结构。

待确定的参数中，horizontal/vertical lenslet pitch ($ld$/$lp$)指的是每行中各个相邻的MI的间距（$ld$，约等于直径），以及两行的间距（$lp$，约等于$\sqrt{3}/2$倍直径）。后面作者又用connecting来描述一行中MI相紧邻的现象，以及staggered来描述行与行之间交错的位置关系。

各参数对应的物理含义如下图所示。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/00-ll-structure.png" alt="00-ll-structure" />

第一步，获取四个角上的MI的坐标$(x_i,y_i), \ i=1,2,3,4$。

第二步，计算连续（connecting）的一行中MI的数量$L_W$，以及交错（staggered）行的数量$L_H$。

第三步，计算每个连续行中相邻MI的间距$ld$，以及交错行之间的间距$lp$。

第四步，计算整个光场图像的旋转角$rot$。

第五第六步他们用了一套非常复杂的方法来确定其他MI的中心，这里略过。

### Luminance Compensation

本小节对应步骤：亮度校正。

因为MI的边缘区域（Fringe Area）存在亮度衰减（Luminance Decay），因此需要在这些区域实施亮度补偿。

首先，通过下图所示的左上、右上、左下、右下和当前MI这五个位置计算亮度衰减曲线。其中中心MI应当为纯色。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/10-selection.jpg" alt="10-selection" />

下图为亮度衰减曲线。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/20-decay-curve.png" alt="20-decay-curve" />

选取衰减速率最小的（shallowest）的一条曲线，并使用一次函数（疑似）拟合其衰减部分。然后我就看不懂了，为什么亮度衰减可以只出现在图像的一半区域？为什么需要分别计算起始斜率和终止斜率？

随后作者还说除了MI内的衰减，在图像的某些大块区域还存在方向性的亮度衰减。我已经看不懂了，跳过。

### Patch Size (Depth) from Matching Distance

本小节对应步骤：估计patch尺寸。

文中Main Lens Image意为主透镜所成实像。下图展示了主透镜实像经过微透镜阵列在传感器上成像的过程。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/30-imaging.png" alt="30-imaging" />

主透镜实像上的同一点在传感器上的距离记作$md$（matching distance，如下图所示）。$ps$即patch大小（patch size），$D$为各个patch在主透镜实像上对应的大小。并且二者存在几何关系$ps=sD/t$。只要按照正确的$ps$取出这些patch再将它们缩放至大小$D$并拼合，我们就能无缝地还原主透镜图像。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/40-md.jpg" alt="40-md" />

计算patch size的流程如下：

第一步，手动敲定mdMax和mdMin，也就是$md$的取值范围，以使得匹配区域不超出MI边界。

第二步，确定如下图所示的大小为$2X \times 2Y$的最大边缘搜索区域（maximum edge search area）。这个图比较抽象，是上中下三个MI叠在一起的效果，其中edge search area来自中间MI，其余两个关联边缘区域（corresponding edge area）分别来自上下两个MI。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/50-edge-search-area.png" alt="50-edge-search-area" />

然后就是让我比较懵逼的一句话：“由于MI中的补丁尺寸是共通的，因此在计算匹配误差时采用了最陡的边缘，这样既减少了补丁尺寸的搜索，又提高了补丁尺寸的可靠性。在选择最小误差进行立体匹配时，该边缘必须包含在三个连接的小透镜中。”虽然不完全理解这么做的理由，但作者这里应该是想用【同时出现在相邻三个MI中的那个最陡的边缘】来计算$md$。

搜索区域的高度被近似地设定为：

$$Y = 2/3 \left( sR - D_{max} \right) / 3$$

其中sR为safe radius的缩写。作者认为虽然MI边缘已经做了亮度补偿，但其中的纹理依然“不可信”，属于“unsafe area”，并把$sR$设置为$0.4D$，也就是$0.8R$。而$D_{max} = mdMax - D$是为后续的滑动窗口匹配预留的高度。

有了$Y$，搜索区域的宽$X$可直接利用勾股定理求得。

第三步，计算梯度权重图；第四步，估计patch大小。作者认为，梯度$s$越大的区域，纹理可信度越高；纹理越可信，匹配失误所导致的惩罚（$W_s$乘上那一堆DDD的东西）就越大；惩罚越大，接受当前patch尺寸的估计结果所需要的cost就越大。如果【当前帧中某个MI的patch大小的估计结果所对应的cost】大于【前一帧同一位置MI的cost】，则拒绝当前的估计结果，沿用之前的patch大小。除了针对各个MI计算局部cost，作者还会计算一个全局cost。如果全局cost过高，则一次性拒绝当前帧下所有的估计结果，转而采用上一帧的估计结果。

而“匹配失误”是通过当前MI的patch大小与周围MI的patch大小作差得到。m57813中改进了这个量化策略。

### Multi-view Conversion

按原本的MI位置拼贴就行。作者这里还做了一个很复杂的亚像素插值，我感觉1/4像素精度就很可以了。

## m57813 - Conversion of Multi-Focused Plenoptic 2.0 Lenslet Image to Multiview Image

[文章链接](https://dms.mpeg.expert/doc_end_user/current_document.php?id=80273)

该提案以m57272为基础，引入json格式的配置文件，并减少了输入参数。

步骤划分和m57272基本一致。

读标定数据并确定lenslet结构这一步，新提案减少了参数量。

估计patch尺寸这一步，新提案添加了一个额外的前置步骤，用高斯模糊减少原图中的高频噪声。

亮度补偿的步骤被删除，因为从主透镜虚像成像时MI边缘会变得更亮，而从主透镜实像成像时MI边缘会更暗。主透镜实像虚像的区别如下图所示。

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/60-ml-real.png" alt="60-ml-real" />

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/70-ml-virtual.png" alt="70-ml-virtual" />

作者认为兼顾这两种情况的补偿方案太复杂，就干脆只从直径70%的区域提取多视角，亮度异常的区域舍弃不用。

后续，作者还优化了匹配失误的度量步骤，使用当前MI中的待匹配区域（3x3大小）与周围MI中的对应区域作差，差值越大则匹配失误越大，如下图所示：

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2405_senoh-multiview/80-error.png" alt="80-error" />

## Multiview from micro-lens image of multi-focused plenoptic camera

[文章链接](https://ieeexplore.ieee.org/document/9687243)

是m57813的精简版，跳过。
