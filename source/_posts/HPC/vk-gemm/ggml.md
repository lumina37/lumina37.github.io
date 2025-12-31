---
title: 源码解析 - 高性能Vulkan矩阵乘实现
categories: hpc
date: 2025/10/11 13:29:00
mathjax: true
---

# 前言

最近准备写一套基于GLSL Compute Shader的高性能算子，其中也包括矩阵乘。为了能够站在巨人的肩膀上，这一系列文章中我会深入解读一下[ggml](https://github.com/ggml-org/llama.cpp/tree/master/ggml/src/ggml-vulkan/vulkan-shaders)和[ncnn](https://github.com/Tencent/ncnn/tree/master/src/layer/vulkan/shader)这两套框架的GLSL矩阵乘写法，然后再尝试自己写一个性能比较接近cutlass 3.x的矩阵乘算子。

# ggml

先介绍[ggml](https://github.com/ggml-org/ggml)。ggml是llama.cpp的底层张量库。其中实现了包括SIMT、coopmat1、coopmat2三种形式的GEMM。

## SIMT

所谓SIMT是指不使用Tensor Core的传统实现方式。对应的源码位于[mul_mm.comp](https://github.com/ggml-org/ggml/blob/master/src/ggml-vulkan/vulkan-shaders/mul_mm.comp)。

算子的作者把SIMT和Tensor Core的实现放在一块了，随后再通过宏来选择具体的实现，非常混乱。再加上各种诸如`ne02`之类的让人摸不着头脑的缩写命名，属实防御力拉满。我这里将SIMT的部分单独剥离出来放在下面，同时去掉MOE和batch相关的内容（MOE的部分由`MUL_MAT_ID`宏包裹），然后丢给Claude让他再对一些重点变量的含义做一些注释，最后做一些人工修订。

这里补充解释一下ggml中的命名规则，方便阅读。以`ne13`为例：

1. `ne`指num of element。说明`ne13`指的是元素数量而不是字节数量。
2. `1`指1号矩阵也就是B矩阵。0号矩阵对应A矩阵。ggml中的计数都是从0开始的。
3. `3`指3号维度的大小。3号维度就是[N,C,H,W]中的最高维度N。

那么`ne13`的意思就是B矩阵（1号矩阵）的批次数量（3号维度，N维度的大小）。同理，对于行优先的A矩阵，`ne00`就是A矩阵（0号矩阵）的列数（0号维度，W维度的大小）。

```glsl
#version 450

#extension GL_EXT_control_flow_attributes : enable
#extension GL_EXT_shader_16bit_storage : require
#extension GL_EXT_shader_explicit_arithmetic_types_float16 : require

// 设置work group大小
layout(local_size_x_id = 0, local_size_y = 1) in;

// binding 0: 输入矩阵A
layout (binding = 0) readonly buffer A {f16vec4 data_a[];};
// binding 1: 输入矩阵B
layout (binding = 1) readonly buffer B {f16vec4 data_b[];};
// binding 2: 输出矩阵D
layout (binding = 2) writeonly buffer D {float data_d[];};

// Push Constants - 运行时参数（在Pipeline创建后设置）
layout (push_constant) uniform parameter
{
    uint M;              // 矩阵A的行数
    uint N;              // 矩阵B的列数
    uint K;              // 矩阵A的列数 / 矩阵B的行数（内积维度）
    uint stride_a;       // 矩阵A的stride（几乎等于行优先存储时的列元素数）
    uint stride_b;       // 矩阵B的stride
    uint stride_d;       // 输出矩阵D的stride

    uint k_split;        // K维度的分割大小（用于K维度的并行化）
} p;

// Spec Constants - 编译时常量（在Pipeline创建时设置）
layout (constant_id = 0) const uint BLOCK_SIZE = 64;  // 工作组大小（线程数）
layout (constant_id = 1) const uint BM = 64;          // Block tile M维度
layout (constant_id = 2) const uint BN = 64;          // Block tile N维度
layout (constant_id = 3) const uint BK = 16;          // Block tile K维度（量化时假设为32）
layout (constant_id = 4) const uint WM = 32;          // Warp tile M维度
layout (constant_id = 5) const uint WN = 32;          // Warp tile N维度
layout (constant_id = 6) const uint WMITER = 2;       // Warp M维度迭代次数
layout (constant_id = 7) const uint TM = 4;           // Thread tile M维度
layout (constant_id = 8) const uint TN = 2;           // Thread tile N维度
layout (constant_id = 9) const uint TK = 1;           // Thread tile K维度（仅用于coopmat）
layout (constant_id = 10) const uint WARP = 32;       // Warp大小（NVIDIA: 32, AMD: 64）

// 共享内存配置
// Stride加1是为了避免bank conflict
#define SHMEM_STRIDE (BK / 2 + 1)

// 共享内存缓冲区：用于缓存A和B的tile数据
shared f16vec2 buf_a[BM * SHMEM_STRIDE];
shared f16vec2 buf_b[BN * SHMEM_STRIDE];

#define NUM_WARPS (BLOCK_SIZE / WARP)

void load_a_to_shmem(const uint pos_a, const uint row, const uint col, const uint idx_m, const uint block, const uint end_k) {
    const uint idx = pos_a + col * p.stride_a / 4 + row;
    const uint buf_idx = col * SHMEM_STRIDE + row * 4 / 2;
    f16vec4 aa = data_a[idx];
    buf_a[buf_idx] = aa.xy;
    buf_a[buf_idx + 1] = aa.zw;
}

void load_b_to_shmem(const uint pos_b, const uint row, const uint col, const uint idx_n, const uint block, const uint end_k) {
    const uint idx = pos_b + col * p.stride_b / 4 + row;
    const uint buf_idx = col * SHMEM_STRIDE + row * 4 / 2;
    f16vec4 bb = data_b[idx];
    buf_b[buf_idx + 0] = bb.xy;
    buf_b[buf_idx + 1] = bb.zw;
}


void main() {

    // ------------------------------------------------------------------------
    // 1. 计算当前工作组负责的block-tile位置
    // ------------------------------------------------------------------------
    const uint blocks_m = (p.M + BM - 1) / BM;    // M维度需要多少个block
    const uint ir = gl_WorkGroupID.x % blocks_m;  // 当前block的行索引
    const uint ik = gl_WorkGroupID.x / blocks_m;  // K维度的split索引
    const uint ic = gl_WorkGroupID.y;             // 当前block的列索引

    // ------------------------------------------------------------------------
    // 2. 计算warp/thread-level的分块配置
    // ------------------------------------------------------------------------
    const uint WNITER = (WM * WN) / (WARP * TM * TN * WMITER);  // warp-level的N维度迭代次数
    const uint WSUBM = WM / WMITER;  // warp-level子块的M维度大小
    const uint WSUBN = WN / WNITER;  // warp-level子块的N维度大小

    // 当前线程所在的warp索引
    const uint warp_i = gl_LocalInvocationID.x / WARP;

    // 当前线程在warp内的索引
    const uint tiw = gl_LocalInvocationID.x % WARP;

    // thread在warp-tile内的行列位置
    const uint tiwr = tiw % (WSUBM / TM);  // 行索引
    const uint tiwc = tiw / (WSUBM / TM);  // 列索引

    // Warp在block内的位置
    const uint warp_r = warp_i % (BM / WM);  // 行索引
    const uint warp_c = warp_i / (BM / WM);  // 列索引

    // ------------------------------------------------------------------------
    // 3. 计算数据加载的索引
    // ------------------------------------------------------------------------
    // 每个线程负责加载的位置
    const uint loadr_a = gl_LocalInvocationID.x % (BK / 4);
    const uint loadc_a = gl_LocalInvocationID.x / (BK / 4);
    const uint loadr_b = gl_LocalInvocationID.x % (BK / 4);
    const uint loadc_b = gl_LocalInvocationID.x / (BK / 4);

    // 加载时的stride（多个线程协作加载）
    const uint loadstride_a = gl_WorkGroupSize.x * 4 / BK;
    const uint loadstride_b = gl_WorkGroupSize.x * 4 / BK;

    // ------------------------------------------------------------------------
    // 4. 计算K维度的处理范围（用于K维度并行化）
    // ------------------------------------------------------------------------
    const uint start_k = ik * p.k_split;
    const uint end_k = min(p.K, (ik + 1) * p.k_split);

    // 计算全局内存中A和B的起始位置
    uint pos_a = (ir * BM * p.stride_a + start_k) / 4;
    uint pos_b = (ic * BN * p.stride_b + start_k) / 4;

    // ------------------------------------------------------------------------
    // 5. 初始化累加器和缓存
    // ------------------------------------------------------------------------
    float sums[WMITER * TM * WNITER * TN];   // 累加结果数组
    f16vec2 cache_a[WMITER * TM];       // A矩阵的寄存器缓存
    f16vec2 cache_b[TN];                // B矩阵的寄存器缓存

    // 清零累加器
    [[unroll]] for (uint i = 0; i < WMITER*TM*WNITER*TN; i++) {
        sums[i] = float(0.0f);
    }

    // ------------------------------------------------------------------------
    // 6. 主循环：遍历K维度的所有block
    // ------------------------------------------------------------------------
    for (uint block = start_k; block < end_k; block += BK) {
        // 阶段1：从全局内存加载数据到共享内存
        [[unroll]] for (uint l = 0; l < BM; l += loadstride_a) {
            load_a_to_shmem(pos_a, loadr_a, loadc_a + l, ir * BM + loadc_a + l, block, end_k);
        }
        [[unroll]] for (uint l = 0; l < BN; l += loadstride_b) {
            load_b_to_shmem(pos_b, loadr_b, loadc_b + l, ic * BN + loadc_b + l, block, end_k);
        }

        // 同步：确保所有线程都完成了数据加载
        barrier();

        // 更新全局内存指针，准备下一个block
        pos_a += BK / 4;
        pos_b += BK / 4;

        // 阶段2：计算当前block的矩阵乘法
        // 遍历BK维度（除以二是因为每次处理2个逻辑元素）
        [[unroll]] for (uint i = 0; i < BK / 2; i++) {
            // 从共享内存加载到寄存器缓存
            [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {
                [[unroll]] for (uint j = 0; j < TM; j++) {
                    cache_a[wsir * TM + j] = buf_a[(warp_r * WM + wsir * WSUBM + tiwr * TM + j) * SHMEM_STRIDE + i];
                }
            }
            
            // 对N维度进行迭代计算
            [[unroll]] for (uint wsic = 0; wsic < WNITER; wsic++) {
                // 加载B矩阵数据
                [[unroll]] for (uint j = 0; j < TN; j++) {
                    cache_b[j] = buf_b[(warp_c * WN + wsic * WSUBN + tiwc * TN + j) * SHMEM_STRIDE + i];
                }

                // 执行外积累加：C[m,n] += A[m,k] * B[k,n]
                [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {
                    [[unroll]] for (uint cc = 0; cc < TN; cc++) {
                        [[unroll]] for (uint cr = 0; cr < TM; cr++) {
                            const uint sums_idx = (wsic * TN + cc) * (WMITER * TM) + wsir * TM + cr;
                            // 使用FMA指令：fma(a,b,c) = a*b+c
                            // 同时处理vec2的x和y分量
                            sums[sums_idx] = fma(float(cache_a[wsir * TM + cr].x), float(cache_b[cc].x), 
                                             fma(float(cache_a[wsir * TM + cr].y), float(cache_b[cc].y), sums[sums_idx]));
                        }
                    }
                }
            }
        }

        // 同步：确保所有线程完成计算后再加载下一个block
        barrier();
    }

    // ------------------------------------------------------------------------
    // 7. 将结果写回全局内存
    // ------------------------------------------------------------------------
    // 计算当前block在全局结果矩阵中的起始位置
    const uint dr = ir * BM + warp_r * WM;
    const uint dc = ic * BN + warp_c * WN;

    // 将累加结果写入全局内存
    [[unroll]] for (uint wsic = 0; wsic < WNITER; wsic++) {
        [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {

            const uint dr_warp = dr + wsir * WSUBM + tiwr * TM;
            const uint dc_warp = dc + wsic * WSUBN + tiwc * TN;
            
            // 写入每个thread tile的结果
            [[unroll]] for (uint cc = 0; cc < TN; cc++) {
                [[unroll]] for (uint cr = 0; cr < TM; cr++) {
                    // 边界检查：确保不越界
                    if (dr_warp + cr < p.M && dc_warp + cc < p.N) {
                        data_d[(dc_warp + cc) * p.stride_d + dr_warp + cr] = float(sums[(wsic * TN + cc) * (WMITER * TM) + wsir * TM + cr]);
                    }
                }
            }
        }
    }
}
```

大体上的思路和Simon的[这篇博客](https://siboehm.com/articles/22/CUDA-MMM)中演示的wrap-tile优化很像，都是做了Block-Wrap-Thread的三级分块，如下图所示。

<img src="https://siboehm.com/assets/img/CUDA-MMM/kernel_10_warp_tiling.png">

我也在我的demo仓库里复刻了ggml的这个版本

## coopmat1

TODO

说到矩阵乘，自然离不开Tensor Core。事实上，在NVIDIA所实现的各种加速特性中，Tensor Core几乎是Vulkan唯一支持的。其他特性如TMA、Wrap Specialization、Thread Group Cluster、绕过L1/L2缓存的异步load/store等等，都不受当前的Vulkan支持。Vulkan通过`VK_KHR_cooperative_matrix`（下称coopmat1）和`VK_NV_cooperative_matrix2`（下称coopmat2）这两个扩展来调用Tensor Core。

本小节我们先来阅读基于coopmat1扩展的GEMM写法。同上一小节，我们先对源码做一下宏替换，不然太难读了。
