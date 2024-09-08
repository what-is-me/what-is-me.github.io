---
title: "泛读笔记：《dLSM: An LSM-Based Index for Memory Disaggregation》"
date: 2024-08-13 15:46:31
tags: 
    - Reading Note
    - DataBase
    - Hardware
    - RDMA
    - LSM-Tree
category: "泛读笔记"
mathjax: true
---
[Paper](https://www.cs.purdue.edu/homes/csjgwang/pubs/ICDE23_dLSM.pdf)

## 摘要

> The emerging trend of memory disaggregation where CPU and memory are physically separated from each other and are connected via ultra-fast networking, e.g., over RDMA, allows elastic and independent scaling of compute (CPU) and main memory. This paper investigates how indexing can be efficiently designed in the memory disaggregated architecture. Although existing research has optimized the B-tree for this new architecture, its performance is moderate. This paper focuses on LSM-based indexing and proposes dLSM, the first highly optimized LSM-tree for disaggregated memory. dLSM introduces a suite of optimizations including reducing software overhead, leveraging near-data computing, tuning for byte-addressability, and an instantiation over RDMA as a case study with RDMAspecific customizations to improve system performance. Experiments illustrate that dLSM achieves 1.6× to 11.7× higher write throughput than running the optimized B-tree and four adaptations of existing LSM-tree indexes over disaggregated memory. dLSM is written in C++ (with approximately 41,000 LOC), and is open-sourced.

新兴趋势内存分离（memory disaggregation）将CPU和内存物理上分开，并通过超快的网络（如 RDMA）连接。这使得计算（CPU）和主内存可以弹性地独立扩展。本文研究了如何在内存分离架构中高效设计索引。虽然现有的研究已对B树进行了优化，但其性能仍然一般。本文重点关注基于LSM树的索引，并提出了dLSM，这是首个针对分离内存高度优化的LSM树。dLSM引入了一系列优化措施，包括减少软件开销、利用近数据计算、针对字节寻址进行调优，以及以RDMA为例进行的定制化实现，以提升系统性能。实验结果表明，dLSM的写入吞吐量是优化后的B树和四种现有LSM树索引在分离内存上的适配的1.6到11.7倍。dLSM使用C++编写（约41,000行代码），并且是开源的。

## 介绍

### 面临的挑战

1. **软件开销**：高速网络大大拉近了访问本地和远端内存的性能鸿沟，相比于较慢的设备（SSD、HDD），使用分离内存时，软件开销更加突出
2. **内存节点计算资源的利用**：内存节点存在少量计算资源，具有提升性能的优化空间
3. **字节寻址特性**：相比于块寻址设备，内存节点时可字节寻址的，设计索引应当利用该特性
4. **有效利用通信接口**：比如单边、双边RDMA协议

### 本文贡献

1. 设计实现了一个分离式内存索引dLSM
2. 减少软件开销：比如，减少了同步开销
3. 近数据计算：把压缩操作下推给内存节点以减少数据传输
4. 专门为字节寻址特性优化
5. 专门为RDMA优化

### 总览

![](overview.png)

## 减少软件开销

![](F3.png)

- 使用无锁跳表
- 原子操作为每个写操作生成序列号
- 每张MemTable在生成时确定能插入的序列号范围，否则需要双重检查锁定来保证新MemTable中的数据比旧MemTable中的序列号大

## 近数据处理

- sstable的压缩不需要太多计算资源
- 把sstable读到计算节点，压缩再写回有极大通信开销
- 挑战：
  1. 如何放置LSM树元数据来促进查询
  2. 如何GC

### 元数据放置

- 元数据放置在计算节点
- 计算节点通过RPC发送压缩请求给内存节点，内存节点将压缩任务拆成子任务并行执行，执行完成后返回新元信息给计算节点，计算节点更新元信息使新sstable可见
- 元信息的更新使用排他锁同步，频率不高

### GC

- 为内存节点压缩分配的空间由内存节点回收，计算节点分配的用于flush的空间由计算节点回收
- 计算节点调用RPC命令回收远端内存，回收命令批量打包发送来节省带宽
- 读之前建立元数据快照并pin住要读的sstable，读完后unpin

## 为字节寻址特性进行的优化

![](F4.png)

- 点查直接返回一个kv数组而不是一块

## 多节点

![](F5.png)

使用分片，计算节点数c，内存节点数m，每个计算节点分成$\lambda$片
