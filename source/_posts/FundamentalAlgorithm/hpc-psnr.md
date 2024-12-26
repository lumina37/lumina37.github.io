---
title: PSNR的SIMD优化
categories: FundamentalAlgorithm
date: 2024/11/20 19:49:00
mathjax: true
---

## 前言

PSNR的算法非常简单，但与实际工程的距离又非常近。你可以在各种项目中看到它和它的变体。个人认为，PSNR非常适合作为入门SIMD（Single Instruction Multiple Data）汇编的研究对象。

性能优化的过程离不开各种开发工具，而“开发脚手架”的搭建，以及性能分析的过程往往不会在源代码中有所体现。这篇文章是我入门SIMD汇编的自学记录，同时也希望能给后来者做一些脚手架搭建方法上的参考。如有错漏欢迎直接给我的[blog](https://lumina.icu/)提PR或给我发邮件。

本文将涉及以下知识点：

- 如何使用docker搭建一个不干扰宿主机的调试环境。
- 利用Compiler Explorer编译代码片段，并阅读理解SIMD汇编。
- 如何使用perf工具的PMU（Performance Monitor Unit）统计功能来分析实际执行过程，进而明确性能提升的来源。
- 如何使用llvm-mca分析汇编执行方案。

最终，通过引入vpmaddwd指令以及循环展开，我们的实现相较ffmpeg加快了42倍（仅统计用户态耗时）。

## PSNR简介

PSNR，峰值信噪比，全称Peak Signal-to-Noise Ratio，在图像相似度、音频相似度等等各种相似度评估场景中你都能看到它的身影。

PSNR与均方误差（Mean Squared Error）有很直接的关联。所谓均方误差，就是先逐项作差，再取平方，最后对序列中的各个平方项取平均。PSNR与MSE关系可以用下式表示：

$$PSNR = 10 \cdot \log_{10}\left(\frac{MAX_I^2}{MSE}\right)$$

其中$MAX_I$为对应数据类型的最大值（对`uint8`为255）。

需要注意的是，PSNR是一个无量纲量。引入$MAX_I^2$正是为了“抹去”MSE的平方量纲。经由这个操作，不论数据类型的位宽是多少，PSNR都能提供同一尺度的相似性度量。

## 性能统计汇总

- 参评对象：ffmpeg及main-v1~v4
- 数据：300帧yuv420p的视频序列
- 硬件：AMD 7950X 定频4.5GHz
- 实验方案：8次缓存预热后重复计算128次全序列平均PSNR，统计用户态耗时的均值和标准差

时间统计脚本（由Claude3.5编写）

```sh
#!/bin/bash

# 检查是否提供了命令行参数
if [ $# -eq 0 ]; then
    echo "用法: $0 \"要执行的命令\""
    exit 1
fi

# 保存要执行的命令
COMMAND="$1"

# 预热缓存：执行8次命令
echo "预热缓存中（执行8次）..."
for ((i=1; i<=8; i++)); do
    $COMMAND > /dev/null 2>&1
done

# 存储执行时间的临时文件
TEMP_FILE=$(mktemp)

# 执行128次并记录用户态时间
echo "开始性能测试（执行128次）..."
for ((i=1; i<=128; i++)); do
    # 使用 time 命令，-f 格式化输出，%U 表示用户态时间
    /usr/bin/time -f "%U" -o "$TEMP_FILE.tmp" $COMMAND > /dev/null 2>&1
    
    # 追加时间到临时文件
    cat "$TEMP_FILE.tmp" >> "$TEMP_FILE"
done

# 计算均值和标准差
# 使用 awk 进行统计计算
STATS=$(awk '
    {
        sum += $1;
        sumsq += ($1 * $1);
        count++;
    }
    END {
        mean = sum / count;
        variance = (sumsq / count) - (mean * mean);
        stddev = sqrt(variance);
        printf "平均用户态时间: %.6f 秒\n标准差: %.6f 秒", mean, stddev
    }' "$TEMP_FILE")

# 显示统计结果
echo "测试命令: $COMMAND"
echo "$STATS"

# 清理临时文件
rm "$TEMP_FILE" "$TEMP_FILE.tmp"
```

运行命令参考：

```shell
./time.sh "./clang/main-v1 2048 2048 300 /path/to/lhs.yuv /path/to/rhs.yuv"
```

性能统计结果：

| 版本     | 平均用户态耗时 (ms) | 标准差 (ms) |
| -------- | ------------------- | ----------- |
| ffmpeg   | 951.797             | 239.062     |
| clang-v1 | 131.563             | 3.631       |
| clang-v2 | 49.141              | 3.069       |
| clang-v3 | 24.766              | 9.840       |
| clang-v4 | 22.656              | 15.835      |
| gcc-v1   | 127.500             | 8.004       |
| gcc-v2   | 40.078              | 1.975       |
| gcc-v3   | 28.047              | 22.223      |
| gcc-v4   | 24.609              | 10.451      |

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

测试数据为2GB长度的数组。`minicase-v1`的执行情况如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v1':

            850.99 msec task-clock                       #    1.000 CPUs utilized             
                 4      context-switches                 #    4.700 /sec                      
                 3      cpu-migrations                   #    3.525 /sec                      
         1,048,704      page-faults                      #    1.232 M/sec                     
     3,829,412,311      cycles                           #    4.500 GHz                       
       627,711,647      stalled-cycles-frontend          #   16.39% frontend cycles idle      
     7,341,906,139      instructions                     #    1.92  insn per cycle            
                                                  #    0.09  stalled cycles per insn   
       825,780,655      branches                         #  970.373 M/sec                     
         2,344,399      branch-misses                    #    0.28% of all branches           

       0.851319970 seconds time elapsed

       0.347096000 seconds user
       0.504139000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v1':

            832.69 msec task-clock                       #    1.000 CPUs utilized             
                 7      context-switches                 #    8.406 /sec                      
                 7      cpu-migrations                   #    8.406 /sec                      
         1,048,705      page-faults                      #    1.259 M/sec                     
     3,747,029,610      cycles                           #    4.500 GHz                       
       634,324,072      stalled-cycles-frontend          #   16.93% frontend cycles idle      
     7,921,290,568      instructions                     #    2.11  insn per cycle            
                                                  #    0.08  stalled cycles per insn   
       890,017,626      branches                         #    1.069 G/sec                     
         2,407,707      branch-misses                    #    0.27% of all branches           

       0.833023709 seconds time elapsed

       0.354981000 seconds user
       0.477974000 seconds sys
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

            696.73 msec task-clock                       #    1.000 CPUs utilized             
                 4      context-switches                 #    5.741 /sec                      
                 3      cpu-migrations                   #    4.306 /sec                      
         1,048,705      page-faults                      #    1.505 M/sec                     
     3,135,228,127      cycles                           #    4.500 GHz                       
       625,961,052      stalled-cycles-frontend          #   19.97% frontend cycles idle      
     6,314,586,514      instructions                     #    2.01  insn per cycle            
                                                  #    0.10  stalled cycles per insn   
     1,024,910,637      branches                         #    1.471 G/sec                     
         2,414,921      branch-misses                    #    0.24% of all branches           

       0.697014161 seconds time elapsed

       0.202994000 seconds user
       0.493987000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v2':

            705.42 msec task-clock                       #    1.000 CPUs utilized             
                 2      context-switches                 #    2.835 /sec                      
                 2      cpu-migrations                   #    2.835 /sec                      
         1,048,705      page-faults                      #    1.487 M/sec                     
     3,174,385,509      cycles                           #    4.500 GHz                       
       637,967,638      stalled-cycles-frontend          #   20.10% frontend cycles idle      
     6,169,517,815      instructions                     #    1.94  insn per cycle            
                                                  #    0.10  stalled cycles per insn   
     1,023,282,578      branches                         #    1.451 G/sec                     
         2,431,497      branch-misses                    #    0.24% of all branches           

       0.705632828 seconds time elapsed

       0.216196000 seconds user
       0.489444000 seconds sys
```

可以看到v2相较v1大幅减少了指令数和分支数。

### v3 - 改用madd

在SIMD指令集中，有一类特殊的指令可以实现先相乘后水平相加，即madd系列指令。下面使用`vpmaddwd`来优化`dump_unit`函数。

`vpmaddwd`的输入为两个`8 x i16`向量`[0,1,2,3,4,5,6,7]`和`[8,9,10,11,12,13,14,15]`，输出为一个`4 x i32`向量`[0*8+1*9,2*10+3*11,...]`。即先逐项相乘，再水平相加。

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

补充一下，简单的指令周期数（Latency）和吞吐量倒数（RThroughput）可以在[Intel官网](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)查询，更详细的参数可以在[uops.info](https://uops.info)查询。其中`RThroughput=x`的意思是每隔x个时钟周期可以发射一条指令，例如`RThroughput=0.5`意味着每时钟周期可以发射两条指令。

`vpmaddwd`和`vpmullw`的吞吐量和消耗的周期数几乎一致。修改后我们在热点循环中节省了两条unpack指令和一条add指令。

v3版本的执行耗时如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v3':

            677.86 msec task-clock                       #    1.000 CPUs utilized             
                 3      context-switches                 #    4.426 /sec                      
                 3      cpu-migrations                   #    4.426 /sec                      
         1,048,705      page-faults                      #    1.547 M/sec                     
     3,050,371,910      cycles                           #    4.500 GHz                       
       628,063,961      stalled-cycles-frontend          #   20.59% frontend cycles idle      
     5,908,509,861      instructions                     #    1.94  insn per cycle            
                                                  #    0.11  stalled cycles per insn   
     1,024,316,223      branches                         #    1.511 G/sec                     
         2,406,187      branch-misses                    #    0.23% of all branches           

       0.678125205 seconds time elapsed

       0.198028000 seconds user
       0.480069000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v3':

            680.48 msec task-clock                       #    1.000 CPUs utilized             
                 3      context-switches                 #    4.409 /sec                      
                 1      cpu-migrations                   #    1.470 /sec                      
         1,048,704      page-faults                      #    1.541 M/sec                     
     3,062,133,602      cycles                           #    4.500 GHz                       
       633,752,352      stalled-cycles-frontend          #   20.70% frontend cycles idle      
     5,779,014,107      instructions                     #    1.89  insn per cycle            
                                                  #    0.11  stalled cycles per insn   
     1,025,058,381      branches                         #    1.506 G/sec                     
         2,388,288      branch-misses                    #    0.23% of all branches           

       0.680707686 seconds time elapsed

       0.194193000 seconds user
       0.486484000 seconds sys
```

在AVX VNNI指令集中引入了一个`vpdpwssd`指令作为`vpmaddwd`和`vpaddd`的融合版本，其延迟和吞吐量均与`vpmaddwd`一致。如果使用vpdpwssd指令，则`dump_unit`还可以进一步简化为：

```nasm
vpsubw    ymm1, ymm1, ymm2
vpdpwssd  ymm0, ymm1, ymm1
```

不过很可惜支持这一指令集的CPU范围十分有限，因此我这里就不做benchmark了。

### v4 - 循环展开

下面使用手动循环展开进一步提高性能。

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
    constexpr size_t unroll = 8;
    constexpr size_t group_len = u32max / (u8max * u8max * 2) * unroll;
    const size_t unroll_cnt = m128_cnt / unroll;
    // 还有`nounroll_cnt`个`__m128i`不成组
    const size_t nounroll_cnt = m128_cnt - unroll_cnt * unroll;

    uint64_t sqr_diff_acc = 0;
    std::array<__m256i, unroll> u32sqr_diff_acc{};

    auto dump_unit = [&](const __m256i u8l, const __m256i u8r, const size_t i) mutable {
        const __m256i i16diff = _mm256_sub_epi16(u8l, u8r);
        const __m256i u32sqr_diff = _mm256_madd_epi16(i16diff, i16diff);
        u32sqr_diff_acc[i] = _mm256_add_epi32(u32sqr_diff_acc[i], u32sqr_diff);
    };

    auto dump_u32sqr_diff_acc = [&](const size_t i) mutable {
        __m256i u64sqr_diff_acc_p0 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc[i], 0));
        __m256i u64sqr_diff_acc_p1 = _mm256_cvtepu32_epi64(_mm256_extractf128_si256(u32sqr_diff_acc[i], 1));
        __m256i u64sqr_diff_acc = _mm256_add_epi64(u64sqr_diff_acc_p0, u64sqr_diff_acc_p1);
        auto* tmp = (uint64_t*)&u64sqr_diff_acc;
        sqr_diff_acc += (tmp[0] + tmp[1] + tmp[2] + tmp[3]);
    };

    size_t count = 0;
    for (size_t i = 0; i < unroll_cnt; i++) {
        for (size_t j = 0; j < unroll; j++) {
            const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
            const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
            dump_unit(u8l, u8r, j);
            lhs_cursor += step;
            rhs_cursor += step;
        }
        count++;

        if (count == group_len) [[unlikely]] {
            for (size_t j = 0; j < unroll; j++) {
                dump_u32sqr_diff_acc(j);
                u32sqr_diff_acc[j] = _mm256_setzero_si256();
            }
            count = 0;
        }
    }

    dump_u32sqr_diff_acc(0);
    u32sqr_diff_acc[0] = _mm256_setzero_si256();

    for (size_t i = 0; i < nounroll_cnt; i++) {
        const __m256i u8l = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)lhs_cursor));
        const __m256i u8r = _mm256_cvtepu8_epi16(_mm_load_si128((__m128i*)rhs_cursor));
        dump_unit(u8l, u8r, 0);
        lhs_cursor += step;
        rhs_cursor += step;
    }

    for (size_t j = 0; j < unroll; j++) {
        dump_u32sqr_diff_acc(j);
    }

    return sqr_diff_acc;
}
```

v4的性能表现如下：

gcc

```log
 Performance counter stats for './gcc/minicase-v4':

            681.10 msec task-clock                       #    1.000 CPUs utilized             
                 2      context-switches                 #    2.936 /sec                      
                 2      cpu-migrations                   #    2.936 /sec                      
         1,048,705      page-faults                      #    1.540 M/sec                     
     3,064,939,989      cycles                           #    4.500 GHz                       
       630,948,088      stalled-cycles-frontend          #   20.59% frontend cycles idle      
     5,251,876,822      instructions                     #    1.71  insn per cycle            
                                                  #    0.12  stalled cycles per insn   
       791,899,979      branches                         #    1.163 G/sec                     
         2,397,838      branch-misses                    #    0.30% of all branches           

       0.681312237 seconds time elapsed

       0.183080000 seconds user
       0.498218000 seconds sys
