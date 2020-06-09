---
title: JVM系列--jstat工具详解
date: 2020-05-21 11:01:19
tags:
- Java
- JDK
- JVM
- JVM系列
categories: Java
keywords: jstat
cover: https://i.loli.net/2020/05/21/yKdPetDNVBxRQri.jpg
---
### 解读
#### 介绍
&emsp;&emsp;jstat是JDK自带的一个轻量级小工具。全称“Java Virtual Machine statistics monitoring tool”，它位于java的bin目录下，主要利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控，包括了对Heap size和垃圾回收状况的监控。可见，Jstat是轻量级的、专门针对JVM的工具，非常适用。
#### 命令格式
&emsp;&emsp;jstat工具特别强大，有众多的可选项，详细查看堆内各个部分的使用量，以及加载类的数量。使用时，需加上查看进程的进程id，和所选参数。
&emsp;&emsp;参考格式如下：
```shell
jstat -options 
```
可以列出当前JVM版本支持的选项，常见的有:
class (类加载器) 
compiler (JIT) 
gc (GC堆状态) 
gccapacity (各区大小) 
gccause (最近一次GC统计和原因) 
gcnew (新区统计)
gcnewcapacity (新区大小)
gcold (老区统计)
gcoldcapacity (老区大小)
gcpermcapacity (永久区大小)
gcutil (GC统计汇总)
printcompilation (HotSpot编译统计)

#### 使用说明
##### jstat –class<pid> : 显示加载class的数量，及所占空间等信息。
Loaded: 装载的类的数量
Bytes: 装载类所占用的字节数
Unloaded: 卸载类的数量
Bytes: 卸载类的字节数
Time: 装载和卸载类所花费的时间
##### jstat -compiler <pid>显示VM实时编译的数量等信息。
Compiled: 编译任务执行数量
Failed: 编译任务执行失败数量
Invalid: 编译任务执行失效数量
Time: 编译任务消耗时间
FailedType: 最后一个编译失败任务的类型
FailedMethod: 最后一个编译失败任务所在的类及方法
##### jstat -gc <pid>: 可以显示gc的信息，查看gc的次数，及时间。
S0C: 年轻代中第一个survivor（幸存区）的容量 (字节)
S1C: 年轻代中第二个survivor（幸存区）的容量 (字节)
S0U: 年轻代中第一个survivor（幸存区）目前已使用空间 (字节)
S1U: 年轻代中第二个survivor（幸存区）目前已使用空间 (字节)
EC: 年轻代中Eden（伊甸园）的容量 (字节)
EU: 年轻代中Eden（伊甸园）目前已使用空间 (字节)
OC: Old代的容量 (字节)
OU: Old代目前已使用空间 (字节)
PC: Perm(持久代)的容量 (字节)
PU: Perm(持久代)目前已使用空间 (字节)
YGC: 从应用程序启动到采样时年轻代中gc次数
YGCT: 从应用程序启动到采样时年轻代中gc所用时间(s)
FGC: 从应用程序启动到采样时old代(全gc)gc次数
FGCT: 从应用程序启动到采样时old代(全gc)gc所用时间(s)
GCT: 从应用程序启动到采样时gc用的总时间(s)
##### jstat -gccapacity <pid>:可以显示，VM内存中三代（young,old,perm）对象的使用和占用大小
NGCMN: 年轻代(young)中初始化(最小)的大小(字节)
NGCMX: 年轻代中最大的大小(字节)
NGC: 年轻代中当前的的容量(字节)
S0C: 年轻代中第一个survivor中的容量
S1C: 年轻代中第二个survivor中的容量
EC: 年轻代中Eden区的容量
OGCMN: Old代初始化容量
OGCMX: Old代最大容量
OGC: Old代当前新生成的容量
OC: Old代的容量
PGCMN: Perm代初始化容量
PGCMX: Perm代最大容量
PGC: Perm代当前新生成的容量
PC: Perm代的容量
YGC: 从应用程序启动到采样时年轻代中GC的次数
FGC: 从应用程序启动到采样时老年代中GC（FULL GC）的次数
##### jstat -gcutil <pid>:统计gc信息
S0: 年轻代中一个幸存区已使用的占当前容量百分比
S1: 年轻代中另一个幸存区已使用的占当前容量百分比
E: 年轻代中Eden区已使用的占当前容量百分比
O: 老年代已使用的占当前容量百分比
P: 永久代已使用的占当前容量百分比
YGC: 从应用程序启动到采样时年轻代中GC次数
YGCT: 从应用程序启动到采样时年轻代中GC所用时间(s)
FGC: 从应用程序启动到采样时Old代(FULL GC)GC次数
FGCT: 从应用程序启动到采样时Old代(FULL GC)GC所用时间(s)
GCT: 从应用程序启动到采样时GC的总时间(s)
##### jstat -gcnew <pid>:年轻代对象的信息。
S0C: 年轻代中其中一个幸存区的容量（字节）
S1C: 年轻代中另一个幸存区的容量（字节）
S0U: 年轻代中其中一个幸存区目前已使用的空间（字节）
S1U: 年轻代中另一个幸存区目前已使用的空间（字节）
TT: 持有次数限制
MTT: 最大持有次数限制
EC: 年轻代中Eden区的容量（字节）
EU: 年轻代中Eden区已使用的空间（字节）
YGC: 见上
YGCT: 见上
##### jstat -gcnewcapacity<pid>: 年轻代对象的信息及其占用量。
NGCMN: 见上
NGCMX: 见上
NGC: 见上
S0CMX: 年轻代中一个幸存区的最大容量（字节）
S0C: 见上
S1CMX: 年轻代中另一个幸存区的最大容量（字节）
S1C: 见上
ECMX: 年轻代中Eden区的最大容量（字节）
EC: 见上
YGC: 见上
FGC: 见上
##### jstat -gcold <pid>：old代对象的信息。
PC: 见上
PU: 见上
OC: 见上
OU: 见上
YGC: 见上
FGC: 见上
FGCT: 见上
GCT: 见上
##### stat -gcoldcapacity <pid>: old代对象的信息及其占用量。
OGCMN: 见上
OGCMX: 见上
OGC: 见上
OC: 见上
YGC: 见上
FGC: 见上
FGCT: 见上
GCT: 见上
##### jstat -gcpermcapacity<pid>: perm对象的信息及其占用量。
PGCMN: 见上
PGCMX: 见上
PGC: 见上
PC: 见上
YGC: 见上
FGC: 见上
FGCT: 见上
GCT: 见上
##### jstat -printcompilation <pid>：当前VM执行的信息。
Compiled: 编译任务的数目
Size: 方法生成的字节码的大小
Type: 编译类型
Method: 类名和方法名用来标识编译的方法。类名使用/做为一个命名空间分隔符。方法名是给定类中的方法。上述格式是由-溪溪:+PrintComplation选项进行设置的
