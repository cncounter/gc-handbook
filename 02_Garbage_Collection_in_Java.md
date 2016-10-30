# 2. Java垃圾收集简介


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

我们曾指出,对象间可能会有跨代的引用, 所以需要一种方法来标记从其他分代中指向Eden的所有引用。这样做又会遭遇各个分代之间一遍又一遍的引用。JVM在实现时采用了一些绝招: 卡片标记(card-marking)。从本质上讲,JVM只需要记住Eden区中 “脏”对象的粗略位置, 可能有老年代的对象引用指向这部分区间。更多细节请参考: [Nitsan 的博客](http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html) 。


![](02_04_TLAB-in-Eden-memory.png)



After the marking phase is completed, all the live objects in Eden are copied to one of the Survivor spaces. The whole Eden is now considered to be empty and can be reused to allocate more objects. Such an approach is called “Mark and Copy”: the live objects are marked, and then copied (not moved) to a survivor space.

标记阶段完成后, Eden中所有存活的对象都会被复制到存活区(Survivor spaces)里面。整个Eden区就可以被认为是空的, 然后就能用来分配新对象。这种方法称为 “标记-复制”(Mark and Copy): 存活的对象被标记, 然后拷贝到一个存活区(注意,是复制,而不是移动)。



Survivor Spaces

存活区(Survivor Spaces)


Next to the Eden space reside two Survivor spaces called from and to. It is important to notice that one of the two Survivor spaces is always empty.

Eden 区的旁边是两个存活区, 称为 from 空间和 to 空间。需要着重强调的的是, 其中总有一个存活区是空的(empty)。



The empty Survivor space will start having residents next time the Young generation gets collected. All of the live objects from the whole of the Young generation (that includes both the Eden space and the non-empty ‘from’ Survivor space) are copied to the ‘to’ survivor space. After this process has completed, ‘to’ now contains objects and ‘from’ does not. Their roles are switched at this time.

空的那个存活区用于在下一次年轻代GC时存放收集的对象。年轻代中所有的存活对象(包括Edenq区和非空的那个 "from" 存活区)都会被复制到 ”to“ 存活区。GC过程完成后, ”to“ 区有对象,而 'from' 区里没有对象。两者的角色进行正好切换 。


![](02_05_how-java-gc-works.png)



This process of copying the live objects between the two Survivor spaces is repeated several times until some objects are considered to have matured and are ‘old enough’. Remember that, based on the generational hypothesis, objects which have survived for some time are expected to continue to be used for very long time.

存活的对象会在两个存活区之间拷贝多次, 直到某些对象的存活 时间达到一定的阀值。分代理论假设, 存活超过一定时间的对象很可能会继续存活更长时间。


Such ‘tenured’ objects can thus be promoted to the Old Generation. When this happens, objects are not moved from one survivor space to another but instead to the Old space, where they will reside until they become unreachable.

这类“ 年老” 的对象因此被提升到老年代。提升的时候， 存活区的对象不再是拷贝到另一个存活区,而是迁移到老年代, 并在老年代一直驻留, 直到变为不可达对象。


To determine whether the object is ‘old enough’ to be considered ready for propagation to Old space, GC tracks the number of collections a particular object has survived. After each generation of objects finishes with a GC, those still alive have their age incremented. Whenever the age exceeds a certain tenuring threshold the object will be promoted to Old space.

为了确定一个对象是否“足够老”, 可以被提升(Promotion)到老年代，GC模块跟踪记录每个存活区对象存活的次数。每次分代GC完成后,存活对象的年龄就会增长。当年龄超过提升阈值, 就会被提升到老年代区域。


The actual tenuring threshold is dynamically adjusted by the JVM, but specifying -XX:+MaxTenuringThreshold sets an upper limit on it. Setting -XX:+MaxTenuringThreshold=0 results in immediate promotion without copying it between Survivor spaces. By default, this threshold on modern JVMs is set to 15 GC cycles. This is also the maximum value in HotSpot.

具体的提升阈值由JVM动态调整,但也可以用参数 `-XX:+MaxTenuringThreshold` 来指定上限。如果设置 `-XX:+MaxTenuringThreshold=0` , 则GC时存活对象不在存活区之间拷贝，直接提升到老年代。现代 JVM 中这个阈值默认设置为**15**个 GC周期。这也是HotSpot中的最大值。


Promotion may also happen prematurely if the size of the Survivor space is not enough to hold all of the live objects in the Young generation.

