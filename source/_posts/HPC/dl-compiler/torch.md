---
title: torch.compile选型指南
categories: hpc
date: 2025/12/29 16:31:00
mathjax: true
---

# 前言

`torch.compile`是PyTorch 2.0的核心feature之一。作用是将那些基于PyTorch的Python代码捕获为一张静态计算图，再将这张图融合为一个或多个kernel，也可能进一步转换为CUDA Graph，通过减少kernel调度与执行的overhead以及一些内存布局上的优化来提高执行效率。

## 前置阅读

[Pytorch TORCH.COMPILE 使用指南](https://zhuanlan.zhihu.com/p/620163218)：是PyTorch官方文章[Introduction to `torch.compile`](https://docs.pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)的翻译。文章内容包含**`torch.compile`的基本使用方式**、**初步的工作原理**以及**和现有的TorchScript和FX Tracing等方案的对比**。

看完上面这篇入门的，对下面这篇文章的接受度可能更高一点。

[理解torch.compile基本原理和使用方式](https://zhuanlan.zhihu.com/p/12712224407)：相较上篇文章额外讲解了一点点**最佳实践**和**编译缓存机制**相关的内容。

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


