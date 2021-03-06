# Java Garbage Collector

### 第一阶段：Serial 收集器

Serial收集器是最基本、发展最悠久的收集器，曾经（jdk 1.3.1之前）是唯一的虚拟机**新生代收集器**。它是虚拟机运行在Client模式下的默认新生代收集器。

它最大的特性是单线程。这不仅是只利用一个CPU或者一个线程，此外，在进行垃圾收集的时候，必须停止所有其他的工作线程，直到垃圾收集结束。由于没有线程交互的开销，Serial简单而高效。运行过程如下。

![Serial收集器运行过程](https://pic.yupoo.com/crowhawk/6b90388c/6c281cf0.png)

需要使用Serial收集器可以用以下参数：`-XX:+UseSerialGC`。

Serial收集器有一些变种，如老年代版本Serial Old收集器：单线程，使用“标记-整理”算法，工作流程和Serial收集器大致相同；多线程版本ParNew收集器：新生代收集器，只有它能与CMS收集器配合工作。虽然是多线程，但许多部分与Serial收集器重合。

### 第二阶段：Parallel 收集器

Parallel收集器是一个**并行**的**多线程新生代收集器**。可以充分利用多核的特性，大幅降低GC时间。

CMS等收集器关注重点在于尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel收集器关注如何达到一个**可控制的吞吐量**。也称之为**吞吐量优先**收集器。高吞吐量的收集器可以高效率地利用CPU时间，尽快完成程序运算任务，该收集器适合主要是后台运算而不需要太多交互的任务。运行过程如下。

![](https://pic.yupoo.com/crowhawk/fffcf9a2/f60599b2.png)

此外，Parallel收集器具有自适应调节策略，即让虚拟机根据当前系统的性能监控信息，自动调节参数，如新生代的大小、晋升老年代的年龄等。可以通过`-XX:+UseAdaptiveSizePolicy`选项开通。

使用Parallel收集器可以使用以下参数：`-XX:+UseParallelGC -XX:+UseParallelOldGC`。

Parallel收集器有老年代收集器：Parallel Old 收集器，使用多线程和“标记-整理”算法，经常与Parallel收集器搭配使用，但出现时间晚于Parallel收集器。

在JDK 9 之前Parallel 收集器是默认收集器。虽然它有高吞吐量，但可能会造成较高的暂停时间。JDK 9 用G1收集器代替了Parallel收集器，因为它有更好的并发性、更短的延迟时间（可能会导致降低吞吐量）、更少碎片化。

### 第三阶段：CMS收集器

CMS（Concurrent Mark Sweep）收集器是以获取最短回收停顿时间为目的的收集器。它使用“标记-清除”算法。这是一个非常优秀的收集器。

它分为五个阶段：初始标记、并发标记、再次标记、并发清理、重置。每个阶段的工作如下表。

| 阶段                            | 说明                                       |
| ----------------------------- | ---------------------------------------- |
| (1) 初始标记 (Initial Mark)       | (Stop the World Event,所有应用线程暂停) 在老年代(old generation)中的对象, 如果从年轻代(young generation)中能访问到, 则被 “标记,marked” 为可达的(reachable).对象在旧一代“标志”可以包括这些对象可能可以从年轻一代。暂停时间一般持续时间较短,相对小的收集暂停时间. |
| (2) 并发标记 (Concurrent Marking) | 在Java应用程序线程运行的同时遍历老年代(tenured generation)的可达对象图。扫描从被标记的对象开始,直到遍历完从root可达的所有对象. 调整器(mutators)在并发阶段的2、3、5阶段执行,在这些阶段中新分配的所有对象(包括被提升的对象)都立刻标记为存活状态. |
| (3) 再次标记(Remark)              | (Stop the World Event, 所有应用线程暂停) 查找在并发标记阶段漏过的对象，这些对象是在并发收集器完成对象跟踪之后由应用线程更新的. |
| (4) 并发清理(Concurrent Sweep)    | 回收在标记阶段(marking phases)确定为不可及的对象. 死对象的回收将此对象占用的空间增加到一个空闲列表(free list),供以后的分配使用。死对象的合并可能在此时发生. 请注意,存活的对象并没有被移动. |
| (5) 重置(Resetting)             | 清理数据结构,为下一个并发收集做准备.                      |

耗时最长的阶段是并发标记和并发清除阶段。它们可以与用户线程一起工作。在 Minor GC 时会暂停所有的应用进程，在 Full GC 时不再暂停，只是定期扫描老年代空间。总体来看，CMS收集器的内存回收过程是与用户线程一起并发执行的。所以所需的停顿时间极短。

![img](https://pic.yupoo.com/crowhawk/fffcf9a2/f60599b2.png)

下面我们具体来了解CMS垃圾回收的工作流程。

首先是堆的结构。CMS将堆划分成三个部分：一个新生代空间(Eden)、两块存活区空间(survivor spaces)、一大块连续的空间作为老年代空间(Old generation)。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/03_1_CMS_Heap_Structure_CN.png)

然后是新生代GC。如同之前介绍的分代垃圾回收的工作方式，新生代空间和一个存活区空间满后，将其中的存活对象拷贝到另一个空的存活区空间。存活时间达到阈值的对象被拷贝到老年代空间。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/03_3_Yong_Generation_Collection_CN.png)

GC完成后堆空间如图所示。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/03_4_After_Young_GC_CN.png)

