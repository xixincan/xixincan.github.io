---
title: JVM进阶--优化原理与案例分析
date: 2020-05-21 19:47:25
tags: 
- Java
- JVM
- JVM系列
categories: Java
keywords: 
- JVM
- JVM优化
cover: https://i.loli.net/2020/05/21/yscedjz713tY8hF.jpg
---
## 目标
* 熟悉JVM参数优化步骤（...就是套路嗨...）
* 重视系统的稳定性

## 回顾
### JVM内存结构
&emsp;&emsp;先简单回顾一下JVM内存结构和常见的垃圾回收器。
&emsp;&emsp;当代主流的虚拟机(Hotspot VM)的垃圾回收都采⽤“分代回收”的算法。“分代回收”是基于这样一个事实: 对象的⽣命周期不同，所以针对不同生命周期的对象可以采取不同的回收方式，以便提高回收效率。
&emsp;&emsp;Hotspot VM将内存结构划分为不通的物理区，就是“分代”思想的体现。如图所示，JVM内存主要由新生代、老年代、永久代构成。
![JVM](https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=2273912787,2835056962&fm=26&gp=0.jpg)
&emsp;&emsp;⼴泛地说，JVM堆内存被分为两部分——年轻代(Young Generation)和老年代(Old Generation)。
- 年轻代
&emsp;&emsp;年轻代是所有新对象产⽣的地方。当年轻代内存空间被用完时，就会触发垃圾回收。这个垃圾回收叫做Minor GC。年轻代被分为3个部分——Eden区和两个Survivor区。新生代内又分三个区:一个Eden区，两个Survivor区(⼀般而⾔)，⼤部分对象在Eden区中⽣成。当Eden区满时，还存活的对象将被复制到两个Survivor区(中的⼀个)。当这个Survivor区满时，此区的存活 且不满⾜足“晋升”条件的对象将被复制到另外⼀个Survivor区。对象每经历一次Minor GC，年龄加1，达到“晋升年龄阈值”后，被放到⽼年代，这个过程也称为“晋升”。显然，“晋升年龄阈值”的大⼩直接影响着对象在新⽣代中的停留留时间，在Serial和ParNew GC两种回收器中，“晋升年龄阈值”通过参数MaxTenuringThreshold设定，默认值为15。
- 年老代
&emsp;&emsp;年老代内存里包含了⻓期存活的对象和经过多次Minor GC后依然存活下来的对象。通常会在老年代内存被占满时进行垃圾回收。⽼年代的垃圾收集叫做Major GC。Major GC会花费更多的 时间。
- 永久代
&emsp;&emsp;Java8中永久代已经被移除，被“元空间”取代。元空间不在虚拟机中，使⽤了本地内存。主要存放元数据，例如Class、Method的元信息，与垃圾回收要回收的Java对象关系不大。
- JVM堆大小设置相关参数  

| 参数 | 描述 |
| :-- | :-- |
| -Xms | JVM启动时堆的初始化大小 |
| -Xmx | 设置堆的最大值 |
| -XX:Newsize | 设置Yong Gen的初始值⼤小 |
| -XX:MaxNewSize | 设置Yong Gen的最⼤值⼤小 |
| -XX:NewRatio | 设置Yong和Old的⽐例 |
| -XX:SurvivorRatio | 设置Eden和一个Suivior的比例 |
| -XX:MetaspaceSize | 设置Metaspace的初始值大⼩ |
| -XX:MaxMetaspaceSize | 设置Metaspace的最大值大⼩ |

&emsp;&emsp;解释:
参数-Xms和-Xmx设置一样，避免每次GC之后重新分配堆的⼤小。
参数-XX:NewRatio设置老年代与新⽣代的⽐例。例如-XX:NewRatio=3指定⽼年代/新⽣代为3/1，⽼年代占堆大⼩的3/4，新⽣代占1/4。
参数-XX:SurvivorRatio指定Eden与一个Survivor幸存者⼤⼩⽐例。例如 -XX:SurvivorRatio=10表示Eden是To⼤小的10倍(也是From的10倍)。所以,Eden占新生代⼤⼩的10/12, From和To每个占新⽣代的1/12。
- 其他CMS相关参数