```

clang

```log
 Performance counter stats for './clang/minicase-v4':

            674.71 msec task-clock                       #    1.000 CPUs utilized             
                 3      context-switches                 #    4.446 /sec                      
                 1      cpu-migrations                   #    1.482 /sec                      
         1,048,704      page-faults                      #    1.554 M/sec                     
     3,036,197,988      cycles                           #    4.500 GHz                       
       629,450,363      stalled-cycles-frontend          #   20.73% frontend cycles idle      
     5,121,711,337      instructions                     #    1.69  insn per cycle            
                                                  #    0.12  stalled cycles per insn   
       789,771,090      branches                         #    1.171 G/sec                     
         2,336,245      branch-misses                    #    0.30% of all branches           

       0.674943834 seconds time elapsed

       0.186978000 seconds user
       0.487943000 seconds sys
```

虽然分支数进一步减少，但错误分支计数并没有下降太多，因此性能提升并不显著。

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

其中clang v19.1.0生成的汇编如下：

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

相较我们在v3中的手写实现，由于需要直接累加到`uint64`，缺少`uint32`作为中继，编译器在累加部分额外使用了大量指令用来将`4 x u32`扩展为`4 x u64`。

此外，我们的v4版本的优化方案事实上参考了这里clang生成的汇编，包括循环展开，以及使用四个独立的累加器这两项优化。其中，独立的累加器可以避免过早地合并结果（Reduce），减少数据依赖导致的流水线停顿。

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

上述结果表明，`vextracti128`依赖于`vmovdqu`的结果，并且一个周期只能发射一条`vextracti128`指令（`RThroughput=1`）；而`vmovdqu`不存在数据依赖，且单周期可以发射两条指令（`RThroughput=0.5`）。因而我们可以断定，`load_2sse`（连续使用两次`_mm_load_si128`）的方案效率往往更优——在需要循环展开的场景，由于`vmovdqu`的发射效率高于`vextracti128`，`load_2sse`的方案更占优势。在需要做“长”时间（大于等于4个时钟周期）数据处理的常见场景，`load_2sse`可以缩短下游指令等待数据的耗时。只有在不做循环展开且几乎不怎么处理数据的罕见应用场景下，瓶颈才会落在`vmovdqu`上而不是`vextracti128`上，此时两个方案的效率才能堪堪打成平手。

### 为什么gcc编译出的main-v1性能更好？

gcc v14.2.0对v1版本的编译结果相较clang v19.1.0的编译结果的最大区别就是，gcc使用`vmovdqu`一次性加载32个`uint8`，而clang使用`vpmovzxbd`分四次读取共计16个`uint8`进行处理。当输入的首地址未按32字节对齐时，`vmovdqu`的性能会相当糟糕。但凑巧的是，本人实现的yuv读取会将首地址统统对齐到常见的cache line宽度，也就是64字节，这正巧导致采用一次性`ymmword`加载的gcc版本性能略优于clang版本（前者127.500ms对比后者131.563ms）。至于gcc版本的汇编分析这里就不赘述了，感兴趣的读者可以在Complier Explorer自行对比。

另外，在标量处理部分，gcc做了一个匪夷所思的循环展开，每个小执行块的后面都跟了一个分支跳转指令，这大幅膨胀了代码体积，而带来的性能收益，对于普遍巨大的数据长度而言十分有限。

那么，我们是否能提示编译器：“输入的`len`一般非常大”，来“诱导”优化呢？很遗憾，截止定稿的时候gcc和clang都不会对诸如`__builtin_expect(len >= 8192, true);`的提示作出任何反应。只能期待一下后续某位编译器高手的PR了。

## 深入研究clang对ffmpeg实现的优化问题

### 发现问题

在ffmpeg中，核心的SSE（Sum of Squared Error，累加均方误差）计算函数是一个短短数行的`sse_line_8bit`。

```c
uint64_t sse_line_8bit(const uint8_t *main_line, const uint8_t *ref_line, int outw) {
    int j;
    unsigned m2 = 0;

    for (j = 0; j < outw; j++) {
        unsigned error = main_line[j] - ref_line[j];

        m2 += error * error;
    }

    return m2;
}
```

我们的v1版本与ffmpeg实现的主要区别在于，v1中的`acc`的数据类型是`uint64`，而ffmpeg中的`acc`的数据类型是`uint32`。将`acc`的数据类型改成`uint32`后，编译出的求差值平方和的部分就与ffmpeg实现一致了。

此外，不论`error`的数据类型是`uint32`还是`int16`都不会影响编译结果。编译器前端在生成LLVM IR时会自行分析需要的中间变量类型。

由于下一个步骤需要魔改汇编，因此我们需要先给`sse_line_8bit`补一个生成输入数据的环境。

```c
#include <stdlib.h>
#include <stdint.h>
#include <stdio.h>

