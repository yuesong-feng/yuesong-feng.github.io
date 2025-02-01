---
layout: post
title: MySQL InnoDB mtr源码解析
date: 2025-02-01 12:44 +0800
author: yuesong-feng
category:
- 数据库内核
- MySQL
tags:
- mysql
---
# Mini Transaction

MySQL InnoDB中，mtr是一个非常重要的模块，主要控制redo日志和B树页锁。

## redo日志

redo日志即数据库的预写日志（WAL），有以下两个作用：

1. 提高事务的提交速度。事务修改的数据页在磁盘上极有可能是不连续的，刷盘将会是随机IO，效率较差。事务会将所有对数据页的修改记录成redo日志，提交时只需保证这些redo日志刷盘成功即可，不需要等待修改的脏页落盘。redo日志是顺序写入磁盘的，效率较高，故redo日志可以提高事务的提交速度，提高数据库性能。

2. 保证事务的持久性。事务提交时，对数据页的修改可能不会立即刷盘，而是将内存中的页标记为脏页并延迟刷盘。若脏页刷盘前数据库宕机，重启恢复时，由于内存的易失性，事务对数据页的修改丢失。此时由于事务的redo日志已刷盘成功，通过redo日志可还原该事务对数据页的修改并重做，以保证事务的持久性。

## 页锁

MySQL中数据页可以被并发事务访问，但需要加页锁，5.6版本只有S锁和X锁。访问数据页但不修改需要加S锁，S锁和S锁不冲突，即一个页可以同时被多个线程访问。若需要修改页数据，则需要加X锁，X锁和S锁、X锁冲突，即一个页只能同时被一个线程修改。

mtr可以记录下访问B树过程中对数据页加的页锁，放到mtr->memo中，在mtr_commit时会一起释放。mtr_commit时，会先将对数据页的修改的redo日志落盘，然后即可释放页锁。

## mtr源码分析

MySQL中，redo日志需要先放到mtr->log，mtr_commit时刷盘。一个mtr的所有redo日志要么都落盘成功、要么都不落盘，不可能落盘一部分，所以一个mtr中的redo日志落盘具有原子性。

```c
struct mtr_t{
	dyn_array_t	memo;	            // 记录mtr锁住的数据页
	dyn_array_t	log;	            // redo日志
	unsigned	inside_ibuf:1;      // 内部ibuf是否改变
	unsigned	modifications:1;    // mtr修改了buffer页
	unsigned	made_dirty:1;       // mtr让至少一个buffer页变成脏页
	ulint		n_log_recs;         // 写入mtr log的页初始化记录
	ulint		n_freed_pages;      // mtr释放的页
	ulint		log_mode;           // 决定哪些操作需要生成redo日志
	lsn_t		start_lsn;          // 起始lsn
	lsn_t		end_lsn;            // 结束lsn
};
```

mtr_start开启一个mtr，后续使用该mtr访问B树时，对数据页加的锁会被记录到memo中，对数据页的修改的redo日志会被记录到log中。

mtr_commit主要做了两件事：调用mtr_log_reserve_and_write将所有redo日志落盘，然后调用mtr_memo_pop_all将memo中记录的所有锁释放。

未完待续