# 2. Java的垃圾收集


The introduction to Mark and Sweep Garbage Collection is a mostly theoretical one. When things come to practice, numerous adjustments need to be done to accommodate for real-world scenarios and needs. For a simple example, let us take a look at what sorts of bookkeeping the JVM needs to do so that we can safely continue allocating objects.


**标记-清除**(Mark and Sweep)是最经典的垃圾收集算法。将理论用于生产实践时,  会有很多需要优化调整的地点,  以适应具体环境。下面通过一个简单的例子, 让我们一步步记录下来,  看看如何才能保证JVM能安全持续地分配对象。



Fragmenting and Compacting

碎片整理(Fragmenting and Compacting)



Whenever sweeping takes place, the JVM has to make sure the areas filled with unreachable objects can be reused. This can (and eventually will) lead to memory fragmentation which, similarly to disk fragmentation, leads to two problems:


每次执行清除(sweeping), JVM 都必须保证不可达对象占用的内存能被回收重用。但这(最终)有可能会产生内存碎片(类似于磁盘碎片),  进而引发两个问题:


- Write operations become more time-consuming as finding the next free block of sufficient size is no longer a trivial operation.

- When creating new objects, JVM is allocating memory in contiguous blocks. So if fragmentation escalates to a point where no individual free fragment is large enough to accommodate the newly created object, an allocation error occurs.

- 写入操作越来越耗时, 因为寻找一块足够大的空闲内存会变得非常麻烦。

- 在创建新对象时, JVM在连续的块中分配内存。如果碎片问题很严重, 直至没有空闲片段能存放下新创建的对象,就会发生内存分配错误(allocation error)。



To avoid such problems, the JVM is making sure the fragmenting does not get out of hand. So instead of just marking and sweeping, a ‘memory defrag’ process also happens during garbage collection. This process relocates all the reachable objects next to each other, eliminating (or reducing) the fragmentation. Here is an illustration of that:


要避免这类问题,JVM 必须确保碎片问题不失控。因此在垃圾收集过程中, 不仅仅是标记和清除, 还需要执行 “内存碎片整理” 过程。这个过程让所有可达对象(reachable objects)依次排列, 以消除(或减少)碎片。示意图如下所示:


![](02_01_fragmented-vs-compacted-heap.png)





Generational Hypothesis

分代假设(Generational Hypothesis)


As we have mentioned before, doing a garbage collection entails stopping the application completely. It is also quite obvious that the more objects there are the longer it takes to collect all the garbage. But what if we would have a possibility to work with smaller memory regions? Investigating the possibilities, a group of researchers has observed that most allocations inside applications fall into two categories:

我们前面提到过,执行垃圾收集需要停止整个应用。很明显,对象越多则收集所有垃圾消耗的时间就越长。但可不可以只处理一个较小的内存区域呢? 为了探究这种可能性,研究人员发现,程序中的大多数可回收的内存可归为两类:


- Most of the objects become unused quickly

- The ones that do not usually survive for a (very) long time


- 大部分对象很快就不再使用

- 还有一部分不会立即无用,但也不会持续(太)长时间



These observations come together in the Weak Generational Hypothesis. Based on this hypothesis, the memory inside the VM is divided into what is called the Young Generation and the Old Generation. The latter is sometimes also called Tenured.

这些观测形成了 **弱代假设**(Weak Generational Hypothesis)。基于这一假设, VM中的内存被分为**年轻代(Young Generation)**和**老年代(Old Generation)**。老年代有时候也称为年老区(Tenured)。

![](02_02_object-age-based-on-GC-gen-hypothesis.png)


Having such separate and individually cleanable areas allows for a multitude of different algorithms that have come a long way in improving the performance of the GC.

拆分为这样两个可清理的单独区域，允许采用不同的算法来大幅提高GC的性能。


This is not to say there are no issues with such an approach. For one, objects from different generations may in fact have references to each other that also count as ‘de facto’ GC roots when collecting a generation.

