# 7. GC 调优(实战篇)


This chapter covers several typical performance problems that one may encounter with garbage collection. The examples given here are derived from real applications, but are simplified for the sake of clarity.

本章介绍几种垃圾收集可能导致的典型的性能问题。其中的示例都来自于真实的应用,但为了演示方便,做了一些简化处理。


## 高分配率(High Allocation Rate)


Allocation rate is a term used when communicating the amount of memory allocated per time unit. Often it is expressed in MB/sec, but you can use PB/year if you feel like it. So that is all there is – no magic, just the amount of memory you allocate in your Java code measured over a period of time.


分配率(Allocation rate)是用来表示每时间单位内,内存分配数量的术语。通常表示为 MB/sec, 但也可以使用 PB/year。很简单, 没什么复杂的, 在Java代码中通过一段时间周期来衡量内存分配的数量。




An excessively high allocation rate can mean trouble for your application’s performance. When running on a JVM, the problem will be revealed by garbage collection posing a large overhead.

过高的分配率可能会严重影响程序的性能。在JVM中, 这个问题的影响是带来大量的GC开销。


### 如何衡量分配率?


One way to measure the allocation rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses similar to the following:

测量分配率的一种方法是,通过指定 `-XX:+PrintGCDetails -XX:+PrintGCTimeStamps` 标志开启JVM的GC日志. 则JVM开始记录类似下面这样的GC停顿日志:


	0.291: [GC (Allocation Failure) 
			[PSYoungGen: 33280K->5088K(38400K)] 
			33280K->24360K(125952K), 0.0365286 secs] 
		[Times: user=0.11 sys=0.02, real=0.04 secs] 
	0.446: [GC (Allocation Failure) 
			[PSYoungGen: 38368K->5120K(71680K)] 
			57640K->46240K(159232K), 0.0456796 secs] 
		[Times: user=0.15 sys=0.02, real=0.04 secs] 
	0.829: [GC (Allocation Failure) 
			[PSYoungGen: 71680K->5120K(71680K)] 
			112800K->81912K(159232K), 0.0861795 secs] 
		[Times: user=0.23 sys=0.03, real=0.09 secs]




From the GC log above, we can calculate the allocation rate as the difference between the sizes of the young generation after the completion of the last collection and before the start of the next one. Using the example above, we can extract the following information:

从上面的GC日志中,我们就可以计算出分配率. 通过上一次垃圾收集之后,与下一次GC开始之前的年轻代大小的差值. 比如上面的示例中, 我们可以得到如下信息:


- At 291 ms after the JVM was launched, 33,280 K of objects were created. The first minor GC event cleaned the young generation, after which there were 5,088 K of objects in the young generation left.
- At 446 ms after launch, the young generation occupancy had grown to 38,368 K, triggering the next GC, which managed to reduce the young generation occupancy to 5,120 K.
- At 829 ms after the launch, the size of the young generation was 71,680 K and the GC reduced it again to 5,120 K.

<br/>

- 在JVM启动后 **291ms**, 创建了 `33,280 KB` 的对象。 第一次清理年轻代的 Minor GC(小型GC)事件, 完成之后还有 `5,088 KB` 的对象存在于年轻代中。
- 在启动 **446 ms**之后, 年轻一代使用量增长至 `38,368 KB`,触发了下一次GC, 然后让年轻代使用量减少到了 `5120 KB`。
- 在启动后 **829 ms**, 年轻代的大小为 `71,680 KB`, GC后再让其减少到 `5,120 KB`。



This data can then be expressed in the following table calculating the allocation rate as deltas of the young occupancy:

然后可以通过年轻代的变化来计算出分配率,如下表所示:


<table class="data compact">
<thead>
<tr>
<th><strong>Event</strong></th>
<th><strong>Time</strong></th>
<th><strong>Young before</strong></th>
<th><strong>Young after</strong></th>
<th><strong>Allocated during</strong></th>
<th><strong>Allocation rate</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>1st GC</td>
<td>291ms</td>
<td>33,280KB</td>
<td>5,088KB</td>
<td>33,280KB</td>
<td><strong>114MB/sec</strong></td>
</tr>
<tr>
<td>2nd GC</td>
<td>446ms</td>
<td>38,368KB</td>
<td>5,120KB</td>
<td>33,280KB</td>
<td><strong>215MB/sec</strong></td>
</tr>
<tr>
<td>3rd GC</td>
<td>829ms</td>
<td>71,680KB</td>
<td>5,120KB</td>
<td>66,560KB</td>
<td><strong>174MB/sec</strong></td>
</tr>
<tr>
<td>Total</td>
<td>829ms</td>
<td>N/A</td>
<td>N/A</td>
<td>133,120KB</td>
<td><strong>161MB/sec</strong></td>
</tr>
</tbody>
</table>




Having this information allows us to say that this particular piece of software had the allocation rate of 161 MB/sec during the period of measurement.

通过这个信息,我们可以说在测量期间, 该软件的分配率是 **161 MB/秒**。。



### 为什么要关心分配率?


After measuring the allocation rate we can understand how the changes in allocation rate affect application throughput by increasing or reducing the frequency of GC pauses. First and foremost, you should notice that only minor GC pauses cleaning the young generation are affected. Neither the frequency nor duration of the GC pauses cleaning the old generation are directly impacted by the allocation rate, but instead by the promotion rate, a term that we will cover separately in the next section.


测量分配率之后,我们就可以理解分配率的变化是如何影响应用程序的吞吐量的: 因为分配率会增加或减少GC暂停的频率。首先,也是最重要的,你应该注意到,只有清理年轻代的 minor GC pauses 受影响.而清理老年代的GC暂停, 其频率和持续时间都不受分配率的直接影响, 而是受**提升率**(promotion rate,晋升率)的影响,在下一节中我们将单独介绍这个术语。


Knowing that we can focus only on Minor GC pauses, we should next look into the different memory pools inside the young generation. As the allocation takes place in Eden, we can immediately look into how sizing Eden can impact the allocation rate. So we can hypothesize that increasing the size of Eden will reduce the frequency of minor GC pauses and thus allow the application to sustain faster allocation rates.

这里我们只关心 Minor GC 暂停, 接下来应该考虑的是年轻代中的不同内存池。因为分配发生在 Eden 区, 我们可以立即调查如何设置 Eden 的大小来影响分配率. 所以我们可以增加 Eden 的大小来看看是否会减少 Minor GC 暂停的频率, 从而使程序能够维持更快的分配率。


And indeed, when running the same application with different Eden sizes using -XX:NewSize -XX:MaxNewSize & -XX:SurvivorRatio parameters, we can see a two-fold difference in allocation rates.

的确,当通过 `-XX:NewSize -XX:MaxNewSize & -XX:SurvivorRatio` 参数设置不同的 Eden 空间来运行同一应用程序时, 我们可以看到分配率有两种不同的差别。


