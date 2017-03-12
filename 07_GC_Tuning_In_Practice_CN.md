# 7. GC 调优(实战篇)


本章介绍会导致GC性能问题的典型情况。相关示例都来源于生产环境, 为了演示的需要,做了一定简化。


## 高分配速率(High Allocation Rate)


分配速率(`Allocation rate`)用来表示单位时间内分配的内存总量。单位通常是 `MB/sec`, 也可以使用 `PB/year` 等。这很容易理解, 通过一段时间来衡量Java代码创建的内存总量。


过高的分配速率会严重影响程序性能。在JVM中, 这个问题会导致大量的GC开销。


### 如何测量分配速率?


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



根据GC日志就可以计算出分配速率。 计算 `上一次垃圾收集之后`,与`下一次GC开始之前`的年轻代使用量的差值。 比如上面的日志中, 可以获取以下信息:


- JVM启动之后 `291ms`, 共创建了 `33,280 KB` 的对象。 第一次 Minor GC(小型GC) 完成后, 年轻代中还有 `5,088 KB` 的对象存活。
- 在启动之后 `446 ms`, 年轻代的使用量增加到 `38,368 KB`, 触发第二次GC, 完成后年轻代的使用量减少到 `5,120 KB`。
- 在启动之后 `829 ms`, 年轻代的使用量为 `71,680 KB`, GC后变为 `5,120 KB`。


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



通过这些信息可以知道, 在测量期间, 该程序的内存分配速率为 `161 MB/sec`。


### 分配速率的意义


计算出分配速率, 就会知道分配速率如何影响吞吐量: 分配速率的变化,会增加或降低GC暂停的频率。 请记住, 只有年轻代的 [minor GC](http://blog.csdn.net/renfufei/article/details/54144385#t9) 受分配速率的影响。 而老年代GC的频率和持续时间不受 **分配速率**(`allocation rate`)的直接影响, 而是受 **提升速率**(`promotion rate`)的影响,我们在下一节中介绍。


这样我们就只关心 [Minor GC](http://blog.csdn.net/renfufei/article/details/54144385#t9) 暂停, 接下来看年轻代的这几个内存池。因为对象分配在 [Eden 区](http://blog.csdn.net/renfufei/article/details/54144385#t3), 所以我们来审查 Eden 区的大小和分配速率的关系.  看看增加 Eden 区的容量能不能减少 Minor GC 暂停次数, 从而使程序能够维持更快的分配速率。


事实上, 通过参数 `-XX:NewSize`、 `-XX:MaxNewSize` 以及 `-XX:SurvivorRatio` 设置不同的 Eden 空间来运行同一程序时, 可以看到:


- Eden 空间为 `100 MB` 时, 分配速率低于 `100 MB/秒`。
- 将 Eden 区增大为 `1 GB`, 分配速率也随之增长,大约等于 `200 MB/秒`。


为什么会这样? —— 因为减少GC暂停,就等价于减少任务线程的停顿，就可以做更多工作, 也就创建了更多对象, 所以对同一应用程序, 分配速率越高越好。


在得出 “Eden去越大越好” 这种结论前, 我们注意到, 分配速率可能会,也可能不会直接影响程序的实际吞吐量。 吞吐量和分配速率有一定关系, 分配速率会影响 minor GC 暂停, 但对总体吞吐量的影响, 还要考虑 Major GC(大型GC)暂停, 而且吞吐量的单位不是 `MB/秒`， 而是应用所处理的业务量。


### 示例


参考 [Demo程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java)。假设系统连接了一个外部的数字传感器。应用通过专有线程, 不断地获取传感器的值,(此处使用随机数模拟), 其他线程会调用 `processSensorValue()` 方法, 传入传感器的值来执行某些操作, :


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


如同类名所示, 这个Demo是模拟 boxing 的。为了判断 null 值, 使用的是包装类型 `Double`。 基于传感器的最新值进行计算, 但从传感器取值是一个耗时操作, 所以采用了异步方式： 一个线程不断获取新值, 计算线程则直接使用暂存的最新值, 从而避免同步获取。


[Demo 程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java)在运行的过程中, 由于分配速率太大而受GC拖累。下一节将确认问题, 并给出解决办法。


### 对JVM会造成什么影响?


首先，我们应该检查程序的吞吐量是否降低。如果创建了过多的临时对象, 小型GC的次数就会增加。如果并发较大, 则GC很可能会严重影响吞吐量。


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


很显然 minor GC 的频率太高了。这说明创建了大量的对象。另外, 年轻代在 GC 之后的使用量又很低, 也没有 full GC 发生。 这种症状表明, GC对吞吐量有严重的影响。



### 解决方案


在某些情况下,只要增加年轻代空间的大小, 即可降低分配速率过高的影响。增加年轻代空间并不会降低分配速率, 但是会减少GC的频率。如果每次GC后只有少量对象存活, minor GC 的暂停时间就不会明显增加。


运行 [示例程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/Boxing.java) , 增加堆内存大小,(同时也就增大了年轻代的大小), 使用的JVM参数为 `-Xmx64m`:


	2.808: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003748 secs]
	2.831: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0004538 secs]
	2.855: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0003355 secs]
	2.879: [GC (Allocation Failure) 
			[PSYoungGen: 20512K->32K(20992K)], 0.0005592 secs]



