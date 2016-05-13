# 1. 垃圾回收简介


> **说明**: 在本文中, `Garbage Collection` 翻译为 “`垃圾收集`”, `garbage collector` 翻译为 “`垃圾收集器`”; 一般来说, **垃圾回收** 和 **垃圾收集** 是同义词。
>
> `Minor GC` 翻译为： **小型GC**; 而不是 <del>次要GC</del>
>
> `Major GC` 翻译为： **大型GC**; 而不是 <del>主要GC</del>
>
> 原因在于,大部分情况下, 发生在年轻代的 `Minor GC` 次数要多很多,翻译为次要GC明显不对。
>
> `Full GC` 翻译为： **完整GC**; 其中大型GC和完整GC差不多, 这些术语出自官方的各种分析工具和垃圾收集日志。并不是很统一。
>
> 



At the first sight, garbage collection should be dealing with what the name suggests – finding and throwing away the garbage. In reality it is doing exactly the opposite. Garbage Collection is tracking down all the objects which are still used and marks the rest as garbage. Bearing this in mind, we start digging into more details of how the process of automated memory reclamation called ‘Garbage Collection’ is implemented for Java Virtual Machine.


顾名思义,垃圾收集(Garbage Collection)就是指 —— 发现垃圾并进行处理。但实际上GC的行为却是相反的。**垃圾收集是跟踪所有仍在使用的对象,把其他的部分标记为垃圾**。记住这一点,GC就是自动内存回收的过程,接下来我们开始挖掘JVM中GC实现的更多细节。



Instead of rushing into specifics, we shall start from the very beginning, explaining the general nature of garbage collection and the core concepts and approaches.


不去深究细节,我们先介绍垃圾收集的特性、核心概念和实现方法。


>**Disclaimer**: This content focuses on Oracle Hotspot and OpenJDK behaviour. In other runtimes or even on other JVMs, such as jRockit or IBM J9, some of the aspects covered in this handbook can behave differently.

> **免责声明**: 本文内容侧重于 Hotspot 和 OpenJDK 的行为。其他JVM, 如 jRockit 或 IBM J9, 在某些方面可能表现不同。


## Manual Memory Management

## 手动内存管理


Before we can start covering Garbage Collection in its modern form, let’s do a quick recap of days where you had to manually and explicitly allocate and free memory for your data. And if you ever forgot to free it, you would not be able to reuse the memory. The memory would be claimed but not used. Such a scenario is called a **`memory leak`**.

在介绍现代GC之前, 我们先回顾一下手动管理内存的年代。那时候必须显式地为数据分配内存(allocate)和释放内存(free)。如果忘记释放(free),那么程序就无法重用那一块内存。系统认为程序在用着，但实际上程序并没有使用那块内存，这种情况被称为**内存泄漏**(`memory leak`)。


Here is a simple example written in C using manual memory management:

下面是使用手工管理内存的一段C程序:


	int send_request() {
	    size_t n = read_size();
	    int *elements = malloc(n * sizeof(int));
	
	    if(read_elements(n, elements) < n) {
	        // elements not freed!
	        return -1;
	    }
	
	    // …
	
	    free(elements)
	    return 0;
	}


As we can see, it is fairly easy to forget to free memory. Memory leaks used to be a lot more common problem than today. You could only really fight them by fixing your code. Thus, a much better approach would be to automate the reclamation of unused memory, eliminating the possibility of human error altogether. Such automation is called **`Garbage Collection`** (or GC for short).


我们可以看到, 很容易忘记释放内存。内存泄漏在以前曾经是非常普遍的问题。只能通过修复代码来解决这种问题。因此,急切地需要有一种更好的办法来自动回收未使用的内存,完全消除人为错误的可能性。这种自动化机制就叫做**垃圾收集**(简称GC)。



### Smart Pointers

### 智能指针


One of the first ways to automate garbage collection was built upon reference counting. For each object, you simply know how many times it is referred to and when that count reaches zero the object can be safely reclaimed. A well-known example of that would be the shared pointers of C++:


