---
title: PSNR的SIMD优化
categories: FundamentalAlgorithm
date: 2024/11/20 19:49:00
mathjax: true
---

## 前言

PSNR有一套非常简单的算法，以它作为入门SIMD（Single Instruction Multiple Data）汇编的研究对象，个人认为非常合适。性能优化离不开各种工具，而“脚手架”的搭建和性能分析过程往往不会在源代码中有所体现。这篇文章是我入门SIMD汇编的自学记录，同时也希望能给后来者做一些脚手架搭建方法上的参考。如有错漏欢迎直接PR或邮件拍砖。

本文将涉及以下知识点：

- 如何使用docker搭建一个不干扰宿主机的调试环境。
- 利用Compiler Explorer编译代码片段，并阅读理解SIMD汇编。
- 如何使用perf工具的PMU（Performance Monitor Unit）统计功能来分析实际执行过程，进而明确性能提升的来源。
- 如何使用llvm-mca分析汇编执行方案。

最终，通过引入madd和循环展开，我们的实现相较ffmpeg加快了33倍（仅统计用户态耗时）。

## PSNR简介

PSNR，峰值信噪比，全称Peak Signal-to-Noise Ratio，被广泛应用于各种相似度评估场景中。

均方误差（Mean Squared Error）就是先逐项作差，再取平方，最后算各项的平均。PSNR由MSE计算而来，二者的关系如下式表示：

$$PSNR = 10 \cdot \log_{10}\left(\frac{MAX_I^2}{MSE}\right)$$

其中$MAX_I$为对应数据类型的最大值（对`uint8`为255）。需要注意的是，PSNR是一个无量纲量。引入$MAX_I^2$正是为了“抹去”MSE的平方量纲。这样，不论数据类型的位宽是多少，PSNR都能提供同一尺度的相似性度量。

## 性能统计汇总

- 参评对象：ffmpeg及main-v1~v5
- 数据：300帧yuv420p的视频序列
- 硬件：AMD 7950X 定频4.5GHz
- 实验方案：8次缓存预热后重复计算128次全序列平均PSNR，统计用户态耗时的均值和标准差

| 版本     | 平均用户态耗时 (ms) | 标准差 (ms) |
| -------- | ------------------- | ----------- |
| ffmpeg   | 758.203             | 263.039     |
| clang-v1 | 130.703             | 4.363       |
| clang-v2 | 48.281              | 3.773       |
| clang-v3 | 23.828              | 4.861       |
| clang-v4 | 23.125              | 4.635       |
| clang-v5 | 22.422              | 4.284       |
| gcc-v1   | 126.484             | 4.935       |
| gcc-v2   | 39.766              | 2.641       |
| gcc-v3   | 26.094              | 4.879       |
| gcc-v4   | 24.531              | 4.978       |
| gcc-v5   | 22.266              | 4.186       |

## 实现与优化思路

### 仓库结构

