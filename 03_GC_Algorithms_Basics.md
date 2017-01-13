# 3. GC 算法(基础篇)

> **相关术语翻译说明**: 
>
> Mark,标记;
>
> Sweep,清除; 
>
> Compact,整理; 也有人翻译为压缩,译者认为GC时不存在压缩这回事。
>
> Copy,复制; copy 用作名词时一般翻译为拷贝/副本,用作动词时翻译为复制。
>
> 注: 《[垃圾回收算法手册](https://book.douban.com/subject/26740958/)》将 Mark and Sweep 翻译为: **标记-清扫**算法; 译者认为 **标记-清除** 更容易理解。


Before diving into the practical implementation details of Garbage Collection algorithms it will be beneficial to define the required terminology and to understand the basic principles supporting the implementations. Specific details vary from collector to collector but in general all collectors focus in two areas

本章简要介绍GC的基本原理和相关技术, [下一章节](04_GC_Algorithms_Implementations.md)再详细讲解GC算法的具体实现。各种垃圾收集器的实现细节虽然并不相同,但总体而言,垃圾收集器都专注于两件事情:


- find out all objects that are still alive
- get rid of everything else – the supposedly dead and unused objects.

- 查找所有存活对象
- 抛弃其他的部分,即死对象,不再使用的对象。




First part, the census on live objects, is implemented in all collectors with the help of a process called **Marking**.

第一步, 记录(census)所有的存活对象, 在垃圾收集中有一个叫做 **标记(Marking)** 的过程专门干这件事。



## 标记可达对象(Marking Reachable Objects)



Every modern GC algorithm used in JVM starts its job with finding out all objects that are still alive. This concept is best explained using the following picture representing your JVM’s memory layout:

现代JVM中所有的GC算法,第一步都是找出所有存活的对象。下图是JVM的内存布局,对这一步做了最好的诠释:


![](03_01_Java-GC-mark-and-sweep.png)




First, GC defines some specific objects as **Garbage Collection Roots**. Examples of such GC roots are:

首先,有一些特定的对象被指定为 **Garbage Collection Roots**(GC根元素)。包括:


- Local variable and input parameters of the currently executing methods
- Active threads
- Static field of the loaded classes
- JNI references

- 当前正在执行的方法里的局部变量和输入参数
- 活动线程(Active threads)
- 内存中所有类的静态字段(static field)
- JNI引用


Next, GC traverses the whole object graph in your memory, starting from those Garbage Collection Roots and following references from the roots to other objects, e.g. instance fields. Every object the GC visits is **marked** as alive.


其次, GC遍历(traverses)内存中整体的对象关系图(object graph),从GC根元素开始扫描, 到直接引用，以及其他对象(通过对象的属性域)。所有GC访问到的对象都被**标记(marked)**为存活对象。



Live objects are represented as blue on the picture above. When the marking phase finishes, every live object is marked. All other objects (grey data structures on the picture above) are thus unreachable from the GC roots, implying that your application cannot use the unreachable objects anymore. Such objects are considered garbage and GC should get rid of them in the following phases.


存活对象在上图中用蓝色表示。标记阶段完成后, 所有存活对象都被标记了。而其他对象(上图中灰色的数据结构)就是从GC根元素不可达的, 也就是说程序不能再使用这些不可达的对象(unreachable object)。这样的对象被认为是垃圾, GC会在接下来的阶段中清除他们。




There are important aspects to note about the marking phase:

在标记阶段有几个需要注意的点:



The application threads need to be stopped for the marking to happen as you cannot really traverse the graph if it keeps changing under your feet all the time. Such a situation when the application threads are temporarily stopped so that the JVM can indulge in housekeeping activities is called a **safe point** resulting in a **Stop The World pause**. Safe points can be triggered for different reasons but garbage collection is by far the most common reason for a safe point to be introduced.


在标记阶段,需要暂停所有应用线程, 以遍历所有对象的引用关系。因为不暂停就没法跟踪一直在变化的引用关系图。这种情景叫做 **Stop The World pause** (**全线停顿**),而可以安全地暂停线程的点叫做安全点(safe point), 然后, JVM就可以专心执行清理工作。安全点可能有多种因素触发, 当前, GC是触发安全点最常见的原因。



The duration of this pause depends neither on the total number of objects in heap nor on the size of the heap but on the number of **alive objects**. So increasing the size of the heap does not directly affect the duration of the marking phase.


此阶段暂停的时间, 与堆内存大小,对象的总数没有直接关系, 而是由**存活对象**(alive objects)的数量来决定。所以增加堆内存的大小并不会直接影响标记阶段占用的时间。



When the **mark** phase is completed, the GC can proceed to the next step and start removing the unreachable objects.


**标记** 阶段完成后, GC进行下一步操作, 删除不可达对象。



## 删除不可达对象(Removing Unused Objects)


Removal of unused objects is somewhat different for different GC algorithms but all such GC algorithms can be divided into three groups: sweeping, compacting and copying. Next sections will discuss each of such algorithms in more detail.


各种GC算法在删除不可达对象时略有不同, 但总体可分为三类: 清除(sweeping)、整理(compacting)和复制(copying)。[下一章节](04_GC_Algorithms_Implementations.md)将详细讲解这些算法。



### Sweep

### Sweep(清除)


**Mark and Sweep** algorithms use conceptually the simplest approach to garbage by just ignoring such objects. What this means is that after the marking phase has completed all space occupied by unvisited objects is considered free and can thus be reused to allocate new objects.




**Mark and Sweep(标记-清除)** 算法的概念非常简单: 直接忽略所有的垃圾。也就是说在标记阶段完成后, 所有不可达对象占用的内存空间, 都被认为是空闲的, 因此可以用来分配新对象。




The approach requires using the so called **free-list** recording of every free region and its size. The management of the free-lists adds overhead to object allocation. Built into this approach is another weakness – there may exist plenty of free regions but if no single region is large enough to accommodate the allocation, the allocation is still going to fail (with an [OutOfMemoryError](http://plumbr.eu/outofmemoryerror) in Java).


这种算法需要使用 **空闲表(free-list)**,来记录所有的空闲区域, 以及每个区域的大小。维护空闲表增加了对象分配时的开销。此外还存在另一个弱点 —— 明明还有很多空闲内存, 却可能没有一个区域的大小能够存放需要分配的对象, 从而导致分配失败(在Java 中就是 [OutOfMemoryError](http://plumbr.eu/outofmemoryerror))。



![](03_02_GC-sweep.png)



### Compact(整理)


**Mark-Sweep-Compact** algorithms solve the shortcomings of Mark and Sweep by moving all marked – and thus alive – objects to the beginning of the memory region. The downside of this approach is an increased GC pause duration as we need to copy all objects to a new place and to update all references to such objects. The benefits to Mark and Sweep are also visible – after such a compacting operation new object allocation is again extremely cheap via pointer bumping. Using such approach the location of the free space is always known and no fragmentation issues are triggered either.


**标记-清除-整理算法(Mark-Sweep-Compact)**, 将所有被标记的对象(存活对象), 迁移到内存空间的起始处, 消除了标记-清除算法的缺点。 相应的缺点就是GC暂停时间会增加, 因为需要将所有对象复制到另一个地方, 然后修改指向这些对象的引用。此算法的优势也很明显, 碎片整理之后, 分配新对象就很简单, 只需要通过指针碰撞(pointer bumping)即可。使用这种算法, 内存空间剩余的容量一直是清楚的, 不会再导致内存碎片问题。



![](03_03_GC-mark-sweep-compact.png)





### Copy

### Copy(复制)


**Mark and Copy** algorithms are very similar to the Mark and Compact as they too relocate all live objects. The important difference is that the target of relocation is a different memory region as a new home for survivors. Mark and Copy approach has some advantages as copying can occur simultaneously with marking during the same phase. The disadvantage is the need for one more memory region, which should be large enough to accommodate survived objects.


**标记-复制算法(Mark and Copy)** 和 标记-整理算法(Mark and Compact) 十分相似: 两者都会移动所有存活的对象。区别在于, 标记-复制算法是将内存移动到另外一个空间: 存活区。标记-复制方法的优点在于:  标记和复制可以同时进行。缺点则是需要一个额外的内存区间, 来存放所有的存活对象。



![](03_04_GC-mark-and-copy-in-Java.png)





原文链接: [GC Algorithms: Basics](https://plumbr.eu/handbook/garbage-collection-algorithms)





<div style="page-break-after : always;"> </div>