- Re-running with 100 M of Eden reduces the allocation rate to below 100 MB/sec.
- Increasing Eden size to 1 GB increases the allocation rate to just below 200 MB/sec.

<br/>

- 用 100 MB的 Eden 空间来运行, 分配率会低于 **100 MB/秒**。
- 增加Eden区的大小为 **1 GB**, 分配率也跟着增长,大约是 **200 MB/秒** 的样子。


If you are still wondering how this can be true – if you stop your application threads for GC less frequently you can do more useful work. More useful work also happens to create more objects, thus supporting the increased allocation rate.


为什么会这样? —— 因为减少暂停所有线程的GC次数，则可以做更多有用功。更有用的工作也创造了更多的对象, 因此同样的程序,分配率增加是挺好的事情。


Now, before you jump to the conclusion that “bigger Eden is better”, you should notice that the allocation rate might and probably does not directly correlate with the actual throughput of your application. It is a technical measurement contributing to throughput. The allocation rate can and will have an impact on how frequently your minor GC pauses stop application threads, but to see the overall impact, you also need to take into account major GC pauses and measure throughput not in MB/sec but in the business operations your application provides.

那么, 在得出 “Eden去越大越好” 这种结论之前, 你应该注意到分配率可能会,也可能不会直接影响到程序的实际吞吐量。 这是和吞吐量有关系的一个技术指标. 分配率会影响让所以线程停止的 minor GC暂停, 但对于总体影响, 还要考虑 Major GC(大型GC)暂停, 而且衡量吞吐量的单位不是 **MB/秒**， 而是程序处理的业务量。


### 示例


Meet the demo application. Suppose that it works with an external sensor that provides a number. The application continuously updates the value of the sensor in a dedicated thread (to a random value, in this example), and from other threads sometimes uses the most recent value to do something meaningful with it in the processSensorValue() method:

为了满足演示的需要。假设应用与提供一个数字的外部传感器协同工作。应用程序通过一个专用线程,不断更新传感器的值,(在本例中使用的是随机值), 有时从其他线程用最新的值来做一些有意义的事情, 调用 `processSensorValue()` 方法:


	public class BoxingFailure {
	  private static volatile Double sensorValue;
	
	  private static void readSensor() {
	    while(true) sensorValue = Math.random();
	  }
	
	  private static void processSensorValue(Double value) {
	    if(value != null) {
	      //...
	    }
	  }
	}




As the name of the class suggests, the problem here is boxing. Possibly to accommodate the null check, the author made the sensorValue field a capital-D Double. This example is quite a common pattern of dealing with calculations based on the most recent value, when obtaining this value is an expensive operation. And in the real world, it is usually much more expensive than just getting a random value. Thus, one thread continuously generates new values, and the calculating thread uses them, avoiding the expensive retrieval.

如同类名所示, 这个示例问题是拳击(boxing)。为了适应 null 检查, 作者将 `sensorValue` 属性设置为包装类型 `Double`. 这个例子是基于最新数值进行计算的普遍模式, 如果获取这个值是一个笨重的操作时。在实际情况下, 通常比获取随机值的代价要高很多。因此,一个线程不断生成新值, 而计算线程使用它们,避免昂贵的检索。


The demo application is impacted by the GC not keeping up with the allocation rate. The ways to verify and solve the issue are given in the next sections.

演示程序中由于GC跟不上分配率而受到影响。在接下来的部分将验证并给出解决问题的方法。


### Could my JVMs be Affected?

### 我的 JVM 会受影响吗?



First and foremost, you should only be worried if the throughput of your application starts to decrease. As the application is creating too much objects that are almost immediately discarded, the frequency of minor GC pauses surges. Under enough of a load this can result in GC having a significant impact on throughput.

首先，最重要的是,你应该只关心应用程序的吞吐量是否开始减少。如果程序创建了太多立即丢弃的对象,小型GC暂停就会激增。负荷足够的情况下这会导致GC对吞吐量产生显著的影响。


When you run into a situation like this, you would be facing a log file similar to the following short snippet extracted from the GC logs of the demo application introduced in the previous section. The application was launched as with the -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m command line arguments:

当遇到这种情况时,你将看到类似下面这样的日志片段，从前一节中介绍的[示例程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java) 产生的GC日志中提取出来的. 命令行启动参数为 **-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m**:


	2.808: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003076 secs]
	2.819: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003079 secs]
	2.830: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0002968 secs]
	2.842: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
	2.853: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0004672 secs]
	2.864: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003371 secs]
	2.875: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003214 secs]
	2.886: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
	2.896: [GC (Allocation Failure) 
			[PSYoungGen: 9760K->32K(10240K)], 0.0003588 secs]




What should immediately grab your attention is the frequency of minor GC events. This indicates that there are lots and lots of objects being allocated. Additionally, the post-GC occupancy of the young generation remains low, and no full collections are happening. These symptoms indicate that the GC is having significant impact to the throughput of the application at hand.

吸引你眼光的应该是小型GC的频率。这表明有很多很多的对象被分配。另外,年轻代的 post-GC 入住率仍然很低,也没有 full GC 发生.这些症状表明, GC对应用程序的吞吐量有重大影响。



### 解决方案


In some cases, reducing the impact of high allocation rates can be as easy as increasing the size of the young generation. Doing so will not reduce the allocation rate itself, but will result in less frequent collections. The benefit of the approach kicks in when there will be only a few survivors every time. As the duration of a minor GC pause is impacted by the number of surviving objects, they will not noticeably increase here.

在某些情况下,减少高分配率的影响也很容易,只要增加年轻代的大小即可。这样做不会减少分配率本身,但是会减少垃圾收集的频率。这样的好处是每次都只有少数的存活对象.小型GC的暂停时间受存活对象数量的影响,但这个数量并不会明显增加。


The result is visible when we run the very same demo application with increased heap size and, with it, the young generation size, by using the -Xmx64m parameter:

结果是可见的,当我们增加堆内存的大小来运行 [示例程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java) 时,同时也增大了年轻代的大小,通过使用参数 `-Xmx64m`:


	2.808: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003748 secs]
	2.831: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0004538 secs]
	2.855: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003355 secs]
	2.879: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0005592 secs]




However, just throwing more memory at it is not always a viable solution. Equipped with the knowledge on allocation profilers from the previous chapter, we may find out where most of the garbage is produced. Specifically, in this case, 99% are Doubles that are created with the readSensor method. As a simple optimization, the object can be replaced with a primitive double, and the null can be replaced with Double.NaN. Since primitive values are not actually objects, no garbage is produced, and there is nothing to collect. Instead of allocating a new object on the heap, a field in an existing object is directly overwritten.

