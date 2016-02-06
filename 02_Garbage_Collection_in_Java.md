# 2. Java的垃圾收集


The introduction to Mark and Sweep Garbage Collection is a mostly theoretical one. When things come to practice, numerous adjustments need to be done to accommodate for real-world scenarios and needs. For a simple example, let us take a look at what sorts of bookkeeping the JVM needs to do so that we can safely continue allocating objects.


标记-清扫算法是垃圾收集的一个主要理论。当理论需要实践, 很多东西需要做调整,以适应现实场景和需求。最简单的例子,让我们用小本本记下JVM需要做哪些事, 才能够安全地继续分配对象存储空间。



Fragmenting and Compacting

碎片与压缩(Fragmenting and Compacting)



Whenever sweeping takes place, the JVM has to make sure the areas filled with unreachable objects can be reused. This can (and eventually will) lead to memory fragmentation which, similarly to disk fragmentation, leads to two problems:


每发生一次清除(sweeping), JVM 必须确保不可达对象所占用的内存区域可以被重用。这可能(最终)产生内存碎片(类似于磁盘碎片的东西),并导致两个问题:


- Write operations become more time-consuming as finding the next free block of sufficient size is no longer a trivial operation.

- When creating new objects, JVM is allocating memory in contiguous blocks. So if fragmentation escalates to a point where no individual free fragment is large enough to accommodate the newly created object, an allocation error occurs.

- 写操作变得更加耗时, 因为寻找下一个足够大的空闲内存块不再是一个简单的操作。

- 在创建新对象时,JVM在相邻的块分配内存。如果碎片问题升级到没有任何一个空闲的片段来容纳新创建的对象,就会发生内存分配错误(allocation error)。



To avoid such problems, the JVM is making sure the fragmenting does not get out of hand. So instead of just marking and sweeping, a ‘memory defrag’ process also happens during garbage collection. This process relocates all the reachable objects next to each other, eliminating (or reducing) the fragmentation. Here is an illustration of that:


为了避免这些问题,JVM 必须确保碎片问题不失控。因此不仅仅是标记和清除, 一个“内存碎片整理”的过程也需要在垃圾收集过程中执行。这个过程让所有可访问的对象一个挨着一个,以消除(或减少)的碎片。示意图如下:


![](02_01_fragmented-vs-compacted-heap.png)



Generational Hypothesis

分代假说(Generational Hypothesis)


As we have mentioned before, doing a garbage collection entails stopping the application completely. It is also quite obvious that the more objects there are the longer it takes to collect all the garbage. But what if we would have a possibility to work with smaller memory regions? Investigating the possibilities, a group of researchers has observed that most allocations inside applications fall into two categories:

正如我们之前提到过的,执行垃圾收集需要完全停止应用程序。很明显,对象越多则收集所有垃圾的时间就越长。但可不可以只收集处理一个较小的内存区域呢?为了调查这种可能性,研究人员观察到,程序中的大多数内部分配可以归为两类:


- Most of the objects become unused quickly

- The ones that do not usually survive for a (very) long time


- 大多数对象很快就不再使用

- 有一部分并不会(很)长时间地生存



These observations come together in the Weak Generational Hypothesis. Based on this hypothesis, the memory inside the VM is divided into what is called the Young Generation and the Old Generation. The latter is sometimes also called Tenured.

这些观察形成了 **弱代假设**(Weak Generational Hypothesis)。基于这一假设, VM中的内存被分为 年轻代(Young Generation)和老年代(Old Generation)。老年代有时候也称为年老区(Tenured)。

![](02_02_object-age-based-on-GC-gen-hypothesis.png)


Having such separate and individually cleanable areas allows for a multitude of different algorithms that have come a long way in improving the performance of the GC.

分为这样两个单独的清理区域，允许各自采用不同的算法来大幅度改善GC的性能。


This is not to say there are no issues with such an approach. For one, objects from different generations may in fact have references to each other that also count as ‘de facto’ GC roots when collecting a generation.

这并不是说这种方法没有问题。例如，在不同分代中的对象有可能相互引用, 在收集某一个分代时也可以算是“实际上的”GC root。


