---
title: jmap工具详解
date: 2020-05-21 16:07:18
tags:
- Java
- JDK
- JVM
categories: Java
keywords: jmap
cover: https://i.loli.net/2020/05/20/urITwzmxVQWyAbK.jpg
---
### 介绍
&emsp;&emsp;jmap用于打印出某个java进程（使用pid）内存内的所有‘对象’的情况（如：产生那些对象，及其数量）。
&emsp;&emsp;jmap可以输出所有内存中的对象，甚至可以将VM 中的heap，以二进制输出成文本。使用方法 `jmap -histo pid`。如果连用SHELL `jmap -histo pid>a.log`可以将其保存到文本中去，在一段时间后，使用文本对比工具，可以对比出GC回收了哪些对象。`jmap -dump:format=b,file=outfile 3024`可以将3024进程的内存heap输出出来到outfile文件里，再配合MAT（内存分析工具(Memory Analysis Tool），使用参见：MAT(Memory Analyzer Tool)工具入门介绍）或与jhat (Java Heap Analysis Tool)一起使用，能够以图像的形式直观的展示当前内存是否有问题。
&emsp;&emsp;64位机上使用需要使用如下方式：
```shell
jmap -J-d64 -heap pid
```
### 命令格式
```shell
jmap [ option ] pid
jmap [ option ] executable core
jmap [ option ] [server-id@]remote-hostname-or-IP
```

### 参数解读
- options
 executable Java executable from which the core dump was produced.(可能是产生core dump的java可执行程序)
core 将被打印信息的core dump文件
remote-hostname-or-IP 远程debug服务的主机名或ip
server-id 唯一id,假如一台主机上多个远程debug服务 
- 基本参数
-dump:[live,]format=b,file=<filename> 使用hprof二进制形式,输出jvm的heap内容到文件，live子选项是可选的，假如指定live选项,那么只输出活的对象到文件。 
-finalizerinfo 打印正等候回收的对象的信息。
-heap 打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况。
-histo[:live] 打印每个class的实例数目,内存占用,类全名信息. VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量。
-permstat 打印classload和jvm heap长久层的信息. 包含每个classloader的名字,活泼性,地址,父classloader和加载的class数量. 另外,内部String的数量和占用内存数也会打印出来。
-F 强迫在pid没有相应的时候使用-dump或者-histo参数，在这个模式下live子参数无效。
-h | -help 打印辅助信息 
-J 传递参数给jmap启动的jvm
pid 需要被打印配相信息的java进程id，可以用jps查询    

### 示例
- jmap -histo 5644
```shell
[xixincandeMacBook-Pro$~]jmap -histo 68679

 num     #instances         #bytes  class name
\\----------------------------------------------
   1:       1455337      616150368  [B
   2:       9329742      522465552  org.apache.rocketmq.store.schedule.ScheduleMessageService$DeliverDelayedMessageTimerTask
   3:       2851513      352057144  [C
   4:      10596345      339083040  java.util.concurrent.locks.AbstractQueuedSynchronizer$Node
   5:       9557923      152926768  java.lang.Object
   6:         49552       97763336  [I
   7:       2014663       48351912  java.lang.String
   8:        457198       23377296  [Ljava.lang.String;
   9:        479381       19466848  [Ljava.lang.Object;
  10:        225531       16238232  ch.qos.logback.classic.spi.LoggingEvent
  11:        480913       11541912  java.lang.StringBuilder
  12:        283490        9071680  java.util.LinkedList
  13:        226305        9052200  java.lang.ref.Finalizer
  14:        144576        8096256  java.util.concurrent.ConcurrentHashMap$EntryIterator
  15:         82807        7287016  com.alibaba.fastjson.serializer.JSONSerializer
  16:        226139        7236448  java.io.FileDescriptor
  17:        216052        6913664  org.apache.rocketmq.store.StoreStatsService$CallSnapshot
  18:        225528        5412672  org.slf4j.helpers.FormattingTuple
  19:         82807        5299648  com.alibaba.fastjson.serializer.SerializeWriter
  20:        164060        5249920  java.io.File
```
- jmap -dump:format=b,file=test.hprof 5644
```shell
[xixincandeMacBook-Pro$~]jmap -dump:format=b,file=test.hprof 5644
Dumping heap to \usr\local\bin\jdk1.8\bin\test.hprof ...
Heap dump file created
```