所有的代码实现已开源于[lumina37/fast-psnr](https://github.com/lumina37/fast-psnr/)。

项目采用了可以简化构建流程的header-only设计。所有的源文件都位于根目录的src文件夹下。目录树及各目录的含义如下所示。

```
├─src
│  ├─bin      // main-v_.cpp 用于计算PSNR
│  ├─include  // 包含了核心功能相关的所有头文件
│  │  └─psnr
│  │      ├─common       // 通用的编译参数
│  │      ├─concepts     // 编译期约束（c++20 concepts）
│  │      ├─helper       // 一些数学函数
│  │      │  └─constexpr
│  │      ├─impl         // *本期主角
│  │      │  └─mse
│  │      └─io           // 读取yuv
│  └─lib      // 定义了一个用于协助构建的CMake Interface Library
└─tests       // minicase-v_.cpp 用于性能测试
```

下面讲解一下泛型的实现细节，`psnr::PsnrOp<T>`是一个模板类。任何定义了静态成员函数`mse(const Tv* lhs, const Tv* rhs, const size_t len) -> uint64_t`的类型`T`，都可以嵌入`psnr::PsnrOp`中作为MSE的具体实现。通过切换不同的`T`，我们可以方便地实现在不同版本（v1~4）的MSE实现之间的切换。毕竟，说是优化PSNR，其实也就是在优化MSE，因为PSNR计算的热点毫无疑问就在MSE计算。

### Docker搭建调试环境

推荐使用如下Dockerfile搭建调试环境：

```Dockerfile
FROM silkeh/clang:19 AS builder

RUN sed -i 's|deb.debian.org|mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list.d/debian.sources && \
    apt update && \
    apt install -y --no-install-recommends git && \
    apt clean && \
    rm -r /etc/apt/sources.list.d

RUN git clone --depth 1 https://github.com/lumina37/fast-psnr.git && \
    cd fast-psnr && \
    cmake -S . -B build && \
    cmake --build build --config Release --parallel $($(nproc)-1)

WORKDIR /fast-psnr
CMD ["bash"]
```

使用`docker build . -t psnr`构建镜像，`docker run --name psnr -itd psnr bash`拉起容器，`docker exec -it psnr bash`进入容器。

### v2 - 手写AVX2

基线版本（v1）的实现与ffmpeg在[vf_psnr.c](https://ffmpeg.org/doxygen/4.2/vf__psnr_8c_source.html#l00083)中的实现几乎完全一致。

测试数据为2GB长度的数组。minicase-v1的执行情况如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v2':

            720.80 msec task-clock                       #    1.000 CPUs utilized             
                 3      context-switches                 #    4.162 /sec                      
                 1      cpu-migrations                   #    1.387 /sec                      
         1,048,704      page-faults                      #    1.455 M/sec                     
     3,243,461,786      cycles                           #    4.500 GHz                       
       646,369,783      stalled-cycles-frontend          #   19.93% frontend cycles idle      
     6,326,750,058      instructions                     #    1.95  insn per cycle            
                                                  #    0.10  stalled cycles per insn   
     1,026,020,713      branches                         #    1.423 G/sec                     
         2,344,145      branch-misses                    #    0.23% of all branches           

       0.721014382 seconds time elapsed

       0.193999000 seconds user
       0.526997000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v1':

            852.94 msec task-clock                       #    1.000 CPUs utilized             
                 0      context-switches                 #    0.000 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.230 M/sec                     
     3,838,153,603      cycles                           #    4.500 GHz                       
       627,655,061      stalled-cycles-frontend          #   16.35% frontend cycles idle      
     7,903,899,440      instructions                     #    2.06  insn per cycle            
                                                  #    0.08  stalled cycles per insn   
       887,704,043      branches                         #    1.041 G/sec                     
         2,373,633      branch-misses                    #    0.27% of all branches           

       0.853116171 seconds time elapsed

       0.347068000 seconds user
       0.506099000 seconds sys
```

其中`perf stat`的各统计数据的意义可以参考[这篇文章](http://www.lenzhao.com/topic/5981b1212e95f0fd0a981870)。

在v2版本中，我们通过手写AVX2来进一步提高效率。以下是v2版本的`sqrdiff`函数的实现与注释：

```cpp
#include <cstddef>
#include <cstdint>
#include <immintrin.h>
#include <limits>

uint64_t sqrdiff(const uint8_t* lhs, const uint8_t* rhs, size_t len) noexcept
{
    const uint8_t* lhs_cursor = lhs;
    const uint8_t* rhs_cursor = rhs;
    // 以__m128i为加载单元，`len`中共包含`m128_cnt`组__m128i
    const size_t m128_cnt = len / sizeof(__m128i);
    // step是加载步长，也就是16字节
    constexpr size_t step = sizeof(__m128i) / sizeof(uint8_t);
    // 为避免uint32溢出，每隔`group_len`组就需要把累加值dump一次到uint64
    // 最后的`*2`是因为每一组uint16都对应了两组uint32
    constexpr size_t u8max = std::numeric_limits<uint8_t>::max();
    constexpr size_t u32max = std::numeric_limits<uint32_t>::max();
    constexpr size_t group_len = u32max / (u8max * u8max * 2);

    uint64_t sqr_diff_acc = 0;
    __m256i u32sqr_diff_acc = _mm256_setzero_si256();

    // 累加一组SIMD向量
    auto dump_unit = [&](const __m256i u8l, const __m256i u8r) mutable {
        const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
        const __m256i u16sqr_diff = _mm256_mullo_epi16(i16diff, i16diff);
        const __m256i u32sqr_diff_lo = _mm256_unpacklo_epi16(u16sqr_diff, _mm256_setzero_si256());
        u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff_lo);
        const __m256i u32sqr_diff_hi = _mm256_unpackhi_epi16(u16sqr_diff, _mm256_setzero_si256());
        u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff_hi);
    };

    // 将`u32sqr_diff_acc`转移到`sqr_diff_acc`
    auto dump_u32sqr_diff_acc = [&]() mutable {
        const __m256i u64sqr_diff_acc_p0 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc, 0));
        const __m256i u64sqr_diff_acc_p1 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc, 1));
        const __m256i u64sqr_diff_acc = _mm256_add_epi64(u64sqr_diff_acc_p0, u64sqr_diff_acc_p1);
        const auto* tmp = (uint64_t*)&u64sqr_diff_acc;
        sqr_diff_acc += (tmp[0] + tmp[1] + tmp[2] + tmp[3]);
    };

    size_t count = 0;
    for (size_t i = 0; i < m128_cnt; i++) {
        const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
        const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
        dump_unit(u8l, u8r);
        lhs_cursor += step;
        rhs_cursor += step;
        count++;

        if (count == group_len) [[unlikely]] {
            dump_u32sqr_diff_acc();
            u32sqr_diff_acc = _mm256_setzero_si256();
            count = 0;
        }
    }

    dump_u32sqr_diff_acc();

    return sqr_diff_acc;
}
```

在[Compiler Explorer](https://godbolt.org/)中使用clang 19.1.0以及参数`-std=c++20 -O3 -mavx2`编译得到的汇编如下：

```nasm
sqrdiff(unsigned char const*, unsigned char const*, unsigned long):
        cmp     rdx, 16  ; rdx就是len参数
        jae     .LBB0_2  ; 如果rdx大于等于16，就跳到负责向量求和的.LBB0_2去
        vpxor   xmm1, xmm1, xmm1
        xor     ecx, ecx
.LBB0_6:
        ; LBB0_6对应函数返回前的收尾工作
        ; clang把最后一次对dump_u32sqr_diff_acc的调用提前到了这个位置
        ; 个人感觉放到最底下也许顺序化执行对缓存更友好？
        vpmovzxdq       ymm0, xmm1      ; ymm1就是u32sqr_diff_acc
                                        ; 这条指令把ymm1的低128位从u32扩展成u64然后放入ymm0
        vextracti128    xmm1, ymm1, 1   ; 把ymm1的高128位挪到前半
        vpmovzxdq       ymm1, xmm1      ; ymm1的高128位做u32扩u64，并放进ymm1
        vpaddq  ymm0, ymm0, ymm1
                                        ; 从这往后的一段是编译器生成的u64x4求和
                                        ; 对应源码`auto* tmp`开始的两行
        vextracti128    xmm1, ymm0, 1   ; 把ymm0的高128位送进xmm1
        vpaddq  xmm0, xmm0, xmm1        ; 相当于把ymm0的前后两半折叠求和成xmm0
                                        ; xmm0 = [0+2, 1+3] （低位到高位）
        vpshufd xmm1, xmm0, 238         ; uint8立即数238就是0b11_10_11_10
                                        ; 表示目标操作数的各32位分别来自源操作数的哪个部分
                                        ; 比如最低位（最右侧）的10就表示xmm1[0]来自xmm0[2]
                                        ; xmm1 = [1+3, 1+3]
        vpaddq  xmm0, xmm0, xmm1        ; xmm0 = [0+1+2+3, ...]
        vmovq   rax, xmm0               ; rax = 0+1+2+3
        add     rax, rcx                ; 把sqr_diff_acc中保存的结果加到返回值rax里
        vzeroupper
        ret
