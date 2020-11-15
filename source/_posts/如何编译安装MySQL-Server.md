---
title: 如何编译安装MySQL Server
date: 2020-06-13 16:20:47
categories: DataBase
tags: MySQL
---

众所周知，[MySQL](https://www.mysql.com/)依然是[最流行的开源数据库](https://db-engines.com/en/ranking)没有之一。那么如何使用他，如何揭开他流行的奥秘呢？这一切的一切可能要从最基础的编译安装开始说起。
	
<!-- more -->

## 1. 编译 release 版本

```shell
$ git clone https://github.com/TCeason/tokudb.git mysql
$ cd mysql
$ git checkout releases-tags
$ git submodule init
$ git submodule update
$ mkdir build; cd build
$ cmake ..\
  -DCMAKE_BUILD_TYPE=RelWithDebInfo\
  -DBUILD_CONFIG=mysql_release\
  -DFEATURE_SET=community\
  -DWITH_EMBEDDED_SERVER=OFF\
  -DTOKUDB_VERSION=7.5.6\
  -DBUILD_TESTING=OFF\
  -DWITHOUT_ROCKSDB=ON\
  -DWITH_BOOST=../extra/boost/boost_1_59_0.tar.gz\
  -DCOMPILATION_COMMENT="Study MySQL build $(date +%Y%m%d.%H%M%S.$(git rev-parse --short HEAD))"\
  -DCMAKE_INSTALL_PREFIX=  # MySQL basedir 路径
  # cmake 的时候，如果没有依赖项需要自行安装
$ make -jn # n的数量取决于 CPU Core 数，如果 make -j报错，尝试 make
$ make install

```

## 2. 编译 Debug 版本

```shell
$ git clone https://github.com/TCeason/tokudb.git mysql
$ cd mysql
$ git checkout releases-tags
$ git submodule init
$ git submodule update
$ mkdir build
$ cd build
$ cmake ..\
  -DCMAKE_BUILD_TYPE=Debug\
  -DBUILD_CONFIG=mysql_release\
  -DFEATURE_SET=community\
  -DWITH_EMBEDDED_SERVER=OFF\
  -DTOKUDB_VERSION=7.5.6\
  -DBUILD_TESTING=OFF\
  -DWITHOUT_ROCKSDB=ON\
  -DWITH_BOOST=../extra/boost/boost_1_59_0.tar.gz\
  -DCMAKE_INSTALL_PREFIX=  # MySQL basedir 路径
$ make -jn # n的数量取决于 CPU Core 数，如果 make -j报错，尝试 make
$ make install

```

随着 MySQL 版本的不断更新，编译参数可能会发生变动，当出现一些问题时，还是需要参考下面的信息：

https://dev.mysql.com/doc/refman/8.0/en/installing.html

https://github.com/TCeason/tokudb

https://github.com/XeLabs/tokudb/wiki

如果有对于 Percona Server Backport 感兴趣的朋友，欢迎讨论。