这种方法也不是没有问题。例如，在不同分代中的对象可能会互相引用, 在收集某一个分代时就会成为 "事实上的" GC root。


But most importantly, the generational hypothesis may in fact not hold for some applications. Since the GC algorithms are optimized for objects which either ‘die young’ or ‘are likely to live forever’, the JVM behaves rather poorly with objects with ‘medium’ life expectancy.

当然,要着重强调的是,分代假设并不适用于所有程序。因为GC算法专门针对“要么死得快”，“否则活得长” 这类特征的对象来进行优化, JVM对收集那种存活时间半长不长的对象就显得非常尴尬了。


Memory Pools

内存池(Memory Pools)


The following division of memory pools within the heap should be familiar. What is not so commonly understood is how Garbage Collection performs its duties within the different memory pools. Notice that in different GC algorithms some implementation details might vary but, again, the concepts in this chapter remain effectively the same.

堆内存中的内存池划分也是类似的。不太容易理解的地方在于各个内存池中的垃圾收集是如何运行的。请注意,不同的GC算法在实现细节上可能会有所不同,但和本章所介绍的相关概念都是一致的。


![](02_03_java-heap-eden-survivor-old.png)



Eden

新生代(Eden,伊甸园)


Eden is the region in memory where the objects are typically allocated when they are created. As there are typically multiple threads creating a lot of objects simultaneously, Eden is further divided into one or more Thread Local Allocation Buffer (TLAB for short) residing in the Eden space. These buffers allow the JVM to allocate most objects within one thread directly in the corresponding TLAB, avoiding the expensive synchronization with other threads.

Eden 是内存中的一个区域, 用来分配新创建的对象。通常会有多个线程同时创建多个对象, 所以 Eden 区被划分为多个 线程本地分配缓冲区(Thread Local Allocation Buffer, 简称TLAB)。通过这种缓冲区划分,大部分对象直接由JVM 在对应线程的TLAB中分配, 避免与其他线程的同步操作。



When allocation inside a TLAB is not possible (typically because there’s not enough room there), the allocation moves on to a shared Eden space. If there’s not enough room in there either, a garbage collection process in Young Generation is triggered to free up more space. If the garbage collection also does not result in sufficient free memory inside Eden, then the object is allocated in the Old Generation.

如果 TLAB 中没有足够的内存空间, 就会在共享Eden区(shared Eden space)之中分配。如果共享Eden区也没有足够的空间, 就会触发一次 年轻代GC 来释放内存空间。如果GC之后 Eden 区依然没有足够的空闲内存区域, 则对象就会被分配到老年代空间(Old Generation)。



When Eden is being collected, GC walks all the reachable objects from the roots and marks them as alive.

当 Eden 区进行垃圾收集时, GC将所有从 root 可达的对象过一遍, 并标记为存活对象。



We have previously noted that objects can have cross-generational links so a straightforward approach would have to check all the references from other generations to Eden. Doing so would unfortunately defeat the whole point of having generations in the first place. The JVM has a trick up its sleeve: card-marking. Essentially, the JVM just marks the rough location of ‘dirty’ objects in Eden that may have links to them from the Old Generation. You can read more on that in Nitsan’s blog entry.

我们曾指出,对象间可能会有跨代的引用, 所以需要一种方法来标记从其他分代中指向Eden的所有引用。这样做又会遭遇各个分代之间一遍又一遍的引用。JVM在实现时采用了一些绝招: 卡片标记(card-marking)。从本质上讲,JVM只需要记住Eden区中 “脏”对象的粗略位置, 可能有老年代的对象引用指向这部分区间。更多细节请参考: [Nitsan 的博客](http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html) 中深入了解。


![](02_04_TLAB-in-Eden-memory.png)