But most importantly, the generational hypothesis may in fact not hold for some applications. Since the GC algorithms are optimized for objects which either ‘die young’ or ‘are likely to live forever’, the JVM behaves rather poorly with objects with ‘medium’ life expectancy.

但最重要的是,分代假说可能并不适用于某些程序。因为GC算法是针对“要么英年早逝”，“要么得到永生”的对象来进行优化的, JVM有时候对那种活到半途的对象就显得黔驴技穷了。


Memory Pools

内存池


The following division of memory pools within the heap should be familiar. What is not so commonly understood is how Garbage Collection performs its duties within the different memory pools. Notice that in different GC algorithms some implementation details might vary but, again, the concepts in this chapter remain effectively the same.

堆内存中内存池的划分也是类似的。不太容易理解的是在不同的内存池中垃圾收集是如何进行的。请注意,不同的GC算法在实现细节上可能会有所不同,但相关概念都和本章所介绍的是一致的。


![](02_03_java-heap-eden-survivor-old.png)



Eden

新生儿(Eden,伊甸园)


Eden is the region in memory where the objects are typically allocated when they are created. As there are typically multiple threads creating a lot of objects simultaneously, Eden is further divided into one or more Thread Local Allocation Buffer (TLAB for short) residing in the Eden space. These buffers allow the JVM to allocate most objects within one thread directly in the corresponding TLAB, avoiding the expensive synchronization with other threads.

Eden 是内存中的一个区域, 通常新创建的对象都是在这个区域分配内存。通常会有多个线程同时创建多个对象, 所以 Eden 一般会被进一步划分成一到多个线程本地分配缓冲区(Thread Local Allocation Buffer, 简称TLAB)。这些缓冲区允许JVM直接在线程对应的TLAB中分配大部分对象, 避免昂贵的同步操作。



When allocation inside a TLAB is not possible (typically because there’s not enough room there), the allocation moves on to a shared Eden space. If there’s not enough room in there either, a garbage collection process in Young Generation is triggered to free up more space. If the garbage collection also does not result in sufficient free memory inside Eden, then the object is allocated in the Old Generation.

如果不能继续在 TLAB 中分配内存(通常因为没有足够的空间),就会在共享Eden空间(shared Eden space)中分配。如果还没有足够的空间, 就会触发一次 年轻代垃圾收集过程, 以释放更多的空间。如果垃圾收集后在 Eden 区还没有足够的空闲内存, 那么对象就会被分配到老年代之中。



When Eden is being collected, GC walks all the reachable objects from the roots and marks them as alive.

在 Eden 区垃圾回收时, GC将所有从 root 可达的对象过一遍, 并标记为存活对象。



We have previously noted that objects can have cross-generational links so a straightforward approach would have to check all the references from other generations to Eden. Doing so would unfortunately defeat the whole point of having generations in the first place. The JVM has a trick up its sleeve: card-marking. Essentially, the JVM just marks the rough location of ‘dirty’ objects in Eden that may have links to them from the Old Generation. You can read more on that in Nitsan’s blog entry.

我们曾指出,对象可能会存在跨代引用, 所以需要一个简单的方法来检查所有从其他分代中指向Eden的引用。这样做又会遭遇各个分代之间一遍又一遍的引用。JVM在小包包里面有一件法宝: 卡片标记(card-marking)。从本质上讲,JVM只需要记住Eden区中 “脏”对象的粗略位置,可能有老年代的引用指向这些地方。你可以从 [Nitsan 的博客](http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html) 中深入了解。


![](02_04_TLAB-in-Eden-memory.png)



After the marking phase is completed, all the live objects in Eden are copied to one of the Survivor spaces. The whole Eden is now considered to be empty and can be reused to allocate more objects. Such an approach is called “Mark and Copy”: the live objects are marked, and then copied (not moved) to a survivor space.

标记阶段完成后, Eden中所有存活的对象都会被复制到存活区(Survivor spaces)中的一个。整个Eden接着被认为是空的,可以用来分配其他新对象。这种方法被称为“标记-复制”(Mark and Copy): 存活的对象被标记, 然后复制到一个存活区空间(注意,是复制,不是移动)。


Survivor Spaces

存活区空间


