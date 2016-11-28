# 4. GC 算法(实现篇)



Now that we have reviewed the core concepts behind GC algorithms, let us move to the specific implementations one can find inside the JVM. An important aspect to recognize first is the fact that, for most JVMs out there, two different GC algorithms are needed – one to clean the Young Generation and another to clean the Old Generation.

学习了GC算法的相关概念之后, 我们将介绍在JVM中这些算法的具体实现。首先要记住的是, 大多数JVM都需要使用两种不同的GC算法 —— 一种用来清理年轻代, 另一种用来清理老年代。


You can choose from a variety of such algorithms bundled into the JVM. If you do not specify a garbage collection algorithm explicitly, a platform-specific default will be used. In this chapter, the working principles of each of those algorithms will be explained.

我们可以选择JVM内置的各种算法。如果不通过参数明确指定垃圾收集算法, 则会使用宿主平台的默认实现。本章会详细介绍各种算法的实现原理。


For a quick cheat sheet, the following list is a fast way to get yourself up to speed with which algorithm combinations are possible. Note that this stands true for Java 8, for older Java versions the available combinations might differ a bit:

下面是一个概要列表, 列出了各种可能的组合方式。但需要注意,这是Java 8中的列表,对于之前的Java版本来说,可用组合会有一些不同:

#### 校对到此处

<table>
	<thead>
		<tr>
			<th><b>Young</b></th>
			<th><b>Tenured</b></th>
			<th><b>JVM options</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>Incremental(增量GC)</td>
			<td>Incremental</td>
			<td>-Xincgc</td>
		</tr>
		<tr>
			<td><b>Serial</b></td>
			<td><b>Serial</b></td>
			<td><b>-XX:+UseSerialGC</b></td>
		</tr>
		<tr>
			<td>Parallel Scavenge</td>
			<td>Serial</td>
			<td>-XX:+UseParallelGC -XX:-UseParallelOldGC</td>
		</tr>
		<tr>
			<td>Parallel New</td>
			<td>Serial</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td>Serial</td>
			<td>Parallel Old</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td><b>Parallel Scavenge</b></td>
			<td><b>Parallel Old</b></td>
			<td><b>-XX:+UseParallelGC -XX:+UseParallelOldGC</b></td>
		</tr>
		<tr>
			<td>Parallel New</td>
			<td>Parallel Old</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td>Serial</td>
			<td>CMS</td>
			<td>-XX:-UseParNewGC -XX:+UseConcMarkSweepGC</td>
		</tr>
		<tr>
			<td>Parallel Scavenge</td>
			<td>CMS</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td><b>Parallel New</b></td>
			<td><b>CMS</b></td>
			<td><b>-XX:+UseParNewGC -XX:+UseConcMarkSweepGC</b></td>
		</tr>
		<tr>
			<td colspan="2" align="middle"><b>G1</b></td>
			<td><b>-XX:+UseG1GC</b></td>
		</tr>
	</tbody>
</table>




If the above looks too complex, do not worry. In reality it all boils down to just four combinations highlighted in the table above. The rest are either deprecated, not supported or just impractical to apply in real world. So, in the following chapters we cover the working principles of the following combinations:

如果看起来太复杂,不要担心。实际使用的主要是上表中黑体字显示的这四种组合。其余的要么是弃用(deprecated),要么是不支持或者是不适用于生产环境。所以,接下来我们只介绍下面这些组合以及工作原理:


- Serial GC for both the Young and Old generations
- Parallel GC for both the Young and Old generations
- Parallel New for Young + Concurrent Mark and Sweep (CMS) for the Old Generation
- 
- 
- 
- 
- , which encompasses collection of both Young and Old generations

<br/>

- 年轻代和老年代的串行GC(Serial GC)
- 年轻代和老年代的并行GC(Parallel GC)
- 年轻代的并行GC() + 老年代的CMS(Concurrent Mark and Sweep)
- G1, 包括年轻代和老年代的集合



## 串行GC(Serial GC)


This collection of garbage collectors uses mark-copy for the Young Generation and mark-sweep-compact for the Old Generation. As the name implies – both of these collectors are single-threaded collectors, incapable of parallelizing the task at hand. Both collectors also trigger stop-the-world pauses, stopping all application threads.

这一类垃圾收集器针对年轻代使用 mark-copy(标记-复制) 算法, 对老年代使用 mark-sweep-compact(标记-清除-整理)算法. 顾名思义,两者都是单线程的垃圾收集器,不能进行并行处理。两者都会触发全线暂停(STW),停止所有的应用程序线程。


This GC algorithm cannot thus take advantage of multiple CPU cores commonly found in modern hardware. Independent of the number of cores available, just one is used by the JVM during garbage collection.

因此这种GC算法不能充分利用多核CPU。不管有多少个可用内核, JVM在垃圾收集期间都只会使用单个核心。


Enabling this collector for both the Young and Old Generation is done via specifying a single parameter in the JVM startup script:

要启用这个收集器, 只需要在JVM启动脚本中指定一个参数即可,同时对年轻代和老年代生效:


	java -XX:+UseSerialGC com.mypackages.MyExecutableClass




This option makes sense and is recommended only for the JVM with a couple of hundreds megabytes heap size, running in an environment with a single CPU. For the majority of server-side deployments this is a rare combination. Most server-side deployments are done on platforms with multiple cores, essentially meaning that by choosing Serial GC you are setting artificial limits on the use of system resources. This results in idle resources which otherwise could be used to reduce latency or increase throughput.

这个选项只在JVM堆内存只有几百MB,而且是单核CPU时有意义,而且只在这种情况下推荐使用. 对于大多数服务器,因为是多核CPU, 一般不推荐这样使用, 除非你需要限制JVM所使用的资源。大多数服务器端部署与多核平台上完成,本质上意味着通过选择串行GC你设置人工限制系统资源的使用. 最简单的就是导致资源闲置,不能有效降低延迟或增加吞吐量。


Let us now review how garbage collector logs look like when using Serial GC and what useful information one can obtain from there. For this purpose, we have turned on GC logging on the JVM using the following parameters:

现在让我们看看Serial GC的垃圾收集器日志, 以及从中可用提取什么有用的信息。为了这个目的, 我们使用下面的参数来打开GC日志记录:


	-XX:+PrintGCDetails -XX:+PrintGCDateStamps 
	-XX:+PrintGCTimeStamps




The resulting output is similar to the following:

由此产生的输出类似下面这样:


	2015-05-26T14:45:37.987-0200: 
			151.126: [GC (Allocation Failure) 
			151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs] 
			1619346K->1273247K(2027264K), 0.0585007 secs]
		[Times: user=0.06 sys=0.00, real=0.06 secs]
	2015-05-26T14:45:59.690-0200: 
			172.829: [GC (Allocation Failure) 
			172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]
			172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs] 
			1832479K->755802K(2027264K), 
			[Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs] 
		[Times: user=0.18 sys=0.00, real=0.18 secs]



Such short snippet from the GC logs exposes a lot of information about what is taking place inside the JVM. As a matter of fact, in this snippet there were two Garbage Collection events taking place, one of them cleaning the Young Generation and another taking care of the entire heap. Let’s start by analyzing the first collection that is taking place in the Young Generation.

这样一个简短的GC日志片段,暴露了发生在JVM中的很多信息。 事实上,在这段日志中有两次GC事件发生, 其中一次清理的是年轻代,另一次处理的是整个堆内存。让我们先来分析发生在年轻代的第一次GC。


### 小型GC(Minor GC)


Following snippet contains the information about a GC event cleaning the Young Generation:

以下代码片段包含信息GC事件打扫年轻代:




