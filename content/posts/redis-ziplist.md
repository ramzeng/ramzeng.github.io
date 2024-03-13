---
title: "Redis ZipList 结构"
date: 2024-03-05T10:22:12+08:00
draft: false
---
## 结构

```C
 <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```

- zlbytes: uint32_t，记录整个 ziplist 的字节数
- zltail: uint32_t，记录 ziplist 中最后一个节点的偏移量
- zllen: uint16_t，记录 ziplist 中节点的数量
- entry: ziplist 中的节点，包含 prevlen、encoding、entry-data（可能没有）
- zlend: uint8_t，标识 ziplist 的结束

## Entry 结构

```C
<prevlen> <encoding> <entry-data>（可能没有）
```

- prevlen: 表示前一个 entry 的长度
  - 如果前一个 entry 的长度小于 254 字节，prevlen 占用 1 字节，记录前一个 entry 的长度
  - 如果前一个 entry 的长度大于等于 254 字节，prevlen 占用 5 字节，第一个字节为 0xFE，后面 4 字节记录前一个 entry 的长度
- encoding: 表示当前 entry 的类型和长度
  - 前两位表示类型，当前两位为 11 时，表示 entry 为 int 类型，否则为 string 类型
- entry-data: 实际存储的数据，如果是 int 类型，entry-data 会合并到 encoding 中，否则会单独存储在 entry-data 中

## 优势

- 利用 encoding 字段来细化存储空间长度，压缩存储空间
- 元素长度不相同，所以用 prevlen 字段满足从后向前遍历的需求

## 缺点

- 不预留空间，每次写操作都会进行内存分配
- 如果节点扩容，会导致它后面的节点的 prevlen 需要重新计算

