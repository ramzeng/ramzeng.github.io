---
title: "Redis SDS 结构"
date: 2024-02-29T18:38:05+08:00
draft: false
tags: ["Redis", "面试", "Redis 数据结构"]
---
## 介绍
SDS 全称为 Simple Dynamic String，用于存储二进制数据的一种结构，具有动态扩容、空间预分配、二进制安全等特性。Redis 中的字符串对象就是使用 SDS 结构实现的。

## 总体结构
```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
- len：记录 buf 中已使用的字节数
- alloc：记录 buf 中已分配的字节数
- flags：记录 buf 的类型，低 3 位表示类型，高 5 位未使用
- buf：存储字符串的字节数组

## 优势
- 常数复杂度获取字符串长度：读取 len 字段即可，不需要遍历整个字符串
- 不存在缓冲区溢出问题：根据 len 和 alloc 字段，判断是否需要扩容
- 空间预分配，减少内存重分配次数：每次扩容都会多分配一些内存，减少内存重分配次数
- 二进制安全，可以存储任意二进制数据：以 len 字段的长度来判断字符串的结束，而不是以空字符 `\0` 来判断
- 兼容部分 C 字符串函数：遵从每个字符串的最后一个字节是空字符 `\0` 的惯例，可以使用部分 C 字符串函数

## 空间预分配
当执行字符串拼接操作时，如果字符串的长度超过了已分配的内存，Redis 会对字符串进行扩容。扩容的规则如下：
1. 如果字符串的长度小于 1 MB，那么 Redis 会将字符串的长度扩大为原来的两倍
2. 如果字符串的长度大于等于 1 MB，那么 Redis 会将字符串的长度扩大 1 MB

> 预分配空间不会被释放，可能会造成一定的内存浪费，在内存分配次数和内存浪费之间做了一个权衡