> <a>`2015-05-26T14:45:37.987-0200`<sup>1</sup></a> : <a>`151.1262`<sup>2</sup></a> : [ <a>`GC`<sup>3</sup></a> (<a>`Allocation Failure`<sup>4</sup></a> 151.126: <br/>
> [<a>`DefNew`<sup>5</sup></a> : <a>`629119K->69888K`<sup>6</sup></a> <a>`(629120K)`<sup>7</sup></a> , 0.0584157 secs] <a>`1619346K->1273247K`<sup>8</sup></a> <br/>
> <a>`(2027264K)`<sup>9</sup></a>, <a>`0.0585007 secs`<sup>10</sup></a>] <a>`[Times: user=0.06 sys=0.00, real=0.06 secs]`<sup>11</sup></a>



> 1. <a>`2015-05-26T14:45:37.987-0200`</a> – Time when the GC event started. GC事件开始的时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`。
>
> 2. <a>`151.126`</a> – Time when the GC event started, relative to the JVM startup time. Measured in seconds. 相对于JVM启动时间,GC事件开始的时间,单位是秒。
> 
> 3. <a>`GC`</a> – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC. 用来区分 Minor GC 还是 Full GC 的标志。`GC`表明这是一次**小型GC**(Minor GC)
> 
> 4. <a>`Allocation Failure`</a> – Cause of the collection. In this case, the GC is triggered due to a data structure not fitting into any region in the Young Generation. 触发垃圾收集的原因。本次GC事件, 是由于年轻代中没有适当的空间存放新的数据结构引起的。
> 
> 5. <a>`DefNew`</a> – Name of the garbage collector used. This cryptic name stands for the single-threaded mark-copy stop-the-world garbage collector used to clean Young generation. 垃圾收集器的名称。这个神秘的名字表示的是在年轻代中使用的: 单线程, 标记-复制(mark-copy), 全线暂停(STW) 垃圾收集器。
> 
> 6. <a>`629119K->69888K`</a> – Usage of the Young Generation before and after collection. 在垃圾收集之前和之后的年轻代使用量。
> 
> 7. <a>`(629120K)`</a> – Total size of the Young Generation. 年轻代的总大小。
> 
> 8. <a>`1619346K->1273247K`</a> – Total used heap before and after collection. 在垃圾收集之前和之后堆内存的使用情况。
> 
> 9. <a>`(2027264K)`</a> – Total available heap. 可用堆的总大小。
> 
> 10. <a>`0.0585007 secs`</a> – Duration of the GC event in seconds. GC事件持续的时间,以秒为单位。
> 
> 11. <a>`[Times: user=0.06 sys=0.00, real=0.06 secs]`</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
> 
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间

> 
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
> 
 - real – Clock time for which your application was stopped. As Serial Garbage Collector always uses just a single thread, real time is thus equal to the sum of user and system times. 应用程序被停止的系统时钟时间。因为串行垃圾收集器(Serial Garbage Collector)只使用单线程, 因此 real time 等于 user 和 system 时间的总和



From the above snippet we can understand exactly what was happening with the memory consumption inside JVM during the GC event. Before this collection, heap usage totaled at 1,619,346K. Out of this, the Young Generation consumed 629,119K. From this we can calculate the Old Generation usage being equal to 990,227K.

我们可以从上面的日志片段了解到, 在GC事件中,JVM里面的内存使用情况到底发生了什么变化。在此次垃圾收集之前, 堆内存总的使用了 **1,619,346K**。其中,年轻代使用了 **629,119K**。并可以计算出老年代使用量等于 **990,227K**。


A more important conclusion is hidden in the next batch of numbers indicating that, after the collection, Young Generation usage decreased by 559,231K but total heap usage decreased only by 346,099K. From this we can again derive that 213,132K of objects were promoted from the Young Generation to the Old Generation.

更重要的结论隐藏在下一批数字之中, 垃圾收集之后, 年轻代(Young Generation)的使用量减少了 **559231 k**, 但总的堆内存使用量只下降了 **346099 k**. 从中我们可以计算出有 **213132 k** 的对象从年轻代提升到了老年代的。


This GC event is also illustrated with the following snapshots showing memory usage right before the GC started and right after it finished:

这次GC事件也可用下面的示意图来说明, 显示的是GC开始之前, 以及刚刚结束之后, 这两个时刻的内存使用快照:


![](04_01_serial-gc-in-young-generation.png)




### Full GC


After understanding the first minor GC event, lets look into the second GC event in the logs:


在理解了第一个小型GC事件后,让我们看看日志中的第二个GC事件:




> <a>`2015-05-26T14:45:59.690-0200`<sup>1</sup></a> : <a>`172.829`<sup>2</sup></a> : [GC (Allocation Failure 172.829: <br/>
> <a>`[DefNew: 629120K->629120K(629120K), 0.0000372 secs`<sup>3</sup></a>] 172.829:[<a>`Tenured`<sup>4</sup></a>: <br/>
> <a>`1203359K->755802K`<sup>5</sup></a> <a>`(1398144K)`<sup>6</sup></a>, <a>`0.1855567 secs`<sup>7</sup></a> ] <a>`1832479K->755802K`<sup>8</sup></a> <br/>
> <a>`(2027264K)`<sup>9</sup></a>, <a>`[Metaspace: 6741K->6741K(1056768K)]`<sup>10</sup></a> <br/>
> <a>`[Times: user=0.18 sys=0.00, real=0.18 secs]`<sup>11</sup></a>




> 
> 1. <a>`2015-05-26T14:45:59.690-0200`</a> – Time when the GC event started. GC事件开始的时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`。
> 1. <a>`172.829`</a> – Time when the GC event started, relative to the JVM startup time. Measured in seconds. 相对于JVM启动时间,GC事件开始的时间,单位是秒。
> 1. <a>`[DefNew: 629120K->629120K(629120K), 0.0000372 secs`</a> – Similar to the previous example, a minor garbage collection in the Young Generation happened during this event due to Allocation Failure. For this collection the same DefNew collector was run as before and it decreased the usage of the Young Generation from 629120K to 0. Notice that JVM reports this incorrectly due to buggy behavior and instead reports the Young Generation as being completely full. This collection took 0.0000372 seconds.
> 1. <a>`Tenured`</a> – Name of the garbage collector used to clean the Old space. The name Tenured indicates a single-threaded stop-the-world mark-sweep-compact garbage collector being used.  用于清理老年代空间的垃圾收集器名称。名称 **Tenured** 表明使用的是单线程的垃圾收集器,STW, 算法为 标记-清除-整理(mark-sweep-compact )。
> 1. <a>`1203359K->755802K`</a>  – Usage of Old generation before and after the event. 在垃圾收集之前和之后的老年代使用量。
> 1. <a>`(1398144K)`</a>  – Total capacity of the Old generation. 老年代的总大小。
> 1. <a>`0.1855567 secs`</a> – Time it took to clean the Old Generation. 清理老年代所花的时间。
> 1. <a>`1832479K->755802K`</a> – Usage of the whole heap before and after the collection of the Young and Old Generations. 在垃圾收集之前和之后堆内存的使用情况。
> 1. <a>`(2027264K)`</a> – Total heap available for the JVM. 可用堆的总大小。
> 1. <a>`[Metaspace: 6741K->6741K(1056768K)]`</a> – Similar information about Metaspace collection. As seen, no garbage was collected in Metaspace during the event.  Metaspace 空间中类似的信息。可以看到, 在GC事件中 Metaspace 中没有回收到任何垃圾。
> 1. <a>`[Times: user=0.18 sys=0.00, real=0.18 secs]`</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
 - real – Clock time for which your application was stopped. As Serial Garbage Collector always uses just a single thread, real time is thus equal to the sum of user and system times. 应用程序被停止的系统时钟时间。因为串行垃圾收集器(Serial Garbage Collector)只使用单线程, 因此 real time 等于 user 和 system 时间的总和.



The difference with Minor GC is evident – in addition to the Young Generation, during this GC event the Old Generation and Metaspace were also cleaned. The layout of the memory before and after the event would look like the situation in the following picture:

和 Minor GC 的区别是很明显的 —— 在此次GC事件中, 除了年轻代, 还清理了老年代和Metaspace. 在GC事件前后的内存布局如下图所示:


![](04_02_serial-gc-in-old-gen-java.png)




## 并行GC(Parallel GC)


This combination of Garbage Collectors uses mark-copy in the Young Generation and mark-sweep-compact in the Old Generation. Both Young and Old collections trigger stop-the-world events, stopping all application threads to perform garbage collection. Both collectors run marking and copying / compacting phases using multiple threads, hence the name ‘Parallel’. Using this approach, collection times can be considerably reduced.

这种垃圾收集器在年轻代使用 标记-复制(mark-copy)算法, 在老年代使用 标记-清除-整理(mark-sweep-compact)算法。年轻代和老年代的垃圾回收都会触发STW事件,暂停所有的应用程序线程再执行垃圾收集。两者在执行 标记-复制/整理阶段时都使用多个线程, 因此得名“**并行(Parallel)**”。通过使用并行方式, 使得GC时间可以大幅减少。


The number of threads used during garbage collection is configurable via the command line parameter -XX:ParallelGCThreads=NNN . The default value is equal to the number of cores in your machine.

垃圾收集时使用的线程数通过命令行参数 `-XX:ParallelGCThreads=NNN` 配置。 默认值等于机器上CPU内核的数量。


Selection of Parallel GC is done via the specification of any of the following combinations of parameters in the JVM startup script:

下面的命令行参数, 任何一组都可以用来指定并行GC:


	java -XX:+UseParallelGC com.mypackages.MyExecutableClass
	java -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
	java -XX:+UseParallelGC -XX:+UseParallelOldGC com.mypackages.MyExecutableClass




Parallel Garbage Collector is suitable on multi-core machines in cases where your primary goal is to increase throughput. Higher throughput is achieved due to more efficient usage of system resources:

并行垃圾收集器适用于在多核服务器上,主要目标是增加吞吐量的情况下。因为对系统资源的有效使用,能达到更搞的吞吐量:


- during collection, all cores are cleaning the garbage in parallel, resulting in shorter pauses
- between garbage collection cycles neither of the collectors is consuming any resources

<br/>

- 在GC期间, 所有 CPU 内核都在并行地清理垃圾, 所以暂停时间更短
- 在两次GC周期之间, 没有任何垃圾收集器在运行,不会消耗任何系统资源



On the other hand, as all phases of the collection have to happen without any interruptions, these collectors are still susceptible to long pauses during which your application threads are stopped. So if latency is your primary goal, you should check the next combinations of garbage collectors.

另一方面, 因为GC的所有阶段都不能中断, 所以这些收集器很容易出现长时间的停顿. 如果延迟是系统的主要目标, 那么应该选择下面的其他垃圾收集器组合。


Let us now review how garbage collector logs look like when using Parallel GC and what useful information one can obtain from there. For this, let’s look again at the garbage collector logs that expose once more one minor and one major GC pause:

让我们看看如何并行垃圾收集器的GC日志长什么样, 从中我们可以获取什么有用的信息。下面的GC日志中显示了一次 minor GC 和一次 major GC 暂停:


	2015-05-26T14:27:40.915-0200: 116.115: [GC (Allocation Failure) 
			[PSYoungGen: 2694440K->1305132K(2796544K)] 
		9556775K->8438926K(11185152K)
		, 0.2406675 secs] 
		[Times: user=1.77 sys=0.01, real=0.24 secs]
	2015-05-26T14:27:41.155-0200: 116.356: [Full GC (Ergonomics) 
			[PSYoungGen: 1305132K->0K(2796544K)] 
			[ParOldGen: 7133794K->6597672K(8388608K)] 8438926K->6597672K(11185152K), 
			[Metaspace: 6745K->6745K(1056768K)]
		, 0.9158801 secs]
		[Times: user=4.49 sys=0.64, real=0.92 secs]



### 小型GC(Minor GC)


The first of the two events indicates a GC event taking place in the Young Generation:

第一次GC事件表示发生在年轻代的垃圾收集:


> <a>`2015-05-26T14:27:40.915-0200`<sup>1</sup></a>: <a>`116.115`<sup>2</sup></a>:  <a>`[ GC`<sup>3</sup></a> (<a>`Allocation Failure`<sup>4</sup></a>)<br/>
> [<a>`PSYoungGen`<sup>5</sup></a>: <a>`2694440K->1305132K`<sup>6</sup></a> <a>`(2796544K)`<sup>7</sup></a>] <a>`9556775K->8438926K`<sup>8</sup></a><br/>
> <a>`(11185152K)`<sup>9</sup></a>, <a>`0.2406675 secs`<sup>10</sup></a>]<br/>
> <a>`[Times: user=1.77 sys=0.01, real=0.24 secs]`<sup>11</sup></a>



>
> 1. <a>`2015-05-26T14:27:40.915-0200`</a> – Time when the GC event started. GC事件开始的时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`。
> 1. <a>`116.115`</a> – Time when the GC event started, relative to the JVM startup time. Measured in seconds. 相对于JVM启动时间,GC事件开始的时间,单位是秒。
> 1. <a>`GC`</a> – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC. 用来区分 Minor GC 还是 Full GC 的标志。`GC`表明这是一次**小型GC**(Minor GC)
> 1. <a>`Allocation Failure`</a> – Cause of the collection. In this case, the GC is triggered due to a data structure not fitting into any region in the Young Generation. 触发垃圾收集的原因。本次GC事件, 是由于年轻代中没有适当的空间存放新的数据结构引起的。
> 1. <a>`PSYoungGen`</a> – Name of the garbage collector used, representing a parallel mark-copy stop-the-world garbage collector used to clean the Young generation. 垃圾收集器的名称。这个名字表示的是在年轻代中使用的: 并行的 标记-复制(mark-copy), 全线暂停(STW) 垃圾收集器。
> 1. <a>`2694440K->1305132K`</a> – Usage of the Young Generation before and after collection. 在垃圾收集之前和之后的年轻代使用量。
> 1. <a>`(2796544K)`</a> – Total size of the Young Generation. 年轻代的总大小。
> 1. <a>`9556775K->8438926K`</a> – Total heap usage before and after collection. 在垃圾收集之前和之后整个堆内存的使用量。
> 1. <a>`(11185152K)`</a> – Total available heap. 可用堆的总大小。
> 1. <a>`0.2406675 secs`</a> – Duration of the GC event in seconds. GC事件持续的时间,以秒为单位。
> 1. <a>`[Times: user=1.77 sys=0.01, real=0.24 secs]`</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount. 应用程序被停止的系统时钟时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有固定比例的处理过程是不能并行化的。



So, in short, the total heap consumption before the collection was 9,556,775K. Out of this Young generation was 2,694,440K. This means that used Old generation was 6,862,335K. After the collection young generation usage decreased by 1,389,308K, but total heap usage decreased only by 1,117,849K. This means that 271,459K was promoted from Young generation to Old.

所以,简而言之, 垃圾收集前的堆内存总消耗量为 **9,556,775K**。 其中年轻代是 **2,694,440K**。同时可以算出老年代使用了 **6,862,335K**. 在垃圾收集之后, 年轻代使用量下降到 **1,389,308K**, 但总的堆内存使用量只下降了 `1,117,849K`。这表示有 **271,459K** 从年轻代被提升到老年代。


![](04_03_ParallelGC-in-Young-Generation-Java.png)





### 完全GC(Full GC)


After understanding how Parallel GC cleans the Young Generation, we are ready to look at how the whole heap is being cleaned by analyzing the next snippet from the GC logs:

年轻代了解并行GC清洁后,我们准备看看整个堆正在清理GC日志分析下一个片段:


> <a>`2015-05-26T14:27:41.155-0200`</a> : <a>`116.356`</a> : [<a>`Full GC`</a>  (<a>`Ergonomics`</a>)<br/>
> <a>`[PSYoungGen: 1305132K->0K(2796544K)]`</a> [<a>`ParOldGen`</a> :<a>`7133794K->6597672K `</a> <br/>
> <a>`(8388608K)`</a>] <a>`8438926K->6597672K`</a> <a>`(11185152K)`</a>, <br/>
> <a>`[Metaspace: 6745K->6745K(1056768K)] `</a>, <a>`0.9158801 secs`</a>,<br/>
> <a>`[Times: user=4.49 sys=0.64, real=0.92 secs]`</a>



> 1. <a>`2015-05-26T14:27:41.155-0200`</a> – Time when the GC event started. GC事件开始的时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`
> 1. <a>`116.356`</a> – Time when the GC event started, relative to the JVM startup time. Measured in seconds. In this case we can see the event started right after the previous Minor GC finished. 相对于JVM启动时间,GC事件开始的时间,单位是秒。 我们可以看到, 此次事件在前一次 MinorGC完成之后立刻就开始了。
> 1. <a>`Full GC`</a> – Flag indicating that the event is Full GC event cleaning both the Young and Old generations. 用来表示此次是 Full GC 的标志。`Full GC`表明本次清理的是年轻代和老年代
> 1. <a>`Ergonomics`</a> – Reason for the GC taking place. This indicates that the JVM internal ergonomics decided this is the right time to collect some garbage. 触发垃圾收集的原因。这表明JVM内部环境决定此刻适合进行一次垃圾收集。
> 1. <a>`[PSYoungGen: 1305132K->0K(2796544K)]`</a> – Similar to previous example, a parallel mark-copy stop-the-world garbage collector named “PSYoungGen” was used to clean the Young Generation. Usage of Young Generation shrank from 1305132K to 0, which is the typical result of a Full GC. 和上面的示例一样, 清理年轻代的垃圾收集器是名为 “PSYoungGen” 的STW收集器,采用**标记-复制**(mark-copy)算法。 年轻代使用量从 **1305132K** 变为 `0`, 一般 Full GC 的结果都是这样。
> 1. <a>`ParOldGen`</a> – Type of the collector used to clean the Old Generation. In this case, parallel mark-sweep-compact stop-the-world garbage collector named ParOldGen was used. 用于清理老年代空间的垃圾收集器类型。在这里使用的是名为 **ParOldGen** 的垃圾收集器, 这是一种并行的STW垃圾收集器, 算法为 标记-清除-整理(mark-sweep-compact)。
> 1. <a>`7133794K->6597672K `</a> – Usage of the Old Generation before and after the collection. 在垃圾收集之前和之后老年代内存的使用情况。
> 1. <a>`(8388608K)`</a> – Total size of the Old Generation. 老年代的总大小。
> 1. <a>`8438926K->6597672K`</a> – Usage of the whole heap before and after the collection. 在垃圾收集之前和之后堆内存的使用情况。
> 1. <a>`(11185152K)`</a> – Total heap available. 可用堆的总大小。
> 1. <a>`[Metaspace: 6745K->6745K(1056768K)] `</a> – Similar information about Metaspace region. As we can see, no garbage was collected in Metaspace during this event. Metaspace 空间中类似的信息。可以看到, 在GC事件中 Metaspace 中没有回收到任何垃圾。
> 1. <a>`0.9158801 secs`</a> – Duration of the GC event in seconds. GC事件持续的时间,以秒为单位。
> 1. <a>`[Times: user=4.49 sys=0.64, real=0.92 secs]`</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount. 应用程序被停止的系统时钟时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有固定比例的处理过程是不能并行化的。



Again, the difference with Minor GC is evident – in addition to the Young Generation, during this GC event the Old Generation and Metaspace were also cleaned. The layout of the memory before and after the event would look like the situation in the following picture:


同样,和 Minor GC 的区别是很明显的 —— 在此次GC事件中, 除了年轻代, 还清理了老年代和 Metaspace. 在GC事件前后的内存布局如下图所示:


![](04_04_Java-ParallelGC-in-Old-Generation.png)







## CMS(并发标记-清除)


The official name for this collection of garbage collectors is “Mostly Concurrent Mark and Sweep Garbage Collector”. It uses the parallel stop-the-world mark-copy algorithm in the Young Generation and the mostly concurrent mark-sweep algorithm in the Old Generation.

这种垃圾收集器的官方名称是 “**Mostly Concurrent Mark and Sweep Garbage Collector**”(主要并发标记-清除垃圾收集器). 对年轻代使用并行的 STW方式的 mark-copy 算法, 在老年代主要使用并发标记-清除算法(mark-sweep)。


This collector was designed to avoid long pauses while collecting in the Old Generation. It achieves this by two means. Firstly, it does not compact the Old Generation but uses free-lists to manage reclaimed space. Secondly, it does most of the job in the mark-and-sweep phases concurrently with the application. This means that garbage collection is not explicitly stopping the application threads to perform these phases. It should be noted, however, that it still competes for CPU time with the application threads. By default, the number of threads used by this GC algorithm equals to ¼ of the number of physical cores of your machine.

这种收集器的设计目标是为了避免在收集老年代时出现长时间的停顿。通过两种手段达成这个目标。第一,它不对老年代进行整理, 而是使用空闲列表(free-lists)来管理回收空间。第二,它在 **标记-清除**(mark-and-sweep) 阶段与程序线程并发执行,完成大部分工作。也就是说垃圾收集的这些阶段中没有显式地停止应用程序线程。然而,值得注意的是,它仍然与应用程序线程争抢CPU时间。默认情况下,这种GC算法使用的线程数量等于(1/4)的CPU内核数。



This garbage collector can be chosen by specifying the following option on your command line:

可以在命令行中通过以下选项指定这款垃圾收集器:


	java -XX:+UseConcMarkSweepGC com.mypackages.MyExecutableClass




This combination is a good choice on multi-core machines if your primary target is latency. Decreasing the duration of an individual GC pause directly affects the way your application is perceived by end-users, giving them a feel of a more responsive application. As most of the time at least some CPU resources are consumed by the GC and not executing your application’s code, CMS generally often worse throughput than Parallel GC in CPU-bound applications.

如果主要目标是降低延迟,那么在多核机器上使用这种组合是很好的选择. 减少每次GC暂停时间能直接影响终端用户对系统的感知, 给他们应用程序更加敏感的感觉. 因为多数时候至少有一部分CPU资源被GC消耗, 而没用来执行程序的代码, 所以在CPU资源受限的情况下,CMS通常比并行GC的吞吐量要差一些。


As with previous GC algorithms, let us now see how this algorithm is applied in practice by taking a look at the GC logs that once again expose one minor and one major GC pause:

与和前面一样, 我们先通过GC日志来看看这种算法在实践中的应用, 也是一个小型GC,和一个大型GC暂停:


	2015-05-26T16:23:07.219-0200: 64.322: [GC (Allocation Failure) 64.322: 
				[ParNew: 613404K->68068K(613440K), 0.1020465 secs]
				10885349K->10880154K(12514816K), 0.1021309 secs]
			[Times: user=0.78 sys=0.01, real=0.11 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) 
				[1 CMS-initial-mark: 10812086K(11901376K)] 
				10887844K(12514816K), 0.0001997 secs] 
			[Times: user=0.00 sys=0.00, real=0.00 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] 
			[Times: user=0.07 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] 
			[Times: user=0.02 sys=0.00, real=0.02 secs]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
	2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] 
			[Times: user=0.20 sys=0.00, real=1.07 secs]
	2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) 
				[YG occupancy: 387920 K (613440 K)]
				65.550: [Rescan (parallel) , 0.0085125 secs]
				65.559: [weak refs processing, 0.0000243 secs]
				65.559: [class unloading, 0.0013120 secs]
				65.560: [scrub symbol table, 0.0008345 secs]
				65.561: [scrub string table, 0.0001759 secs]
				[1 CMS-remark: 10812086K(11901376K)] 
				11200006K(12514816K), 0.0110730 secs] 
			[Times: user=0.06 sys=0.00, real=0.01 secs]
	2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
	2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] 
			[Times: user=0.03 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
	2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] 
			[Times: user=0.01 sys=0.00, real=0.01 secs]




