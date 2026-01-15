---
title: torch.compile选型指南
categories: hpc
date: 2025/12/29 16:31:00
mathjax: true
---

# 前言

`torch.compile`是PyTorch 2.0的核心feature之一。

## 前置阅读

[Pytorch TORCH.COMPILE 使用指南](https://zhuanlan.zhihu.com/p/620163218)：是PyTorch官方文章[Introduction to `torch.compile`](https://docs.pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)的翻译。文章内容包含`torch.compile`的基本使用方式、初步的工作原理以及和现有的TorchScript和FX Tracing等方案的对比。

看完上面这篇入门的，对下面这篇文章的接受度可能更高一点。

[理解torch.compile基本原理和使用方式](https://zhuanlan.zhihu.com/p/12712224407)：相较上篇文章额外讲解了一点点**最佳实践**和**编译缓存机制**相关的内容。