.LBB0_2:
        ; LBB0_2对应向量化处理前的准备工作
        shr     rdx, 4                  ; 将len除以__m128i的长度16，算出m128_cnt
        vpxor   xmm0, xmm0, xmm0        ; 归零归零归零
        xor     eax, eax
        xor     r8d, r8d
        xor     ecx, ecx
        vpxor   xmm1, xmm1, xmm1
.LBB0_3:
        vpmovzxbw       ymm2, xmmword ptr [rdi + rax]
        vpmovzxbw       ymm3, xmmword ptr [rsi + rax]
        vpsubw  ymm2, ymm2, ymm3            ; 这里开始就是dump_unit函数的内联
        vpmullw ymm2, ymm2, ymm2
        vpunpcklwd      ymm3, ymm2, ymm0
        vpaddd  ymm1, ymm1, ymm3
        vpunpckhwd      ymm2, ymm2, ymm0
        vpaddd  ymm1, ymm1, ymm2            ; dump_unit结束
        inc     r8
        cmp     r8, 33025                   ; 33025就是group_len
        je      .LBB0_4                     ; 如果group计数达到33025，跳到dump_u32sqr_diff_acc
        add     rax, 16                     ; 地址偏移增加16字节
        dec     rdx                         ; for循环计数器减1
        jne     .LBB0_3                     ; 不满足退出条件就跳回开头
        jmp     .LBB0_6                     ; 否则跳到LBB0_6做收尾工作
.LBB0_4:
        ; dump_u32sqr_diff_acc在循环体中内联所生成的汇编
        vpmovzxdq       ymm2, xmm1
        vextracti128    xmm1, ymm1, 1
        vpmovzxdq       ymm1, xmm1
        vpaddq  ymm1, ymm2, ymm1
        vextracti128    xmm2, ymm1, 1
        vpaddq  xmm1, xmm1, xmm2
        vpshufd xmm2, xmm1, 238
        vpaddq  xmm1, xmm1, xmm2
        vmovq   r8, xmm1                ; 到这里为止都和LBB0_6一模一样
        add     rcx, r8                 ; 累加到计数器rcx
                                        ; 在LBB0_6和LBB0_4中，clang分别用了rax和rcx来表示sqr_diff_acc
                                        ; 有人可能会问为什么不统一用rax表示？
                                        ; 我猜测这可能是一个失误？
                                        ; 但其实现代处理器的寄存器重命名技术已经相当成熟，这里应该无须担心性能问题
        vpxor   xmm1, xmm1, xmm1        ; 累加器归零
        xor     r8d, r8d
        add     rax, 16
        dec     rdx
        jne     .LBB0_3                 ; 跳回for循环开头
        jmp     .LBB0_6                 ; 或者准备返回
```

v2版本的执行情况如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v2':

            715.63 msec task-clock                       #    1.000 CPUs utilized             
                 1      context-switches                 #    1.397 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.465 M/sec                     
     3,220,258,049      cycles                           #    4.500 GHz                       
       645,569,842      stalled-cycles-frontend          #   20.05% frontend cycles idle      
     6,297,236,854      instructions                     #    1.96  insn per cycle            
                                                  #    0.10  stalled cycles per insn   
     1,022,511,180      branches                         #    1.429 G/sec                     
         2,332,402      branch-misses                    #    0.23% of all branches           

       0.715805466 seconds time elapsed

       0.203239000 seconds user
       0.512604000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v2':

            716.08 msec task-clock                       #    1.000 CPUs utilized             
                 2      context-switches                 #    2.793 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,703      page-faults                      #    1.464 M/sec                     
     3,222,263,484      cycles                           #    4.500 GHz                       
       630,123,974      stalled-cycles-frontend          #   19.56% frontend cycles idle      
     6,150,346,906      instructions                     #    1.91  insn per cycle            
                                                  #    0.10  stalled cycles per insn   
     1,020,598,509      branches                         #    1.425 G/sec                     
         2,340,725      branch-misses                    #    0.23% of all branches           

       0.716288318 seconds time elapsed

       0.213090000 seconds user
       0.503213000 seconds sys
```

可以看到v2相较v1大幅减少了指令数和分支数。

### v3 - 改用madd

在SIMD指令集中，有一类特殊的指令可以实现先乘后加，即madd系列指令。下面使用madd来优化`dump_unit`函数。

修改前的`dump_unit`：

```cpp
auto dump_unit = [&](const __m256i u8l, const __m256i u8r) mutable {
    const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
    const __m256i u16sqr_diff = _mm256_mullo_epi16(i16diff, i16diff);
    const __m256i u32sqr_diff_lo = _mm256_unpacklo_epi16(u16sqr_diff, _mm256_setzero_si256());
    u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff_lo);
    const __m256i u32sqr_diff_hi = _mm256_unpackhi_epi16(u16sqr_diff, _mm256_setzero_si256());
    u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff_hi);
};
```

对应的汇编：

```nasm
vpsubw      ymm2, ymm2, ymm3
vpmullw     ymm2, ymm2, ymm2
vpunpcklwd  ymm3, ymm2, ymm0
vpaddd      ymm1, ymm1, ymm3
vpunpckhwd  ymm2, ymm2, ymm0
vpaddd      ymm1, ymm1, ymm2
```

修改后的`dump_unit`：

```cpp
auto dump_unit = [&](const __m256i u8l, const __m256i u8r) mutable {
    const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
    const __m256i u32sqr_diff = _mm256_madd_epi16(i16diff, i16diff);
    u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff);
};
```

对应的汇编：

```nasm
vpsubw    ymm1, ymm1, ymm2
vpmaddwd  ymm1, ymm1, ymm1
vpaddd    ymm0, ymm1, ymm0
```