老年代GC回收使用并发清理的方式，不对内存进行压缩。将没有标记的对象内存回收。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/03_6_Concurrent_Sweep_CN.png)

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/03_7_After_Sweeping_CN.png)

CMS的最大特点就是垃圾收集时极短的停顿时间，所以非常适用于互联网或者B/S系统的服务端的Java应用，这些应用重视服务的响应速度。

然而，它也有些不可忽略的缺点：对CPU资源十分敏感，并发清理时新产生的垃圾（也称为浮动垃圾）当次无法回收、会产生大量空间碎片。

使用CMS收集器可以使用以下参数：`-XX:+UseParNewGC -XX:+UseConcMarkSweepGC`。

### 第四阶段： G1收集器

G1（Garbage First）收集器由Hotspot团队开发，在 2012年 jdk 7 update 4 后可用。oracle计划在 jdk9 中将G1变成默认的垃圾收集器，以替代CMS。它的执行过程如下图。

![img](https://pic.yupoo.com/crowhawk/53b7a589/0bce1667.png)

G1的优点包括以下几点：

- **并行与并发**：G1能充分利用多CPU、多核环境下的硬件优势，降低停顿时间。此外，G1收集器可以通过并发的方式，在进行垃圾收集的时候不停止原有java程序的运行。
- **分代收集**：分代概念在G1收集器得以保留。它不需要其他GC收集器配合来管理整个GC堆。此外，它采用不同方式处理新生代和老年代，取消新生代和老年代的物理空间划分。
- **空间整合**：G1收集器整体基于“标记-整理”算法，但局部实现是基于“复制”算法（收集器将java堆划分成多个等大的独立区域，称为Region。局部指两个Region的收集）。这保证了运行收集器时不会产生空间碎片。
- **可预测的停顿**：CMS和G1都关注于如何降低停顿时间，但G1还建立了可预测的停顿时间模型，使用者可以明确指定一个时间片N内，消耗在垃圾收集上的时间不超过M。

#### 堆空间分配

前面已经提到，新生代和老年代不再使用独立的物理空间，而是将内存划分成多个等大独立的Region，每个Region逻辑上连续，每个区可以根据需要，分配给新生代、存活区、老年代。这使得内存使用有更好的灵活性。此外，每个Region都维护了一个Remembered Set，避免了全堆扫描。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/02_2_G1HeapAllocation_CN.png)

此外，还有一种特殊的区域：Humongous，专门用来存放巨型对象（占用空间超过分区容量50%以上）。如果一个分区不能容纳巨型对象，G1会寻找连续分区来存放。

