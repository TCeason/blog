---
title: mysqldump 源码分析
date: 2020-11-15 16:20:47
categories: DataBase
tags: MySQL
---


## 1. 前言

mysqldump 作为 MySQL 源生支持的逻辑复制工具自从上古时代就已经被DBA广泛使用，时至今日，很多人都依然在用他作为轻量级数据库的备份工具。

虽然在 MySQL 复制工具中 mysqldump 具有 OG 地位，但是不可否认在当代社会，`单线程复制`，`逻辑导出`完全无法 cover 大部分的备份需求场景，比如对于近TB级别的数据如果用mysqldump导出并不落盘的直接写入到其他节点中，可能要以天为时间计量单位去计算一次恢复耗时。试问，谁能忍受我们常用的网站突然某一天弹出一个提示框：`今天网站数据迁移，请后天进行访问。` 

而今天本着对时代的缅怀，我想再好好看看 `mysqldump` 是如何实现的。而快速了解基本实现逻辑，最好的莫过于从他刚刚出生不久时候开始。接下来，我们穿越回上古，看看他年轻时候的样子吧。

<!-- more -->

## 2. 源码分析

### 2.1 源码版本

```
git remote -vv 
origin	https://github.com/TCeason/MySQL-4.1.21.git (fetch)
origin	https://github.com/TCeason/MySQL-4.1.21.git (push)

$ git log -1|more
commit 1a6487c79b109dc944191dbaff1fe077290c1af9

$ ls client/mysqldump.c 
client/mysqldump.c

```

在本版本中，mysqldump 作为一个客户端命令，位于 `client/mysqldump.c`，仅仅有两千行代码。

而有意思的是，在 4.0.10 版本中，只有一千多行代码，也许是因为4.1 相对于 4.0 增加了隔离级别的概念，因此功能相对丰富了。而在 5.7 中已经接近万行了，也许未来会是一个破万的C代码。

从是 4.0 版本到 5.7 版本，他就这么静静的矗立在这里，即使会被时代渐渐抛弃，也依然坚挺。不得不说，respect！！


### 2.3 分析

下面的分析仅仅基于目前 MySQL DB Engine 的顶流 INNODB。

