---
title: "Redis IntSet 结构"
date: 2024-02-16T22:29:02+08:00
draft: false
tags: ["Redis", "面试", "Redis 数据结构"]
---
## 什么时候会使用 IntSet
1. 集合中的元素都是整数
2. 元素数量不超过 `set-max-intset-entries` 参数设置的阈值

> set-max-intset-entries：这个参数用于设置 intset 的最大元素数量。当集合的元素数量超过这个阈值时，Redis 会将底层实现从 intset 转换为 hashtable。默认值通常为 512

## 数据结构
```c
typedef struct intset {
    uint32_t encoding; // 编码方式
    uint32_t length; // 元素数量
    int8_t contents[]; // 元素数组
} intset;
```

contents 数组中的元素是有序的，且不重复。encoding 用于标识 contents 数组中的元素是什么类型的整数。encoding 的值有三种：
- INTSET_ENC_INT16 16 位整数
- INTSET_ENC_INT32 32 位整数
- INTSET_ENC_INT64 64 位整数

## 升级操作
如果插入的元素超过了 intset 的编码方式，Redis 会将 intset 升级为更大的编码方式。例如，如果 intset 的编码方式是 INTSET_ENC_INT16，而插入的元素是 32 位整数，那么 Redis 会将 intset 的编码方式升级为 INTSET_ENC_INT32

步骤如下：
1. 扩容 intset，将 intset 的编码方式升级为更大的编码方式
2. 从后往前，将 intset 中的元素转换为更大的编码方式
3. 插入新的元素

图解如下：
![](https://cdn4.codesign.qq.com/materials/2024/02/20/GD5Oj24EPoJE02Z3eAX20/cq1at6zql9bciluk/3c813895-76cc-4b73-8a99-702600e791e7.png)

![](https://cdn4.codesign.qq.com/materials/2024/02/20/GD5Oj24EPoJE02Z3eAX20/cq1at6zql9bciluk/23a60d05-52b9-493a-b79c-f5770b90dfcb.png)

![](https://cdn4.codesign.qq.com/materials/2024/02/20/GD5Oj24EPoJE02Z3eAX20/cq1at6zql9bciluk/fab08587-1a7c-4fa7-b7f3-a6f933a9b36b.png)

> 不支持降级操作，一旦升级，就不会再降级！！！

## 优势
1. 元素是有序的，且不重复
2. 元素是紧凑的，不会浪费空间，使用最少空间存储元素
3. 查找、插入、删除操作的时间复杂度都是 O(logN)，因为有序，可以使用二分查找

## 源码
https://github.com/redis/redis/blob/unstable/src/intset.h
https://github.com/redis/redis/blob/unstable/src/intset.c
