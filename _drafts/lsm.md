---
layout: post
title: lsm
---

lsm 一些介绍

<!--more-->

TPC: 事务处理性能委员会
https://www.ibm.com/developerworks/cn/data/library/db-cn-tpc-ctest/index.html

Five Minute Rule:
用来确认数据应该是放在内存中，还是存储在磁盘里再读回内存。
如果数据每 5 分钟内访问一次，数据放内存中是赚的，如果大于 5 分钟，那么就放在硬盘里。

当 page reference 的频率超过每 60 秒一次的时候， 我们可以通过使用 memofy buffer space 让 page 保存在内存中以减少系统开销。因此也减少了磁盘 io。

LSM
LSM 有两个组建，其中小的一个是 C0，全部放在内存之中。
C1 是放在磁盘里的。虽然 C1 一直在磁盘里，但是也有一部分频繁引用的部分放在内存里。
新数据到来的时候，用于恢复的日志率先插入序列日志文件，随后索引项目回插入内存的 C0中，随后一段时间后，回迁移合并到 C1 树的磁盘上。
每一个搜索都是先看 C0，然后再看 C1.
疑问？索引是单独文件吗？挂掉后，C0 索引数据如何恢复

当 C0 的内存达到一个阈值的时候，我们会开启 rolling merge。将一些连续的数据从 C0 中删除，再合并到 C1 树中去。
C1 有与 B 树很相似的结构，但是优化了顺序磁盘访问。（SB-tree）。
C1 中父亲索引节点会在 buffer 中停留更长的时间，也是为了降低 io 服务的。
