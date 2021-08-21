---
title: 更方便的使用git
date: 2021-08-21 12:26:09
categories: Command
tags: Git
---


在我们日常学习和工作中， [git](https://git-scm.com/) 已经逐渐的和程序员绑定到了一起。

关于 git ，大家是这样去定义的：

```
Git is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency.

Git is easy to learn and has a tiny footprint with lightning fast performance. It outclasses SCM tools like Subversion, CVS, Perforce, and ClearCase with features like cheap local branching, convenient staging areas, and multiple workflows.

```

<!-- more -->

## 1. 简单的使用方法

目前来看，一套命令就可以完成一次 pr （代码的提交）。

```powershell
$ git add .
$ git commit -m 'init commit'
$ git push 

```

而更简单的一行命令就可以看到一个文件的所有提交日志，我们以 ClickHouse 任意一个文件为例：

```powershell
➜  ~/database/source-code/ClickHouse👆 git:(21.1.3.32) ⚡ git log src/Core/Settings.h

commit eaef015c6b3c3f86c43b98ba6fcc305779bac90c
Author: robot-clickhouse <robot-clickhouse@yandex-team.ru>
Date:   Thu Jan 14 11:53:35 2021 +0300

    Backport #18981 to 21.1: Disable optimize out of any

commit 7e9120b34fbf357c578c8035e2abeab2ec5c9d94
Merge: aa8d4eec69 8387ac86ef
Author: Alexey Milovidov <milovidov@yandex-team.ru>
Date:   Sun Jan 10 04:04:47 2021 +0300

    Merge branch 'master' into chenziliang/feature/wildcard-dynamic-columns

```

## 2. 要优雅

或许平时觉得这些就足够应付工作了。但是，做事要优雅，我们希望一眼就能看穿 “前世今生”。

因此单独介绍两个小命令：

> git log --graph
> git blame -L

### 2.1. git log --graph

我们还是以上面的 `Settings.h` 为例子，当单纯的看 git log 时，确实可以看到每一次提交，但是，如何找到每个提交的关系呢？是在哪个分支之下呢？

在使用命令之前，请让我们先了解到底它在做什么

> [--graph](https://git-scm.com/docs/git-log#Documentation/git-log.txt---graph)
> Draw a text-based graphical representation of the commit history on the left hand side of the output. This may cause extra lines to be printed in between commits, in order for the graph history to be drawn properly. Cannot be combined with --no-walk.
> 
> This enables parent rewriting, see History Simplification above.

简单来说，会生成一个点线图。好了，话不多说，看结果：

```
➜  ~/database/source-code/ClickHouse👆 git:(21.1.3.32) ⚡ git log --graph  src/Core/Settings.h
* commit eaef015c6b3c3f86c43b98ba6fcc305779bac90c
| Author: robot-clickhouse <robot-clickhouse@yandex-team.ru>
| Date:   Thu Jan 14 11:53:35 2021 +0300
| 
|     Backport #18981 to 21.1: Disable optimize out of any
|   
*   commit 7e9120b34fbf357c578c8035e2abeab2ec5c9d94
|\  Merge: a7c350adda a3d19fa64d
| | Author: Alexey Milovidov <milovidov@yandex-team.ru>
| | Date:   Sun Jan 10 04:04:47 2021 +0300
| | 
| |     Merge branch 'master' into chenziliang/feature/wildcard-dynamic-columns
| | 
| * commit a3d19fa64df8fc70fc6b6e5a0bc434721ae571e7
| | Author: Amos Bird <amosbird@gmail.com>
| | Date:   Fri Jan 8 12:28:09 2021 +0800
| | 
| |     Correctly override default settings remotely
| | 
| * commit 0260953a4785ac1daa6b3c17d6dc555ce59f8caf
| | Author: Amos Bird <amosbird@gmail.com>
| | Date:   Wed Jan 6 17:18:48 2021 +0800
| | 
| |     better
| | 
| * commit a157a5b3b3c4dd26d208cdb65c35bf579b4bcb52
| | Author: Amos Bird <amosbird@gmail.com>
| | Date:   Mon Jan 4 12:40:48 2021 +0800
| | 
| |     add max_partitions_to_read setting
| | 

```

很遗憾，这个图形非常让人眩晕，有时可能还会是这样：

```
| * | | | | commit 85c1bc1253fef760b6f329381b694eed3e122e6d
| |\| | | | Merge: 4bfcb2fc16 34e4ace029
| | | | | | Author: Alexander Kuzmenkov <akuzm@yandex-team.ru>
| | | | | | Date:   Mon Dec 21 10:46:21 2020 +0300
| | | | | | 
| | | | | |     Merge remote-tracking branch 'origin/master' into tmp
| | | | | |   
| | * | | |   commit 34e4ace02948cd3d3bc5d43dba9a8e6c05249c85
| | |\ \ \ \  Merge: e4433157e7 a0f7a12c4f
| | | | | | | Author: alexey-milovidov <milovidov@yandex-team.ru>
| | | | | | | Date:   Fri Dec 18 18:00:34 2020 +0300
| | | | | | | 
| | | | | | |     Merge pull request #17525 from ClickHouse/null-as-default-default
| | | | | | |     
| | | | | | |     Attempt to make NULL as default by default
| | | | | | |   
| | | * | | |   commit a0f7a12c4fa40bf04ef9ffedc0c8892fbf9d08a3
| | | |\ \ \ \  Merge: 20e56da8ac cb91d64fe7
| | | | | |_|/  Author: Alexey Milovidov <milovidov@yandex-team.ru>
| | | | |/| |   Date:   Fri Dec 18 08:09:35 2020 +0300
| | | | | | |   
| | | | | | |       Merge branch 'master' into null-as-default-default

```

但是，当我们理解了每个符号的意思，也就可以比较清晰的理顺其中的关系。

> `|` 表示分支向前
> `*` 表示一个 COMMIT ID，至于要管*在哪一条主线上
> `/` 表示从当前分支分叉，比如执行了 `git checkout -b 或者 git switch -c`
> `\` 表示被合并

因此
*  `|/` 表示从 `|` 所在分支切出来一个新的分支。
* `|\` 表示从当前的 `|` 分支，合并到并向的 `|` 分支。

当我们独立操作两次之后，反而会对这些乱七八糟的毛线说一句 `LOVE YOU`。

## 2.2. git blame -L

同理，操作之前，请先明白你在做什么。废话不多说，上文档。

> [git blame](https://git-scm.com/docs/git-blame)
> git-blame - Show what revision and author last modified each line of a file

blame 含有责备的意思，实际上，这个命令确实可以用来责任。因为他可以显示每一行最后一次的 commit 信息。

比如我们下面的例子所示，展示了 `Settings.h` 从第 20 -30 行，每一行最新的 commit id。里面包含了修改时间，作者以及修改记录摘要。

```powershell
➜  ~/database/source-code/ClickHouse👆 git:(21.1.3.32) ⚡ git blame -L 20,30 src/Core/Settings.h
7d9f3035995 dbms/include/DB/Interpreters/Settings.h (Alexey Milovidov     2012-03-05 00:09:41 +0000 20) {
bd4d8a6766e dbms/src/Interpreters/Settings.h        (Vitaliy Lyudvichenko 2018-05-17 19:01:41 +0300 21) class IColumn;
8277e9d8f12 dbms/src/Core/Settings.h                (Vitaly Baranov       2019-04-19 02:29:32 +0300 22) 
f5adbceed2d dbms/src/Interpreters/Settings.h        (Alexey Milovidov     2018-06-03 23:39:06 +0300 23) 
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 24) /** List of settings: type, name, default value, description, flags
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 25)   *
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 26)   * This looks rather unconvenient. It is done that way to avoid repeating settings in different places.
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 27)   * Note: as an alternative, we could implement settings to be completely dynamic in form of map: String -> Field,
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 28)   *  but we are not going to do it, because settings is used everywhere as static struct fields.
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 29)   *
56665a15f7f src/Core/Settings.h                     (Vitaly Baranov       2020-07-20 12:57:17 +0300 30)   * `flags` can be either 0 or IMPORTANT.

```

## 2.3. 组合拳？？

其实，无论是 git log --graph 还是 git blame -L 都是比较单独的手段。

但是当他们结合起来，是否可以发挥妙用呢？我想聪明的你已经想出了答案。

