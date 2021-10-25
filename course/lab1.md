## Lab 1: TinySQL 与 Parser

### 1.1 简介

从现在开始，我们将开始 TinySQL 的实验课程。在本次课程中，你将：

1. 了解 TinySQL 的基本框架
2. 学习部分 SQL 语法
3. 学习 TinySQL 中 Parser 的实现

你可以通过如下命令获取实验代码并切换到该实验对应的分支：

```bash
git clone git://github.com/xxxxx/tinysql
cd tinysql
git checkout lab1
```

强烈建议你创建一个新的分支来完成实验。你可以任意命名该分支，这里我们用 `dev` 来代表新分支。你可以通过下面的命令来完成该操作。

```bash
git checkout -b dev
```

### 1.2 TinySQL 基本框架

TinySQL 是以 [TiDB](https://github.com/pingcap/tidb) 为蓝本的教学向分布式关系型数据库。它的架构与 TiDB 基本保持一致。从系统框架上来讲，最终实现完成的 TinySQL 可以被分为三层：

- MySQL 协议层
- SQL 层
- KV 层

MySQL 协议层主要负责接收 MySQL Client 的请求，解析 MySQL 协议，并将其转换为 TinySQL 能够处理的格式。将解析后的请求发给 SQL 层并接收最后的处理结果，将处理结果包装为符合 MySQL 协议的格式，最终返回给 MySQL Client。

TinySQL 本质上可以理解为一个 `server`。它通常以 MySQL 协议对外提供服务。与其他 `server` 相似，TinySQL 需要监听端口、建立并管理连接、然后为其提供服务。这些功能也都在 MySQL 协议层中。简单来讲，MySQL 协议层就是 TinySQL 与用户的接口，负责对外提供服务。

SQL 层是 TinySQL 的核心内容，它包含了从解析 SQL 语句、制定并生成查询计划到通过执行器执行。最后将结果返回给 MySQL 协议层。这一部分是整个 TinySQL 实验的重点内容，我们将会通过实验逐步让大家了解并实现 SQL 层的代码。

KV 层其本质是存储(例如 TinyKV、TiKV)和 SQL 层之间的一层抽象。它为 SQL 层提供了使用存储的接口，而不需要关心底层存储的实现。

#### 1.2.1 框架代码指南

##### 1.2.1.1 `package tidb-server`

`tidb-server/main.go:main()` 是整个项目的入口函数。该函数会进行一系列的初始化工作，例如，`config`、`log`等通用组件的初始化。然后调用`createServer()`初始化一个 `Server` 实例。`server.Server(server/server.go)`定义了一个 MySQL 协议 server。最终通过 `Server.Run()`方法将控制权交给了 `Server`。

##### 1.2.1.2 `package server`

`package server` 实现了 MySQL 协议层。其主要功能有：

1. 监听端口、建立并管理连接
2. 解析协议，向 SQL 层传递指令或查询

`Server.Run()` 方法会通过

```go
go s.startNetworkListener(s.listener, errChan)
```

启动一个 [goroutine](https://golangdocs.com/goroutines-in-golang) 异步监听端口。当监听到符合 MySQL 协议的请求时，会通过

```go
clientConn := s.newConn(conn)
go s.onConn(clientConn)
```

新建一个连接并通过 `goroutine` 异步处理链接。

`Server.onConn()`主要包含两个方面的内容：

1. 验证并建立连接
2. 进入到处理查询的主循环

`clientConn.handshake()`会对请求进行验证，并对连接进行初始化。

随后，通过`clientConn.Run()`进入主循环。

主循环中通过`cc.Status`来控制连接状态。通常来讲，`Status`会在 [dispatching] <=> [reading] 之间发生变化。但当某些事件发生时，`server` 可能会通过将状态设置为特殊值来通知此连接，例如，`kill`。

主循环中会通过

```go
!atomic.CompareAndSwapInt32(&cc.status, connStatusDispatching, connStatusReading)
```

来判断是否出现特殊事件。

然后，主循环会通过 `clientConn.dispatch()`来处理查询或指令。

这里我们主要关注`clientConn.handleQuery()`，它将用来处理 SQL 查询。

到此为止，MySQL 协议层的逻辑就基本结束。MySQL 协议层的内容较浅但涉及内容繁杂，因此，框架代码中提供了 MySQL 协议层的一个简单实现，不将其放入实验内容。

#### 1.2.2 SQL 语法

在这一部分我们将介绍利用 SQL 在逻辑上表示以及处理数据。你可以在阅读的同时使用 [A Tour of TiDB](https://tour.pingcap.com/) 来动手操作数据库。 

对于数据库来说，关键的问题可能就是如何表示数据以及如何处理这些数据了。在关系型数据库中，数据是以“表”的形式存在的，例如在  [A Tour of TiDB](https://tour.pingcap.com/) 中，有一个 person 表：

| number |  name | region |   birthday |
| -----: | ----: | -----: | ---------: |
|      1 |   tom |  north | 2019-10-27 |
|      2 |   bob |   west | 2018-10-27 |
|      3 |   jay |  north | 2018-10-25 |
|      4 | jerry |  north | 2018-10-23 |

该表一共有 4 行，每行有 4 列信息，这样每一行就成为了一个最小的完整信息单元，利用这样的一个或多个表，我们便可以在上面做出各种操作以得出想要的信息。

在表上最简单的操作便是直接输出全部行了，例如：

```sql
TiDB> select * from person;
+--------+-------+--------+------------+
| number | name  | region | birthday   |
+--------+-------+--------+------------+
| 1      | tom   | north  | 2019-10-27 |
| 2      | bob   | west   | 2018-10-27 |
| 3      | jay   | north  | 2018-10-25 |
| 4      | jerry | north  | 2018-10-23 |
+--------+-------+--------+------------+
4 row in set (0.00 sec)
```

还可以指定只输出需要的列，例如：

```sql
TiDB> select number,name from person;
+--------+-------+
| number | name  |
+--------+-------+
| 1      | tom   |
| 2      | bob   |
| 3      | jay   |
| 4      | jerry |
+--------+-------+
4 row in set (0.01 sec)
```

当然，有的时候可能我们只对满足某些条件的行感兴趣，例如我们可能只关心位于 north 的人：

```sql
TiDB> select name, birthday from person where region = 'north';
+-------+------------+
| name  | birthday   | 
+-------+------------+
| tom   | 2019-10-27 | 
| jay   | 2018-10-25 | 
| jerry | 2018-10-23 | 
+-------+------------+
3 row in set (0.01 sec)
```

通过 where 语句以及各种条件的组合，我们可以只得到满足某些信息的行。

有些时候，我们还需要一些概括性的数据，例如表里满足某个条件的一共有多少行，这个时候我们需要聚合函数来统计概括后的信息：


```sql
TiDB> select count(*) from person where region = 'north';
+----------+
| count(*) | 
+----------+
| 3        | 
+----------+
1 row in set (0.01 sec)
```

常见的聚合有 max，min，sum 和 count 等。上面的语句只是输出了满足 region = ‘north’ 的行数，如果我们同时也想知道其他所有 region 的总人数呢？ 此时 `group by`就排上用场了：



```sql
TiDB> select region, count(*) from person group by region;
+--------+----------+
| region | count(*) |
+--------+----------+
| north  | 3        |
| west   | 1        |
+--------+----------+
2 row in set (0.01 sec)
```

当然，对于聚合的结果我们可能还是需要过滤一些行，不过此时前面介绍的 where 语句就不能使用了，因为 where 后面的过滤条件是在 group by 之前生效的，在 group by 之后过滤需要使用 having:

```
TiDB> select region, count(*) from person group by region having count(*) > 1;
+--------+----------+
| region | count(*) | 
+--------+----------+
| north  | 3        | 
+--------+----------+
1 row in set (0.02 sec)
```

此外，还有一些常见的操作例如 order by，limit 等，这里不再一一介绍。
除了单表上的操作，往往我们可能需要结合多个表的信息，例如还有另外一张 address 表：

```sql
TiDB> create table address(number int, address varchar(50));
Execute success (0.05 sec)
TiDB> insert into address values (1, 'a'),(2, 'b'),(3, 'c'), (4, 'd');
Execute success (0.02 sec)
```

最简单的结合两张表的信息便是将分别取两张表中的任意一行结合起来，这样我们一共会有 4*4 种可能的组合，也就是会得到 16 行：

```sql
TiDB> select name, address from person inner join address;
+-------+---------+
| name  | address |
+-------+---------+
| tom   | a       |
| tom   | b       |
| tom   | c       |
| tom   | d       |
| bob   | a       |
| bob   | b       |
| bob   | c       |
| bob   | d       |
| jay   | a       |
| jay   | b       |
| jay   | c       |
| jay   | d       |
| jerry | a       |
| jerry | b       |
| jerry | c       |
| jerry | d       |
+-------+---------+
16 row in set (0.02 sec)
```

但这样的信息产生的信息爆炸往往是我们不需要的，幸运的是我们可以指定组合任意行的策略，例如如果想要同时知道某个人的地址以及名字，那我们只需要取两张表中有相同 number 值的人接合在一起，这样只会产生 4 行结果：

```sql
TiDB> select name, address from person inner join address on person.number = address.number;
+-------+---------+
| name  | address |
+-------+---------+
| tom   | a       |
| bob   | b       |
| jay   | c       |
| jerry | d       |
+-------+---------+
4 row in set (0.02 sec)
```

需要注意的是这里的 join 为 inner join，除此以外还有 outer join，在这里就不再赘述。

#### 1.2.3 SQL Parser

SQL Parser 的功能是把 SQL 语句按照 SQL 语法规则进行解析，将文本转换成抽象语法树（AST）。这里简单介绍一下背景知识。

TinySQL 是使用 [goyacc ](https://github.com/cznic/goyacc)根据预定义的 SQL 语法规则文件 `parser.y` 生成 SQL 语法解析器。变异的时候，我们会先 build `goyacc`工具，然后使用 `goyacc`根据 `parser.y`生成解析器 `parser.go`。其中，[goyacc ](https://github.com/cznic/goyacc)是 [yacc ](http://dinosaur.compilertools.net/)的 `Golang` 版，所以要想看懂语法规则定义文件 [parser.y ](https://github.com/pingcap/tidb/blob/source-code/parser/parser.y)，了解解析器是如何工作的，先要对 [Lex & Yacc ](http://dinosaur.compilertools.net/)有些了解。

##### 1.2.3.1 Lex & Yacc 介绍

https://pingcap.com/blog-cn/tidb-source-code-reading-5

#### 1.2.4 练习

1. 在 `parser.y` 中为 `SelectStmt` 补充上 `where` 和 `group by` 字段，并通过测试。
2. 在 `handleQuery()` 中根据注释添加上处理语句 `Select` 语句(部分) 的逻辑，并通过测试。

### 1.x 参考资料

- Lexer & Parser
  - http://dinosaur.compilertools.net/
- goyacc
  - https://godoc.org/modernc.org/goyacc
  - https://pingcap.com/blog-cn/tidb-source-code-reading-5/

