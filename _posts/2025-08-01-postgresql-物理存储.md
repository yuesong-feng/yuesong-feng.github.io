---
layout: post
title: PostgreSQL物理存储
date: 2025-08-01 13:00 +0800
author: yuesong-feng
category:
- 数据库内核
- PostgreSQL
tags:
- postgresql
---
# PostgreSQL物理存储

PostgreSQL内部，所有数据库对象都通过oid（object identifier，对象标识符）进行管理，包括数据库、表、索引等。

数据库的信息在pg_database系统表里，可以通过以下语句查询数据库的oid：
```
select datname, oid from pg_database;
```
每个数据库都和base目录下一个子目录对应，目录名与数据库oid相同。若oid为16384，则目录为base/16384

表和索引在数据库目录中存储为单个（小于1G）或多个文件，文件名和其oid可以不同，可通过以下语句查询：
```
select relname, oid, relfilenode from pg_class where relname = 't1';
```
若relfilenode为18740，则表对应的文件路径为base/16384/18740  
relfilenode可能会被一些命令（如truncate、reindex、cluster）改变，比如truncate表时，老的数据文件被删除，用一个新的数据文件替代。  
pg_relation_filepath能根据oid或名称返回关系对应的文件路径
```
select pg_relation_filepath(16385);
select pg_relation_filepath('t1');
```
表和索引的文件大小超过1G时，会依次relfilenode.1、relfilenode.2文件，构建PostgreSQL时使用配置选项--with-segsize可更改最大文件大小
每个表都有后缀为_fsm和_vm的两个关联文件，即空闲空间映射和可见性映射文件。索引没有可见性映射文件，只有空闲空间映射文件。具体作用参考《PostgreSQL空闲空间映射表(FSM)》和《PostgreSQL可见性映射表(VM)和VACUUM操作》文章