补充一下，简单的指令周期数（Latency）和吞吐量倒数（RThroughput）可以在[Intel官网](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)查询，更详细的参数可以在[uops.info](https://uops.info)查询。其中RThroughput为x的意思是每隔x个时钟周期可以发射一条指令，例如RThroughput=0.5意味着每时钟周期可以发射两条指令。

vpmaddwd和vpmullw的吞吐量和消耗的周期数几乎一致。修改后我们在热点循环中节省了两条unpack指令和一条add指令。

v3版本的执行耗时如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v3':

            705.02 msec task-clock                       #    1.000 CPUs utilized             
                 3      context-switches                 #    4.255 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,705      page-faults                      #    1.487 M/sec                     
     3,172,446,887      cycles                           #    4.500 GHz                       
       640,578,837      stalled-cycles-frontend          #   20.19% frontend cycles idle      
     5,884,826,983      instructions                     #    1.85  insn per cycle            
                                                  #    0.11  stalled cycles per insn   
     1,020,946,737      branches                         #    1.448 G/sec                     
         2,339,989      branch-misses                    #    0.23% of all branches           

       0.705276369 seconds time elapsed

       0.175070000 seconds user
       0.530212000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v3':

            696.70 msec task-clock                       #    1.000 CPUs utilized             
                 0      context-switches                 #    0.000 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.505 M/sec                     
     3,135,102,182      cycles                           #    4.500 GHz                       
       633,710,248      stalled-cycles-frontend          #   20.21% frontend cycles idle      
     5,755,898,657      instructions                     #    1.84  insn per cycle            
                                                  #    0.11  stalled cycles per insn   
     1,021,842,610      branches                         #    1.467 G/sec                     
         2,349,301      branch-misses                    #    0.23% of all branches           

       0.696893328 seconds time elapsed

       0.189982000 seconds user
       0.506953000 seconds sys
```

### v4 - 减少分支

在v3中，每个循环都要判断两次分支，一次退出条件判断，一次`group_len`有关的uint32溢出判断。我们可以预先计算`len`中有多少个`group_len`，以减少后一种分支。

v4版本的`sqrdiff`实现如下：

```cpp
#include <cstddef>
#include <cstdint>
#include <immintrin.h>
#include <limits>

uint64_t sqrdiff(const uint8_t* lhs, const uint8_t* rhs, size_t len) noexcept
{
    const uint8_t* lhs_cursor = lhs;
    const uint8_t* rhs_cursor = rhs;
    const size_t m128_cnt = len / sizeof(__m128i);
    constexpr size_t step = sizeof(__m128i) / sizeof(uint8_t);
    constexpr size_t u8max = std::numeric_limits<uint8_t>::max();
    constexpr size_t u32max = std::numeric_limits<uint32_t>::max();
    constexpr size_t group_len = u32max / (u8max * u8max * 2);
    // 总共group_cnt大组
    const size_t group_cnt = m128_cnt / group_len;
    // 还剩下resi_len个小组
    const size_t resi_len = m128_cnt - group_cnt * group_len;

    uint64_t sqr_diff_acc = 0;
    __m256i u32sqr_diff_acc = _mm256_setzero_si256();

    auto dump_unit = [&](const __m256i u8l, const __m256i u8r) mutable {
        const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
        const __m256i u32sqr_diff = _mm256_madd_epi16(i16diff, i16diff);
        u32sqr_diff_acc = _mm256_add_epi32(u32sqr_diff_acc, u32sqr_diff);
    };

    auto dump_u32sqr_diff_acc = [&]() mutable {
        const __m256i u64sqr_diff_acc_p0 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc, 0));
        const __m256i u64sqr_diff_acc_p1 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc, 1));
        const __m256i u64sqr_diff_acc = _mm256_add_epi64(u64sqr_diff_acc_p0, u64sqr_diff_acc_p1);
        const auto* tmp = (uint64_t*)&u64sqr_diff_acc;
        sqr_diff_acc += (tmp[0] + tmp[1] + tmp[2] + tmp[3]);
    };

    for (size_t igroup = 0; igroup < group_cnt; igroup++) {
        for (size_t i = 0; i < group_len; i++) {
            const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
            const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
            dump_unit(u8l, u8r);
            lhs_cursor += step;
            rhs_cursor += step;
        }

        dump_u32sqr_diff_acc();
        u32sqr_diff_acc = _mm256_setzero_si256();
    }

    for (size_t i = 0; i < resi_len; i++) {
        const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
        const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
        dump_unit(u8l, u8r);
        lhs_cursor += step;
        rhs_cursor += step;
    }

    dump_u32sqr_diff_acc();

    return sqr_diff_acc;
}
```

v4版本的汇编如下：

```nasm
sqrdiff(unsigned char const*, unsigned char const*, unsigned long):
        push    r15                         ; 保护寄存器
        push    r14
        push    r12
        push    rbx
        mov     rcx, rdx                    ; rcx是len
        mov     r8, rdx
        shr     r8, 4                       ; r8是len除以16后得到的m128_cnt
        movabs  rdx, 1143949488658808833    ; 这个魔法数用来计算整数除法，后面解释
        mov     rax, rcx                    ; 现在rax中是len的原始值
        mul     rdx
        shr     rdx, 15                     ; 现在rdx是group_cnt
        imul    r9, rdx, -33025             ; group_cnt乘-33025，也就是-group_cnt*group_len
        lea     rax, [r9 + r8]              ; 用lea计算加法得到resi_len
        cmp     rcx, 528400                 ; 如果len大于33025*16，即至少有一个大组，跳到.LBB0_2
        jae     .LBB0_2
        xor     ecx, ecx
        test    rax, rax                    ; 如果resi_len也为0那就直接返回
        je      .LBB0_9
.LBB0_10:
        ; 这部分判断要不要处理residual
        mov     edx, 1
        sub     rdx, r8                     ; %rdx = 1 - m128_cnt
        cmp     r9, rdx                     ; 如果m128_cnt - group_cnt * group_len != 1
        jne     .LBB0_12                    ; 说明还能做循环展开，就跳去.LBB0_12
        vpxor   xmm0, xmm0, xmm0
        jmp     .LBB0_14                    ; 否则，就跳到.LBB0_14做单个处理
