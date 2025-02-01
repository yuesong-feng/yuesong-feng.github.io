---
layout: post
title: MySQL InnoDB pcur源码解析
date: 2025-02-01 21:28 +0800
author: yuesong-feng
category:
- 数据库内核
- MySQL
tags:
- mysql
---
# btr_pcur源码分析

MySQL对B树进行查询、插入、更新、删除等操作时，经常使用btr_pcur(Persistent B-tree Cursor)完成，本文分析btr_pcur源码。

