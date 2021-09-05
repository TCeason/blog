---
title: 用 OceanBase 实现 Hello World
date: 2021-09-04 09:15:20
categories: DataBase
tags:
---

首先我们来看一下什么是 `OceanBase`, 下面是来自官方的定义:

> [OceanBase](https://github.com/oceanbase/oceanbase) 社区版是一款开源分布式 HTAP（Hybrid Transactional/Analytical Processing）数据库管理系统，具有原生分布式架构，支持金融级高可用、透明水平扩展、分布式事务、多租户和语法兼容等企业级特性。OceanBase 内核通过大规模商用场景的考验，已服务众多行业客户，现面向未来持续构建内核技术竞争力。

而作为一个数据库疯狂热爱者，在看到它的那一刻直接下载了一份源码。而在经过一番折腾之后，终于让 OceanBase 老老实实的说了一句 Hello World。接下来跟大家分享一下我是如何做到的。

```sql
> SELECT 'Hello World';

```

本章完~ 哈哈，开个小玩笑。接下来让我们进入正文。

## 1. 如何向 OceanBase 贡献代码

<!-- more -->

众所周知，OB 在 2021 年儿童节宣布开源。或许准备稍有仓促导致很多文档介绍都没有跟上，因此如果希望给 OB 贡献代码相对就比较困难了。

而函数部分功能目前看来并不是特别多，大家还是有机会从这一块入手并且向着 PMC 前进的。而作为项目启动的第一步，我们依然沿袭各个语言的经典项目 `Hello World`。

那么，我们就开始不那么正式的给 OB 实现一个 hello_world() 函数吧。借此来熟悉一些 SQL 相关的模块。

### 1.1. 代码模块

 OB 作为一个分布式数据库，代码结构是较为庞大的。而我们需要的就是抽丝剥茧的能力，摒弃掉无关部分，找到我们需要的几个目录和文件。

熟悉 MySQL 的朋友应该都比较了解：如果需要向 MySQL 支持一个函数或者找一下某个已有函数的源码实现，需要去 [item_create.h](https://github.com/mysql/mysql-server/blob/8.0/sql/item_create.h) & [item_create.cpp](https://github.com/mysql/mysql-server/blob/8.0/sql/item_create.cc) 找它的原生定义。

而 OB 同样有类似的门户，在 OB 中严格的按照 SQL 的生命周期命名了目录。因此，我们会看到这样的结构。

```
➜  ~/database/source-code/oceanbase👆 git:(hello_world) ⚡ tree -L 1 -d src/sql
src/sql
├── CMakeFiles
├── code_generator
├── dtl
├── engine
├── executor
├── monitor
├── optimizer
├── parser
├── plan_cache
├── privilege_check
├── resolver
├── rewrite
└── session

13 directories

```

我们这次要修改的主体部分在 [engine](https://github.com/oceanbase/oceanbase/tree/master/src/sql/engine)目录中。

```
➜  ~/database/source-code/oceanbase👆 git:(hello_world) ⚡ tree -L 1 -d src/sql/engine 

src/sql/engine
├── aggregate
├── basic
├── cmd
├── connect_by
├── dml
├── expr
├── join
├── pdml
├── prepare
├── px
├── recursive_cte
├── sequence
├── set
├── sort
├── subquery
├── table
├── user_defined_function
└── window_function

18 directories

```

直到这里，我们还是没有看到熟悉的 func。不要着急，因为在 OB 眼中所有的函数本质都是表达式。因此，他将其命名为 [expr](https://github.com/oceanbase/oceanbase/tree/master/src/sql/engine/expr)。
举个现成的例子，比如我们都熟悉的 abs() 函数，就安安静静的在 expr 目录中。

```
➜  ~/database/source-code/oceanbase/src/sql/engine/expr👆 git:(hello_world) ⚡ ls |grep ob_expr_abs.cpp
ob_expr_abs.cpp

```

因此，我们确实找到了 `expr(表达式)` 的老巢。而接下来要做的就是搭一个架子出来，这个架子能够嵌入到 OB 目前已有的支持 expr 框架中。简单来说就是让 OB 能够认识函数的名字。下面是一些简单的介绍:

> * [src/sql/CMakeLists.txt](https://github.com/oceanbase/oceanbase/blob/master/src/sql/CMakeLists.txt) : 它是那么的明显以至于我都无法忽视它的存在。
> * [src/sql/parser/ob_item_type.h](https://github.com/oceanbase/oceanbase/blob/master/src/sql/parser/ob_item_type.h) : 当一条 SQL 通过了词法解析，获取每个 TOKEN 后，parser 层就需要做一些工作, 因此, 我们需要去 parser 目录中找一下有没有对应的文件, 经过一番筛选，确认是它。因为在[该行注释](https://github.com/oceanbase/oceanbase/blob/master/src/sql/parser/ob_item_type.h#L417)中已经非常清晰的告知了我们他的用途。
> * [src/sql/engine/expr/ob_expr_eval_functions.cpp](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_eval_functions.cpp) : 在之前的一些分享中已经有提到，OB 有新老 SQL 引擎的区分，而如果需要注册成为新引擎能识别的 expr 则需要在[g_expr_eval_functions](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_eval_functions.cpp#L260) 数组中进行 append。
> * [deps/oblib/src/lib/ob_name_def.h](https://github.com/oceanbase/oceanbase/blob/master/deps/oblib/src/lib/ob_name_def.h) : 这里面定义的是所有的关键字, 函数等等, 所有可以被日志使用的宏定义。还是以我们熟悉的 abs() 为例子，他就那么不显山不漏水的躺在[这里](https://github.com/oceanbase/oceanbase/blob/master/deps/oblib/src/lib/ob_name_def.h#L363)。
> * [src/sql/engine/expr/ob_expr_operator_factory.cpp](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_operator_factory.cpp) : 从文件命名来看，是所有 表达式算子（expr operator）的工厂，所以，理论上需要在这里注册函数，被 OB 所识别。从 [这里](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_operator_factory.cpp#L414) 的注释可以看到更详细的信息。

**注意:**
核心改动在于:
> * [src/sql/CMakeLists.txt](https://github.com/oceanbase/oceanbase/blob/master/src/sql/CMakeLists.txt)
> * [src/sql/parser/ob_item_type.h](https://github.com/oceanbase/oceanbase/blob/master/src/sql/parser/ob_item_type.h)
> * [src/sql/engine/expr/ob_expr_eval_functions.cpp](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_eval_functions.cpp)
> * [deps/oblib/src/lib/ob_name_def.h](https://github.com/oceanbase/oceanbase/blob/master/deps/oblib/src/lib/ob_name_def.h)
> * [src/sql/engine/expr/ob_expr_operator_factory.cpp](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_operator_factory.cpp)
> * [src/sql/engine/expr](https://github.com/oceanbase/oceanbase/tree/master/src/sql/engine/expr)

## 2. Just Do It

我们直接在 [src/sql/engine/expr](https://github.com/oceanbase/oceanbase/tree/master/src/sql/engine/expr) 中增加一组 C++ Class `ob_expr_hello`

因为本函数希望实现形式是:

```
HELLO_WORLD()

```

可以确定这是一个入参为 0 且返回值类型为 String 的表达式. 因此, 我们进一步确定头文件中只需要这些函数, 并且作为 [ObStringExprOperator](https://github.com/oceanbase/oceanbase/blob/master/src/sql/engine/expr/ob_expr_operator.h#L1452) 的子类:

```cpp
class ObExprHELLO : public ObStringExprOperator {
public:
  explicit ObExprHELLO(common::ObIAllocator& alloc);
  virtual ~ObExprHELLO();
  virtual int calc_result_type0(ObExprResType& type, common::ObExprTypeCtx& type_ctx) const;
  virtual int calc_result0(common::ObObj& result, common::ObExprCtx& expr_ctx) const;
  static int eval_hello(const ObExpr& expr, ObEvalCtx& ctx, ObDatum& expr_datum);
  virtual int cg_expr(ObExprCGCtx& op_cg_ctx, const ObRawExpr& raw_expr, ObExpr& rt_expr) const override;
private:
  DISALLOW_COPY_AND_ASSIGN(ObExprHELLO);
};

```

其中 calc_result_type0 是在真实计算前对返回值的类型做一次定义，以此来进行计算时优化。但是，在本例子中，暂时不会使用到这个功能，因为我们没有传入参数，返回的也是一个固定类型的字符串。

calc_result0 则是真正在计算时需要做的一些算子实现，而在本例子中就比较轻松，只需要用 ObString 这个类来初始化一段 char "Hello World~" 即可。

eval_hello 和 cg_expr 则作为新的 SQL 引擎实现，与 calc_result0 相比，提前针对 final result 进行了参数转换并且做了一些兜底机制。如果感兴趣，可以看看他们具体被调用的位置，而通过这几个函数，我们就把整个表达式的架子搭起来了。

## 3. 来吧展示

```SQL
>select HELLO_WORLD();
+---------------+
| HELLO_WORLD() |
+---------------+
| Hello World~  |
+---------------+
1 row in set (0.00 sec)

```


## 4. 测试

OB 的测试和 MySQL 的 mtr 测试基本一致。位于：

https://github.com/oceanbase/oceanbase/blob/master/test/mysql_test/test_suite/expr/t/
https://github.com/oceanbase/oceanbase/tree/master/test/mysql_test/test_suite/expr/r/mysql

可以在本次的 commit 中查看整体代码。

## 5. 总结

整体来看，一个表达式的书写没有那么复杂。

> 1. 可以被编译。
> 2. 可以被 OB 识别。
> 3. 实现 expr 内部逻辑。

在这里附上 [完整代码](https://github.com/TCeason/oceanbase/commit/ba3d88ccbf01d4c7dc77d3cff7e1c07485d2f041) 感兴趣的朋友可以看一看，也欢迎给 [OB](https://github.com/oceanbase/oceanbase) 点 Star 哦。