自动垃圾收集的第一种实现方法依赖于**引用计数**(reference counting)。对每一个对象,只需知道引用了多少次, 当计数变为0时那么这个对象就可以安全地回收(reclaimed)了。一个著名的例子是C++的共享指针(shared pointers):



	int send_request() {
	    size_t n = read_size();
	    shared_ptr<vector<int>> elements 
	              = make_shared<vector<int>>();
	
	    if(read_elements(n, elements) < n) {
	        return -1;
	    }
	
	    return 0;
	}


The `shared_ptr` that we are making use of keeps track of the number of references to it. This number increases as you pass it around and decreases as it leaves scope. As soon as the number of references reaches zero, the `shared_ptr` automatically deletes the underlying vector. This example, as one of our readers points out, is not exactly prevalent in the real world, but it is enough for demonstrational purposes.


我们用 `shared_ptr` 来跟踪引用的数量。当你传递参数时这个数字增加, 在离开作用域时数字减少。一旦引用计数达到0,  `shared_ptr` 自动删除底层的向量。需要指出的是，这个例子在实际应用中并不普遍, 仅仅用来演示。



## Automated Memory Management

## 自动内存管理


In the C++ code above, we still had to explicitly say when we want to have memory management to be taken care of. But what if we could make all the objects behave this way? That would be very handy, since the developers no longer have to think about cleaning up after themselves. The runtime will automatically understand that some memory is no longer used and frees it. In other words, it automatically **collects the garbage**. The first garbage collector was created in 1959 for Lisp and the technology has only advanced since then.


在上面的C++代码中,仍要显式地声明什么时候需要进行内存管理。但我们可不可以让所有对象都具有这种行为呢? 那就太方便了, 这样开发人员就不再需要考虑什么时候需要进行清理了。运行时会自动理解哪些内存不再使用，然后将其释放。换句话说, 它会自动收集垃圾。第一个垃圾收集器是1959年为Lisp创建的, 之后Lisp在这方面一直保持先进水平。


## Reference Counting

## 引用计数


The idea that we have demonstrated with the shared pointers of C++ can be applied to all objects. Many languages such as Perl, Python or PHP take this approach. This is best illustrated with a picture:


前面演示的C++共享指针的办法可以应用到所有对象。许多语言,比如Perl、Python 和 PHP都采用了这种方法。下面是一幅最佳的演示示意图:



![](01_01_Java-GC-counting-references1.png)


The green clouds indicate that the object that they point to is still in use by the programmer. Technically, these may be things like a local variable in the currently executing method or a static variable or something else. It may vary from programming language to programming language so we will not focus on it here.


绿色的云(GC ROOTS)表示的是程序员仍在使用的对象。从技术上讲, 这些可能是当前执行方法中的局部变量，或者静态变量等。在不同的编程语言中形式可能不一样,所以不必过多关注。



The blue circles are the objects in memory as you can see by the number of references to them. Finally, the grey circles are objects which are not referenced from any of the scopes. The grey objects are thus garbage and could be cleaned by the Garbage Collector.


蓝色的圆圈是可以通过引用数量看到的对象。最后,灰色的圆圈是各个作用域都没有引用到的对象。灰色的对象就是可以被垃圾收集器清理的垃圾。



This all looks really good, does it not? Well, it does, but the whole method has a huge drawback. It is quite easy to end up with a **`detached cycle`** of objects none of which are in scope yet due to cyclic references the count of their reference is not zero. Here’s an illustration:


看起来很完美,是吧? 但这个方法有一个巨大的缺点, 很容易被**循环引用**(`detached cycle`) 给搞死。作用域中没有引用指向某些对象，但是由于循环引用导致引用计数不为零。示意图如下所示:


![](01_02_Java-GC-cyclical-dependencies.png)

See? The red objects are in fact garbage that the application does not use. But due to the limitations of reference counting there is still a memory leak.


