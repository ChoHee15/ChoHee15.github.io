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


存储：绕不过的memory  hierarchy


面向芯片编程：更底层的控制能力


存储统一抽象为local/global memory





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

算子类包装



## host侧

tiling分块：动态shape，算出固定内存用量





# 实验

第一次用这种开发板，连接香橙派有问题，RJ45口不闪。最后换了一块新的就正常了。

ssh连上之后发现用的Linux arm64，``cat /proc/cpuinfo``发现是4核。~~搜了一下要800多~~

环境是准备好的，gcc/g++都在。之后按实验手册做下去，一开始result就是错的很奇怪，可能是以前人动过没还原。

后面要在不用高级数学库的情况下拼一个``sinh``出来，刚开始完全没理解要干啥。然后觉得这不得要中间变量存结果吗……搜了一下这么搞中间变量。结果老师说可以覆盖写，不用加新东西

waitwhat覆盖写是个啥意思，没懂。后面尝试直接用yLocal搞了。

直接用vim写的，感觉没有配置的vim有点痛苦

不知道能不能vscode remote之




