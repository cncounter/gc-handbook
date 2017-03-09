# 7. GC 调优(实战篇)


This chapter covers several typical performance problems that one may encounter with garbage collection. The examples given here are derived from real applications, but are simplified for the sake of clarity.

本章介绍会导致GC性能问题的典型情况。相关示例都来源于生产环境, 为了演示的需要,做了一定简化。


## 高分配速率(High Allocation Rate)


Allocation rate is a term used when communicating the amount of memory allocated per time unit. Often it is expressed in MB/sec, but you can use PB/year if you feel like it. So that is all there is – no magic, just the amount of memory you allocate in your Java code measured over a period of time.


分配速率(`Allocation rate`)用来表示单位时间内分配的内存总量。单位通常是 `MB/sec`, 也可以使用 `PB/year` 等。这很容易理解, 通过一段时间来衡量Java代码创建的内存总量。




An excessively high allocation rate can mean trouble for your application’s performance. When running on a JVM, the problem will be revealed by garbage collection posing a large overhead.

过高的分配速率会严重影响程序性能。在JVM中, 这个问题会导致大量的GC开销。


### 如何衡量分配速率?


One way to measure the allocation rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses similar to the following:

通过指定JVM参数: `-XX:+PrintGCDetails -XX:+PrintGCTimeStamps` ,记录GC日志, 可以用来测量分配速率. GC日志类似这样:


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

根据GC日志就可以计算出分配速率。 计算 `上一次垃圾收集之后`,与`下一次GC开始之前`的年轻代使用量的差值。 比如上面的日志中, 可以获取以下信息:


- At 291 ms after the JVM was launched, 33,280 K of objects were created. The first minor GC event cleaned the young generation, after which there were 5,088 K of objects in the young generation left.
- At 446 ms after launch, the young generation occupancy had grown to 38,368 K, triggering the next GC, which managed to reduce the young generation occupancy to 5,120 K.
- At 829 ms after the launch, the size of the young generation was 71,680 K and the GC reduced it again to 5,120 K.

<br/>

- JVM启动之后 `291ms`, 共创建了 `33,280 KB` 的对象。 第一次 Minor GC(小型GC) 完成后, 年轻代中还有 `5,088 KB` 的对象存活。
- 在启动之后 `446 ms`, 年轻代的使用量增加到 `38,368 KB`, 触发第二次GC, 完成后年轻代的使用量减少到 `5,120 KB`。
- 在启动之后 `829 ms`, 年轻代的使用量为 `71,680 KB`, GC后变为 `5,120 KB`。



This data can then be expressed in the following table calculating the allocation rate as deltas of the young occupancy:

可以通过年轻代的使用量来计算分配速率, 如下表所示:


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

通过这些信息可以知道, 在测量期间, 该程序的内存分配速率为 `161 MB/sec`。



### 为何要关心分配速率?


After measuring the allocation rate we can understand how the changes in allocation rate affect application throughput by increasing or reducing the frequency of GC pauses. First and foremost, you should notice that only minor GC pauses cleaning the young generation are affected. Neither the frequency nor duration of the GC pauses cleaning the old generation are directly impacted by the allocation rate, but instead by the promotion rate, a term that we will cover separately in the next section.


计算出分配速率, 就会知道分配速率如何影响吞吐量: 分配速率的变化,会增加或降低GC暂停的频率。 请记住, 只有年轻代的 minor GC 受分配速率的影响。 而老年代GC的频率和持续时间不受分配速率的直接影响, 而是受**提升速率**(`promotion rate`)的影响,我们在下一节中介绍。


Knowing that we can focus only on Minor GC pauses, we should next look into the different memory pools inside the young generation. As the allocation takes place in Eden, we can immediately look into how sizing Eden can impact the allocation rate. So we can hypothesize that increasing the size of Eden will reduce the frequency of minor GC pauses and thus allow the application to sustain faster allocation rates.

这样我们就只关心 Minor GC 暂停, 接下来看年轻代的这几个内存池。因为对象分配在 Eden 区, 所以我们来审查 Eden 区的大小和分配速率的关系.  看看增加 Eden 区的容量能不能减少 Minor GC 暂停次数, 从而使程序能够维持更快的分配速率。


And indeed, when running the same application with different Eden sizes using -XX:NewSize -XX:MaxNewSize & -XX:SurvivorRatio parameters, we can see a two-fold difference in allocation rates.

事实上, 通过参数 `-XX:NewSize`、 `-XX:MaxNewSize` 以及 `-XX:SurvivorRatio` 设置不同的 Eden 空间来运行同一程序时, 可以看到:


- Re-running with 100 M of Eden reduces the allocation rate to below 100 MB/sec.
- Increasing Eden size to 1 GB increases the allocation rate to just below 200 MB/sec.

<br/>

- Eden 空间为 `100 MB` 时, 分配速率低于 `100 MB/秒`。
- 将 Eden 区增大为 `1 GB`, 分配速率也随之增长,大约等于 `200 MB/秒`。


If you are still wondering how this can be true – if you stop your application threads for GC less frequently you can do more useful work. More useful work also happens to create more objects, thus supporting the increased allocation rate.


为什么会这样? —— 因为减少GC暂停,就等价于减少任务线程的停顿，就可以做更多工作, 也就创建了更多对象, 所以对同一应用程序, 分配速率越高越好。



