---
layout: post
title: 解密列存 parquet
---

在做数据分析的时候，相对于传统关系型数据库，我们更倾向于计算列之间的关系。在使用传统关系型数据库时，基于此的设计，我们会扫描很多我们并不关心的列，这导致了查询效率的低下，大部分数据库 io 比较低效。因此目前出现了列式存储。[Apache Parquet](http://parquet.apache.org/) 是一个列式存储的文件格式。从这里入手，提升对列存的理解。当我还没看的时候，我还是很疑惑的，跟大家一样，向 hdfs 还是 lsmt 的设计，为了提升写入性能，都是使用 append 进行操作，而 append 如何将行式的数据转换成列式的数据呢？

<!--more-->

## Parquet
在深入 parquet 之前，我们需要先了解一下后面需要学习的术语。

### 术语
#### Block
Block（hdfs block）：这是指 hdfs 的一个 block，parquet 运行在 hadoop 生态之中，文件的格式需要很好的与这些特征契合起来。
因为我之前也不知道这是什么，所以我们先来学习一下 Block 是啥。
Block 是 hdfs 中的最小的存储单元，使得其能将大文件切分为多个小文件，实现大文件的存储。并可以对多个小文件做合适的 replication，实现错误容忍（精髓）以及HA。
例如我们现在有一个文件一共是 518 MB，而设置的 hdfs block size 是 128 MB，那么它会由 5 个 block（128MB + 128MB + 128MB + 128MB + 6MB）来组成。所以我们处理数据的时候需要考虑同一行数据被写在两个不同的 block 上的情况，这里就不展开了（hdfs 细节还不是很清楚）。
#### File
一个 hdfs 文件，必定有 metadata。并不必须要数据。
#### Row Group
将我们的数据水平上的一个逻辑分区，按 mysql 来理解就是一些数据行，这些行组成一个 row group。这个 group 中包含数据集中每一列的 chunk（Column Chunk）。
#### Column Chunk
特定列的一大块数据。它存在于特定的一个 row group 中，并在物理上认为是连续的。
#### Page
column chunk 被划分为 pages。一个 page 认为是最小不可分割单元（就压缩和编码而言）。会有很多种 page type 在一个 column chunk 中交错存储。

一个文件会由一个或者多个 row group 组成，每个 row group 对其中一列只会存在一个 column chunk。column chunk 中含有多个 pages。







## Reference
- [Apache Parquet Document](http://parquet.apache.org/documentation/latest/)
- [hdfs design](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html)