void randbytes(uint8_t *buffer, size_t size) {
    for (size_t i = 0; i < size; i++) {
        buffer[i] = rand() & 0xFF;
    }
}

__attribute__((noinline)) uint64_t sse_line_8bit(const uint8_t *main_line, const uint8_t *ref_line, int outw) {
    int j;
    unsigned m2 = 0;

    for (j = 0; j < outw; j++) {
        unsigned error = main_line[j] - ref_line[j];

        m2 += error * error;
    }

    return m2;
}

int main() {
    srand(37);

    const int size = 4096;

    void *buffer = malloc(size * 2);
    if (buffer == NULL) {
        return -1;
    }

    randbytes(buffer, size * 2);

    const uint8_t *main_line = (uint8_t *) buffer;
    const uint8_t *ref_line = main_line + size;
    const uint64_t sse = sse_line_8bit(main_line, ref_line, size);

    free(buffer);

    printf("%lu\n", sse);
}
```

编译并运行：

```shell
clang sse.c -O3 -mavx2 -o sse-auto
./sse-auto
```

期望输出为`45530600`。

使用以下指令生成汇编：

```shell
clang sse.c -O3 -mavx2 -S -masm=intel -o sse-noymm.S
```

找到关键循环节如下：

```nasm
.LBB1_4:
        ; 这里开始作差+取平方（用了一个4x循环展开）
        vpmovzxbw       xmm4, qword ptr [rdi + rax]
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
        vpsubw  xmm7, xmm7, xmm9
        vpmaddwd        xmm4, xmm4, xmm4
        vpaddd  ymm0, ymm4, ymm0
        vpmaddwd        xmm4, xmm5, xmm5
        vpaddd  ymm1, ymm4, ymm1
        vpmaddwd        xmm4, xmm6, xmm6
        vpaddd  ymm2, ymm4, ymm2
        vpmaddwd        xmm4, xmm7, xmm7
        vpaddd  ymm3, ymm4, ymm3
        ; 下面判断是否退出循环
        add     rax, 32
        cmp     rdx, rax
        jne     .LBB1_4
        ; 下面将四个累加器ymm0~3中的结果归集到eax中
        vpaddd  ymm0, ymm1, ymm0
        vpaddd  ymm0, ymm2, ymm0
        vpaddd  ymm0, ymm3, ymm0
        vextracti128    xmm1, ymm0, 1
        vpaddd  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 238
        vpaddd  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 85
        vpaddd  xmm0, xmm0, xmm1
        vmovd   eax, xmm0
        cmp     edx, ecx
        je      .LBB1_7