如果存活区空间不够存放年轻代中的存活对象，提升(Promotion)也可能更早地进行。


Old Generation

老年代(Old Generation)




The implementation for the Old Generation memory space is much more complex. Old Generation is usually significantly larger and is occupied by objects that are less likely to be garbage.

老年代的GC实现要复杂得多。老年代内存空间通常会更大，里面的对象是垃圾的概率也更小。



GC in the Old Generation happens less frequently than in the Young Generation. Also, since most objects are expected to be alive in the Old Generation, there is no Mark and Copy happening. Instead, the objects are moved around to minimize fragmentation. The algorithms cleaning the Old space are generally built on different foundations. In principle, the steps taken go through the following:

老年代GC发生的频率比年轻代小很多。同时, 因为预期老年代中的对象大部分是存活的, 所以不再使用标记和复制(Mark and Copy)算法。而是采用移动对象的方式来实现最小化内存碎片。老年代空间的清理算法通常是建立在不同的基础上的。原则上,会执行以下这些步骤:



- Mark reachable objects by setting the marked bit next to all objects accessible through GC roots

- Delete all unreachable objects

- Compact the content of old space by copying the live objects contiguously to the beginning of the Old space

- 通过标志位(marked bit),标记所有通过 GC roots 可达的对象.

- 删除所有不可达对象

- 整理老年代空间中的内容，方法是将所有的存活对象复制,从老年代空间开始的地方,依次存放。


As you can see from the description, GC in Old Generation has to deal with explicit compacting to avoid excessive fragmentation.

通过上面的描述可知, 老年代GC必须明确地进行整理,以避免内存碎片过多。


PermGen

永久代(PermGen)



Prior to Java 8 there existed a special space called the ‘Permanent Generation’. This is where the metadata such as classes would go. Also, some additional things like internalized strings were kept in Permgen. It actually used to create a lot of trouble to Java developers, since it is quite hard to predict how much space all of that would require. Result of these failed predictions took the form of java.lang.OutOfMemoryError: Permgen space. Unless the cause of such OutOfMemoryError was an actual memory leak, the way to fix this problem was to simply increase the permgen size similar to the following example setting the maximum allowed permgen size to 256 MB: 

在Java 8 之前有一个特殊的空间,称为“永久代”(Permanent Generation)。这是存储元数据(metadata)的地方,比如 class 信息等。此外,这个区域中也保存有其他的数据和信息, 包括 内部化的字符串(internalized strings)等等。实际上这给Java开发者造成了很多麻烦,因为很难去计算这块区域到底需要占用多少内存空间。预测失败导致的结果就是产生 `java.lang.OutOfMemoryError: Permgen space` 这种形式的错误。除非 ·OutOfMemoryError· 确实是内存泄漏导致的,否则就只能增加 permgen 的大小，例如下面的示例，就是设置 permgen 最大空间为 256 MB:


	java -XX:MaxPermSize=256m com.mycompany.MyApplication



Metaspace

元数据区(Metaspace)



As predicting the need for metadata was a complex and inconvenient exercise, the Permanent Generation was removed in Java 8 in favor of the Metaspace. From this point on, most of the miscellaneous things were moved to regular Java heap.

既然估算元数据所需空间那么复杂, Java 8直接删除了永久代(Permanent Generation)，改用 Metaspace。从此以后, Java 中很多杂七杂八的东西都放置到普通的堆内存里。


The class definitions, however, are now loaded into something called Metaspace. It is located in the native memory and does not interfere with the regular heap objects. By default, Metaspace size is only limited by the amount of native memory available to the Java process. This saves developers from a situation when adding just one more class to the application results in the java.lang.OutOfMemoryError: Permgen space. Notice that having such seemingly unlimited space does not ship without costs – letting the Metaspace to grow uncontrollably you can introduce heavy swapping and/or reach native allocation failures instead.

当然，像类定义(class definitions)之类的信息会被加载到 Metaspace 中。元数据区位于本地内存(native memory),不再影响到普通的Java对象。默认情况下, Metaspace的大小只受限于 Java进程可用的本地内存。这样程序就不再因为多加载了几个类/JAR包就导致 `java.lang.OutOfMemoryError: Permgen space. ` 。注意, 这种不受限制的空间也不是没有代价的 —— 如果 Metaspace 失控, 则可能会导致很严重的内存交换(swapping), 或者导致本地内存分配失败。


In case you still wish to protect yourself for such occasions you can limit the growth of Metaspace similar to following, limiting Metaspace size to 256 MB:

如果需要避免这种最坏情况，那么可以通过下面这样的方式来限制 Metaspace 的大小, 如 256 MB:


	java -XX:MaxMetaspaceSize=256m com.mycompany.MyApplication



Minor GC vs Major GC vs Full GC

小型GC-大型GC-完全GC事件


The Garbage Collection events cleaning out different parts inside heap memory are often called Minor, Major and Full GC events. In this section we cover the differences between these events. Along the way we can hopefully see that this distinction is actually not too relevant.

垃圾收集事件(Garbage Collection events)通常分为: 小型GC(Minor GC) - 大型GC(Major GC) - 和完全GC(Full GC) 事件。本节介绍这些事件及其区别。然后你会发现这些区别也不是特别清晰。


What typically is relevant is whether the application meets its SLAs, and to see that you monitor your application for latency or throughput. And only then are GC events linked to the results. What is important about these events is whether they stopped the application and how long it took.

最重要的是,应用程序是否满足 服务级别协议(Service Level Agreement, SLA), 并通过监控程序查看响应延迟和吞吐量。也只有那时候才能看到GC事件相关的结果。重要的是这些事件是否停止整个程序,以及持续多长时间。


But as the terms Minor, Major and Full GC are widely used and without a proper definition, let us look into the topic in a bit more detail.

虽然 Minor, Major 和 Full GC 这些术语被广泛应用, 但并没有标准的定义, 我们还是来深入了解一下具体的细节吧。


Minor GC

小型GC(Minor GC)


Collecting garbage from the Young space is called Minor GC. This definition is both clear and uniformly understood. But there are still some interesting takeaways you should be aware of when dealing with Minor Garbage Collection events:

年轻代内存的垃圾收集事件称为小型GC。这个定义既清晰又得到广泛共识。对于小型GC事件,有一些有趣的事情你应该了解一下:


Minor GC is always triggered when the JVM is unable to allocate space for a new object, e.g. Eden is getting full. So the higher the allocation rate, the more frequently Minor GC occurs.

当JVM无法为新对象分配内存空间时总会触发 Minor GC,比如 Eden 区占满时。所以(新对象)分配频率越高, Minor GC 的频率就越高。


During a Minor GC event, Tenured Generation is effectively ignored. References from Tenured Generation to Young Generation are considered to be GC roots. References from Young Generation to Tenured Generation are simply ignored during the mark phase.

Minor GC 事件实际上忽略了老年代。从老年代指向年轻代的引用都被认为是GC Root。而从年轻代指向老年代的引用在标记阶段全部被忽略。


Against common belief, Minor GC does trigger stop-the-world pauses, suspending the application threads. For most applications, the length of the pauses is negligible latency-wise if most of the objects in the Eden can be considered garbage and are never copied to Survivor/Old spaces. If the opposite is true and most of the newborn objects are not eligible for collection, Minor GC pauses start taking considerably more time.

与一般的认识相反, Minor GC 每次都会引起全线停顿(stop-the-world ), 暂停所有的应用线程。对大多数程序而言,暂停时长基本上是可以忽略不计的, 因为 Eden 区的对象基本上都是垃圾, 也不怎么拷贝到存活区/老年代。如果情况不是这样, 大部分新创建的对象不能被垃圾回收清理掉, 则 Minor GC的停顿就会持续更长的时间。


So defining Minor GC is easy – Minor GC cleans the Young Generation.

所以 Minor GC 的定义很简单 —— Minor GC 清理的就是年轻代。


Major GC vs Full GC

Major GC 与 Full GC 对比



It should be noted that there are no formal definitions for those terms – neither in the JVM specification nor in the Garbage Collection research papers. But on the first glance, building these definitions on top of what we know to be true about Minor GC cleaning Young space should be simple:

值得一提的是, 这些术语并没有正式的定义 —— 无论是在JVM规范还是在GC相关论文中。

我们知道,  Minor GC 清理的是年轻代空间(Young space)，相应的,其他定义也很简单:


Major GC is cleaning the Old space.

Major GC(大型GC) 清理的是老年代空间(Old space)。
wq234

Full GC is cleaning the entire Heap – both Young and Old spaces.

完整GC(Full GC)清理的是整个堆, 包括年轻代和老年代空间。


Unfortunately it is a bit more complex and confusing. To start with – many Major GCs are triggered by Minor GCs, so separating the two is impossible in many cases. On the other hand – modern garbage collection algorithms like G1 perform partial garbage cleaning so, again, using the term ‘cleaning’ is only partially correct.