After the marking phase is completed, all the live objects in Eden are copied to one of the Survivor spaces. The whole Eden is now considered to be empty and can be reused to allocate more objects. Such an approach is called “Mark and Copy”: the live objects are marked, and then copied (not moved) to a survivor space.

标记阶段完成后, Eden中所有存活的对象都会被复制到存活区(Survivor spaces)里面。整个Eden区就可以被认为是空的, 然后就能用来分配新对象。这种方法叫做 “标记-复制”(Mark and Copy): 存活的对象被标记, 然后拷贝到一个存活区(注意,是复制,而不是移动)。


<br/>
## !!!!!!!!!!!!1校对到此处
<br/>


Survivor Spaces

存活区空间


Next to the Eden space reside two Survivor spaces called from and to. It is important to notice that one of the two Survivor spaces is always empty.

Eden 区的旁边是两个存活区, 称为 from 空间和 to 空间。重要的是请注意, 两个存活区空间中总有一个是空闲的(empty)。



The empty Survivor space will start having residents next time the Young generation gets collected. All of the live objects from the whole of the Young generation (that includes both the Eden space and the non-empty ‘from’ Survivor space) are copied to the ‘to’ survivor space. After this process has completed, ‘to’ now contains objects and ‘from’ does not. Their roles are switched at this time.

空闲的那个存活区是下一次年轻代垃圾收集时的存放空间。年轻代()(包括伊甸园空间和非空”从“幸存者空间)中的所有存活对象都会被复制到 ”to“ 存活区。这个过程完成后, ”to“ 区包含对象而 'from' 区没有。两者的角色进行调换。


![](02_05_how-java-gc-works.png)



This process of copying the live objects between the two Survivor spaces is repeated several times until some objects are considered to have matured and are ‘old enough’. Remember that, based on the generational hypothesis, objects which have survived for some time are expected to continue to be used for very long time.

在两个存活区之间拷贝对象的过程会重复多次, 直到某些对象存活的时间已经足够 “老了”。请记住,分代假设认为, 存活超过一定次数的对象会继续存活更长时间。


Such ‘tenured’ objects can thus be promoted to the Old Generation. When this happens, objects are not moved from one survivor space to another but instead to the Old space, where they will reside until they become unreachable.

这类“终身”的对象可以被提升到老年代。这种情况下, 存活区的对象不是被移动到另一个存活区,而是迁移到老年代, 在老年代一直驻留, 直到变为不可达对象。


To determine whether the object is ‘old enough’ to be considered ready for propagation to Old space, GC tracks the number of collections a particular object has survived. After each generation of objects finishes with a GC, those still alive have their age incremented. Whenever the age exceeds a certain tenuring threshold the object will be promoted to Old space.

为了判断一个对象是否“足够老”, 可以提升(Promotion)到老年代，GC跟踪每个特定对象存活的次数。每次分代GC完成后,那些仍然存活的对象的年龄就会递增。当年龄超过一定的阈值, 对象将被提升到老年代空间。


The actual tenuring threshold is dynamically adjusted by the JVM, but specifying -XX:+MaxTenuringThreshold sets an upper limit on it. Setting -XX:+MaxTenuringThreshold=0 results in immediate promotion without copying it between Survivor spaces. By default, this threshold on modern JVMs is set to 15 GC cycles. This is also the maximum value in HotSpot.

实际的提升阈值是由JVM动态调整的,但可以手动指定 `-XX:+MaxTenuringThreshold` 作为上限。设置 `-XX:+MaxTenuringThreshold=0` 的结果是不在存活区之间拷贝，直接提升到老年代。在现代 JVM 中这个阈值默认设置为**15**个 GC周期。在HotSpot中这也是最大值。


Promotion may also happen prematurely if the size of the Survivor space is not enough to hold all of the live objects in the Young generation.

如果存活区空间不足以存放年轻代中的存活对象，提升(Promotion)也可能发生更早地发生。


Old Generation

老年代(Old Generation)




