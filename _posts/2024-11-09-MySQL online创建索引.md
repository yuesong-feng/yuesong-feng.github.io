---
layout: post
title: MySQL online创建索引
date: 2024-11-09 22:46 +0800
author: yuesong-feng
categories: [数据库内核, MySQL]
tags: [mysql]
---
# online创建索引

    > mysql_alter_table
        > mysql_prepare_alter_table
        > create_table_impl 创建临时表
        > open_table_uncached 打开临时表
        > mysql_inplace_alter_table 
            > upgrade_shared_lock 将mdl升级成MDL_EXCLUSIVE
            > lock_tables 锁表
            > prepare_inplace_alter_table
            > downgrade_lock 将排他mdl锁降级成共享锁
            > inplace_alter_table inplace方式修改表
                > row_merge_build_indexes 通过扫描聚集索引创建新的索引
                    > row_merge_read_clustered_index 读聚集索引、创建二级索引
                    > row_log_apply 将row_log回放到已经创建好的索引中
            > wait_while_table_is_used
                > upgrade_shared_lock 将共享mdl锁升级成排他锁
            > commit_inplace_alter_table 提交或回滚在prepare_inplace_alter_table和inplace_alter_table期间的改变
                > row_merge_lock_table 对表上X锁，排空所有事物
                > commit_try_rebuild 重建表
                    > row_log_table_apply 将row_log_table中的日志应用到rebuild的表上
                > commit_try_norebuild 提交改变，但不重建表
                > trx_commit_for_mysql 提交数据字典的改变
                > row_mysql_unlock_data_dictionary 释放表锁
            > close_all_tables_for_name
            > close_temporary_table
            > mysql_rename_table 将临时表重命名为原表名