.LBB0_2:
        imul    r11, rdx, 528400            ; r11是group_cnt的长度
        lea     r10, [rdi + r11]            ; r10是group_cnt后面不成组的residual部分的起始地址
        xor     ebx, ebx
        mov     r14, rsi                    ; 把rsi暂存在r14
        xor     ecx, ecx
        jmp     .LBB0_3
.LBB0_6:
        add     r14, 528400                 ; 步进一个大组
        vpmovzxdq       ymm1, xmm0          ; 把累加结果dump到uint64
        vextracti128    xmm0, ymm0, 1
        vpmovzxdq       ymm0, xmm0
        vpaddq  ymm0, ymm1, ymm0
        vextracti128    xmm1, ymm0, 1
        vpaddq  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 238
        vpaddq  xmm0, xmm0, xmm1
        vmovq   rdi, xmm0
        add     rcx, rdi                    ; dump完毕
        inc     rbx
        mov     rdi, r15
        cmp     rbx, rdx                    ; 这里rdx是group_cnt
        je      .LBB0_7
.LBB0_3:
        ; 向量化处理前的一些准备工作
        lea     r15, [rdi + 528400]
        vpxor   xmm0, xmm0, xmm0
        xor     r12d, r12d
.LBB0_4:
        ; 由于循环体指令数大幅减少
        ; 这里clang做了一个2x循环展开
        vpmovzxbw       ymm1, xmmword ptr [rdi + r12]
        vpmovzxbw       ymm2, xmmword ptr [r14 + r12]
        vpsubw  ymm1, ymm1, ymm2
        vpmaddwd        ymm1, ymm1, ymm1
        vpaddd  ymm0, ymm1, ymm0
        cmp     r12, 528384     ; 528384/32=33024/2
        je      .LBB0_6         ; 因为跳转指令是放在中间的，而不是add 32之后立马跳，所以总共执行了33025次
                                ; 把跳转放中间是循环展开中一个常见的技巧
        vpmovzxbw       ymm1, xmmword ptr [rdi + r12 + 16]
        vpmovzxbw       ymm2, xmmword ptr [r14 + r12 + 16]
        vpsubw  ymm1, ymm1, ymm2
        vpmaddwd        ymm1, ymm1, ymm1
        vpaddd  ymm0, ymm1, ymm0
        add     r12, 32
        jmp     .LBB0_4
.LBB0_7:
        add     rsi, r11        ; rsi后移到residual部分的起始地址
        mov     rdi, r10        ; 复习：r10是group_cnt后面不成组的residual部分的起始地址
        test    rax, rax        ; 搞不懂这里为什么又要判断一次resi_len的大小，感觉可以去掉？
        jne     .LBB0_10
.LBB0_9:
        vpxor   xmm0, xmm0, xmm0
        jmp     .LBB0_16
.LBB0_12:
        ; 2x循环展开处理residual前的准备工作
        mov     rdx, rax
        and     rdx, -2             ; -2的补码是0b11...110，这里的操作相当于向下对齐到最近的2的倍数
        vpxor   xmm0, xmm0, xmm0
.LBB0_13:
        ; 2x循环展开处理residual
        vpmovzxbw       ymm1, xmmword ptr [rdi]
        vpmovzxbw       ymm2, xmmword ptr [rsi]
        vpsubw  ymm1, ymm1, ymm2
        vpmaddwd        ymm1, ymm1, ymm1
        vpaddd  ymm0, ymm1, ymm0
        vpmovzxbw       ymm1, xmmword ptr [rdi + 16]
        vpmovzxbw       ymm2, xmmword ptr [rsi + 16]
        vpsubw  ymm1, ymm1, ymm2
        vpmaddwd        ymm1, ymm1, ymm1
        vpaddd  ymm0, ymm1, ymm0
        add     rdi, 32
        add     rsi, 32
        add     rdx, -2
        jne     .LBB0_13
.LBB0_14:
        ; 尝试处理结尾的单个__m128i
        test    al, 1       ; 如果al&1==0，即可以被2整除的话
        je      .LBB0_16    ; 就直接跳到结尾
        vpmovzxbw       ymm1, xmmword ptr [rdi]
        vpmovzxbw       ymm2, xmmword ptr [rsi]
        vpsubw  ymm1, ymm1, ymm2
        vpmaddwd        ymm1, ymm1, ymm1
        vpaddd  ymm0, ymm1, ymm0
.LBB0_16:
        ; 收尾工作，准备函数返回
        vpmovzxdq       ymm1, xmm0
        vextracti128    xmm0, ymm0, 1
        vpmovzxdq       ymm0, xmm0
        vpaddq  ymm0, ymm1, ymm0
        vextracti128    xmm1, ymm0, 1
        vpaddq  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 238
        vpaddq  xmm0, xmm0, xmm1
        vmovq   rax, xmm0
        add     rax, rcx
        pop     rbx
        pop     r12
        pop     r14
        pop     r15
        vzeroupper
        ret
