---
layout: post
title: PostgreSQL B树插入
date: 2025-02-22 19:32 +0800
author: yuesong-feng
category:
- 数据库内核
- PostgreSQL
tags:
- postgresql
---
# PostgreSQL B树插入

PG使用B-Link Tree作为存储结构，本文分析其插入流程

_bt_doinsert将一个tuple插入到B树：
```c
_bt_doinsert
{
    // 构造scan key
    _bt_mkscankey

    // WRITE模式定位待插入的叶子页。路径上的节点都只会上S锁，访问孩子页后解锁，叶子节点会上X锁。搜索时会登记搜索路径到stack中
    _bt_search

    // 在页中找到插入位置，若空间不足，可能插入到右页第一个。返回待插入的页和插入位置
    _bt_findinsertloc

    // 将itup插入到页中，若页满则分裂并维护索引结构
     bt_insertonpg
}
```

_bt_insertonpg的比较关键的参数和流程如下：

```c
_bt_insertonpg(
    buf // 待插入的页，已上X锁
    cbuf // 插入到叶子页则传NULL；若插入到内页，则在为新分裂出的右页补全downlink，cbuf是新分裂出的页的左兄弟页，此函数会将页上的分裂未完成标记清除，并释放X锁
    stack // 定位到此页的搜索栈，保存路径上的pivot tuple
    newitemoff // 待插入的位置
)
{
    if (页满，需要分裂)
    {
        // 找到分裂点，决定itup插入到左页还是右页
        _bt_findsplitloc

        // 将页分裂成左右页，将itup插入到其中一页，正确维护相关页的左右指针
        rbuf = _bt_split

        // 此时，已经分裂，左页还是buf，右页是rbuf，都上了X锁，需要向父节点插入指向右页的元组，这个元组的key也是左页的high key
        _bt_insert_parent
    }
    else
    {
        // 将itup插入到页中
        _bt_pgaddtup

        if (cbuf)
        {
            // 如果是在为新分裂的页插入downlink，则将cbuf的分裂未完成标记清除
        }

        // 写redo日志

        // 释放cbuf和buf
    }
}
```

可以看到，如果不发生分裂，插入只会拥有待插入页的X锁。如果发生分裂，会先将待插入页分裂成两个页并插入，然后在父节点中补全新分裂出的右页的downlink。这是两个原子步骤，中间有窗口。如果此时数据库故障，那么新分裂出的右页将会缺少父亲页上的downlink，而左页将会有分裂未完成标记。

```c
rbuf // 返回分裂得到的新的右页
_bt_split(
    buf // 待分裂的页，已上X锁
    cbuf // 分裂非叶子页时，cbuf是新分裂出的页的左兄弟页，此函数会将页上的分裂未完成标记清除，并释放X锁
    firstright // 分裂点，需要移到新的右页的第一条记录
    newitemoff // 待插入的记录的位置
    newitemonleft // 待插入的记录在左页还是右页
)
{
    // 分配一个新的右页，上X锁
    rbuf = _bt_getbuf

    // 拷贝一个新左页，维护左右页的指针

    if (待分裂的页不是最右页)
    {
        // 将待分裂页的high key拷贝到新分裂出的右页
    }

    // 分裂后新左页的high key是新右页的第一个key，可能是之前存在的元组，也可能是待插入的新元组
    if (!newitemonleft && newitemoff == firstright)
    {
        // 待插入的元组是右页第一个
    }
    else
    {
        // 之前存在的firstright是右页第一个
    }
    // 将上面得到的high key插入到新左页

    // 将所有元组移动到正确的页，包括待插入的新元组

    if (分裂后的右页不是最右页)
    {
        // 获取分裂前的页之前的右页，设置其prev指针，指向分裂后的右页
    }

    if (不是叶子页)
    {
        // 插入的新元组修复了未完成的分裂，清空cbuf的分裂未完成标记
    }

    // 写redo日志

    // 释放分裂前的页之前的右页，以及cbuf，此时仅分裂后新的左右页有X锁
}
```

可以看到，_bt_split的作用仅仅是将页分裂，并维护本层正确的指针。调用前拥有待分裂页的X锁，调用后拥有分裂出的新左右页的X锁。分裂结束后，需要将新右页的downlink插入到父节点中

```c
_bt_insert_parent(
    buf // 分裂后的左页，已上X锁
    rbuf // 分裂后的右页，已上X锁
    stack // 搜索栈
)
{
    if (is_root)
    {
        // 分裂的是根页，则创建一个新的根页
        _bt_newlevel
        // 释放分裂后的左右页以及新根页的锁
    }
    else // 分裂的不是根页
    {
        // 获取左页的high key，也就是新的右页的最小key

        // 创建一个指向新右页的pivot tuple

        // 利用搜索栈，找到父亲页，上X锁
        _bt_getstackbuf

        // 释放右页rbuf的X锁，左页buf的锁会在下面_bt_insertonpg中被释放

        // 递归将元组插入到父亲页中
        _bt_insertonpg(pbuf, buf, stack.parent)
    }
    // 释放相关页的封锁
}
```

// 利用搜索栈，找到父亲页的逻辑

```c
_bt_getstackbuf
{
    for(;;)
    {
        // 获取当前搜索栈的页

        if (WRITE方式访问，且页存在分裂未完成标记，则修复未完成的分裂)
        {
            _bt_finish_split
            continue
        }

        // 查找当前页中有没有搜索栈保存的记录，找到则返回当前页

        // 当前页中没有，说明已经分裂，右移继续寻找
    }
}
```

下面分析完成此前未完成的分裂、修复B树结构的函数逻辑：

```c
_bt_finish_split
{
    _bt_getbuf(next, BT_WRITE)          // 获取当前页的右兄弟页，上X锁，这个页缺少了父亲内页向下的指针

    wasroot = true/false                // 判断当前页是否为root页
    wasonly = P_LEFTMOST(lbuf) && P_RIGHTMOST(next) // 如果当前页是最左页，其右兄弟页是最右页，说明分离前当前页是这一层唯一一页

    _bt_insert_parent()                 // 将右兄弟页的privot tuple插入到父亲页中，完成分裂
}
```

可以看到，数据库故障导致的B树结构不完整，只会有一种情况，即分离时，父亲页中还没有插入新的右兄弟页的索引元组。

在BT_READ方式访问B树时，即使B树结构不完整，也能保证搜索的正确性。

BT_WRITE方式访问B树时，访问到分裂未完成的页，则会修复B树结构，即将新的右兄弟页的pivot tuple插入到父亲页中。

## 删除

PostgreSQL中对B树的删除都是标记删除，后续通过vacuum清理B树，B树删除接口btbulkdelete和btvaruumcleanup都是一次删除一批记录，没有单个记录删除的实现。

## 更新

PostgreSQL中不会对B树元组进行更新，都会变成删除+插入。