Next to the Eden space reside two Survivor spaces called from and to. It is important to notice that one of the two Survivor spaces is always empty.

Eden 区的旁边是两个存活区, 称为 from 空间和 to 空间。重要的是请注意, 两个存活区空间中总有一个是空闲的(empty)。



The empty Survivor space will start having residents next time the Young generation gets collected. All of the live objects from the whole of the Young generation (that includes both the Eden space and the non-empty ‘from’ Survivor space) are copied to the ‘to’ survivor space. After this process has completed, ‘to’ now contains objects and ‘from’ does not. Their roles are switched at this time.

空闲的那个存活区是下一次年轻代垃圾收集时的存放空间。年轻代()(包括伊甸园空间和非空”从“幸存者空间)中的所有存活对象都会被复制到 ”to“ 存活区。这个过程完成后, ”to“ 区包含对象而 'from' 区没有。两者的角色进行调换。


![](02_05_how-java-gc-works.png)



This process of copying the live objects between the two Survivor spaces is repeated several times until some objects are considered to have matured and are ‘old enough’. Remember that, based on the generational hypothesis, objects which have survived for some time are expected to continue to be used for very long time.

在两个存活区之间拷贝对象的过程会重复多次, 直到某些对象存活的时间已经足够 “老了”。请记住,分代假说认为, 存活超过一定次数的对象会继续存活更长时间。


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

- 压缩老年代空间中的内容，将所有存活对象连续地复制到老年代空间开始的地方。


As you can see from the description, GC in Old Generation has to deal with explicit compacting to avoid excessive fragmentation.

正如你所看到的描述, 老年代GC必须明确地进行压缩,以避免过度的碎片。


PermGen

永久代(PermGen)



Prior to Java 8 there existed a special space called the ‘Permanent Generation’. This is where the metadata such as classes would go. Also, some additional things like internalized strings were kept in Permgen. It actually used to create a lot of trouble to Java developers, since it is quite hard to predict how much space all of that would require. Result of these failed predictions took the form of java.lang.OutOfMemoryError: Permgen space. Unless the cause of such OutOfMemoryError was an actual memory leak, the way to fix this problem was to simply increase the permgen size similar to the following example setting the maximum allowed permgen size to 256 MB: 

在Java 8之前存在一个特殊的空间,叫做“永久代”(Permanent Generation)。这是元数据(metadata)存放的区域,如 class 信息等。此外,其他一些额外的信息也存放在这个区域中, 例如内部化的字符串(internalized strings)等。实际上这给Java开发者造成了很多麻烦,因为很难预测到底需要占用多少空间。这些失败的预测导致的就是 `java.lang.OutOfMemoryError: Permgen space`这种形式的错误。除非导致 OutOfMemoryError 的原因确实是内存泄漏,否则就只需要增加 permgen 的大小，例如下面的示例就是设置 permgen 最大允许的空间为 256 MB:


	java -XX:MaxPermSize=256m com.mycompany.MyApplication




## -------------------------------------------------------
## 到这里
## -------------------------------------------------------


Metaspace

元数据区(Metaspace)



As predicting the need for metadata was a complex and inconvenient exercise, the Permanent Generation was removed in Java 8 in favor of the Metaspace. From this point on, most of the miscellaneous things were moved to regular Java heap.

预测需要元数据是一个复杂的和不方便运动,永久的一代是在Java 8赞成Metaspace中删除。从这一点上,大多数的杂七杂八的东西都搬到普通Java堆。


The class definitions, however, are now loaded into something called Metaspace. It is located in the native memory and does not interfere with the regular heap objects. By default, Metaspace size is only limited by the amount of native memory available to the Java process. This saves developers from a situation when adding just one more class to the application results in the java.lang.OutOfMemoryError: Permgen space. Notice that having such seemingly unlimited space does not ship without costs – letting the Metaspace to grow uncontrollably you can introduce heavy swapping and/or reach native allocation failures instead.

类定义,然而,现在加载到Metaspace。它位于本机内存,不干扰正常堆对象。默认情况下,Metaspace本机内存的大小是由数量有限的可用的Java进程。这节省了开发人员的情况只是一个类添加到应用程序导致. lang。OutOfMemoryError:Permgen空间。注意到有这样看似无限的空间不船没有成本——让Metaspace增长失控你可以介绍沉重的交换和/或达到本地分配失败。