```

还记得那个巨大的魔法数1143949488658808833吗，它的作用是用乘法和位运算来实现整数除。

记$n=33025*16$，如果我们需要计算$a/n, a \le 2^{63}-1$。只需要找到这样一个$s$来放大分子分母，以使得整数乘法搭配按位右移的精度可以达到模拟整数除法的要求。用公式表示便是：

$$\forall a \le 2^{63}-1, \lvert \frac{a}{n} - \frac{a \cdot \mathrm{round}(2^s / n)}{2^s} \rvert \lt \frac{1}{2}$$

这里clang和gcc都选择了$s=79$，也就是$2^{79}/1143949488658808833 \approx 528400$。需要注意的是，mul指令会将高64位的结果放在rdx寄存器，低64位放在rax寄存器，因此`shr rdx, 15`其实等效于将乘法的结果右移了15+64位。

v4版本的执行时间如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v4':

            700.74 msec task-clock                       #    1.000 CPUs utilized             
                 0      context-switches                 #    0.000 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.497 M/sec                     
     3,153,257,995      cycles                           #    4.500 GHz                       
       641,459,746      stalled-cycles-frontend          #   20.34% frontend cycles idle      
     5,491,092,163      instructions                     #    1.74  insn per cycle            
                                                  #    0.12  stalled cycles per insn   
       888,158,656      branches                         #    1.267 G/sec                     
         2,385,160      branch-misses                    #    0.27% of all branches           

       0.700922038 seconds time elapsed

       0.182989000 seconds user
       0.517969000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v4':

            689.72 msec task-clock                       #    1.000 CPUs utilized             
                 1      context-switches                 #    1.450 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.520 M/sec                     
     3,103,660,300      cycles                           #    4.500 GHz                       
       635,350,340      stalled-cycles-frontend          #   20.47% frontend cycles idle      
     5,212,856,511      instructions                     #    1.68  insn per cycle            
                                                  #    0.12  stalled cycles per insn   
       886,595,796      branches                         #    1.285 G/sec                     
         2,379,553      branch-misses                    #    0.27% of all branches           

       0.689924576 seconds time elapsed

       0.195984000 seconds user
       0.493961000 seconds sys
```

v4将分支数减少了约10%，性能有微小提升，不过基本可以忽略不计了。

### v5 - 循环展开

在v4中我们注意到clang自动生成了两处2x循环展开。因此我们可以考虑手动循环展开以进一步提高性能。

```cpp
#include <array>
#include <cstddef>
#include <cstdint>
#include <immintrin.h>
#include <limits>

uint64_t sqrdiff(const uint8_t* lhs, const uint8_t* rhs, size_t len) noexcept
{
    const uint8_t* lhs_cursor = lhs;
    const uint8_t* rhs_cursor = rhs;
    const size_t m128_cnt = len / sizeof(__m128i);
    constexpr size_t step = sizeof(__m128i) / sizeof(uint8_t);
    constexpr size_t u8max = std::numeric_limits<uint8_t>::max();
    constexpr size_t u32max = std::numeric_limits<uint32_t>::max();
    // 做8x循环展开
    constexpr size_t unroll = 8;
    constexpr size_t group_len = (u32max / (u8max * u8max * 2)) * unroll;
    const size_t group_cnt = m128_cnt / group_len;
    // 还有`nogroup_len`个`__m128i`不成组
    const size_t nogroup_len = m128_cnt - group_cnt * group_len;
    // 将它们打包成`nogroup_unroll_cnt`个长`unroll`的小组以执行循环展开
    const size_t nogroup_unroll_cnt = nogroup_len / unroll;
    // 剩下`resi_cnt`个`__m128i`无法做向量展开，单独处理
    const size_t resi_cnt = nogroup_len - nogroup_unroll_cnt * unroll;

    uint64_t sqr_diff_acc = 0;
    // 使用独立的累加器
    std::array<__m256i, unroll> u32sqr_diff_acc{};

    auto dump_unit = [&](const __m256i u8l, const __m256i u8r, const size_t i) mutable {
        const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
        const __m256i u32sqr_diff = _mm256_madd_epi16(i16diff, i16diff);
        u32sqr_diff_acc[i] = _mm256_add_epi32(u32sqr_diff_acc[i], u32sqr_diff);
    };

    auto dump_u32sqr_diff_acc = [&]() mutable {
        for (size_t i = 0; i < unroll; i++) {
            __m256i u64sqr_diff_acc_p0 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc[i], 0));
            __m256i u64sqr_diff_acc_p1 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc[i], 1));
            __m256i u64sqr_diff_acc = _mm256_add_epi64(u64sqr_diff_acc_p0, u64sqr_diff_acc_p1);
            auto* tmp = (uint64_t*)&u64sqr_diff_acc;
            sqr_diff_acc += (tmp[0] + tmp[1] + tmp[2] + tmp[3]);
        }
    };

    for (size_t igroup = 0; igroup < group_cnt; igroup++) {
        for (size_t i = 0; i < (group_len / unroll); i++) {
            for (size_t j = 0; j < unroll; j++) {
                const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
                const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
                dump_unit(u8l, u8r, j);
                lhs_cursor += step;
                rhs_cursor += step;
            }
        }

        dump_u32sqr_diff_acc();
        for (size_t j = 0; j < unroll; j++) {
            u32sqr_diff_acc[j] = _mm256_setzero_si256();
        }
    }

    for (size_t i = 0; i < nogroup_unroll_cnt; i++) {
        for (size_t j = 0; j < unroll; j++) {
            const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
            const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
            dump_unit(u8l, u8r, j);
            lhs_cursor += step;
            rhs_cursor += step;
        }
    }

    for (size_t i = 0; i < resi_cnt; i++) {
        const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
        const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
        dump_unit(u8l, u8r, i);
        lhs_cursor += step;
        rhs_cursor += step;
    }

    dump_u32sqr_diff_acc();

    return sqr_diff_acc;
}
```

v5的性能表现如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v5':

            670.61 msec task-clock                       #    1.000 CPUs utilized             
                 4      context-switches                 #    5.965 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.564 M/sec                     
     3,017,593,452      cycles                           #    4.500 GHz                       
       633,550,760      stalled-cycles-frontend          #   21.00% frontend cycles idle      
     5,175,065,818      instructions                     #    1.71  insn per cycle            
                                                  #    0.12  stalled cycles per insn   
       771,170,437      branches                         #    1.150 G/sec                     
         2,318,819      branch-misses                    #    0.30% of all branches           

       0.670819307 seconds time elapsed

       0.166203000 seconds user
       0.504618000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v5':

            678.38 msec task-clock                       #    1.000 CPUs utilized             
                 1      context-switches                 #    1.474 /sec                      
                 0      cpu-migrations                   #    0.000 /sec                      
         1,048,704      page-faults                      #    1.546 M/sec                     
     3,052,595,529      cycles                           #    4.500 GHz                       
       645,160,486      stalled-cycles-frontend          #   21.13% frontend cycles idle      
     4,997,264,098      instructions                     #    1.64  insn per cycle            
                                                  #    0.13  stalled cycles per insn   
       769,661,725      branches                         #    1.135 G/sec                     
         2,319,870      branch-misses                    #    0.30% of all branches           

       0.678572887 seconds time elapsed

       0.162143000 seconds user
       0.516455000 seconds sys
