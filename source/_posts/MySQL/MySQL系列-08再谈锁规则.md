---
title: MySQL系列--08再谈锁规则
date: 2020-08-20 11:19:11
tags: 
- SQL
- MySQL
- MySQL系列
categories: MySQL 
keywords: MySQL,锁,加锁规则
cover: https://i.loli.net/2020/08/20/64HlxeIsjnfFDym.png
---
这段时间工作有点忙没有继续更新，MySQL锁重新梳理了一下，前面的文章中整理了几种锁的概念，但是没有整理加锁的规则，尤其是间隙锁和Next-Key Lock，容易在判断锁等待的问题上犯错。所以这篇找来了几个案例，从案例分析加锁规则。在进入案例之前，先看下前提条件说明已经加锁规则的总结。

### 前提
MySQL的不同版本可能加锁的策略会不太一样，我整理的策略限于MySQL 5.X系列，且不高于5.7.24。

另外，间隙锁前面也介绍过他的用途（解决幻读问题，这里不再赘述），它是在RR级别下才有效，所以这里整理的案例没有特殊说明，都是默认RR级别。

### 加锁规则
加锁的规则的总结来自于阿里的一位大牛，包含了两个“原则”、两个“优化”和一个“BUG”。
1. 原则1: 加锁的基本单位是Next-Key Lock。它是一个前开后闭的半开区间。
2. 原则2: 查找过程中访问到的对象才会加锁。
3. 优化1: 索引上的**等值查询**，给唯一索引加锁的时候，Next-Key Lock会退化为行锁。
4. 优化2: 索引上的**等值查询**，向**右**遍历时且最后一个值不满足等值条件的时候，Next-Key Lock会退化为间隙锁。
5. BUG: 唯一索引上的**范围查询**会访问到不满足条件的第一个值为止。

