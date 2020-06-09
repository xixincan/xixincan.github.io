---
title: MySQL系列--02更新语句的执行
date: 2020-06-07 10:33:52
tags: 
- SQL
- MySQL
- MySQL系列
categories: MySQL 
keywords:
- MySQL
- MySQL日志系统
cover: https://i.loli.net/2020/06/07/Onkho7mDQaN4Lw3.png
---
## 楔子
我们了解了MySQL架构和一条查询SQL的执行流程，并介绍了执行过程中设计的各个组件模块；一条查询的SQL语句执行过程一般是经过连接器、分析器、优化器、执行器等组件模块，这些都是在Server层，最后达到存储引擎层。那么，一条更新SQL语句的执行流程又是怎么样的呢？

先从一张简单的表说起，假设现在有一张简单的表t，这个表只有一个主键id和一个整形的字段c，如果我们想将id=2这一行的值价1，SQL语句就会这样写：
```SQL
update t set c = c + 1 where id = 2;
```
首先，可以明确的是，查询语句的那一套执行流程，更新语句也是同样会走一遍。回顾一下结构图：
![](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图片来自互联网
执行语句前要先连接数据库，这是连接器的工作。

我们已经知道，在一张表上有更新时，跟这个表有关的查询缓存就会失效，所以这条语句会把表t上所有缓存结果清空。PS：这也是一般不建议使用查询缓存的原因。

接下来，分析器会通过词法和语法解析知道这是一条更新SQL。优化器决定要使用id这个索引。然后执行器负责具体执行，找到这一行，然后更新。

与查询流程不一样的是，更新流程还涉及两个重要的日志模块，也是这篇整理的真正的主角：redo log和binlog。

## 重要的日志模块：redo log
介绍redo log之前，先要了解一个名词：WAL技术 - Write Ahead Logging，它的关键点就是先写日志，等空闲时再写入磁盘。MySQL中的更新操作正是使用了WAL技术，想象一下：如果每次的更新操作都写进磁盘的话，那么磁盘也要找到对应记录的那条记录，然后执行更新，整个过程的IO成本、查找成本就会很高！一旦是更新请求多的时候，就成为了性能瓶颈。

具体来说，当有一条记录需要更新时，InnoDB引擎就会先把记录写到redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB引擎会在**适时**将这个操作记录更新到磁盘中，而这个更新往往是在系统比较空闲的时间去做。

如果redo log记录的量不多，InnoDB会在空闲时更新到磁盘，但是如果redo log写满了了怎么办呢？

因为InnoDB中redo log大小是配置固定的，比如可以配置为一组4个文件，每个文件大小为1GB，那么redo log总共可以记录4GB的操作。它从头开始写，写到末尾就会又回到开头循环写，如下图所示：
![](https://static001.geekbang.org/resource/image/b0/9c/b075250cad8d9f6c791a52b6a600f69c.jpg)
* write pos是当前记录的位置，一边写一边后移，写到3号文件尾端后就回到0号文件开头。
* checkpoint是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要先把记录更新到数据文件中（磁盘）。

write pos后面的部分以及checkpoint前面的部分是redo log还“空着”的部分，可以用来记录新的操作。如果write pos追上了checkpoint位置，表示redo log写满了，没有空余的部分可供新的记录新的操作了，这个时候就不能再执行新的更新，得停下来先擦掉一些记录，checkpoint向后推进一下。

又了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称之为**crash-safe**。因为只要记录记在了redo log上或者写入了磁盘中，即使重启，恢复后依然可以通过他们来明确操作记录。

## 重要的日志模块：binlog
MySQL整体来看其实就是两大块：Server层 - 主要负责MySQL功能层面的事情；引擎层 - 负责存储相关的事情。**redo log就是InnoDB引擎特有的日志**，而Server层也有自己的日志，即binlog。

为什么会有两份日志呢？

因为MySQL一开始并没有InnoDB引擎。MySQL自带的引擎是MyISAM，但是MyISAM没有crash-safe能力，binlog日志只能用于归档。InnoDB是另外一个公司以插件的形式引入MySQL的，使用redo log并以此来实现crash-safe能力。

这两种日志有一下三点不同：
* binlog是MySQL的Server层实现的，所有引擎都可以使用；redo log是InnoDB引擎特有的。
* binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给id=1这一行的c字段加1”；redo log是物理日志，记录的是“在某个数据页上做了什么修改”。
* binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。redo log是循环写的，空间固定且会用完。

## 更新流程
有了redo log和binlog的理解，再来看一下执行器和InnoDB引擎在执行这个简单的更新语句时的内部流程：
1. 执行器先找InnoDB取id=2这一行。id是主键，InnoDB直接用树搜索到这一行。如果id=2这一行所在的数据页本来就在内存中，就直接返回给执行器；否则需要先从磁盘读入内存然后再返回。
2. 执行器拿到行数据，把这个值加1，得到新的一行数据，再调用InnoDB接口写入这行新数据。
3. InnoDB将这行新数据更新到内存中，同时将这个更新操作记录到redo log中，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的binlog，并把binlog写入磁盘。
5. 执行器调用InnoDB的提交事务接口，引擎把刚刚写入的redo log改成commit状态，更新完成。

下图是更新语句饿执行流程图，深色表示执行器中执行的部分，浅色表示InnoDB执行部分：
![](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)
注意到redo log的写入其实分为两个步骤：prepare和commit，这就是**两阶段提交**。

## 两阶段提交
使用两阶段提交的目的就是为了让两份日志之间的逻辑保持一致。那为什么需要两阶段提交呢？

由于redo log和binlog是两个独立的逻辑，如果不用两阶段提交，要么就是先写完redo log再写binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。

仍然使用前面的更新语句做例子，假设当前id=2的行，字段c的值为0.
* **假设先写redo log**
如果redo log写完后，还没来得及写完binlog时，MySQL进程异常重启。系统进行恢复数据时如果根据redo log恢复则值为1，而根据binlog恢复则值为0.
* **假设先写binlog**
如果在binlog写完后MySQL进程crash，还没写redo log，崩溃恢复后同样binlog中记录与redo log中记录不一致。

可以看到，如果不使用两阶段提交，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。

可能会觉得这个场景概率很低，谁会没事去恢复临时库？其实不然，不只是误操作后需要这个过程来恢复数据。当需要扩容时，也就是需要再多搭建一些备库来增加系统的读能力的时候，现在常见的做法也是用全量备份加上应用binlog来实现的，如果两份日志不一致，就有可能导致出现数据库主从数据不一致的情况。

简单来说，**redo log和binlog都可以用于表示事务的提交状态，而两阶段提交就是为了保持这两个状态保持逻辑上的一致性**。

## 小结
MySQL里面最重要的两个日志，即物理日志redo log和逻辑日志binlog。

redo log用于保证crash-safe能力。innodb_flush_log_at_trx_commit这个参数设置成1的时候，表示每次事务的redo log都直接持久化到磁盘。这样可以保证MySQL异常重启之后数据不丢失。

sync_binlog这个参数设置成1的时候，表示每次事务的binlog都持久化到磁盘。这个参数我也建议你设置成1，这样可以保证MySQL异常重启之后binlog不丢失。

与MySQL日志系统密切相关的“两阶段提交”。两阶段提交是跨系统维持数据逻辑一致性时常用的一个方案，即使你不做数据库内核开发，日常开发中也有可能会用到。