```

虽然分支数进一步减少，但错误分支计数并没有下降太多。如果是生产应用建议优化到v3就差不多了，v4和v5的阅读和维护难度相比收益而言不太成正比。

### v1 - 自动向量化

完成了以上的一系列优化后，我们再回头看看最开始的naive版本。

```
#include <cstddef>
#include <cstdint>

uint64_t sqrdiff(const uint8_t* lhs, const uint8_t* rhs, size_t len) noexcept
{
    uint64_t acc = 0;
    for (size_t i = 0; i < len; i++) {
        const int16_t diff = lhs[i] - rhs[i];
        acc += diff * diff;
    }
    return acc;
}
```

添加`-mavx2`编译选项后，gcc和clang都会生成自动向量化的汇编。

其中clang19生成的汇编如下：

```nasm
sqrdiff(unsigned char const*, unsigned char const*, unsigned long):
        test    rdx, rdx    ; 长度为0就直接退出
        je      .LBB0_1
        cmp     rdx, 15     ; 长度大于等于16就跳到.LBB0_6处理长数组
        ja      .LBB0_6
        xor     eax, eax
        mov     rcx, rdi
        mov     r8, rsi
        xor     r9d, r9d
        jmp     .LBB0_4     ; .LBB0_4处理短数组
.LBB0_1:
        xor     eax, eax
        ret
.LBB0_6:
        ; 这一段处理长数组
        mov     r9, rdx
        and     r9, -16             ; 读dword需要将末地址向下对齐到16字节
        lea     rcx, [rdi + r9]     ; rcx和r8是向量处理的结束地址
        lea     r8, [rsi + r9]
        vpxor   xmm0, xmm0, xmm0
        xor     eax, eax
        vpxor   xmm1, xmm1, xmm1
        vpxor   xmm2, xmm2, xmm2
        vpxor   xmm3, xmm3, xmm3
        vpxor   xmm4, xmm4, xmm4
.LBB0_7:
        vpmovzxbd       xmm5, dword ptr [rdi + rax]         ; 加载数据并作差
        vpmovzxbd       xmm6, dword ptr [rdi + rax + 4]
        vpmovzxbd       xmm7, dword ptr [rdi + rax + 8]
        vpmovzxbd       xmm8, dword ptr [rdi + rax + 12]
        vpmovzxbd       xmm9, dword ptr [rsi + rax]
        vpsubd  xmm5, xmm5, xmm9
        vpmovzxbd       xmm9, dword ptr [rsi + rax + 4]
        vpsubd  xmm6, xmm6, xmm9
        vpmovzxbd       xmm9, dword ptr [rsi + rax + 8]
        vpmovzxbd       xmm10, dword ptr [rsi + rax + 12]
        vpsubd  xmm7, xmm7, xmm9
        vpsubd  xmm8, xmm8, xmm10                           ; 加载+减法部分结束
        vpmulld xmm5, xmm5, xmm5                            ; 自乘求平方
        vpmulld xmm6, xmm6, xmm6
        vpmulld xmm7, xmm7, xmm7
        vpmulld xmm8, xmm8, xmm8                            ; 求平方部分结束
        vpblendw        xmm5, xmm5, xmm1, 170               ; 170=0b10101010
        vpblendw        xmm6, xmm6, xmm1, 170               ; 1表示目标元素来自xmm1，反之来自xmm5~8
        vpblendw        xmm7, xmm7, xmm1, 170               ; 这四句汇编的作用就是把xmm5~8的各uint32的高16位清零
        vpblendw        xmm8, xmm8, xmm1, 170
        vpmovzxdq       ymm5, xmm5                          ; uint32x4扩展为uint64x4
        vpaddq  ymm0, ymm0, ymm5                            ; 然后累加到ymm0/2/3/4上
        vpmovzxdq       ymm5, xmm6
        vpaddq  ymm2, ymm2, ymm5
        vpmovzxdq       ymm5, xmm7
        vpaddq  ymm3, ymm3, ymm5
        vpmovzxdq       ymm5, xmm8
        vpaddq  ymm4, ymm4, ymm5
        add     rax, 16
        cmp     r9, rax                                     ; 循环是否结束？
        jne     .LBB0_7
        vpaddq  ymm0, ymm2, ymm0                            ; ymm2~4加到ymm0
        vpaddq  ymm0, ymm3, ymm0
        vpaddq  ymm0, ymm4, ymm0
        vextracti128    xmm1, ymm0, 1                       ; 熟悉的uint64x4求和逻辑
        vpaddq  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 238
        vpaddq  xmm0, xmm0, xmm1
        vmovq   rax, xmm0
        cmp     r9, rdx                                     ; 是否还需要处理结尾？
        je      .LBB0_9                                     ; 不需要就直接退出
.LBB0_4:
        sub     rdx, r9
        xor     esi, esi
.LBB0_5:
        ; 此处是结尾段的标量处理，不再分析
        movzx   edi, byte ptr [rcx + rsi]
        movzx   r9d, byte ptr [r8 + rsi]
        sub     edi, r9d
        imul    edi, edi
        movzx   edi, di
        add     rax, rdi
        inc     rsi
        cmp     rdx, rsi
        jne     .LBB0_5
.LBB0_9:
        vzeroupper
        ret