```

可以看到，当`m2`（对应我们的v1中的`acc`）为uint32时，clang也使用了vpmaddwd来优化乘加运算。

然而，对比我们手动实现的优化，这里clang的自动优化存在三个问题：

1. `vpmaddwd`的目标操作数是`xmm`，也就是仅产生了低128位的有效数据，而在累加阶段，`vpaddd`却使用了256位宽的`ymm`，为什么要把高128位的无效数据纳入计算？
2. 为什么不在作差（`vpsubw`）和自乘（`vpmaddwd`）部分使用`ymm`寄存器？
3. 在加载数据时为什么只使用64位宽的`qword`加载而不使用128位宽的`xmmword`加载？

### 初步验证问题1和2的改进可行性

在深入LLVM的内部实现之前，我们首先通过两个简单的实验来验证问题1和2确实存在改进的可行性。

首先验证vpaddd并不需要使用ymm寄存器的高128位，魔改后的汇编代码如下：

```nasm
.LBB1_4:
        ; 这里开始作差+取平方
        vpmovzxbw       xmm4, qword ptr [rdi + rax]
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
        vpsubw  xmm7, xmm7, xmm9
        vpmaddwd        xmm4, xmm4, xmm4
        vpaddd  xmm0, xmm4, xmm0
        vpmaddwd        xmm4, xmm5, xmm5
        vpaddd  xmm1, xmm4, xmm1
        vpmaddwd        xmm4, xmm6, xmm6
        vpaddd  xmm2, xmm4, xmm2
        vpmaddwd        xmm4, xmm7, xmm7
        vpaddd  xmm3, xmm4, xmm3
        ; 下面判断是否退出循环
        add     rax, 32
        cmp     rdx, rax
        jne     .LBB1_4
        ; 下面将四个累加器xmm0~3中的结果归集到eax中
        vpaddd  xmm0, xmm1, xmm0
        vpaddd  xmm0, xmm2, xmm0
        vpaddd  xmm0, xmm3, xmm0
        vpshufd xmm1, xmm0, 238
        vpaddd  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 85
        vpaddd  xmm0, xmm0, xmm1
        vmovd   eax, xmm0
        cmp     edx, ecx
        je      .LBB1_7