The implementation for the Old Generation memory space is much more complex. Old Generation is usually significantly larger and is occupied by objects that are less likely to be garbage.

老年代内存空间的GC实现要复杂得多。老年代通常会越来越大，其中的对象可能是垃圾的概率也越来越小。



GC in the Old Generation happens less frequently than in the Young Generation. Also, since most objects are expected to be alive in the Old Generation, there is no Mark and Copy happening. Instead, the objects are moved around to minimize fragmentation. The algorithms cleaning the Old space are generally built on different foundations. In principle, the steps taken go through the following:

老年代GC发生的频率比年轻代小很多。同时, 因为在老年代中的大多数对象都是存活的, 所以就没有标记和复制(Mark and Copy)。相反,对象会被移动以实现内存碎片最小化。清理老年代空间的算法通常是建立在不同的基础上的。原则上,会采取以下这些步骤:



- Mark reachable objects by setting the marked bit next to all objects accessible through GC roots

- Delete all unreachable objects

- Compact the content of old space by copying the live objects contiguously to the beginning of the Old space

- 通过标志位(marked bit) 标记所有通过 GC roots 可访问的对象.

- 删除所有不可达对象

- 整理老年代空间中的内容，将所有存活对象连续地复制到老年代空间开始的地方。


As you can see from the description, GC in Old Generation has to deal with explicit compacting to avoid excessive fragmentation.

正如你所看到的描述, 老年代GC必须明确地进行整理,以避免碎片过多。


PermGen

永久代(PermGen)



Prior to Java 8 there existed a special space called the ‘Permanent Generation’. This is where the metadata such as classes would go. Also, some additional things like internalized strings were kept in Permgen. It actually used to create a lot of trouble to Java developers, since it is quite hard to predict how much space all of that would require. Result of these failed predictions took the form of java.lang.OutOfMemoryError: Permgen space. Unless the cause of such OutOfMemoryError was an actual memory leak, the way to fix this problem was to simply increase the permgen size similar to the following example setting the maximum allowed permgen size to 256 MB: 

在Java 8之前存在一个特殊的空间,叫做“永久代”(Permanent Generation)。这是元数据(metadata)存放的区域,如 class 信息等。此外,其他一些额外的信息也存放在这个区域中, 例如内部化的字符串(internalized strings)等。实际上这给Java开发者造成了很多麻烦,因为很难预测到底需要占用多少空间。这些失败的预测导致的就是 `java.lang.OutOfMemoryError: Permgen space`这种形式的错误。除非导致 OutOfMemoryError 的原因确实是内存泄漏,否则就只需要增加 permgen 的大小，例如下面的示例就是设置 permgen 最大允许的空间为 256 MB:


	java -XX:MaxPermSize=256m com.mycompany.MyApplication



Metaspace

元数据区(Metaspace)



As predicting the need for metadata was a complex and inconvenient exercise, the Permanent Generation was removed in Java 8 in favor of the Metaspace. From this point on, most of the miscellaneous things were moved to regular Java heap.

预测元数据需要多大的空间是一个很复杂的事, 所以在Java 8之中删除了永久代(Permanent Generation)，改用了 Metaspace。从此以后, 很多杂七杂八的东西都被移动到了常规的Java堆里面。


The class definitions, however, are now loaded into something called Metaspace. It is located in the native memory and does not interfere with the regular heap objects. By default, Metaspace size is only limited by the amount of native memory available to the Java process. This saves developers from a situation when adding just one more class to the application results in the java.lang.OutOfMemoryError: Permgen space. Notice that having such seemingly unlimited space does not ship without costs – letting the Metaspace to grow uncontrollably you can introduce heavy swapping and/or reach native allocation failures instead.