```

相较我们在v3中的手写实现，由于需要直接累加到uint64，缺少uint32作为中继，编译器在累加部分额外使用了大量指令用来将uint32x4扩展为uint64x4。

该版本与ffmpeg实现的主要区别在于，v1中的`acc`的数据类型是uint64，而ffmpeg中的`acc`的数据类型是uint32。将`acc`的数据类型改成uint32后，编译出的求差值平方和的部分如下所示：

```nasm
vpmovzxbw       xmm4, qword ptr [rdi + rax]         ; 这里做了个4x循环展开
vpmovzxbw       xmm5, qword ptr [rdi + rax + 8]
vpmovzxbw       xmm6, qword ptr [rdi + rax + 16]
vpmovzxbw       xmm7, qword ptr [rdi + rax + 24]
vpmovzxbw       xmm8, qword ptr [rsi + rax]
vpsubw  xmm4, xmm4, xmm8
vpmovzxbw       xmm8, qword ptr [rsi + rax + 8]
vpsubw  xmm5, xmm5, xmm8
vpmovzxbw       xmm8, qword ptr [rsi + rax + 16]
vpmovzxbw       xmm9, qword ptr [rsi + rax + 24]
vpsubw  xmm6, xmm6, xmm8
vpsubw  xmm7, xmm7, xmm9                            ; 减法部分到此结束
vpmaddwd        xmm4, xmm4, xmm4                    ; 使用madd指令做乘加
vpaddd  ymm0, ymm4, ymm0
vpmaddwd        xmm4, xmm5, xmm5
vpaddd  ymm1, ymm4, ymm1
vpmaddwd        xmm4, xmm6, xmm6
vpaddd  ymm2, ymm4, ymm2
vpmaddwd        xmm4, xmm7, xmm7
vpaddd  ymm3, ymm4, ymm3
add     rax, 32                                     ; 步进32字节
cmp     rcx, rax                                    ; 判断是否结束循环
jne     .LBB0_5
```

可以看到，当`acc`为uint32时，clang非常聪明地用上了vpmaddwd来优化乘加运算。然而，由于clang不知道输入数据的规模通常远大于4个xmmword的长度（64字节），因此并没有采用加载xmmword的激进方法，仅使用了加载qword的保守方法。此外，我们的v5版本的优化方案事实上参考了这里clang生成的汇编，包括4x循环展开，以及使用四个独立的累加器这两项优化。其中，独立的累加器可以避免过早地合并结果（术语叫规约，Reduce），减少数据依赖导致的流水线停顿。

## 更多思考

### 加载位宽越大越好？

是否加载指令的位宽越大，性能就越好？采用`_mm256_load_si256`+`_mm256_extracti128_si256`读两个`__m128i`会比连续使用两次`_mm_load_si128`更快吗？对第二个问题，答案是否定的，因此第一个问题的答案是：不总是，因此不要盲目增大加载位宽，哪怕地址已经做了适当的对齐。下面使用llvm-mca来简单分析一下第二个问题。

对于使用单次256位加载的情况，汇编如下：

```nasm
load_avx:
    vmovdqu ymm0, [rdi]
    vextracti128 xmm1, ymm0, 1
```

使用llvm-mca分析时间线，使用的shell如下：

```bash
llvm-mca avx.s -x86-asm-syntax=intel -iterations=1 --timeline
```

分析结果摘录如下：

```log
Instruction Info:
[1]: #uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 1      8     0.50    *                   vmovdqu	ymm0, ymmword ptr [rdi]
 1      4     1.00                        vextracti128	xmm1, ymm0, 1


Timeline view:
                    01234
Index     0123456789

[0,0]     DeeeeeeeeER   .   vmovdqu	ymm0, ymmword ptr [rdi]
[0,1]     D========eeeeER   vextracti128	xmm1, ymm0, 1
```

对于使用两次128位加载的情况，汇编如下：

```nasm
load_2sse:
    vmovdqu xmm0, [rdi]
    vmovdqu xmm1, [rsi]
```

分析结果摘录如下：

```log
Instruction Info:
[1]: #uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 1      8     0.50    *                   vmovdqu	xmm0, xmmword ptr [rdi]
 1      8     0.50    *                   vmovdqu	xmm1, xmmword ptr [rsi]


Timeline view:
                    0
Index     0123456789 

[0,0]     DeeeeeeeeER   vmovdqu	xmm0, xmmword ptr [rdi]
[0,1]     DeeeeeeeeER   vmovdqu	xmm1, xmmword ptr [rsi]
```

上述结果表明，vextracti128依赖于vmovdqu的结果，并且一个周期只能发射一条vextracti128指令（RThroughput=1）；而vmovdqu不存在数据依赖，且单周期可以发射两条指令（RThroughput=0.5）。因而我们可以断定，load_2sse（连续使用两次`_mm_load_si128`）的方案效率往往更优——在需要循环展开的场景，由于vmovdqu的发射效率高于vextracti128，load_2sse的方案更占优势。在需要做“长”时间（大于等于4个时钟周期）数据处理的常见场景，load_2sse可以缩短下游指令等待数据的耗时。只有在不做循环展开且几乎不怎么处理数据的罕见应用场景下，瓶颈才会落在vmovdqu上而不是vextracti128上，此时两个方案的效率才能堪堪打成平手。

### 为什么gcc编译出的main-v1性能更好？

gcc 14.2.0对v1版本的编译结果相较clang 19.1.0的编译结果的最大区别就是，gcc使用vmovdqu一次性加载32个uint8，而clang使用vpmovzxbd分四次读取共计16个uint8进行处理。当输入的首地址未按32字节对齐时，vmovdqu的性能会相当糟糕。但凑巧的是，本人实现的yuv读取会将首地址统统对齐到常见的cache line宽度，也就是64字节，这正巧导致采用一次性ymmword加载的gcc版本性能略优于clang版本（前者126.484ms对比后者130.703ms）。至于gcc版本的汇编分析这里就不赘述了，感兴趣的读者可以在Complier Explorer自行对比。

另外，在标量处理部分，gcc做了一个匪夷所思的循环展开，每个小执行块的后面都跟了一个分支跳转指令，这大幅膨胀了代码体积，而带来的性能收益，对于普遍巨大的数据长度而言十分有限。

那么，我们是否能提示编译器：“输入的`len`一般非常大”，来“诱导”优化呢？很遗憾，截止定稿的时候gcc和clang都不会对诸如`__builtin_expect(len >= 8192, true);`的提示作出任何反应。只能期待一下后续某位编译器高手的PR了。