```

编译魔改后的汇编代码：

```shell
clang sse-noymm.S -o sse-noymm
./sse-noymm
```

输出为`45530600`，与先前结果一致，说明这里的ymm的高128位确实没有意义。

再验证一次性加载128位的方法确实可行。因为我懒，不想改循环条件，就把循环展开的次数缩减了一半。

```nasm
.LBB1_4:
        ; 这里开始作差+取平方
        vpmovzxbw       ymm4, xmmword ptr [rdi + rax]
        vpmovzxbw       ymm5, xmmword ptr [rdi + rax + 16]
        vpmovzxbw       ymm8, xmmword ptr [rsi + rax]
        vpsubw  ymm4, ymm4, ymm8
        vpmovzxbw       ymm8, xmmword ptr [rsi + rax + 16]
        vpsubw  ymm5, ymm5, ymm8
        vpmaddwd        ymm4, ymm4, ymm4
        vpaddd  ymm0, ymm4, ymm0
        vpmaddwd        ymm4, ymm5, ymm5
        vpaddd  ymm1, ymm4, ymm1
        ; 下面判断是否退出循环
        add     rax, 32
        cmp     rdx, rax
        jne     .LBB1_4
        ; 下面将累加器ymm0和ymm1中的结果归集到eax中
        vpaddd  ymm0, ymm1, ymm0
        vextracti128    xmm1, ymm0, 1
        vpaddd  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 238
        vpaddd  xmm0, xmm0, xmm1
        vpshufd xmm1, xmm0, 85
        vpaddd  xmm0, xmm0, xmm1
        vmovd   eax, xmm0
        cmp     edx, ecx
        je      .LBB1_7
