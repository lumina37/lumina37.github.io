---
title: 源码解析 - ggml与ncnn中的Vulkan矩阵乘实现
categories: hpc
date: 2025/10/11 13:29:00
mathjax: true
---

# 前言

最近准备写一套面向Compute Shader的高性能算子，其中也包括矩阵乘。为了能够站在巨人的肩膀上，这一系列文章中我会深入解读一下ggml

说到矩阵乘自然离不开Tensor Core。事实上，在NVIDIA所实现的各种加速特性中，Tensor Core几乎是Vulkan唯一支持的。其他特性如TMA、Wrap Specialization、Thread Group Cluster、绕过L1/L2缓存的异步load/store等等，都不受当前的Vulkan支持。那么在这篇文章中，我将带大家一起了解一下`VK_NV_cooperative_matrix2`这个专门针对NVIDIA硬件的Vulkan扩展，在`VK_KHR_cooperative_matrix`的基础上，新增了哪些关键特性，可以被用于解决哪些关键问题。

# 参考资料

1. 来自NVIDIA的Jeff Bolz在今年二月的分享Machine Learning in Vulkan with Cooperative Matrix 2：[PPT链接](https://www.vulkan.org/user/pages/09.events/vulkanised-2025/T47-Jeff-Bolz-NVIDIA.pdf)，[演讲视频链接](https://www.bilibili.com/video/BV1BR9uYvEBq/)
2. `GLSL_NV_cooperative_matrix2`的官方文档：[链接](https://github.com/KhronosGroup/GLSL/blob/main/extensions/nv/GLSL_NV_cooperative_matrix2.txt)
3. Vulkan文档中关于`VK_NV_cooperative_matrix2`的一些背景介绍：[链接](https://docs.vulkan.net.cn/features/latest/features/proposals/VK_NV_cooperative_matrix2.html)

# 