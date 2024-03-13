---
title: "Redis 数据结构"
date: 2024-03-13T09:30:02+08:00
draft: false
tags: ["Redis", "面试"]
---
## String
### 介绍
最基本的数据类型，一个 key 对应一个 value
### 底层结构
- [SDS](/posts/redis-sds/)
### 应用场景
- 缓存
- 计数器
### 常用命令
| 命令                 | 描述                     |
|---------------------|--------------------------|
| SET key value       | 设置 key 对应的值为 value  |
| GET key             | 获取 key 对应的值          |
| DEL key             | 删除 key 对应的值          |
| INCR key            | key 对应的值加 1           |
| DECR key            | key 对应的值减 1           |
| INCRBY key increment| key 对应的值加 increment  |
| DECRBY key decrement| key 对应的值减 decrement  |
## List
### 介绍
双端链表，支持从两端插入和删除
### 底层结构
- [QuickList](/posts/redis-quicklist/)
### 应用场景
- 消息队列
### 常用命令
| 命令                 | 描述                     |
|---------------------|--------------------------|
| LPUSH key value1 [value2 ... valueN] | 从左边插入一个或多个值 |
| RPUSH key value1 [value2 ... valueN] | 从右边插入一个或多个值 |
| LPOP key            | 从左边删除一个值           |
| RPOP key            | 从右边删除一个值           |
| LINDEX key index    | 获取指定索引的值           |
| LRANGE key start stop | 获取指定范围的值         |
## Hash
### 介绍
哈希表，类似于 Java 中的 HashMap
### 底层结构
- [HashTable](/posts/redis-hashtable/)
- [ZipList](/posts/redis-ziplist/)
> 当哈希表中的键值对数量小于一定阈值时，使用 ZipList，否则使用 HashTable
### 应用场景
- 缓存
### 常用命令
| 命令                 | 描述                     |
|---------------------|--------------------------|
| HSET key field value | 设置 key 对应的哈希表中 field 的值为 value |
| HGET key field      | 获取 key 对应的哈希表中 field 的值 |
| HGETALL key         | 获取 key 对应的哈希表中所有的 field 和 value |
| HDEL key field1 [field2 ... fieldN] | 删除 key 对应的哈希表中一个或多个 field |
## Set
### 介绍
无序集合，元素不重复
### 底层结构
- [IntSet](/posts/redis-intset/)
- [HashTable](/posts/redis-hashtable/)
> 当集合中的元素都是整数且元素数量小于一定阈值时，使用 IntSet，否则使用 HashTable
### 应用场景
- 标签
- 点赞、收藏
### 常用命令
| 命令                 | 描述                     |
|---------------------|--------------------------|
| SADD key member1 [member2 ... memberN] | 添加一个或多个元素 |
| SCARD key           | 获取集合的元素数量         |
| SMEMBERS key        | 获取集合的所有元素         |
| SISMEMBER key member | 判断 member 是否是集合的元素 |
## Sorted Set
### 介绍
有序集合，元素不重复，每个元素都会关联一个分数
### 底层结构
- [ZipList](/posts/redis-ziplist/)
- [ZSkipList](/posts/redis-zskiplist/)
> 当有序集合中的元素数量小于一定阈值时，使用 ZipList，否则使用 ZSkipList
### 应用场景
- 排行榜
### 常用命令
| 命令                 | 描述                     |
|---------------------|--------------------------|
| ZADD key score1 member1 [score2 member2 ... scoreN memberN] | 添加一个或多个元素 |
| ZRANGE key start stop [WITHSCORES] | 获取指定范围的元素 |
| ZREM key member1 [member2 ... memberN] | 删除一个或多个元素 |