### 数据准备
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values
(0,0,0),
(5,5,5),
(10,10,10),
(15,15,15),
(20,20,20),
(25,25,25);
```

### 案例分析
#### 1-等值查询间隙锁
案例1是关于等值条件查询操作间隙：

| session A | session B | session C |
| --- | --- | --- |
| begin;<br> update t set d=d+1 where id=7; |  |  |
|  | insert into t values(8,8,8);<br> (blocked) |  |
|  |  | update t set d=d+1 where id=10;<br> (Query OK) |

由于表t中没有id=7的记录，所以用上面总结的加锁规则判断一下的话：
- 根据规则1，加锁单位是Next-Key Lock，session A加锁范围就是(5,10];
- 同时根据规则2，这是一个等值查询，而id=10不满足查询条件，Next-Key Lock退化为间隙锁，因此最终加锁范围是(5,10)。

所以，session B要往这个间隙里插入id=8的记录会被锁住，但是session C修改id=10这行是可以的。

#### 2-非唯一索引等值查询锁
案例2时关于覆盖索引的锁：

| session A | session B | session C |
| --- | --- | --- |
| begin;<br>select id from t where c=5 lock in share mode; |  |  |
|  | update t set d=d+1 where id=5;<br>(Query OK) |  |
|  |  | insert into t values(7,7,7);<br>(blocked) |

看到这个案例，是不是有一种该锁的不锁，不该锁的乱锁的感觉？我们来分析一下吧。

这里session A要给索引c上c=5的这一行加上读锁。
- 根据原则1，加锁单位是next-key lock，因此会给(0,5]加上next-key lock。
- 要注意c是普通索引，因此仅访问c=5这一条记录是不能马上停下来的，需要向右遍历，查到c=10才放弃。根据原则2，访问到的都要加锁，因此还要给(5,10]加next-key lock。
- 但是同时这个符合优化2：等值判断，向右遍历，最后一个值不满足c=5这个等值条件，因此退化成间隙锁(5,10)。
- 根据原则2 ，只有访问到的对象才会加锁，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么session B的update语句可以执行完成。
- 但session C要插入一个(7,7,7)的记录，就会被session A的间隙锁(5,10)锁住。

需要注意，在这个例子中，lock in share mode只锁覆盖索引，但是如果是for update就不一样了。执行 for update时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

这个例子说明，**锁是加在索引上的**；同时，它给我们的指导是，如果你要用lock in share mode来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段。比如，将session A的查询语句改成select d from t where c=5 lock in share mode。

#### 3-主键索引范围锁
这个案例时关于范围查询的。在看案例之前，先思考一个问题：对于这个表t，下面的两个查询语句，加锁的范围相同吗？
```sql
select * from t where id=10 for update;
select * from t where id>=10 and id<11 for update;
```
你可能会想，id定义为int类型，这两个语句就是等价的吧？其实，它们并不完全等价。

在逻辑上，这两条查语句肯定是等价的，但是它们的加锁规则不太一样。现在，我们就让session A执行第二个查询语句，来看看加锁效果。

| sessionA | sessionB | sessionC |
| --- | --- | --- |
| begin;<br>select * from t where id>=10 and id<11 for update; |  |  |
|  | insert into t values(8,8,8);<br>(Query OK)<br>insert into t values(13,13,13);<br>(blocked) |  |
|  |  | update t set d=d+1 where id=15;<br>(blocked) |

现在我们就用前面提到的加锁规则，来分析一下session A 会加什么锁：
- 开始执行的时候，要找到第一个id=10的行，因此本该是Next-Key Lock(5, 10]。根据优化1，主键上id的等值条件，退化成行锁，只加了id=10的行锁。
- 范围查找就往后继续，找到id=15的这一行停下来，因此需要加Next-Key Lock(10,15]。

所以，session A这时候锁的范围就是主键索引上，行锁id=10和Next Key Lock(10,15]。这样，session B和session C的结果就能理解了。

这里需要注意一点，首次session A定位查找id=10的行的时候，是当作等值查询来判断的，而向右扫描id=15的时候，用的是范围查询判断。

#### 4-非唯一索引范围锁
接下来，我们再看两个范围查询加锁的例子，可以对照着案例3来看。需要注意的是，与案例3不同的是，案例四中查询语句的where部分用的是字段c。

| session A | session B | session C |
| --- | --- | --- |
| begin;<br>select * from t where c>=10 and c<11 for update; |  |  |
|  | insert into t values(8,8,8);<br>(blocked) |  |
|  |  | update t set d=d+1 where c=15;<br>(blocked) |

这次session A用字段c来判断，加锁规则跟案例三唯一的不同是：
在第一次用c=10定位记录的时候，索引c上加了(5,10]这个Next-Key Lock后，由于索引c是非唯一索引，没有优化规则，也就是说不会退化为行锁，因此最终sesion A加的锁是，索引c上的(5,10] 和(10,15] 这两个Next-Key Lock。

所以从结果上来看，sesson B要插入（8,8,8)的这个insert语句时就被堵住了。这里需要扫描到c=15才停止扫描，是合理的，因为InnoDB要扫到c=15，才知道不需要继续往后找了。

#### 5-唯一索引范围锁BUG
前面的4个案例，我们已经用到了加锁规则中的两个原则和两个优化，接下来再看一个关于加锁规则中bug的案例。

| session A | session B | session C |
| --- | --- | --- |
| begin;<br>select * from t where id>10 and id<=15 for update; |  |  |
|  | update t set d=d+1 where id=20;<br>(blocked) |  |
|  |  | insert into t values(16,16,16);<br>(blocked) |

session A是一个范围查询，按照原则1的话，应该是索引id上只加(10,15]这个Next-Key Lock，并且因为id是唯一键，所以循环判断到id=15这一行就应该停止了。

但是实现上，InnoDB会往前扫描到第一个不满足条件的行为止，也就是id=20。而且由于这是个范围扫描，因此索引id上的(15,20]这个Next-Key Lock也会被锁上。

所以session B要更新id=20这一行，是会被锁住的。同样地，session C要插入id=16的一行，也会被锁住。

照理说，这里锁住id=20这一行的行为，其实是没有必要的。因为扫描到id=15，就可以确定不用往后再找了。但实现上MySQL还是这么做了，因此很多大牛认为这是个BUG。官方bug系统上也有提到，但是并未被verified。

#### 6-非唯一索引上存在“等值”的例子
接下来的例子，是为了更好地说明“间隙”这个概念。这里给表t插入一条新记录。
```sql
insert into t values(30, 10, 30);
```
新插入的这一行c=10，也就是说现在表里有两个c=10的行。那么，这时候索引c上的间隙是什么状态了呢？你要知道，由于非唯一索引上包含主键的值，所以是不可能存在“相同”的两行的。
![](https://i.loli.net/2020/08/20/lywnbWdqTFVA6Kj.png)
可以看到，虽然有两个c=10，但是它们的主键值id是不同的（分别是10和30），因此这两个c=10的记录之间，也是有间隙的。图中画出了索引c上的主键id。为了跟间隙锁的开区间形式进行区别，用(c=10,id=30)这样的形式，来表示索引上的一行。

这次使用delete语句来验证。注意，delete语句加锁的逻辑其实和select...for update是类似的，也就是在文章开始的总结规则。

| session A | session B | session C |
| --- | --- | --- |
| begin;<br>delete from t where c=10; |  |  |
|  | insert into t values(12,12,12);<br>(blocked) |  |
|  |  | update t set d=d+1 where c=15;<br>(Query OK) |

这时，session A在遍历的时候，先访问第一个c=10的记录。同样地根据原则1，这里加的是(c=5,id=5)到(c=10,id=10)这个Next-Key Lock。

然后，session A向右查找，直到碰到(c=15,id=15)这一行，循环才结束。根据优化2，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成(c=10,id=10) 到 (c=15,id=15)的间隙锁。

也就是说，这个delete语句在索引c上的加锁范围，就是下图中蓝色区域覆盖的部分。
![](https://i.loli.net/2020/08/20/qJxXEdbfQWrsLSi.png)
这个蓝色区域左右两边都是虚线，表示开区间，即(c=5,id=5)和(c=15,id=15)这两行上都没有锁。

#### 7-limit语句加锁
案例6的一个对照案例，场景如下：

| session A | session B |
| :-: | :-: |
| begin;<br>delete from t where c=10 limit 2; |  |
|  | insert into t values(12,12,12);<br>(Query OK) |

这个例子里，session A的delete语句加了 limit 2。已知表t里c=10的记录其实只有两条，因此加不加limit 2，删除的效果都是一样的，但是加锁的效果却不同。可以看到，session B的insert语句执行通过了，跟案例6的结果不同。

这是因为，本案例里的delete语句明确加了limit 2的限制，因此在遍历到(c=10, id=30)这一行之后，满足条件的语句已经有两条，遍历就结束了。因此，索引c上的加锁范围就变成了从（c=5,id=5)到（c=10,id=30)这个前开后闭区间，如下图所示：
![](https://i.loli.net/2020/08/20/gFXvenTDNP5pKlY.png)
可以看到，(c=10,id=30）之后的这个间隙并没有在加锁范围里，因此insert语句插入c=12是可以执行成功的。

这个例子对我们实践的指导意义就是，**在删除数据的时候尽量加limit**。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。

#### 8-一个死锁的例子
前面的例子中，我们在分析的时候，是按照Next-Key Lock的逻辑来分析的，因为这样分析比较方便。最后我们再看一个案例，目的是说明：Next-Key Lock实际上是间隙锁和行锁加起来的结果。

| session A | session B |
| :-: | :-: |
| begin;<br>select id from t where c=10 lock in share mode |  |
|  | update t set d=d+1 where c=10;<br>(blocked) |
| insert into t values(8,8,8); |  |
|  | ERROR 1213(40001):Deadlock found when trying to get lock; try restarting transaction |

现在，我们按时间顺序来分析一下为什么是这样的结果:
- session A 启动事务之后执行查询语句加lock in share mode，在索引c上加了Next-Key Lock(5,10]和间隙锁(10,15)；
- sesion B 的update语句也要在索引c上加Next-Key Lock(5,10] ，进入锁等待；
- 然后session A要再插入(8,8,8)这一行，被session B的间隙锁锁住。由于出现了死锁，InnoDB让session B回滚。

你可能会问，session B的Next-Key Lock不是还没申请成功吗？

其实是这样的，session B的“加Next-Key Lock(5,10] ”操作，实际上分成了两步，先是加(5,10)的间隙锁，加锁成功；然后加c=10的行锁，这时候才被锁住的。也就是说，我们在分析加锁规则的时候可以用Next-Key Lock来分析。但是要知道，具体执行的时候，是要分成间隙锁和行锁两段来执行的。

### 思考案例
经过这篇文章的介绍，还是使用我们在文章开头初始化的表t，里面有6条记录我们尝试分析一下这两个场景：
#### 场景一
如下表中的语句序列中，session B的insert操作会被锁住吗？

| session A | session B |
| --- | --- |
| begin;<br>select * from t where c>=15 and c<=20 order by c desc lock in share mode; |  |
|  | insert into t values(6,6,6); |

#### 场景二
如下表中的语句序列中，session B和session C的insert语句会进入锁等待状态吗?

| session A | session B | session C |
| --- | --- | --- |
| begin;<br>select * from t where c>=15 and c<=20 order by c desc for update; |  |  |
|  | insert into t values(1,11,11); |  |
|  |  | insert into t values(6,6,6); |

欢迎留言。