---
layout: post
title: PostgreSQL可见性映射表(VM)和VACUUM操作
date: 2024-11-10 22:18 +0800
author: yuesong-feng
category:
- 数据库内核
- PostgreSQL
tags:
- postgresql
---
# PostgreSQL可见性映射表(VM)和VACUUM操作

PostgreSQL为了实现多版本并发控制（MVCC），当事务删除或者更新元组时，并非从物理上删除，而是将其标记无效，最终再通过`VACUUM`命令清理这些无效元组，真正的物理删除发生在清理过程。清理无效元组时，需要先找到无效元组，再进行清理。如果没有其他技术，需要遍历查找每一个页，找到页中的无效元组。如果更新和删除不是很频繁，表文件中大部分页都没有无效元组，如果遍历所有页，开销会非常大。为了解决这个问题，PostgreSQL使用了可见性映射表（VM）来记录文件的无效元组信息。

在PostgreSQL中，对于每一个表文件（包括系统表）都会同时存在一个名为`OID_vm`的空闲空间映射表，其中`OID`是对应表的oid。VM中为表的每个页设置了一位标识来表示该页是否存在无效元组。当页中所有的元组对于当前事务都是可见的，VM中对应的位为1，表示没有无效元组。对某个页中的元组进行更新或删除后，该页在VM中的标志为为0，表示有无效元组。注意设置VM时需要加锁。当页在VM中对应的标志为是1时，VACUUM会忽略对应的页，大大提高VACUUM的效率。

VACUUM有两种方式，即快速清理(Lazy VACUUM)和完全清理(FULL VACUUM)。VM文件仅在快速清理中被用到，对于FULL VACUUM，由于要执行跨页清理等复杂操作，还是需要扫描整个表文件，此时VM作用不大。当前VM文件是一个提示（hint）来加快VACUUM的速度，即使损坏了VM文件，仅仅会导致VACUUM忽略那些需要清理的页面，但元组的可见性依然不变，所以对数据不会产生任何影响。

每个VM文件的页中，可用的字节为`blcksz - SizeOfPageHeaderData`个，所以每个VM页能记录`size = (blcksz - SizeOfPageHeaderData) * 8`个表文件页的信息。