当然，类定义(class definitions)现在会被加载到 Metaspace 里面。它位于本地内存(native memory),不再影响到常规的对象。默认情况下, Metaspace的大小只由Java进程可用的本地内存大小限制。这让程序员不再因为加载了几个类就导致 `java.lang.OutOfMemoryError: Permgen space. ` 这种问题。注意, 这种看似无限的空间也不是没有成本—— 如果 Metaspace 增长失控则可能会导致很严重的内存交换(swapping) 或者 引起本地内存分配失败。


In case you still wish to protect yourself for such occasions you can limit the growth of Metaspace similar to following, limiting Metaspace size to 256 MB:

如果你仍然希望在这样的场合保护机器，那可以使用下面这样的方式来限制 Metaspace 的增长, 如 256 MB:


	java -XX:MaxMetaspaceSize=256m com.mycompany.MyApplication



Minor GC vs Major GC vs Full GC

对比: Minor GC vs Major GC vs Full GC

> Minor GC(小型GC,小型GC) - 大型GC(Major GC) - 以及完全GC(Full GC)



The Garbage Collection events cleaning out different parts inside heap memory are often called Minor, Major and Full GC events. In this section we cover the differences between these events. Along the way we can hopefully see that this distinction is actually not too relevant.

清理堆内存中不同部分的垃圾收集事件(Garbage Collection events)通常称为: 小型GC(Minor GC) - 大型GC(Major GC) - 和完全GC(Full GC) 事件。本节介绍这些事件的区别。在此过程 我们可以看到这些区别并不是完全相关。


What typically is relevant is whether the application meets its SLAs, and to see that you monitor your application for latency or throughput. And only then are GC events linked to the results. What is important about these events is whether they stopped the application and how long it took.

而且你可以通过监控程序的延迟和吞吐量看到, 是否满足SLA(Service Level Agreement，服务水平协议)和这个是有关系的。也只有那时候GC事件才关联到其结果。重要的是这些事件是否停止整个程序,以及耗费多长时间。


But as the terms Minor, Major and Full GC are widely used and without a proper definition, let us look into the topic in a bit more detail.

虽然 Minor, Major and Full GC 这些术语被广泛应用, 也没有标准的定义, 但我们还是来深入地了解一下这个话题吧。


Minor GC

小型GC


Collecting garbage from the Young space is called Minor GC. This definition is both clear and uniformly understood. But there are still some interesting takeaways you should be aware of when dealing with Minor Garbage Collection events:

收集年轻代内存空间的事件叫做小型GC。这个定义既清晰又得到广泛共识。但在处理小型GC事件时有一些是有趣的事情你应该了解一下:


Minor GC is always triggered when the JVM is unable to allocate space for a new object, e.g. Eden is getting full. So the higher the allocation rate, the more frequently Minor GC occurs.

Minor GC总是在JVM无法为新对象分配内存空间时触发,如 Eden 区占满。所以新对象分配频率越高, Minor GC 就越频繁。


During a Minor GC event, Tenured Generation is effectively ignored. References from Tenured Generation to Young Generation are considered to be GC roots. References from Young Generation to Tenured Generation are simply ignored during the mark phase.

Minor GC 事件实际上忽略老年代。从老年代指向年轻代的引用都被认为是GC Root。从年轻代指向老年代的引用在标记阶段被完全忽略。


Against common belief, Minor GC does trigger stop-the-world pauses, suspending the application threads. For most applications, the length of the pauses is negligible latency-wise if most of the objects in the Eden can be considered garbage and are never copied to Survivor/Old spaces. If the opposite is true and most of the newborn objects are not eligible for collection, Minor GC pauses start taking considerably more time.

与一般的认识相反, Minor GC 每次都会触发全线停顿(stop-the-world ), 暂停所有的应用程序线程。对大多数程序而言, 如果 Eden 区的对象基本上都是垃圾, 也不怎么拷贝到存活区/老年代的话，那么暂停时长是可以忽略的。如果情况不是这样, 大多数新创建对象不被垃圾回收清理掉, 那么 Minor GC的停顿就会花费更多时间。