| 参数 | 描述 |
| :-- | :-- |
| -XX:+UseConcMarkSweepGC | 年老代启⽤CMS |
| -XX:+UseParNewGC | 新生代启用并行回收 |
| -XX:ConcGCThreads | 并发的GC线程数 |
| -XX:+UseCMSCompactAtFullCollection | FullGC之后做压缩（减少碎片） |
| -XX:CMSFullGCsBeforeCompaction | 设置多少次full GC后进⾏内存压缩 |
| -XX:CMSInitiatingOccupancyFraction=70 | CMS是在tenured generation(年老代)占满 70%的时候开始进行CMS收集，如果你的年老代增长不是那么快，并且希望降低CMS次数的话，可以适当调⾼此值; |
| -XX:+CMSScavengeBeforeRemark | FullGC之前先做Minor GC |

### Minor GC && Full GC
* 从年轻代空间(包括 Eden 和 Survivor 区域)回收内存被称为 Minor GC; 
* 对⽼年代GC称为Major GC;
* Full GC是对整个堆来说的;
&emsp;&emsp;Major GC的速度一般会比Minor GC慢10倍以上。下边看有那种情况触发JVM进行Full GC及应对策略。  

#### Minor GC触发条件
&emsp;&emsp;当Eden区满时，触发Minor GC。当发生 Minor GC时，Eden区和from指向的Survivor区中的存活对象会被复制(此处采⽤标-复制算法)到to指向的Survivor区中，然后交换from和to指针，以保证下一次 Minor GC时，o指向的Survivor区还是空的。
#### Full GC触发条件
1. System.gc()⽅法的调用此方法的调⽤是建议JVM进行Full GC,虽然只是建议⽽⾮一定,但很多情况下 它会触发Full GC,从而增加Full GC的频率,也即增加了间歇性停顿的次数。强烈影响系建议能不使⽤此方法就别使用，让虚拟机自⼰管理它的内存，可通过-XX:+ DisableExplicitGC来禁止 RMI(Java远程⽅法调用)调⽤System.gc。
2. 年老代空间不足JVM中堆空间主要由新生代和老年代组成。新创建的对象大多在新生代中创建，当对象经过⼏次Minor GC依然存活，才有机会被转⼊⽼年代。这时问题就来了，如果此时老年代的空间不⾜以容纳从新生代转入的对象，那么JVM就会进行FullGC，以清理老年代的空间。由Eden区、From Space区向To Space区复制时，对象⼤小大于To Space可用内存，则把该对象转存到⽼年代，且⽼年代的可用内存⼩于该对象⼤小
3. 永久代(元空间)空间不⾜永久代(Permanent Generation)一般是⽤来存放类信息、字符串串常量的地方，如果我们永久代设置的空间⽐比较⼩无法容纳⾜够的类信息时，或者因为频繁热加载类信息，又或者存储了太多的字符串常量，那么系统就会触发Full GC，以清理永久代。
4. 通过Minor GC后进⼊年老代的平均⼤⼩⼤于年老代的可用内存如果发现统计数据说之前Minor GC的平均晋升⼤小比⽬前old gen剩余的空间⼤，则不会触发Minor GC⽽是转为触发full GC
5. 执行jmap -histo:live或者jmap -dump:live的时候

