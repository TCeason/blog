---
title: 空有 MySQL Server 如何成为高玩
date: 2020-06-13 16:26:24
categories: DataBase
tags: MySQL
---

通过 {% post_link 如何编译安装MySQL-Server 如何编译安装MySQL-Server %}，我们终于了解了如何进行对 MySQL Server 进行编译安装。那么如何进一步了解成为真正的 MySQL ”高玩“ 呢？

<!-- more -->

这个问题，我相信，网络上有很多答案。但是，我认为，真相只有一个～，自己动手编译 debug 版本，然后，进行 gdb 来判断。你可能觉得，这样的方式过于”老土“，而且非常不友好，毕竟还要掌握gdb，多一种学习路径，对于初入者的难度无疑是几何级增长。

没关系，MySQL 提供了一个贴心的功能，可以让我们通过类似阅读日志的方式读一下 Server 和 Client 到底在做什么。

>注意： 
>  1) 接下来的方法一定要在编译 `debug` 版本后在进行使用哦。
>     接下来的方法一定要在编译 `debug` 版本后在进行使用哦。
>     接下来的方法一定要在编译 `debug` 版本后在进行使用哦。
>  重要的事情说三遍，嘻嘻。 
>
>  2) 需要自行将编译好的 MySQL binary 包加入到使用主机的`PATH`变量中。

## 1. debug: mysqld

```shell
# Session1: 
$ mysqld-debug --initialize-insecure --debug=d,info,error,query,general,where:O,/tmp/mysqld.trace
# Session2:
$ tail -f /tmp/mysqld.trace
```

**Note:** 不做任何 defaults-file 设置的话，datadir 位于 $basedir/data

## 2. debug: mysql client

```shell
# Session1:
$ mysql -uusername --debug=d:t:O,/tmp/client.trace
# Session2:
$tail -f /tmp/client.trace
```

加油吧少年，欢乐时光开始啦。（手动滑稽）

如果还是有些许不明白可以参考官方链接，不过个人建议还是先折磨一下自己哦

https://dev.mysql.com/doc/refman/8.0/en/porting.html

