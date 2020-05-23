---
title: jhat工具详解
date: 2020-05-21 16:24:49
tags:
- Java
- JDK
- JVM
categories: Java
keywords: jhat
cover: https://i.loli.net/2020/05/20/urITwzmxVQWyAbK.jpg
---
### 介绍
&emsp;&emsp;jhat的英文全称是Java Virtual Machine Heap Analysis Tool 虚拟机堆转储快照分析工具，用于分析heapdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果。
&emsp;&emsp;Sun JDK提供了jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在浏览器中查看，不过实事求是地说，在实际工作中，除非真的没有别的工具可用，否则一般不会去直接使用jhat命令来分析demp文件，主要原因有二：一是一般不会在部署应用程序的服务器上直接分析dump文件，即使可以这样做，也会尽量将dump文件拷贝到其他机器上进行分析，因为分析工作是一个耗时且消耗硬件资源的过程，既然都要在其他机器上进行，就没必要受到命令行工具的限制了；另外一个原因是jhat的分析功能相对来说很简陋，VisualVM以及专门分析dump文件的Eclipse Memory Analyzer、IBM HeapAnalyzer等工具，都能实现比jhat更强大更专业的分析功能。
### 命令格式
```shell
jhat [options] heap-dump-file
```
### 解读
&emsp;&emsp; option具体选项及作用如下: 
* -J< flag > 因为 jhat 命令实际上会启动一个JVM来执行, 通过 -J 可以在启动JVM时传入一些启动参数. 例如, -J-Xmx512m 则指定运行 jhat 的Java虚拟机使用的最大堆内存为 512 MB. 如果需要使用多个JVM启动参数,则传入多个 -Jxxxxxx
* -stack false|true 关闭跟踪对象分配调用堆栈。如果分配位置信息在堆转储中不可用. 则必须将此标志设置为 false. 默认值为 true.
* -refs false|true 关闭对象引用跟踪。默认情况下, 返回的指针是指向其他特定对象的对象,如反向链接或输入引用(referrers or incoming references), 会统计/计算堆中的所有对象。
* -port port-number 设置 jhat HTTP server 的端口号. 默认值 7000。
* -exclude exclude-file 指定对象查询时需要排除的数据成员列表文件。 例如, 如果文件列出了 java.lang.String.value , 那么当从某个特定对象 Object o 计算可达的对象列表时, 引用路径涉及 java.lang.String.value 的都会被排除。
* -baseline exclude-file 指定一个基准堆转储(baseline heap dump)。 在两个 heap dumps 中有相同 object ID 的对象会被标记为不是新的(marked as not being new). 其他对象被标记为新的(new). 在比较两个不同的堆转储时很有用。
* -debug int 设置 debug 级别. 0 表示不输出调试信息。 值越大则表示输出更详细的 debug 信息。
* -version 启动后只显示版本信息就退出。  

### 示例
```shell
>>> jps -l
11008 org.jetbrains.jps.cmdline.Launcher
10992
17380 sun.tools.jps.Jps
>>> jmap -dump:format=b,file=jmap.bin 11008
Dumping heap to /Users/kay/learn/jvm/jmap.bin ...
Heap dump file created
>>> ll
total 94856
-rw-------  1 kay  staff  48563843  6 17 09:50 jmap.bin
>>> jhat jmap.bin
Reading from jmap.bin...
Dump file created Sun Jun 17 09:50:16 CST 2018
Snapshot read, resolving...
Resolving 160370 objects...
Chasing references, expect 32 dots................................
Eliminating duplicate references................................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```
可以看到，hat内置的HTTP/HTML服务器启动成功了，端口为默认端口7000，浏览器访问http://localhost:7000
分析结果截取了部分内容显示如下：
![20180617154925891](https://i.loli.net/2020/05/21/KpcmGnTuQ6ZOJbs.jpg)