```

编译魔改后的汇编代码：

```shell
clang sse-allymm.S -o sse-allymm
./sse-allymm
```

输出为`45530600`，还是与先前结果一致，说明一次性加载128位后续再用`ymm`操作也是可以的。

小结一下，使用`qword`加载并使用`xmm`作为累加器是一种方案，使用`xmmword`加载并使用`ymm`作为累加器是一种方案，clang却偏偏使用了这么一种`qword`加载+使用`ymm`作为累加器的别扭方案。这到底是为什么呢？

### 排查各优化步骤（分锅大会）

下面开始详细排查各个优化步骤。在Compiler Explorer中打开LLVM的Optimization Pipeline视图。通过逐个查看diff可以发现，虽然Partial Reduction拉了最大的一坨，但其实拉下最关键一坨的关键先生还是Unroll Loop。可以说如果没有Unroll Loop的那一坨，Partial Reduction甚至不会起作用。

我们还是先来看看Partial Reduction的作用，其执行前的LLVM IR大致如下：

```llvm
vector.body:
  ; 变量加载
  ...
  ; 作差
  %16 = sub nsw <8 x i32> %4, %12
  %17 = sub nsw <8 x i32> %5, %13
  %18 = sub nsw <8 x i32> %6, %14
  %19 = sub nsw <8 x i32> %7, %15
  ; 自乘
  %20 = mul nsw <8 x i32> %16, %16
  %21 = mul nsw <8 x i32> %17, %17
  %22 = mul nsw <8 x i32> %18, %18
  %23 = mul nsw <8 x i32> %19, %19
  ; 累加
  %24 = add <8 x i32> %20, %vec.phi
  %25 = add <8 x i32> %21, %vec.phi14
  %26 = add <8 x i32> %22, %vec.phi15
  %27 = add <8 x i32> %23, %vec.phi16
  ; 循环条件判断
  ...
