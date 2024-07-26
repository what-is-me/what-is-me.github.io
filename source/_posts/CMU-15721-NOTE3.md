---
title: CMU15-721 Note3 | Data Formats & Encoding II
date: 2024-07-22 17:57:30
tags: 
    - DataBase
    - CMU15-721
    - Learning Note
category: "CMU15-721"
---
许多现存的格式没有针对需要远程访问的Datalake、现代高吞吐网络以及数据并行化的CPU进行优化。

## BtrBlocks

- PAX存储模型
- 比Parquet/ORC更激进的嵌套结构编码方案
- 使用基于抽样的贪心算法选择最佳编码方案，并尝试在输出上再次编码
  - 不使用单纯的块压缩(Snappy, zstd)
- 将文件的元数据和数据分开存放

### 编码方案的选择

- 抽样法：
  - 从数据中抽样并尝试所有可用的编码方案，重复三遍
  - 使用多个不重叠小数据块，而不是单个的值(10\*64而非640\*1)
    - 对于64000个值，使用10个大小为64的块
- 使用了如下编码方案：
  - RLE / One Value
    - Run-length encoding (RLE)：${A, A, A, A, A}\rarr (A, 5)$
  - Frequency Encoding：依据频率进行的编码(比如哈夫曼树)
  - FOR + Bitpacking
    - Frame of Reference：${105, 103, 113}\rarr (100, {5, 3, 13})$
    - Bitpacking：使用4bit储存{5, 3, 13}而不是8bit
  - Dictionary Encoding：$"ABCABCZXZXZX"\rarr ({0:"ABC",1:"ZX"}, {0,0,1,1,1})$
  - Fast Static Symbol Table (FSST)
    - 用于压缩字符串
    - 寻找“最佳”符号表，将出现频率最高的子串(不超过8B长)替换为1B的代码
    - 快速压缩(SIMD并行加速)
    - 压缩后的文件可以随机读
    - 寻找符号表：一种自底向上生成符号表的启发式算法
      - 每轮迭代：
        1. 使用当前符号表计算压缩因数，同时计算出每个符号出现次数以及哪些符号是连续出现的
        2. 选择(出现次数\*长度)最大的那些符号，以及连续出现的符号组成的新符号作为新符号表
      - 尽量使用长的出现频率高的符号
  - Roaring Bitmaps for NULLs and Exceptions
    - 在不同数据库中使用广泛
    - 每$2^{16}$数据为一个数据块，值$k$储存在第$\frac{k}{2^{16}}$块
    - 数据量小于4096的数据块使用数组储存数据
    - 数据量大于4096的数据块使用bitmap储存数据
  - Pseudodecimals：压缩小数$double(3.25)\rarr (int(+325), int(2))$

### 评论

- BtrBlocks、Parquet、ORC都生成变长数据块，对向量化不友好，解码时浪费性能
- Parquet、ORC使用Delta Encoding，使得每个元组的值依赖前面的元组，导致无法使用SIMD

## FastLanes

- 专门针对SIMD优化的一系列编码方案。
- 作者设计了一个拥有1024bit SIMD寄存器的虚拟ISA，以期适用于现在和未来所有寄存器带宽
- 重排序使得delta decoding能向量化

![ULT](ult.png)

## BitWeaving

- 背景
  - 例如整数大小的比较等操作并不一定需要全部32bit乃至64bit的数据
  - 可以将int64看作bit[64](Bit-sliced encoding)
    - 该方案可以加速某些聚集函数：例如使用POPCNT加速加法$\sum_{i=0}^{n} int(x_i) = \sum_{j=0}^{32}(\sum_{i=0}^{n} x_i >> j) << j = \sum_{j=0}^{32} POPCNT_j(x)<<j$
- 支持使用SIMD对压缩数据进行高效的谓词评估
  - 支持有序字典编码($encode(s_1)<encode(s_2) \lrArr s_1<s_2$)
  - bit级并行化
  - 只需要通用指令(不依赖SIMD硬件级别的实现)
- 实现方式：
  - Horizontal：
    - bit级别的行存
    - (需要使用位运算来代替比较)计算$X<Y$($X=1$,$Y=5$)：
      1. X和Y前面要补1bit的0作为结果位:$X=0001$, $Y=0101$
      2. $mask=0111$
      3. $Z=(Y+(X\oplus mask))\wedge \neg mask=1000$ 结果位非零，真
      4. $X=0001,0010,0011,0100; Y=0101,0101,0101,0101; mask=0111,0111,0111,0111$ 并行计算
  - Vertical：
    - bit级别的列存
    - SIMD优化的Bit-sliced
    - 比较时从高位向低位迭代，若结果全为非(使用POPCNT)，则可以结束比较并跳过
