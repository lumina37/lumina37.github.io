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

算子的作者把SIMT和Tensor Core的实现放在一块了，随后再通过宏来选择具体的实现，非常混乱。再加上各种诸如`ne02`之类的让人摸不着头脑的缩写命名，属实是防御性拉满。我这里将SIMT的部分单独剥离出来放在下面，同时去掉MOE相关的内容（由`MUL_MAT_ID`宏包裹），再对一些重点变量的含义做一些注释，方便阅读。

```glsl
#version 450

#extension GL_EXT_control_flow_attributes : enable
#extension GL_EXT_shader_16bit_storage : require

#ifdef FLOAT16
#extension GL_EXT_shader_explicit_arithmetic_types_float16 : require
#endif
#if defined(DATA_A_IQ1_M)
#extension GL_EXT_shader_explicit_arithmetic_types_int16 : require
#endif

#include "types.glsl"

#ifndef LOAD_VEC_A
#define LOAD_VEC_A 1
#endif
#ifndef LOAD_VEC_B
#define LOAD_VEC_B 1
#endif

// Load 2 values at once without affecting index calculations through LOAD_VEC
#if (defined(DATA_A_F32) || defined(DATA_A_F16) || defined(DATA_A_BF16)) && !defined(ALIGNED)
#define LOAD_VEC_BATCH_A 2
#else
#define LOAD_VEC_BATCH_A 1
#endif
#if !defined(ALIGNED)
#define LOAD_VEC_BATCH_B 2
#else
#define LOAD_VEC_BATCH_B 1
#endif

#if !defined(TO_FLOAT_TYPE)
#define TO_FLOAT_TYPE FLOAT_TYPE
#endif

layout(local_size_x_id = 0, local_size_y = 1, local_size_z = 1) in;

layout (binding = 0) readonly buffer A {A_TYPE data_a[];};
#if defined(A_TYPE_PACKED16)
layout (binding = 0) readonly buffer A_PACKED16 {A_TYPE_PACKED16 data_a_packed16[];};
#endif
#if defined(A_TYPE_PACKED32)
layout (binding = 0) readonly buffer A_PACKED32 {A_TYPE_PACKED32 data_a_packed32[];};
#endif

layout (binding = 1) readonly buffer B {B_TYPE data_b[];};
layout (binding = 2) writeonly buffer D {D_TYPE data_d[];};

layout (push_constant) uniform parameter
{
    uint M;
    uint N;
    uint K;
    uint stride_a;
    uint stride_b;
    uint stride_d;

    uint batch_stride_a;
    uint batch_stride_b;
    uint batch_stride_d;

    uint k_split;
    uint ne02;
    uint ne12;
    uint broadcast2;
    uint broadcast3;
} p;

layout (constant_id = 0) const uint BLOCK_SIZE = 64;
layout (constant_id = 1) const uint BM = 64;
layout (constant_id = 2) const uint BN = 64;
layout (constant_id = 3) const uint BK = 16;  // Assumed to be 32 if working with a quant
layout (constant_id = 4) const uint WM = 32;
layout (constant_id = 5) const uint WN = 32;
layout (constant_id = 6) const uint WMITER = 2;
layout (constant_id = 7) const uint TM = 4;
layout (constant_id = 8) const uint TN = 2;
layout (constant_id = 9) const uint TK = 1;  // Only needed for coopmat
layout (constant_id = 10) const uint WARP = 32;

#define SHMEM_STRIDE (BK / 2 + 1)

shared FLOAT_TYPE_VEC2 buf_a[BM * SHMEM_STRIDE];
shared FLOAT_TYPE_VEC2 buf_b[BN * SHMEM_STRIDE];

#define NUM_WARPS (BLOCK_SIZE / WARP)

#include "mul_mm_funcs.glsl"

void main() {
    const uint batch_idx = gl_GlobalInvocationID.z;

    const uint i13 = batch_idx / p.ne12;
    const uint i12 = batch_idx % p.ne12;

    const uint i03 = i13 / p.broadcast3;
    const uint i02 = i12 / p.broadcast2;

    const uint batch_idx_a = i03 * p.ne02 + i02;

    const uint blocks_m = (p.M + BM - 1) / BM;
    const uint ir = gl_WorkGroupID.x % blocks_m;
    const uint ik = gl_WorkGroupID.x / blocks_m;
    const uint ic = gl_WorkGroupID.y;

    const uint WNITER = (WM * WN) / (WARP * TM * TN * WMITER);
    const uint WSUBM = WM / WMITER;
    const uint WSUBN = WN / WNITER;

    const uint warp_i = gl_LocalInvocationID.x / WARP;

    const uint tiw = gl_LocalInvocationID.x % WARP;

    const uint tiwr = tiw % (WSUBM / TM);
    const uint tiwc = tiw / (WSUBM / TM);

    const uint warp_r = warp_i % (BM / WM);
    const uint warp_c = warp_i / (BM / WM);

    const uint loadr_a = gl_LocalInvocationID.x % (BK / LOAD_VEC_A / LOAD_VEC_BATCH_A);
    const uint loadc_a = gl_LocalInvocationID.x / (BK / LOAD_VEC_A / LOAD_VEC_BATCH_A);
    const uint loadr_b = gl_LocalInvocationID.x % (BK / LOAD_VEC_B / LOAD_VEC_BATCH_B);
    const uint loadc_b = gl_LocalInvocationID.x / (BK / LOAD_VEC_B / LOAD_VEC_BATCH_B);

    const uint loadstride_a = gl_WorkGroupSize.x * LOAD_VEC_A * LOAD_VEC_BATCH_A / BK;
    const uint loadstride_b = gl_WorkGroupSize.x * LOAD_VEC_B * LOAD_VEC_BATCH_B / BK;

    const uint start_k = ik * p.k_split;
    const uint end_k = min(p.K, (ik + 1) * p.k_split);

    uint pos_a = (
        batch_idx_a * p.batch_stride_a +
        ir * BM * p.stride_a + start_k) / LOAD_VEC_A;

    uint pos_b = (batch_idx * p.batch_stride_b + ic * BN * p.stride_b + start_k) / LOAD_VEC_B;

    ACC_TYPE sums[WMITER * TM * WNITER * TN];
    FLOAT_TYPE_VEC2 cache_a[WMITER * TM];
    FLOAT_TYPE_VEC2 cache_b[TN];

    [[unroll]] for (uint i = 0; i < WMITER*TM*WNITER*TN; i++) {
        sums[i] = ACC_TYPE(0.0f);
    }

    for (uint block = start_k; block < end_k; block += BK) {
        [[unroll]] for (uint l = 0; l < BM; l += loadstride_a) {
            load_a_to_shmem(pos_a, loadr_a, loadc_a + l, ir * BM + loadc_a + l, block, end_k);
        }
        [[unroll]] for (uint l = 0; l < BN; l += loadstride_b) {
            load_b_to_shmem(pos_b, loadr_b, loadc_b + l, ic * BN + loadc_b + l, block, end_k);
        }

        barrier();

        pos_a += BK / LOAD_VEC_A;
        pos_b += BK / LOAD_VEC_B;

        [[unroll]] for (uint i = 0; i < BK / 2; i++) {
            // Load from shared into cache
            [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {
                [[unroll]] for (uint j = 0; j < TM; j++) {
                    cache_a[wsir * TM + j] = buf_a[(warp_r * WM + wsir * WSUBM + tiwr * TM + j) * SHMEM_STRIDE + i];
                }
            }
            [[unroll]] for (uint wsic = 0; wsic < WNITER; wsic++) {
                [[unroll]] for (uint j = 0; j < TN; j++) {
                    cache_b[j] = buf_b[(warp_c * WN + wsic * WSUBN + tiwc * TN + j) * SHMEM_STRIDE + i];
                }

                [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {
                    [[unroll]] for (uint cc = 0; cc < TN; cc++) {
                        [[unroll]] for (uint cr = 0; cr < TM; cr++) {
                            const uint sums_idx = (wsic * TN + cc) * (WMITER * TM) + wsir * TM + cr;
                            sums[sums_idx] = fma(ACC_TYPE(cache_a[wsir * TM + cr].x), ACC_TYPE(cache_b[cc].x), fma(ACC_TYPE(cache_a[wsir * TM + cr].y), ACC_TYPE(cache_b[cc].y), sums[sums_idx]));
                        }
                    }
                }
            }
        }

        barrier();
    }

    const uint dr = ir * BM + warp_r * WM;
    const uint dc = ic * BN + warp_c * WN;

    const uint offsets = batch_idx * p.batch_stride_d + ik * p.batch_stride_d * gl_NumWorkGroups.z;

    [[unroll]] for (uint wsic = 0; wsic < WNITER; wsic++) {
        [[unroll]] for (uint wsir = 0; wsir < WMITER; wsir++) {

            const uint dr_warp = dr + wsir * WSUBM + tiwr * TM;
            const uint dc_warp = dc + wsic * WSUBN + tiwc * TN;
            [[unroll]] for (uint cc = 0; cc < TN; cc++) {
                [[unroll]] for (uint cr = 0; cr < TM; cr++) {
                    if (dr_warp + cr < p.M && dc_warp + cc < p.N) {
                        data_d[offsets + (dc_warp + cc) * p.stride_d + dr_warp + cr] = D_TYPE(sums[(wsic * TN + cc) * (WMITER * TM) + wsir * TM + cr]);
                    }
                }
            }
        }
    }
}
```

## coopmat1

TODO

说到矩阵乘，自然离不开Tensor Core。事实上，在NVIDIA所实现的各种加速特性中，Tensor Core几乎是Vulkan唯一支持的。其他特性如TMA、Wrap Specialization、Thread Group Cluster、绕过L1/L2缓存的异步load/store等等，都不受当前的Vulkan支持。Vulkan通过`VK_KHR_cooperative_matrix`（下称coopmat1）和`VK_NV_cooperative_matrix2`（下称coopmat2）这两个扩展来调用Tensor Core。

本小节我们先来阅读基于coopmat1扩展的GEMM写法。同上一小节，我们先对源码做一下宏替换，不然太难读了。