```

这里我省略了一些变量加载和循环条件判断的细节。在Partial Reduction之后，自乘部分的实现发生了巨大变化，以下是Partial Reduction后的LLVM IR：

```
vector.body:
  ; 变量加载
  ...
  ; 作差
  %8 = sub nsw <8 x i32> %0, %4
  %9 = sub nsw <8 x i32> %1, %5
  %10 = sub nsw <8 x i32> %2, %6
  %11 = sub nsw <8 x i32> %3, %7
  ; 自乘
  %12 = mul <8 x i32> %8, %8
  %13 = shufflevector <8 x i32> %12, <8 x i32> %12, <4 x i32> <i32 0, i32 2, i32 4, i32 6>
  %14 = shufflevector <8 x i32> %12, <8 x i32> %12, <4 x i32> <i32 1, i32 3, i32 5, i32 7>
  %15 = add <4 x i32> %13, %14
  %16 = shufflevector <4 x i32> %15, <4 x i32> zeroinitializer, <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7>
  %17 = mul <8 x i32> %9, %9
  %18 = shufflevector <8 x i32> %17, <8 x i32> %17, <4 x i32> <i32 0, i32 2, i32 4, i32 6>
  %19 = shufflevector <8 x i32> %17, <8 x i32> %17, <4 x i32> <i32 1, i32 3, i32 5, i32 7>
  %20 = add <4 x i32> %18, %19
  %21 = shufflevector <4 x i32> %20, <4 x i32> zeroinitializer, <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7>
  %22 = mul <8 x i32> %10, %10
  %23 = shufflevector <8 x i32> %22, <8 x i32> %22, <4 x i32> <i32 0, i32 2, i32 4, i32 6>
  %24 = shufflevector <8 x i32> %22, <8 x i32> %22, <4 x i32> <i32 1, i32 3, i32 5, i32 7>
  %25 = add <4 x i32> %23, %24
  %26 = shufflevector <4 x i32> %25, <4 x i32> zeroinitializer, <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7>
  %27 = mul <8 x i32> %11, %11
  %28 = shufflevector <8 x i32> %27, <8 x i32> %27, <4 x i32> <i32 0, i32 2, i32 4, i32 6>
  %29 = shufflevector <8 x i32> %27, <8 x i32> %27, <4 x i32> <i32 1, i32 3, i32 5, i32 7>
  %30 = add <4 x i32> %28, %29
  %31 = shufflevector <4 x i32> %30, <4 x i32> zeroinitializer, <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7>
  ; 累加
  %32 = add <8 x i32> %16, %vec.phi
  %33 = add <8 x i32> %21, %vec.phi14
  %34 = add <8 x i32> %26, %vec.phi15
  %35 = add <8 x i32> %31, %vec.phi16
  ; 循环条件判断
  ...
