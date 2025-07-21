---
title: GLSL高斯模糊优化
categories: Vulkan
date: 2025/04/16 19:25:00
---

## 前言

// 模板参数:
// RADIUS: 高斯核的半径（决定模糊强度）
// KX: CUDA线程分配的X方向块尺寸
template <int RADIUS, int KX>
__global__ void gGaussOptim(const float* __restrict__ src, float* __restrict__ dst, int width, int height, int stride)
{
    // 共享内存用于水平方向处理
    // 大小: (2*RADIUS)行 x (KX + 2*RADIUS)列，用于处理边界像素
    __shared__ float pmem[RADIUS * 2][KX + RADIUS * 2];
    
    // 共享内存用于存储水平方向处理后的中间结果
    // 大小: (4*RADIUS)行 x KX列
    __shared__ float smem[RADIUS * 4][KX];

    // 预计算一些常用值，避免重复计算
    const int R2 = RADIUS << 1;     // 2*RADIUS，使用位移运算（乘以2）
    const int R4 = RADIUS << 2;     // 4*RADIUS，使用位移运算（乘以4）
    const int R8 = RADIUS << 3;     // 8*RADIUS，使用位移运算（乘以8）
    
    // 块内的线程索引
    const int tix = threadIdx.x;    // 块内的X方向线程索引
    const int tiy = threadIdx.y;    // 块内的Y方向线程索引
    
    // 全局线程坐标（整个图像中的位置）
    const int ix = blockIdx.x * KX + tix;   // 全局X坐标
    const int iy = blockIdx.y * R8 + tiy;   // 全局Y坐标（每个块处理8*RADIUS行）
    
    // 数组用于存储反射坐标，用于访问边界像素
    int xs[R2 + 1];
    
    // 计算中心像素坐标，在图像边界处进行反射
    xs[RADIUS] = reflectBorder(ix, width);
    
    // 计算核半径范围内所有方向的反射坐标
    for (int i = 1; i <= RADIUS; ++i)
    {
        xs[RADIUS - i] = reflectBorder(ix - i, width);  // 左侧反射
        xs[RADIUS + i] = reflectBorder(ix + i, width);  // 右侧反射
    }
    
    // 共享内存索引常量
    const int six = tix + RADIUS;   // 共享内存X索引，偏移RADIUS
    const int pix = tix + R2;       // 用于最右边界像素的额外共享内存索引
    
    // 处理边界情况的条件
    const bool left_cond = tix < RADIUS;                // 对于处理左边界的线程为真
    const bool righ_cond = six >= KX || ix + RADIUS >= width;  // 对于处理右边界的线程为真

    // ---- 第一个半径范围 - 将初始数据加载到共享内存 ----
    
    // 计算源图像第一行的偏移量（目标行上方RADIUS像素）
    int offset = reflectBorder(iy - RADIUS, height) * stride;
    
    // 将中心像素加载到共享内存
    pmem[tiy][six] = src[offset + xs[RADIUS]];
    
    // 将左右边界像素加载到共享内存（仅适用于相应的线程）
    if (left_cond) pmem[tiy][tix] = src[offset + xs[0]];        // 左边界
    if (righ_cond) pmem[tiy][pix] = src[offset + xs[R2]];       // 右边界
    
    // 同步以确保所有线程都完成了数据加载到共享内存
    __syncthreads();

    // ---- 处理第二到第四个半径范围 ----
    #pragma unroll  // 编译器指令，展开循环以提高性能
    for (int range = RADIUS; range < R4; range += RADIUS)
    {
        // 将新行数据加载到共享内存
        int piy = (tiy + range) % R2;  // 以循环方式重用共享内存行
        offset = reflectBorder(iy + range - RADIUS, height) * stride;  // 计算源偏移量
        
        // 加载中心、左边界和右边界像素
        pmem[piy][six] = src[offset + xs[RADIUS]];
        if (left_cond) pmem[piy][tix] = src[offset + xs[0]];
        if (righ_cond) pmem[piy][pix] = src[offset + xs[R2]];

        // 执行水平方向模糊（行方向的一维卷积）
        int siy = tiy + range - RADIUS;
        piy = siy % R2;
        
        // 应用中心权重
        float res = __fmul_rn(d_const_kernel[0], pmem[piy][six]);
        
        // 添加相邻像素的加权贡献
        for (int i = 1; i <= RADIUS; ++i)
        {
            // 添加对称像素的加权和
            res += __fmul_rn(d_const_kernel[i], pmem[piy][six - i] + pmem[piy][six + i]);
        }
        
        // 将水平方向处理结果存储在中间缓冲区
        smem[siy][tix] = res;
        
        // 在下一次迭代前同步
        __syncthreads();
    }

    // ---- 处理中间范围（第四到R8-RADIUS）----
    #pragma unroll  // 展开以提高性能
    for (int range = RADIUS; range < R8 - RADIUS; range += RADIUS)
    {
        // 将新行数据加载到共享内存
        int siy = tiy + range;
        int piy = (siy + RADIUS) % R2;  // 计算循环缓冲区中的行
        offset = reflectBorder(iy + range + R2, height) * stride;  // 获取源偏移量
        
        // 加载中心、左边界和右边界像素
        pmem[piy][six] = src[offset + xs[RADIUS]];
        if (left_cond) pmem[piy][tix] = src[offset + xs[0]];
        if (righ_cond) pmem[piy][pix] = src[offset + xs[R2]];

        // 执行垂直方向模糊并写入已完成行的输出
        int ny = iy + range - RADIUS;  // 输出的y坐标
        if (ix < width && ny < height)  // 检查是否在图像边界内
        {
            // 应用中心权重
            float res = __fmul_rn(d_const_kernel[0], smem[siy % R4][tix]);
            
            // 添加相邻行的加权贡献（垂直方向处理）
            for (int i = 1; i <= RADIUS; ++i)
            {
                // 添加对称行的加权和
                res += __fmul_rn(d_const_kernel[i], smem[(siy - i) % R4][tix] + smem[(siy + i) % R4][tix]);
            }
            
            // 将最终模糊像素写入目标
            dst[ny * stride + ix] = res;
        }    

        // 处理新行的水平方向处理（为未来的垂直方向处理准备）
        siy += R2;
        piy = siy % R2;
        float res = __fmul_rn(d_const_kernel[0], pmem[piy][six]);
        for (int i = 1; i <= RADIUS; ++i)
        {
            res += __fmul_rn(d_const_kernel[i], pmem[piy][six - i] + pmem[piy][six + i]);
        }
        smem[siy % R4][tix] = res;

        // 在下一次迭代前同步
        __syncthreads();
    }

    // ---- 处理最后两个范围（最终输出行）----
    if (ix < width)  // 检查是否在图像宽度内
    {
        // 对最后一行执行水平方向处理
        int siy = tiy + R8 + RADIUS;
        int piy = siy % R2;
        float res = __fmul_rn(d_const_kernel[0], pmem[piy][six]);
        for (int i = 1; i <= RADIUS; ++i)
        {
            res += __fmul_rn(d_const_kernel[i], pmem[piy][six - i] + pmem[piy][six + i]);
        }
        smem[siy % R4][tix] = res;

        // 处理并写入倒数第二行
        int ny = iy + R8 - R2;
        if (ny >= height) return;  // 如果超出图像高度则提前退出
        
        siy = tiy + R8 - RADIUS;
        res = __fmul_rn(d_const_kernel[0], smem[siy % R4][tix]);
        for (int i = 1; i <= RADIUS; ++i)
        {
            res += __fmul_rn(d_const_kernel[i], smem[(siy - i) % R4][tix] + smem[(siy + i) % R4][tix]);
        }
        dst[ny * stride + ix] = res;

        // 处理并写入最后一行
        ny += RADIUS;
        if (ny >= height) return;  // 如果超出图像高度则提前退出
        
        siy += RADIUS;
        res = __fmul_rn(d_const_kernel[0], smem[siy % R4][tix]);
        for (int i = 1; i <= RADIUS; ++i)
        {
            res += __fmul_rn(d_const_kernel[i], smem[(siy - i) % R4][tix] + smem[(siy + i) % R4][tix]);
        }
        dst[ny * stride + ix] = res;
    }
}