---
title: MySQL系列--07Explain详解
date: 2020-06-24 14:32:31
tags: 
- SQL
- MySQL
- MySQL系列
categories: MySQL 
keywords: MySQL,Explain,执行计划
cover: https://i.loli.net/2020/06/07/2quU1hGkY5QwHJt.png
---
前面的整理中已经涉及到了EXPLAIN命令，EXPLAIN命令是MySQL查询性能优化不可缺少的一部分。这篇就主要整理了EXPLAIN命令的使用和参数说明，更深入地去了解一下这个常用工具，以便更好得优化SQL性能！同时后面也整理了几个典型的SQL优化案例。
### EXPLAIN全貌
当我需要优化一条SQL语句或者想看一条SQL语句的索引使用情况和执行计划时，会在这个SQL前加上`explain`命令来查看。其执行的结果会展现很多信息，如下表所示：

| 列名 | 说明 |
| :-: | :-: |
| id | 执行编号，标识select所属的行。如果在语句中没子查询或关联查询，只有唯一的select，每行都将显示1。否则，内层的select语句一般会顺序编号，对应于其在原始语句中的位置 |
| select_type | 显示本行是简单或复杂select。如果查询有任何复杂的子查询，则最外层标记为PRIMARY（DERIVED、UNION、UNION RESUlT） |
| table | 访问引用哪个表 |
| type | 数据访问/读取操作类型（ALL、index、range、ref、eq_ref、const/system、NULL） |
| possible_keys | 揭示哪一些索引可能有利于高效的查找 |
| key | 显示MySQL决定采用哪个索引来优化查询 |
| key_len | 显示MySQL在索引里使用的字节数 |
| ref | 显示了之前的表在key列记录的索引中查找值所用的列或常量 |
| rows | 扫描行数，估算值，不精确 |
| filtered | 针对表里符合某个条件的记录数的百分比估算 |
| Extra | 额外信息，如using index、filesort等 |

#### id
id字段是用来顺序标识整个查询中的select语句的，在嵌套查询中id越大的语句越先执行。该值可能为NULL，如果该值用来说明的是其他行的联合结果。
#### select_type
表示查询的类型，其类型有如下几种：
* **simple**  
简单的查询，不包含子查询和union
* **primary**  
包含union或者子查询，最外层的部分标记为primary
* **subquery**  
一般查询中的子查询被标记为subquery，也就是位于select列表中的查询
* **derived**  
派生表 --- 该临时表是从子查询中派生出来的，位于from中的子查询
* **union**  
位于union中第二个及其以后的子查询被标记为union，第一个就被标记为primary如果是union位于from中则标记为derived
* **union result**  
用来从匿名临时表中检索结果的select被标记为union result
* **dependent union**  
首先需要先满足union的条件以及union中第二个以及后面的select语句，同时该语句依赖外部的查询
* **dependent subquery**  
和DEPENDENT UNION相对UNION一样

#### table
对应正在访问哪一张表，表名或者别名。
* 关联优化器会为查询选择关联顺序，左侧深度优先
* 当from中有子查询的时候，表名是derivedN的形式，N执行子查询，也就是explain结果中的下一列
* 当有union result的时候，表名是union 1,2等的形式，1,2表示参与union 的query id

#### type
一个重要的指标，显示的是访问的类型。结果值的好坏程度依次是：
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > **range** > index > ALL
> 阿里编程规范要求：至少要达到range级别，要求是ref级别，如果可以是const最好。

* **ALL**  
最坏的情况，全表扫描
* **index**  
和全表扫描一样。只是扫描表的时候按照索引的次序进行而不是按行进行。主要优点是避免了排序，但是开销仍非常大。如果在Extra中看到了Using index，说明正在使用覆盖索引，只扫描索引的数据，它比按索引次序全表扫描的开销要小很多
* **range**  
范围扫描，一个有限制的索引扫描。key列显示使用了哪个索引。当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range。
* **ref**  
一种索引访问，它返回所有匹配某个单个值的行。此类索引访问只有当使用非唯一索引或者唯一性性索引的非唯一性前缀时才会发生。这个类型跟eq_ref不同的是，他用在关联操作只使用了索引的最左前缀或者索引不是唯一性的索引。ref可以使用=或者<=>操作符的带索引的列。
* **eq_ref**  
最多只返回一条符合条件的记录。使用唯一索引或主键索引查找时会发生。（高效）
* **const**  
当确定最多只有一行匹配时，MySQL优化器会在查询前读取它而且只读取一次，因此非常快。当主键放入where子句时，MySQL把这个查询转换为一个常量。（高效）
* **system**  
这是const连接类型的一种特列，表仅一行满足条件。（高效）
* **Null**  
意味着MySQL能在优化阶段分解查询语句，在执行阶段甚至用不到访问表或着索引。（高效）

> 如何查看MySQL优化器优化之后的SQL：
> ```sql
> EXPLAIN SELECT * FROM demo WHERE name='AAA' AND age=18;
> show warnings;
> ```
> 为什么要做这件事？我们知道MySQL有一个最左匹配原则，那么如果我的索引建的是(age, name)，那么我们以name, age的顺序去查询能够使用到索引吗？实际上是可以的，因为MySQL查询优化器可以帮助我们自动对SQL的执行顺序进行优化，以选取代价最低的方式进行查询（注意是代价最低，不是时间最少，MySQL可能会选错索引，上一篇有介绍）。