```

其中，自乘部分的各个单元片段都将在后续的isel步骤中被替换为`vpmaddwd`指令。

单元片段详解如下：

```llvm
%12 = mul <8 x i32> %8, %8
%13 = shufflevector <8 x i32> %12, <8 x i32> %12, <4 x i32> <i32 0, i32 2, i32 4, i32 6>
%14 = shufflevector <8 x i32> %12, <8 x i32> %12, <4 x i32> <i32 1, i32 3, i32 5, i32 7>
%15 = add <4 x i32> %13, %14
%16 = shufflevector <4 x i32> %15, <4 x i32> zeroinitializer, <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7>
```

如果`mul <8 x i32> %8, %8`的输出是`[0,1,2,3,4,5,6,7]`，那么`%13`是所有的偶数位即`[0,2,4,6]`，`%14`是奇数位。`%13`与`%14`加和后的`%15`是`[0+1,2+3,...]`。最后由一个`shufflevector`将`[0+1,2+3,...]`的高128位全部补零，扩展为`[0+1,2+3,4+5,6+7,0,0,0,0]`。

这一步莫名其妙的Partial Reduction使得累加阶段虽然在名义上使用了ymm寄存器，但实际只有低128位（`4 x i32`）中的数据才是有效数据。至于作差时为何不用ymm，我认为这是由于LLVM知道值域局限在i16的范围内，用xmm对应的`<8 x i16>`即可，不需要真的用上LLVM IR中的`<8 x i32>`。

### 分锅大会

虽然看上去Partial Reduction拉了最大的一坨，但其实拉下最关键一坨的关键先生还是Unroll Loop，它在作差-自乘-累加的三个阶段都使用了`8 x i32`作为向量表示。如果没有Unroll Loop图省事糊了三组`8 x i32`上去，后面都没Partial Reduction什么事了。

TODO：Unroll Loop的实现实在太复杂了，后面再看

### TODO：循环展开的实现优化