In case you still wish to protect yourself for such occasions you can limit the growth of Metaspace similar to following, limiting Metaspace size to 256 MB:

如果你仍然希望保护自己在这样的场合可以限制Metaspace相似的增长后,限制Metaspace大小为256 MB:


java -XX:MaxMetaspaceSize=256m com.mycompany.MyApplication



Minor GC vs Major GC vs Full GC

小主要GC和GC和GC


The Garbage Collection events cleaning out different parts inside heap memory are often called Minor, Major and Full GC events. In this section we cover the differences between these events. Along the way we can hopefully see that this distinction is actually not too relevant.

垃圾收集事件清理内部不同部分堆内存通常被称为小,主要和完整GC事件。在本节中,我们介绍这些事件之间的区别。一路上我们可以希望看到这种区别是不太相关。


What typically is relevant is whether the application meets its SLAs, and to see that you monitor your application for latency or throughput. And only then are GC events linked to the results. What is important about these events is whether they stopped the application and how long it took.

通常是相关的应用程序是否满足sla,并看到你监视应用程序延迟和吞吐量。也只有到那时GC事件与结果。什么是重要的对这些事件是他们是否停止应用程序,用了多长时间。


But as the terms Minor, Major and Full GC are widely used and without a proper definition, let us look into the topic in a bit more detail.

但作为次要条款,主要和完整GC被广泛使用,没有一个适当的定义,让我们更详细地研究这个话题。


Minor GC

小GC


Collecting garbage from the Young space is called Minor GC. This definition is both clear and uniformly understood. But there are still some interesting takeaways you should be aware of when dealing with Minor Garbage Collection events:

收集垃圾从年轻的空间被称为小GC。这个定义既清晰又统一理解。但是仍有一些有趣的外卖你应该意识到在处理小垃圾收集事件:


Minor GC is always triggered when the JVM is unable to allocate space for a new object, e.g. Eden is getting full. So the higher the allocation rate, the more frequently Minor GC occurs.

小GC总是触发当JVM无法为新对象分配空间,如伊甸园越来越完整。所以分配率越高,越频繁发生轻微的GC。


During a Minor GC event, Tenured Generation is effectively ignored. References from Tenured Generation to Young Generation are considered to be GC roots. References from Young Generation to Tenured Generation are simply ignored during the mark phase.

小GC事件期间,终身代实际上是忽视了。引用从终身代年轻代被认为是GC根。引用从年轻代终身代标记阶段完全被忽视。


Against common belief, Minor GC does trigger stop-the-world pauses, suspending the application threads. For most applications, the length of the pauses is negligible latency-wise if most of the objects in the Eden can be considered garbage and are never copied to Survivor/Old spaces. If the opposite is true and most of the newborn objects are not eligible for collection, Minor GC pauses start taking considerably more time.

对付共同的信念,小GC并触发停止一切暂停,暂停应用程序线程。对于大多数应用程序,暂停的长度是可以忽略的latency-wise如果伊甸园中大部分的对象可以被认为是垃圾,从来没有被复制到幸存者/旧空间。如果正好相反,大多数新生的对象是没有资格获得收集、小GC暂停开始花更多时间。


So defining Minor GC is easy – Minor GC cleans the Young Generation.

所以定义小GC是容易的——小GC清洁年轻代。


Major GC vs Full GC

主要GC和GC


It should be noted that there are no formal definitions for those terms – neither in the JVM specification nor in the Garbage Collection research papers. But on the first glance, building these definitions on top of what we know to be true about Minor GC cleaning Young space should be simple:

应该注意的是,没有正式的定义这些术语——无论是在JVM规范还是在垃圾收集研究论文。第一眼,之上构建这些定义我们所知道的是真实的小GC清洁小空间应该是简单的:


Major GC is cleaning the Old space.

主要的GC是打扫旧空间。


Full GC is cleaning the entire Heap – both Young and Old spaces.

完整GC清洗整个堆,无论是年轻的或年老的空间。


