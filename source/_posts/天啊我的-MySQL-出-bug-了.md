---
title: 天啊我的 MySQL 出 bug 了
date: 2020-06-13 16:52:40
categories: DataBase
tags: MySQL
---

有这么一天，我们吃着火锅，唱着歌，突然就被手机铃声给劫了。一接电话

问曰：咋回事捏？

骂曰：看看你这破数据库，error log 里面居然有 bug 信息。赶紧回来，不然别想混了！！！

得，骑上我心爱的小毛驴飞过去处理吧。

<!-- more -->

其实，数据库也是一种软件，而软件怎么会没有 bug 呢？曾记得小时候的武侠中都是这样的，独孤九剑，没有固定的招式，而想要学会太极拳就要忘记太极拳。

那么怎么让软件没有 bug 呢？或许就是消灭这个软件吧。那么什么样的软件 bug 少呢？或许就是使用者极少吧。

好了，言归正传，当我们真的遇到 MySQL 日志中打印了一堆 bug 信息该怎么解决呢？

## 1. 这是一段 bug

当我们看到 error log 中出现下面的描述，一般是使用方法错误，或者真的遇到了 bug。这个时候，只需要根据 stack 信息进行 trace，就能进一步定位问题。

```txt
03:51:06 UTC - mysqld got signal 11 ;
This could be because you hit a bug. It is also possible that this binary
or one of the libraries it was linked against is corrupt, improperly built,
or misconfigured. This error can also be caused by malfunctioning hardware.
We will try our best to scrape up some info that will hopefully help
diagnose the problem, but since we have already crashed, 
something is definitely wrong and this may fail.
Please help us make Percona Server better by reporting any
bugs at https://bugs.percona.com/

key_buffer_size=8388608
read_buffer_size=131072
max_used_connections=0
max_threads=153
thread_count=0
connection_count=0
It is possible that mysqld could use up to 
key_buffer_size + (read_buffer_size + sort_buffer_size)*max_threads = 69061 K  bytes of memory
Hope that's ok; if not, decrease some variables in the equation.

Thread pointer: 0x0
Attempting backtrace. You can use the following information to find out
where mysqld died. If you see no messages after this, something went
terribly wrong...

```

你看，MySQL 告诉你了，如果真的啥都没有，那么可就糟糕透了。万幸的是，我们继续往下看，能看到一些东西。

```txt
stack_bottom = 0 thread_stack 0x40000
./bin/mysqld(my_print_stacktrace+0x3b)[0x8f7eab]
./bin/mysqld(handle_fatal_signal+0x49a)[0x66d55a]
/lib/x86_64-linux-gnu/libpthread.so.0(+0x11390)[0x7f6f44fc2390]
./bin/mysqld(_ZN9MYSQL_LOG17generate_new_nameEPcPKc+0x9d)[0x657c4d]
./bin/mysqld(_ZN9MYSQL_LOG26init_and_set_log_file_nameEPKcS1_13enum_log_type10cache_type+0x57)[0x657d07]
./bin/mysqld(_ZN13MYSQL_BIN_LOG11open_binlogEPKcS1_10cache_typembbbbP28Format_description_log_event+0x7b)[0x89a22b]
./bin/mysqld(_Z11mysqld_mainiPPc+0x1bab)[0x5acbfb]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7f6f44366830]
./bin/mysqld(_start+0x29)[0x59e659]

```

好家伙，虚惊一场啊。有了 bug 果还有 bug 因，接下来，我们只需要沿着脉络去探索就好了。或许问题不大。

## 2. 理清 bug 迎难而上吧

对于一些”高玩“们，看懂这段 bug info 并不是什么难事，而对于我们，难如登天啊有木有，一段段的内存地址，密密麻麻的英文描述。所以，这个时候，我们需要什么，需要工具！！


我相信如果你能看到这里，已经通过 {% post_link 空有-MySQL-Server-如何成为高玩 空有-MySQL-Server-如何成为高玩 %} 将 debug 后的 mysqld 加入到了机器的 PATH 变量中。

