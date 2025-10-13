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

算子的作者把SIMT和Tensor Core的实现放在一块了，随后再通过宏来选择具体的实现，非常混乱。再加上各种诸如`ne02`之类的让人摸不着头脑的缩写命名，属实防御力拉满。我这里将SIMT的部分单独剥离出来放在下面，同时去掉MOE相关的内容（由`MUL_MAT_ID`宏包裹），然后丢给Claude让他再对一些重点变量的含义做一些注释，方便阅读理解。

```glsl
// 提供输入输出类型的定义
#include "types.glsl"

// LOAD_VEC_A/B: 第一层向量化 - 数据类型层面的向量宽度
// 表示单个"逻辑元素"包含多少个标量
// 例如：
// - 对于量化类型（如Q4_0），一个"逻辑元素"可能包含8个4-bit值 -> LOAD_VEC_A=8
// - 对于普通float32，一个"逻辑元素"就是1个标量 -> LOAD_VEC_A=1
// 这个参数影响索引计算：pos_a / LOAD_VEC_A
#ifndef LOAD_VEC_A
#define LOAD_VEC_A 1
#endif
#ifndef LOAD_VEC_B
#define LOAD_VEC_B 1
#endif

// LOAD_VEC_BATCH_A/B: 第二层向量化 - 加载操作层面的批处理
// 表示一次内存加载操作读取多少个"逻辑元素"
// 为什么叫BATCH？因为它批量处理多个LOAD_VEC单元
// 
// 浮点类型在未对齐时设为2，通过连续读取2个元素来提高带宽利用率
// 其余情况保持1，因为量化类型已经有自己的打包格式
// TODO: 为什么非浮点就不做连续读取，因为int只有int8/int4？
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

// 总结：实际加载宽度 = LOAD_VEC × LOAD_VEC_BATCH
// 例如：对于fp8的矩阵，若LOAD_VEC_A=4, LOAD_VEC_BATCH_A=2则意味着一次操作处理8个fp8

#if !defined(TO_FLOAT_TYPE)
#define TO_FLOAT_TYPE FLOAT_TYPE
#endif

// 设置work group大小
layout(local_size_x_id = 0, local_size_y = 1, local_size_z = 1) in;

// binding 0: 输入矩阵A
layout (binding = 0) readonly buffer A {A_TYPE data_a[];};
#if defined(A_TYPE_PACKED16)
layout (binding = 0) readonly buffer A_PACKED16 {A_TYPE_PACKED16 data_a_packed16[];};
#endif
#if defined(A_TYPE_PACKED32)
layout (binding = 0) readonly buffer A_PACKED32 {A_TYPE_PACKED32 data_a_packed32[];};
#endif

// binding 1: 输入矩阵B
layout (binding = 1) readonly buffer B {B_TYPE data_b[];};
// binding 2: 输出矩阵D
layout (binding = 2) writeonly buffer D {D_TYPE data_d[];};

// ============================================================================
// Push Constants - 运行时参数（在Pipeline创建后设置）
// ============================================================================
layout (push_constant) uniform parameter
{
    uint M;              // 矩阵A的行数
    uint N;              // 矩阵B的列数
    uint K;              // 矩阵A的列数 / 矩阵B的行数（内积维度）
    uint stride_a;       // 矩阵A的stride（行优先存储时的列数）
    uint stride_b;       // 矩阵B的stride
    uint stride_d;       // 输出矩阵D的stride

    uint batch_stride_a; // 批处理时A的batch间距
    uint batch_stride_b; // 批处理时B的batch间距
    uint batch_stride_d; // 批处理时D的batch间距

    uint k_split;        // K维度的分割大小（用于K维度的并行化）
    
    // ggml的张量维度命名约定：
    // ne02: A矩阵的第2维大小（batch维度）
    // ne12: B矩阵的第2维大小（batch维度）
    uint ne02;           
    uint ne12;
    
    // 广播参数：用于支持不同形状矩阵的广播操作
    uint broadcast2;     // 第2维度的广播因子
    uint broadcast3;     // 第3维度的广播因子
} p;

// ============================================================================
// Specialization Constants - 编译时常量（在Pipeline创建时设置）
// ============================================================================
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

// ============================================================================
// 共享内存配置
// ============================================================================
// Stride加1是为了避免bank conflict（内存访问冲突）
#define SHMEM_STRIDE (BK / 2 + 1)

// 共享内存缓冲区：用于缓存A和B的tile数据
shared FLOAT_TYPE_VEC2 buf_a[BM * SHMEM_STRIDE];
shared FLOAT_TYPE_VEC2 buf_b[BN * SHMEM_STRIDE];

#define NUM_WARPS (BLOCK_SIZE / WARP)

#include "mul_mm_funcs.glsl"

// ============================================================================
// 主函数 - GEMM计算核心
// ============================================================================
void main() {
    // ------------------------------------------------------------------------
    // 1. 计算batch索引和广播索引
    // ------------------------------------------------------------------------
    const uint batch_idx = gl_GlobalInvocationID.z;

    // 从batch_idx解码出高维索引（支持3D/4D张量）
    const uint i13 = batch_idx / p.ne12;  // B矩阵的第3维索引
    const uint i12 = batch_idx % p.ne12;  // B矩阵的第2维索引

    // 计算A矩阵对应的索引（考虑广播）
    const uint i03 = i13 / p.broadcast3;
    const uint i02 = i12 / p.broadcast2;

    const uint batch_idx_a = i03 * p.ne02 + i02;

    // ------------------------------------------------------------------------
    // 2. 计算当前工作组负责的tile位置
    // ------------------------------------------------------------------------
    const uint blocks_m = (p.M + BM - 1) / BM;  // M维度需要多少个block
    const uint ir = gl_WorkGroupID.x % blocks_m;  // 当前block的行索引
    const uint ik = gl_WorkGroupID.x / blocks_m;  // K维度的split索引
    const uint ic = gl_WorkGroupID.y;              // 当前block的列索引

    // ------------------------------------------------------------------------
    // 3. Warp和Thread的分块配置
    // ------------------------------------------------------------------------
    const uint WNITER = (WM * WN) / (WARP * TM * TN * WMITER);  // Warp N维度迭代次数
    const uint WSUBM = WM / WMITER;  // Warp子块M维度
    const uint WSUBN = WN / WNITER;  // Warp子块N维度

    // 当前线程所在的warp索引
    const uint warp_i = gl_LocalInvocationID.x / WARP;

    // 当前线程在warp内的索引
    const uint tiw = gl_LocalInvocationID.x % WARP;

    // Thread在warp tile内的行列位置
    const uint tiwr = tiw % (WSUBM / TM);  // 行索引
    const uint tiwc = tiw / (WSUBM / TM);  // 列索引

    // Warp在block内的位置
    const uint warp_r = warp_i % (BM / WM);  // 行索引
    const uint warp_c = warp_i / (BM / WM);  // 列索引

    // ------------------------------------------------------------------------
    // 4. 计算数据加载的索引
    // ------------------------------------------------------------------------
    // 每个线程负责加载的位置
    const uint loadr_a = gl_LocalInvocationID.x % (BK / LOAD_VEC_A / LOAD_VEC_BATCH_A);
    const uint loadc_a = gl_LocalInvocationID.x / (BK / LOAD_VEC_A / LOAD_VEC_BATCH_A);
    const uint loadr_b = gl_LocalInvocationID.x % (BK / LOAD_VEC_B / LOAD_VEC_BATCH_B);
    const uint loadc_b = gl_LocalInvocationID.x / (BK / LOAD_VEC_B / LOAD_VEC_BATCH_B);

    // 加载时的stride（多个线程协作加载）
    const uint loadstride_a = gl_WorkGroupSize.x * LOAD_VEC_A * LOAD_VEC_BATCH_A / BK;
    const uint loadstride_b = gl_WorkGroupSize.x * LOAD_VEC_B * LOAD_VEC_BATCH_B / BK;

    // ------------------------------------------------------------------------
    // 5. 计算K维度的处理范围（用于K维度并行化）
    // ------------------------------------------------------------------------
    const uint start_k = ik * p.k_split;
    const uint end_k = min(p.K, (ik + 1) * p.k_split);

    // 计算全局内存中A和B的起始位置
    uint pos_a = (
        batch_idx_a * p.batch_stride_a +
        ir * BM * p.stride_a + start_k) / LOAD_VEC_A;

    uint pos_b = (batch_idx * p.batch_stride_b + ic * BN * p.stride_b + start_k) / LOAD_VEC_B;

    // ------------------------------------------------------------------------
    // 6. 初始化累加器和缓存
    // ------------------------------------------------------------------------
    ACC_TYPE sums[WMITER * TM * WNITER * TN];  // 累加结果数组
    FLOAT_TYPE_VEC2 cache_a[WMITER * TM];       // A矩阵的寄存器缓存
    FLOAT_TYPE_VEC2 cache_b[TN];                // B矩阵的寄存器缓存

    // 清零累加器
    [[unroll]] for (uint i = 0; i < WMITER*TM*WNITER*TN; i++) {
        sums[i] = ACC_TYPE(0.0f);
    }

    // ------------------------------------------------------------------------
    // 7. 主循环：遍历K维度的所有block
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
        pos_a += BK / LOAD_VEC_A;
        pos_b += BK / LOAD_VEC_B;

        // 阶段2：计算当前block的矩阵乘法
        // 遍历BK维度（每次处理2个元素，因为使用vec2）
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
                            // 使用FMA指令：fma(a, b, c) = a*b + c
                            // 同时处理vec2的x和y分量
                            sums[sums_idx] = fma(ACC_TYPE(cache_a[wsir * TM + cr].x), ACC_TYPE(cache_b[cc].x), 
                                             fma(ACC_TYPE(cache_a[wsir * TM + cr].y), ACC_TYPE(cache_b[cc].y), sums[sums_idx]));
                        }
                    }
                }
            }
        }

        // 同步：确保所有线程完成计算后再加载下一个block
        barrier();
    }

    // ------------------------------------------------------------------------
    // 8. 将结果写回全局内存
    // ------------------------------------------------------------------------
    // 计算当前block在全局结果矩阵中的起始位置
    const uint dr = ir * BM + warp_r * WM;
    const uint dc = ic * BN + warp_c * WN;

    // 计算输出偏移量（考虑batch和k_split）
    const uint offsets = batch_idx * p.batch_stride_d + ik * p.batch_stride_d * gl_NumWorkGroups.z;

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
                        data_d[offsets + (dc_warp + cc) * p.stride_d + dr_warp + cr] = D_TYPE(sums[(wsic * TN + cc) * (WMITER * TM) + wsir * TM + cr]);
                    }
                }
            }
        }
    }
}

// ============================================================================
// 总结：这是一个高度优化的tiled GEMM实现
// ============================================================================
// 优化技术：
// 1. 分块（Tiling）：Block -> Warp -> Thread三级分块，提高数据重用
// 2. 共享内存：缓存频繁访问的数据，减少全局内存访问
// 3. 寄存器缓存：进一步减少共享内存访问
// 4. 循环展开：使用[[unroll]]减少循环开销
// 5. FMA指令：融合乘加操作，提高计算效率
// 6. Bank conflict避免：SHMEM_STRIDE = BK/2 + 1
// 7. 向量化加载：使用vec2批量加载数据
// 8. K维度分割：支持K维度的并行化
// 9. 广播支持：处理不同形状张量的矩阵乘法
// ============================================================================
```

大体上的思路和Simon的[这篇博客](https://siboehm.com/articles/22/CUDA-MMM)中演示的wrap-tile优化很像，都是做了Block-Wrap-Thread的三级分块，如下图所示。

<img src="https://siboehm.com/assets/img/CUDA-MMM/kernel_10_warp_tiling.png">


## coopmat1

TODO

说到矩阵乘，自然离不开Tensor Core。事实上，在NVIDIA所实现的各种加速特性中，Tensor Core几乎是Vulkan唯一支持的。其他特性如TMA、Wrap Specialization、Thread Group Cluster、绕过L1/L2缓存的异步load/store等等，都不受当前的Vulkan支持。Vulkan通过`VK_KHR_cooperative_matrix`（下称coopmat1）和`VK_NV_cooperative_matrix2`（下称coopmat2）这两个扩展来调用Tensor Core。

本小节我们先来阅读基于coopmat1扩展的GEMM写法。同上一小节，我们先对源码做一下宏替换，不然太难读了。