Unfortunately it is a bit more complex and confusing. To start with – many Major GCs are triggered by Minor GCs, so separating the two is impossible in many cases. On the other hand – modern garbage collection algorithms like G1 perform partial garbage cleaning so, again, using the term ‘cleaning’ is only partially correct.

不幸的是这是一个更加复杂和混乱。开始——许多主要GCs由小GCs,所以分离两个在很多情况下是不可能的。另一方面,现代垃圾收集算法像G1执行部分垃圾清理,再一次,使用术语“清洗”只是部分正确的。



This leads us to the point where instead of worrying about whether the GC is called Major or Full GC, you should focus on finding out whether the GC at hand stopped all the application threads or was able to progress concurrently with the application threads.

这引导我们,而不是担心是否GC被称为主要或全部GC,你应该专注于找出是否手头的GC停止了所有应用程序并发线程或能够进展与应用程序线程。


This confusion is even built right into the JVM standard tools. What I mean by that is best explained via an example. Let us compare the output of two different tools tracing the GC on a JVM running with Concurrent Mark and Sweep collector (-XX:+UseConcMarkSweepGC)


这种混淆甚至到JVM标准构建工具。我的意思是最好通过一个例子来解释。让我们比较两个不同的工具的输出跟踪GC在JVM运行并发标记和清扫收集器(- xx:+ UseConcMarkSweepGC)



First attempt is to get the insight via the jstat output:

第一次尝试通过jstat获得洞察力输出:


my-precious: me$ jstat -gc -t 4235 1s



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

这个片段是提取后的第一个17秒JVM启动。根据这些信息我们可以得出这样的结论:经过12小GC运行两个完整GC运行进行,跨越50微秒。你会得到相同的确认通过基于gui的工具,如jconsole或jvisualvm。


Before nodding at this conclusion, let’s look at the output of the garbage collection logs gathered from the same JVM launch. Apparently -XX:+PrintGCDetails tells us a different and a more detailed story:

在点头这个结论之前,让我们看看收集的垃圾收集日志的输出相同JVM启动。显然- xx:+ PrintGCDetails告诉我们一个不同的和更详细的故事:


java -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC eu.plumbr.demo.GarbageProducer

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

根据这些信息我们可以看到,经过12小GC运行确实不同的事情发生。而是两个完整GC运行时,这种“不同”实际上只是一个GC运行在老年代和组成不同的阶段:


Initial Mark phase, spanning for 0.0041705 seconds or approximately 4ms. This phase is a stop-the-world event stopping all application threads for initial marking.

初始标记阶段,跨越大约0.0041705秒或4 ms。这个阶段是一个停止一切事件停止所有应用程序线程初始标记。


Markup and Preclean phases. were executed concurrently with the application threads.

标记和Preclean阶段。与应用程序并发执行线程。


Final Remark phase, spanning for 0.0462010 seconds or approximately 46ms. This phase is again stop-the-world event.

最后评价阶段,跨越大约0.0462010秒46女士。这一阶段再次停止一切活动。


Sweep operation was executed concurrently, without stopping the application threads.

扫描操作并发执行,没有停止应用程序线程。


So what we see from the actual garbage collection logs is that, instead of two Full GC operations, just one Major GC cleaning Old space was actually executed.

所以我们看到实际的垃圾收集日志,而不是两个完整GC操作,只是一个主要GC清洗旧空间实际上是执行。


If you were after latency then the data revealed by jstat would have led you towards correct decisions. It correctly listed the two stop-the-world events totaling 50ms affecting the latency for all the active threads at that very moment. But if you were trying to optimize for throughput, you would have been misguided – listing just the stop-the-world initial mark and final remark phases, the jstat output completely hides the concurrent work being done.


如果你延迟后的数据显示jstat会引导你走向正确的决定。它正确地列出了两个停止一切事件总计50毫秒的延迟影响所有活动线程的那一刻。但是如果你试图优化吞吐量,你会一直在误导——清单只是停止一切初始马克和最终评价阶段,jstat输出完全隐藏了并发工作被做。


原文链接: [Garbage Collection in Java](https://plumbr.eu/handbook/garbage-collection-in-java)