So defining Minor GC is easy – Minor GC cleans the Young Generation.

所以 Minor GC 的定义很简单 —— Minor GC 清理的是年轻代。


Major GC vs Full GC

对比: Major GC vs Full GC



It should be noted that there are no formal definitions for those terms – neither in the JVM specification nor in the Garbage Collection research papers. But on the first glance, building these definitions on top of what we know to be true about Minor GC cleaning Young space should be simple:

值得一提的是, 这些术语并没有正式的定义 —— 无论是在JVM规范还是在GC相关的论文中。我们知道,  Minor GC是清理年轻代空间的，对应的 Major GC 定义应该也是很简单的:


Major GC is cleaning the Old space.

大型GC(Major GC) 清理的是老年代空间(Old space)。


Full GC is cleaning the entire Heap – both Young and Old spaces.

完整GC(Full GC)清理的是整个堆, 包括年轻代和老年代空间。


Unfortunately it is a bit more complex and confusing. To start with – many Major GCs are triggered by Minor GCs, so separating the two is impossible in many cases. On the other hand – modern garbage collection algorithms like G1 perform partial garbage cleaning so, again, using the term ‘cleaning’ is only partially correct.

杯具的是又有更复杂和混乱的了。很多 Major GC 是由 Minor GC触发的, 所以在很多情况下分离这两者是不可能的。另一方面, 像G1这样的现代垃圾收集算法执行的是部分垃圾清理, 所以，额，使用术语“cleaning”并不是完全的准确。



This leads us to the point where instead of worrying about whether the GC is called Major or Full GC, you should focus on finding out whether the GC at hand stopped all the application threads or was able to progress concurrently with the application threads.

这也教育我们,不要再去操心是应该叫 Major GC 呢还是应该叫 Full GC, 你应该考虑的是 这次GC 是停止所有线程呢还是与其他线程一起执行的呢。


This confusion is even built right into the JVM standard tools. What I mean by that is best explained via an example. Let us compare the output of two different tools tracing the GC on a JVM running with Concurrent Mark and Sweep collector (-XX:+UseConcMarkSweepGC)


这种混淆甚至根植于标准的JVM工具中。我的意思是最好通过实例来说明。让我们来比较跟踪使用 同一个JVM中GC信息的两个不同工具的输出吧。这个JVM使用的是**并发标记和清除收集器*（Concurrent Mark and Sweep collector，`-XX:+UseConcMarkSweepGC`).



First attempt is to get the insight via the jstat output:

首先我们来看 `jstat` 的输出:


> `jstat -gc -t 4235 1s`



	Time S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
	 5.7 34048.0 34048.0  0.0   34048.0 272640.0 194699.7 1756416.0   181419.9  18304.0 17865.1 2688.0 2497.6      3    0.275   0      0.000    0.275
	 6.7 34048.0 34048.0 34048.0  0.0   272640.0 247555.4 1756416.0   263447.9  18816.0 18123.3 2688.0 2523.1      4    0.359   0      0.000    0.359
	 7.7 34048.0 34048.0  0.0   34048.0 272640.0 257729.3 1756416.0   345109.8  19072.0 18396.6 2688.0 2550.3      5    0.451   0      0.000    0.451
	 8.7 34048.0 34048.0 34048.0 34048.0 272640.0 272640.0 1756416.0  444982.5  19456.0 18681.3 2816.0 2575.8      7    0.550   0      0.000    0.550
	 9.7 34048.0 34048.0 34046.7  0.0   272640.0 16777.0  1756416.0   587906.3  20096.0 19235.1 2944.0 2631.8      8    0.720   0      0.000    0.720
	10.7 34048.0 34048.0  0.0   34046.2 272640.0 80171.6  1756416.0   664913.4  20352.0 19495.9 2944.0 2657.4      9    0.810   0      0.000    0.810
	11.7 34048.0 34048.0 34048.0  0.0   272640.0 129480.8 1756416.0   745100.2  20608.0 19704.5 2944.0 2678.4     10    0.896   0      0.000    0.896
	12.7 34048.0 34048.0  0.0   34046.6 272640.0 164070.7 1756416.0   822073.7  20992.0 19937.1 3072.0 2702.8     11    0.978   0      0.000    0.978
	13.7 34048.0 34048.0 34048.0  0.0   272640.0 211949.9 1756416.0   897364.4  21248.0 20179.6 3072.0 2728.1     12    1.087   1      0.004    1.091
	14.7 34048.0 34048.0  0.0   34047.1 272640.0 245801.5 1756416.0   597362.6  21504.0 20390.6 3072.0 2750.3     13    1.183   2      0.050    1.233
	15.7 34048.0 34048.0  0.0   34048.0 272640.0 21474.1  1756416.0   757347.0  22012.0 20792.0 3200.0 2791.0     15    1.336   2      0.050    1.386
	16.7 34048.0 34048.0 34047.0  0.0   272640.0 48378.0  1756416.0   838594.4  22268.0 21003.5 3200.0 2813.2     16    1.433   2      0.050    1.484




