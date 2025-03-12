---
layout: post
title: PostgreSQL B树搜索
date: 2025-02-22 22:00 +0800
author: yuesong-feng
category:
- 数据库内核
- PostgreSQL
tags:
- postgresql
---
# PostgreSQL B树搜索

PG使用B-Link Tree作为存储结构，本文分析其搜索流程

_bt_first第一次搜索B树，先处理key，处理并行扫描，决定比较模式(<, <=, =, >, >=)，扫描顺序(向前或向后)，初始化scankey，最终进入_bt_search搜索key所在页，然后调用_bt_binsrch二分搜索页中记录。

_bt_first中还会决定nextkey。若nextkey = false，_bt_search和_bt_binsrch定位到的item >= scankey；若nextkey = true，定位到的item > scankey。nextkey由比较模式和扫描顺序决定。

_bt_search使用key搜索B树，返回key可能所在的第一个叶子页

```c
BTStack // 返回搜索栈
_bt_search(
    scankey // 搜索key
    nextkey // false则搜索第一个>= scankey的记录，true则搜索第一个> scankey的记录
    *bufP // 返回定位到的页
    access // BT_READ表示读B树，BT_WRITE表示会对B树进行修改，如插入、更新、删除等
)
{
    // 获取根页，上S锁
    _bt_getroot

    // 搜索到叶子层
    for (;;)
    {
        // 顺着downlink访问下层页时，由于无锁，下层页可能已经分裂，若分裂则key可能已经不再此页中了，需要右移。WRITE模式，此接口内可能会修复叶子页未完成的分裂，非叶子页的未完成分裂会在_bt_getstackbuf中修复，但在此接口内修复也无妨
        _bt_moveright // 非叶子页都是S锁

        // 此时抓到的页，一定是搜索key所在页

        if(这是叶子页)
            break;

        // 找到内页的pivot tuple，获取孩子页指针
        _bt_binsrch

        // 将搜索路径上的pivot tuple登记到搜索栈

        // 上面的搜索，对路径上的页上的锁都是S锁，如果到了叶子页上一层，且以BT_WRITE访问B树，则对叶子节点上X锁，否则还是S锁
        if (level == 1 && access == BT_WRITE)
            page_access = BT_WRITE;

        // 先释放父亲页的锁，再获取孩子页的锁
        _bt_relandgetbuf
    }

    // 若以BT_WRITE方式访问，但实际上以BT_READ方式访问了，即对叶子页上的是S锁
    // 说明B树只有一层，根页就是叶子页，此时需要加上X锁
    if (access == BT_WRITE && page_access == BT_READ)
    {
        _bt_unlockbuf                   // 解S锁
        _bt_lockbuf(BT_WRITE)           // 加X锁
        _bt_moveright(BT_WRITE)         // 对根页解锁后、加锁前，可能已经分裂，若分裂则需要右移到右兄弟页
    }
}
```

可以看到，搜索的流程非常简单，对路径上的节点都上的S锁，BT_WRITE方式则对叶子节点上X锁，BT_READ方式则对叶子节点上S锁。

```c
_bt_moveright(
    nextkey // false则搜索第一个>= scankey的记录，true则搜索第一个> scankey的记录
    forupdate // true则尝试修复遇到的未完成的分裂
    stack // 仅用于修复未完成的分裂
    access // READ或WRITE
)
{
    // nextkey = false：如果scankey > high key，则页已分裂，需要右移（若scankey等于high key，可能需要右移，需要扫描右页）
    // nextkey = true：如果scankey >= high key，则右移
    cmpval = nextkey ? 0 : 1

    for(;;)
    {
        if (已经是最右页)
            break;

        // 修复未完成的分裂
        if (forupdate && INCOMPLETE_SPLIT)
        {
            // access是READ模式，则buf是S锁，需要升级成X锁

            // 释放S锁、获取X锁有窗口，可能已经完成分裂
            if (INCOMPLETE_SPLIT)
                _bt_finish_split
            else
                _bt_relbuf // 释放X锁
            // 重新根据access获取锁，重新检查
            buf = _bt_getbuf
            continue；
        }
        
        if (_bt_compare(scankey, P_HIKEY) >= cmpval)
        {
            // 如果scankey大于(或等于)页的high key，则右移，会释放当前页
            _bt_relandgetbuf(next)
            continue; // 可能右移多次
        }
        else
            break;  // 结束
    }
}
```