[main方法：](https://github.com/TCeason/MySQL-4.1.21/blob/main/client/mysqldump.c#L2638)

```
int main(int argc, char **argv)
{
  compatible_mode_normal_str[0]= 0;
  default_charset= (char *)mysql_universal_client_charset;
  bzero((char*) &ignore_table, sizeof(ignore_table));

  MY_INIT("mysqldump");
  if (get_options(&argc, &argv))
    ...
 
  if (dbConnect(current_host, current_user, opt_password))
    exit(EX_MYSQLERR);
  
  ...

  if ((opt_lock_all_tables || opt_master_data) &&
      do_flush_tables_read_lock(sock))
    goto err;
  if (opt_single_transaction && start_transaction(sock, test(opt_master_data)))
      goto err;
  
  ...
  
  if (opt_lock_all_tables || opt_master_data)
  {
    if (flush_logs && mysql_refresh(sock, REFRESH_LOG))
      goto err;
    flush_logs= 0; /* not anymore; that would not be sensible */
  }
  if (opt_master_data && do_show_master_status(sock))
    goto err;
  if (opt_single_transaction && do_unlock_tables(sock)) /* unlock but no commit! */
    goto err;

  if (opt_alldbs)
    dump_all_databases();
  else if (argc > 1 && !opt_databases)
  {
    /* Only one database and selected table(s) */
    dump_selected_tables(*argv, (argv + 1), (argc - 1));
  }
  else
  {
    /* One or more databases, all tables */
    dump_databases(argv);
  }
#ifdef HAVE_SMEM
  my_free(shared_memory_base_name,MYF(MY_ALLOW_ZERO_PTR));
#endif
  /*
    No reason to explicitely COMMIT the transaction, neither to explicitely
    UNLOCK TABLES: these will be automatically be done by the server when we
    disconnect now. Saves some code here, some network trips, adds nothing to
    server.
  */
err:
  dbDisconnect(current_host);
  if (!path)
    write_footer(md_result_file);
  if (md_result_file != stdout)
    my_fclose(md_result_file, MYF(0));
  my_free(opt_password, MYF(MY_ALLOW_ZERO_PTR));
  if (hash_inited(&ignore_table))
    hash_free(&ignore_table);
  if (extended_insert)
    dynstr_free(&extended_row);
  if (insert_pat_inited)
    dynstr_free(&insert_pat);
  my_end(0);
  return(first_error);
} /* main */


```

#### 2.3.1.  get_options

主要是用来检查选项设置是否合理。比如当指定 --single-transaction =1 的同时还指定了 --lock-all-tables = 1 这时，由于两者的行为是冲突的，就会直接报错。

> * --lock-all-tables 
> Locks all tables across all databases. This 
>   is achieved by taking a global read lock for the duration of the whole 
> 
> *  --single-transaction
>  single-transaction is specified too (in which case a 
>   global read lock is only taken a short time at the beginning of the dump 
>   don't forget to read about --single-transaction below). In all cases 
>   any action on logs will happen at the exact moment of the dump.dump.
> 
> * --master-data
> This causes the binary log position and filename to be appended to the 
>   output. If equal to 1, will print it as a CHANGE MASTER command; if equal
>    to 2, that command will be prefixed with a comment symbol. 
>   This option will turn --lock-all-tables on, unless 
>   --single-transaction is specified too (in which case a 
>   global read lock is only taken a short time at the beginning of the dump 
>   don't forget to read about --single-transaction below). In all cases 
>   any action on logs will happen at the exact moment of the dump.
>  Option automatically turns --lock-tables off.


其实，从参数解释来看，我们在生产中大多数情况下，应该要设置--master-data=1, --single-transaction =1; 而当 --master-data=1 时，--lock-all-tables 会被该函数自动处理成 !single-transaction。

接下来的流程分析，也将会按照这三个配置进行下去。
```
--master-data = 1;
--single-transaction = 1; 
--lock-all-tables = 0;
```

#### 2.3.2. dbConnect

顾名思义，主要是给 MYSQL 类型的 sock 赋值。并选择。核心内容如下：

```
static MYSQL mysql_connection,*sock=0;

static int dbConnect(char *host, char *user,char *passwd)
{
	...
	mysql_init(&mysql_connection);
	mysql_options(&mysql_connection, MYSQL_SET_CHARSET_NAME, default_charset);
	if (!(sock= mysql_real_connect(&mysql_connection,host,user,passwd,
	       NULL,opt_mysql_port,opt_mysql_unix_port,
	       0)))
	{
	  DBerror(&mysql_connection, "when trying to connect");
	  return 1;
	}
	...
	return 0;
}  
```

#### 2.3.3. FTWRL

```cpp
if ((opt_lock_all_tables || opt_master_data) &&
   do_flush_tables_read_lock(sock))
 goto err;
```

在这一步中，由于需要记录当前的binlog位点，因此，需要对所有的表加锁，主要执行了下面两个 SQL 。

```
FLUSH TABLES；
FLUSH TABLES WITH READ LOCK;
```

直接加 FTWRL 其实就可以获得当前 Server 的一致位点，但是，为了防止当前有 long query运行，导致长时间锁住 Server，先通过 Flush tables 等待 long query 结束，然后快速对整个 DB 加锁。但是，如果刚好在两个查询执行期间有 long query，就无能为力了。

[FLUSH TABLES传送门](https://dev.mysql.com/doc/internals/en/flush-tables.html)
[FTWRL传送门](https://dev.mysql.com/doc/refman/5.7/en/flush.html#flush-tables-with-read-lock)


#### 2.3.4. start_transaction

```cpp
if (opt_single_transaction && start_transaction(sock, test(opt_master_data)))
     goto err;
static int start_transaction(MYSQL *mysql_con, my_bool consistent_read_now)
{
  return (mysql_query_with_error_report(mysql_con, 0,
                                        "SET SESSION TRANSACTION ISOLATION "
                                        "LEVEL REPEATABLE READ") ||
          mysql_query_with_error_report(mysql_con, 0,
                                        consistent_read_now ?
                                        "START TRANSACTION "
                                        "WITH CONSISTENT SNAPSHOT" :
                                        "BEGIN"));
}
```

在代码里面有一个 test(opt_master_data)，其实是这样实现的：

```
#define MY_TEST(a)		((a) ? 1 : 0)
```

因此，执行的sql是将隔离级别设置为RR，并且开启一致性事务。

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION WITH CONSISTENT SNAPSHOT;

```

[事务隔离级别传送门](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)
[一致性事务传送门](https://dev.mysql.com/doc/refman/5.7/en/commit.html)

#### 2.3.5. Refresh logs

```cpp
if (opt_lock_all_tables || opt_master_data)
 {
   if (flush_logs && mysql_refresh(sock, REFRESH_LOG))
     goto err;
   flush_logs= 0; /* not anymore; that would not be sensible */
 }
```

这里主要是执行sql `flush logs`，用来刷新一下binlog，为了更好的区分开始dump的数据点和未来需要继续重放的增量位点。


#### 2.3.6. SHOW MASTER STATUS

```
if (opt_master_data && do_show_master_status(sock))
   goto err;
```

在上一步中，我们给了server全局锁并且拿到了一致性事务视图，为了降低难度更是刷新了binlog文件，就可以放心的获取当前位点状态了。直接执行 `SHOW MASTER STATUS` 获取位点。


#### 2.3.7. UNLOCK TABLES 

```cpp
if (opt_single_transaction && do_unlock_tables(sock)) /* unlock but no commit! */
   goto err;
```

拿到了位点之后，需要将锁释放掉，让 Server 恢复读写，也因此，需要执行 SQL `UNLOCCK TABLES`。而接下来，我们就是要分情况去 dump 数据了。

#### 2.3.8. DUMP

```cpp
if (opt_alldbs)
  dump_all_databases();
else if (argc > 1 && !opt_databases)
{
  /* Only one database and selected table(s) */
  dump_selected_tables(*argv, (argv + 1), (argc - 1));
}
else
{
  /* One or more databases, all tables */
  dump_databases(argv);
}
```

根据命令行传入的参数不同，决定是 dump 全量 DB 还是某个 DB 下的一些表还是某些 DB 下的所有表。

而关键做事情的函数是：

```
static int dump_all_tables_in_db(char *database)
{
  ...
  if (lock_tables)
  {
    DYNAMIC_STRING query;
    init_dynamic_string(&query, "LOCK TABLES ", 256, 1024);
    for (numrows= 0 ; (table= getTableName(1)) ; numrows++)
    {
      dynstr_append(&query, quote_name(table, table_buff, 1));
      dynstr_append(&query, " READ /*!32311 LOCAL */,");
    }
    if (numrows && mysql_real_query(sock, query.str, query.length-1))
      DBerror(sock, "when using LOCK TABLES");
            /* We shall continue here, if --force was given */
    dynstr_free(&query);
  }
  ...
  while ((table= getTableName(0)))
  {
    char *end= strmov(afterdot, table);
    if (include_table(hash_key, end - hash_key))
    {
      numrows = getTableStructure(table, database);
      if (!dFlag && numrows > 0)
	dumpTable(numrows,table);
      my_free(order_by, MYF(MY_ALLOW_ZERO_PTR));
      order_by= 0;
    }
  }
  ...
  if (lock_tables)
    mysql_query_with_error_report(sock, 0, "UNLOCK TABLES");
  return 0;
}

static int dump_selected_tables(char *db, char **table_names, int tables)
{ ... }

```
`dump_all_tables_in_db` 和 `dump_selected_tables` 实现方式大同小异，不过 dump_all_tables_in_db 由于不需要手动指定 DB ，所以，只需要获取当前一致性事务中，存在的 tables 即可，可以从系统表中或者直接 show tables 获取，而 dump_selected_tables 由于有一个手动输入过程，所以，还要去校验一下输入的 tables 是否在 Server 中存在。


他们相同的地方就是，都对某个库下的 tables 加锁，然后获取完整个库下面的 Tables 数据后，再执行`UNLOCK TABLES`。

由于 MySQL 的 DDL 并不支持事务，所以，很不幸，如果你在 dump 过程中做了 DDL，可能会出现一些问题。而在目前的 8.0 版本中，依然没有对 DDL 支持事务。
 
#### 2.3.9. COMMIT

找了通篇，都不会找到的。因为，仅仅是执行了 DQL ，所以在链接断开时，自动去做 commit 是没有问题的。


## 3. 总结

当有了隔离级别以后，可以比较准确的获取数据信息，轻松拿到待同步的增量位点，而 binlog 也是可以反向解析成 SQL 的。因此，虽然 mysqldump 已经老的提不动刀，无法继续冲杀在一线生产环境。但是，他依然为很多异构数据库提供了同步 MySQL 数据的思路和方法。

比如，16年开源的迅速在中国成为时下最火爆的 AP 数据库 [ClickHouse](https://clickhouse.yandex/docs/)，他实现了 [MaterializeMySQL 引擎](https://bohutang.me/3030/12/12/clickhouse-and-friends-mysql-replication-materializemysql/#7-%E7%9B%B8%E5%85%B3%E5%8D%9A%E6%96%87)，可以实现对 MySQL 5.7 和 MySQL 8.0 的全量 + 增量同步。目前这个功能的使用者和体验者还不是很多，但是预估在不久的未来，会成为一个企业级可用的功能。
