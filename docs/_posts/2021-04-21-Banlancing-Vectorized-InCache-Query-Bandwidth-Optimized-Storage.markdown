---
layout: post
title:  "【论文笔记 1】平衡向量化查询引擎和面向带宽优化存储"
date:   2021-04-21 14:11:54 +0800
categories: MonetDB/X100 notes
---


# 【论文笔记 1】平衡向量化查询引擎和面向带宽优化存储

## 硬件发展趋势

CPU 发展趋势：

1. 新的CPU引入了以下技术
    1. 多核心
    2. SIMD 指令
    3. 超标量乱序指令执行
2. CPU频率的快速增长增加了内存延迟和处理器速度的不平衡
    1. 导致当前计算器越来越依赖多级缓存系统来改善内存访问时间

磁盘发展趋势：

1. 随机磁盘读写比顺序磁盘读写速度的增长速度慢的多
2. 两种磁盘读写和带宽速度的增长都显著的比现代多核CPU处理器处理能力慢的多

## 问题描述

查询执行：

- 传统查询基于 tuple-at-a-time 的流水线模型，类似于迭代处理N元组的模式，这样的模型引如了一些问题：
    - 这样让CPU耗费大部分时间在于遍历查询执行树，而不是在于真正进行数据处理
    - 这样的行为会带来比较差的指令缓存性能、高频的分支错误预测，会大幅降低应用性能
    - 更差的情况是，可能会让编译器在这种场景下无法使用很多常用的优化手段，比如循环展开、SIMD指令化
- 和科学计算的差别是，科学计算是一个真正的数据密集的方法，绝大部分都在数据处理。绝大部分问题都可以规约成一个简单场景，所以非常适合根据硬件优化性能。
- 类似MonetDB 系统与传统查询系统不一样，常常有以下技术特点：
    - 优势
        - 使用column-at-a-time的物化运算，就可以对于一个数组的值进行相对简单的运算
        - 使用这样的技术结果是，bulk processing / 批量处理，避免了对每个N元组进行循环解释执行的额外开销，并且为编译器优化提供条件
    - 劣势
        - 全物化的查询会带来非常庞大的中间结果，在大数据量处理的情况下，会带来额外的内存和磁盘传输开销
        - 一次性处理一个列，常常会使用大量内存，会阻碍使用一些可用的优化技术，如内存预取等
        - 列代数在多属性运算的时候较为难以实现，会引入更多的处理步骤

存储：

- 当前很多数据库系统没有充分使用硬件的属性
    - 顺序读写比随机读写的速度要快很多，但是很多的数据库系统仍然大量依赖随机读写，如未经聚合的索引等
    - 所以当前数据库系统会使用大量的硬盘来提升随机读写的吞吐量
- 即便使用了顺序读写，数据传输效率比起数据处理效率也依然低很多，是整个系统中的瓶颈
    - 存储模型：
        - 在N元组存储的模型中，这一点劣势尤其大，因为即便需要少量的列，也需要传输大量的属性
        - 列式存储模型能更好一些，对于磁盘带宽的要求更低
    - 两种存储模型都需要比较好的数据压缩技术，能够保障解压缩效率足够高，使得不会占用大量数据处理的时间
    - 同时多用户并发查询的场景对于存储系统也有非常大得挑战，在并发场景下系统经常会因为争抢资源而降级
        - 往往有潜在的可能，一些常见查询任务会给其他用户的查询任务带来性能提升

## 研究方向

研究方向可以总结为：

> 如何将不同的架构相关的优化技术结合起来构建一个数据库架构，同时在基于内存和基于磁盘的数据密集型场景充分发掘现代硬件的性能。

> How can various architecture-conscious optimization techniques be combined to construct a coherent database architecture that efficiently exploits the performance of modern hardware, for both in-memory and disk-based data-intensive problems?


1. 第一个方向是 vectorized in-cache execution ，即 MonetDB / X100 系统。
    - 该系统扩展了流水线模型，使运算基于向量（元组的集合）。每个向量包含几百或者几千的单个属性值
    - 这个执行分成两部分一种是通用的逻辑，一种是高度定制、高效的数据处理运算逻辑。这种设计让系统一方面通过批量处理获得高性能，另一方面也不会牺牲系统的扩展性。
2. 第二个方向是存储层的研究和设计，使用一些基于顺序扫描的技术提升数据传输效率，例如 ColumnBM 存储系统
    - 当达到了基于内存的100GB TPC-H 非常高的性能测试结果后，发现高性能的CPU处理能力，很难适配典型的磁盘系统。所以研究方向转向了存储层的研究
    - 这个技术会本文内两个领域方向进行评价
        - 数据仓库和决策支撑，如TPC-H 性能测试
        - 大规模信息检索，如Terabyte TREC 性能测试

## 大致研究成果和解决问题

### 问题一：是否有一个系统可以同时享受 tuple-at-a-time 和 批量计算带来的查询执行优势？

向量化缓存内执行模型。 Vectorized in-cache execution model

参考文献：

1. Peter Boncz, Marcin Zukowski, and Niels Nes. MonetDB/X100: Hyper-Pipelining Query Execution. In Proc. CIDR, 2005.
2. Marcin Zukowski, Sandor Heman, and Peter Boncz. Architecture- Conscious Hashing. In Proc. SIGMOD DaMoN Workshop, Chicago, IL, USA, 2006.
3. Marcin Zukowski, Niels Nes, and Peter Boncz. DSM vs. NSM: CPU Performance Tradeoffs in Block-Oriented Query Processing. In Proc. SIGMOD DaMoN Workshop, 2008.


### 问题二：什么技术可以让数据库引擎尽可能依赖顺序扫描，而非随机I/O？


面向带宽优化的磁盘访问模型。Bandwidth-optimizing disk access model.

1. Peter Boncz, Marcin Zukowski, and Niels Nes. MonetDB/X100: Hyper-Pipelining Query Execution. In Proc. CIDR, 2005.
2. Marcin Zukowski. Improving I/O Bandwidth for Data-Intensive Applications. In Proc. BNCOD Doctoral Consortium, 2005.
3. Marcin Zukowski, Niels Nes, and Peter Boncz. DSM vs. NSM: CPU Performance Tradeoffs in Block-Oriented Query Processing. In Proc. SIGMOD DaMoN Workshop, 2008.


### 问题三：什么技术可以提高单个查询的数据库I/O 性能？

超轻量的数据压缩 Ultra-lightweight data compression

1. Marcin Zukowski, Sandor Heman, Niels Nes, and Peter Boncz. Super-Scalar RAM-CPU Cache Compression. In Proc. ICDE, 2006.

### 问题四：什么技术可以提升多用户/大量查询的I/O 性能？

协作顺序扫描 Cooperative scans

1. Marcin Zukowski, Sandor Heman, Niels Nes, and P. Boncz. Coop- erative Scans: Dynamic Bandwidth Sharing in a DBMS. In Proc. VLDB, 2007.  