## 案例分析
### 案例一：Major GC和Minor GC频繁
#### 优化
首先优化Minor GC频繁问题。通常情况下，由于新生代空间较⼩，Eden区很快被填满，就会导致频繁Minor GC，因此可以通过增大新⽣代空间来降低Minor GC的频率。例如在相同的内存分配率的前提下，新⽣代中的Eden区增加一倍，Minor GC的次数就会减少一半。这时就有这样的疑问，扩容Eden区虽然可以减少Minor GC的次数，但会增加单次Minor GC时间么?如果单次Minor GC时间也增加，很难保证最后的优化效果。我们结合下⾯情况来分析，单次Minor GC时间主要受哪些因素影响?是否和新⽣代⼤小存在线性关系? 首先，单次Minor GC时间由以下两部分组成:T1(扫描新⽣代)和T2(复制存活对象到Survivor区)如下图：
![WX20200521-213837.png](https://i.loli.net/2020/05/21/bi6ewE2uYkGP47R.png)
- 扩容前:新生代容量为R ，假设对象A的存活时间为750ms，Minor GC间隔500ms，那么本次 Minor GC时间=T1(扫描新生代R)+T2(复制对象A到S)。
- 扩容后:新⽣代容量为2R ，对象A的生命周期为750ms，那么Minor GC间隔增加为1000ms，此时Minor GC对象A已不再存活，不需要把它复制到Survivor区，那么本次GC时间 = 2×T1(扫描新⽣代R)，没有T2复制时间。 可见，扩容后，Minor GC时增加了T1(扫描时间)，但省去T2(复制对象)的时间，更重要的是对于虚拟机来说，复制对象的成本要远高于扫描成本，所以，单次Minor GC时间更多取决于GC后存活对象的数量，⽽非Eden区的⼤小。因此如果堆中短期对象很多，那么扩容新⽣代，单次Minor GC时间不会显著增加。具体还要分析。 由此可⻅，服务中如果是存在⼤量短期临时对象，扩容新⽣代空间后，Minor GC频率降低，对象在新生代得到充分回收，只有生命周期长的对象才进入⽼年代。这样⽼年代增速变慢，Major GC频率⾃然也会降低。

#### 小结
如何选择各分区⼤小应该依赖应⽤程序中对象生命周期的分布情况: 如果应用存在⼤量的短期对象，应该选择较大的年轻代; 如果存在相对较多的持久对象，⽼年代应该适当增大。
#### 更多思考
如果你查询对象生命周期分布的话，就会发现虽然 MaxTenuringThreshold=15，对象仍然仅经历不到次数就会Minor GC，就晋升到⽼年代?这里涉及到“动态年龄计算”的概念。动态年龄计算:Hotspot遍历所有对象时，按照年龄从⼩到大对其所占用的⼤小进行累积，当累积的某个年龄⼤⼩超过了survivor区的⼀半时，取这个年龄和MaxTenuringThreshold中更⼩的⼀个值，作为新的晋升年龄阈值。
在本案例中，调优前:Survivor区=64M，desired survivor=32M，此时Survivor区中age<=2的对象累计⼤小为41M，41M⼤于32M，所以晋升年龄阈值被设置为2，下次Minor GC时将年龄超过2的对象被晋升到⽼年代。
JVM引⼊动态年龄计算，主要基于如下两点考虑:
1. 如果固定按照MaxTenuringThreshold设定的阈值作为晋升条件: MaxTenuringThreshold设置的过大，原本应该晋升的对象一直停留在Survivor区，直到Survivor区溢出，一旦溢出发生，Eden+Svuvivor中对象将不再依据年龄全部提升到⽼年代，这样对象老化的机制就失效了。MaxTenuringThreshold设置的过小，“过早晋升”即对象不能在新生代充分被回收，⼤量短期对象被晋升到⽼年代，⽼年代空间迅速增⻓，引起频繁的Major GC。分代回收失去了意义，严重影响GC性能。
2. 相同应⽤在不同时间的表现不同:特殊任务的执行或者流量成分的变化，都会导致对象的生命周期 分布发⽣波动，那么固定的阈值设定，因为⽆法动态适应变化，会造成和上面相同的问题。
总结来说，为了更好的适应不同程序的内存情况，虚拟机并不总是要求对象年龄必须达到 Maxtenuringthreshhold再晋级⽼年代。

### 案例二：请求⾼峰期发生GC，导致服务可⽤性下降
#### 确定目标
由于存在Stop-The-World(简称STW)，即在执⾏垃圾回收时，Java应⽤程序中除了垃圾回收器器线程之外其他所有线程都被挂起，意味着在此期间，⽤户正常⼯作的线程全部被暂停下来，这是低延时服务不能接受的。本次优化目标是降低Remark时间。
#### 优化
解决问题前，先回顾一下CMS的四个主要阶段，以及各个阶段的工作内容。下图展示了CMS各个阶段可 以标记的对象，用不同颜色区分。
1. Init-mark初始标记(STW)，该阶段进行可达性分析，标记GC ROOT能直接关联到的对象，所以很 快。
2. Concurrent-mark并发标记，由前阶段标记过的绿色对象出发，所有可到达的对象都在本阶段中标记。
3. Remark重标记(STW)，暂停所有⽤户线程，重新扫描堆中的对象，进行可达性分析，标记活着的对象。因为并发标记阶段是和⽤户线程并发执行的过程，所以该过程中可能有用户线程修改某些活跃对象的字段，指向了一个未标记过的对象，如下图中红⾊对象在并发标记开始时不可达，但是并行期间引⽤发生变化，变为对象可达，这个阶段需要重新标记出此类对象，防⽌在下一阶段被清理掉，这个过程也是需要STW的。特别需要注意⼀点，这个阶段是以新生代中对象为根来判断对象是否存活的。
4. 并发清理，进⾏并发的垃圾清理。
![222.png](https://i.loli.net/2020/05/21/dHtpVl3CePJk8L2.png)
我们来分析新⽣代对象的特点是“朝⽣夕灭”，这样如果Remark前执行⼀次Minor GC，大部分对象就会被回收。CMS就采⽤了这样的⽅式，在Remark前增加了一个可中断的并发预清理理(CMS-concurrent-abortable-preclean)，该阶段主要工作仍然是并发标记对象是否存活，只是这个过程可被中断。如果此阶段执⾏时等到了Minor GC，那么上述灰⾊对象将被回收，Reamark阶段需要扫描的对象就少了。
Remark阶段主要是通过扫描堆来判断对象是否存活。CMS对⽼年代做回收，但是扫描的话是针对全堆扫描(新⽣代+年老代)。
除此之外CMS为了避免这个阶段没有等到Minor GC而陷⼊⽆限等待，提供了参数 CMSMaxAbortablePrecleanTime ，默认为5s，含义是如果可中断的预清理执⾏超过5s，不管发没发生Minor GC，都会中⽌此阶段，进入Remark。
所以针对于此，CMS提供CMSScavengeBeforeRemark参数，⽤来保证Remark前强制进行⼀次Minor GC。
CMS在Remark阶段扫描整个堆，同时为了避免扫描时新生代有很多对象，我们的调优策略是，通过参数强制Remark前进行一次Minor GC，从而降低Remark阶段的时间。
总结⼀下就是CMS GC会以新生代作为GC Root的⼀部分，因为在Remark之前过⼀个Minor GC，可以回收掉大部分的对象，从而减少GC Root扫描的开销。如果Remark不是性能瓶颈，那么这个参数也可以设置，默认false，因为毕竟多了⼀次Minor GC。

## 总结
结合上述GC优化案例做个总结:
1. ⾸先需要明确，在进⾏GC优化之前，是确认项目的涉及和代码等已经没有优化空间。我们不能指望一个系统架构有缺陷或者代码层次优化没有穷尽的应⽤，通过GC优化令其性能达到⼀个质的⻜跃。
2. 其次，通过上述分析，可以看出虚拟机内部已有很多优化来保证应用的稳定运行，所以不要为了调优而调优，不当的调优可能适得其反。
3. 最后，GC优化是一个系统而复杂的工作，没有万能的调优策略可以满足所有的性能指标。GC优化 必须建立在我们深入理解各种垃圾回收器的基础上，才能有事半功倍的效果。