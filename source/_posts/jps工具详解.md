---
title: jps工具详解
date: 2020-05-21 11:43:55
tags:
- Java
- JDK
- JVM
categories: Java
keywords: jps
cover: https://i.loli.net/2020/05/20/urITwzmxVQWyAbK.jpg
---
### 介绍
jps是jdk提供的一个**查看当前java进程**的小工具， 可以看做是JavaVirtual Machine Process Status Tool的缩写。用法简单，非常实用。

### 命令格式
```shell
    jps [options] [hostid] 
```
### 解读
- [options]选项  
\-q：仅输出VM标识符，不包括classname, jar name, arguments in main method 
\-m：输出main method的参数 
\-l：输出完全的包名，应用主类名，jar的完全路径名 
\-v：输出JVM参数 
\-V：输出通过flag文件传递到JVM中的参数(.hotspotrc文件或-XX:Flags=所指定的文件 
\-Joption: 传递参数到vm, 例如: -J-Xms512m 
- [hostid]选项  
```shell
    [protocol:][[//]hostname][:port][/servername]
```
命令的输出格式 ：
```shell
lvmid [ [ classname| JARfilename | "Unknown"] [ arg* ] [ jvmarg* ] ]
```

### 示例
#### jps
```shell
[xixincandeMacBook-Pro$~]jps
29091 Jps
68677 NamesrvStartup
53653 
68679 BrokerStartup
96697 Launcher
```
#### jps -l
输出主类或者jar的完全路径名
```shell
[xixincandeMacBook-Pro$~]jps -l
29106 sun.tools.jps.Jps
68677 org.apache.rocketmq.namesrv.NamesrvStartup
53653 
68679 org.apache.rocketmq.broker.BrokerStartup
96697 org.jetbrains.jps.cmdline.Launcher
```
#### jps -v
输出JVM参数
```shell
[xixincandeMacBook-Pro$~]jps -v
29122 Jps -Denv.class.path=/Library/Java/JavaVirtualMachines/jdk1.8.0_152.jdk/Contents/Home/lib/tools.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_152.jdk/Contents/Home/lib/dt.jar -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_152.jdk/Contents/Home -Xms8m
68677 NamesrvStartup -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 -XX:-UseParNewGC -verbose:gc -Xloggc:/Volumes/RAMDisk/rmq_srv_gc_%p_%t.log -XX:+PrintGCDetails -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m -XX:-OmitStackTraceInFastThrow -XX:-UseLargePages -Djava.ext.dirs=/Library/Java/JavaVirtualMachines/jdk1.8.0_152.jdk/Contents/Home/jre/lib/ext:/Users/xixincan/RocketMQ-4_7_0/bin/../lib
53653  -Xms128m -Xmx1500m -XX:ReservedCodeCacheSize=240m -XX:+UseCompressedOops -Dfile.encoding=UTF-8 -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -Djdk.http.auth.tunneling.disabledSchemes="" -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none -XX:ErrorFile=/Users/xixincan/java_error_in_idea_%p.log -XX:HeapDumpPath=/Users/xixincan/java_error_in_idea.hprof -javaagent:/Applications/IntelliJ IDEA.app/Contents/bin/JetbrainsCrack.jar -Djb.vmOptionsFile=/Users/xixincan/Library/Preferences/IntelliJIdea2019.1/idea.vmoptions -Didea.home.path=/Applications/IntelliJ IDEA.app/Contents -Didea.executable=idea -Didea.paths.selector=IntelliJIdea2019.1
```
#### jps -q
仅仅显示java进程号
```shell
[xixincandeMacBook-Pro$~]jps -q
68677
53653
68679
96697
29128
```
***注意：如果需要查看其他机器上的jvm进程，需要在待查看机器上启动jstatd。***