杯具的是更复杂的情况出现了。很多 Major GC 是由 Minor GC 触发的, 所以很多情况下这两者是不可分离的。另一方面, 像G1这样的垃圾收集算法执行的是部分区域垃圾回收, 所以，额，使用术语“cleaning”并不是非常准确。



This leads us to the point where instead of worrying about whether the GC is called Major or Full GC, you should focus on finding out whether the GC at hand stopped all the application threads or was able to progress concurrently with the application threads.

这也让我们认识到,不应该去操心是叫 Major GC 呢还是叫 Full GC, 我们应该关注的是: 某次GC事件 是否停止所有线程,或者是与其他线程并发执行。


This confusion is even built right into the JVM standard tools. What I mean by that is best explained via an example. Let us compare the output of two different tools tracing the GC on a JVM running with Concurrent Mark and Sweep collector (-XX:+UseConcMarkSweepGC)


这些混淆甚至根植于标准的JVM工具中。我的意思可以通过实例来说明。让我们来对比同一JVM中两款工具的GC信息输出吧。这个JVM使用的是 **并发标记和清除收集器**（Concurrent Mark and Sweep collector，`-XX:+UseConcMarkSweepGC`).



First attempt is to get the insight via the jstat output:

首先我们来看 `jstat` 的输出:


> `jstat -gc -t 4235 1s`



	Time   S0C    S1C     S0U    S1U      EC       EU        OC         OU       MC       MU     CCSC   CCSU     YGC    YGCT    FGC    FGCT     GCT   
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

此片段截取自JVM启动后的前17秒。根据这些信息可以得知: 有2次Full GC在12次Minor GC(YGC)之后触发执行, 总计耗时 50ms。当然,也可以通过具备图形界面的工具得出同样的信息, 比如 jconsole 或者 jvisualvm (或者最新的 jmc)。


Before nodding at this conclusion, let’s look at the output of the garbage collection logs gathered from the same JVM launch. Apparently -XX:+PrintGCDetails tells us a different and a more detailed story:

在下结论之前, 让我们看看此JVM进程的GC日志。显然需要配置 `-XX:+PrintGCDetails` 参数,GC日志的内容更详细,结果也有一些不同:


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

通过GC日志可以看到, 在12 次 Minor GC之后发生了一些 "不同的事情"。并不是两个 Full GC, 而是在老年代执行了一次 GC, 分为多个阶段执行:




Initial Mark phase, spanning for 0.0041705 seconds or approximately 4ms. This phase is a stop-the-world event stopping all application threads for initial marking.

初始标记阶段(Initial Mark phase),耗时 0.0041705秒(约4ms)。此阶段是全线停顿(STW)事件,暂停所有应用线程,以便执行初始标记。


Markup and Preclean phases. were executed concurrently with the application threads.

标记和预清理阶段(Markup and Preclean phase)。和应用线程并发执行。


Final Remark phase, spanning for 0.0462010 seconds or approximately 46ms. This phase is again stop-the-world event.

最终标记阶段(Final Remark phase), 耗时 0.0462010秒(约46ms)。此阶段也是全线停顿(STW)事件。



Sweep operation was executed concurrently, without stopping the application threads.

清除操作是并发执行的, 不需要暂停应用线程。


So what we see from the actual garbage collection logs is that, instead of two Full GC operations, just one Major GC cleaning Old space was actually executed.

所以从实际的GC日志可以看到, 并不是执行了两次 Full GC操作, 而是只执行了一次清理老年代空间的 Major GC 。


If you were after latency then the data revealed by jstat would have led you towards correct decisions. It correctly listed the two stop-the-world events totaling 50ms affecting the latency for all the active threads at that very moment. But if you were trying to optimize for throughput, you would have been misguided – listing just the stop-the-world initial mark and final remark phases, the jstat output completely hides the concurrent work being done.


如果只关心延迟, 通过后面 jstat 显示的数据, 也能得出正确的结果。它正确地列出了两次  STW 事件,总计耗时 50 ms。这段时间影响了所有应用线程的延迟。如果想要优化吞吐量, 这个结果就会有误导性 —— jstat 只列出初始标记阶段和最终标记阶段, 而 jstat 的输出则完全隐藏了并发执行的GC阶段。


原文链接: [Garbage Collection in Java](https://plumbr.eu/handbook/garbage-collection-in-java)



<div style="page-break-after : always;"> </div>


