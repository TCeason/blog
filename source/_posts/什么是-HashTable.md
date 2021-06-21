---
layout: title
title: 什么是 HashTable
date: 2021-06-21 15:45:03
tags:
---

HashTable 是一种存储 key-value pairs（键值对）的数据结构。

Key 是被发送到一个用来对其实行运算的 hash 函数。

这个过程称为 Hashing（散列）
```
hash 函数运算后的结果（通常被称为 `hash 值` 或者 hash）是 key-value pair 在 HashTable 中的索引。
```

<!-- more -->

## 组成部分

基本的 HashTable 包括两个部分：

### Hash Function

正如我们上面提到的，hash 函数决定了我们 key-value pair 的索引。选择一个高效的 hash 函数是创建一个好的 HashTable 至关重要的部分。你应该始终确保他是一个单向函数（one-way function）。
即，无法从哈希中检索密钥。

好的的 hash 函数的另一个特性是：对于不同的 key 可以产生不同的 hash。

### Array

Array 包含 HashTable 中所有 key-value 条目。数组的大小应该根据期望的数据量设置。

## Collisions in hash tables & resolutions

当两个或者多个 key map（映射）到相同的 index（索引）会发生 collisions（`冲突`）。有几种处理 index 冲突的方法：

### Liner probing

如果一对 key-value 被分配到一个已经被占用的 slot（插槽），会线性的在当前 HashTable 搜索下一个空闲的插槽。

### Chaining

HashTable 将会是一个链表数组。map 到同一 index 的所有 key 都将存储为该索引处的链表节点。

优势：

* 通过 chaining，在哈希表中的插入总是在 O(1) 中发生，因为链表允许在恒定时间内插入。

* 理论上，只要有足够的空间，链式哈希表就可以无限增长。

* 使用 chaining 的哈希表永远不需要调整大小。

### Resizing the HashTable

可以增加 HashTable 的大小，以便将 Hash 条目分割的更远。Threshold value（临界点）表示在调整大小之前需要占用的 HashTable 的百分比。HashTable threshold value 为 0.6 表示当 HashTable 空间使用率为 60 % 时进行一次扩容。按照惯例，HashTable 的大小扩容至二倍。这是内存密集型的。