Now, before you jump to the conclusion that “bigger Eden is better”, you should notice that the allocation rate might and probably does not directly correlate with the actual throughput of your application. It is a technical measurement contributing to throughput. The allocation rate can and will have an impact on how frequently your minor GC pauses stop application threads, but to see the overall impact, you also need to take into account major GC pauses and measure throughput not in MB/sec but in the business operations your application provides.

在得出 “Eden去越大越好” 这种结论前, 我们注意到, 分配速率可能会,也可能不会直接影响程序的实际吞吐量。 吞吐量和分配速率有一定关系, 分配速率会影响 minor GC 暂停, 但对总体吞吐量的影响, 还要考虑 Major GC(大型GC)暂停, 而且吞吐量的单位不是 `MB/秒`， 而是应用所处理的业务量。



### 示例


Meet the demo application. Suppose that it works with an external sensor that provides a number. The application continuously updates the value of the sensor in a dedicated thread (to a random value, in this example), and from other threads sometimes uses the most recent value to do something meaningful with it in the processSensorValue() method:

为了演示需要。假设系统连接了一个外部的数字传感器。应用通过专有线程, 不断地获取传感器的值,(此处使用随机数模拟), 其他线程会调用 `processSensorValue()` 方法, 传入传感器的值来执行某些操作, :


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

如同类名所示, 这个Demo是模拟 boxing 的。为了判断 null 值, 使用的是包装类型 `Double`。 基于传感器的最新值进行计算, 但从传感器取值是一个耗时操作, 所以采用了异步方式： 一个线程不断获取新值, 计算线程则直接使用暂存的最新值, 从而避免同步获取。


The demo application is impacted by the GC not keeping up with the allocation rate. The ways to verify and solve the issue are given in the next sections.

Demo 程序在运行的过程中, 由于分配速率太大而受GC拖累。下一节将确认问题, 并给出解决办法。


### Could my JVMs be Affected?

### 对JVM会造成什么影响?



First and foremost, you should only be worried if the throughput of your application starts to decrease. As the application is creating too much objects that are almost immediately discarded, the frequency of minor GC pauses surges. Under enough of a load this can result in GC having a significant impact on throughput.

首先，我们应该检查程序的吞吐量是否降低。如果创建了过多的临时对象, 小型GC的次数就会增加。如果并发较大, 则GC很可能会严重影响吞吐量。


When you run into a situation like this, you would be facing a log file similar to the following short snippet extracted from the GC logs of the demo application introduced in the previous section. The application was launched as with the -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m command line arguments:

遇到这种情况时, GC日志将会像下面这样，当然这是从上一节的[示例程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java) 产生的GC日志中提取的。 JVM启动参数为 `-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m`:


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

很显然 minor GC 的频率太高了。这说明创建了大量的对象。另外, 年轻代在 GC 之后的使用量又很低, 也没有 full GC 发生。 这种症状表明, GC对吞吐量有严重的影响。



### 解决方案


In some cases, reducing the impact of high allocation rates can be as easy as increasing the size of the young generation. Doing so will not reduce the allocation rate itself, but will result in less frequent collections. The benefit of the approach kicks in when there will be only a few survivors every time. As the duration of a minor GC pause is impacted by the number of surviving objects, they will not noticeably increase here.

在某些情况下,只要增加年轻代空间的大小, 即可降低分配速率过高的影响。增加年轻代空间并不会降低分配速率, 但是会减少GC的频率。如果每次GC后只有少量对象存活, minor GC 的暂停时间就不会明显增加。


The result is visible when we run the very same demo application with increased heap size and, with it, the young generation size, by using the -Xmx64m parameter:

运行 [示例程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java) , 增加堆内存大小,(同时也就增大了年轻代的大小), 使用的JVM参数为 `-Xmx64m`:


	2.808: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003748 secs]
	2.831: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0004538 secs]
	2.855: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003355 secs]
	2.879: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0005592 secs]




However, just throwing more memory at it is not always a viable solution. Equipped with the knowledge on allocation profilers from the previous chapter, we may find out where most of the garbage is produced. Specifically, in this case, 99% are Doubles that are created with the readSensor method. As a simple optimization, the object can be replaced with a primitive double, and the null can be replaced with Double.NaN. Since primitive values are not actually objects, no garbage is produced, and there is nothing to collect. Instead of allocating a new object on the heap, a field in an existing object is directly overwritten.

