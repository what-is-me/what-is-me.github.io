---
title: CMU15-721 Note2 | Data Formats & Encoding I
date: 2024-07-17 18:19:04
tags: 
    - DataBase
    - CMU15-721
    - Learning Note
category: "CMU15-721"
---
## AP和TP工作负载

- OLAP：大段的只读的数据上的**顺序扫描**
  - 只要通过顺序扫描查找到目标元组
- OLTP：使用索引查找元组而不是顺序扫描
  - 针对low selectivity predicates (where, HAVING, ...)创建索引
  - 同样需要容纳新增的更新

## 顺序扫描的优化方案

- 数据编码/压缩
- 预取
- 并行化
- 聚集/排序
- 推迟元组拼接(Late Materialization)
- 物化视图/结果缓存
- 数据跳过(Data Skipping)
- 数据向量化(比如SIMD)
- 代码专业化/编译(比如JIT)

## 存储模型

存储模型指数据库在物理上如何在磁盘和内存中组织元组。

### N-ary Storage Model (NSM)

即行存：

- 单个元组的几乎所有属性连续存放在单个文件页中(除非有超过文件页大小的属性)
- 对以下OLTP负载较为理想：
  - 访问个别实体的事务
  - 插入查询较多的事务
- 倾向使用迭代器模型(火山模型)
- 一个文件页的大小通常是4KB的倍数(Oracle 4 KB, Postgres 8 KB, MySQL 16 KB)

### Decomposition Storage Model (DSM)

即列存：

- 在一个文件中连续存储所有元组的某一个属性
- 对以下OLAP负载较为理想：
  - 很大的只读查询，但只使用了表的某几列
- 倾向于使用向量化模型(批处理模型)
- 文件块的大小更大(几百MB)，但也可能仍然使用更小的文件块来组织元组
- 可能需要将变长数据转换成定长数据来储存(因此可以使用偏移量来访问一个属性)
- 页头会储存元信息(比如null-bitmap)
- 元组ID两种方案
  1. 定长偏移
     - 所有值都是定长，根据ID查询元组方便
     - 变长数据要转换成定长数据(Dictionary Compression可以一定程度上压缩定长数据)
  2. 嵌入元组ID(Bad Idea)
     - 某列的值会和ID存在一起
     - 根据ID查询元组需要额外的数据结构辅助

### Hybrid Storage Model (PAX)

![PAX](pax.png)

- 优势：
  - 列存结构：快速处理
  - 元组各属性距离近

### 开源持久数据格式

- 大多数数据库使用专有二进制格式来存储持久数据，在各系统间传输数据需要将数据转换成基于文本的格式(CSV, JSON, XML)
- 开源二进制文件格式使得跨系统读取数据文件更加容易和高效

## 数据文件格式设计要点

### 文件元数据

- 文件需要是自包含的。这可以增加其可移植性。应当包含所有必要的用来解释其内容的信息，而不需要额外数据依赖。
- 每个文件维护全局元数据(通常在尾部)，比如：表架构、行组偏移量、元组总数等

### 数据格式布局

- 大部分格式使用PAX存储模型
- 不同的实现，行组的大小不一定相同，会权衡计算和内存
  - Parquet: 元组数量 (e.g. 1 million)
  - ORC: 物理存储大小 (e.g. 250 MB)
  - Arrow: 元组数量 (e.g., 1024 *1024)

### 类型系统

- 类型系统决定了格式支持哪些类型：
  - 物理类型：低级字节表示(e.g. IEEE-754, 浮点数的表示)
  - 逻辑类型：映射到物理类型的辅助类型
- 类型系统的复杂性决定了上游生产者/消费者需要实现多少
  - Parquet：实现了很少物理类型，逻辑类型提供了描述原始数据的注解
  - ORC：提供了更完整的物理类型支持

### 编码方案

编码方案指定了格式如何存储相邻或相关的数据。可以在某种编码方案上再进行某种编码方案来提升压缩度。常用的有字典编码、RLE、Bitpacking等

#### 字典编码

- 使用很小的定长整数代码来代替经常出现的值，同时维护从代码向原始值的映射
  - 代码可以是原始值在字典中的序号(使用哈希表)或者在字典中的字节偏移量
  - 可以选择将字典中的值排序
  - 可以进一步压缩字典和被编码的列
- 当字典中不同值的数量过大时，必须有对应的处理方案
- 设计决策点：
  1. 适合使用字典编码的数据类型：
     - Parquet: 任何类型
     - ORC: 仅字符串
  2. 对已编码数据压缩方案：
     - Parquet: RLE + Bitpacking
     - ORC: RLE, Delta Encoding, Bitpacking, FOR
  3. 是否支持导出字典：
     - Parquet: 不支持
     - ORC: 不支持

### 文件块压缩

- 使用通用方式压缩数据
- 需要考虑：
  - 计算开销
  - 压缩与解压缩速度
  - 数据不透明性
- 因为对象存储便宜且空间巨大，该项优化越来越无效果

### 过滤器

- 区域映射(Zone Maps)
  - 在文件级别和行组级别维护列最大最小值
  - Parquet和ORC在行组头储存区域映射
- 布隆过滤器(Bloom Filters)
  - 记录行组中每列值是否存在，如果数值是聚集的会更有效
  - 布隆过滤器可以判断值是否一定不存在这块，但不能判断是否一定存在

### 嵌套数据

- 真实世界的数据经常包含半结构化数据(JSON, Protobufs等)
- 文件格式希望能像一般的列一样编码这些数据

#### Record Shredding

额外记录以下内容：

- Repetition Level：该值路径上，该值和上一个值的最近公共祖先节点的孩子节点的层数
- Definition Level：该值所在路径上，非空节点所在的最深层的层数
![Repetition Level](r.png)
![Definition Level](d.png)

#### Length+Presence Encoding

额外记录以下内容：

- Length：子节点个数
- Presence：记录是否存在

## 实验的分析结论与总结

- 字典编码是所有数据类型的可行选择，而不仅仅是字符串：现实世界中数据的重复性和相关性更高
- 简单的编码方案利于现代系统：运行时判断每块使用哪种编码方案增加分支预测错误
- 块压缩变得不必要：网络和磁盘的速度越来越快，而CPU正在成为瓶颈