然而,仅仅增加内存的大小,并不总是一个可行的方案。通过前面章节了解到的分配分析器知识,我们可以找到大部分垃圾产生的地方。具体地说,在这种情况下,99%的是 **readSensor** 方法中创建的 **Double** 对象。作为一个简单的优化,对象可以被替换为原生类型 `double` , 而 null 值可以用  [Double.NaN](https://docs.oracle.com/javase/7/docs/api/java/lang/Double.html#NaN) 来替换。由于原始值不是实际的对象,所以不产生垃圾,也就没有垃圾收集。不在堆上分配一个新对象,而是直接覆盖对象的属性域。


The simple change (diff) will, in the demo application, almost completely remove GC pauses. In some cases, the JVM may be clever enough to remove excessive allocations itself by using the escape analysis technique. To cut a long story short, the JIT compiler may in some cases prove that a created object never “escapes” the scope it is created in. In such cases, there is no actual need to allocate it on the heap and produce garbage this way, so the JIT compiler does just that: it eliminates the allocation. See this benchmark for an example.

对示例程序进行 [简单的改变](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/FixedBoxing.java)( [diff](https://gist.github.com/gvsmirnov/0270f0f15f9498e3b655) ),几乎完全消除了GC暂停。在某些情况下,JVM可能会足够聪明,通过使用逃离分析技术来避免过度分配。长话短说,JIT编译器在某些情况下,创建的对象可能永远不会“逃出”创建它的作用域。在这种情况下,不需要在堆上进行实际的分配并产生垃圾,所以JIT编译器做的就是: 消除了分配。请参考 这个[基准测试的例子](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/jit/EscapeAnalysis.java)。



## 过早的晋升(Premature Promotion)



##
##
##
##
##
##


Before explaining the concept of premature promotion, we should familiarize ourselves with the concept it builds upon – the promotion rate. The promotion rate is measured in the amount of data propagated from the young generation to the old generation per time unit. It is often measured in MB/sec, similarly to the allocation rate.

Before explaining the concept of premature promotion, we should familiarize ourselves with the concept of it builds upon - the promotion rate.促进率测量的数据从旧一代的年轻一代传播/时间单位。通常以MB /秒,同样分配率。


Promoting long-lived objects from the young generation to the old is how JVM is expected to behave. Recalling the generation hypothesis we can now construct a situation where not only long-lived objects end up in the old generation. Such a situation, where objects with a short life expectancy are not collected in the young generation and get promoted to the old generation, is called premature promotion.

促进长寿对象从年轻一代老预计JVM是如何表现的.回忆一代假设我们现在可以构建一个情况不仅长寿对象最终在旧的一代。这种情况下,对象的短寿命没有收集在年轻一代和老一代得到提升,被称为过早晋升。


Cleaning these short-lived objects now becomes a job for major GC, which is not designed for frequent runs and results in longer GC pauses. This significantly affects the throughput of the application.

清理这些短暂的对象现在成为主要的GC工作,这不是专为频繁运行,导致GC暂停时间长.这大大影响应用程序的吞吐量。


### How to Measure Promotion Rate

### 如何衡量促销利率


One of the ways you can measure the promotion rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses just like in the following snippet:

你可以测量提升速度的方法之一是打开GC日志记录通过指定- xx:+ PrintGCDetails - xx:+ PrintGCTimeStamps旗帜的JVM.JVM现在开始记录GC暂停就像在以下代码片段:


	0.291: [GC (Allocation Failure) 
			[PSYoungGen: 33280K->5088K(38400K)] 
			33280K->24360K(125952K), 0.0365286 secs] 
		[Times: user=0.11 sys=0.02, real=0.04 secs] 
	0.446: [GC (Allocation Failure) 
			[PSYoungGen: 38368K->5120K(71680K)] 
			57640K->46240K(159232K), 0.0456796 secs] 
		[Times: user=0.15 sys=0.02, real=0.04 secs] 
	0.829: [GC (Allocation Failure) 
			[PSYoungGen: 71680K->5120K(71680K)] 
			112800K->81912K(159232K), 0.0861795 secs] 
		[Times: user=0.23 sys=0.03, real=0.09 secs]




From the above we can extract the size of the young Generation and the total heap both before and after the collection event. Knowing the consumption of the young generation and the total heap, it is easy to calculate the consumption of the old generation as just the delta between the two. Expressing the information in GC logs as:

从上面我们可以提取年轻一代的大小和总堆前后收集的事件。了解年轻一代的消费和总堆,很容易计算旧一代的消费只是两者之间的增量。GC日志中的信息表达为:


<table class="data compact">
<thead>
<tr>
<th><strong>Event</strong></th>
<th><strong>Time</strong></th>
<th><strong>Young decreased</strong></th>
<th><strong>Total decreased</strong></th>
<th><strong>Promoted</strong></th>
<th><strong>Promotion rate</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>1st GC</td>
<td>291ms</td>
<td>28,192K</td>
<td>8,920K</td>
<td>19,272K</td>
<td><strong>66.2 MB/sec</strong></td>
</tr>
<tr>
<td>2nd GC</td>
<td>446ms</td>
<td>33,248K</td>
<td>11,400K</td>
<td>21,848K</td>
<td><strong>140.95 MB/sec</strong></td>
</tr>
<tr>
<td>3rd GC</td>
<td>829ms</td>
<td>66,560K</td>
<td>30,888K</td>
<td>35,672K</td>
<td><strong>93.14 MB/sec</strong></td>
</tr>
<tr>
<td>Total</td>
<td>829ms</td>
<td></td>
<td></td>
<td>76,792K</td>
<td><strong>92.63 MB/sec</strong></td>
</tr>
</tbody>
</table>




will allow us to extract the promotion rate for the measured period. We can see that on average the promotion rate was 92 MB/sec, peaking at 140.95 MB/sec for a while.

将使我们能够提取的晋升速度测量。我们可以看到,平均提升率为92 MB /秒,峰值为140.95 MB /秒。


Notice that you can extract this information only from minor GC pauses. Full GC pauses do not expose the promotion rate as the change in the old generation usage in GC logs also includes objects cleaned by the major GC.

请注意,您可以提取这些信息只能从次要的GC暂停.完整GC暂停不公开促进率的改变在旧一代使用GC日志中还包括清洗的对象主要的GC。


### 为什么要关心


Similarly to the allocation rate, the main impact of the promotion rate is the change of frequency in GC pauses. But as opposed to the allocation rate that affects the frequency of minor GC events, the promotion rate affects the frequency of major GC events. Let me explain – the more stuff you promote to the old generation the faster you will fill it up. Filling the old generation faster means that the frequency of the GC events cleaning the old generation will increase.

同样的分配率,促进率的主要影响是GC暂停时间频率的变化。但随着反对分配率影响很小的GC事件的频率,促进利率影响重大GC事件的频率。让我解释——更多的东西你促进旧一代越快你就会把它填平.填补旧一代更快意味着GC事件清理旧一代的频率将会增加。


![](07_01_how-java-garbage-collection-works.png)




As we have shown in earlier chapters, full garbage collections typically require much more time, as they have to interact with many more objects, and perform additional complex activities such as defragmentation.

在之前的章节中所示,完整的垃圾收集通常需要更多的时间,与更多的对象进行交互,并执行碎片整理等附加的复杂活动。


### 示例


Let us look at a demo application suffering from premature promotion. This app obtains chunks of data, accumulates them, and, when a sufficient number is reached, processes the whole batch at once:

让我们看一个演示应用程序遭受过早晋升。这个程序获得的大量数据,积累,当达到足够数量时,整个批处理流程:


	public class PrematurePromotion {
	
	   private static final Collection<byte[]> accumulatedChunks 
					= new ArrayList<>();
	
	   private static void onNewChunk(byte[] bytes) {
	       accumulatedChunks.add(bytes);
	
	       if(accumulatedChunks.size() > MAX_CHUNKS) {
	           processBatch(accumulatedChunks);
	           accumulatedChunks.clear();
	       }
	   }
	}




The demo application is impacted by premature promotion by the GC. The ways to verify and solve the issue are given in the next sections.

演示应用程序由GC过早促进影响。验证并给出解决问题的方法在接下来的部分。


### Could my JVMs be Affected?

### 我的jvm会受影响吗?


In general, the symptoms of premature promotion can take any of the following forms:

一般来说,过早的症状促销可以采取下列形式:


- The application goes through frequent full GC runs over a short period of time.
- The old generation consumption after each full GC is low, often under 10-20% of the total size of the old generation.
- Facing the promotion rate approaching the allocation rate.

- 应用程序通过频繁的完整GC运行在很短的时间内。
- 旧一代消费每满GC很低,通常在10 - 20%的总大小的一代。
- 面临着晋升率接近分配率。



Showcasing this in a short and easy-to-understand demo application is a bit tricky, so we will cheat a little by making the objects tenure to the old generation a bit earlier than it happens by default. If we ran the demo with a specific set of GC parameters (-Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1), we would see this in the garbage collection logs:

在短和易于理解的演示应用程序展示这是有点棘手,所以我们将作弊通过使对象任期早一点旧一代比默认情况下发生。如果我们的演示与一组特定的GC参数(-Xmx24m - xx:NewSize = 16 m - xx:MaxTenuringThreshold = 1),我们可以看到这个垃圾收集日志:


	2.176: [Full GC (Ergonomics) 
			[PSYoungGen: 9216K->0K(10752K)] 
			[ParOldGen: 10020K->9042K(12288K)] 
			19236K->9042K(23040K), 0.0036840 secs]
	2.394: [Full GC (Ergonomics) 
			[PSYoungGen: 9216K->0K(10752K)] 
			[ParOldGen: 9042K->8064K(12288K)] 
			18258K->8064K(23040K), 0.0032855 secs]
	2.611: [Full GC (Ergonomics) 
			[PSYoungGen: 9216K->0K(10752K)] 
			[ParOldGen: 8064K->7085K(12288K)] 
			17280K->7085K(23040K), 0.0031675 secs]
	2.817: [Full GC (Ergonomics) 
			[PSYoungGen: 9216K->0K(10752K)] 
			[ParOldGen: 7085K->6107K(12288K)] 
			16301K->6107K(23040K), 0.0030652 secs]




At first glance it may seem that premature promotion is not the issue here. Indeed, the occupancy of the old generation seems to be decreasing on each cycle. However, if few or no objects were promoted, we would not be seeing a lot of full garbage collections.

乍一看似乎过早促销不是问题。事实上,老一辈的入住率似乎减少在每个周期。然而,如果很少或根本没有对象被提升,我们将不会看到很多垃圾收集。


There is a simple explanation for this GC behavior: while many objects are being promoted to the old generation, some existing objects are collected. This gives the impression that the old generation usage is decreasing, while in fact, there are objects that are constantly being promoted, triggering full GC.

有一个简单的解释GC行为:虽然许多对象被提升为旧的一代,收集一些现有的对象.这给了旧一代使用减少的印象,而事实上,有对象,不断提升,引发完整GC。


### 解决方案


In a nutshell, to fix this problem, we would need to make the buffered data fit into the young generation. There are two simple approaches for doing this. The first is to increase the young generation size by using -Xmx64m -XX:NewSize=32m parameters at JVM startup. Running the application with this change in configuration will make Full GC events much less frequent, while barely affecting the duration of minor collections:

简而言之,为了解决这个问题,我们需要使缓冲数据符合年轻一代。有两个简单的方法来做这个.第一是增加年轻一代的大小通过-Xmx64m - xx:NewSize = 32 m参数在JVM启动时.


	2.251: [GC (Allocation Failure) 
			[PSYoungGen: 28672K->3872K(28672K)] 
			37126K->12358K(61440K), 0.0008543 secs]
	2.776: [GC (Allocation Failure) 
			[PSYoungGen: 28448K->4096K(28672K)] 
			36934K->16974K(61440K), 0.0033022 secs]




Another approach in this case would be to simply decrease the batch size, which would also give a similar result. Picking the right solution heavily depends on what is really happening in the application. In some cases, business logic does not permit decreasing batch size. In this case, increasing available memory or redistributing in favor of the young generation might be possible.

另一种方法在这种情况下,就是简单地减少批量大小,这也会给一个类似的结果.选择正确的解决方案很大程度上取决于应用程序中真正发生了什么。在某些情况下,业务逻辑不允许减少批量大小。在这种情况下,增加可用内存或重新分配支持年轻一代的可能。


If neither is a viable option, then perhaps data structures can be optimized to consume less memory. But the general goal in this case remains the same: make transient data fit into the young generation.

如果没有一个可行的选择,那么也许可以优化消耗更少的内存数据结构。但在这种情况下,总体目标是相同的:使瞬态数据符合年轻一代。


## Weak, Soft and Phantom References

## Phantom薄弱、软性和参考文献


Another class of issues affecting GC is linked to the use of non-strong references in the application. While this may help to avoid an unwanted OutOfMemoryError in many cases, heavy usage of such references may significantly impact the way garbage collection affects the performance of your application.

另一类问题影响GC与non-strong引用在应用程序的使用。虽然这可能有助于避免在许多情况下,一个不受欢迎的OutOfMemoryError,应当参照使用重型garbage切尔诺贝利影响传媒may收藏有关业绩的你的执行。


## 为什么要关心


When using weak references, you should be aware of the way the weak references are garbage-collected. Whenever GC discovers that an object is weakly reachable, that is, the last remaining reference to the object is a weak reference, it is put onto the corresponding ReferenceQueue, and becomes eligible for finalization. One may then poll this reference queue and perform the associated cleanup activities. A typical example for such cleanup would be the removal of the now missing key from the cache.

使用弱引用时,您应该知道的弱引用的垃圾收集。每当GC发现弱可及对象,,的最后一个对象的引用是一个弱引用,放在相应的ReferenceQueue,成为合格的终结.一个可能会调查这个引用队列并执行相关的清理活动。等清理一个典型的例子是现在失踪的删除键从缓存中。


The trick here is that at this point you can still create new strong references to the object, so before it can be, at last, finalized and reclaimed, GC has to check again that it really is okay to do this. Thus, the weakly referenced objects are not reclaimed for an extra GC cycle.

这里的诀窍是,此时,您仍然可以创建新的强引用对象,所以它可以之前,最后,完成和回收,GC再次检查,真的很好。因此,弱引用的对象不是一个GC循环再生。


Weak references are actually a lot more common than you might think. Many caching solutions build the implementations using weak referencing, so even if you are not directly creating any in your code, there is a strong chance your application is still using weakly referenced objects in large quantities.

弱引用实际上是很多比你想象的更常见。许多缓存解决方案构建实现使用弱引用,所以即使你不直接在代码中创建任何,应用程序有一种强烈的机会仍在大量使用弱引用的对象。


When using soft references, you should bear in mind that soft references are collected much less eagerly than the weak ones. The exact point at which it happens is not specified and depends on the implementation of the JVM. Typically the collection of soft references happens only as a last ditch effort before running out of memory. What it implies is that you might find yourself in situations where you face either more frequent or longer full GC pauses than expected, since there are more objects resting in the old generation.

当使用软引用,你应该记住,软引用比弱的收集更热切.它发生的精确的点没有指定,取决于JVM的实现.通常只会发生软引用的集合作为最后放弃努力之前耗尽内存.这意味着,你可能会发现自己在你面临的情况下更频繁或完整GC暂停时间比预期更长的时间,因为有更多的对象在旧的一代。


When using phantom references, you have to literally do manual memory management in regards of flagging such references eligible for garbage collection. It is dangerous, as a superficial glance at the javadoc may lead one to believe they are the completely safe to use:

使用幻影引用时,你必须做手动内存管理关于资格垃圾收集的萎靡不振的引用。是很危险的,表面看一眼javadoc可能让人相信它们是完全安全的使用:


In order to ensure that a reclaimable object remains so, the referent of a phantom reference may not be retrieved: The get method of a phantom reference always returns null.

为了确保可教化的对象仍是如此,幻影的referent参考不得检索:幽灵的get方法参考总是返回null。


Surprisingly, many developers skip the very next paragraph in the same javadoc (emphasis added):

令人惊讶的是,许多开发人员跳过第二段在同一javadoc(重点):


Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.

与柔软和弱引用,幻影引用不会自动被垃圾收集器清除队列.虚引用一个对象,可以通过将继续,直到所有这些引用被清除或自己成为遥不可及的。


That is right, we have to manually clear() up phantom references or risk facing a situation where the JVM starts dying with an OutOfMemoryError. The reason why the Phantom references are there in the first place is that this is the only way to find out when an object has actually become unreachable via the usual means. Unlike with soft or weak references, you cannot resurrect a phantom-reachable object.
 

这是正确的,我们必须手动清楚()幻影引用或面临情况JVM开始死亡风险OutOfMemoryError.虚引用的原因是首先是,这是唯一的方法来找出当一个对象通过通常意味着实际上已经成为遥不可及的.与软或弱引用,不能复活phantom-reachable对象。


## 示例


Let us take a look at another demo application that allocates a lot of objects, which are successfully reclaimed during minor garbage collections. Bearing in mind the trick of altering the tenuring threshold from the previous section on promotion rate, we could run this application with -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1 and see this in GC logs:

让我们看看另一个演示应用程序分配的对象,这是小垃圾收集期间成功再生.牢记的技巧改变任期阈值前一节的推广速度,我们可以运行这个应用程序与-Xmx24m - xx:NewSize = 16 m - xx:MaxTenuringThreshold = 1,看看这个GC日志:


	2.330: [GC (Allocation Failure)  20933K->8229K(22528K), 0.0033848 secs]
	2.335: [GC (Allocation Failure)  20517K->7813K(22528K), 0.0022426 secs]
	2.339: [GC (Allocation Failure)  20101K->7429K(22528K), 0.0010920 secs]
	2.341: [GC (Allocation Failure)  19717K->9157K(22528K), 0.0056285 secs]
	2.348: [GC (Allocation Failure)  21445K->8997K(22528K), 0.0041313 secs]
	2.354: [GC (Allocation Failure)  21285K->8581K(22528K), 0.0033737 secs]
	2.359: [GC (Allocation Failure)  20869K->8197K(22528K), 0.0023407 secs]
	2.362: [GC (Allocation Failure)  20485K->7845K(22528K), 0.0011553 secs]
	2.365: [GC (Allocation Failure)  20133K->9501K(22528K), 0.0060705 secs]
	2.371: [Full GC (Ergonomics)  9501K->2987K(22528K), 0.0171452 secs]




Full collections are quite rare in this case. However, if the application also starts creating weak references (-Dweak.refs=true) to these created objects, the situation may change drastically. There may be many reasons to do this, starting from using the object as keys in a weak hash map and ending with allocation profiling. In any case, making use of weak references here may lead to this:

在这种情况下完整的集合是相当罕见的。然而,如果应用程序也开始创建弱引用(-Dweak.refs = true)来创建这些对象,这种情况可能会改变显著.可能有很多原因,从使用弱散列映射中的对象作为键,最后分配分析。在任何情况下,利用弱引用在这里可能会导致:


	2.059: [Full GC (Ergonomics)  20365K->19611K(22528K), 0.0654090 secs]
	2.125: [Full GC (Ergonomics)  20365K->19711K(22528K), 0.0707499 secs]
	2.196: [Full GC (Ergonomics)  20365K->19798K(22528K), 0.0717052 secs]
	2.268: [Full GC (Ergonomics)  20365K->19873K(22528K), 0.0686290 secs]
	2.337: [Full GC (Ergonomics)  20365K->19939K(22528K), 0.0702009 secs]
	2.407: [Full GC (Ergonomics)  20365K->19995K(22528K), 0.0694095 secs]




As we can see, there are now many full collections, and the duration of the collections is an order of magnitude longer! Another case of premature promotion, but this time a tad trickier. The root cause, of course, lies with the weak references. Before we added them, the objects created by the application were dying just before being promoted to the old generation. But with the addition, they are now sticking around for an extra GC round so that the appropriate cleanup can be done on them. Like before, a simple solution would be to increase the size of the young generation by specifying -Xmx64m -XX:NewSize=32m:

我们可以看到,现在有完整的集合,集合时间是一个数量级了!过早推广的另一个例子,但这次有点棘手.当然,问题的根源在于弱引用。我们添加了他们之前,应用程序创建的对象的死前被提升为旧的一代.但是,他们现在粘在一个额外的GC轮,这样可以做适当的清理。像以前一样,一个简单的解决方案是增加年轻一代的大小通过指定-Xmx64m - xx:NewSize = 32 m:


	2.328: [GC (Allocation Failure)  38940K->13596K(61440K), 0.0012818 secs]
	2.332: [GC (Allocation Failure)  38172K->14812K(61440K), 0.0060333 secs]
	2.341: [GC (Allocation Failure)  39388K->13948K(61440K), 0.0029427 secs]
	2.347: [GC (Allocation Failure)  38524K->15228K(61440K), 0.0101199 secs]
	2.361: [GC (Allocation Failure)  39804K->14428K(61440K), 0.0040940 secs]
	2.368: [GC (Allocation Failure)  39004K->13532K(61440K), 0.0012451 secs]




The objects are now once again reclaimed during minor garbage collection.

回收的对象现在再次在次要的垃圾收集。


The situation is even worse when soft references are used as seen in the next demo application. The softly-reachable objects are not reclaimed until the application risks getting an OutOfMemoryError. Replacing weak references with soft references in the demo application immediately surfaces many more Full GC events:

情况更糟的是当使用软引用见下一个演示应用程序。软可及的对象不回收,直到应用程序可能会获得一个OutOfMemoryError.取代弱引用和软引用演示应用程序立即表面更多的完整GC事件:


	2.162: [Full GC (Ergonomics)  31561K->12865K(61440K), 0.0181392 secs]
	2.184: [GC (Allocation Failure)  37441K->17585K(61440K), 0.0024479 secs]
	2.189: [GC (Allocation Failure)  42161K->27033K(61440K), 0.0061485 secs]
	2.195: [Full GC (Ergonomics)  27033K->14385K(61440K), 0.0228773 secs]
	2.221: [GC (Allocation Failure)  38961K->20633K(61440K), 0.0030729 secs]
	2.227: [GC (Allocation Failure)  45209K->31609K(61440K), 0.0069772 secs]
	2.234: [Full GC (Ergonomics)  31609K->15905K(61440K), 0.0257689 secs]




And the king here is the phantom reference as seen in the third demo application. Running the demo with the same sets of parameters as before would give us results that are pretty similar as the results in the case with weak references. The number of full GC pauses would, in fact, be much smaller because of the difference in the finalization described in the beginning of this section.

这里的国王是幻影参考见第三个演示应用程序.运行演示与同一组参数之前会给我们的结果是非常相似的结果与弱引用。完整的GC暂停的数量,实际上,小得多,因为不同的终结中描述这一节的开始。


However, adding one flag that disables phantom reference clearing (-Dno.ref.clearing=true) would quickly give us this:

然而,添加一个标志,禁用幻影参考结算(-Dno.ref.clearing = true)将很快给我们:


	4.180: [Full GC (Ergonomics)  57343K->57087K(61440K), 0.0879851 secs]
	4.269: [Full GC (Ergonomics)  57089K->57088K(61440K), 0.0973912 secs]
	4.366: [Full GC (Ergonomics)  57091K->57089K(61440K), 0.0948099 secs]




Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

异常的线程“主要”. lang。OutOfMemoryError:Java堆空间


One must exercise extreme caution when using phantom references and always clear up the phantom reachable objects in a timely manner. Failing to do so will likely end up with an OutOfMemoryError. And trust us when we say that it is quite easy to fail at this: one unexpected exception in the thread that processes the reference queue, and you will have a dead application at your hand.

一个使用幻时必须非常谨慎引用,总是及时清理幽灵的可访问的对象。未能这样做,可能会得到一个OutOfMemoryError.和相信我们,当我们说很容易失败在这个:一个意想不到的异常处理引用队列的线程,你将有一个死去的应用程序在你的手。


### Could my JVMs be Affected?

### 我的jvm会受影响吗?


As a general recommendation, consider enabling the -XX:+PrintReferenceGC JVM option to see the impact that different references have on garbage collection. If we add this to the application from the WeakReference example, we will see this:

一般建议,考虑启用- xx:+ PrintReferenceGC JVM选项看到引用对垃圾收集的影响不同.如果我们添加这个WeakReference引用的应用程序示例中,我们将会看到:


	2.173: [Full GC (Ergonomics) 
			2.234: [SoftReference, 0 refs, 0.0000151 secs]
			2.234: [WeakReference, 2648 refs, 0.0001714 secs]
			2.234: [FinalReference, 1 refs, 0.0000037 secs]
			2.234: [PhantomReference, 0 refs, 0 refs, 0.0000039 secs]
			2.234: [JNI Weak Reference, 0.0000027 secs]
				[PSYoungGen: 9216K->8676K(10752K)] 
				[ParOldGen: 12115K->12115K(12288K)] 
				21331K->20792K(23040K), 
			[Metaspace: 3725K->3725K(1056768K)], 
			0.0766685 secs] 
		[Times: user=0.49 sys=0.01, real=0.08 secs] 
	2.250: [Full GC (Ergonomics) 
			2.307: [SoftReference, 0 refs, 0.0000173 secs]
			2.307: [WeakReference, 2298 refs, 0.0001535 secs]
			2.307: [FinalReference, 3 refs, 0.0000043 secs]
			2.307: [PhantomReference, 0 refs, 0 refs, 0.0000042 secs]
			2.307: [JNI Weak Reference, 0.0000029 secs]
				[PSYoungGen: 9215K->8747K(10752K)] 
				[ParOldGen: 12115K->12115K(12288K)] 
				21331K->20863K(23040K), 
			[Metaspace: 3725K->3725K(1056768K)], 
			0.0734832 secs] 
		[Times: user=0.52 sys=0.01, real=0.07 secs] 
	2.323: [Full GC (Ergonomics) 
			2.383: [SoftReference, 0 refs, 0.0000161 secs]
			2.383: [WeakReference, 1981 refs, 0.0001292 secs]
			2.383: [FinalReference, 16 refs, 0.0000049 secs]
			2.383: [PhantomReference, 0 refs, 0 refs, 0.0000040 secs]
			2.383: [JNI Weak Reference, 0.0000027 secs]
				[PSYoungGen: 9216K->8809K(10752K)] 
				[ParOldGen: 12115K->12115K(12288K)] 
				21331K->20925K(23040K), 
			[Metaspace: 3725K->3725K(1056768K)], 
			0.0738414 secs] 
		[Times: user=0.52 sys=0.01, real=0.08 secs]




As always, this information should only be analyzed when you have identified that GC is having impact to either the throughput or latency of your application. In such case you may want to check these sections of the logs. Normally, the number of references cleared during each GC cycle is quite low, in many cases exactly zero. If this is not the case, however, and the application is spending a significant period of time clearing references, or just a lot of them are being cleared, then further investigation is required.

像往常一样,这些信息只能当你已经确定了GC分析有影响应用程序的吞吐量和延迟.在这样的情况下,您可能希望查看这些部分的日志。通常情况下,引用了在每一个GC循环的数量很低,在许多情况下完全零。如果不是这种情况,然而,支出过很长一段时间才清理应用程序引用,或者只是被清除的很多人,然后还需要进一步调查。


### 解决方案


When you have verified the application actually is suffering from the mis-, ab- or overuse of either weak, soft or phantom references, the solution often involves changing the application’s intrinsic logic. This is very application specific and generic guidelines are thus hard to offer. However, some generic solutions to bear in mind are:

当你有验证应用程序实际上是遭受mis - ab -或过度使用弱,软或幻影引用,解决方案通常需要改变应用程序的内在逻辑。这是非常特定于应用程序的,因此很难提供通用的指导方针。然而,要记住一些通用的解决方案是:


- Weak references – if the problem is triggered by increased consumption of a specific memory pool, an increase in the corresponding pool (and possibly the total heap along with it) can help you out. As seen in the example section, increasing the total heap and young generation sizes alleviated the pain.
- Phantom references – make sure you are actually clearing the references. It is easy to dismiss certain corner cases and have the clearing thread to not being able to keep up with the pace the queue is filled or to stop clearing the queue altogether, putting a lot of pressure to GC and creating a risk of ending up with an OutOfMemoryError.
- Soft references – when soft references are identified as the source of the problem, the only real way to alleviate the pressure is to change the application’s intrinsic logic.

- 弱引用,如果问题是引发了消费的增加一个特定的内存池,增加相应的池(也可能是总堆随之)可以帮助你。正如在此示例中所看到的部分,增加总堆和年轻一代的大小来减轻疼痛。
- 幻影引用——确保你实际上是清除引用。很容易忽略某些角落病例和清理线程无法跟上步伐的队列或完全停止清除队列,施加很多压力GC和创建一个可能到最后只能依靠一个OutOfMemoryError。
- 软引用,当软引用被认为是问题的根源,唯一真正的缓解压力的方法是改变应用程序的内在逻辑。



## Other Examples

## 其他的例子


Previous chapters covered the most common problems related to poorly behaving GC. Unfortunately, there is a long list of more specific cases, where you cannot apply the knowledge from previous chapters. This section describes a few of the more unusual problems that you may face.

之前的章节涵盖最常见的问题表现不佳的GC。不幸的是,有一长串的更具体的情况,你不能应用知识从先前的章节。本节描述的一些更不寻常的你可能会面临的问题。


### RMI & GC

### RMI和GC


When your application is publishing or consuming services over RMI, the JVM periodically launches full GC to make sure that locally unused objects are also not taking up space on the other end. Bear in mind that even if you are not explicitly publishing anything over RMI in your code, third party libraries or utilities can still open RMI endpoints. One such common culprit is for example JMX, which, if attached to remotely, will use RMI underneath to publish the data.

当你的应用程序发布在RMI或消费服务,定期JVM启动完整GC确保本地未使用的对象也不占用空间的另一端.记住,即使你没有明确发布任何在RMI在代码中,第三方库或工具仍然可以打开RMI端点.例如JMX,就是这样一个常见的罪魁祸首,如果连接到远程,将使用RMI在发布数据。


The problem is exposed by seemingly unnecessary and periodic full GC pauses. When you check the old generation consumption, there is often no pressure to the memory as there is plenty of free space in the old generation, but full GC is triggered, stopping the application threads.

暴露的问题看似不必要的GC暂停时间和周期。当你检查旧一代消费,通常是没有内存的压力,有足够的自由空间在旧的一代,但完整的GC被触发,停止应用程序线程。


This behavior of removing remote references via System.gc() is triggered by the sun.rmi.transport.ObjectTable class requesting garbage collection to be run periodically as specified in the sun.misc.GC.requestLatency() method.

这种行为将通过远程引用system . gc()是sun.rmi.transport引发的。ObjectTable类请求垃圾收集运行定期sun.misc中指定.GC.requestLatency()方法。


For many applications, this is not necessary or outright harmful. To disable such periodic GC runs, you can set up the following for your JVM startup scripts:

对于许多应用程序,这是没有必要的或完全有害的。禁用这种周期性GC运行时,您可以设置为您的JVM启动脚本如下:


	java -Dsun.rmi.dgc.server.gcInterval=9223372036854775807L 
		-Dsun.rmi.dgc.client.gcInterval=9223372036854775807L 
		com.yourcompany.YourApplication




This sets the period after which System.gc() is run to Long.MAX_VALUE; for all practical matters, this equals eternity.

这System.gc which之后至sets)is赛跑长期(。MAX_VALUE;所有自由,equals实用事项。


An alternative solution for the problem would to disable explicit calls to System.gc() by specifying -XX:+DisableExplicitGC in the JVM startup parameters. We do not however recommend this solution as it can have other side effects.

另一种解决方案的问题会禁用显式地调用system . gc()通过指定- xx:+ DisableExplicitGC JVM启动参数.然而我们不推荐这种解决方案,因为它有其他的副作用。


### JVMTI tagging & GC

### JVMTI标签& GC


Whenever the application is run alongside with a Java Agent (-javaagent), there is a chance that the agent can tag the objects in the heap using JVMTI tagging. Agents can use tagging for various reasons that are not in the scope of this handbook, but there is a GC-related performance issue that can start affecting the latency and throughput of your application if tagging is applied to a large subset of objects inside the heap.

Whenever the application is run alongside with a Java Agent (- javaagent), there is a chance that the Agent can tag the objects in the heap using JVMTI tagging.代理可以使用标签由于种种原因不了本手册的范围,但有一个GC-related性能问题,可以影响应用程序的延迟和吞吐量如果标签应用于大型子集内的对象堆。


The problem is hidden in the native code where JvmtiTagMap::do_weak_oops iterates over all the tags during each garbage collection event and performs a number of not-so-cheap operations for all of them. To make things worse, this operation is performed sequentially and is not parallelized.

问题是隐藏在本机代码JvmtiTagMap::do_weak_oops遍历所有的标签在每个垃圾收集事件和执行一系列每操作。更糟的是,这个操作是按顺序执行的,而不是并行。


With a large number of tags, this implies that a large part of the GC process is now carried out in a single thread and all the benefits of parallelism disappear, potentially increasing the duration of GC pauses by an order of magnitude.

与大量的标签,这意味着现在GC过程的很大一部分是在一个线程并行消失所带来的好处,可能增加GC暂停的时间由一个数量级。


To check whether or not a particular agent can be the reason for extended GC pauses, you would need to turn on the diagnostic option of –XX:+TraceJVMTIObjectTagging. Enabling the trace will allow you to get an estimate of how much native memory the tag map consumes and how much time the heap walks take.

检查是否一个特定的代理可以延长GC暂停的原因,你需要打开的诊断选项- xx:+ TraceJVMTIObjectTagging.启用跟踪会让你得到一个估计的本机内存多少标签映射消耗和堆走多少时间。


If you are not the author of the agent yourself, fixing the problem is often out of your reach. Apart from contacting the vendor of a particular agent you cannot do much. In case you do end up in a situation like this, recommend that the vendor clean up the tags that are no longer needed.

如果你没有代理的作者自己,解决问题往往是遥不可及的。除了联系供应商特定的代理你不能做太多.如果你在这种情况下,建议供应商清除不再需要的标签。


### Humongous Allocations

### 巨大无比的分配


Whenever your application is using the G1 garbage collection algorithm, a phenomenon called humongous allocations can impact your application performance in regards of GC. To recap, humongous allocations are allocations that are larger than 50% of the region size in G1.

当您的应用程序使用G1垃圾收集算法,这种现象称为巨大无比的分配会影响应用程序性能的GC。回顾一下,巨大无比的分配是分配在G1大于50%的区域大小。


Having frequent humongous allocations can trigger GC performance issues, considering the way that G1 handles such allocations:

频繁的巨大无比的分配可以触发GC性能问题,考虑到G1在处理这样的分配方法:


- If the regions contain humongous objects, space between the last humongous object in the region and the end of the region will be unused. If all the humongous objects are just a bit larger than a factor of the region size, this unused space can cause the heap to become fragmented.
- Collection of the humongous objects is not as optimized by the G1 as with regular objects. It was especially troublesome with early Java 8 releases – until Java 1.8u40 the reclamation of humongous regions was only done during full GC events. More recent releases of the Hotspot JVM free the humongous regions at the end of the marking cycle during the cleanup phase, so the impact of the issue has been reduced significantly for newer JVMs.

- 
- 



To check whether or not your application is allocating objects in humongous regions, the first step would be to turn on GC logs similar to the following:

检查应用程序是否分配对象在巨大无比的地区,第一步是打开GC日志类似如下:


	java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 
		-XX:+PrintReferenceGC -XX:+UseG1GC 
		-XX:+PrintAdaptiveSizePolicy -Xmx128m 
		MyClass




Now, when you check the logs and discover sections like these:

现在,当您检查日志,发现这样的部分:


	 0.106: [G1Ergonomics (Concurrent Cycles) 
			request concurrent cycle initiation, 
			reason: occupancy higher than threshold, 
			occupancy: 60817408 bytes, 
			allocation request: 1048592 bytes, 
			threshold: 60397965 bytes (45.00 %), 
			source: concurrent humongous allocation]
	 0.106: [G1Ergonomics (Concurrent Cycles) 
			request concurrent cycle initiation, 
			reason: requested by GC cause, 
			GC cause: G1 Humongous Allocation]
	 0.106: [G1Ergonomics (Concurrent Cycles) 
			initiate concurrent cycle, 
			reason: concurrent cycle initiation requested]
	 0.106: [GC pause (G1 Humongous Allocation) 
			(young) (initial-mark) 
			0.106: [G1Ergonomics (CSet Construction) 
				start choosing CSet, 
				_pending_cards: 0, 
				predicted base 
				time: 10.00 ms, 
				remaining time: 190.00 ms, 
				target pause time: 200.00 ms]




you have evidence that the application is indeed allocating humongous objects. The evidence is visible in the cause for a GC pause being identified as G1 Humongous Allocation and in the “allocation request: 1048592 bytes” section, where we can see that the application is trying to allocate an object with the size of 1,048,592 bytes, which is 16 bytes larger than the 50% of the 2 MB size of the humongous region specified for the JVM.

你有确实证据表明应用程序分配巨大无比的对象.证据是可见的GC暂停的原因被确定为G1巨大无比的分配和在“分配请求:1048592字节”部分,在那里我们可以看到应用程序试图分配1048592字节大小的一个对象,这是16字节大于50%的2 MB大小的巨型地区指定JVM。


The first solution for humongous allocation is to change the region size so that (most) of the allocations would not exceed the 50% limit triggering allocations in the humongous regions. The region size is calculated by the JVM during startup based on the size of the heap. You can override the size by specifying -XX:G1HeapRegionSize=XX in the startup script. The specified region size must be between 1 and 32 megabytes and has to be a power of two.

巨大无比的分配第一个解决方案是修改区域大小,以便(大多数)的分配将不超过50%的限制引发极大的地区分配.计算区域大小由JVM启动期间基于堆的大小。你可以覆盖大小通过指定- XX:G1HeapRegionSize = XX在启动脚本.指定的区域大小必须介于1和32字节,必须是2的幂。


This solution can have side effects – increasing the region size reduces the number of regions available so you need to be careful and run extra set of tests to see whether or not you actually improved the throughput or latency of the application.

这个解决方案可以有副作用,增加区域大小减少可用区域的数量你需要小心和运行额外设置的测试是否你真的 提高了应用程序的吞吐量和延迟。


A more time-consuming but potentially better solution would be to understand whether or not the application can limit the size of allocations. The best tools for the job in this case are profilers. They can give you information about humongous objects by showing their allocation sources with stack traces.

更耗费时间,但潜在的更好的解决方案是了解应用程序是否可以限制分配的大小。最好的工具工作在这种情况下是分析器.他们可以给你巨大无比的对象通过展示它们的分配信息来源和堆栈跟踪。



### 结论


With the enormous number of possible applications that one may run on the JVM, coupled with the hundreds of JVM configuration parameters that one may tweak for GC, there are astoundingly many ways in which the GC may impact your application’s performance.

因为有很多各种各样的程序在JVM上运行, 加上有几百个 JVM 参数, 其中很多会影响到 GC, 所以有很多方法可以调优程序的GC性能。


Therefore, there is no real silver bullet approach to tuning the JVM to match the performance goals you have to fulfill. What we have tried to do here is walk you through some common (and not so common) examples to give you a general idea of how problems like these can be approached. Coupled with the tooling overview and with a solid understanding of how the GC works, you have all the chances of successfully tuning garbage collection to boost the performance of your application.


因此,没有真正的银弹, 能满足所有的JVM性能调优目标. 我们能做的只是介绍一些常见的(和不常见的)示例, 让你在碰到类似的问题时知道是怎么回事. 通过对工具的熟练使用, 以及对GC工作原理的深入理解, 你应该能成功地调优垃圾收集, 来有效提高应用程序的性能。


原文链接:  [GC Tuning: In Practice](https://plumbr.eu/handbook/gc-tuning-in-practice)





