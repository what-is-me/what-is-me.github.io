---
title: 泛读笔记：《Towards Optimal Transaction Scheduling》
date: 2024-09-03 14:00:27
tags: 
    - Reading Note
    - DataBase
    - Transaction Scheduling
category: "泛读笔记"
mathjax: true
---
[Paper](https://www.vldb.org/pvldb/vol17/p2694-cheng.pdf)

## 摘要

> Maximizing transaction throughput is key to high-performance database systems, which focus on minimizing data access conflicts to improve performance. However, finding efficient schedules that reduce conflicts remains an open problem. For efficiency, previous scheduling techniques consider only a small subset of possible schedules. In this work, we propose systematically exploring the entire schedule space, proactively identifying efficient schedules, and executing them precisely during execution to improve throughput. We introduce a greedy scheduling policy, SMF, that efficiently finds fast schedules and outperforms state-of-the-art search techniques. To realize the benefits of these schedules in practice, we develop a schedule-first concurrency control protocol, MVSchedO, that enforces fine-grained operation orders. We implement both in our system R-SMF, a modified version of RocksDB, to achieve up to a 3.9× increase in throughput and 3.2× reduction in tail latency on a range of benchmarks and real-world workloads.

最大化事务吞吐量是高性能数据库系统的关键，高性能数据库系统关注于最小化数据访问冲突以提高性能。然而，找到有效的时间表，减少冲突仍然是一个悬而未决的问题。为了提高效率，以前的调度技术只考虑可能的时间表的一个小子集。在这项工作中，我们建议系统地探索整个时间表空间，主动识别有效的时间表，并在执行过程中精确地执行它们，以提高吞吐量。我们引入了一个贪婪的调度策略，SMF，有效地找到快速的时间表，并优于最先进的搜索技术。为了实现这些时间表在实践中的好处，我们开发了一个时间表优先的并发控制协议，MVSchedO，执行细粒度的操作顺序。我们在我们的系统R-SMF（RocksDB的修改版本）中实现了这两个功能，在一系列基准测试和实际工作负载上实现了高达3.9倍的吞吐量增加和3.2倍的尾延迟减少。

## 背景

- 寻找最优调度是NP难的
- 大多数现有并发控制协议(2PL、MVCC)：
  1. 基于先入先服务的顺序
  2. 在冲突到达或在执行结束时处理冲突
- 最近的工作(确定性数据库):
  1. 通过利用关于访问集的完整信息调度事务，或者预测将要访问的键
  2. 仅考虑了调度空间的小子集
- 距离最优化差距较大

### 挑战1 如何高效通过部分信息找到快速调度方案

1. **事务间冲突成本**不同造成了调度之间执行时间的差异
2. 寻找最优化调度方案是不现实的
3. 大多数工作负载中存在的一小部分热键对执行时间有巨大的影响

因此，本文提出了一种贪心算法**最短最大跨度优先**(SMF)：

1. SMF通过追加导致执行时间增量最小的事务来迭代地构造调度
2. SMF根据每个事务与调度中所有其他请求的冲突程度做出决策，而不是只考虑单个事务的特征
3. SMF**不**依赖于完整读写访问集的先验知识
4. SMF会使用应用程序中包含的元数据
5. SMF开销小，实现了线性复杂度(对于正在运行的事务而言)

### 挑战2 如何在不违反隔离级别的情况下强制执行调度

1. 现有并发控制协议不主动控制单个操作完成顺序
2. 强制执行调度的系统会产生很高的开销：
   - 要么在单个线程中序列化所有请求
   - 要么要求开发人员手动构建自定义集合点来协调依赖关系

因此，本文开发了一种新的并发控制协议MVSchedO：

1. 采用多版本时间戳排序
2. 利用了预期的热键访问
3. 对于各个热键维护一个调度队列

### 效果

相比于基线(RocksDB)提升至3.9倍吞吐，有3.2倍尾延迟降低

## 引入定义：Schedule Makespan

Makespan，即所有事务完成所需要的总时间

对于给定的调度，事务执行受到以下约束：

1. 事务内部的操作顺序
2. 事务之间的依赖性，由在运行时期间强制执行指定隔离级别的并发控制协议确定

![Figure 1: Two schedules of the same workload under MVTSO.](F1.png)

不同的调度方案会导致makespan不同，差距可能非常显著。

表1: FIFO 相比于已知最优调度所增加的 Makespan

|Epinions|SmallBank|TAOBench|TPC-C|YCSB|
|:--:|:--:|:--:|:--:|:--:|
|43.2%|8.0%|96.2%|101.6%|99.3%|

## SMF贪心算法

- 流程：
  - 首先，SMF随机选取一个事务加入调度
  - 在每次迭代中，随机抽取k个事务，将增加Makespan最小的事务加入调度
- 时间复杂度为$O(n\times k)$，实验证明，较小的k就足够了(e.g. k=5)

- 实验证明1（研究TPC-C）：
  - New-Order和Payment这两种事务主要在Warehouse和District的键上冲突
    - 实验：研究分析了500个New-Order和Payment事务(10仓)
    - SMF能够使访问不同Warehouse和District键的事务并行执行，实验中FIFO调度的makespan是SMF的1.7倍
  - 由于事务性工作负载倾向于在一小部分键上发生冲突，这些键的影响较大
    - 实验：运行了一个仅知道Warehouse和District表中键的SMF版本，和一个事先知道所有键访问情况的版本，产生的调度计划性能几乎相同
- 反例，出现概率应当较小：

![Figure 2: SMF can make suboptimal scheduling decisions
under an adversarial workload with few conflict patterns.](F2.png)

- 实验证明2：
  - 通过将SMF生成的结果和对所有调度的均匀采样的结果进行比较，SMF的结果比大多数调度的makespan都小
    - 一个调度相当于是一个有向无环图，各个调度的底（无向边）相同
    - 使用现有的Interval-Reversal (IR) Random Walk方法对调度进行均匀抽样
    - 实验证明，在均匀抽样数为299个时，SMF有95%的置信度优于99%的调度
    - 在实验中评估了100k个均匀抽样的样本，SMF有99.99%的置信度优于99.99%的样本，SMF最好的结果落在了99.999百分位上
  - 这种均匀抽样的方法被用于在运行时和SMF对比，如果SMF生成的结果有多次较差，可以切回FIFO

## 系统构建

系统组件：

1. 预测热键冲突模式的分类器
2. SMF调度器，确定事务如何排序
3. 调度优先并发控制协议MVSchedO，强制执行细粒度操作执行

### 在线调度

- SMF需要知道事务中的热键
- 离线状态下，所有热键都是已知的
- 在线状态下，热键未知，需要额外hint来预测
  - 每个事务有额外的hint，构成了MetadataVec，一个整数向量：
    1. 事务开始时程序向数据库提供事务种类
    2. 事务开始时从sql语句、clint元信息等中获取的一些元信息
  - 对过往事务训练和验证knn，并分类，每类中出现频率最高的操作就是该类的热键操作
  - 对新开始的事务，通过其MetadataVec得到其在knn的某一类，该类的热键操作集合即预测该事务将进行的热键操作
  - 在线调度只需要考虑将要加入的事务对正在运行中的事务的影响

### MVSchedO

1. 每个事务开始时
   1. 预测该事务将访问热键的操作`pred_hot_key_ops`
   2. 使用SMF分配时间戳`ts`(而不是FIFO分配时间戳)
   3. 对于每一个热键`k`，全局维护的时间戳数组`pred_ops[k]`中会插入`ts`，这相当于维护了一个操作队列
2. 每个对热键操作时：
   1. 等待`min(pred_ops[k]) >= ts`，即，SMF分配在该事务前的其他事务对该热键的访问已经结束
   2. 操作结束后依据访问类型将`ts`从`pred_ops[k]`中去除
3. 正确性：由于MVTSO中的时间戳分配是任意的，因此SMF选择的调度不会影响可串行化保证
