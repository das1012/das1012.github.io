---
title: 'Compute Shader中的各种ID'
date: 2021-05-19 00:36:12 +0800
tags: [Shader]
categories: [Shader]
math: true
memarid: true
---

在HLSL的Compute Shader中，main函数上方会有如下定义[numthreads(10, 8, 3)]；GLSL中也有类似的写法layout (local_size_x = 16, local_size_y = 16) in;

为了弄清楚该定义的意义，需要首先明确几个概念：

**Dispatch**：由多个线程组构成的组合。按照X，Y，Z三个维度进行分布。如Dispatch(5, 3, 2)表示5*3*2=30个线程组。

**numthreads(X, Y, Z)**以三维数组的形式定义了每个线程组中线程的数目。X * Y * Z的总数即为需要的总线程数。对于numthreads(10, 8, 3)，在X轴的索引范围为[0, 9]；在Y轴的索引范围为[0, 7]；在Z轴的索引范围为[0, 2]。

numthreads的参数取决于计算着色器的版本

| 着色器版本 | 最大Z | 最大线程(X * Y * Z) |
| ---------- | ----- | ------------------- |
| cs_4_x     | 1     | 768                 |
| cs_5_0     | 64    | 1024                |

**GroupID**：表示一个Dispatch中线程组的索引。

**GroupThreadID**：表示一个线程组里的线程相对于线程组(0, 0, 0)的索引。

**GroupIndex**：表示一个线程组里线程的一维索引，GroupIndex = GroupID.x + GroupID.y * numthreads.x + GroupID.z * numthreads.x * numthreads.y 

57 = 7 + 10 * 5 + 0 * 10 * 8  

**DispatchThreadID**：表示一个线程在DIspatch中的绝对索引，DispatchThreadID = GroupThreadID + GroupID * numthreads；(27, 13, 0) = (7, 5, 0) + (2, 1, 0) * (10, 8, 3)

![threadgroupids](./threadgroupids.png)

**参考资料**：

https://docs.microsoft.com/zh-cn/windows/win32/direct3dhlsl/sm5-attributes-numthreads