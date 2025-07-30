---
mathjax: true
title: 泛读笔记：《How to Grow an LSM-tree?》
tags:
  - Reading Note
  - DataBase
  - LSM Tree
category: 泛读笔记
date: 2025-07-30 10:53:02
---
[Paper](https://dl.acm.org/doi/pdf/10.1145/3725310)

## 摘要

基于LSM-tree（Log-Structured Merge Tree）的键值存储已被广泛采用，作为现代大数据应用中的数据存储后端。LSM-tree 随着数据的摄入而增长，这种增长方式主要有两种：一种是通过增加固定容量的层级来扩展（称为Vertical方案），另一种是通过在固定层级数量下扩展每层的容量（称为Horizontal方案）。在当前的系统设计中，如 RocksDB、LevelDB 和 WiredTiger，Vertical方案逐渐成为主流；而Horizontal方案在工业界的采用则呈下降趋势。

增长策略在多个方面（如读取、写入以及空间成本）都会深刻影响LSM系统的性能。本文试图从一个基本设计问题出发，提供新的见解：**如何扩展LSM-tree以获得更理想的性能？**我们的分析揭示了Vertical方案在实现最佳读写权衡方面的局限性，以及Horizontal方案在空间成本管理方面的不足。

在此基础上，本文提出了一种新颖的策略：Vertiorizon，它结合了Vertical和Horizontal两种方案的优势，在查找、更新和空间成本之间实现了更优的平衡。其自适应设计使其能够兼容各种类型的工作负载。与Vertical方案相比，Vertiorizon 显著提升了读写性能之间的权衡效率；而与Horizontal方案相比，Vertiorizon 通过对 Bentley 和 Saxe 理论的非平凡推广，极大地扩展了权衡范围，同时大幅降低了空间开销。

在与 RocksDB 集成时，Vertiorizon 显示出比Vertical方案更好的写入性能，并且其额外空间成本仅为Horizontal方案的六分之一。

<!--more-->

## 背景

![Horizontal和Vertical增长方案](image-20250730111644496.png)

![examples](image-20250730112803339.png)

**问题**：不完善的LSM树增长方案损害系统性能：

- 增长方案决定了合并的时间
- 不当的合并时机可能导致更高的写放大和读放大
- Vertical方案（由RocksDB和LevelDB使用）在平衡读写权衡方面并非最优，因为它在每一层以固定频率执行合并，随着数据规模扩大，会降低系统吞吐量
- Horizontal方案（由BigTable采用）仅通过leveling合并策略实现，难以处理更新密集型工作负载（现代系统中的常见负载类型），还需要在合并过程中进行完全合并（full compaction），导致高空间开销（这里应该指写放大）

**Key Intuition 1.** Horizontal + Tiering 的方案空缺，值得研究

**Key Intuition 2.** Vertical和Horizontal能否结合起来达到更好的效果

**贡献**：

1. 仔细审查了LSM树两种主要增长方案的优缺点，并激发了设计更优增长方案的基本问题
2. **提出了Horizontal-Tiering 方案。还从理论上证明，所设计的Horizontal分层方案在压缩过程中实现了最小累积成本**
3. **提出了VERTIORIZON，一种创新的增长方案，融合了Vertical和Horizontal方案的优势，并具备良好的自适应特性，能根据不同的工作负载动态调整自身行为**
4. 设计了一种新颖算法，使我们的设计能够适应更具偏斜性的工作负载。同时伴随深入分析以验证此类设计背后的合理性
5. 展示了VERTIORIZON如何通过引入一种专为我们的设计定制的新颖动态布隆过滤器布局，嵌入到成熟的LSM系统中
6. 真实系统环境中对VERTIORIZON进行了广泛的实验评估，结果表明，VERTIORIZON的吞吐量最高可达Vertical方案（RocksDB采用）的3.2倍，且相比Horizontal方案额外的空间放大程度减少了约6倍

### 增长方案

![vertical and horizontal growth schemes](image-20250730135845296.png)

Horizontal中的C指合并次数，当$C_i > C_{i+1}$就会触发向下一层的合并

![合并频次](image-20250730140151531.png)

Horizontal方案实现了更优的读写权衡。图中相同数据量下，Horizontal的写入量更小。对于给定层级数量，Horizontal方案产生的写入成本最小。由于在分层合并策略下LSM树的读取成本与层级数成正比，因此Horizontal方案始终能在读取和写入成本之间实现最优权衡。

但是，大多数键值存储系统倾向于采用Vertical方案，可能因为：

1. Horizontal方案处理更新密集型工作负载能力有限。Vertical方案通过切换到Tiering策略可以有效适应这种负载，Horizontal方案目前还没有结合Tiering策略的方案。->贡献2
2. 理论与实践之间有差距。Horizontal方案基于完整的压缩粒度（full compaction granularity），而文件级的部分压缩粒度（file-level partial compaction granularity）是大多数工业系统的标准，并已在某些实证系统性能方面展现出实际优势，包括显著降低的空间放大（space amplification）和改善最坏情况下的吞吐量（throughput）。单次完整压缩会一次性将某一层的所有内容合并到下一层。当参与压缩的层级数据量较大时，此过程会导致显著的空间放大，并引发写入阻塞。相反，每次部分压缩仅合并来自某一层的一个文件与下一层中少量重叠文件，从而只影响一小部分数据，大幅缓解这些问题。->贡献3

### Horizontal-Tiering方案

在Tiering策略下，从长期看无论何时将某一层的数据压缩到下一层，写放大都是相同的，因为每个条目在通过每一层时仅经历一次写入放大。因此，优化整体性能的关键在于选择最佳时机以最小化读放大。

![examples](image-20250730143504149.png)

图4a. 一种等频率合并策略，即每插入20MB数据就发生一次从Level 1到Level 2的合并

图4b. 递增频率的合并策略，分别在摄入30MB，20MB和10MB的数据后发生合并

图4c. 插入60MB之后连续3次合并（极端情况，后续不讨论）

基于以上观察，在某层较空闲时，应该减少向其合并的频率。

![算法](image-20250730165318805.png)

B指flush下来的sstable的大小。在$\binom{k+l-1}{l}$次flush之后，$C_i$都变成0。N根据存储设备的容量设置，或者根据估算。

![example](image-20250730170027757.png)

最优性证明略。

### VERTIORIZON

LSM上部使用Horizontal，下部使用Vertical管理最大的两个层级。

Horizontal容量为n，最后两层容量是n的$T$和$T^2$倍，T是一个可调参数。

Horizontal每次清空后将n增加$1/T$倍来缓慢增长，从而动态扩容。

Vertical使用部分合并策略来避免过度的空间放大和写停滞，相比全程使用完整合并的LSM树，可以大致将空间放大和写停滞减少T倍。

![overview](image-20250730172954723.png)

**通过调整大小比例优化写放大**：可以通过调整最后两层的容量比来优化写放大。由于Vertical部分采用部分压缩，其第一层的数据量始终接近其容量，因此从Horizontal部分到Vertical部分第一层的压缩始终会产生写放大因子T，因为该层的数据量被维持在Horizontal部分容量的T倍。从Vertical部分第一层到第二层的压缩所导致的平均写放大因子仅为$\frac{T+1}{2}$，因为部分压缩会导致该层内数据密度不均匀。假设将Horizontal部分到Vertical部分第一层的大小比例设置为$T'$，Vertical部分两层之间的大小比例设置成$T^2/T'$，这两部分写放大为：
$$
T'+(\frac{T^2}{T'}+1)/2\geq 2\sqrt{T'\cdot \frac{T^2}{2T'}}+\frac{1}{2}=\sqrt{2}T+\frac{1}{2}
$$
$T'=T/\sqrt{2}$时取等号

### 其他优化

- 针对不同工作负载的自调优
- 偏斜工作负载下的优化
- 引入新的布隆过滤器布局
- ...