看明白了吗? 红色的对象实际上属于垃圾。但由于引用计数的局限仍然存在内存泄漏。



There are some ways to overcome this, such as using special ‘weak’ references or applying a separate algorithm for collecting cycles. The aforementioned languages – Perl, Python and PHP – all handle cycles in one way or another, but this is outside the scope of this handbook. Instead, we will start investigating the approach taken by the JVM in more details.



也有一些方法可以处理这种情况, 如使用特殊的“弱引用”(‘weak’ references)或者一个单独的算法来排查循环引用。上述语言——Perl、Python和PHP - 都使用了一种或多种方式来处理循环引用,这些算法超出了本文的范围。我们要关注的是JVM的实现方式。



###Mark and Sweep

### 标记清除


First of all, the JVM is more specific about what constitutes reachability of an object. Instead of the vaguely defined green clouds that we saw on earlier chapters, we have a very specific and explicit set of objects that are called the **`Garbage Collection Roots`**:


首先, JVM明确定义了什么是对象可达性。并不是前面我们看到的绿色的云这样模糊的定义, 有一个非常具体和明确的对象集合, 叫做 **垃圾收集根元素**(`Garbage Collection Roots`)，包括:


- Local variables
- Active threads
- Static fields
- JNI references
- Others (will be discussed later)



- 局部变量(Local variables)
- 活动对象(Active threads)
- 静态属性域(Static fields)
- JNI引用
- 其他(稍后会涉及)


The method used by JVM to track down all the reachable (live) objects and to make sure the memory claimed by non-reachable objects can be reused is called the Mark and Sweep algorithm. It consists of two steps:


JVM使用标记和清扫算法(Mark and Sweep algorithm)来跟踪所有的可达对象(也就是存活对象),确保所有不可达对象所声明(claimed)的内存可以被重用。包含两个步骤:



- **`Marking`** is walking through all reachable objects and keeping a ledger in native memory about all such objects


- **`Sweeping`** is making sure the memory addresses occupied by non-reachable objects can be reused by the next allocations.


- **`Marking`**(标记)是将所有可访问的对象过一遍，并在本地内存中分门别类记下所有这类对象。


- **`Sweeping`**(清除) 用来确保不可达对象所占用的内存在下一次内存分配时是可重用的。



Different GC algorithms within the JVM, such as Parallel Scavenge, Parallel Mark+Copy or CMS, are implementing those phases slightly differently, but at conceptual level the process remains similar to the two steps described above.


在JVM中有多种不同的GC算法, 如Parallel Scavenge(并行清除),Parallel Mark+Copy(并行标记+拷贝) 或 CMS,在实现这些阶段时略有不同, 但在概念层面上所有算法都类似于上面所描述的这两个步骤。



A crucially important thing about this approach is that the cycles are no longer leaked:


标记清除算法最重要的特性是循环引用不再导致内存泄露:


![](01_03_Java-GC-mark-and-sweep.png)


The not-so-good thing is that the application threads need to be stopped for the collection to happen as you cannot really count references if they keep changing all the time. Such a situation when the application is temporarily stopped so that the JVM can indulge in housekeeping activities is called a **`Stop The World pause`**. They may happen for many reasons, but garbage collection is by far the most popular one.


不好的地方是在垃圾收集的过程中，所有线程需要停止，否则如果一直在改变那就没办法进行统计了。为了JVM可以进行清理, 而让应用程序暂时停止的情形，叫做 **全线停顿**(**`Stop The World pause`**)。发生全线停顿的原因很多, 但垃圾收集是最主要的一个。



In this handbook, we will explain how Garbage Collection works in the JVM and how to get the best out of it.

在本手册中,我们将介绍JVM中垃圾收集的工作原理，以及如何更好地使用它。



原文链接: [What Is Garbage Collection?](https://plumbr.eu/handbook/what-is-garbage-collection)



<div style="page-break-after : always;"> </div>


