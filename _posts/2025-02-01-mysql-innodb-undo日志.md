---
layout: post
title: MySQL InnoDB undo日志
date: 2025-02-01 21:12 +0800
author: yuesong-feng
category:
- 数据库内核
- MySQL
tags:
- mysql
---
# Undo日志

undo日志有两个作用：

1. 实现事务的隔离性。由于事务隔离级别的要求，其他事务对表的修改可能对当前事务不可见，此时当前事务需要通过undo日志拼装出最后可见的记录。

2. 保证数据的一致性和正确性。事务回滚时，需要通过undo日志重做对表的操作。

MySQL中，redo日志是mtr实现的，undo日志则是trx实现的。

未完待续