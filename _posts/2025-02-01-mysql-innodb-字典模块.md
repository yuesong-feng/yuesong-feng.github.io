---
layout: post
title: MySQL InnoDB 字典模块
date: 2025-02-01 21:31 +0800
author: yuesong-feng
category:
- 数据库内核
- MySQL
tags:
- mysql
---
# MySQL字典

字典是表、索引、模式、约束等数据库对象的描述信息，需要访问时会从系统表加载到内存。

MySQL中，字典有以下几种：

```c
dict_table_t    // 表
dict_col_t      // 列
dict_index_t    // 索引
dict_foreign_t  // 外键约束
```