This snippet is extracted from the first 17 seconds after the JVM was launched. Based on this information we could conclude that after 12 Minor GC runs two Full GC runs were performed, spanning 50ms in total. You would get the same confirmation via GUI-based tools, such as the jconsole or jvisualvm.

这个片段是从JVM启动后的第17秒开始截取的。根据这些信息我们可以得出这样的结论: 经过12次 Minor GC之后执行了2次Full GC, 总计耗时 50ms。通过基于GUI的工具也会得出相同的结论, 比如 jconsole 或者 jvisualvm (或者最新的 jmc)。


Before nodding at this conclusion, let’s look at the output of the garbage collection logs gathered from the same JVM launch. Apparently -XX:+PrintGCDetails tells us a different and a more detailed story:

在同意这个结论之前, 让我们看看从同一个JVM进程收集到的GC日志。显然 `-XX:+PrintGCDetails` 讲述的是另外一个更详细的故事:


> `java -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC eu.plumbr.demo.GarbageProducer`

	3.157: [GC (Allocation Failure) 3.157: [ParNew: 272640K->34048K(306688K), 0.0844702 secs] 272640K->69574K(2063104K), 0.0845560 secs] [Times: user=0.23 sys=0.03, real=0.09 secs] 
	4.092: [GC (Allocation Failure) 4.092: [ParNew: 306688K->34048K(306688K), 0.1013723 secs] 342214K->136584K(2063104K), 0.1014307 secs] [Times: user=0.25 sys=0.05, real=0.10 secs] 
	... cut for brevity ...
	
	11.292: [GC (Allocation Failure) 11.292: [ParNew: 306686K->34048K(306688K), 0.0857219 secs] 971599K->779148K(2063104K), 0.0857875 secs] [Times: user=0.26 sys=0.04, real=0.09 secs] 
	12.140: [GC (Allocation Failure) 12.140: [ParNew: 306688K->34046K(306688K), 0.0821774 secs] 1051788K->856120K(2063104K), 0.0822400 secs] [Times: user=0.25 sys=0.03, real=0.08 secs] 
	12.989: [GC (Allocation Failure) 12.989: [ParNew: 306686K->34048K(306688K), 0.1086667 secs] 1128760K->931412K(2063104K), 0.1087416 secs] [Times: user=0.24 sys=0.04, real=0.11 secs] 
	13.098: [GC (CMS Initial Mark) [1 CMS-initial-mark: 897364K(1756416K)] 936667K(2063104K), 0.0041705 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
	13.102: [CMS-concurrent-mark-start]
	13.341: [CMS-concurrent-mark: 0.238/0.238 secs] [Times: user=0.36 sys=0.01, real=0.24 secs] 
	13.341: [CMS-concurrent-preclean-start]
	13.350: [CMS-concurrent-preclean: 0.009/0.009 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
	13.350: [CMS-concurrent-abortable-preclean-start]
	13.878: [GC (Allocation Failure) 13.878: [ParNew: 306688K->34047K(306688K), 0.0960456 secs] 1204052K->1010638K(2063104K), 0.0961542 secs] [Times: user=0.29 sys=0.04, real=0.09 secs] 
	14.366: [CMS-concurrent-abortable-preclean: 0.917/1.016 secs] [Times: user=2.22 sys=0.07, real=1.01 secs] 
	14.366: [GC (CMS Final Remark) [YG occupancy: 182593 K (306688 K)]14.366: [Rescan (parallel) , 0.0291598 secs]14.395: [weak refs processing, 0.0000232 secs]14.395: [class unloading, 0.0117661 secs]14.407: [scrub symbol table, 0.0015323 secs]14.409: [scrub string table, 0.0003221 secs][1 CMS-remark: 976591K(1756416K)] 1159184K(2063104K), 0.0462010 secs] [Times: user=0.14 sys=0.00, real=0.05 secs] 
	14.412: [CMS-concurrent-sweep-start]
	14.633: [CMS-concurrent-sweep: 0.221/0.221 secs] [Times: user=0.37 sys=0.00, real=0.22 secs] 
	14.633: [CMS-concurrent-reset-start]
	14.636: [CMS-concurrent-reset: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]



Based on this information we can see that after 12 Minor GC runs ‘something different’ indeed started happening. But instead of two Full GC runs, this ‘different thing’ was in reality just a single GC running in Old generation and consisting of different phases:

根据这些信息我们可以看到,在12 次 Minor GC之后发生了一些 "不同的事情"。不是两个 Full GC, 而是只在老年代执行了单次 GC, 并且由多个不同的阶段组成:




Initial Mark phase, spanning for 0.0041705 seconds or approximately 4ms. This phase is a stop-the-world event stopping all application threads for initial marking.

初始标记阶段,跨越大约0.0041705秒或4 ms。这个阶段是一个停止一切事件停止所有应用程序线程初始标记。


Markup and Preclean phases. were executed concurrently with the application threads.

标记和预清理阶段(Markup and Preclean)。是和应用程序线程并发执行的。


Final Remark phase, spanning for 0.0462010 seconds or approximately 46ms. This phase is again stop-the-world event.

最终评价阶段(Final Remark), 持续大约 0.0462010秒, 约 46ms。这一阶段又是一次全线停工事件(stop-the-world event)。



Sweep operation was executed concurrently, without stopping the application threads.

清除操作是并发执行的, 没有停止应用程序线程。


So what we see from the actual garbage collection logs is that, instead of two Full GC operations, just one Major GC cleaning Old space was actually executed.

所以我们从实际的垃圾收集日志看到, 并不是两个 Full GC操作,实际上只执行了一次清理老年代空间的 Major GC 。


If you were after latency then the data revealed by jstat would have led you towards correct decisions. It correctly listed the two stop-the-world events totaling 50ms affecting the latency for all the active threads at that very moment. But if you were trying to optimize for throughput, you would have been misguided – listing just the stop-the-world initial mark and final remark phases, the jstat output completely hides the concurrent work being done.


如果程序有延迟，那么使用 jstat 显示的数据也能让你得出正确的结果。它正确地列出了两次  stop-the-world 事件总计 50 ms。所有活动线程在那一刻受到了影响发生延迟。但如果你需要优化吞吐量, 你就会被误导 —— 清单只显示了 stop-the-world 的初始标记阶段和最终评价阶段, 而 jstat 的输出完全隐藏了其他并发执行的GC过程。


原文链接: [Garbage Collection in Java](https://plumbr.eu/handbook/garbage-collection-in-java)



<div style="page-break-after : always;"> </div>


