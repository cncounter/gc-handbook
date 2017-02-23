# 6. GC 调优(工具篇)


进行GC性能调优时, 需要明确了解, 当前的GC行为对系统和用户有多大的影响。有多种监控GC的工具和方法,  本章将逐一介绍常用的工具。


JVM 在程序执行的过程中, 提供了GC行为的原生数据。那么, 我们就可以利用这些原生数据来生成各种报告。原生数据(*raw data*) 包括:


- 各个内存池的当前使用情况,
- 各个内存池的总容量,
- 每次GC暂停的持续时间,
- GC暂停在各个阶段的持续时间。


可以通过这些数据算出各种指标, 例如:  程序的内存分配率, 提升率等等。本章主要介绍如何获取原生数据。 后续的章节将对重要的派生指标(derived metrics)展开讨论, 并引入GC性能相关的话题。


## JMX API


从 JVM 运行时获取GC行为数据, 最简单的办法是使用标准 [JMX API 接口](https://docs.oracle.com/javase/tutorial/jmx/index.html). JMX是获取 JVM内部运行时状态信息 的标准API. 可以编写程序代码, 通过 JMX API 来访问本程序所在的JVM，也可以通过JMX客户端执行(远程)访问。


最常见的 JMX客户端是 [JConsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html) 和 [JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) (可以安装各种插件,十分强大)。两个工具都是标准JDK的一部分, 而且很容易使用. 如果使用的是 JDK 7u40 及更高版本, 还可以使用另一个工具: [Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)( 大致翻译为 Java控制中心, `jmc.exe`)。

> JVisualVM安装MBeans插件的步骤: 通过 工具(T) -- 插件(G) -- 可用插件 -- 勾选VisualVM-MBeans -- 安装 -- 下一步 -- 等待安装完成...... 其他插件的安装过程基本一致。


所有 JMX客户端都是独立的程序,可以连接到目标JVM上。目标JVM可以在本机, 也可能是远端JVM. 如果要连接远端JVM, 则目标JVM启动时必须指定特定的环境变量,以开启远程JMX连接/以及端口号。 示例如下:


	java -Dcom.sun.management.jmxremote.port=5432 com.yourcompany.YourApp


在此处, JVM 打开端口`5432`以支持JMX连接。


通过 JVisualVM  连接到某个JVM以后, 切换到 MBeans 标签, 展开 “java.lang/GarbageCollector” . 就可以看到GC行为信息, 下图是 JVisualVM 中的截图:


![](06_01_JMX-view.png)



下图是Java Mission Control 中的截图: 

![](06_02_JMX-view-Mbean.png)



从以上截图中可以看到两款垃圾收集器。其中一款负责清理年轻代(**PS Scavenge**)，另一款负责清理老年代(**PS MarkSweep**); 列表中显示的就是垃圾收集器的名称。可以看到 , jmc 的功能和展示数据的方式更强大。


对所有的垃圾收集器, 通过 JMX API 获取的信息包括:


- **CollectionCount** :  垃圾收集器执行的GC总次数,
- **CollectionTime**: 收集器运行时间的累计。这个值等于所有GC事件持续时间的总和,
- **LastGcInfo**: 最近一次GC事件的详细信息。包括 GC事件的持续时间(duration),  开始时间(startTime) 和 结束时间(endTime), 以及各个内存池在最近一次GC之前和之后的使用情况,
- **MemoryPoolNames**:  各个内存池的名称,
- **Name**: 垃圾收集器的名称
- **ObjectName**: 由JMX规范定义的 MBean的名字,,
- **Valid**: 此收集器是否有效。本人只见过 "`true`"的情况 (^_^)


根据经验, 这些信息对GC的性能来说,不能得出什么结论.  只有编写程序,  获取GC相关的 JMX 信息来进行统计和分析。 在下文可以看到, 一般也不怎么关注 MBean , 但 MBean  对于理解GC的原理倒是挺有用的。


## JVisualVM


[JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) 工具的 “[VisualGC](http://www.oracle.com/technetwork/java/visualgc-136680.html)” 插件提供了基本的 JMX客户端功能,  还实时显示出 GC事件以及各个内存空间的使用情况。


Visual GC 插件常用来监控本机运行的Java程序, 比如开发者和性能调优专家经常会使用此插件, 以快速获取程序运行时的GC信息。


![](06_03_jvmsualvm-garbage-collection-monitoring.png)


左侧的图表展示了各个内存池的使用情况: Metaspace/永久代, 老年代, Eden区以及两个存活区。


在右边, 顶部的两个图表与 GC无关, 显示的是 JIT编译时间 和 类加载时间。下面的6个图显示的是内存池的历史记录, 每个内存池的GC次数,GC总时间, 以及最大值，峰值, 当前使用情况。


再下面是 HistoGram, 显示了年轻代对象的年龄分布。至于对象的年龄监控(objects tenuring monitoring),  本章不进行讲解。


与纯粹的JMX工具相比, VisualGC 插件提供了更友好的界面, 如果没有其他趁手的工具, 请选择VisualGC. 本章接下来会介绍其他工具, 这些工具可以提供更多的信息, 以及更好的视角. 当然, 在“Profilers(分析器)”一节中，也会介绍 JVisualVM 的适用场景 —— 如: 分配分析(allocation profiling), 所以我们绝不会贬低哪一款工具, 关键还得看实际情况。


## jstat


[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html) 也是标准JDK提供的一款监控工具(Java Virtual Machine statistics monitoring tool),可以统计各种指标。既可以连接到本地JVM,也可以连到远程JVM.  查看支持的指标和对应选项可以执行 “`jstat -options`” 。例如:


	+-----------------+---------------------------------------------------------------+
	|     Option      |                          Displays...                          |
	+-----------------+---------------------------------------------------------------+
	|class            | Statistics on the behavior of the class loader                |
	|compiler         | Statistics  on  the behavior of the HotSpot Just-In-Time com- |
	|                 | piler                                                         |
	|gc               | Statistics on the behavior of the garbage collected heap      |
	|gccapacity       | Statistics of the capacities of  the  generations  and  their |
	|                 | corresponding spaces.                                         |
	|gccause          | Summary  of  garbage collection statistics (same as -gcutil), |
	|                 | with the cause  of  the  last  and  current  (if  applicable) |
	|                 | garbage collection events.                                    |
	|gcnew            | Statistics of the behavior of the new generation.             |
	|gcnewcapacity    | Statistics of the sizes of the new generations and its corre- |
	|                 | sponding spaces.                                              |
	|gcold            | Statistics of the behavior of the old and  permanent  genera- |
	|                 | tions.                                                        |
	|gcoldcapacity    | Statistics of the sizes of the old generation.                |
	|gcpermcapacity   | Statistics of the sizes of the permanent generation.          |
	|gcutil           | Summary of garbage collection statistics.                     |
	|printcompilation | Summary of garbage collection statistics.                     |
	+-----------------+---------------------------------------------------------------+



jstat 对于快速确定GC行为是否健康非常有用。启动方式为: “`jstat -gc -t PID 1s`” , 其中,PID 就是要监视的Java进程ID。可以通过 `jps` 命令查看正在运行的Java进程列表。


	jps

	jstat -gc -t 2428 1s


以上命令的结果, 是 jstat 每秒向标准输出输出一行新内容,  比如:


	Timestamp  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
	200.0  	 8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169344.0  21248.0 20534.3 3072.0 2807.7     34    0.720  658   133.684  134.404
	201.0 	 8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169343.2  21248.0 20534.3 3072.0 2807.7     34    0.720  662   134.712  135.432
	202.0 	 8448.0 8448.0 8102.5  0.0   67712.0  67598.5   169344.0   169343.6  21248.0 20534.3 3072.0 2807.7     34    0.720  667   135.840  136.559
	203.0 	 8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
	204.0 	 8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
	205.0 	 8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
	206.0 	 8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
	207.0 	 8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009
	208.0 	 8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009



稍微解释一下上面的内容。参考 [jstat manpage](http://www.manpagez.com/man/1/jstat/) , 我们可以知道:


- jstat 连接到 JVM 的时间, 是JVM启动后的 200秒。此信息从第一行的 “**Timestamp**” 列得知。继续看下一行, jstat 每秒钟从JVM 接收一次信息, 也就是命令行参数中 "`1s`" 的含义。
- 从第一行的 “**YGC**” 列得知年轻代共执行了34次GC,   由  “**FGC**” 列得知整个堆内存已经执行了 658次 full GC。
- 年轻代的GC耗时总共为 `0.720 秒`, 显示在“**YGCT**” 这一列。
- Full GC 的总计耗时为 `133.684 秒`, 由“**FGCT**”列得知。 这立马就吸引了我们的目光,  总的JVM 运行时间只有 200 秒, **但其中有 66% 的部分被 Full GC 消耗了**。


再看下一行, 问题就更明显了。


- 在接下来的一秒内共执行了 4 次 Full GC。参见 "**FGC**" 列.
- 这4次 Full GC 暂停占用了差不多 1秒的时间(根据 **FGCT**列的差得知)。与第一行相比,  Full GC 耗费了`928 毫秒`, 即 `92.8%` 的时间。
- 根据 “**OC** 和 “**OU**” 列得知, **整个老年代的空间**为 `169,344.0 KB` (“OC“), 在 4 次 Full GC 后依然占用了 `169,344.2 KB` (“OU“)。用了 `928ms` 的时间却只释放了 800 字节的内存, 怎么看都觉得很不正常。


只看这两行的内容, 就知道程序出了很严重的问题。继续分析下一行, 可以确定问题依然存在,而且变得更糟。


JVM几乎完全卡住了(stalled), 因为GC占用了90%以上的计算资源。GC之后, 所有的老代空间仍然还在占用。事实上, 程序在一分钟以后就挂了, 抛出了 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)”  错误。


可以看到, 通过 jstat 能很快发现对JVM健康极为不利的GC行为。一般来说, 只看 jstat 的输出就能快速发现以下问题:


- 最后一列 “**GCT**”, 与JVM的总运行时间 “**Timestamp**” 的比值, 就是GC 的开销。如果每一秒内, "**GCT**" 的值都会明显增大, 与总运行时间相比, 就暴露出GC开销过大的问题.  不同系统对GC开销有不同的容忍度, 由性能需求决定, 一般来讲, 超过 `10%` 的GC开销都是有问题的。
- “**YGC**” 和 “**FGC**” 列的快速变化往往也是有问题的征兆。频繁的GC暂停会累积,并导致更多的线程停顿(stop-the-world pauses), 进而影响吞吐量。
- 如果看到 “**OU**” 列中,老年代的使用量约等于老年代的最大容量(**OC**), 并且不降低的话, 就表示虽然执行了老年代GC, 但基本上属于无效GC。


## GC日志(GC logs)


通过日志内容也可以得到GC相关的信息。因为GC日志模块内置于JVM中,  所以日志中包含了对GC活动最全面的描述。 这就是事实上的标准, 可作为GC性能评估和优化的最真实数据来源。


GC日志一般输出到文件之中, 是纯 text 格式的, 当然也可以打印到控制台。有多个可以控制GC日志的JVM参数。例如,可以打印每次GC的持续时间, 以及程序暂停时间(`-XX:+PrintGCApplicationStoppedTime`), 还有GC清理了多少引用类型(`-XX:+PrintReferenceGC`)。


要打印GC日志, 需要在启动脚本中指定以下参数:


	-XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:<filename>


以上参数指示JVM:  将所有GC事件打印到日志文件中,  输出每次GC的日期和时间戳。不同GC算法输出的内容略有不同. ParallelGC 输出的日志类似这样:


	199.879: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1473386 secs] [Times: user=0.43 sys=0.01, real=0.15 secs]
	200.027: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1567794 secs] [Times: user=0.41 sys=0.00, real=0.16 secs]
	200.184: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1621946 secs] [Times: user=0.43 sys=0.00, real=0.16 secs]
	200.346: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1547695 secs] [Times: user=0.41 sys=0.00, real=0.15 secs]
	200.502: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1563071 secs] [Times: user=0.42 sys=0.01, real=0.16 secs]
	200.659: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1538778 secs] [Times: user=0.42 sys=0.00, real=0.16 secs]

