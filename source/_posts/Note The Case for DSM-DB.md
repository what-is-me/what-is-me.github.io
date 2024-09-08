---
title: 泛读笔记：《The Case for Distributed Shared-Memory Databases with RDMA-Enabled Memory Disaggregation》
date: 2024-08-13 13:12:27
tags: 
    - Reading Note
    - DataBase
    - Hardware
    - RDMA
category: "泛读笔记"
---
[Paper](https://www.vldb.org/pvldb/vol16/p15-wang.pdf)

## 摘要

内存解耦（Memory Disaggregation，简称MD）通过将计算（CPU）与内存分离，允许数据中心设计实现可扩展性和弹性。在MD架构下，计算和内存不再被耦合到同一个服务器中，而是通过诸如RDMA（远程直接内存访问）等超高速网络相互连接。MD带来了许多优势，例如更高的内存利用率、更好的独立扩展（计算和内存的独立扩展）以及更低的拥有成本。分布式共享内存数据库（DSM-DB）潜力巨大。本文列出了构建分布式共享内存数据库将会遇到的挑战。

## 内存解耦的优势

1. 通过内存池化提高了内存利用率，减少内存碎片化
2. 提供了计算和内存的独立弹性扩展，利于云计算
3. 更高可靠性和更低运营成本，因为计算和内存的故障和升级都是独立的
4. 为用户提供了近乎无限的内存池

![](image0.png)

## 分布式共享内存数据库的优势（相比于shared-nothing架构）

1. **独立弹性**：内存解耦，计算和内存有独立弹性扩展能力
2. **降低总成本**：内存利用率增加，总成本降低
3. **高可用性**：支持计算和内存节点的独立故障，从而实现高可用性
4. **鲁棒性**：数据可以轻松重新分片，相比于shared-nothing，运行某些查询和面对数据偏斜时更稳健
5. **可扩展性**：可以实现多主节点的更好可扩展性，这是现有的分布式共享存储数据库所不具备的

## 实现分布式共享内存数据库面临的挑战

### 分布式共享内存

#### 挑战1 分布式共享内存部分的接口设计

1. 内存分配接口：
    - 需要提供类似单节点的内存管理接口，如分配、释放、重新分配
    - 内存地址的表示：应当是逻辑地址而非物理地址
    - 为了减少内存碎片，可以分配大块空间并追踪用户空间的使用[CoRM](http://eggmqtz.unixer.de/publications/img/corm-taranov.pdf)
2. 数据传输接口：提供单边双边RDMA、内存读写、原子操作的接口
3. 操作下推接口：数据库可以把部分操作下推到内存结点以减少数据传输[TELEPORT](https://www.cis.upenn.edu/~sga001/papers/teleport-sigmod22.pdf)

#### 挑战2 持久性

**日志**的储存需要高性能低成本。

- 方法1 向持久储存中写入日志
  - 可以考虑云存储，成本低廉
  - 可以采用内存数据库相似方案进行优化
- 方法2 内存复制[RAMCloud](https://web.stanford.edu/~ouster/cgi-bin/papers/ramcloud.pdf)
  - 日志被同步写入不同节点，如果一条日志被写入所有节点，即认为它已经被持久化了
  - 性能较好
  - 如果所有节点都崩溃了，数据会丢失，需要额外的解决方案，如battery-backed memory、PM、SSD

#### 挑战3 可用性

内存是易失性的，DSM-DB 在崩溃时可能变得不可用。目标是实现合理的可用性，以最小化停机时间，同时保持低成本。

- 方法1 数据复制到多个节点
  - 耗费内存大，成本高
- 方法2 使用纠缠码[Hydra](https://www.usenix.org/system/files/fast22-lee.pdf)
  - 恢复时间长
- 方法3 向持久设备定时写入checkpoint，恢复时回放部分日志[RAMCloud](https://web.stanford.edu/~ouster/cgi-bin/papers/ramcloud.pdf)，但RAMCloud提供kv接口，无法将操作下推，且使用TCP/IP而不是RDMA

### 并发控制

#### 挑战4 缓存一致性

单机上缓存一致性有硬件层面的保证，但在DSM上没有。
![](image1.png)

- 方法1 无缓存，无分片 [1](https://alchem.usc.edu/portal/static/download/rdma_tx.pdf) [2](https://arxiv.org/pdf/2112.07320)：
  - 先获取锁再访问
  - 没有缓存一致性问题
  - 存在性能开销
- 方法2 有缓存，无分片：
  - 计算节点利用本地内存进行读写操作
  - 需要一个软件级的缓存一致性协议来广播计算节点所做的更改[1](https://www.vldb.org/pvldb/vol11/p1604-cai.pdf)[2](https://users.cs.utah.edu/~lifeifei/papers/polardbserverless-sigmod21.pdf)，性能受软件实现细节多方面影响
  - 懒惰的缓存一致性协议可以用互斥一致性换取性能
- 方法3 有缓存，有分片：
  - 执行逻辑分片，每个计算节点维护它负责的数据的分片信息
  - 并发控制与分布式无共享数据库中的情况类似
  - 计算节点不用存储整个分片，能够最佳利用本地缓存
  - 能很好的支持弹性：如果添加了新的计算节点，只需将元数据复制到新节点，而无需立即物理移动数据（由于逻辑分片），过时的数据可以在旧计算节点中异步回收
  - 对跨分片事务不利，通过动态分片缓解

#### 挑战5 分布式提交

- 使用无分片缓存或者无缓存则不需要考虑分布式提交（例如2PC）
- 使用分片方案一般需要考虑分布式提交，因为每个计算节点只负责一个数据页，事务可能是跨分片事务，访问多个计算节点，需要考虑分布式提交
- 使用分片方案可能不需要考虑分布式提交，因为如果使用单边RDMA来访问内存节点，它知道是否成功写入（？）

#### 挑战6 并发控制协议

- 基于锁的并发控制：
  - 在RDMA上实现2PL中不同类型的锁（共享、独占锁和意图锁）具有挑战性
  - RDMA在单次往返中只能通过CAS原语实现一个简单独占自旋锁，高级锁需要更多RDMA往返次数。通过RDMA上锁开销较大
- 非基于锁的并发控制：
  - 需要使用锁来独占访问某些共享状态（比如全局时间戳）
  - 生成时间戳时单边操作（RDMA Fetch & Add）比双边操作性能更好
  - 对于时间戳，可以考虑选用其他方案，比如向量时钟和时钟同步

#### 挑战7 支持大规模并发

DSM-DB可以支持更多的cpu核心（更多的计算节点），需要考虑众核（比如，百万核）情况下不同并发控制协议的性能，需要考虑全局并发控制和局部并发控制

### Buffer管理

#### 挑战8 设计轻量级缓存管理

- 现有的缓存管理是针对存在巨大的缓存命中和未命中性能差距的层次结构进行优化的，主内存和磁盘之间的延迟差距可以达到100,000倍，因此更关注缓存命中率。而本地和远程内存之间的性能差距已经大大缩小，需要关注实际运行时间，而不仅仅是缓存命中率。需要关注查找成本、由于多线程访问带来的同步成本等
- 与本地PM访问不同，如果未命中，cpu可以直接操作PM中的数据；而在DSM中，需要数据先传输到本地缓存再操作
- 缓存页的压缩：解压缩的开销可能大于直接从远程获取数据，因此轻量的压缩很重要

#### 挑战9 Caching vs. Offloading

Caching 和 offloading(pushdown) 在分离式数据库中是两种常见的提升性能的方法，两者并不互斥。

- RDMA 的影响：因为计算节点具有更强的计算能力，RDMA延迟较小，将数据拉到计算节点缓存并计算更有利
- 软件开销：需要轻量模型来分析决策何时需要将数据缓存或将计算下推
- 数据一致性问题：数据缓存和将计算下推会造成数据不一致

### 索引设计

#### 挑战10 RDMA 友好的索引设计

- 选择 RDMA 原语：例如，使用单边 RDMA 还是双边 RDMA、同步 RDMA 还是异步 RDMA
- 如何在每个计算节点中最好地利用有限的缓存内存
- 如何减少软件开销以充分利用高性能 RDMA
- 如何利用内存节点的计算能力来减少数据传输

以上这些决策并不独立。例如，拥有更大的本地内存倾向于使用单边 RDMA，以减少往返次数。同时，近数据计算会影响缓存的使用方式。另一个方向是设计能够自适应平衡可用计算、内存和 RDMA 资源的索引，以防止资源利用不均，例如，计算节点的 CPU 完全饱和，而内存节点的 CPU 或 RDMA 带宽大部分闲置。

#### 挑战11 并发索引操作

- RDMA 锁的开销可能非常高，这会影响并发性能
- 没有硬件支持的缓存一致性可能会导致额外的复杂性和性能问题
- 基于锁的方法（比如ART），不清楚它应该用什么种类的锁
- 基于LSM的索引可能更值得研究