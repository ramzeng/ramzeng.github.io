---
title: "Redis HashMap"
date: 2024-03-12T23:19:45+08:00
draft: false
tags: ["redis", "面试", "Redis 数据结构"]
---
## 结构
```c
typedef struct dict {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dict;

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;
} dictEntry;
```
- key: 键
- val: 值，值可以是一个指针，也可以是 uint64_t 或 int64_t 类型的整数
- next: 指向下一个节点，形成链表

## 哈希算法
```c
// 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

// 使用哈希表的 sizemask 属性和第一步得到的哈希值，计算索引值
index = hash & dict->ht[x].sizemask;
```

## 关注点
- 解决哈希冲突：拉链表，使用 *next 指向下一个具有相同索引值的哈希表节点
- 扩容和缩容：根据原哈希表已使用的空间扩大一倍或缩小一半
- 扩容条件：
    - 没有执行 BGSAVE 或 BGREWRITEAOF 命令，且负载因子大于 1
    - 在执行 BGSAVE 或 BGREWRITEAOF 命令，负载因子大于 5
- 渐进式 rehash：在 rehash 的过程中，新哈希表和旧哈希表同时存在，查询时先查询新哈希表，再查询旧哈希表，写入时只写入新哈希表
