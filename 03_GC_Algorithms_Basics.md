# 3. GC 算法(基础篇)




Before diving into the practical implementation details of Garbage Collection algorithms it will be beneficial to define the required terminology and to understand the basic principles supporting the implementations. Specific details vary from collector to collector but in general all collectors focus in two areas

在深入实际实现垃圾收集算法的细节,这将有利于定义所需的术语和理解支持实现的基本原则。具体细节不同收集器收藏家,但总的来说所有收藏家集中在两个领域


- find out all objects that are still alive
- get rid of everything else – the supposedly dead and unused objects.

找出所有对象,还活着
——摆脱一切所谓的死和未使用的对象。




First part, the census on live objects, is implemented in all collectors with the help of a process called Marking.

第一部分,普查活动对象,实现在所有收藏家的帮助下这一过程被称为标记。




Marking Reachable Objects

标记可访问的对象



Every modern GC algorithm used in JVM starts its job with finding out all objects that are still alive. This concept is best explained using the following picture representing your JVM’s memory layout:

每个现代GC算法用于JVM开始工作,发现所有的对象都还活着。这个概念是最好的解释使用以下图片代表你的JVM的内存布局:


![](03_01_Java-GC-mark-and-sweep.png)





First, GC defines some specific objects as Garbage Collection Roots. Examples of such GC roots are:

首先,GC将一些特定的对象定义为垃圾收集的根源。这样的GC根的例子:


Local variable and input parameters of the currently executing methods

局部变量和输入参数的当前执行的方法


Active threads

活动线程


Static field of the loaded classes

加载的类的静态字段


JNI references

JNI引用


Next, GC traverses the whole object graph in your memory, starting from those Garbage Collection Roots and following references from the roots to other objects, e.g. instance fields. Every object the GC visits is marked as alive.


接下来,GC遍历整个对象图在你的记忆中,从那些垃圾收集后根和从根部到其他对象的引用,例如实例字段。每个对象GC访问被标记为活着。



Live objects are represented as blue on the picture above. When the marking phase finishes, every live object is marked. All other objects (grey data structures on the picture above) are thus unreachable from the GC roots, implying that your application cannot use the unreachable objects anymore. Such objects are considered garbage and GC should get rid of them in the following phases.

活动对象表示为上图蓝色。当标记阶段完成,每个生活对象是显著的。所有其他对象(灰色数据结构在上图)因此GC根遥不可及的,这意味着您的应用程序不能使用遥不可及的对象了。这样的对象被认为是垃圾和GC应该摆脱他们在接下来的阶段。




There are important aspects to note about the marking phase:

有重要方面需要注意的标记阶段:


The application threads need to be stopped for the marking to happen as you cannot really traverse the graph if it keeps changing under your feet all the time. Such a situation when the application threads are temporarily stopped so that the JVM can indulge in housekeeping activities is called a safe point resulting in a Stop The World pause. Safe points can be triggered for different reasons but garbage collection is by far the most common reason for a safe point to be introduced.


标记的应用程序线程需要停止发生,你不能遍历图如果它改变你的脚下。这种情况下,当应用程序线程暂时停止JVM可以沉溺于家务活动被称为安全点导致暂停停止世界。安全点可以为不同的原因,但触发垃圾收集到目前为止最常见的原因是一个安全点。



The duration of this pause depends neither on the total number of objects in heap nor on the size of the heap but on the number of alive objects. So increasing the size of the heap does not directly affect the duration of the marking phase.


暂停的时间取决于堆上对象的总数和堆的大小,而是活着的对象的数量。所以增加堆的大小并不直接影响的持续时间标记阶段。



When the mark phase is completed, the GC can proceed to the next step and start removing the unreachable objects.


标记阶段完成时,GC可以继续下一步,开始删除的对象。



Removing Unused Objects

删除未使用的对象


Removal of unused objects is somewhat different for different GC algorithms but all such GC algorithms can be divided into three groups: sweeping, compacting and copying. Next sections will discuss each of such algorithms in more detail.


删除未使用的对象不同的GC算法有所不同,但所有这些GC算法可分为三组:扫地、压实和复制。下一个部分将更详细地讨论每一个这样的算法。



Sweep

清除


Mark and Sweep algorithms use conceptually the simplest approach to garbage by just ignoring such objects. What this means is that after the marking phase has completed all space occupied by unvisited objects is considered free and can thus be reused to allocate new objects.




标记和清扫算法概念上使用最简单的方法就可以忽略这样的垃圾对象。这意味着在标记阶段完成后既无对象占用的空间都被认为是免费的,因此可以重用来分配新对象。




The approach requires using the so called free-list recording of every free region and its size. The management of the free-lists adds overhead to object allocation. Built into this approach is another weakness – there may exist plenty of free regions but if no single region is large enough to accommodate the allocation, the allocation is still going to fail (with an OutOfMemoryError in Java).


使用所谓的空闲列表的方法需要记录每一个自由区域及其大小。管理对象分配的自由列表增加了开销。构建到这个方法是另一个弱点——区域可能存在有很多免费的,但是如果没有一个区域足够大来容纳分配,分配仍会失败(在Java OutOfMemoryError)。



![](03_02_GC-sweep.png)





Compact

整理


Mark-Sweep-Compact algorithms solve the shortcomings of Mark and Sweep by moving all marked – and thus alive – objects to the beginning of the memory region. The downside of this approach is an increased GC pause duration as we need to copy all objects to a new place and to update all references to such objects. The benefits to Mark and Sweep are also visible – after such a compacting operation new object allocation is again extremely cheap via pointer bumping. Using such approach the location of the free space is always known and no fragmentation issues are triggered either.


标记-清除-整理算法解决标记和清扫的缺点通过移动所有标记——因此活着——对象的内存区域的开始。这种方法的缺点是增加GC暂停时间,我们需要将所有对象复制到一个新地方,更新所有对这些对象的引用。标记和清扫的好处,也可见,这样一个压实新对象分配行动后再通过指针碰撞非常便宜。使用这种方法的位置自由空间始终是已知的,没有碎片问题也引发了。



![](03_03_GC-mark-sweep-compact.png)





Copy

复制(Copy, 拷贝)


Mark and Copy algorithms are very similar to the Mark and Compact as they too relocate all live objects. The important difference is that the target of relocation is a different memory region as a new home for survivors. Mark and Copy approach has some advantages as copying can occur simultaneously with marking during the same phase. The disadvantage is the need for one more memory region, which should be large enough to accommodate survived objects.


马克和复制算法非常相似的马克和紧凑也搬迁所有活动对象。最重要的区别是,搬迁是一个不同的内存区域的目标作为幸存者一个新家。马克和复制方法有一些优点与标志复制可以同时出现在同一阶段。缺点是需要一个内存区域,应足够大,以适应存活对象。



![](03_04_GC-mark-and-copy-in-Java.png)





原文链接: [GC Algorithms: Basics](https://plumbr.eu/handbook/garbage-collection-algorithms)



