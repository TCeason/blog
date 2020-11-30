---
title: ClickHouse OOM 分析
date: 2020-11-30 19:20:47
categories: DataBase
tags: ClickHouse
---

## 1. 前言

Out Of Memory （OOM）到目前为止已经出现了 40 年。大概就是当某个应用想要使用的内存超过了现有可用的内存总和，本文将不会具体进行赘述。为了防止发生OOM ，采取了各种各样的方式，最常见的就是，当申请内存时，发现无法申请所需要的内存，系统主动 `kill` 当前内存占用最大的应用。

这样带来的好处是，当前应用可以正常使用了。但是，缺点也是显而易见的：会让当前最大内存占用的应用无法正常运行。

在数据库中，这尤为常见。比如，在一台数据库服务器中运行了  MySQL Server ，同时要对这个数据库进行备份，而执行 mysqldump 或者 xtrabackup 或者 mydumper 等等备份命令时，由于机器内存不足以运行备份命令，而 `kill` 掉了内存占用最大的 `MySQL Server`。着带来的后果就很直接了，业务读写失败。

同样的，当下的 OLAP 骄子 `ClickHouse` 也是存在这种问题，那么他的 OOM 一般是如何引起的呢，又要如何避免呢？

<!-- more -->

## 2. 发生 OOM 的原因

OOM 是什么我们已经知道了，那么，ClickHouse OOM 的原因其实应该主要分为

* 查询导致 OOM 

* 写入导致 OOM 

查询导致 OOM 比较好理解，就是，当某个 query 要读取的数据量过大了，内存不够用。

写入会导致 OOM 可能大家不是很理解，有人如果认为 insert into select 是属于写入导致 OOM ，这应该不算全对，毕竟还进行了查询。所以，接下来，我们仔细区分一下这两类原因。

### 2.1. 查询导致OOM

比如某个大数据量的表做聚合排序（ `GROUP BY` 和 `ORDER BY`）操作，导致需要将大量的数据读取到内存中，然后按照 SQL 要求进行分组和排序。数据对内存的消耗基本是 数据:内存> 1:1 的。也就是，如果又 1GB 数据需要做聚合排序操作，那么他需要的内存是要超过 1GB 的。数据量越大，所需内存会更加巨大。常见的机器是无法满足这种需求的。


### 2.2. 写入导致OOM

对于大多数小内存的数据库服务器来说，如果一次写入的batch过大都有可能会引起OOM。但是，对于 ClickHouse 来说，可能却不是这样。并发线程数目20 ，每个线程只写入一行，每行写入的列只有 15 个。都有可能引发 OOM 。

可能看到这里会觉得 ClickHouse 一定设计的不合理，要不然为什么如此小的数据写入都会引发 OOM 呢？

如果会有这样的问题，说明不是很 ClickHouse 。ClickHouse 支持多种表引擎。其中有一个 MergeTree 族群表引擎，MergeTree 在 ClickHouse 的地位基本等同于 Innodb 在 MySQL 的地位。MergeTree 引擎是基于 LSM 算法实现的。每次写入就会生成一个小文件，然后 ClickHouse Server 再去合并每个小文件到数据文件中。关于 MergeTree 原理将会在后面的文章中进行详细介绍。

理解了MergeTree的 merge 工作后，就比较清晰了。当多个线程每次只写入一行数据时， insert query 的每个列会生成两个文件。因此，按照上面的写入方式，计算出消耗的内存为：

> 2MB * 15 * 20 = 600 MB

600MB 可能不大，但是，换算一下百分比，如果机器内存是 8GB ，8GB 的 10% 内存也只有 800MB 左右呀。而一个数据库机器，一般负载下内存占用达到机器的 70% 左右。如果突然来这么一次20行数据的写入，内存就会飙升 10% ，我想这会是很令人疑惑的一件事。而且，如果有人在 AP 中一次写入只包含一行数据，那他可能确实没有理解 AP 数据库的精髓所在。


## 3. 如何避免 OOM 

OOM 的原因我们简单分析过了，主要是查询和不正当写入导致的。因此，避免 OOM 也就变得简单起来。

### 3.1. 避免查询时 OOM 

对于如何避免查询时发生 OOM ，数据库常见的做法就是外排到磁盘。比如，众所周知的，MySQL 查询慢了，就去 explain，看到结果有臭名昭著的 `Using filesort` 而刚好要排序的数据量大于session的 sort_buffer 时，就会自动使用磁盘排序了。

而 ClickHouse 的做法也是比较类似。同样也有 setting 进行控制：

> [max_bytes_before_external_group_by](https://clickhouse.tech/docs/en/sql-reference/statements/select/group-by/#select-group-by-in-external-memory): The max_bytes_before_external_group_by setting determines the threshold RAM consumption for dumping GROUP BY temporary data to the file system. If set to 0 (the default), it is disabled.
> 
> 
> [max_bytes_before_external_sort](https://clickhouse.tech/docs/en/sql-reference/statements/select/order-by/#implementation-details): If there is not enough RAM, it is possible to perform sorting in external memory (creating temporary files on a disk). Use the setting max_bytes_before_external_sort for this purpose. If it is set to 0 (the default), external sorting is disabled. If it is enabled, when the volume of data to sort reaches the specified number of bytes, the collected data is sorted and dumped into a temporary file. After all data is read, all the sorted files are merged and the results are output. Files are written to the /var/lib/clickhouse/tmp/ directory in the config (by default, but you can use the tmp_path parameter to change this setting).

### 3.2. 避免写入时 OOM

而避免写入时OOM ，就不应该在强求 ClickHouse 来实现了。而是需要对业务做一些修改，比如:

* 降低并发线程数；

* 每个 insert 中采用更大的 batch 。

毕竟，我们同样不能苛责向小型 MySQL Server 服务器 一次写入 10万行数据时性能不佳呀。