基于这样的堆空间划分，G1分配对象可以分为三个阶段：TLAB（Thread Local Allocation Buffer）区中分配、Eden区中分配、Humongous区分配。TLAB是线程本地缓冲区，位于Eden空间中。在共享空间中分配对象，需要同步机制来管理这些空闲空间指针，为了减小同步的开销，更快地分配空间，每个线程拥有自己的空间，可直接分配不需要同步。TLAB空间不足以分配对象时，会转到Eden空间分配。Eden空间分配也失败的话，只能在老年代中分配。最后巨型对象在Humongous区分配。

#### 新生代GC

它主要是针对Eden区进行垃圾回收，在Eden区耗尽时触发。Eden空间的数据移动到Survivor空间，如果Survivor空间不足，部分直接晋升到老年代空间中。最后Eden区清空。

在Eden区执行垃圾回收的过程中需要用到之前提到的Remembered Set。在CMS中也有Rset，它存在于老年代空间中，记录了所有老年代对新生代的引用（point-out）。因为G1中分区过多，一个分区过小，扫描全部分区耗时过多。它采取的策略是每个新生代空间记录老年代对其引用（point-in）。这就避免了全局扫描。为了解决引用过多时处理的困难，G1采用了卡片标记方式（Ungar的垃圾回收中提到），可以降低赋值器开销。

Young-GC可以分为五个阶段：

+ **根扫描**：扫描所有静态和本地对象。

+ **更新Rset**：处理被引用的对象，更新Rset。

+ **处理Rset**：检测新生代对老年代对象的引用。

+ **对象拷贝**：将存活对象拷贝到Survivor区和老年代区，拷贝的同时进行压缩，不会出现空间碎片。

+ **处理引用队列**：包括软引用、弱引用、虚引用。

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_4_A_Young_GC_in_G1_CN.png)

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_5_End_of_Young_GC_with_G1_CN.png)

#### Mix GC

Mix GC不仅进行正常的新生代垃圾收集，同时也回收部分后台扫描线程标记的老年代分区。

GC算法分为两个阶段：全局并发标记、拷贝存活对象。

全局并发标记分为五个阶段：

+ **初始标记**：标记GC Roots能直接关联到的对象，修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象。这有一段很短的线程停顿时间。

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_6_Initial_Marking_Phase_CN.png)

+ **根区域扫描**：在初始标记的存活区扫描对老年代的引用，并标记被引用的对象。该阶段与用户程序并发执行，完成后才能进行下一次年轻代垃圾回收。

+ **并发标记**：从GC Roots开始标记所有可达的存活对象，耗时较长，与用户程序并发执行。

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_7_Concurrent_Marking_Phase_CN.png)

+ **再次标记**：在并发标记阶段，由于用户程序的运行，使得有些标记记录产生变动（之前提到的浮动垃圾），这个阶段主要处理这些记录。这个阶段需要停止线程，但可以并行执行。

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_8_Remark_Phase_CN.png)

+ **筛选回收**：对各个Region的回收价值和成本进行排序，根据用户所选定的停顿时间来制定回收计划。这个阶段也可以与用户程序并发执行。但停顿线程可以大幅提高回收效率。

  ![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_9_Copying_Cleanup_Phase_CN.png)

执行完MIX GC之后的堆空间如图所示。

