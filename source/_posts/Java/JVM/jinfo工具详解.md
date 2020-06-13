---
title: JVM系列--jinfo工具详解
date: 2020-05-21 16:31:30
tags:
- Java
- JDK
- JVM
- JVM系列
categories: Java
keywords: jinfo
cover: https://i.loli.net/2020/05/21/lAy3M4kgtw7BzO8.jpg
---
### 介绍
&emsp;&emsp; jinfo命令较为简单，可以用来查看 Java 进程运行的 JVM 参数，命令如下：
```shell
[root@admin ~]# jinfo --help
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message

```

我们先用 jps 命令查到 PID ，然后可以通过 jinfo 来查看对应进程的参数信息：
```shell
[root@admin ~]# jps
43520 Test
35900 Jps
```
查看 JVM 参数：

```shell
[root@admin ~]# jinfo -flags 43520
Attaching to process ID 43520, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.45-b08

-Dfile.encoding=GBK
```
查看系统参数：
```shell
[root@admin ~]# jinfo -sysflags 43520
```

虚拟机的参数可以通过这个命令查看：

```shell
java -XX:+PrintFlagsFinal -version | grep manageable
```
除了通过启动脚本可以设置参数，PrintGC 默认是打开的，因此我们只需要打开 PrintGCDetails 参数。
```shell
jinfo -flag +PrintGC 43520
jinfo -flag +PrintGCDetails 43520
```
关闭 GC 日志的话同理：
```shell
jinfo -flag -PrintGC 43520
jinfo -flag -PrintGCDetails 43520
```
查看是否开启 GC 日志的打印：

```shell
jinfo -flag PrintGC 43520
jinfo -flag PrintGCDetails 43520


[root@admin ~]# jinfo -flag PrintGC 43520
-XX:-PrintGC

[root@admin ~]# jinfo -flag PrintGCDetails 43520
-XX:-PrintGCDetails
```
### 解读
常用 JVM 参数：
* -Xms：初始堆大小，默认为物理内存的1/64(<1GB)；默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制
* -Xmx：最大堆大小，默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制
* -Xmn：新生代的内存空间大小，注意：此处的大小是（eden+ 2 survivor space)。与jmap -heap中显示的New gen是不同的。整个堆大小=新生代大小 + 老生代大小 + 永久代大小。在保证堆大小不变的情况下，增大新生代后,将会减小老生代大小。此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8。
* -XX:SurvivorRatio：新生代中Eden区域与Survivor区域的容量比值，默认值为8。两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10。
* -Xss：每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M,以前每个线程堆栈大小为256K。应根据应用的线程所需内存大小进行适当调整。在相同物理内存下,减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。一般小的应用， 如果栈不是很深， 应该是128k够用的，大的应用建议使用256k。这个选项对性能影响比较大，需要严格的测试。和threadstacksize选项解释很类似,官方文档似乎没有解释,在论坛中有这样一句话:"-Xss is translated in a VM flag named ThreadStackSize”一般设置这个值就可以了。
* -XX:PermSize：设置永久代(perm gen)初始值。默认值为物理内存的1/64。
* -XX:MaxPermSize：设置持久代最大值。物理内存的1/4。