### 小型GC(Minor GC)


First of the GC events in log denotes a minor GC cleaning the Young space. Let’s analyze how this collector combination behaves in this regard:

日志中的第一次GC事件代表清理年轻代空间的小型GC(Minor GC)。让我们来分析这款垃圾收集器组合在这方面的表现:


> <a>`2015-05-26T16:23:07.219-0200`</a>: <a>`64.322`</a>:[<a>`GC`</a>(<a>`Allocation Failure`</a>) 64.322: <br/>
> [<a>`ParNew`</a>: <a>`613404K->68068K`</a><a>`(613440K) `</a>, <a>`0.1020465 secs`</a>]<br/>
> <a>`10885349K->10880154K `</a><a>`(12514816K)`</a>, <a>`0.1021309 secs`</a>]<br/>
> <a>`[Times: user=0.78 sys=0.01, real=0.11 secs]`</a>


>
> 1. <a>`2015-05-26T16:23:07.219-0200`</a> – Time when the GC event started. GC事件开始的时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`。
> 1. <a>`64.322`</a> – Time when the GC event started, relative to the JVM startup time. Measured in seconds. 相对于JVM启动时间,GC事件开始的时间,单位是秒。
> 1. <a>`GC`</a> – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC. 用来区分 Minor GC 还是 Full GC 的标志。`GC`表明这是一次**小型GC**(Minor GC)
> 1. <a>`Allocation Failure`</a> – Cause of the collection. In this case, the GC is triggered due to a requested allocation not fitting into any region in Young Generation. 触发垃圾收集的原因。本次GC事件, 是由于年轻代中没有适当的空间存放新的数据结构引起的。
> 1. <a>`ParNew`</a> – Name of the collector used, this time it indicates a parallel mark-copy stop-the-world garbage collector used in the Young Generation, designed to work in conjunction with Concurrent Mark & Sweep garbage collector in the Old Generation. 使用的垃圾收集器的名称。这个名字表示的是在年轻代中使用的: 并行的 标记-复制(mark-copy), 全线暂停(STW) 垃圾收集器, 专门设计来配合老年代使用的 Concurrent Mark & Sweep 垃圾收集器。
> 1. <a>`613404K->68068K`</a> – Usage of the Young Generation before and after collection. 在垃圾收集之前和之后的年轻代使用量。
> 1. <a>`(613440K) `</a> – Total size of the Young Generation. 年轻代的总大小。
> 1. <a>`0.1020465 secs`</a> – Duration for the collection w/o final cleanup. 垃圾收集器在 w/o final cleanup 阶段消耗的时间
> 1. <a>`10885349K->10880154K `</a> – Total used heap before and after collection. 在垃圾收集之前和之后堆内存的使用情况。
> 1. <a>`(12514816K)`</a> – Total available heap. 可用堆的总大小。
> 1. <a>`0.1021309 secs`</a> – The time it took for the garbage collector to mark and copy live objects in the Young Generation. This includes communication overhead with ConcurrentMarkSweep collector, promotion of objects that are old enough to the Old Generation and some final cleanup at the end of the garbage collection cycle. 垃圾收集器在标记和复制年轻代存活对象消耗的时间。这包括和ConcurrentMarkSweep收集器的通信开销, 提升存活够长时间的对象到老年代,以及在垃圾收集周期后期的一些最后清理。
> 1. <a>`[Times: user=0.78 sys=0.01, real=0.11 secs]`</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by the Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount. 应用程序被停止的系统时钟时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有固定比例的处理过程是不能并行化的。



From the above we can thus see that before the collection the total used heap was 10,885,349K and the used Young Generation share was 613,404K. This means that the Old Generation share was 10,271,945K. After the collection, Young Generation usage decreased by 545,336K but total heap usage decreased only by 5,195K. This means that 540,141K was promoted from the Young Generation to Old.

从上面我们可以看到,在收集前堆内存总的使用量是 **10,885,349K**, 年轻代使用的为 **613,404K**。这意味着老年代使用量是 **10,271,945K**。收集之后,年轻代的使用量减少了 545,336K, 而总的堆内存使用只下降了 **5,195K**。可以算出有 **540,141K** 的对象从年轻代提升到老年代。


![](04_05_ParallelGC-in-Young-Generation-Java.png)




### Full GC

### 完全GC


Now, just as you are becoming accustomed to reading GC logs already, this chapter will introduce a completely different format for the next garbage collection event in the logs. The lengthy output that follows consists of all the different phases of the mostly concurrent garbage collection in the Old Generation. We will review them one by one but in this case we will cover the log content in phases instead of the entire event log at once for more concise representation. But to recap, the whole event for the CMS collector looks like the following:



现在, 我们已经对解读GC日志越来越熟悉, 本节接下来介绍一种日志格式完全不同的GC事件。下面这一段很长的日志,就是CMS在老年代中进行垃圾收集时各个阶段的输出。为了简洁,我们将根据这些阶段逐个介绍。 首先来看看全貌, CMS收集器的整个GC事件日志如下:




	2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) 
			[1 CMS-initial-mark: 10812086K(11901376K)] 
		10887844K(12514816K), 0.0001997 secs] 
		[Times: user=0.00 sys=0.00, real=0.00 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] 
		[Times: user=0.07 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] 
		[Times: user=0.02 sys=0.00, real=0.02 secs]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
	2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] 
		[Times: user=0.20 sys=0.00, real=1.07 secs]
	2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) 
			[YG occupancy: 387920 K (613440 K)]
			65.550: [Rescan (parallel) , 0.0085125 secs]
			65.559: [weak refs processing, 0.0000243 secs]
			65.559: [class unloading, 0.0013120 secs]
			65.560: [scrub symbol table, 0.0008345 secs]
			65.561: [scrub string table, 0.0001759 secs]
			[1 CMS-remark: 10812086K(11901376K)] 
		11200006K(12514816K), 0.0110730 secs] 
		[Times: user=0.06 sys=0.00, real=0.01 secs]
	2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
	2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] 
		[Times: user=0.03 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
	2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] 
		[Times: user=0.01 sys=0.00, real=0.01 secs]




Just to bear in mind – in real world situation Minor Garbage Collections of the Young Generation can occur anytime during concurrent collecting the Old Generation. In such case the major collection records seen below will be interleaved with the Minor GC events covered in previous chapter.

只是要记住 —— 在实际情况下, 在老年代进行并发收集时, 可能随时会有多次年轻代的小型GC发生. 在这种情况下, 大型GC的日志中就会掺杂着多次小型GC事件, 像前面一样。


**Phase 1: Initial Mark**. This is one of the two stop-the-world events during CMS. The goal of this phase is to mark all the objects in the Old Generation that are either direct GC roots or are referenced from some live object in the Young Generation. The latter is important since the Old Generation is collected separately.

**阶段 1: 初始标记(Initial Mark)**. 这是CMS中两次STW事件中的一次. 此阶段的目标是标记老年代中的所有存活对象, 或者直接是GC ROOR 引用, 或者是由年轻代中的存活对象引用. 后者也非常重要, 因为老年代是分开进行回收的。


![](04_06_g1-06.png)




><a>`2015-05-26T16:23:07.321-0200: 64.42`<sup>1</sup></a>: [GC (<a>`CMS Initial Mark`<sup>1</sup></a><br/>
>[1 CMS-initial-mark: <a>`10812086K`<sup>1</sup></a><a>`(11901376K)`<sup>1</sup></a>] <a>`10887844K`<sup>1</sup></a><a>`(12514816K)`<sup>1</sup></a>,<br/>
> <a>`0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]`<sup>1</sup></a>




>
> 1. <a>`2015-05-26T16:23:07.321-0200: 64.42`</a> – Time the GC event started, both clock time and relative to the time from the JVM start. For the following phases the same notion is used throughout the event and is thus skipped for brevity. GC事件开始的时间. 其中-0200是时区,而中国所在的东8区为 +0800。 而 **64.42** 是相对于JVM启动的时间。 下面的其他阶段也是一样,所以就不再重复介绍。
> 1. <a>`CMS Initial Mark`</a> – Phase of the collection – “Initial Mark” in this occasion – that is collecting all GC Roots. 垃圾回收的阶段名称为 “Initial Mark”。 标记所有的 GC Root。
> 1. <a>`10812086K`</a> – Currently used Old Generation. 老年代的当前使用量。
> 1. <a>`(11901376K)`</a> – Total available memory in the Old Generation. 老年代中总的可用内存。
> 1. <a>`10887844K`</a> – Currently used heap. 当前堆内存的使用量
> 1. <a>`(12514816K)`</a> – Total available heap. 可用堆的总大小。
> 1. <a>`0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]`</a> – Duration of the phase, measured also in user, system and real time. 此暂停的持续时间, 以 user, system 和 real time 3个部分进行衡量。





**Phase 2: Concurrent Mark.** During this phase the Garbage Collector traverses the Old Generation and marks all live objects, starting from the roots found in the previous phase of “Initial Mark”. The “Concurrent Mark” phase, as its name suggests, runs concurrently with your application and does not stop the application threads. Note that not all the live objects in the Old Generation may be marked, since the application is mutating references during the marking.


**第二阶段:并发标记(Concurrent Mark).** 在此阶段, 垃圾收集器遍历老年代,从前一阶段 “Initial Mark” 找到的 root 根开始, 标记所有存活的对象。 “并发标记”阶段, 顾名思义, 与应用程序同时运行,不会暂停程序线程. 请注意,此阶段并非所有老年代中存活的对象都会被标记,因为在标记过程中引用还会发生变化。


![](04_07_g1-07.png)




In the illustration, a reference pointing away from the “Current object” was removed concurrently with the marking thread.

在上面的插图中, 一个“当前对象”指向的引用被标记线程并发移除。


>2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]<br/>
>2015-05-26T16:23:07.357-0200: 64.460: [<a>`CMS-concurrent-mark`<sup>1</sup></a>: <a>`035/0.035 secs`<sup>1</sup></a>]<br/>
><a>`[Times: user=0.07 sys=0.00, real=0.03 secs]`<sup>1</sup></a><br/>

>
> 1. <a>`CMS-concurrent-mark`</a> – Phase of the collection – “Concurrent Mark” in this occasion – that is traversing the Old Generation and marking all live objects. 垃圾收集的一个阶段，这里是 "Concurrent Mark", 并发标记,遍历老年代并标记所有存活的对象
> 1. <a>`035/0.035 secs`</a> – Duration of the phase, showing elapsed time and wall clock time correspondingly. 此阶段的持续时间, 分别是运行时间和对应的实际时间
> 1. <a>`[Times: user=0.07 sys=0.00, real=0.03 secs]`</a> – “Times” section is less meaningful for concurrent phases as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking. <br/>Times 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行




**Phase 3: Concurrent Preclean.** This is again a concurrent phase, running in parallel with the application threads, not stopping them. While the previous phase was running concurrently with the application, some references were changed. Whenever that happens, the JVM marks the area of the heap (called “Card”) that contains the mutated object as “dirty” (this is known as Card Marking).

**阶段3: 并发预清理(Concurrent Preclean).** 此阶段也是并发进行的, 与程序线程并行执行, 而不停止线程. 因为前一阶段与程序同时进行的,可能有一些引用已经改变。每当发生这种情况,JVM将堆中的发生了对象改变的区域(称为“Card”)标记为“脏”区域(这就是所谓的卡片标记,Card Marking)。


![](04_08_g1-08.png)




In the pre-cleaning phase, these dirty objects are accounted for, and the objects reachable from them are also marked. The cards are cleaned when this is done.

在预清理阶段,统计这些脏对象,从他们可达的对象也标记下来。当(此阶段)完成时 card 也被清理了。


![](04_09_g1-09.png)




Additionally, some necessary housekeeping and preparations for the Final Remark phase are performed.

此外, 也会执行一些必要的细节处理, 以及为 Final Remark 阶段做一些前期准备工作。


>
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
>
2015-05-26T16:23:07.373-0200: 64.476: [<a>`CMS-concurrent-preclean`</a>: <a>`0.016/0.016 secs`</a>] <a>`[Times: user=0.02 sys=0.00, real=0.02 secs]`</a>



>
> 1. <a>`CMS-concurrent-preclean`</a> – Phase of the collection – “Concurrent Preclean” in this occasion – accounting for references being changed during previous marking phase. 并发预清理阶段, 统计在前面的标记阶段中发生了改变的对象。
> 1. <a>`0.016/0.016 secs`</a> – Duration of the phase, showing elapsed time and wall clock time correspondingly. 此阶段的持续时间, 分别是运行时间和对应的实际时间
> 1. <a>`[Times: user=0.02 sys=0.00, real=0.02 secs]`</a> – The “Times” section is less meaningful for concurrent phases as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking. Times 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行



**Phase 4: Concurrent Abortable Preclean.** Again, a concurrent phase that is not stopping the application’s threads. This one attempts to take as much work off the shoulders of the stop-the-world Final Remark as possible. The exact duration of this phase depends on a number of factors, since it iterates doing the same thing until one of the abortion conditions (such as the number of iterations, amount of useful work done, elapsed wall clock time, etc) is met.

**Phase 4: 并发可取消预清理(Concurrent Abortable Preclean).** 这也是不停止应用程序线程的一个并发阶段. 此阶段尝试在 STW 的 Final Remark 之前尽可能多地做一些工作。这一阶段的具体时间取决于很多因素, 因为它迭代做同样的事情,直到满足某个退出条件( 如迭代次数, 有用工作量,经过的时钟时间,等等)。


>
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
>
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean1: 0.167/1.074 secs2] [Times: user=0.20 sys=0.00, real=1.07 secs]3



>
> 1. <a>`CMS-concurrent-abortable-preclean`</a> – Phase of the collection “Concurrent Abortable Preclean” in this occasion。此阶段的名称: “Concurrent Abortable Preclean”
> 1. <a>`0.167/1.074 secs`</a> – Duration of the phase, showing elapsed and wall clock time respectively. It is interesting to note that the user time reported is a lot smaller than clock time. Usually we have seen that real time is less than user time, meaning that some work was done in parallel and so elapsed clock time is less than used CPU time. Here we have a little amount of work – for 0.167 seconds of CPU time, and garbage collector threads were doing a lot of waiting. Essentially, they were trying to stave off for as long as possible before having to do an STW pause. By default, this phase may last for up to 5 seconds. 此阶段的持续时间, 运行时间和对应的实际时间。有趣的是, 用户时间明显比时钟时间要小很多。通常情况下我们看到的都是时钟时间小于用户时间, 这意味着因为有一些并行工作, 所以运行时间小于使用的CPU时间。这里只进行了少量的工作 — 0.167秒的CPU时间,GC线程进行了很多系统等待。从本质上讲,他们试图在必须做一个STW暂停之前等待尽可能长的时间。默认情况下,这个阶段可以持续5秒钟。

> 1. <a>`[Times: user=0.20 sys=0.00, real=1.07 secs]`</a> – The “Times” section is less meaningful for concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking. Times 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行



This phase may significantly impact the duration of the upcoming stop-the-world pause, and has quite a lot of non-trivial configuration options and fail modes.

此阶段可能显著影响STW停顿的持续时间, 并且有许多重要的配置选项和失败模式。


**Phase 5: Final Remark.** This is the second and last stop-the-world phase during the event. The goal of this stop-the-world phase is to finalize marking all live objects in the Old Generation. Since the previous preclean phases were concurrent, they may have been unable to keep up with the application’s mutating speeds. A stop-the-world pause is required to finish the ordeal.


**阶段5: 最终重标记.** 这是本次GC事件中第二次也是最后一次STW阶段。STW阶段的目标是完成老年代中所有存活对象的标记. 因为之前的 preclean 阶段是并发的, 有可能无法跟上应用程序的变异速度。所以需要 STW暂停来解决这种磨难。


Usually CMS tries to run final remark phase when Young Generation is as empty as possible in order to eliminate the possibility of several stop-the-world phases happening back-to-back.

通常CMS会尝试在年轻代尽可能空的情况运行 final remark 阶段, 以避免连续多次 STW 阶段紧挨着发生。


This event looks a bit more complex than previous phases:

此事件看起来比之前的阶段要稍微复杂一些:


> <a>`2015-05-26T16:23:08.447-0200: 65.550`</a>: [GC (<a>`CMS Final Remark`</a>) [<a>`YG occupancy: 387920 K (613440 K)`</a>]
> 65.550: <a>`[Rescan (parallel) , 0.0085125 secs]`</a> 65.559: [<a>`weak refs processing, 0.0000243 secs]65.559`</a>:  [<a>`class unloading, 0.0013120 secs]65.560`</a>: [<a>`scrub string table, 0.0001759 secs`</a>]
> [1 CMS-remark: <a>`10812086K(11901376K)`</a>] <a>`11200006K(12514816K) `</a>,<a>`0.0110730 secs`</a>] <a>`[Times: user=0.06 sys=0.00, real=0.01 secs]`

<br/>

>
> 1. <a>`2015-05-26T16:23:08.447-0200: 65.550`</a> – Time the GC event started, both clock time and relative to the time from the JVM start. GC事件开始的时间, 包括时钟时间,以及相对于JVM启动时间. 其中`-0200`是时区,而中国所在的东8区为 `+0800`。
> 1. <a>`CMS Final Remark`</a> – Phase of the collection – “Final Remark” in this occasion – that is marking all live objects in the Old Generation, including the references that were created/modified during previous concurrent marking phases. 此阶段的名称, “Final Remark”, 标记老年代中的所有存活对象，包括在此前的并发标记阶段创建/修改的引用。
> 1. <a>`YG occupancy: 387920 K (613440 K)`</a> – Current occupancy and capacity of the Young Generation. 当前年轻代的使用量和容量
> 1. <a>`[Rescan (parallel) , 0.0085125 secs]`</a> – The “Rescan” completes the marking of live objects while the application is stopped. In this case the rescan was done in parallel and took 0.0085125 seconds. 重新扫描(Rescan)在程序暂停时完成存活对象的标记。此次 rescan 为并行执行,消耗的时间为  **0.0085125秒**。
> 1. <a>`weak refs processing, 0.0000243 secs]65.559`</a> – First of the sub-phases that is processing weak references along with the duration and timestamp of the phase. 第一个子阶段(sub-phases), 处理弱引用, 以及持续时间和开始的时间戳。
> 1. <a>`class unloading, 0.0013120 secs]65.560`</a> – Next sub-phase that is unloading the unused classes, with the duration and timestamp of the phase. 第二个子阶段, 卸载未使用的类,以及持续时间和开始的时间戳。
> 1. <a>`scrub string table, 0.0001759 secs`</a> – Final sub-phase that is cleaning up symbol and string tables which hold class-level metadata and internalized string respectively. Clock time of the pause is also included. 最后一个子阶段, 清理持有class级别metadata的符号表(symbol tables),以及内部化字符串的 string tables。包括暂停的时钟时间。
> 1. <a>`10812086K(11901376K)`</a> – Occupancy and the capacity of the Old Generation after the phase. 此阶段完成后老年代的使用量和容量
> 1. <a>`11200006K(12514816K) `</a> – Usage and the capacity of the total heap after the phase. 此阶段完成后整个堆内存的使用量和容量
> 1. <a>`0.0110730 secs`</a> – Duration of the phase. 此阶段的持续时间。
> 1. <a>`[Times: user=0.06 sys=0.00, real=0.01 secs]`</a> – Duration of the pause, measured in user, system and real time categories. GC事件的持续时间, 通过不同的类别来衡量: user, system and real time。



After the five marking phases, all live objects in the Old Generation are marked and now garbage collector is going to reclaim all unused objects by sweeping the Old Generation:

在5个标记阶段之后, 老年代中的所有存活对象都被标记了, 现在垃圾收集器将通过清除老年代来回收所有不使用的对象:


**Phase 6: Concurrent Sweep.** Performed concurrently with the application, without the need for the stop-the-world pauses. The purpose of the phase is to remove unused objects and to reclaim the space occupied by them for future use.

**阶段6:并发清除(Concurrent Sweep)。** 与应用程序并发执行,而不需要STW停顿。阶段的目的是删除未使用的对象并收回他们占据的空间,以供将来使用。



![](04_10_g1-10.png)




>
2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start] 2015-05-26T16:23:08.485-0200: 65.588: [<a>`CMS-concurrent-sweep`</a>: <a>`0.027/0.027 secs`</a>] <a>`[Times: user=0.03 sys=0.00, real=0.03 secs] `</a>



>
> 1. <a>`CMS-concurrent-sweep`</a> – Phase of the collection “Concurrent Sweep” in this occasion, sweeping unmarked and thus unused objects to reclaim space. 此阶段的名称, “Concurrent Sweep”, 清除未标记、不使用的对象以释放内存空间。
> 1. <a>`0.027/0.027 secs`</a> – Duration of the phase, showing elapsed time and wall clock time correspondingly. 此阶段的持续时间, 分别是运行时间和对应的实际时间
> 1. <a>`[Times: user=0.03 sys=0.00, real=0.03 secs] `</a> – “Times” section is less meaningful on concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking. 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行



**Phase 7: Concurrent Reset.** Concurrently executed phase, resetting inner data structures of the CMS algorithm and preparing them for the next cycle.

**阶段7:并发重置(Concurrent Reset)。** 并发执行阶段,重置CMS算法的内部数据结构, 为下一次GC循环做准备。


>
2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start] 2015-05-26T16:23:08.497-0200: 65.601: [<a>`CMS-concurrent-reset`</a>: <a>`0.012/0.012 secs`</a>] <a>`[Times: user=0.01 sys=0.00, real=0.01 secs]`</a>



>
> 1. <a>`CMS-concurrent-reset`</a> – The phase of the collection – “Concurrent Reset” in this occasion – that is resetting inner data structures of the CMS algorithm and preparing for the next collection.此阶段的名称, “Concurrent Reset”, 重置CMS算法的内部数据结构, 为下一次GC循环做准备。
> 1. <a>`0.012/0.012 secs`</a> – Duration of the the phase, measuring elapsed and wall clock time respectively. 此阶段的持续时间, 分别是运行时间和对应的实际时间
> 1. <a>`[Times: user=0.01 sys=0.00, real=0.01 secs]`</a> – The “Times” section is less meaningful on concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking. 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行



All in all, the CMS garbage collector does a great job at reducing the pause durations by offloading a great deal of the work to concurrent threads that do not require the application to stop. However, it, too, has some drawbacks, the most notable of them being the Old Generation fragmentation and the lack of predictability in pause durations in some cases, especially on large heaps.

总之, CMS垃圾收集器在减少停顿时间上做了很多给力的工作, 通过大量的并发线程执行不需要暂停应用程序的工作. 然而,它也有一些缺点,其中最著名的是老年代的碎片化, 以及在某些情况下缺乏可预测的暂停时间, 特别是大型堆。


## G1 – Garbage First(垃圾优先)


One of the key design goals of G1 was to make the duration and distribution of stop-the-world pauses due to garbage collection predictable and configurable. In fact, Garbage-First is a soft real-time garbage collector, meaning that you can set specific performance goals to it. You can request the stop-the-world pauses to be no longer than x milliseconds within any given y-millisecond long time range, e.g. no more than 5 milliseconds in any given second. Garbage-First GC will do its best to meet this goal with high probability (but not with certainty, that would be hard real-time).

G1的主要设计目标就是使的STW停顿的时间和分布变成可预测和可配置的。事实上, G1是一款软实时的垃圾收集器, 这意味着您可以设置特定的性能目标.你可以要求，在给定的任意y 毫秒时间范围内, STW停顿不得超过x毫秒。 例如: 每一秒中不得超过5毫秒. Garbage-First GC将以很大的概率尽力满足这个目标(但也不是完全确定,具体是多少将是真实时[hard real-time])。


To achieve this, G1 builds upon a number of insights. First, the heap does not have to be split into contiguous Young and Old generation. Instead, the heap is split into a number (typically about 2048) smaller heap regions that can house objects. Each region may be an Eden region, a Survivor region or an Old region. The logical union of all Eden and Survivor regions is the Young Generation, and all the Old regions put together is the Old Generation:

为了达成这个目标,G1的有一些独特的实现。首先, 堆不需要分成连续的年轻代和老年代空间。而是分成许多个(通常是2048个)可以存放对象的小型堆区域(heap regions)。每个区域都可以是一个Eden区, Survivor区或者Old区. 逻辑上, 所有的Eden区和Survivor区就是年轻代, 所有的Old区放在一起那就是老年代:


![](04_11_g1-011.png)




This allows the GC to avoid collecting the entire heap at once, and instead approach the problem incrementally: only a subset of the regions, called the collection set will be considered at a time. All the Young regions are collected during each pause, but some Old regions may be included as well:

这样划分使得每次GC不必去收集整个堆,而是以增量的方式解决问题: 每次只处理一部分区域,称为回收集(collection set). 每次暂停会收集所有的年轻区域, 但可能只包含一部分老年代区域:


![](04_12_g1-02.png)




Another novelty of G1 is that during the concurrent phase it estimates the amount of live data that each region contains. This is used in building the collection set: the regions that contain the most garbage are collected first. Hence the name: garbage-first collection.

G1的另一个创新, 是在并发阶段估算每个区域存活对象的数量。这用于构建回收集合(collection set): 垃圾最多的区域会被优先收集。这也是G1名称的由来: garbage-first。


To run the JVM with the G1 collector enabled, run your application as

为JVM启用G1收集器, 使用的命令行参数为:


	java -XX:+UseG1GC com.mypackages.MyExecutableClass



### Evacuation Pause: Fully Young

### 转移暂停(Evacuation Pause): Fully Young 模式


In the beginning of the application’s lifecycle, G1 does not have any additional information from the not-yet-executed concurrent phases, so it initially functions in the fully-young mode. When the Young Generation fills up, the application threads are stopped, and the live data inside the Young regions is copied to Survivor regions, or any free regions that thereby become Survivor.

在应用程序刚启动时, G1还未执行过(not-yet-executed)并发阶段得到其他信息,  所以此时处于 fully-young 模式. 在年轻代用满后, 应用程序线程暂停, 年轻代中的存活数据被复制到存活区, 如果还没有存活区,则任意选择一些空闲的区域作为存活区。


The process of copying these is called Evacuation, and it works in pretty much the same way as the other Young collectors we have seen before. The full logs of evacuation pauses are rather large, so, for simplicity’s sake we will leave out a couple of small bits that are irrelevant in the first fully-young evacuation pause. We will get back to them after the concurrent phases are explained in greater detail. In addition, due to the sheer size of the log record, the parallel phase details and “Other” phase details are extracted to separate sections:

复制的过程叫做转移(Evacuation), 它和前面讲过的年轻代收集器几乎是一样的工作方式。转移暂停的日志信息很长,为简单起见, 我们去除了一部分不重要的日志信息. 我们将在并发阶段之后进行更详细的解释。此外, 由于日志记录很多, 所以并行阶段和“其他”阶段的细节将拆分为多个部分进行讲解:


> <a>`0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]`</a> <br/>
>     <a>`[Parallel Time: 13.9 ms, GC Workers: 8]`</a><br/>
>      <a>`…`</a><br/>
>      <a>`[Code Root Fixup: 0.0 ms]`</a><br/>
>      <a>`[Code Root Purge: 0.0 ms]`</a><br/>
>          [Clear CT: 0.1 ms] <br/>
>          <a>`[Other: 0.4 ms]`</a>
>      <a>`…`</a><br/>
>       <a>`[Eden: 24.0M(24.0M)->0.0B(13.0M) `</a> <a>`Survivors: 0.0B->3072.0K `</a>  <a>`Heap: 24.0M(256.0M)->21.9M(256.0M)]`</a>
>        <a>`[Times: user=0.04 sys=0.04, real=0.02 secs] `</a>



>
> 1. <a>`0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]`</a> – G1 pause cleaning only (young) regions. The pause started 134ms after the JVM startup and the duration of the pause was 0.0144 seconds measured in wall clock time. G1暂停,只清理(年轻代)区域。暂停在JVM启动134 ms 后开始, 持续的系统时间是 **0.0144秒** 。
> 1. <a>`[Parallel Time: 13.9 ms, GC Workers: 8]`</a> – Indicating that for 13.9 ms (real time) the following activities were carried out by 8 threads in parallel  表明下列活动由8线程并行执行，消耗时间为13.9毫秒(real time)
> 1. <a>`…`</a> – Cut for brevity, see the following section below for the details. 为阅读方便单而省略,请参考后文。
> 1. <a>`[Code Root Fixup: 0.0 ms]`</a> – Freeing up the data structures used for managing the parallel activities. Should always be near-zero. This is done sequentially. 释放用于管理并行活动的数据结构。一般都是接近于零。这是串行执行的。
> 1. <a>`[Code Root Purge: 0.0 ms]`</a> – Cleaning up more data structures, should also be very fast, but non necessarily almost zero. This is done sequentially. 清理更多的数据结构, 也是非常快的, 但如非必要则几乎等于零。这是串行执行的。
> 1. <a>`[Other: 0.4 ms]`</a> – Miscellaneous other activities, many of which are also parallelized.  其他活动消耗的时间, 其中也有很多是并行执行的。
> 1. <a>`…`</a> – See the section below for details. 请参考后文
> 1. <a>`[Eden: 24.0M(24.0M)->0.0B(13.0M) `</a> – Eden usage and capacity before and after the pause. 事件前后新生代的使用量
> 1. <a>`Survivors: 0.0B->3072.0K `</a> – Space used by Survivor regions before and after the pause. 事件前后存活区的使用量
> 1. <a>`Heap: 24.0M(256.0M)->21.9M(256.0M)]`</a> – Total heap usage and capacity before and after the pause. 事件前后整个堆内存的使用量与总容量。
> 1. <a>`[Times: user=0.04 sys=0.04, real=0.02 secs] `</a> – Duration of the GC event, measured in different categories: GC事件的持续时间, 通过不同的类别来衡量:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection. 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间
 - sys – Time spent in OS calls or waiting for system event. 花在操作系统调用和等待系统事件的时间
 - real – Clock time for which your application was stopped. With the parallelizable activities during GC this number is ideally close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount. 应用程序被停止的系统时钟时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有固定比例的处理过程是不能并行化的。



> 说明: 系统时间(wall clock time, elapsed time), 是指一段程序从运行到终止，系统时钟走过的时间。一般来说，系统时间都是要大于CPU时间



Most of the heavy-lifting is done by multiple dedicated GC worker threads. Their activities are described in the following section of the log:

最繁重的任务由多个专用的GC worker线程来执行。下面的日志描述了他们的活动:


> <a>`[Parallel Time: 13.9 ms, GC Workers: 8]`</a> <br/>
> <a>`[GC Worker Start (ms)`</a>:  Min: 134.0, Avg: 134.1, Max: 134.1, Diff: 0.1] <br/>
> <a>`[Ext Root Scanning (ms)`</a>: Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.2]<br/>	    [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] <br/>
> 	    [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0] <br/>
> 	    [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] <br/>
>   <a>`[Code Root Scanning (ms)`</a>: Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2] <br/>
>  <a>`[Object Copy (ms)`</a>: Min: 10.8, Avg: 12.1, Max: 12.6, Diff: 1.9, Sum: 96.5]<br/>
>  <a>`[Termination (ms)`</a>: Min: 0.8, Avg: 1.5, Max: 2.8, Diff: 1.9, Sum: 12.2] <br/>
>  <a>`[Termination Attempts`</a>: Min: 173, Avg: 293.2, Max: 362, Diff: 189, Sum: 2346] <br/>
>  <a>`[GC Worker Other (ms)`</a>: Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1] <br/>
>  <a>`GC Worker Total (ms)`</a>: Min: 13.7, Avg: 13.8, Max: 13.8, Diff: 0.1, Sum: 110.2] <br/>
>  <a>`[GC Worker End (ms)`</a>: Min: 147.8, Avg: 147.8, Max: 147.8, Diff: 0.0] <br/>

<br/>

>
> 1. <a>`[Parallel Time: 13.9 ms, GC Workers: 8]`</a> – Indicating that for 13.9 ms (clock time) the following activities were carried out by 8 threads in parallel。 表明下列活动由8线程并行执行，消耗时间为13.9毫秒(real time)
> 1. <a>`[GC Worker Start (ms)`</a> – The moment in time at which the workers started their activity, matching the timestamp at the beginning of the pause. If Min and Max differ a lot, then it may be an indication that too many threads are used or other processes on the machine are stealing CPU time from the garbage collection process inside the JVM. workers 开始启动的的时间, 对应于pause开始的时间戳。如果 Min 和 Max 差别很大,则可能是机器上的其他进程占用线程数量过多的标识。 使用或窃取其他进程的机器上的CPU时间在JVM的垃圾收集过程
> 1. <a>`[Ext Root Scanning (ms)`</a> – How long it took to scan the external (non-heap) roots such as classloaders, JNI references, JVM system roots, etc. Shows elapsed time, “Sum” is CPU time. 用了多长时间来扫描外部(non-heap) root, 例如 classloaders, JNI引用,JVM系统 troot等。显示了运行时间, “Sum” 是指CPU时间。
> 1. <a>`[Code Root Scanning (ms)`</a> – How long it took to scan the roots that came from the actual code: local vars, etc.  用了多长时间来扫描的实际代码中的 root: 例如局部变量等等(local vars)。
> 1. <a>`[Object Copy (ms)`</a> – How long it took to copy the live objects away from the collected regions. 花费了多少时间来复制收集区内的存活对象。
> 1. <a>`[Termination (ms)`</a> – How long it took for the worker threads to ensure that they can safely stop and that there’s no more work to be done, and then actually terminate. worker 线程用了多长时间来确保可以安全地暂停其他线程, 没有更多的工作要做,然后就终止了。
> 1. <a>`[Termination Attempts`</a> – How many attempts worker threads took to try and terminate. An attempt is failed if the worker discovers that there’s in fact more work to be done, and it’s too early to terminate. worker 线程尝试多少次try 和 teminate。如果worker 发现实际上有更多的工作要做,则这一次尝试就是失败的, 它还为时过早, 所以终止了。
> 1. <a>`[GC Worker Other (ms)`</a> – Other miscellaneous small activities that do not deserve a separate section in the logs. 在日志中不配使用一个单独部分的小活动。
> 1. <a>`GC Worker Total (ms)`</a> – How long the worker threads have worked for in total. worker 线程工作的时间总计
> 1. <a>`[GC Worker End (ms)`</a> – The timestamp at which the workers have finished their jobs. Normally they should be roughly equal, otherwise it may be an indication of too many threads hanging around or a noisy neighbor. worker 线程完成作业的时间戳。通常来说应该大致相等, 否则可能是表明有太多的线程挂起和唤醒导致的。



Additionally, there are some miscellaneous activities that are performed during the Evacuation pause. We will only cover a part of them in this section. The rest will be covered later.

此外,有一些杂项疏散期间暂停执行的活动。我们只会讨论的一部分,他们在这一节中。剩下的稍后将覆盖。


> <a>`[Other: 0.4 ms]`</a> <br/>
>    [Choose CSet: 0.0 ms] <br/>
>    <a>`[Ref Proc: 0.2 ms]`</a> <br/>
>    <a>`[Ref Enq: 0.0 ms]`</a> <br/>
>    [Redirty Cards: 0.1 ms] <br/>
>    [Humongous Register: 0.0 ms] <br/>
>    [Humongous Reclaim: 0.0 ms] <br/>
>    <a>`[Free CSet: 0.0 ms]`</a> <br/>


>
> 1. <a>`[Other: 0.4 ms]`</a> – Miscellaneous other activities, many of which are also parallelized. 其他活动消耗的时间, 其中也有很多是并行执行的。
> 1. <a>`[Ref Proc: 0.2 ms]`</a> – The time it took to process non-strong references: clear them or determine that no clearing is needed. 处理非强引用(non-strong)的时间: 清理他们或者决定是否需要清理。
> 1. <a>`[Ref Enq: 0.0 ms]`</a> – The time it took to enqueue the remaining non-strong references to the appropriate ReferenceQueue  用来排队剩下的 non-strong 引用到合适的 ReferenceQueue。
> 1. <a>`[Free CSet: 0.0 ms]`</a> – The time it takes to return the freed regions in the collection set so that they are available for new allocations. 用来返回 CSet 中的释放区域, 以便用于新的分配。



### Concurrent Marking

### 并发标记


The G1 collector builds up on many concepts of CMS from the previous section, so it is a good idea to make sure that you have a sufficient understanding of it before proceeding. Even though it differs in a number of ways, the goals of the Concurrent Marking are very similar.  G1 Concurrent Marking uses the Snapshot-At-The-Beginning approach that marks all the objects that were live at the beginning of the marking cycle, even if they have turned into garbage meanwhile. The information on which objects are live allows to build up the liveness stats for each region so that the collection set could be efficiently chosen afterwards.

G1收集器的很多概念建立在CMS的基础上,所以在开始之前请确保你对CMS有了充分的理解.虽然在很多方面有所不同,但并发标记的目标是非常相似的. G1的并发标记使用 **Snapshot-At-The-Beginning** 的方法在标记开始的时候标记所有的存活对象。即使在标记的同时又有一些变成了垃圾.关于哪些对象是存活的,这种信息允许建立每个区域的存活统计, 以便有效地选择回收集合。


This information is then used to perform garbage collection in the Old regions. It can happen fully concurrently, if the marking determines that a region contains only garbage, or during a stop-the-world evacuation pause for Old regions that contain both garbage and live objects.


然后这些信息会被用来执行老年代区域的垃圾收集。这时候就可以完全并发地执行： 如果标记确定了某个区域只包含垃圾,或者在STW转移暂停期间, 同时包含垃圾和存活对象的老年代区域。



Concurrent Marking starts when the overall occupancy of the heap is large enough. By default, it is 45%, but this can be changed by the InitiatingHeapOccupancyPercent JVM option. Like in CMS, Concurrent Marking in G1 consists of a number of phases, some of them fully concurrent, and some of them requiring the application threads to be stopped.

在整个堆内存的总体占用空间达到一定比例时就会触发并发标记开始。默认值是45%,但可以通过JVM选项 **InitiatingHeapOccupancyPercent** 来调整。和CMS一样,G1中的并发标记由多个阶段组成, 其中一部分是完全并发的, 而有一部分要求暂停应用程序线程。


**Phase 1: Initial Mark.** This phase marks all the objects directly reachable from the GC roots. In CMS, it required a separate stop-the world pause, but in G1 it is typically piggy-backed on an Evacuation Pause, so its overhead is minimal. You can notice this pause in GC logs by the “(initial-mark)” addition in the first line of an Evacuation Pause:

**阶段1: 初始标记(Initial Mark)。** 此阶段标记所有从GC root 直接可达的存活对象。在CMS中, 这个阶段需要一次单独的STW暂停,但在G1中通常是疏散暂停, 所以它的开销是最小的.你可以在 Evacuation Pause 日志的第一行中看到(initial-mark)暂停:


	1.631: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0062656 secs]




**Phase 2: Root Region Scan.** This phase marks all the live objects reachable from the so-called root regions, i.e. the ones that are not empty and that we might end up having to collect in the middle of the marking cycle. Since moving stuff around in the middle of concurrent marking will cause trouble, this phase has to complete before the next evacuation pause starts. If it has to start earlier, it will request an early abort of root region scan, and then wait for it to finish. In the current implementation, the root regions are the survivor regions: they are the bits of Young Generation that will definitely be collected in the next Evacuation Pause.

**阶段2: Root区域扫描。** 此阶段标记从 "根区域" 可达的所有存活对象. 例如，那些非空,在标记过程中我们最终不得不收集的区域。因为在并发标记的过程中迁移对象会造成很多麻烦, 多以此阶段必须在下一次疏散暂停开始之前完成。如果过早启动, 则会请求根区域扫描的早期中止, 然后等待它完成. 在当前版本的实现中, 根区域是存活区: 他们是年轻代的一部分,在未来疏散暂停中肯定会被收集。


	1.362: [GC concurrent-root-region-scan-start]
	1.364: [GC concurrent-root-region-scan-end, 0.0028513 secs]




**Phase 3. Concurrent Mark.** This phase is very much similar to that of CMS: it simply walks the object graph and marks the visited objects in a special bitmap. To ensure that the semantics of snapshot-at-the beginning are met, G1 GC requires that all the concurrent updates to the object graph made by the application threads leave the previous reference known for marking purposes.

**阶段3: 并发标记** 这个阶段非常类似于CMS: 它只是遍历对象图, 并将访问到的对象标记到一个特殊的位图中. 要确保开始时快照的语义, G1 GC要求应用程序线程所有对象图的并发更新, 为了标记的目的, 离开以前的引用。


This is achieved by the use of the Pre-Write barriers (not to be confused with Post-Write barriers discussed later and memory barriers that relate to multithreaded programming). Their function is to, whenever you write to a field while G1 Concurrent Marking is active, store the previous referee in the so-called log buffers, to be processed by the concurrent marking threads.

这是通过使用 `Pre-Write` 屏障(不要和之后介绍的 `Post-Write ` 弄混淆, 也不要和多线程开发中的内存屏障混淆)。他们的功能是, 如果G1正在进行并发标记时,程序写入对象的某个属性, 那么在og buffers 中存储之前的引用。由并发线程进行处理。


	1.364: [GC concurrent-mark-start]
	1.645: [GC co ncurrent-mark-end, 0.2803470 secs]




**Phase 4. Remark.** This is a stop-the-world pause that, like previously seen in CMS, completes the marking process. For G1, it briefly stops the application threads to stop the inflow of the concurrent update logs and processes the little amount of them that is left over, and marks whatever still-unmarked objects that were live when the concurrent marking cycle was initiated. This phase also performs some additional cleaning, e.g. reference processing (see the Evacuation Pause log) or class unloading.

** 阶段4: 再次标记(Remark)。** 这是一个STW停顿, 和CMS相似,完成标记过程。对于G1,它短暂的停止应用程序线程, 暂停流入的并发更新日志, 并处理他们留下的少量信息,并标记所有在并发标记开始时未标记的存活对象。这一阶段也执行一些额外的清理, 请参考(Evacuation Pause log)或 class unloading。


	1.645: [GC remark 1.645: [Finalize Marking, 0.0009461 secs]
	1.646: [GC ref-proc, 0.0000417 secs] 1.646: 
		[Unloading, 0.0011301 secs], 0.0074056 secs]
	[Times: user=0.01 sys=0.00, real=0.01 secs]




**Phase 5. Cleanup.** This final phase prepares the ground for the upcoming evacuation phase, counting all the live objects in the heap regions, and sorting these regions by expected GC efficiency. It also performs all the house-keeping activities required to maintain the internal state for the next iteration of concurrent marking.

** 阶段5: 清理。** 最后的这个小阶段为即将到来的疏散阶段做准备, 计算堆区域中的所有存活对象, 并进行排序, 为了GC的效率. 它也为下一次标记执行所需的所有辅助活动, 维护并发标记的内部状态。


Last but not least, the regions that contain no live objects at all are reclaimed in this phase. Some parts of this phase are concurrent, such as the empty region reclamation and most of the liveness calculation, but it also requires a short stop-the-world pause to finalize the picture while the application threads are not interfering. The logs for such stop-the-world pauses would be similar to:

最后要提醒的是, 所有不包含存活对象的区域在此阶段都被回收了。有一部分是并发的, 例如空区域回收,还有大多数的存活性计算, 但它也需要一个短暂的STW暂停，以完成作业而不影响应用程序线程.这种STW停顿的日志类似这样:


	1.652: [GC cleanup 1213M->1213M(1885M), 0.0030492 secs]
	[Times: user=0.01 sys=0.00, real=0.00 secs]




In case when some heap regions that only contain garbage were discovered, the pause format can look a bit different, similar to:

如果某些堆区域只包含垃圾, 则日志格式可以看起来有点不同,像下面这样:
	
	1.872: [GC cleanup 1357M->173M(1996M), 0.0015664 secs]
	[Times: user=0.01 sys=0.00, real=0.01 secs]
	1.874: [GC concurrent-cleanup-start]
	1.876: [GC concurrent-cleanup-end, 0.0014846 secs]						




### Evacuation Pause: Mixed

# # #疏散暂停:混合


It’s a pleasant case when concurrent cleanup can free up entire regions in Old Generation, but it may not always be the case. After Concurrent Marking has successfully completed, G1 will schedule a mixed collection that will not only get the garbage away from the young regions, but also throw in a bunch of Old regions to the collection set.

能并发清理老年代中整个整个的区域是一种极好的情况, 但现实可能并不总是如此。在并发标记成功完成之后, G1将安排一次混合收集(mixed collection), 不仅清理年轻代区域的垃圾,也将一部分老年代区域放到collection set中。


A mixed Evacuation pause does not always immediately follow the end of the concurrent marking phase. There is a number of rules and heuristics that affect this. For instance, if it was possible to free up a large portion of the Old regions concurrently, then there is no need to do it.

混合模式的转移暂停(Evacuation pause)不一定紧挨着并发标记阶段。有许多规则和数据支撑影响他的启动时机。例如,如果可以在老年代中并发地腾出很大一部分区域,就没有必要启动。


There may, therefore, easily be a number of fully-young evacuation pauses between the end of concurrent marking and a mixed evacuation pause.

因此, 在并发标记,与混合转移暂停之间,很可能有多次的fully-young转移暂停。


The exact number of Old regions to be added to the collection set, and the order in which they are added, is also selected based on a number of rules. These include the soft real-time performance goals specified for the application, the liveness and gc efficiency data collected during concurrent marking, and a number of configurable JVM options. The process of a mixed collection is largely the same as we have already reviewed earlier for fully-young gc, but this time we will also cover the subject of remembered sets.

添加到回收集合的老年代区域的确切数字,以及他们的顺序,也是基于许多规则来处理的. 其中包括程序指定的软实时性能目标, 存活性和在并发标记期间收集的GC效率的数据, 外加一些可配置的JVM选项. 混合收集的过程在很大程度上和前面的 fully-young gc 是一样的,但这一次我们还将介绍 remembered sets 这个主题。


Remembered sets are what allows the independent collection of different heap regions. For instance, when collecting region A,B and C, we have to know whether or not there are references to them from regions D and E to determine their liveness. But traversing the whole heap graph would take quite a while and ruin the whole point of incremental collection, therefore an optimization is employed. Much like we have the Card Table for independently collecting Young regions in other GC algorithms, we have Remembered Sets in G1.

Remembered sets (历史记忆集)是用来支持不同的堆区域独立回收的集合。例如,在收集A、B、C区域时,我们需要知道是否有从区域D和E指向其中的引用, 来确定他们的存活性.但是遍历整个堆需要相当长的一段时间, 这就失去了增量收集的意义, 因此采用一种优化方法. 就像其他GC算法有独立的Card Table 来支持年轻代区域的垃圾收集一样, 在G1中使用的是Remembered Sets。


As shown in the illustration below, each region has a remembered set that lists the references pointing to this region from the outside. These references will then be regarded as additional GC roots. Note that objects in Old regions that were determined to be garbage during concurrent marking will be ignored even if there are outside references to them: the referents are also garbage in that case.

如下面的插图所示,每个区域都有一个**remembered set**, 列出了从外部的指向本区域的引用。这些引用将被视为额外的GC root. 注意,在并发标记过程中,被确定为垃圾的在老年代对象将被忽略, 即使有外部引用指向他们: 因为在这种情况下引用者也是垃圾。


![](04_13_g1-03.png)




What happens next is the same as what other collectors do: multiple parallel GC threads figure out what objects are live and which ones are garbage:

接下来和其他垃圾收集器的行为一模一样: 多个并行的GC线程找出哪些是存活的对象,哪些是垃圾:


![](04_14_g1-04.png)




And, finally, the live objects are moved to survivor regions, creating new if necessary. The now empty regions are freed and can be used for storing objects in again.

最后, 存活对象被移到存活区, 在必要时会创建新的存活区。现在空的区域被释放, 又可用于存放新的对象。


![](04_15_g1-05-v2.png)




To maintain the remembered sets, during the runtime of the application, a Post-Write Barrier is issued whenever a write to a field is performed. If the resulting reference is cross-region, i.e. pointing from one region to another, a corresponding entry will appear in the Remembered Set of the target region. To reduce the overhead that the Write Barrier introduces, the process of putting the cards into the Remembered Set is asynchronous and features quite a number of optimizations. But basically it boils down to the Write Barrier putting the dirty card information into a local buffer, and a specialized GC thread picking it up and propagating the information to the remembered set of the referred region.

为了维护 remembered set, 在应用程序运行的过程中, 只要执行写入一个字段,就会发出一次 Post-Write 屏障。如果生成的引用是跨区域的(cross-region),即从一个区指向另一个区, 就会在目标区域的Remembered Set中,出现一个对应的条目。为了减少Write Barrier造成的开销, 将卡片放入Remembered Set 的过程是异步的, 而且经过相当多的优化.但基本上归结为: Write Barrier 把脏卡信息存放到一个本地缓冲区, 有专门的GC线程来收集处理,并传给被引用的区域。


In the mixed mode, the logs publish certain new interesting aspects when compared to the fully young mode:

在混合模式下的日志, 和完全年轻代模式相比, 可以发现一些新的有趣的地方:


> [<a>`[Update RS (ms)`</a>: Min: 0.7, Avg: 0.8, Max: 0.9, Diff: 0.2, Sum: 6.1] <br/>
> <a>`[Processed Buffers`</a>: Min: 0, Avg: 2.2, Max: 5, Diff: 5, Sum: 18]<br/>
> <a>`[Scan RS (ms)`</a>: Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.8]<br/>
> <a>`[Clear CT: 0.2 ms]`</a> <br/>
> <a>`[Redirty Cards: 0.1 ms]`</a>



> <hr/>
> 1. <a>`[Update RS (ms)`</a> – Since the Remembered Sets are processed concurrently, we have to make sure that the still-buffered cards are processed before the actual collection begins. If this number is high, then the concurrent GC threads are unable to handle the load. It may be, e.g., because of an overwhelming number of incoming field modifications, or insufficient CPU resources. 因为 Remembered Sets 是并发处理的,我们必须确保在实际的垃圾收集之前, 处理缓冲区中的 cards 。如果这个时间很长, 并发GC线程可能无法处理负载。造成的原因可能是。有太多的字段被修改, 或者CPU资源不足。

> 1. <a>`[Processed Buffers`</a> – How many local buffers each worker thread has processed. 缓冲区处理,每个 worker 线程处理了多少个本地缓冲区。
> 1. <a>`[Scan RS (ms)`</a> – How long it took to scan the references coming in from remembered sets. 扫描 RSet, 用了多长时间扫描来自RSet的引用。
> 1. <a>`[Clear CT: 0.2 ms]`</a> – Time to clean the cards in the card table. Cleaning simply removes the “dirty” status that was put there to signify that a field was updated, to be used for Remembered Sets. 清理 card table 中 cards 的时间。清理工作只是简单地删除“脏”状态, 是用来标识一个字段被更新的, 由Remembered Sets使用。
> 1. <a>`[Redirty Cards: 0.1 ms]`</a> – The time it takes to mark the appropriate locations in the card table as dirty. Appropriate locations are defined by the mutations to the heap that GC does itself, e.g. while enqueuing references. 将 card table 中适当的位置标记为 dirty 所花费的时间。适当的位置是由GC本身导致的堆内存改变决定的, 例如引用排队等。



### 总结


This should give one a sufficient basic understanding of how G1 functions. There are, of course, still quite some implementation details that we have left out for brevity, like dealing with humongous objects. All things considered, G1 is the most technologically advanced production-ready collector available in HotSpot. On top of that, it is being relentlessly improved by the HotSpot Engineers, with new optimizations or features coming in with newer java versions.

通过本节的内容, 你应该对G1的基本功能有一定的了解。当然, 为了简洁, 就像处理一个庞大无比的对象一样, 我们省略了很多的实现细节。通过全方位的考虑, G1是HotSpot中可用的最先进的**准产品级(production-ready)**垃圾收集器。最重要的是, HotSpot 工程师的主要精力都用在不断改进G1上面, 在新的java版本中将会带来新的功能和优化。


As we have seen, G1 addressed a wide range of problems that CMS has, starting from pause predictability and ending with heap fragmentation. Given an application not constrained by CPU utilization, but very sensitive to the latency of individual operations, G1 is very likely to be the best available choice for HotSpot users, especially when running the latest versions of Java. However, these latency improvements do not come for free: throughput overhead of G1 is larger thanks to the additional write barriers and more active background threads. So, if the application is throughput-bound or is consuming 100% of CPU, and does not care as much about individual pause durations, then CMS or even Parallel may be better choices.

正如我们所见,G1 解决了 CMS 中各种各样的问题, 包括暂停时间的可预见性和以及终结了堆内存的碎片化。对于CPU使用不受限制, 但对单个业务的延迟非常敏感的系统, G1很可能是HotSpot中最好的选择,特别是在最新的Java版本中。然而,这种低延迟的改进也不是没有代价的: 由于额外的写屏障(write barriers)和更积极的后台线程, G1的开销会更大。所以, 如果系统属于吞吐量优先,或者CPU消耗了100%, 而又不在乎单次GC的暂停时间, 那么CMS应该是更好的选择。


> 总之: G1适合大内存,需要低延迟的场景。


The only viable way to select the right GC algorithm and settings is through trial and errors, but we do give the general guidelines in the next chapter.

选择正确的GC算法唯一可行的方式就是通过尝试和错误, 但在下一章我们将给出通用的指导原则。


Note that G1 will probably be the default GC for Java 9: http://openjdk.java.net/jeps/248

注意,G1可能会称为Java 9的默认GC: [http://openjdk.java.net/jeps/248](http://openjdk.java.net/jeps/248)


## Shenandoah 的性能


We have outlined all of the production-ready algorithms in HotSpot that you can just take and use right away. There is another one in the making, a so-called Ultra-Low-Pause-Time Garbage Collector. It is aimed for large multi-core machines with large heaps, the goal is to manage heaps of 100GB and larger with pauses of 10ms or shorter. This is traded off against throughput: the implementers are aiming at a no more than 10% of a performance penalty for applications with no GC pauses.

我们列出了所有的生产就绪算法热点,你可以马上取和使用。有另一个,所谓Ultra-Low-Pause-Time垃圾收集器.它目的是大堆的大型多核机器,目的是管理大量的100 gb和较大的停顿10 ms或短。这是交易与吞吐量:实现者是针对性能损失不超过10%的应用程序没有GC暂停。


We are not going to go into the implementation details before the new algorithm is released as production-ready, but it also builds upon many of the ideas already covered in earlier chapters, such as concurrent marking and incremental collecting. It does a lot of things differently, however. It does not split the heap into multiple generations, instead having a single space only. That’s right, Shenandoah is not a generational garbage collector. This allows it to get rid of card tables and remembered sets. It also uses forwarding pointers and a Brooks style read barrier to allow for concurrent copying of live objects, thus reducing the number and duration of pauses.

在新算法作为 production-ready 发布之前, 我们不去讨论具体的实现细节, 但它也构建在前面章节中提到的很多想法的基础上,例如并发标记和增量收集。然而,其中有很多东西是不同的。它不将堆分为多个代, 而是只有一个空间. 没错,Shenandoah 并不是一个分代垃圾收集器。这允许它摆脱card tables 和 remembered sets. 它还使用转发指针(forwarding pointers)和布鲁克斯风格的读取屏障(Brooks style read barrier)来允许存活对象的并发复制, 从而减少的停顿的次数和持续时间。


A lot of more detailed and up-to-date information about Shenandoah is available on the Internet, for instance in this blog: https://rkennke.wordpress.com/

关于 Shenandoah 性能的更多信息,请参考博客: [https://rkennke.wordpress.com/](https://rkennke.wordpress.com/), 或Google搜索。


原文链接: [GC Algorithms: Implementations](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations)






<div style="page-break-after : always;"> </div>