![img](https://github.com/cncounter/translation/raw/master/tiemao_2014/G1/04_10_After_Copying_Cleanup_Phase_CN.png)

可以总结出其几个关键点。在**并发标记阶段**，活跃度信息（标识出哪些区域在转移暂停时最适合回收）在程序运行时并发计算得到，该阶段不进行清楚操作。在**再次标记阶段**，使用SATB（Snapshot-at-the-Beginning ）算法，比起CMS的增量更新（Incremental update）更高效。在**筛选回收阶段**，年轻代与老年代同时回收，老年代的选择基于活跃度。

#### 命令行参数

| 选项/默认值                                | 说明                                       |
| ------------------------------------- | ---------------------------------------- |
| -XX:+UseG1GC                          | 使用 G1 垃圾收集器                              |
| -XX:MaxGCPauseMillis=n                | 设置最大GC停顿时间， 这是一个软性指标, JVM 会尽量去达成这个目标.    |
| - XX:InitiatingHeapOccupancyPercent=n | 启动并发GC周期时的堆内存占用百分比。G1之类的垃圾收集器用它来触发并发GC周期，基于整个堆的使用率，而不只是某一代内存的使用比。值为 0 则表示"一直执行GC循环". 默认值为 45。 |
| -XX:NewRatio=n                        | 新生代与老生代的大小比例. 默认值为 2.                    |
| -XX:SurvivorRatio=n                   | Eden/Survivor 空间大小的比例。 默认值为 8.           |
| -XX:MaxTenuringThreshold=n            | 提升年老代的最大临界值。 默认值为 15.                    |
| -XX:ParallelGCThreads=n               | 设置垃圾收集器在并行阶段使用的线程数,默认值随JVM运行的平台不同而不同.    |
| -XX:ConcGCThreads=n                   | 并发垃圾收集器使用的线程数量. 默认值随JVM运行的平台不同而不同.       |
| -XX:G1ReservePercent=n                | 设置堆内存保留为假天花板的总量，以降低提升失败的可能性。默认值是 10.     |
| -XX:G1HeapRegionSize=n                | 使用G1时Java堆会被分为大小统一的的区。此参数可以指定每个heap区的大小. 默认值将根据 heap size 算出最优解。最小值为 1Mb, 最大值为32Mb。 |

####  G1的GC日志

G1可以设置三种级别的日志。

(1) -verbosegc (等价于 -XX:+PrintGC) 设置日志级别为fine。一个样例输出结果如下。

```
[GC pause (G1 Humongous Allocation) (young) (initial-mark) 24M- >21M(64M), 0.2349730 secs]
[GC pause (G1 Evacuation Pause) (mixed) 66M->21M(236M), 0.1625268 secs]  
```

(2) -XX:+PrintGCDetails 设置日志级别为finer。它能输出每个阶段的 Average，Min, 以及 Max 时间、根扫描(Root Scan)， “other” 执行时间、显示 Eden, Survivors 以及总的 Heap 占用信息等。一个样例输出结果如下。

```
[Ext Root Scanning (ms): Avg: 1.7 Min: 0.0 Max: 3.7 Diff: 3.7]
[Eden: 818M(818M)->0B(714M) Survivors: 0B->104M Heap: 836M(4096M)->409M(4096M)]
```

(3)  -XX:+UnlockExperimentalVMOptions -XX:G1LogLevel=finest 设置日志级别为finest。和 finer 级别类似，输出每个 worker 线程信息。一个样例输出结果如下。

```
[Ext Root Scanning (ms): 2.1 2.4 2.0 0.0
           Avg: 1.6 Min: 0.0 Max: 2.4 Diff: 2.3]
       [Update RS (ms):  0.4  0.2  0.4  0.0
           Avg: 0.2 Min: 0.0 Max: 0.4 Diff: 0.4]
           [Processed Buffers : 5 1 10 0
           Sum: 16, Avg: 4, Min: 0, Max: 10, Diff: 10]
```

## Reference

[深入理解 Java G1 垃圾收集器](http://blog.jobbole.com/109170/)

[Garbage-First Garbage Collection](https://dl.acm.org/citation.cfm?id=1029879)

[深入理解JVM(3)——7种垃圾收集器](https://crowhawk.github.io/2017/08/15/jvm_3/)

[What’s the difference between concurrency and parallelism](http://joearms.github.io/2013/04/05/concurrent-and-parallel-programming.html)

[Getting Started with the G1 Garbage Collector](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)，[翻译文章](http://blog.csdn.net/renfufei/article/details/41897113)