#### possible_keys
显示查询使用了哪些索引，表示该索引可以进行高效地查找，但是列出来的索引对于后续优化过程可能是没有用的。
#### key
key列显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用`FORCE INDEX`、`USE INDEX`或者`IGNORE INDEX`。
#### key_len
key_len列显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。使用的索引的长度。在不损失精确性的情况下，长度越短越好。
#### ref
ref列显示使用哪个列或常数与key一起从表中选择行。
#### rows
rows列显示MySQL认为它执行查询时必须检查的行数。注意这是一个预估值。
#### Extra
Extra是EXPLAIN输出中另外一个很重要的列，该列显示MySQL在查询过程中的一些详细信息，MySQL查询优化器执行查询的过程中对查询计划的重要补充信息。
* **Using filesort**  
MySQL有两种方式可以生成有序的结果，通过排序操作或者使用索引，当Extra中出现了Using filesort 说明MySQL使用了外部索引排序，而不是按照索引次序从表里读取数据。
但注意虽然叫filesort但并不是说明就是用了文件来进行排序，只要可能排序都是在内存里完成的。大部分情况下利用索引排序更快，所以一般这时也要考虑优化查询了。
使用文件完成排序操作，这是可能是ordery by，group by语句的结果，这可能是一个CPU密集型的过程，可以通过选择合适的索引来改进性能，用索引来为查询结果排序。
* **Using temporary**  
用临时表保存中间结果，常用于GROUP BY 和 ORDER BY操作中，一般看到它说明查询需要优化了，就算避免不了临时表的使用也要尽量避免硬盘临时表的使用。
* **Not exists**  
MySQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了。
* **Using index**  
说明查询是覆盖了索引的，不需要读取数据文件，从索引树（索引文件）中即可获得信息。如果同时出现using where，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。这是MySQL服务层完成的，但无需再回表查询记录。
* **Using index condition**  
这是MySQL 5.6出来的新特性，叫做“索引条件推送”。简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上。
* **Using where**  
意味着MySQL服务器将在存储引擎检索行后再进行过滤。许多where条件中涉及索引中的列，当它读取索引时，就能被存储引擎检验，因此不是所有带where子句的查询都会显示。有时“using where”的出现就是一个暗示：查询可能受益于不同的索引。
* **Using join buffer**  
使用了连接缓存：Block Nested Loop，连接算法是块嵌套循环连接;Batched Key Access，连接算法是批量索引连接
* **impossible where**  
where子句的值总是false，不能用来获取任何元组。
* **select tables optimized away**  
在没有GROUP BY子句的情况下，基于索引优化MIN/MAX操作，或者对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。
* **distinct**  
优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作。

### SQL优化典型案例
#### 超大分页场景
如果表中数据需要进行深度分页，怎么去提高效率？
> 阿里的变成规范中写道：利用延迟关联或者子查询优化超多分页场景

说明：MySQL并不是跳过offset行，而是取offset+N行，然后放弃前offset行，返回N行。当offset特别大的时候，效率就非常的地下，这时要么控制返回的总页数（现在明白了为什么有的网页中列表限制页面或者限制数据记录数了吧），要么对超过阈值的页数进行SQL改写。

例如，有表a_table总数据量为3400万，id为主键，偏移量达到2000万深度分页查询：
```sql
select * from a_table limit 20000000, 10;   --耗时129s;
```
优化后耗时5s：
```sql
select a.* from a_table a, (select id from a_table limit 20000000, 10) b where a.id = b.id;
```

#### 获取一条数据时使用LIMIT 1
如果数据表的情况已知，某个业务需要获取符合某个where条件下的一条数据，注意使用`limit 1`。

说明：在很多情况下，我们已知数据仅存在一条，此时我们应该告知数据库只用查询一条，否则将会转化为全表扫描

#### 批量化插入
改单个循环插入为批量插入，比较常规，不多做说明了

#### LIKE语句的优化
like语句一半业务要求都是`%关键字%`这种形式，但是依然要思考能否考虑使用右模糊的方式去替代产品的要求，其中阿里的编码规范提到：**页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决**

#### 使用IS NULL 来判断是否为NULL值
NULL与任何值的直接比较结果都是NULL：
1. NULL<>NULL的返回结果是NULL，而不是false
2. NULL=NULL的返回结果是NULL，而不是true
3. NULL<>1的返回结果是NULL，而不是true

> 日常开发使用MyBatis写SQL使用=操作符时，如果变量可能存在NULL，你可以使用if标签判断，也可以业务中过滤。其实还有一种写法，使用<=>。使用<=>操作符可以不用考虑变量NULL值问题。

#### 多表查询
参考阿里的编程规范：**超过三个表禁止JOIN。需要JOIN的字段，数据类型必须绝对一致；多表关联查询时，保证被关联的字段需要有索引。**

#### 明明有索引为什么还走全表扫描
针对查询的数据行占总数据量过多时会转化为全表查询

那么这个过多指代的是多少呢？相关测试结果是50%，但是MySQL优化器不会仅仅根据这个指标来区分是否全表，而是有很多其他因素综合考虑哪一种方式效率更高。充分认识到该问题即可。

#### count(\*)？count(id)？count(1)？
阿里的编码规范中提到：**【强制】不要使用count(列名)或者count(常量)来替代count(*)**

count(\*)是SQL92定义的标准统计行数的语法，跟数据库无关，跟NULL和非NULL无关。count(*)会统计值为NULL的行，而count(列名)不会统计此列为NULL值的行。

#### 字段类型不同导致索引失效
阿里的编码规范中提到：**【推荐】防止因字段类型不同造成的隐式转换，导致索引失效**

实际上数据库在查询的时候会做一层隐式转换，比如varchar类型字段通过数字取查询。
```sql
# 正例
EXPLAIN SELECT * FROM demo where name = '1';
type：ref
ref：const	
rows:1	
Extra:Using index condition

# 反例
EXPLAIN SELECT * FROM demo where name = 1;
type：index
ref：NULL	
rows:3(总记录数)
Extra:Using where; Using index

# 说明
name字段有相应索引，且格式为varchar 
```
