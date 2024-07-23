---
title: ascend c 南京站
date: 2024/07/23 14:08:37
tags:
  - ai
categories:
  - ascend c
share: "true"
---

# AI core

矩阵/向量/标量


存储：绕不过的memory heriachy


面向芯片编程：


统一抽象为local/global memory





# operator



数学上：函数空间->函数空间的映射

名称/类型：类和实例

输入输出：张量tensor

tensor：多维数据，具有shape；可定义数据的物理排布形式


NHWC/NCHW：从后往前遍历，顺序不同

轴axis：数据维度的编号，例如x轴为0，y轴为1


# asecnd c

### host/device：异构计算

重点在于对设备的编程

在ai-core上跑的函数：核函数

```c
__global__ __aicore__ void func(...)



```


核函数调用：

```c
kernel_name<<<在几个核心运行, 来ctrl, stream异步任务队列>>>
```


编程模型：SPMD

数据分块给不同core

各种api，不同粒度的加法


### 编程范式

向量计算：输入，计算，输出

此阶段可以流水线化，通过VECIN/VECOUT阻塞队列完成同步：GM->LM->VECIN


### 开发

算子类



## host侧

tiling分块：动态shape，算出固定内存用量