我们继续以上面的 bug info 为例子，看看如何让他变成代码具体位置。

```shell
$ mysqld_path=$(which mysqld)

$ nm  -n $mysqld_path > /tmp/mysqld.sym

```

**注意**：编译的 debug 版本，可以将 `which mysqld` 替换为 `which mysqld-debug`


```shell
# resolve_stack_dump 位于 MySQL basedir/bin
# 我把上面的 bug info 存到了 /tmp/mysqld.stack 中
$ resolve_stack_dump -s /tmp/mysqld.sym -n /tmp/mysqld.stack

```


当解析完毕后，我们可以看到关键部分已经发生了变化，更具体的打印了函数名，而我们又怎能满足于函数名呢？不要求具体列位置，最起码给个行定位吧大哥。

不要着急，真相就在眼前。

```txt
03:51:06 UTC - mysqld got signal 11 ;
This could be because you hit a bug. It is also possible that this binary...
Thread pointer: 0x0
Attempting backtrace. You can use the following information to find out
where mysqld died. If you see no messages after this, something went
terribly wrong...
stack_bottom = 0 thread_stack 0x40000
0x8f7eab my_print_stacktrace + 59
0x66d55a handle_fatal_signal + 1178
0x7f6f44fc2390 _end + 1136550664
0x657c4d _ZN9MYSQL_LOG17generate_new_nameEPcPKc + 157
0x657d07 _ZN9MYSQL_LOG26init_and_set_log_file_nameEPKcS1_13enum_log_type10cache_type + 87
0x89a22b _ZN13MYSQL_BIN_LOG11open_binlogEPKcS1_10cache_typembbbbP28Format_description_log_event + 123
0x5acbfb _Z11mysqld_mainiPPc + 7083
0x7f6f44366830 _end + 1123592104
0x59e659 _start + 41
```

我们只需要稍稍改动上面的命令，就好了

> resolve_stack_dump -s /tmp/mysqld.sym -n /tmp/mysqld.stack | c++filt

```txt
Thread pointer: 0x0
Attempting backtrace. You can use the following information to find out
where mysqld died. If you see no messages after this, something went
terribly wrong...
stack_bottom = 0 thread_stack 0x40000
0x8f7eab my_print_stacktrace + 59
0x66d55a handle_fatal_signal + 1178
0x7f6f44fc2390 _end + 1136550664
0x657c4d MYSQL_LOG::generate_new_name(char*, char const*) + 157
0x657d07 MYSQL_LOG::init_and_set_log_file_name(char const*, char const*, enum_log_type, cache_type) + 87
0x89a22b MYSQL_BIN_LOG::open_binlog(char const*, char const*, cache_type, unsigned long, bool, bool, bool, bool, Format_description_log_event*) + 123
0x5acbfb mysqld_main(int, char**) + 7083
0x7f6f44366830 _end + 1123592104
0x59e659 _start + 41

```

## 3. 怎么定位这个锤儿 bug 呦

从这个报错来看，在启动过程中，要打开 binlog 文件，而在打开文件过程中失败了。这种报错 99.9% 都是文件系统问题，而 99% 可能是文件系统磁盘空间满了，可以通过一些命令去排查：

```shell
$ df -Pih
文件系统       Inode 已用(I) 可用(I) 已用(I)% 挂载点
udev            2.5M     582    2.5M       1% /dev
tmpfs           2.5M    1.2K    2.5M       1% /run
/dev/nvme0n1p2   15M    1.8M     14M      12% /
...

```

## 4. 最重要的事儿

想必大家都听说过南辕北辙的故事，大概就是要往一边去，然后，从相反的方向出发。而判断 bug 到底出在哪里更是如此，我们一定要选对路，一直走到头。

那么，无论你是否能够坚持，选对路很重要，毕竟，rootcause 可能就在路口等你。

对于这次 debug ，选对路就是选对 mysqld 版本。记得，一定要用报错的 mysqld 去分析问题。否则，就是南辕北辙。毕竟，谁能准确记得每个不同小版本之间，bug info 涉及的代码没有被人修改呢？

