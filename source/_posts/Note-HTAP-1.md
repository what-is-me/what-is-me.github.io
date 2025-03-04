---
mathjax: true
title: '泛读笔记：《Two Birds With One Stone: Designing a Hybrid Cloud Storage Engine for HTAP》'
date: 2025-02-10 16:30:51
tags:
    - Reading Note
    - DataBase
    - HTAP
category: "泛读笔记"
---
[Paper](https://www.vldb.org/pvldb/vol17/p3290-schmidt.pdf)

## 摘要

企业对最新数据的实时分析需求日益增长。然而，当前的解决方案无法在同一系统中高效地结合事务处理和分析处理，而是依赖ETL（提取-转换-加载）流程将事务数据传输到分析系统，从而导致数据洞察的延迟。

本文针对这一需求，提出了一种面向云环境的新型存储引擎设计——Colibri，该引擎支持超越主内存的混合事务与分析处理（HTAP）。Colibri 采用混合列存与行存架构，针对两种工作负载进行优化，并充分利用新兴硬件趋势。它通过有效区分冷热数据，以适应不同的访问模式和存储设备。

我们的广泛实验表明，在固态硬盘（SSD）和云对象存储上处理混合工作负载时，Colibri 可实现高达 10 倍的性能提升。

## 背景

### AP和TP负载特点不同

- AP：列存、压缩、顺序扫描
- TP：点查、行存、索引
- 过去，纯内存的HTAP较容易实现，但是扩展性差、对大数据集来说昂贵
- 对于内存放不下的数据集，必须要考虑再持久设备中如何存放

### 存储技术的发展

- SSD带宽高，延迟低
- 云对象存储带宽高，中等延迟，不受限制的容量，高耐用性，高可用，便宜

### 现存云HTAP技术

![TIDB & ByteDance & SingleStore](fig_ppt1.png)

- TIDB & ByteDance
  - TIDB: TP引擎将日志传给AP引擎
  - ByteDance: TP将日志写入log server，log server将日志传给行存和列存page server
  - 数据被复制了多遍，日志是瓶颈
- SingleStore
  - 避免了数据重复，冷热数据分离，热数据放在内存，冷数据放在对象存储/SSD中
  - 使用LSM树，点查较慢，并且缺少page server，很难融合至page-based数据库中

## 构建云上HTAP的需求

1. 在避免重复储存数据的情况下，设计对于AP和TP负载都有优势的数据格式
2. 利用好存储技术（SSD，对象存储）
3. 日志瓶颈
4. 融合共享存储的设计（log server / page server）

## 本文的设计总览

![Figure 3: Colibri, a cloud-native HTAP architecture, combines the benefts of OLAP and OLTP systems.](f3.png)

1. 分离冷热数据：
   - 热数据以行存的形式与索引一同放在page server
   - 冷数据以列存的形式压缩存放在对象存储中
2. 利用存储设备的高带宽：所有节点都应该能够直接访问对象存储
3. 减少日志条数：主节点能够减少日志传输，log server储存更少的数据，page server更少回放
4. 对于大块的操作（bulk operations）应该能够旁路log server和page server：对于加载大规模数据要能够直接写入对象存储，只需要少量修改元数据

- 使用log server和page server来避免写入对象存储或者其他数据库的高延迟
- 使用传统buffer manager，以及page-based索引

## 混合存储引擎

### 行列混合储存

![Figure 4: Hybrid column-row store. A B+-tree indexes compressed and uncompressed blocks.](f4.png)

使用B+树做索引，冷数据通过压缩以列存形式存在数据块文件(data block file)中，热数据以行存的形式存在数据页(data page)中。如果冷数据需要修改，会先解压以热数据形式存放。图4中各个模块如下：

1. 内部节点(64KB)：内有4093个孩子
2. 叶节点(64KB)：内有51块
3. 键(绿色小块)：RowId（应该是8B）
4. 压缩块(1272B)(白色小块)：存放了一个指向数据块文件的ref，以及一些统计信息(比如minmax-bounds)
5. 数据块文件(大约16MB)：内部能存约262,144条记录
6. 非压缩块(1272B)(白色小块)：存放了(最多78个)指向数据页的ref
7. 数据页(64KB)：内部能存约380条记录（以PAX格式）

其中，写数据块文件不需要额外的wal。

### 插入

插入只会插入到最右端的叶节点。

- 对于少量插入，直接写入最右端的叶节点，等插满后会将非压缩快转换成压缩块
- 对于大批量的插入，直接生成列存压缩的数据块文件，最后更新B+树。这样只需要记录一条日志，并且减少了复制脏页等开销

### 删除

- 对于非压缩块：标记删除
- 对于压缩块，会在page-server中额外存放一个bitset页

- 对于批量删除，会将要删除的RowId排序，同一数据块的删除会一起执行，只需要写一条日志
- 会进行异步维护数据块，如果被压缩的数据块中有超过1/4被删除了，会重写该数据块

### 更新

- 对于非压缩块：in-place更新
- 对于压缩块：标记删除，并按常规逻辑插入，即out-of-place更新

### 维护

1. 合并过小的b+树节点
2. 将热数据块转换成冷数据块格式
3. 将标记删除的行物理去除
4. 将空文件从对象存储中删除

短维护(<50us)会在访问B+树时一并进行；长维护会在背景异步执行。

冷热数据的界限，本文认为是10s没有修改。(如下表，对于buffer size为8~16G的情况下，冷热界限从10s到30s造成的性能损失最大)

![Table 2: Query performance for varying hot-cold thresholds and buffer sizes (TPC-H SF100, queries per second).](t2.png)

### 集成现代缓存管理器

Umbra(本文使用的codebase，该组以往的工作)实现了针对事务工作负载优化的先进缓存管理器。
对于列存，由于需要解压，需使用可变大小的页面([前期工作](https://db.in.tum.de/~freitag/papers/p29-neumann-cidr20.pdf))，并且由于列存是不可变的，不需要写回。

### MVCC

![HyPer的MVCC](ext_f1.png)

使用了HyPer的MVCC实现，版本链处于内存中。

### 恢复

加载6Billion行数据，在写入检查点之前crash掉

|         | Load time | WAL size | #Log entries | Recovery |
| ------- | --------- | -------- | ------------ | -------- |
| Umbra   | 3 099 s   | 1.28 TB  | 6 166 M      | 5 352 s  |
| Colibri | 453 s     | 1.02 GB  | 5.17 M       | 2.99 s   |

### 列存数据格式

列存格式选择Data Blocks：

1. 对于点查，能够快速解压出单个元组
2. 对于带过滤的查询，有较好的性能

|                  | Full Table Scan [row/s] | Filtered Scan [row/s] | Point Lookups [row/s] | Size        |
| ---------------- | ----------------------- | --------------------- | --------------------- | ----------- |
| PAX (uncompr.)   | **68.4 M**              | 98.2 M                | 1.01 M                | 88.7 GB     |
| Arrow            | 34.1 M                  | 16.8 M                | 183                   | 96.8 GB     |
| Data Blocks      | 31.5 M                  | **99.4 M**            | 1.18 M                | 34.1 GB     |
| Parquet          | 19.0 M                  | 7.29 M                | 758                   | 34.5 GB     |
| Parquet (Snappy) | 11.2 M                  | 5.82 M                | 752                   | **23.9 GB** |

### 价格优化

对象存储为每次请求付费。S3推荐每次请求16MB数据，这样最便宜，且能够有效利用带宽。

对于数据块中的262144行记录，每列差不多0.25~0.5MB数据。因此需要将多个数据块合在一起成为一个segement。

### 二级索引

针对TP负载，可以选择构建二级索引，存在数据页中，所有改动都需要先写日志。

针对AP负载，可以选择SMA（为每一小块记录最大值最小值等聚合结果）或者缓存来过滤数据。

## 查询处理