但只增加堆内存的大小,有时候并不能解决问题。通过前面学习的知识, 我们可以通过分配分析器找出大部分垃圾产生的位置。实际上在此示例中, 99%的对象是 `Double` 包装类, 在`readSensor` 方法中创建。可以进行简单的优化, 将创建的 `Double` 对象替换为原生类型 `double`, 而 null 值的判断, 可以使用  [Double.NaN](https://docs.oracle.com/javase/7/docs/api/java/lang/Double.html#NaN) 来进行。由于原生类型不算是对象, 也就不会产生垃圾, 不容易产生GC事件。优化之后, 不在堆内存中分配新对象, 而是直接覆盖一个属性域即可。


对示例程序进行[简单的改造](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/FixedBoxing.java)( [查看diff](https://gist.github.com/gvsmirnov/0270f0f15f9498e3b655) ) 之后, GC暂停基本上完全消除。有时候 JVM 也会很智能, 使用 逃逸分析技术(escape analysis technique) 来避免过度分配。简单来说,JIT编译器可以分析得知, 方法创建的某些对象永远都不会“逃出”此方法的作用域。这时候就不需要在堆上分配这些对象, 也就不会产生垃圾, 所以JIT编译器所做的一种优化就是: 消除内存分配。请参考 [基准测试](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/jit/EscapeAnalysis.java) 。


## 过早提升(Premature Promotion)


先介绍一个基本概念 —— **提升速率**(`premature promotion`)。提升速率用于衡量单位时间内从年轻代提升到老年代的数据量。一般使用 `MB/sec` 作为单位, 和分配速率类似。


JVM会将长时间存活的对象从年轻代提升到老年代。根据分代假设, 可能存在一种情况, 老年代中不仅有存活时间长的对象,也可能有存活时间短的对象。这就是过早提升：存活时间还不长的对象被提升到了老年代。


major GC 不是为这种频繁回收而设计的, 但现在 major GC 也要清理这些生命短暂的对象, 所以会导致GC暂停时间过长。这就严重影响了系统吞吐量。


### 如何测量提升速率


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



根据这些信息, 就可以得出观察周期内的提升速率。平均提升速率为 `92 MB/秒`, 峰值为 `140.95 MB/秒`。


请注意, 只能根据 minor GC 计算这些信息. Full GC 暂停的日志,不能用于计算提升速率, 因为 major GC 会清理老年代中的一部分对象。


### 提升速率的意义


和分配速率一样, 提升速率也会影响GC暂停的频率。但分配速率主要影响 [minor GC](http://blog.csdn.net/renfufei/article/details/54144385#t8), 而提升速率则影响的是 [major GC](http://blog.csdn.net/renfufei/article/details/54144385#t8) 的频率。有大量的数据提升,自然很快就会把老年代填满。 老年代填充的越快, 则 major GC 事件的频率将会越快。


![](07_01_how-java-garbage-collection-works.png)


此前我们说过, full GC 通常需要更多的时间, 因为需要处理更多的对象, 还要额外执行碎片整理等复杂的过程。


### 示例


让我们看一个[过早提升的示例](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/PrematurePromotion.java)。 这个程序创建/获取大量的对象/数据,并暂存到集合之中, 累积到一定数量之后,进行批处理:


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


此 [Demo 程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/PrematurePromotion.java)受到过早提升的影响。在接下来的部分将进行确认并给出解决办法。


### 对JVM会造成什么影响?


一般来说,过早提升的症状表现为以下形式:


- 在很短的时间内频繁地执行 full GC。
- 每次 full GC 后老年代的使用率都很低, 在10-20%或以下。
- 提升速率约等于分配速率。


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



乍一看似乎不是过早提升的问题。事实上,在每次GC之后老年代的使用率似乎在减少。但反过来想, 要是没有对象提升或者提升率很小, 也就不会看到这么多的 Full GC。


简单解释一下这种GC行为: 有很多对象提升到老年代的同时, 也有很多老年代中的对象被回收了, 这就造成了老年代使用量减少的假象. 但事实是大量的对象被不断地提升到老年代, 触发 full GC。


### 解决方案


简单点, 要解决这类问题, 需要让年轻代存放得下暂存的数据。有两种简单的方法:

一是增加年轻代的大小, 设置JVM启动参数, 类似这样: ` -Xmx64m -XX:NewSize=32m`, 程序在执行时, Full GC 自然会减少很多, 只影响 minor GC的持续时间:



	2.251: [GC (Allocation Failure) 
			[PSYoungGen: 28672K->3872K(28672K)] 
			37126K->12358K(61440K), 0.0008543 secs]
	2.776: [GC (Allocation Failure) 
			[PSYoungGen: 28448K->4096K(28672K)] 
			36934K->16974K(61440K), 0.0033022 secs]



二是减少每批次处理的数量, 也能得到类似的结果. 至于哪种方案更好, 很大程度上取决于应用程序中真正执行的逻辑。在某些情况下, 业务逻辑不允许减少批处理的数量, 就只能增加可用内存,或者重新指定年轻代的大小。


如果都不可行, 那就优化数据结构, 减少内存消耗。但总体目标依然是一致的: 让临时数据能够在年轻代存放得下。


## Weak, Soft 以及 Phantom 引用


另一类影响GC的问题是程序中的 non-strong 引用。虽然这类引用在很多情况下可以避免出现 [`OutOfMemoryError`](https://plumbr.eu/outofmemory),  但滥用也会对GC行为造成严重影响, 进而降低系统性能。


## 需要关注弱引用的原因


首先, 你应该知道, `弱引用`(weak reference) 是可以被垃圾收集器强制回收的。当GC发现一个弱可达对象(weakly reachable),即指向该对象的引用只剩下弱引用时, 则会将其置入相应的ReferenceQueue 中, 成为可终结的对象. 之后可能会遍历这个 reference queue, 并执行相应的清理。典型的示例是清除缓存中不再引用的KEY。


当然, 在这个时候, 我们还可以将该对象赋值给新的强引用, 在最后终结和回收前, GC会再次确认是否可以回收。因此, 弱引用对象的回收过程是横跨多个GC周期的。


弱引用实际上使用的很多。大部分缓存框架(caching solution)都是基于弱引用来实现的, 所以我们虽然没有直接编写弱引用代码, 但程序中依然会存在大量的弱引用对象。


其次, `软引用`(soft reference) 比弱引用更难被垃圾收集器回收. 回收软引用没有确切的时间点, 由各个JVM自己决定. 一般只会在即将耗尽可用内存时, 才会回收软引用,作为最后手段。这意味着, 可能会面临更频繁的 full GC, 暂停时间也会比预期更长, 因为老年代中有更多的存活对象。


最后要注意, 在使用`虚引用`(phantom reference)时, 必须进行手动内存管理, 以标识这些对象是否可以安全地回收。表面上看起来很安全, 但实际上并不是这样。 javadoc 中写道:


> In order to ensure that a reclaimable object remains so, the referent of a phantom reference may not be retrieved: The get method of a phantom reference always returns null.

> 为了防止可回收对象的残留, 虚引用对象不应该被获取:  phantom reference 的 `get` 方法总是返回 `null`。


令人惊讶的是, 很多开发者忽略了 javadoc 中的下一段内容(**这才是重点**):


> Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.

> 与软引用和弱引用不同, 虚引用不会被 GC 自动清除, 因为他们被存放到队列中. 通过虚引用可达的对象会继续留在内存, 除非此引用被清除, 或者自身变为不可达对象。


也就是说,我们必须手动调用 [clear()](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/Reference.html#clear()) 来清除虚引用, 否则可能会因为 [OutOfMemoryError](https://plumbr.eu/outofmemory) 而导致 JVM 挂掉.  使用虚引用的理由是, 这是用编程手段来跟踪某个对象何时变为不可达对象的唯一的常规手段。 和软引用/弱引用不同, 我们不能复活虚可达(phantom-reachable)对象。


## 示例


让我们看[另一个示例](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/WeakReferences.java), 其中创建了大量的对象, 并且在 minor GC 时完成回收. 和前面一样,修改提升阀值。使用的JVM参数为: ` -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1` , GC日志如下所示:


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



可以看到, Full GC 的次数很少。但假如程序使用弱引用来指向创建的对象, 使用JVM参数 `-Dweak.refs=true`, 则情况会发生明显变化. 使用弱引用的原因很多, 比如在 weak hash map 中使用对象作为Key, 最后进行分配分析。在任何情况下, 使用弱引用都可能会导致以下情形:


	2.059: [Full GC (Ergonomics)  20365K->19611K(22528K), 0.0654090 secs]
	2.125: [Full GC (Ergonomics)  20365K->19711K(22528K), 0.0707499 secs]
	2.196: [Full GC (Ergonomics)  20365K->19798K(22528K), 0.0717052 secs]
	2.268: [Full GC (Ergonomics)  20365K->19873K(22528K), 0.0686290 secs]
	2.337: [Full GC (Ergonomics)  20365K->19939K(22528K), 0.0702009 secs]
	2.407: [Full GC (Ergonomics)  20365K->19995K(22528K), 0.0694095 secs]



可以看到, 发生了多次 full GC, 比起前一节的示例, GC时间增加了一个数量级! 这是过早提升的另一个例子, 但这次情况更加棘手. 当然,问题的根源在于弱引用。这些临死的对象, 在添加弱引用之后, 被提升到老年代. 但是, 他们现在陷入另一个GC循环之中, 所以需要对其做一些适当的清理。像之前一样, 最简单的解决办法是增加年轻代的大小, 例如指定JVM参数: `-Xmx64m -XX:NewSize=32m`:


	2.328: [GC (Allocation Failure)  38940K->13596K(61440K), 0.0012818 secs]
	2.332: [GC (Allocation Failure)  38172K->14812K(61440K), 0.0060333 secs]
	2.341: [GC (Allocation Failure)  39388K->13948K(61440K), 0.0029427 secs]
	2.347: [GC (Allocation Failure)  38524K->15228K(61440K), 0.0101199 secs]
	2.361: [GC (Allocation Failure)  39804K->14428K(61440K), 0.0040940 secs]
	2.368: [GC (Allocation Failure)  39004K->13532K(61440K), 0.0012451 secs]



这时候, 对象在 minor GC 中就被回收了。


更坏的情况是使用软引用,例如[下面的程序](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/SoftReferences.java)。如果程序不面临 `OutOfMemoryError` , 软引用对象就不会被回收. 在示例程序中,用软引用替代弱引用, 立即出现更多的 Full GC 事件:


	2.162: [Full GC (Ergonomics)  31561K->12865K(61440K), 0.0181392 secs]
	2.184: [GC (Allocation Failure)  37441K->17585K(61440K), 0.0024479 secs]
	2.189: [GC (Allocation Failure)  42161K->27033K(61440K), 0.0061485 secs]
	2.195: [Full GC (Ergonomics)  27033K->14385K(61440K), 0.0228773 secs]
	2.221: [GC (Allocation Failure)  38961K->20633K(61440K), 0.0030729 secs]
	2.227: [GC (Allocation Failure)  45209K->31609K(61440K), 0.0069772 secs]
	2.234: [Full GC (Ergonomics)  31609K->15905K(61440K), 0.0257689 secs]



最关键的是[第三个示例](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/gc/PhantomReferences.java)中的虚引用, 使用同样的JVM启动参数,结果和弱引用示例非常相似。实际上, full GC暂停的次数会小得多, 原因前面说过, 他们有不同的终结方式。


如果增加一个JVM启动参数 (-Dno.ref.clearing=true), 禁用虚引用清理, 则可以看到:


	4.180: [Full GC (Ergonomics)  57343K->57087K(61440K), 0.0879851 secs]
	4.269: [Full GC (Ergonomics)  57089K->57088K(61440K), 0.0973912 secs]
	4.366: [Full GC (Ergonomics)  57091K->57089K(61440K), 0.0948099 secs]



main 线程中抛出异常 ` java.lang.OutOfMemoryError: Java heap space`.


使用虚引用时需要小心谨慎, 并及时清理虚可达对象。如果不清理, 就可能会发生 [`OutOfMemoryError`](https://plumbr.eu/outofmemory). 请相信我们的经验教训:  处理 reference queue 的 线程没 catch 住 exception , 系统很快就会被整挂了。


### 对JVM会造成什么影响?


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



同样, 只有确定 GC 对应用程序的吞吐量和延迟有影响之后, 才应该花心思来分析这些信息. 此时需要查看这部分日志。通常情况下, 每次GC清理的引用数量都是很少的, 大部分情况下都是 `0`。如果GC 花了较多时间来清理这类引用, 或者清除了很多的此类引用, 那就需要进一步分析和调查。


### 解决方案


如果确定程序碰到了 `mis-`,  `ab-` 问题或者滥用 weak, soft 及 phantom  引用, 通常是要修改程序的实现逻辑。每个系统都不一样, 因此很难提供通用指南。但有一些常用的办法:


- `弱引用`(`Weak references`) —— 如果某个内存池的使用量增大, 而引起问题, 那么增加这个内存池的大小(可能也要增加堆内存的最大容量)。如同示例中所看到的, 增加堆内存的大小以及年轻代的大小可以减轻症状。
- `虚引用`(`Phantom references`) —— 请确保在程序中调用了虚引用的 clear 方法。很容易忽略某些代码中的虚引用, 或者清理的速度跟不上产生的速度, 或者清除引用队列的线程退出, 就会对GC 造成很大压力, 最终有可能引起 [`OutOfMemoryError`](http://www.oracle.com/technetwork/articles/javaee/index-jsp-136424.html)。
- `软引用`(`Soft references`) ——  如果确定问题的根源是软引用, 唯一的解决办法是修改程序源码, 改变内部逻辑。


## 其他示例


前面介绍了最常见的GC性能问题。但所学的原理很多没有具体的情景示例展现。本节介绍一些不常发生, 但也可能会碰到的问题。


### RMI 与 GC


如果系统提供或者消费 [RMI](http://www.oracle.com/technetwork/articles/javaee/index-jsp-136424.html) 服务, 则JVM会定期调用 full GC 来确保本地未使用的对象在另一端也不占用空间. 记住, 即使你的代码中没有发布 RMI 服务, 但第三方或者工具库也可能打开 RMI 终端. 最常见的元凶, 是 JMX, 如果通过JMX连接到远端, 在底层就会使用RMI来发布数据。


问题就是有很多不必要的周期性的 full GC暂停。如果查看老年代的使用情况, 一般是没有内存压力, 其中还存在大量的空闲区域, 但 full GC 就是被触发了, 也就暂停了所有的应用线程。


这种周期性调用 `System.gc()` 删除远程引用的行为, 是在  `sun.rmi.transport.ObjectTable` 类中, 调用  `sun.misc.GC.requestLatency(long gcInterval)` 执行的。


对许多应用来说, 这根本没必要, 甚至对性能有害。 禁止这种周期性的 GC 行为, 可以使用以下 JVM 参数:


	java -Dsun.rmi.dgc.server.gcInterval=9223372036854775807L 
		-Dsun.rmi.dgc.client.gcInterval=9223372036854775807L 
		com.yourcompany.YourApplication



这让 `Long.MAX_VALUE` 毫秒之后, 才调用 [`System.gc()`](http://docs.oracle.com/javase/7/docs/api/java/lang/System.html#gc()), 实际运行的系统可能永远也不会触发。

> ObjectTable.class

	private static final long gcInterval = 
	((Long)AccessController.doPrivileged(
		new GetLongAction("sun.rmi.dgc.server.gcInterval", 3600000L)
		)).longValue();


可以看到, 默认值是 `3600000L`,也就是1小时触发一次 Full GC。


另一种方式是指定JVM参数 `-XX:+DisableExplicitGC`, 禁止显式地调用 `System.gc()`.  但我们**墙裂反对** 这种方式, 因为有大坑。


### JVMTI tagging 与 GC


如果程序启动时指定了 Java Agent (`-javaagent`), agent 就可以使用 [JVMTI tagging](http://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html#Heap) 标记堆中的对象。agent 使用tagging的种种原因本手册不进行讲解, 但如果 tagging 标记了堆内存中大量的对象, 很可能会引起 GC 性能问题, 导致延迟增加, 以及吞吐量降低。


问题在 native 代码中, `JvmtiTagMap::do_weak_oops` 在每次GC时,遍历所有的标签(tag),并执行一些比较耗时的操作。更坑的是, 这些操作是按顺序串行执行的。


如果存在大量的标签, 就意味着 GC 中有很大一部分工作是单线程执行的, 可能会增加一个数量级的GC暂停时间。


检查是否因为 agent 增加了GC暂停的时间, 可以使用诊断参数 `–XX:+TraceJVMTIObjectTagging`. 启用跟踪之后, 可以估算出内存中 tag 映射了多少 native 内存, 以及遍历所消耗的时间。


如果你不是 agent 的作者, 那一般是搞不定这种问题的。除了提BUG之外你什么都做不了. 如果面临这种情况, 可以建议厂商清理不必要的标签。


### 巨无霸对象的分配(Humongous Allocations)


如果使用 G1 垃圾收集算法, 会有一种巨无霸对象引起的 GC 性能问题。

> **说明**: 在G1中, 巨无霸对象是指所占空间超过一个小堆区(region) `50%` 的对象。


频繁的创建巨无霸对象, 无疑会造成GC的性能问题, 看看G1的处理方式:


- 如果某个 region 中含有巨无霸对象, 则巨无霸对象之后的空间将不会被分配。如果所有巨无霸对象都占 region size 的某个比例, 则未使用的空间会引起内存碎片问题。
- G1 没有对巨无霸对象进行回收优化。这在 JDK 8 以前是个特别棘手的问题 —— 在 [**Java 1.8u40**](https://bugs.openjdk.java.net/browse/JDK-8027959) 之前的版本中, 巨无霸对象所在区域的回收只能在 full GC 中进行。最新版本的 Hotspot JVM 在 marking 阶段之后的 cleanup 阶段中释放巨无霸区间, 所以这个问题在新版本JVM中的影响已经大大减小了。


要监控是否存在巨无霸对象, 可以打开GC日志, 命令如下:


	java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 
		-XX:+PrintReferenceGC -XX:+UseG1GC 
		-XX:+PrintAdaptiveSizePolicy -Xmx128m 
		MyClass


GC 日志中可能会发现这样的部分:


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



这样的日志就是证据, 应用程序确实创建了巨无霸对象. 可以看到: `G1 Humongous Allocation ` 是 GC暂停的原因。 再看前面一点的 “allocation request: 1048592 bytes” , 可以看到程序试图分配一个 `1,048,592` 字节的对象, 这要比巨无霸区域(`2MB`)的 `50%` 多出 16 个字节。



第一种解决方式, 是修改 region size , 以使得大多数的对象不超过 `50%`, 也就不进行大对象巨无霸对象区域的分配。 region 的大小在启动时根据堆内存的大小算出。可以指定启动参数来覆盖默认设置, `-XX:G1HeapRegionSize=XX`。 指定的 region size 必须在 `1~32MB` 之间, 还必须是2的幂 【2^10 = 1024 = 1KB; 2^20=1MB; 所以大小只能是: `1m`,`2m`,`4m`,`8m`,`16m`,`32m`;】。


这种方式也有副作用, 增加 region 的大小也就变相地减少了 region 的数量, 所以需要谨慎使用,  最好进行一些测试, 看看是否改善了吞吐量和延迟。


更好的方式会需要一些工作量, 即在程序中限制对象的大小, 如果可以的话。最好是使用分析器, 可以展示出巨无霸对象的信息, 以及分配所在的堆栈跟踪信息。


### 总结


JVM上运行的程序形形色色, JVM 启动参数也有上百个, 其中有很多参数会影响 GC, 所以调优GC性能的方法也有很多种。


还是那句话, 没有真正的银弹, 能满足所有的性能调优指标。 我们能做的只是介绍一些常见的/和不常见的示例, 让你在碰到类似问题的时候知道是怎么回事. 深入理解GC工作原理, 熟练应用各种工具, 你就可以进行GC调优, 提高程序性能。


原文链接:  [GC Tuning: In Practice](https://plumbr.eu/handbook/gc-tuning-in-practice)

