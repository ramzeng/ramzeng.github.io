---
title: "分区算法"
date: 2023-08-18T21:54:41+08:00
draft: false
tags: ["分布式"]
---
## 水平分区算法
用于计算某个数据应该划分到哪个分区上，不同的分区算法有着不同的特性。

### 范围分区
根据指定的关键字将数据集拆分为若干连续的范围，每个范围存储到一个单独的节点上，用来分区的关键字称为分区键。 比如按月按年拆分数据表就是一个范围分区的例子。

![range-partition](/range-partition.png)

#### 优点
- 实现相对简单
- 分区键支持范围查询
- 容易重新分区
#### 缺点
- 分区键以外的关键字不支持范围查询
- 可能产生数据分布不均匀问题

### 哈希分区
根据指定的关键字计算哈希值，然后将哈希值映射到指定的分区上。

![hash-partition](/hash-partition.png)

#### 优点
- 数据分布相对均匀

#### 缺点
- 不额外存储数据的情况下，不支持范围查询
- 增减分区时，会导致数据大规模移动，系统可能无法继续提供服务

### 一致性哈希
一种特殊的哈希分区算法，可以缓解哈希分区缺点中的数据大规模移动问题。

将多个节点映射到一个环上，根据数据的哈希值，将数据也映射到该环上，数据存储在顺时针方向遇到的第一个节点上。

![consistent-hash](/consistent-hash.png)

#### 优点
- 增减分区时，只需移动部分数据，不会导致数据大规模移动

#### 缺点
- 不额外存储数据的情况下，不支持范围查询
- 节点数量较少时，数据分布不均匀

#### 改进
- 虚拟节点：将每个节点映射到多个环上，解决数据分布不均匀问题

#### 实现
```go
package main

import (
	"hash/crc32"
	"sort"
	"strconv"
)

type HashFunc func([]byte) uint32

type ConsistentHash struct {
	hashFunc HashFunc
	replicas int
	ring     []int
	nodes    map[int]string
}

func (m *ConsistentHash) Add(nodes ...string) {
	for _, node := range nodes {
		// 副本，虚拟节点
		for i := 0; i < m.replicas; i++ {
			hash := int(m.hashFunc([]byte(strconv.Itoa(i) + node)))
			m.ring = append(m.ring, hash)
			m.nodes[hash] = node
		}
	}

	sort.Ints(m.ring)
}

func (m *ConsistentHash) Get(key string) string {
	if len(m.ring) == 0 {
		return ""
	}

	hash := int(m.hashFunc([]byte(key)))

	// 如果不存在，将返回 len(m.ring) 的值
	index := sort.Search(len(m.ring), func(i int) bool {
		return m.ring[i] >= hash
	})

	// 顺时针取下一个节点
	// index%len(m.ring)，如果没有找到对应的下标，则取第一个节点
	return m.nodes[m.ring[index%len(m.ring)]]
}

func (m *ConsistentHash) Remove(node string) {
	if len(m.ring) == 0 {
		return
	}

	for i := 0; i < m.replicas; i++ {
		hash := int(m.hashFunc([]byte(strconv.Itoa(i) + node)))
		index := sort.SearchInts(m.ring, hash)

		if index >= len(m.ring) {
			continue
		}

		m.ring = append(m.ring[:index], m.ring[index+1:]...)
		delete(m.nodes, hash)
	}
}

func NewConsistentHash(replicas int, hashFunc HashFunc) *ConsistentHash {
	m := &ConsistentHash{
		hashFunc: hashFunc,
		replicas: replicas,
		nodes:    make(map[int]string),
	}

	if m.hashFunc == nil {
		m.hashFunc = crc32.ChecksumIEEE
	}

	return m
}
```