但只增加堆内存的大小,有时候并不能解决问题。通过前面学习的知识, 我们可以通过分配分析器找出大部分垃圾产生的位置。实际上在此示例中, 99%的对象是 `Double` 包装类, 在`readSensor` 方法中创建。可以进行简单的优化, 将创建的 `Double` 对象替换为原生类型 `double`, 而 null 值的判断, 可以使用  [Double.NaN](https://docs.oracle.com/javase/7/docs/api/java/lang/Double.html#NaN) 来进行。由于原生类型不算是对象, 也就不会产生垃圾, 不容易产生GC事件。优化之后, 不在堆内存中分配新对象, 而是直接覆盖一个属性域即可。


The simple change (diff) will, in the demo application, almost completely remove GC pauses. In some cases, the JVM may be clever enough to remove excessive allocations itself by using the escape analysis technique. To cut a long story short, the JIT compiler may in some cases prove that a created object never “escapes” the scope it is created in. In such cases, there is no actual need to allocate it on the heap and produce garbage this way, so the JIT compiler does just that: it eliminates the allocation. See this benchmark for an example.

对示例程序进行[简单的改造](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/FixedBoxing.java)( [diff](https://gist.github.com/gvsmirnov/0270f0f15f9498e3b655) ) 之后, GC暂停基本上完全消除。有时候 JVM 也会很智能, 使用 逃逸分析技术(escape analysis technique) 来避免过度分配。简单来说,JIT编译器可以分析得知, 方法创建的某些对象永远都不会“逃出”此方法的作用域。这时候就不需要在堆上分配这些对象, 也就不会产生垃圾, 所以JIT编译器所做的一种优化就是: 消除内存分配。请参考 [基准测试](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/jit/EscapeAnalysis.java) 。



## 过早提升(Premature Promotion)



Before explaining the concept of premature promotion, we should familiarize ourselves with the concept it builds upon – the promotion rate. The promotion rate is measured in the amount of data propagated from the young generation to the old generation per time unit. It is often measured in MB/sec, similarly to the allocation rate.

先介绍一个基本概念 —— **提升速率**(`premature promotion`)。提升速率用于衡量单位时间内从年轻代提升到老年代的数据量。一般使用 `MB/sec` 作为单位, 和分配速率类似。



Promoting long-lived objects from the young generation to the old is how JVM is expected to behave. Recalling the generation hypothesis we can now construct a situation where not only long-lived objects end up in the old generation. Such a situation, where objects with a short life expectancy are not collected in the young generation and get promoted to the old generation, is called premature promotion.


JVM会将长时间存活的对象从年轻代提升到老年代。根据分代假设, 可能存在一种情况, 老年代中不仅有存活时间长的对象,也可能有存活时间短的对象。这就是过早提升：存活时间还不长的对象被提升到了老年代。



Cleaning these short-lived objects now becomes a job for major GC, which is not designed for frequent runs and results in longer GC pauses. This significantly affects the throughput of the application.


 major GC 不是为这种频繁回收而设计的, 但现在 major GC 也要清理这些生命短暂的对象, 所以会导致GC暂停时间过长。这就严重影响了系统吞吐量。


### How to Measure Promotion Rate

### 如何测量提升速率


One of the ways you can measure the promotion rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses just like in the following snippet:


可以指定JVM参数 `-XX:+PrintGCDetails -XX:+PrintGCTimeStamps` , 通过GC日志来测量提升速率. JVM记录的GC停顿信息如下所示:


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

从上面的信息可以得知, GC前和GC之后,年轻代的使用量,以及堆内存使用的总量。这样就很可以算出, 老年代的使用量就是是两者的差值。GC日志中的信息可以表示为:


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
<tr>
<th><strong>(事件)</strong></th>
<th><strong>(耗时)</strong></th>
<th><strong>(年轻代减少)</strong></th>
<th><strong>(整个堆内存减少)</strong></th>
<th><strong>(提升量)</strong></th>
<th><strong>(提升速率)</strong></th>
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

根据这些信息, 就可以得出观察周期内的提升速率。平均提升速率为 `92 MB/秒`, 峰值为 `140.95 MB/秒`。


Notice that you can extract this information only from minor GC pauses. Full GC pauses do not expose the promotion rate as the change in the old generation usage in GC logs also includes objects cleaned by the major GC.

请注意, 只能根据 minor GC 计算这些信息. Full GC 暂停的日志,不能用于计算提升速率, 因为 major GC 会清理老年代中的一部分对象。



### 提升速率的意义


Similarly to the allocation rate, the main impact of the promotion rate is the change of frequency in GC pauses. But as opposed to the allocation rate that affects the frequency of minor GC events, the promotion rate affects the frequency of major GC events. Let me explain – the more stuff you promote to the old generation the faster you will fill it up. Filling the old generation faster means that the frequency of the GC events cleaning the old generation will increase.

和分配速率一样, 提升速率也会影响GC暂停的频率。但分配速率主要影响 minor GC, 而提升速率则影响的是 major GC 的频率。有大量的数据提升,自然很快就会把老年代填满。 老年代填充的越快, 则 major GC 事件的频率将会越快。



![](07_01_how-java-garbage-collection-works.png)




As we have shown in earlier chapters, full garbage collections typically require much more time, as they have to interact with many more objects, and perform additional complex activities such as defragmentation.

此前我们说过, full GC 通常需要更多的时间, 因为需要处理更多的对象, 还要额外执行碎片整理等复杂的过程。


### 示例


Let us look at a demo application suffering from premature promotion. This app obtains chunks of data, accumulates them, and, when a sufficient number is reached, processes the whole batch at once:

让我们看一个过早提升的示例。 这个程序创建/获取大量的对象/数据,并暂存到集合之中, 累积到一定数量之后,进行批处理:


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

此 Demo 程序受到过早提升的影响。在接下来的部分将进行确认并给出解决办法。


### Could my JVMs be Affected?

### 对JVM会造成什么影响?


In general, the symptoms of premature promotion can take any of the following forms:

一般来说,过早提升的症状表现为以下形式:


- The application goes through frequent full GC runs over a short period of time.
- The old generation consumption after each full GC is low, often under 10-20% of the total size of the old generation.
- Facing the promotion rate approaching the allocation rate.

<br/>

- 在很短的时间内频繁地执行 full GC。
- 每次 full GC 后老年代的使用率都很低, 在10-20%或以下。
- 提升速率约等于分配速率。



Showcasing this in a short and easy-to-understand demo application is a bit tricky, so we will cheat a little by making the objects tenure to the old generation a bit earlier than it happens by default. If we ran the demo with a specific set of GC parameters (-Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1), we would see this in the garbage collection logs:

要演示这种情况稍微有点麻烦, 所以我们使用点特殊手段, 让对象提升到老年代的年龄比默认情况小很多。指定GC参数 `-Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1`, 运行程序之后,可以看到下面的GC日志:


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

乍一看似乎不是过早提升的问题。事实上,在每次GC之后老年代的使用率似乎在减少。但反过来想, 要是没有对象提升或者提升率很小, 也就不会看到这么多的 Full GC。


There is a simple explanation for this GC behavior: while many objects are being promoted to the old generation, some existing objects are collected. This gives the impression that the old generation usage is decreasing, while in fact, there are objects that are constantly being promoted, triggering full GC.

简单解释一下这种GC行为: 有很多对象提升到老年代的同时, 也有很多老年代中的对象被回收了, 这就造成了老年代使用量减少的假象. 但事实是大量的对象被不断地提升到老年代, 触发 full GC。


### 解决方案


In a nutshell, to fix this problem, we would need to make the buffered data fit into the young generation. There are two simple approaches for doing this. The first is to increase the young generation size by using -Xmx64m -XX:NewSize=32m parameters at JVM startup. Running the application with this change in configuration will make Full GC events much less frequent, while barely affecting the duration of minor collections:

简单点, 要解决这类问题, 需要让年轻代存放得下暂存的数据。有两种简单的方法:

一是增加年轻代的大小, 设置JVM启动参数, 类似这样: ` -Xmx64m -XX:NewSize=32m`, 程序在执行时, Full GC 自然会减少很多, 只影响 minor GC的持续时间:



	2.251: [GC (Allocation Failure) 
			[PSYoungGen: 28672K->3872K(28672K)] 
			37126K->12358K(61440K), 0.0008543 secs]
	2.776: [GC (Allocation Failure) 
			[PSYoungGen: 28448K->4096K(28672K)] 
			36934K->16974K(61440K), 0.0033022 secs]




Another approach in this case would be to simply decrease the batch size, which would also give a similar result. Picking the right solution heavily depends on what is really happening in the application. In some cases, business logic does not permit decreasing batch size. In this case, increasing available memory or redistributing in favor of the young generation might be possible.

二是减少每批次处理的数量, 也能得到类似的结果. 至于哪种方案更好, 很大程度上取决于应用程序中真正执行的逻辑。在某些情况下, 业务逻辑不允许减少批处理的数量, 就只能增加可用内存,或者重新指定年轻代的大小。


If neither is a viable option, then perhaps data structures can be optimized to consume less memory. But the general goal in this case remains the same: make transient data fit into the young generation.

如果都不可行, 那就优化数据结构, 减少内存消耗。但总体目标依然是一致的: 让临时数据能够在年轻代存放得下。



## Weak, Soft 以及 Phantom 引用


Another class of issues affecting GC is linked to the use of non-strong references in the application. While this may help to avoid an unwanted OutOfMemoryError in many cases, heavy usage of such references may significantly impact the way garbage collection affects the performance of your application.

另一类影响GC的问题是程序中的 non-strong 引用。虽然这类引用在很多情况下可以避免出现 `OutOfMemoryError`,  但滥用也会对GC行为造成严重影响, 进而降低系统性能。


## 需要关注弱引用的原因



When using weak references, you should be aware of the way the weak references are garbage-collected. Whenever GC discovers that an object is weakly reachable, that is, the last remaining reference to the object is a weak reference, it is put onto the corresponding ReferenceQueue, and becomes eligible for finalization. One may then poll this reference queue and perform the associated cleanup activities. A typical example for such cleanup would be the removal of the now missing key from the cache.

首先, 你应该知道, `弱引用`(weak reference) 是可以被垃圾收集器强制回收的。当GC发现一个弱可达对象(weakly reachable),即指向该对象的引用只剩下弱引用时, 则会将其置入相应的ReferenceQueue 中, 成为可终结的对象. 之后可能会遍历这个 reference queue, 并执行相应的清理。典型的示例是清除缓存中不再引用的KEY。


The trick here is that at this point you can still create new strong references to the object, so before it can be, at last, finalized and reclaimed, GC has to check again that it really is okay to do this. Thus, the weakly referenced objects are not reclaimed for an extra GC cycle.

当然, 在这个时候, 我们还可以将该对象赋值给新的强引用, 在最后终结和回收前, GC会再次确认是否可以回收。因此, 弱引用对象的回收过程是横跨多个GC周期的。


Weak references are actually a lot more common than you might think. Many caching solutions build the implementations using weak referencing, so even if you are not directly creating any in your code, there is a strong chance your application is still using weakly referenced objects in large quantities.

弱引用实际上使用的很多。大部分缓存框架(caching solution)都是基于弱引用来实现的, 所以我们虽然没有直接编写弱引用代码, 但程序中依然会存在大量的弱引用对象。


When using soft references, you should bear in mind that soft references are collected much less eagerly than the weak ones. The exact point at which it happens is not specified and depends on the implementation of the JVM. Typically the collection of soft references happens only as a last ditch effort before running out of memory. What it implies is that you might find yourself in situations where you face either more frequent or longer full GC pauses than expected, since there are more objects resting in the old generation.

其次, `软引用`(soft reference) 比弱引用更难被垃圾收集器回收. 回收软引用没有确切的时间点, 由各个JVM自己决定. 一般只会在即将耗尽可用内存时, 才会回收软引用,作为最后手段。这意味着, 可能会面临更频繁的 full GC, 暂停时间也会比预期更长, 因为老年代中有更多的存活对象。


When using phantom references, you have to literally do manual memory management in regards of flagging such references eligible for garbage collection. It is dangerous, as a superficial glance at the javadoc may lead one to believe they are the completely safe to use:

最后要注意, 在使用`虚引用`(phantom reference)时, 必须进行手动内存管理, 以标识这些对象是否可以安全地回收。表面上看起来很安全, 但实际上并不是这样。 javadoc 中写道:


> In order to ensure that a reclaimable object remains so, the referent of a phantom reference may not be retrieved: The get method of a phantom reference always returns null.

> 为了防止可回收对象的残留, 虚引用对象不应该被获取:  phantom reference 的 `get` 方法总是返回 `null`。


Surprisingly, many developers skip the very next paragraph in the same javadoc (emphasis added):

令人惊讶的是, 很多开发者忽略了 javadoc 中的下一段内容(**这才是重点**):


> Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.

> 与软引用和弱引用不同, 虚引用不会被 GC 自动清除, 因为他们被存放到队列中. 通过虚引用可达的对象会继续留在内存, 除非此引用被清除, 或者自身变为不可达对象。


That is right, we have to manually clear() up phantom references or risk facing a situation where the JVM starts dying with an OutOfMemoryError. The reason why the Phantom references are there in the first place is that this is the only way to find out when an object has actually become unreachable via the usual means. Unlike with soft or weak references, you cannot resurrect a phantom-reachable object.
 

也就是说,我们必须手动调用 clear() 来清除虚引用, 否则可能会因为 OutOfMemoryError 而导致 JVM 挂掉.  使用虚引用的理由是, 这是用编程手段来跟踪某个对象何时变为不可达对象的唯一的常规手段。 和软引用/弱引用不同, 我们不能复活虚可达(phantom-reachable)对象。


## 示例


Let us take a look at another demo application that allocates a lot of objects, which are successfully reclaimed during minor garbage collections. Bearing in mind the trick of altering the tenuring threshold from the previous section on promotion rate, we could run this application with -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1 and see this in GC logs:

让我们看另一个示例, 其中创建了大量的对象, 并且在 minor GC 时完成回收. 和前面一样,修改提升阀值。使用的JVM参数为: ` -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1` , GC日志如下所示:


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

可以看到, Full GC 的次数很少。但假如程序使用弱引用来指向创建的对象, 使用JVM参数 `-Dweak.refs=true`, 则情况会发生明显变化. 使用弱引用的原因很多, 比如在 weak hash map 中使用对象作为Key, 最后进行分配分析。在任何情况下, 使用弱引用都可能会导致以下情形:


	2.059: [Full GC (Ergonomics)  20365K->19611K(22528K), 0.0654090 secs]
	2.125: [Full GC (Ergonomics)  20365K->19711K(22528K), 0.0707499 secs]
	2.196: [Full GC (Ergonomics)  20365K->19798K(22528K), 0.0717052 secs]
	2.268: [Full GC (Ergonomics)  20365K->19873K(22528K), 0.0686290 secs]
	2.337: [Full GC (Ergonomics)  20365K->19939K(22528K), 0.0702009 secs]
	2.407: [Full GC (Ergonomics)  20365K->19995K(22528K), 0.0694095 secs]




As we can see, there are now many full collections, and the duration of the collections is an order of magnitude longer! Another case of premature promotion, but this time a tad trickier. The root cause, of course, lies with the weak references. Before we added them, the objects created by the application were dying just before being promoted to the old generation. But with the addition, they are now sticking around for an extra GC round so that the appropriate cleanup can be done on them. Like before, a simple solution would be to increase the size of the young generation by specifying -Xmx64m -XX:NewSize=32m:

可以看到, 发生了多次 full GC, 比起前一节的示例, GC时间增加了一个数量级! 这是过早提升的另一个例子, 但这次情况更加棘手. 当然,问题的根源在于弱引用。这些临死的对象, 在添加弱引用之后, 被提升到老年代. 但是, 他们现在陷入另一个GC循环之中, 所以需要对其做一些适当的清理。像之前一样, 最简单的解决办法是增加年轻代的大小, 例如指定JVM参数: `-Xmx64m -XX:NewSize=32m`:


	2.328: [GC (Allocation Failure)  38940K->13596K(61440K), 0.0012818 secs]
	2.332: [GC (Allocation Failure)  38172K->14812K(61440K), 0.0060333 secs]
	2.341: [GC (Allocation Failure)  39388K->13948K(61440K), 0.0029427 secs]
	2.347: [GC (Allocation Failure)  38524K->15228K(61440K), 0.0101199 secs]
	2.361: [GC (Allocation Failure)  39804K->14428K(61440K), 0.0040940 secs]
	2.368: [GC (Allocation Failure)  39004K->13532K(61440K), 0.0012451 secs]




The objects are now once again reclaimed during minor garbage collection.

这时候, 对象在 minor GC 中就被回收了。


The situation is even worse when soft references are used as seen in the next demo application. The softly-reachable objects are not reclaimed until the application risks getting an OutOfMemoryError. Replacing weak references with soft references in the demo application immediately surfaces many more Full GC events:

更坏的情况是使用软引用,例如下面的程序。如果程序不面临 `OutOfMemoryError` , 软引用对象就不会被回收. 在示例程序中,用软引用替代弱引用, 立即出现更多的 Full GC 事件:


	2.162: [Full GC (Ergonomics)  31561K->12865K(61440K), 0.0181392 secs]
	2.184: [GC (Allocation Failure)  37441K->17585K(61440K), 0.0024479 secs]
	2.189: [GC (Allocation Failure)  42161K->27033K(61440K), 0.0061485 secs]
	2.195: [Full GC (Ergonomics)  27033K->14385K(61440K), 0.0228773 secs]
	2.221: [GC (Allocation Failure)  38961K->20633K(61440K), 0.0030729 secs]
	2.227: [GC (Allocation Failure)  45209K->31609K(61440K), 0.0069772 secs]
	2.234: [Full GC (Ergonomics)  31609K->15905K(61440K), 0.0257689 secs]




And the king here is the phantom reference as seen in the third demo application. Running the demo with the same sets of parameters as before would give us results that are pretty similar as the results in the case with weak references. The number of full GC pauses would, in fact, be much smaller because of the difference in the finalization described in the beginning of this section.

最关键的是第三个示例中的虚引用, 使用同样的JVM启动参数,结果和弱引用示例非常相似。实际上, full GC暂停的次数会小得多, 原因前面说过, 他们有不同的终结方式。


However, adding one flag that disables phantom reference clearing (-Dno.ref.clearing=true) would quickly give us this:

如果增加一个JVM启动参数 (-Dno.ref.clearing=true), 禁用虚引用清理, 则可以看到:


	4.180: [Full GC (Ergonomics)  57343K->57087K(61440K), 0.0879851 secs]
	4.269: [Full GC (Ergonomics)  57089K->57088K(61440K), 0.0973912 secs]
	4.366: [Full GC (Ergonomics)  57091K->57089K(61440K), 0.0948099 secs]




Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

main 线程中抛出异常 ` java.lang.OutOfMemoryError: Java heap space`.


One must exercise extreme caution when using phantom references and always clear up the phantom reachable objects in a timely manner. Failing to do so will likely end up with an OutOfMemoryError. And trust us when we say that it is quite easy to fail at this: one unexpected exception in the thread that processes the reference queue, and you will have a dead application at your hand.

使用虚引用时需要小心谨慎, 并及时清理虚可达对象。如果不清理, 就可能会发生 `OutOfMemoryError`. 请相信我们的经验教训:  处理 reference queue 的 线程没 catch 住 exception , 系统很快就会被整挂了。


### Could my JVMs be Affected?

### 对JVM会造成什么影响?


As a general recommendation, consider enabling the -XX:+PrintReferenceGC JVM option to see the impact that different references have on garbage collection. If we add this to the application from the WeakReference example, we will see this:

建议使用JVM参数 `-XX:+PrintReferenceGC` 来看看各种引用对GC的影响. 如将这个参数用于 WeakReference 的例子, 将会看到:


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

同样, 只有确定 GC 对应用程序的吞吐量和延迟有影响之后, 才应该花心思来分析这些信息. 此时需要查看这部分日志。通常情况下, 每次GC清理的引用数量都是很少的, 大部分情况下都是 `0`。如果GC 花了较多时间来清理这类引用, 或者清除了很多的此类引用, 那就需要进一步分析和调查。


### 解决方案


When you have verified the application actually is suffering from the mis-, ab- or overuse of either weak, soft or phantom references, the solution often involves changing the application’s intrinsic logic. This is very application specific and generic guidelines are thus hard to offer. However, some generic solutions to bear in mind are:

如果确定程序碰到了 `mis-`,  `ab-` 问题或者滥用 weak, soft 及 phantom  引用, 通常是要修改程序的实现逻辑。每个系统都不一样, 因此很难提供通用指南。但有一些常用的办法:


- Weak references – if the problem is triggered by increased consumption of a specific memory pool, an increase in the corresponding pool (and possibly the total heap along with it) can help you out. As seen in the example section, increasing the total heap and young generation sizes alleviated the pain.
- Phantom references – make sure you are actually clearing the references. It is easy to dismiss certain corner cases and have the clearing thread to not being able to keep up with the pace the queue is filled or to stop clearing the queue altogether, putting a lot of pressure to GC and creating a risk of ending up with an OutOfMemoryError.
- Soft references – when soft references are identified as the source of the problem, the only real way to alleviate the pressure is to change the application’s intrinsic logic.

<br/>

- `弱引用`(`Weak references`) —— 如果某个内存池的使用量增大, 而引起问题, 那么增加这个内存池的大小(可能也要增加堆内存的最大容量)。如同示例中所看到的, 增加堆内存的大小以及年轻代的大小可以减轻症状。
- `虚引用`(`Phantom references`) —— 请确保在程序中调用了虚引用的 clear 方法。很容易忽略某些代码中的虚引用, 或者清理的速度跟不上产生的速度, 或者清除引用队列的线程退出, 就会对GC 造成很大压力, 最终有可能引起 `OutOfMemoryError`。
- `软引用`(`Soft references`) ——  如果确定问题的根源是软引用, 唯一的解决办法是修改程序源码, 改变内部逻辑。




## Other Examples

## 其他示例


Previous chapters covered the most common problems related to poorly behaving GC. Unfortunately, there is a long list of more specific cases, where you cannot apply the knowledge from previous chapters. This section describes a few of the more unusual problems that you may face.

前面介绍了最常见的GC性能问题。但所学的原理很多没有具体的情景示例展现。本节介绍一些不常发生, 但也可能会碰到的问题。


### RMI 与 GC


When your application is publishing or consuming services over RMI, the JVM periodically launches full GC to make sure that locally unused objects are also not taking up space on the other end. Bear in mind that even if you are not explicitly publishing anything over RMI in your code, third party libraries or utilities can still open RMI endpoints. One such common culprit is for example JMX, which, if attached to remotely, will use RMI underneath to publish the data.

如果系统提供或者消费 RMI 服务, 则JVM会定期调用 full GC 来确保本地未使用的对象在另一端也不占用空间. 记住, 即使你的代码中没有发布 RMI 服务, 但第三方或者工具库也可能打开 RMI 终端. 最常见的元凶, 是 JMX, 如果通过JMX连接到远端, 在底层就会使用RMI来发布数据。


The problem is exposed by seemingly unnecessary and periodic full GC pauses. When you check the old generation consumption, there is often no pressure to the memory as there is plenty of free space in the old generation, but full GC is triggered, stopping the application threads.

问题就是有很多不必要的周期性的 full GC暂停。如果查看老年代的使用情况, 一般是没有内存压力, 其中还存在大量的空闲区域, 但 full GC 就是被触发了, 也就暂停了所有的应用线程。


This behavior of removing remote references via System.gc() is triggered by the sun.rmi.transport.ObjectTable class requesting garbage collection to be run periodically as specified in the sun.misc.GC.requestLatency() method.

这种周期性调用 `System.gc()` 删除远程引用的行为, 是在  `sun.rmi.transport.ObjectTable` 类中, 调用  `sun.misc.GC.requestLatency(long gcInterval)` 执行的。


For many applications, this is not necessary or outright harmful. To disable such periodic GC runs, you can set up the following for your JVM startup scripts:

对许多应用来说, 这根本没必要, 甚至对性能有害。 禁止这种周期性的 GC 行为, 可以使用以下 JVM 参数:


	java -Dsun.rmi.dgc.server.gcInterval=9223372036854775807L 
		-Dsun.rmi.dgc.client.gcInterval=9223372036854775807L 
		com.yourcompany.YourApplication



This sets the period after which System.gc() is run to Long.MAX_VALUE; for all practical matters, this equals eternity.

这让 `Long.MAX_VALUE` 毫秒之后, 才调用 `System.gc()`, 实际运行的系统可能永远也不会触发。

> ObjectTable.class

	private static final long gcInterval = 
	((Long)AccessController.doPrivileged(
		new GetLongAction("sun.rmi.dgc.server.gcInterval", 3600000L)
		)).longValue();


可以看到, 默认值是 `3600000L`,也就是1小时触发一次 Full GC。


An alternative solution for the problem would to disable explicit calls to System.gc() by specifying -XX:+DisableExplicitGC in the JVM startup parameters. We do not however recommend this solution as it can have other side effects.

另一种方式是指定JVM参数 `-XX:+DisableExplicitGC`, 禁止显式地调用 `System.gc()`.  但我们**墙裂反对** 这种方式, 因为有大坑。


-- 校对到此处 !!!!!

### JVMTI tagging 与 GC


Whenever the application is run alongside with a Java Agent (-javaagent), there is a chance that the agent can tag the objects in the heap using JVMTI tagging. Agents can use tagging for various reasons that are not in the scope of this handbook, but there is a GC-related performance issue that can start affecting the latency and throughput of your application if tagging is applied to a large subset of objects inside the heap.

Whenever the application is run alongside with a Java Agent (- javaagent), there is a chance that the Agent can tag the objects in the heap using JVMTI tagging.代理可以使用标签由于种种原因不了本手册的范围,但有一个GC-related性能问题,可以影响应用程序的延迟和吞吐量如果标签应用于大型子集内的对象堆。


The problem is hidden in the native code where JvmtiTagMap::do_weak_oops iterates over all the tags during each garbage collection event and performs a number of not-so-cheap operations for all of them. To make things worse, this operation is performed sequentially and is not parallelized.

问题是隐藏在本机代码JvmtiTagMap::do_weak_oops遍历所有的标签在每个垃圾收集事件和执行一系列每操作。更糟的是,这个操作是按顺序执行的,而不是并行。


With a large number of tags, this implies that a large part of the GC process is now carried out in a single thread and all the benefits of parallelism disappear, potentially increasing the duration of GC pauses by an order of magnitude.

与大量的标签,这意味着现在GC过程的很大一部分是在一个线程并行消失所带来的好处,可能增加GC暂停的时间由一个数量级。


To check whether or not a particular agent can be the reason for extended GC pauses, you would need to turn on the diagnostic option of –XX:+TraceJVMTIObjectTagging. Enabling the trace will allow you to get an estimate of how much native memory the tag map consumes and how much time the heap walks take.

检查是否一个特定的代理可以延长GC暂停的原因,你需要打开的诊断选项- xx:+ TraceJVMTIObjectTagging.启用跟踪会让你得到一个估计的本机内存多少标签映射消耗和堆走多少时间。


If you are not the author of the agent yourself, fixing the problem is often out of your reach. Apart from contacting the vendor of a particular agent you cannot do much. In case you do end up in a situation like this, recommend that the vendor clean up the tags that are no longer needed.

如果你没有代理的作者自己,解决问题往往是遥不可达的。除了联系供应商特定的代理你不能做太多.如果你在这种情况下,建议供应商清除不再需要的标签。



### 巨无霸对象的分配(Humongous Allocations)


Whenever your application is using the G1 garbage collection algorithm, a phenomenon called humongous allocations can impact your application performance in regards of GC. To recap, humongous allocations are allocations that are larger than 50% of the region size in G1.

如果使用G1算法来进行垃圾收集, 这种被称为巨无霸对象的分配就会因为GC影响程序的性能。回顾一下,  在G1中巨无霸对象是指超过 region size 50% 的空间分配。


Having frequent humongous allocations can trigger GC performance issues, considering the way that G1 handles such allocations:

频繁的巨无霸对象分配会引起GC的性能问题, 看看G1处理这种分配的方式:


- If the regions contain humongous objects, space between the last humongous object in the region and the end of the region will be unused. If all the humongous objects are just a bit larger than a factor of the region size, this unused space can cause the heap to become fragmented.
- Collection of the humongous objects is not as optimized by the G1 as with regular objects. It was especially troublesome with early Java 8 releases – until Java 1.8u40 the reclamation of humongous regions was only done during full GC events. More recent releases of the Hotspot JVM free the humongous regions at the end of the marking cycle during the cleanup phase, so the impact of the issue has been reduced significantly for newer JVMs.

<br/>

- 如果某个 region 包含巨无霸对象, 则最后一个巨无霸对象到该区域结尾之间的(空闲)空间将不会被使用。如果所有的巨无霸对象都只是比 region size 的某个因子大一点, 则未使用的空间将会导致堆内存碎片问题。
- 对巨无霸对象的垃圾收集没有被 G1 优化。这在Java 8的早期版本中是特别麻烦的 —— 在 **Java 1.8u40** 之前, 巨无霸对象所在区域的回收都只能在完全GC事件中进行。最新版本的 Hotspot JVM 在marking 阶段的结尾, cleanup phase 释放巨无霸空间, 所以这个问题的影响在新版的JVM中已经大大减小了。




To check whether or not your application is allocating objects in humongous regions, the first step would be to turn on GC logs similar to the following:

要检查是否有对象分配在巨无霸区域, 第一步是打开GC日志， 类似下面这样:


	java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 
		-XX:+PrintReferenceGC -XX:+UseG1GC 
		-XX:+PrintAdaptiveSizePolicy -Xmx128m 
		MyClass




Now, when you check the logs and discover sections like these:

现在, 在检查日志时, 会发现类似这样的部分:


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

现在有证据表明, 应用程序确实分配了巨无霸对象.  `G1 Humongous Allocation ` 这个GC暂停的原因就是证据, 在 “allocation request: 1048592 bytes” 这一节, 在这里我们可以看到程序试图分配一个 `1,048,592` 字节大小的对象, 这比JVM指定的巨无霸区域大小(`2MB`)的 50% 多出了 16 个字节。


The first solution for humongous allocation is to change the region size so that (most) of the allocations would not exceed the 50% limit triggering allocations in the humongous regions. The region size is calculated by the JVM during startup based on the size of the heap. You can override the size by specifying -XX:G1HeapRegionSize=XX in the startup script. The specified region size must be between 1 and 32 megabytes and has to be a power of two.

处理巨无霸对象分配的第一种方式, 是修改 region size , 使得(大多数的)分配不超过50%的限制, 引起在大对象区域的分配. 区域大小是由JVM在启动期间基于堆的大小计算得出的。你可以通过指定 `-XX:G1HeapRegionSize=XX`  启动参数来覆盖默认设置. 指定的区域大小必须介于 `1~32MB` 之间, 还必须是2的幂。


This solution can have side effects – increasing the region size reduces the number of regions available so you need to be careful and run extra set of tests to see whether or not you actually improved the throughput or latency of the application.

这个解决方案也有副作用, 增加区域大小也就减少了可用区域的数量, 你需要注意, 最好执行额外的测试来看看是否真的改善了吞吐量和延迟。


A more time-consuming but potentially better solution would be to understand whether or not the application can limit the size of allocations. The best tools for the job in this case are profilers. They can give you information about humongous objects by showing their allocation sources with stack traces.

更耗费时间, 但可能更好的解决方式是了解程序是否可以限制分配的大小。对付这种情况，最好的工具是分析器. 他们可以给你巨无霸对象的信息, 通过堆栈跟踪展示它们是在哪里分配的。




### 总结


With the enormous number of possible applications that one may run on the JVM, coupled with the hundreds of JVM configuration parameters that one may tweak for GC, there are astoundingly many ways in which the GC may impact your application’s performance.

JVM上运行的程序形形色色, JVM 启动参数也有上百个, 其中有很多参数会影响 GC, 所以调优GC性能的方法也有很多种。


Therefore, there is no real silver bullet approach to tuning the JVM to match the performance goals you have to fulfill. What we have tried to do here is walk you through some common (and not so common) examples to give you a general idea of how problems like these can be approached. Coupled with the tooling overview and with a solid understanding of how the GC works, you have all the chances of successfully tuning garbage collection to boost the performance of your application.


还是那句话, 没有真正的银弹, 能满足所有的性能调优指标。 我们能做的只是介绍一些常见的/和不常见的示例, 让你在碰到类似问题的时候知道是怎么回事. 深入理解GC工作原理, 熟练应用各种工具, 你就可以进行GC调优, 提高程序性能。


原文链接:  [GC Tuning: In Practice](https://plumbr.eu/handbook/gc-tuning-in-practice)





<div style="page-break-after : always;"> </div>


