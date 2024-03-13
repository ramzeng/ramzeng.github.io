---
title: "Redis ZSkipList 结构"
date: 2024-03-13T07:34:29+08:00
draft: false
tags: ["Redis", "面试", "Redis 数据结构"]
---
## 结构
```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
- ele: 持有数据，是一个 sds 结构
- score: 分值
- backward: 指向前一个节点
- level: 层级，包含 forward 和 span 两个字段
    - forward: 指向下一个节点
    - span: 记录跨度，即当前节点到下一个节点的距离
## 优势
- 插入、删除、查找的时间复杂度都是 O(logN)
- 有序，可以使用跳表进行范围查询