在 “[04. GC算法:实现篇](http://blog.csdn.net/renfufei/article/details/54885190)” 中详细介绍了这些格式, 如果对此不了解, 可以先阅读该章节。 

分析以上日志内容, 可以得知:


- 这部分日志截取自JVM启动后200秒左右。
- 日志片段中显示, 在`780毫秒`以内, 因为垃圾回收 导致了5次 Full GC 暂停(去掉第六次暂停,这样更精确一些)。
- 这些暂停事件的总持续时间是 `777毫秒`, 占总运行时间的 **99.6%**。
- 在GC完成之后,  几乎所有的老年代空间(`169,472 KB`)依然被占用(`169,318 KB`)。


通过日志信息可以确定, 该应用的GC情况非常糟糕。JVM几乎完全停滞, 因为GC占用了超过`99%`的CPU时间。 而GC的结果是, 老年代空间仍然被占满,  这进一步肯定了我们的结论。 示例程序和jstat 小节中的是同一个, 几分钟之后系统就挂了, 抛出 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误, 不用说, 问题是很严重的.


从此示例可以看出, GC日志对监控GC行为和JVM是否处于健康状态非常有用。一般情况下, 查看 GC 日志就可以快速确定以下症状:


- GC开销太大。如果GC暂停的总时间很长, 就会损害系统的吞吐量。不同的系统允许不同比例的GC开销, 但一般认为, 正常范围在  `10%` 以内。
- 极个别的GC事件暂停时间过长。当某次GC暂停时间太长, 就会影响系统的延迟指标.  如果延迟指标规定交易必须在 `1,000 ms`内完成, 那就不能容忍任何超过 `1000毫秒`的GC暂停。
- 老年代的使用量超过限制。如果老年代空间在 Full GC 之后仍然接近全满, 那么GC就成为了性能瓶颈, 可能是内存太小, 也可能是存在内存泄漏。这种症状会让GC的开销暴增。


可以看到,GC日志中的信息非常详细。但除了这些简单的小程序,  生产系统一般都会生成大量的GC日志, 纯靠人工是很难阅读和进行解析的。


## GCViewer


我们可以自己编写解析器,  来将庞大的GC日志解析为直观易读的图形信息。 但很多时候自己写程序也不是个好办法,  因为各种GC算法的复杂性, 导致日志信息格式互相之间不太兼容。那么神器来了: [GCViewer](https://github.com/chewiebug/GCViewer)。


[GCViewer](https://github.com/chewiebug/GCViewer) 是一款开源的GC日志分析工具。项目的 GitHub 主页对各项指标进行了完整的描述.  下面我们介绍最常用的一些指标。


第一步是获取GC日志文件。这些日志文件要能够反映系统在性能调优时的具体场景. 假若运营部门(operational department)反馈: 每周五下午,系统就运行缓慢, 不管GC是不是主要原因, 分析周一早晨的日志是没有多少意义的。


获取到日志文件之后, 就可以用 GCViewer 进行分析, 大致会看到类似下面的图形界面:


![](06_04_gcviewer-screenshot.png)


使用的命令行大致如下:

	java -jar gcviewer_1.3.4.jar gc.log


当然, 如果不想打开程序界面,也可以在后面加上其他参数,直接将分析结果输出到文件。

命令大致如下: 

	java -jar gcviewer_1.3.4.jar gc.log summary.csv chart.png


以上命令将信息汇总到当前目录下的 Excel 文件 `summary.csv` 之中, 将图形信息保存为 `chart.png` 文件。

点击下载: [gcviewer的jar包及使用示例](http://download.csdn.net/detail/renfufei/9753654) 。


上图中, Chart 区域是对GC事件的图形化展示。包括各个内存池的大小和GC事件。上图中, 只有两个可视化指标: 蓝色线条表示堆内存的使用情况, 黑色的Bar则表示每次GC暂停时间的长短。


从图中可以看到, 内存使用量增长很快。一分钟左右就达到了堆内存的最大值.  堆内存几乎全部被消耗, 不能顺利分配新对象, 并引发频繁的 Full GC 事件. 这说明程序可能存在内存泄露, 或者启动时指定的内存空间不足。


从图中还可以看到 GC暂停的频率和持续时间。`30秒`之后, GC几乎不间断地运行,最长的暂停时间超过`1.4秒`。


在右边有三个选项卡。“**`Summary`**(摘要)” 中比较有用的是 “`Throughput`”(吞吐量百分比) 和 “`Number of GC pauses`”(GC暂停的次数), 以及“`Number of full GC pauses`”(Full GC 暂停的次数). 吞吐量显示了有效工作的时间比例, 剩下的部分就是GC的消耗。


以上示例中的吞吐量为 **`6.28%`**。这意味着有 **`93.72%`** 的CPU时间用在了GC上面. 很明显系统所面临的情况很糟糕 —— 宝贵的CPU时间没有用于执行实际工作, 而是在试图清理垃圾。


下一个有意思的地方是“**Pause**”(暂停)选项卡:


![](06_05_gviewer-screenshot-pause.png)


“`Pause`” 展示了GC暂停的总时间,平均值,最小值和最大值, 并且将 total 与minor/major 暂停分开统计。如果要优化程序的延迟指标, 这些统计可以很快判断出暂停时间是否过长。另外, 我们可以得出明确的信息: 累计暂停时间为 `634.59 秒`, GC暂停的总次数为 `3,938 次`, 这在`11分钟/660秒`的总运行时间里那不是一般的高。


更详细的GC暂停汇总信息, 请查看主界面中的 “**Event details**” 标签:


![](06_06_gcviewer-screenshot-eventdetails.png)


从“**Event details**” 标签中, 可以看到日志中所有重要的GC事件汇总: `普通GC停顿` 和 `Full GC 停顿次数`, 以及`并发执行数`, `非 stop-the-world 事件`等。此示例中, 可以看到一个明显的地方, Full GC 暂停严重影响了吞吐量和延迟, 依据是: `3,928 次 Full GC`, 暂停了`634秒`。


可以看到, GCViewer 能用图形界面快速展现异常的GC行为。一般来说, 图像化信息能迅速揭示以下症状:


- 低吞吐量。当应用的吞吐量下降到不能容忍的地步时, 有用工作的总时间就大量减少. 具体有多大的 “容忍度”(tolerable) 取决于具体场景。按照经验, 低于 90% 的有效时间就值得警惕了, 可能需要好好优化下GC。
- 单次GC的暂停时间过长。只要有一次GC停顿时间过长,就会影响程序的延迟指标. 例如, 延迟需求规定必须在 1000 ms以内完成交易, 那就不能容忍任何一次GC暂停超过1000毫秒。
- 堆内存使用率过高。如果老年代空间在 Full GC 之后仍然接近全满,  程序性能就会大幅降低,  可能是资源不足或者内存泄漏。这种症状会对吞吐量产生严重影响。


业界良心 —— 图形化展示的GC日志信息绝对是我们重磅推荐的。不用去阅读冗长而又复杂的GC日志,通过容易理解的图形, 也可以得到同样的信息。


## 分析器(Profilers)


下面介绍分析器([profilers](http://zeroturnaround.com/rebellabs/developer-productivity-report-2015-java-performance-survey-results/3/), Oracle官方翻译是:抽样器)。相对于前面介绍的工具, 分析器只关心GC的一部分领域. 本节我们只关注分析器相关的GC功能。


首先警告 —— 分析器往往会被认为适合所有的场景。有时候分析器确实功勋卓著, 比如检测代码中的CPU热点时。但对某些情况不一定是个好方案。


对GC调优来说也是同样的。要检测是否因为GC而引起延迟或吞吐量问题时,并不需要使用分析器. 前面提到的工具( jstat 或 原生/可视化GC日志)都能更好更快地检测出是否需要关注GC. 特别是从生产环境收集数据时, 最好不要选择分析器, 因为性能开销实在是太大了。


如果确定需要对GC进行优化, 那么分析器就可以发挥重要的作用, 让 Object 的创建信息一目了然. 再往前看, 造成大量GC暂停的原因不在某个特定内存池中。那就只能发生创建对象的时候. 所有的分析器都能够跟踪对象分配(via allocation profiling), 根据内存分配的轨迹, 让你知道**实际驻留在内存中的是哪些对象**。


分配分析能定位到哪个地方创建了大量的对象. 使用分析器辅助进行GC调优的好处是, 能确定是什么类型最占用内存, 以及是哪些线程生成了最多的对象。


我们将在实例中介绍三种不同的分配分析器: **`hprof`**, **`JVisualV`M** 和 **`AProf`**。实际上还有很多分析器可供选择, 包括商业的和免费的, 但其功能和优点都类似于下面讨论的这些。


### hprof


[hprof 分析器](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html)内置于JDK之中。在各种环境都可以使用, 一般优先使用这款工具。


要让 hprof 和程序一起运行, 需要修改启动脚本, 类似这样:


	java -agentlib:hprof=heap=sites com.yourcompany.YourApplication


在程序退出时,会将分配信息dump(转储)到工作目录下的 java.hprof.txt 文件。使用文本编辑器打开, 并搜索 “**SITES BEGIN**” 关键字, 可以看到:


	SITES BEGIN (ordered by live bytes) Tue Dec  8 11:16:15 2015
	          percent          live          alloc'ed  stack class
	 rank   self  accum     bytes objs     bytes  objs trace name
	    1  64.43% 4.43%   8370336 20121  27513408 66138 302116 int[]
	    2  3.26% 88.49%    482976 20124   1587696 66154 302104 java.util.ArrayList
	    3  1.76% 88.74%    241704 20121   1587312 66138 302115 eu.plumbr.demo.largeheap.ClonableClass0006
	    ... 部分省略 ...
	
	SITES END


从以上片段可以看到, allocations 是根据每次创建的对象数量来排序的。第一行显示所有对象中有 **64.43%** 的对象是整型数组(`int[]`), 在标识为数字 302116 的位置创建。搜索 “**TRACE 302116**” 可以看到:


	TRACE 302116:	
		eu.plumbr.demo.largeheap.ClonableClass0006.<init>(GeneratorClass.java:11)
		sun.reflect.GeneratedConstructorAccessor7.newInstance(<Unknown Source>:Unknown line)
		sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		java.lang.reflect.Constructor.newInstance(Constructor.java:422)



现在, 知道了 64.43% 的对象被分配为整数数组, 在 ClonableClass0006 类的构造函数中, 第11行, 那么就可以优化代码, 以减少GC的压力。


### Java VisualVM


这是本章第二次介绍 JVisualVM 了。在第一部分, 在监控 JVM 的GC行为工具中介绍了 JVisualVM , 本节将介绍其在分配分析上的优点。


JVisualVM 通过GUI方式连接到正在运行的JVM。 连接上profiler以后 :


1. 打开 “工具” --> “选项” 菜单, 点击 **性能分析(Profiler)** 标签, 新增配置, 选择 Profiler 内存, 确保勾选了 “Record allocations stack traces”(记录分配栈跟踪)。
2. 勾选 “Settings”(设置) 复选框, 在内存设置标签下,修改预设配置。
3. 点击 “Memory”(内存) 按钮开始进行内存分析。
4. 让程序运行一段时间,以收集关于对象分配的足够信息。
5. 单击下面一点的 “Snapshot”(快照) 按钮。可以取得收集到的信息快照。


![](06_07_01_trace.png)


完成上面的步骤后, 可以得到类似这样的信息:


![](06_07_jvisualvm-top-objects.png)


上图是按照每个类的对象创建数量来排序的。从第一行可以看到,分配的最多的对象是 `int[]` 数组. 右键单击这行就可以看到这些对象都是在哪个地方分配的:


![](06_08_jvisualvm-allocation-traces.png)


与 `hprof` 相比, JVisualVM 更加容易 —— 比如在上面的截图中, 在一个地方就可以看到所有的`int[]` 分配信息, 所以多次在同一处代码进行分配时就很容易看到了。


### AProf


第三款, 同时也是最重要的工具,是由 Devexperts 开发的 **[AProf](https://code.devexperts.com/display/AProf/About+Aprof)**。AProf 也是打包为 Java agent 的内存分配分析器。


用AProf 来分析应用程序, 需要修改JVM的启动脚本,类似这样:


	java -javaagent:/path-to/aprof.jar com.yourcompany.YourApplication


重启应用程序以后, 工作目录下会生成一个 aprof.txt 文件。这个文件每分钟更新一次, 包含这样的信息:


	========================================================================================================================
	TOTAL allocation dump for 91,289 ms (0h01m31s)
	Allocated 1,769,670,584 bytes in 24,868,088 objects of 425 classes in 2,127 locations
	========================================================================================================================
	
	Top allocation-inducing locations with the data types allocated from them
	------------------------------------------------------------------------------------------------------------------------
	eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 1,423,675,776 (80.44%) bytes in 17,113,721 (68.81%) objects (avg size 83 bytes)
		int[]: 711,322,976 (40.19%) bytes in 1,709,911 (6.87%) objects (avg size 416 bytes)
		char[]: 369,550,816 (20.88%) bytes in 5,132,759 (20.63%) objects (avg size 72 bytes)
		java.lang.reflect.Constructor: 136,800,000 (7.73%) bytes in 1,710,000 (6.87%) objects (avg size 80 bytes)
		java.lang.Object[]: 41,079,872 (2.32%) bytes in 1,710,712 (6.87%) objects (avg size 24 bytes)
		java.lang.String: 41,063,496 (2.32%) bytes in 1,710,979 (6.88%) objects (avg size 24 bytes)
		java.util.ArrayList: 41,050,680 (2.31%) bytes in 1,710,445 (6.87%) objects (avg size 24 bytes)
	          ... cut for brevity ... 



上面的输出是按照 size 进行排序的。可以看出, `80.44%` 的 bytes 和 68.81% 的 objects 是在 `ManyTargetsGarbageProducer.newRandomClassObject()` 方法中分配的。 其中, **int[]** 数组消耗了最多的内存, 占总量的 40.19%。


继续往下看, 你会发现 allocation traces(分配痕迹)相关的内容, 也是根据 allocation size 来排序的:


	Top allocated data types with reverse location traces
	------------------------------------------------------------------------------------------------------------------------
	int[]: 725,306,304 (40.98%) bytes in 1,954,234 (7.85%) objects (avg size 371 bytes)
		eu.plumbr.demo.largeheap.ClonableClass0006.: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
			java.lang.reflect.Constructor.newInstance: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
				eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 38,357,280 (2.16%) bytes in 92,205 (0.37%) objects (avg size 416 bytes)
				java.lang.reflect.Constructor.newInstance: 416 (0.00%) bytes in 1 (0.00%) objects (avg size 416 bytes)
	... cut for brevity ... 



可以看到, `int[]` 数组的分配, 在 `ClonableClass0006` 构造函数中继续增大。


和其他工具一样, AProf 揭露了 分配大小以及位置信息(`allocation size and locations`), 从而能够快速找到最耗内存的部分。在我们看来, **AProf** 是最有用的分配分析器, 因为它只专注于内存分配, 而且做的非常棒。 当然, 此工具是开源免费的, 资源开销也是最小的。


原文链接:  [GC Tuning: Tooling](https://plumbr.eu/handbook/gc-tuning-measuring)

