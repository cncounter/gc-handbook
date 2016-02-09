# 3. GC 算法(基础篇)


Before diving into the practical implementation details of Garbage Collection algorithms it will be beneficial to define the required terminology and to understand the basic principles supporting the implementations. Specific details vary from collector to collector but in general all collectors focus in two areas

在深入实际的垃圾收集算法实现细节前, 先过一遍相关的术语定义以及理解其基本原则是很有用的。具体的细节每一款垃圾收集器(collector)都不一样,但总的来说所有垃圾收集器都集中在两个领域:


- find out all objects that are still alive
- get rid of everything else – the supposedly dead and unused objects.

- 找出所有存活对象
- 丢弃其余部分(get rid of everything else) -- 所谓的死对象和不使用的对象。




First part, the census on live objects, is implemented in all collectors with the help of a process called Marking.

第一部分, 统计(census)存活的对象, 所有垃圾收集器都会使用一个叫做 **标记(Marking)** 的过程。



标记可达对象(Marking Reachable Objects)



Every modern GC algorithm used in JVM starts its job with finding out all objects that are still alive. This concept is best explained using the following picture representing your JVM’s memory layout:

现代JVM中的所有GC算法都是从找出所有存活的对象开始。下图通过JVM的内存布局将这个概念做了最好的诠释:


![](03_01_Java-GC-mark-and-sweep.png)




First, GC defines some specific objects as Garbage Collection Roots. Examples of such GC roots are:

首先,GC将一些特定的对象定义为垃圾收集的根元素(Garbage Collection Roots)。GC根元素的例子:


- Local variable and input parameters of the currently executing methods

- Active threads

- Static field of the loaded classes

- JNI references

- 当前正在执行方法中的局部变量和输入参数
- 活动线程(Active threads)
- 所有加载到内存中的类的静态字段(static field)
- JNI引用


Next, GC traverses the whole object graph in your memory, starting from those Garbage Collection Roots and following references from the roots to other objects, e.g. instance fields. Every object the GC visits is marked as alive.


接下来,GC遍历(traverses)内存中的整个对象图(object graph),从GC根元素以及根元素的直接引用(如对象的属性域)开始,。GC访问得到的所有对象都被标记为存活状态。



Live objects are represented as blue on the picture above. When the marking phase finishes, every live object is marked. All other objects (grey data structures on the picture above) are thus unreachable from the GC roots, implying that your application cannot use the unreachable objects anymore. Such objects are considered garbage and GC should get rid of them in the following phases.

存活对象在上图中表示为蓝色。当标记阶段完成, 所有存活对象都被标记了。其他对象(上图中灰色的数据结构)都是从GC根元素不可达的,也就意味着程序不可能再使用这些不可达对象了。这样的对象被认为是垃圾, GC应该在接下来的阶段中会去除他们。




There are important aspects to note about the marking phase:

在标记阶段有几个需要注意的点:


The application threads need to be stopped for the marking to happen as you cannot really traverse the graph if it keeps changing under your feet all the time. Such a situation when the application threads are temporarily stopped so that the JVM can indulge in housekeeping activities is called a safe point resulting in a Stop The World pause. Safe points can be triggered for different reasons but garbage collection is by far the most common reason for a safe point to be introduced.


标记阶段需要暂停应用程序的线程, 因为如果一直在变化就不可能遍历整个对象图。全线停顿(Stop The World pause)时线程可以暂停的点叫做安全点(safe point), 此时, JVM就可以专心执行清理工作。安全点可能因多种原因触发, 但目前为止GC是产生安全点最常见的原因。



The duration of this pause depends neither on the total number of objects in heap nor on the size of the heap but on the number of alive objects. So increasing the size of the heap does not directly affect the duration of the marking phase.


此阶段暂停的时间跟堆内存的大小，堆中对象总数都没关系， 而是由存活的对象数量决定。所以增加堆内存的大小并不会直接影响标记阶段的持续时间。



When the mark phase is completed, the GC can proceed to the next step and start removing the unreachable objects.


标记阶段完成时,GC可以继续下一步,开始删除不可达的对象。



Removing Unused Objects

删除不使用的对象(Removing Unused Objects)


Removal of unused objects is somewhat different for different GC algorithms but all such GC algorithms can be divided into three groups: sweeping, compacting and copying. Next sections will discuss each of such algorithms in more detail.


不同的GC算法在删除不使用对象时有所不同, 但GC算法可分为三类: 清除(sweeping)、整理(compacting,压实)和复制(copying)。下一节将详细讨论这些算法。



Sweep

清除(Sweep)


Mark and Sweep algorithms use conceptually the simplest approach to garbage by just ignoring such objects. What this means is that after the marking phase has completed all space occupied by unvisited objects is considered free and can thus be reused to allocate new objects.




标记和清除算法(Mark and Sweep)概念上使用最简单的方法: 就是直接忽略垃圾对象。这意味着在标记阶段完成后所有不可访问对象所占用的空间都被认为是空闲的,因此可以重新用来分配新对象。




The approach requires using the so called free-list recording of every free region and its size. The management of the free-lists adds overhead to object allocation. Built into this approach is another weakness – there may exist plenty of free regions but if no single region is large enough to accommodate the allocation, the allocation is still going to fail (with an OutOfMemoryError in Java).


这种方法需要使用空闲列表(free-list)来记录所有的空闲区域及每一个的大小。管理空闲列表增加了对象分配时的开销。这种方法还有另一个弱点 —— 虽然存在很多空闲的空间, 但可能没有一个区域能够存放下需要分配的对象,进而导致分配失败(在Java 中伴随着 OutOfMemoryError)。



![](03_02_GC-sweep.png)





Compact

整理(Compact, 压实)


Mark-Sweep-Compact algorithms solve the shortcomings of Mark and Sweep by moving all marked – and thus alive – objects to the beginning of the memory region. The downside of this approach is an increased GC pause duration as we need to copy all objects to a new place and to update all references to such objects. The benefits to Mark and Sweep are also visible – after such a compacting operation new object allocation is again extremely cheap via pointer bumping. Using such approach the location of the free space is always known and no fragmentation issues are triggered either.


标记-清除-整理算法(Mark-Sweep-Compact) 解决了标记和清除算法的缺点: 将所有标记为存活对象移动到内存区域开始的地方。这种方法的缺点是增加了GC暂停的时间, 因为我们需要将所有对象复制到一个新地方,并更新所有指向这些对象的引用。标记和清扫的好处也很明显, 经过整理之后新对象的分配就很简单, 只需要指针增加(pointer bumping)就行。使用这种方法则可用的空闲内存始终是已知的,也就不再引起碎片问题了。



![](03_03_GC-mark-sweep-compact.png)





Copy

复制(Copy, 拷贝)


Mark and Copy algorithms are very similar to the Mark and Compact as they too relocate all live objects. The important difference is that the target of relocation is a different memory region as a new home for survivors. Mark and Copy approach has some advantages as copying can occur simultaneously with marking during the same phase. The disadvantage is the need for one more memory region, which should be large enough to accommodate survived objects.


标记和复制算法(Mark and Copy) 和 标记-整理算法(Mark and Compact) 非常相似，因为两者都会迁移所有存活对象。最重要的区别在于, 标记-复制算法迁移到的位置是一个不同的内存区域: 存活区。标记-复制方法也有一些优点: 标记与复制阶段可以同时进行。缺点是需要另一个额外的内存区域, 要大到存放下所有的存活对象。



![](03_04_GC-mark-and-copy-in-Java.png)





原文链接: [GC Algorithms: Basics](https://plumbr.eu/handbook/garbage-collection-algorithms)



