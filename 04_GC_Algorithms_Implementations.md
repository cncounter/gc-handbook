# 4. GC 算法(实现篇)






Now that we have reviewed the core concepts behind GC algorithms, let us move to the specific implementations one can find inside the JVM. An important aspect to recognize first is the fact that, for most JVMs out there, two different GC algorithms are needed – one to clean the Young Generation and another to clean the Old Generation.



You can choose from a variety of such algorithms bundled into the JVM. If you do not specify a garbage collection algorithm explicitly, a platform-specific default will be used. In this chapter, the working principles of each of those algorithms will be explained.



For a quick cheat sheet, the following list is a fast way to get yourself up to speed with which algorithm combinations are possible. Note that this stands true for Java 8, for older Java versions the available combinations might differ a bit:


<table>
	<thead>
		<tr>
			<th><b>Young</b></th>
			<th><b>Tenured</b></th>
			<th><b>JVM options</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>Incremental</td>
			<td>Incremental</td>
			<td>-Xincgc</td>
		</tr>
		<tr>
			<td><b>Serial</b></td>
			<td><b>Serial</b></td>
			<td><b>-XX:+UseSerialGC</b></td>
		</tr>
		<tr>
			<td>Parallel Scavenge</td>
			<td>Serial</td>
			<td>-XX:+UseParallelGC -XX:-UseParallelOldGC</td>
		</tr>
		<tr>
			<td>Parallel New</td>
			<td>Serial</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td>Serial</td>
			<td>Parallel Old</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td><b>Parallel Scavenge</b></td>
			<td><b>Parallel Old</b></td>
			<td><b>-XX:+UseParallelGC -XX:+UseParallelOldGC</b></td>
		</tr>
		<tr>
			<td>Parallel New</td>
			<td>Parallel Old</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td>Serial</td>
			<td>CMS</td>
			<td>-XX:-UseParNewGC -XX:+UseConcMarkSweepGC</td>
		</tr>
		<tr>
			<td>Parallel Scavenge</td>
			<td>CMS</td>
			<td>N/A</td>
		</tr>
		<tr>
			<td><b>Parallel New</b></td>
			<td><b>CMS</b></td>
			<td><b>-XX:+UseParNewGC -XX:+UseConcMarkSweepGC</b></td>
		</tr>
		<tr>
			<td colspan="2" align="middle"><b>G1</b></td>
			<td><b>-XX:+UseG1GC</b></td>
		</tr>
	</tbody>
</table>




If the above looks too complex, do not worry. In reality it all boils down to just four combinations highlighted in the table above. The rest are either deprecated, not supported or just impractical to apply in real world. So, in the following chapters we cover the working principles of the following combinations:


- Serial GC for both the Young and Old generations
- Parallel GC for both the Young and Old generations
- Parallel New for Young + Concurrent Mark and Sweep (CMS) for the Old Generation
- G1, which encompasses collection of both Young and Old generations


## Serial GC




This collection of garbage collectors uses mark-copy for the Young Generation and mark-sweep-compact for the Old Generation. As the name implies – both of these collectors are single-threaded collectors, incapable of parallelizing the task at hand. Both collectors also trigger stop-the-world pauses, stopping all application threads.



This GC algorithm cannot thus take advantage of multiple CPU cores commonly found in modern hardware. Independent of the number of cores available, just one is used by the JVM during garbage collection.



Enabling this collector for both the Young and Old Generation is done via specifying a single parameter in the JVM startup script:


	java -XX:+UseSerialGC com.mypackages.MyExecutableClass




This option makes sense and is recommended only for the JVM with a couple of hundreds megabytes heap size, running in an environment with a single CPU. For the majority of server-side deployments this is a rare combination. Most server-side deployments are done on platforms with multiple cores, essentially meaning that by choosing Serial GC you are setting artificial limits on the use of system resources. This results in idle resources which otherwise could be used to reduce latency or increase throughput.



Let us now review how garbage collector logs look like when using Serial GC and what useful information one can obtain from there. For this purpose, we have turned on GC logging on the JVM using the following parameters:


	-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps




The resulting output is similar to the following:


	2015-05-26T14:45:37.987-0200: 151.126: [GC (Allocation Failure) 151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs] 1619346K->1273247K(2027264K), 0.0585007 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]
	2015-05-26T14:45:59.690-0200: 172.829: [GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs] 1832479K->755802K(2027264K), [Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs] [Times: user=0.18 sys=0.00, real=0.18 secs]





Such short snippet from the GC logs exposes a lot of information about what is taking place inside the JVM. As a matter of fact, in this snippet there were two Garbage Collection events taking place, one of them cleaning the Young Generation and another taking care of the entire heap. Let’s start by analyzing the first collection that is taking place in the Young Generation.


### Minor GC



Following snippet contains the information about a GC event cleaning the Young Generation:


> 2015-05-26T14:45:37.987-02001:151.1262:[GC3(Allocation Failure4) 151.126: [DefNew5:629119K->69888K6(629120K)7, 0.0584157 secs]1619346K->1273247K8(2027264K)9,0.0585007 secs10][Times: user=0.06 sys=0.00, real=0.06 secs]11

>1. 2015-05-26T14:45:37.987-0200 – Time when the GC event started.
1. 151.126 – Time when the GC event started, relative to the JVM startup time. Measured in seconds.
1. GC – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC.
1. Allocation Failure – Cause of the collection. In this case, the GC is triggered due to a data structure not fitting into any region in the Young Generation.
1. DefNew – Name of the garbage collector used. This cryptic name stands for the single-threaded mark-copy stop-the-world garbage collector used to clean Young generation.
1. 629119K->69888K – Usage of the Young Generation before and after collection.
1. (629120K) – Total size of the Young Generation.
1. 1619346K->1273247K – Total used heap before and after collection.
1. (2027264K) – Total available heap.
1. 0.0585007 secs – Duration of the GC event in seconds.
1. [Times: user=0.06 sys=0.00, real=0.06 secs] – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by the garbage collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. As Serial Garbage Collector always uses just a single thread, real time is thus equal to the sum of user and system times.




From the above snippet we can understand exactly what was happening with the memory consumption inside JVM during the GC event. Before this collection, heap usage totaled at 1,619,346K. Out of this, the Young Generation consumed 629,119K. From this we can calculate the Old Generation usage being equal to 990,227K.



A more important conclusion is hidden in the next batch of numbers indicating that, after the collection, Young Generation usage decreased by 559,231K but total heap usage decreased only by 346,099K. From this we can again derive that 213,132K of objects were promoted from the Young Generation to the Old Generation.



This GC event is also illustrated with the following snapshots showing memory usage right before the GC started and right after it finished:

![](04_01_serial-gc-in-young-generation.png)


### Full GC




After understanding the first minor GC event, lets look into the second GC event in the logs:


> 2015-05-26T14:45:59.690-02001: 172.8292:[GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs3]172.829:[Tenured4: 1203359K->755802K 5(1398144K) 6,0.1855567 secs7] 1832479K->755802K8(2027264K)9,[Metaspace: 6741K->6741K(1056768K)]10 [Times: user=0.18 sys=0.00, real=0.18 secs]11

> 
1. 2015-05-26T14:45:59.690-0200 – Time when the GC event started.
1. 172.829 – Time when the GC event started, relative to the JVM startup time. Measured in seconds.
1. [DefNew: 629120K->629120K(629120K), 0.0000372 secs – Similar to the previous example, a minor garbage collection in the Young Generation happened during this event due to Allocation Failure. For this collection the same DefNew collector was run as before and it decreased the usage of the Young Generation from 629120K to 0. Notice that JVM reports this incorrectly due to buggy behavior and instead reports the Young Generation as being completely full. This collection took 0.0000372 seconds.
1. Tenured – Name of the garbage collector used to clean the Old space. The name Tenured indicates a single-threaded stop-the-world mark-sweep-compact garbage collector being used.
1. 1203359K->755802K  – Usage of Old generation before and after the event.
1. (1398144K)  – Total capacity of the Old generation.
1. 0.1855567 secs – Time it took to clean the Old Generation.
1. 1832479K->755802K – Usage of the whole heap before and after the collection of the Young and Old Generations.
1. (2027264K) – Total heap available for the JVM.
1. [Metaspace: 6741K->6741K(1056768K)] – Similar information about Metaspace collection. As seen, no garbage was collected in Metaspace during the event.
1. [Times: user=0.18 sys=0.00, real=0.18 secs] – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by Garbage Collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. As Serial Garbage Collector always uses just a single thread, real time is thus equal to the sum of user and system times.





The difference with Minor GC is evident – in addition to the Young Generation, during this GC event the Old Generation and Metaspace were also cleaned. The layout of the memory before and after the event would look like the situation in the following picture:

![](04_02_serial-gc-in-old-gen-java.png)


## Parallel GC




This combination of Garbage Collectors uses mark-copy in the Young Generation and mark-sweep-compact in the Old Generation. Both Young and Old collections trigger stop-the-world events, stopping all application threads to perform garbage collection. Both collectors run marking and copying / compacting phases using multiple threads, hence the name ‘Parallel’. Using this approach, collection times can be considerably reduced.



The number of threads used during garbage collection is configurable via the command line parameter -XX:ParallelGCThreads=NNN . The default value is equal to the number of cores in your machine.



Selection of Parallel GC is done via the specification of any of the following combinations of parameters in the JVM startup script:

	java -XX:+UseParallelGC com.mypackages.MyExecutableClass
	java -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
	java -XX:+UseParallelGC -XX:+UseParallelOldGC com.mypackages.MyExecutableClass





Parallel Garbage Collector is suitable on multi-core machines in cases where your primary goal is to increase throughput. Higher throughput is achieved due to more efficient usage of system resources:

 - during collection, all cores are cleaning the garbage in parallel, resulting in shorter pauses
 - between garbage collection cycles neither of the collectors is consuming any resources
 




On the other hand, as all phases of the collection have to happen without any interruptions, these collectors are still susceptible to long pauses during which your application threads are stopped. So if latency is your primary goal, you should check the next combinations of garbage collectors.



Let us now review how garbage collector logs look like when using Parallel GC and what useful information one can obtain from there. For this, let’s look again at the garbage collector logs that expose once more one minor and one major GC pause:


	2015-05-26T14:27:40.915-0200: 116.115: [GC (Allocation Failure) 
			[PSYoungGen: 2694440K->1305132K(2796544K)] 
		9556775K->8438926K(11185152K)
		, 0.2406675 secs] 
		[Times: user=1.77 sys=0.01, real=0.24 secs]
	2015-05-26T14:27:41.155-0200: 116.356: [Full GC (Ergonomics) 
			[PSYoungGen: 1305132K->0K(2796544K)] 
			[ParOldGen: 7133794K->6597672K(8388608K)] 8438926K->6597672K(11185152K), 
			[Metaspace: 6745K->6745K(1056768K)]
		, 0.9158801 secs]
		[Times: user=4.49 sys=0.64, real=0.92 secs]


### Minor GC




The first of the two events indicates a GC event taking place in the Young Generation:



> 5-26T14:27:40.915-02001: 116.1152:[GC3(Allocation Failure4)[PSYoungGen5: 2694440K->1305132K6(2796544K)7]9556775K->8438926K8(11185152K)9, 0.2406675 secs10][Times: user=1.77 sys=0.01, real=0.24 secs]11

>
1. 2015-05-26T14:27:40.915-0200 – Time when the GC event started.
1. 116.115 – Time when the GC event started, relative to the JVM startup time. Measured in seconds.
1. GC – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC.
1. Allocation Failure – Cause of the collection. In this case, the GC is triggered due to a data structure not fitting into any region in the Young Generation.
1. PSYoungGen – Name of the garbage collector used, representing a parallel mark-copy stop-the-world garbage collector used to clean the Young generation.
1. 2694440K->1305132K – usage of the Young Generation before and after collection
1. (2796544K) – Total size of the Young Generation
1. 9556775K->8438926K – Total heap usage before and after collection
1. (11185152K) – Total available heap
1. 0.2406675 secs – Duration of the GC event in seconds
1. [Times: user=1.77 sys=0.01, real=0.24 secs] – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by Garbage Collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount.





So, in short, the total heap consumption before the collection was 9,556,775K. Out of this Young generation was 2,694,440K. This means that used Old generation was 6,862,335K. After the collection young generation usage decreased by 1,389,308K, but total heap usage decreased only by 1,117,849K. This means that 271,459K was promoted from Young generation to Old.

![](04_03_ParallelGC-in-Young-Generation-Java.png)


### Full GC



After understanding how Parallel GC cleans the Young Generation, we are ready to look at how the whole heap is being cleaned by analyzing the next snippet from the GC logs:


>
2015-05-26T14:27:41.155-02001:116.3562:[Full GC3 (Ergonomics4)[PSYoungGen: 1305132K->0K(2796544K)]5[ParOldGen6:7133794K->6597672K 7(8388608K)8] 8438926K->6597672K9(11185152K)10, [Metaspace: 6745K->6745K(1056768K)] 11, 0.9158801 secs12, [Times: user=4.49 sys=0.64, real=0.92 secs]13

>
1. 2015-05-26T14:27:41.155-0200 – Time when the GC event started
1. 116.356 – Time when the GC event started, relative to the JVM startup time. Measured in seconds. In this case we can see the event started right after the previous Minor GC finished.
1. Full GC – Flag indicating that the event is Full GC event cleaning both the Young and Old generations.
1. Ergonomics – Reason for the GC taking place. This indicates that the JVM internal ergonomics decided this is the right time to collect some garbage.
1. [PSYoungGen: 1305132K->0K(2796544K)] – Similar to previous example, a parallel mark-copy stop-the-world garbage collector named “PSYoungGen” was used to clean the Young Generation. Usage of Young Generation shrank from 1305132K to 0, which is the typical result of a Full GC.
1. ParOldGen – Type of the collector used to clean the Old Generation. In this case, parallel mark-sweep-compact stop-the-world garbage collector named ParOldGen was used.
1. 7133794K->6597672K  – Usage of the Old Generation before and after the collection
1. (8388608K) – Total size of the Old Generation
1. 8438926K->6597672K – Usage of the whole heap before and after the collection.
1. (11185152K) – Total heap available
1. [Metaspace: 6745K->6745K(1056768K)]  – Similar information about Metaspace region. As we can see, no garbage was collected in Metaspace during this event.
1. 0.9158801 secs – Duration of the GC event in seconds
1. [Times: user=4.49 sys=0.64, real=0.92 secs] – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by Garbage Collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount.





Again, the difference with Minor GC is evident – in addition to the Young Generation, during this GC event the Old Generation and Metaspace were also cleaned. The layout of the memory before and after the event would look like the situation in the following picture:

![](04_04_Java-ParallelGC-in-Old-Generation.png)


## Concurrent Mark and Sweep




The official name for this collection of garbage collectors is “Mostly Concurrent Mark and Sweep Garbage Collector”. It uses the parallel stop-the-world mark-copy algorithm in the Young Generation and the mostly concurrent mark-sweep algorithm in the Old Generation.



This collector was designed to avoid long pauses while collecting in the Old Generation. It achieves this by two means. Firstly, it does not compact the Old Generation but uses free-lists to manage reclaimed space. Secondly, it does most of the job in the mark-and-sweep phases concurrently with the application. This means that garbage collection is not explicitly stopping the application threads to perform these phases. It should be noted, however, that it still competes for CPU time with the application threads. By default, the number of threads used by this GC algorithm equals to ¼ of the number of physical cores of your machine.



This garbage collector can be chosen by specifying the following option on your command line:


	java -XX:+UseConcMarkSweepGC com.mypackages.MyExecutableClass




This combination is a good choice on multi-core machines if your primary target is latency. Decreasing the duration of an individual GC pause directly affects the way your application is perceived by end-users, giving them a feel of a more responsive application. As most of the time at least some CPU resources are consumed by the GC and not executing your application’s code, CMS generally often worse throughput than Parallel GC in CPU-bound applications.



As with previous GC algorithms, let us now see how this algorithm is applied in practice by taking a look at the GC logs that once again expose one minor and one major GC pause:


	2015-05-26T16:23:07.219-0200: 64.322: [GC (Allocation Failure) 64.322: [ParNew: 613404K->68068K(613440K), 0.1020465 secs] 10885349K->10880154K(12514816K), 0.1021309 secs] [Times: user=0.78 sys=0.01, real=0.11 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) [1 CMS-initial-mark: 10812086K(11901376K)] 10887844K(12514816K), 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] [Times: user=0.07 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
	2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] [Times: user=0.20 sys=0.00, real=1.07 secs]
	2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) [YG occupancy: 387920 K (613440 K)]65.550: [Rescan (parallel) , 0.0085125 secs]65.559: [weak refs processing, 0.0000243 secs]65.559: [class unloading, 0.0013120 secs]65.560: [scrub symbol table, 0.0008345 secs]65.561: [scrub string table, 0.0001759 secs][1 CMS-remark: 10812086K(11901376K)] 11200006K(12514816K), 0.0110730 secs] [Times: user=0.06 sys=0.00, real=0.01 secs]
	2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
	2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
	2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]


### Minor GC




First of the GC events in log denotes a minor GC cleaning the Young space. Let’s analyze how this collector combination behaves in this regard:


>
2015-05-26T16:23:07.219-02001: 64.3222:[GC3(Allocation Failure4) 64.322: [ParNew5: 613404K->68068K6(613440K) 7, 0.1020465 secs8] 10885349K->10880154K 9(12514816K)10, 0.1021309 secs11][Times: user=0.78 sys=0.01, real=0.11 secs]12

>
1. 2015-05-26T16:23:07.219-0200 – Time when the GC event started.
1. 64.322 – Time when the GC event started, relative to the JVM startup time. Measured in seconds.
1. GC – Flag to distinguish between Minor & Full GC. This time it is indicating that this was a Minor GC.
1. Allocation Failure – Cause of the collection. In this case, the GC is triggered due to a requested allocation not fitting into any region in Young Generation.
1. ParNew – Name of the collector used, this time it indicates a parallel mark-copy stop-the-world garbage collector used in the Young Generation, designed to work in conjunction with Concurrent Mark & Sweep garbage collector in the Old Generation.
1. 613404K->68068K – Usage of the Young Generation before and after collection.
1. (613440K)  – Total size of the Young Generation.
1. 0.1020465 secs – Duration for the collection w/o final cleanup.
1. 10885349K->10880154K  – Total used heap before and after collection.
1. (12514816K) – Total available heap.
1. 0.1021309 secs – The time it took for the garbage collector to mark and copy live objects in the Young Generation. This includes communication overhead with ConcurrentMarkSweep collector, promotion of objects that are old enough to the Old Generation and some final cleanup at the end of the garbage collection cycle.
1. `[Times: user=0.78 sys=0.01, real=0.11 secs]` – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by Garbage Collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. With Parallel GC this number should be close to (user time + system time) divided by the number of threads used by the Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount.




From the above we can thus see that before the collection the total used heap was 10,885,349K and the used Young Generation share was 613,404K. This means that the Old Generation share was 10,271,945K. After the collection, Young Generation usage decreased by 545,336K but total heap usage decreased only by 5,195K. This means that 540,141K was promoted from the Young Generation to Old.


![](04_05_ParallelGC-in-Young-Generation-Java.png)


### Full GC



Now, just as you are becoming accustomed to reading GC logs already, this chapter will introduce a completely different format for the next garbage collection event in the logs. The lengthy output that follows consists of all the different phases of the mostly concurrent garbage collection in the Old Generation. We will review them one by one but in this case we will cover the log content in phases instead of the entire event log at once for more concise representation. But to recap, the whole event for the CMS collector looks like the following:


	2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) [1 CMS-initial-mark: 10812086K(11901376K)] 10887844K(12514816K), 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
	2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] [Times: user=0.07 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
	2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
	2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] [Times: user=0.20 sys=0.00, real=1.07 secs]
	2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) [YG occupancy: 387920 K (613440 K)]65.550: [Rescan (parallel) , 0.0085125 secs]65.559: [weak refs processing, 0.0000243 secs]65.559: [class unloading, 0.0013120 secs]65.560: [scrub symbol table, 0.0008345 secs]65.561: [scrub string table, 0.0001759 secs][1 CMS-remark: 10812086K(11901376K)] 11200006K(12514816K), 0.0110730 secs] [Times: user=0.06 sys=0.00, real=0.01 secs]
	2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
	2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
	2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
	2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]





Just to bear in mind – in real world situation Minor Garbage Collections of the Young Generation can occur anytime during concurrent collecting the Old Generation. In such case the major collection records seen below will be interleaved with the Minor GC events covered in previous chapter.

**Phase 1: Initial Mark**. This is one of the two stop-the-world events during CMS. The goal of this phase is to mark all the objects in the Old Generation that are either direct GC roots or are referenced from some live object in the Young Generation. The latter is important since the Old Generation is collected separately.


![](04_06_g1-06.png)


>
2015-05-26T16:23:07.321-0200: 64.421: [GC (CMS Initial Mark2[1 CMS-initial-mark: 10812086K3(11901376K)4] 10887844K5(12514816K)6, 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]7

>
1. 2015-05-26T16:23:07.321-0200: 64.42 – Time the GC event started, both clock time and relative to the time from the JVM start. For the following phases the same notion is used throughout the event and is thus skipped for brevity.
1. CMS Initial Mark – Phase of the collection – “Initial Mark” in this occasion – that is collecting all GC Roots.
1. 10812086K – Currently used Old Generation.
1. (11901376K) – Total available memory in the Old Generation.
1. 10887844K – Currently used heap
1. (12514816K) – Total available heap
1. 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] – Duration of the phase, measured also in user, system and real time.



**Phase 2: Concurrent Mark.** During this phase the Garbage Collector traverses the Old Generation and marks all live objects, starting from the roots found in the previous phase of “Initial Mark”. The “Concurrent Mark” phase, as its name suggests, runs concurrently with your application and does not stop the application threads. Note that not all the live objects in the Old Generation may be marked, since the application is mutating references during the marking.

![](04_07_g1-07.png)





In the illustration, a reference pointing away from the “Current object” was removed concurrently with the marking thread.

>
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]

>
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark1: 035/0.035 secs2] [Times: user=0.07 sys=0.00, real=0.03 secs]3

>
1. CMS-concurrent-mark – Phase of the collection – “Concurrent Mark” in this occasion – that is traversing the Old Generation and marking all live objects.
1. 035/0.035 secs – Duration of the phase, showing elapsed time and wall clock time correspondingly.
1. [Times: user=0.07 sys=0.00, real=0.03 secs] – “Times” section is less meaningful for concurrent phases as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking.


**Phase 3: Concurrent Preclean.** This is again a concurrent phase, running in parallel with the application threads, not stopping them. While the previous phase was running concurrently with the application, some references were changed. Whenever that happens, the JVM marks the area of the heap (called “Card”) that contains the mutated object as “dirty” (this is known as Card Marking).


![](04_08_g1-08.png)





In the pre-cleaning phase, these dirty objects are accounted for, and the objects reachable from them are also marked. The cards are cleaned when this is done.

![](04_09_g1-09.png)





Additionally, some necessary housekeeping and preparations for the Final Remark phase are performed.

>
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
>
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean1: 0.016/0.016 secs2] [Times: user=0.02 sys=0.00, real=0.02 secs]3

>
1. CMS-concurrent-preclean – Phase of the collection – “Concurrent Preclean” in this occasion – accounting for references being changed during previous marking phase.
1. 0.016/0.016 secs – Duration of the phase, showing elapsed time and wall clock time correspondingly.
1. [Times: user=0.02 sys=0.00, real=0.02 secs] – The “Times” section is less meaningful for concurrent phases as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking.


**Phase 4: Concurrent Abortable Preclean.** Again, a concurrent phase that is not stopping the application’s threads. This one attempts to take as much work off the shoulders of the stop-the-world Final Remark as possible. The exact duration of this phase depends on a number of factors, since it iterates doing the same thing until one of the abortion conditions (such as the number of iterations, amount of useful work done, elapsed wall clock time, etc) is met.


>
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
>
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean1: 0.167/1.074 secs2] [Times: user=0.20 sys=0.00, real=1.07 secs]3

>
1. CMS-concurrent-abortable-preclean – Phase of the collection “Concurrent Abortable Preclean” in this occasion
1. 0.167/1.074 secs – Duration of the phase, showing elapsed and wall clock time respectively. It is interesting to note that the user time reported is a lot smaller than clock time. Usually we have seen that real time is less than user time, meaning that some work was done in parallel and so elapsed clock time is less than used CPU time. Here we have a little amount of work – for 0.167 seconds of CPU time, and garbage collector threads were doing a lot of waiting. Essentially, they were trying to stave off for as long as possible before having to do an STW pause. By default, this phase may last for up to 5 seconds.
1. [Times: user=0.20 sys=0.00, real=1.07 secs] – The “Times” section is less meaningful for concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking.






This phase may significantly impact the duration of the upcoming stop-the-world pause, and has quite a lot of non-trivial configuration options and fail modes.

**Phase 5: Final Remark.** This is the second and last stop-the-world phase during the event. The goal of this stop-the-world phase is to finalize marking all live objects in the Old Generation. Since the previous preclean phases were concurrent, they may have been unable to keep up with the application’s mutating speeds. A stop-the-world pause is required to finish the ordeal.



Usually CMS tries to run final remark phase when Young Generation is as empty as possible in order to eliminate the possibility of several stop-the-world phases happening back-to-back.



This event looks a bit more complex than previous phases:

>
2015-05-26T16:23:08.447-0200: 65.5501: [GC (CMS Final Remark2) [YG occupancy: 387920 K (613440 K)3]65.550: [Rescan (parallel) , 0.0085125 secs]465.559: [weak refs processing, 0.0000243 secs]65.5595: [class unloading, 0.0013120 secs]65.5606: [scrub string table, 0.0001759 secs7][1 CMS-remark: 10812086K(11901376K)8] 11200006K(12514816K) 9, 0.0110730 secs10] [[Times: user=0.06 sys=0.00, real=0.01 secs]11

>
1. 2015-05-26T16:23:08.447-0200: 65.550 – Time the GC event started, both clock time and relative to the time from the JVM start.
1. CMS Final Remark – Phase of the collection – “Final Remark” in this occasion – that is marking all live objects in the Old Generation, including the references that were created/modified during previous concurrent marking phases.
1. YG occupancy: 387920 K (613440 K) – Current occupancy and capacity of the Young Generation.
1. [Rescan (parallel) , 0.0085125 secs] – The “Rescan” completes the marking of live objects while the application is stopped. In this case the rescan was done in parallel and took 0.0085125 seconds.
1. weak refs processing, 0.0000243 secs]65.559 – First of the sub-phases that is processing weak references along with the duration and timestamp of the phase.
1. class unloading, 0.0013120 secs]65.560 – Next sub-phase that is unloading the unused classes, with the duration and timestamp of the phase.
1. scrub string table, 0.0001759 secs – Final sub-phase that is cleaning up symbol and string tables which hold class-level metadata and internalized string respectively. Clock time of the pause is also included.
1. 10812086K(11901376K) – Occupancy and the capacity of the Old Generation after the phase.
1. 11200006K(12514816K)  – Usage and the capacity of the total heap after the phase.
1. 0.0110730 secs – Duration of the phase.
1. [Times: user=0.06 sys=0.00, real=0.01 secs] – Duration of the pause, measured in user, system and real time categories.





After the five marking phases, all live objects in the Old Generation are marked and now garbage collector is going to reclaim all unused objects by sweeping the Old Generation:



Phase 6: Concurrent Sweep. Performed concurrently with the application, without the need for the stop-the-world pauses. The purpose of the phase is to remove unused objects and to reclaim the space occupied by them for future use.


![](04_10_g1-10.png)



>
2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start] 2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep1: 0.027/0.027 secs2] [[Times: user=0.03 sys=0.00, real=0.03 secs] 3

>
1. CMS-concurrent-sweep – Phase of the collection “Concurrent Sweep” in this occasion, sweeping unmarked and thus unused objects to reclaim space.
1. 0.027/0.027 secs – Duration of the phase, showing elapsed time and wall clock time correspondingly.
1. [Times: user=0.03 sys=0.00, real=0.03 secs]  – “Times” section is less meaningful on concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking.


**Phase 7: Concurrent Reset.** Concurrently executed phase, resetting inner data structures of the CMS algorithm and preparing them for the next cycle.

>
2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start] 2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset1: 0.012/0.012 secs2] [[Times: user=0.01 sys=0.00, real=0.01 secs]3

>
1. CMS-concurrent-reset – The phase of the collection – “Concurrent Reset” in this occasion – that is resetting inner data structures of the CMS algorithm and preparing for the next collection.
1. 0.012/0.012 secs – Duration of the the phase, measuring elapsed and wall clock time respectively.
1. [Times: user=0.01 sys=0.00, real=0.01 secs] – The “Times” section is less meaningful on concurrent phases, as it is measured from the start of the concurrent marking and includes more than just the work done for the concurrent marking.





All in all, the CMS garbage collector does a great job at reducing the pause durations by offloading a great deal of the work to concurrent threads that do not require the application to stop. However, it, too, has some drawbacks, the most notable of them being the Old Generation fragmentation and the lack of predictability in pause durations in some cases, especially on large heaps.


## G1 – Garbage First




One of the key design goals of G1 was to make the duration and distribution of stop-the-world pauses due to garbage collection predictable and configurable. In fact, Garbage-First is a soft real-time garbage collector, meaning that you can set specific performance goals to it. You can request the stop-the-world pauses to be no longer than x milliseconds within any given y-millisecond long time range, e.g. no more than 5 milliseconds in any given second. Garbage-First GC will do its best to meet this goal with high probability (but not with certainty, that would be hard real-time).



To achieve this, G1 builds upon a number of insights. First, the heap does not have to be split into contiguous Young and Old generation. Instead, the heap is split into a number (typically about 2048) smaller heap regions that can house objects. Each region may be an Eden region, a Survivor region or an Old region. The logical union of all Eden and Survivor regions is the Young Generation, and all the Old regions put together is the Old Generation:


![](04_11_g1-011.png)





This allows the GC to avoid collecting the entire heap at once, and instead approach the problem incrementally: only a subset of the regions, called the collection set will be considered at a time. All the Young regions are collected during each pause, but some Old regions may be included as well:


![](04_12_g1-02.png)






Another novelty of G1 is that during the concurrent phase it estimates the amount of live data that each region contains. This is used in building the collection set: the regions that contain the most garbage are collected first. Hence the name: garbage-first collection.



To run the JVM with the G1 collector enabled, run your application as

	java -XX:+UseG1GC com.mypackages.MyExecutableClass


### Evacuation Pause: Fully Young




In the beginning of the application’s lifecycle, G1 does not have any additional information from the not-yet-executed concurrent phases, so it initially functions in the fully-young mode. When the Young Generation fills up, the application threads are stopped, and the live data inside the Young regions is copied to Survivor regions, or any free regions that thereby become Survivor.



The process of copying these is called Evacuation, and it works in pretty much the same way as the other Young collectors we have seen before. The full logs of evacuation pauses are rather large, so, for simplicity’s sake we will leave out a couple of small bits that are irrelevant in the first fully-young evacuation pause. We will get back to them after the concurrent phases are explained in greater detail. In addition, due to the sheer size of the log record, the parallel phase details and “Other” phase details are extracted to separate sections:


>
	0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]1
	    [Parallel Time: 13.9 ms, GC Workers: 8]2
	        …3
	    [Code Root Fixup: 0.0 ms]4
	    [Code Root Purge: 0.0 ms]5
	    [Clear CT: 0.1 ms]
	    [Other: 0.4 ms]6
	        …7
	    [Eden: 24.0M(24.0M)->0.0B(13.0M) 8Survivors: 0.0B->3072.0K 9Heap: 24.0M(256.0M)->21.9M(256.0M)]10
	     [Times: user=0.04 sys=0.04, real=0.02 secs] 11

>
1. 0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs] – G1 pause cleaning only (young) regions. The pause started 134ms after the JVM startup and the duration of the pause was 0.0144 seconds measured in wall clock time.
1. [Parallel Time: 13.9 ms, GC Workers: 8] – Indicating that for 13.9 ms (real time) the following activities were carried out by 8 threads in parallel
1. … – Cut for brevity, see the following section below for the details.
1. [Code Root Fixup: 0.0 ms] – Freeing up the data structures used for managing the parallel activities. Should always be near-zero. This is done sequentially.
1. [Code Root Purge: 0.0 ms] – Cleaning up more data structures, should also be very fast, but non necessarily almost zero. This is done sequentially.
1. [Other: 0.4 ms] – Miscellaneous other activities, many of which are also parallelized
1. … – See the section below for details
1. [Eden: 24.0M(24.0M)->0.0B(13.0M)  – Eden usage and capacity before and after the pause
1. Survivors: 0.0B->3072.0K  – Space used by Survivor regions before and after the pause
1. Heap: 24.0M(256.0M)->21.9M(256.0M)] – Total heap usage and capacity before and after the pause.
1. [Times: user=0.04 sys=0.04, real=0.02 secs]  – Duration of the GC event, measured in different categories:
 - user – Total CPU time that was consumed by Garbage Collector threads during this collection
 - sys – Time spent in OS calls or waiting for system event
 - real – Clock time for which your application was stopped. With the parallelizable activities during GC this number is ideally close to (user time + system time) divided by the number of threads used by Garbage Collector. In this particular case 8 threads were used. Note that due to some activities not being parallelizable, it always exceeds the ratio by a certain amount.





Most of the heavy-lifting is done by multiple dedicated GC worker threads. Their activities are described in the following section of the log:


>
	[Parallel Time: 13.9 ms, GC Workers: 8]1
	     [GC Worker Start (ms)2: Min: 134.0, Avg: 134.1, Max: 134.1, Diff: 0.1]
	    [Ext Root Scanning (ms)3: Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.2]
	    [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
	        [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
	    [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
	    [Code Root Scanning (ms)4: Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
	    [Object Copy (ms)5: Min: 10.8, Avg: 12.1, Max: 12.6, Diff: 1.9, Sum: 96.5]
	    [Termination (ms)6: Min: 0.8, Avg: 1.5, Max: 2.8, Diff: 1.9, Sum: 12.2]
	        [Termination Attempts7: Min: 173, Avg: 293.2, Max: 362, Diff: 189, Sum: 2346]
	    [GC Worker Other (ms)8: Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
	    GC Worker Total (ms)9: Min: 13.7, Avg: 13.8, Max: 13.8, Diff: 0.1, Sum: 110.2]
	    [GC Worker End (ms)10: Min: 147.8, Avg: 147.8, Max: 147.8, Diff: 0.0]


>
1. [Parallel Time: 13.9 ms, GC Workers: 8] – Indicating that for 13.9 ms (clock time) the following activities were carried out by 8 threads in parallel
1. [GC Worker Start (ms) – The moment in time at which the workers started their activity, matching the timestamp at the beginning of the pause. If Min and Max differ a lot, then it may be an indication that too many threads are used or other processes on the machine are stealing CPU time from the garbage collection process inside the JVM
1. [Ext Root Scanning (ms) – How long it took to scan the external (non-heap) roots such as classloaders, JNI references, JVM system roots, etc. Shows elapsed time, “Sum” is CPU time
1. [Code Root Scanning (ms) – How long it took to scan the roots that came from the actual code: local vars, etc.
1. [Object Copy (ms) – How long it took to copy the live objects away from the collected regions.
1. [Termination (ms) – How long it took for the worker threads to ensure that they can safely stop and that there’s no more work to be done, and then actually terminate
1. [Termination Attempts – How many attempts worker threads took to try and terminate. An attempt is failed if the worker discovers that there’s in fact more work to be done, and it’s too early to terminate.
1. [GC Worker Other (ms) – Other miscellaneous small activities that do not deserve a separate section in the logs.
1. GC Worker Total (ms) – How long the worker threads have worked for in total
1. [GC Worker End (ms) – The timestamp at which the workers have finished their jobs. Normally they should be roughly equal, otherwise it may be an indication of too many threads hanging around or a noisy neighbor





Additionally, there are some miscellaneous activities that are performed during the Evacuation pause. We will only cover a part of them in this section. The rest will be covered later.


>
	[Other: 0.4 ms]1
	    [Choose CSet: 0.0 ms]
	    [Ref Proc: 0.2 ms]2
	    [Ref Enq: 0.0 ms]3
	    [Redirty Cards: 0.1 ms]
	    [Humongous Register: 0.0 ms]
	    [Humongous Reclaim: 0.0 ms]
	    [Free CSet: 0.0 ms]4

>
1. [Other: 0.4 ms] – Miscellaneous other activities, many of which are also parallelized
1. [Ref Proc: 0.2 ms] – The time it took to process non-strong references: clear them or determine that no clearing is needed.
1. [Ref Enq: 0.0 ms] – The time it took to enqueue the remaining non-strong references to the appropriate ReferenceQueue
1. [Free CSet: 0.0 ms] – The time it takes to return the freed regions in the collection set so that they are available for new allocations.


### Concurrent Marking




The G1 collector builds up on many concepts of CMS from the previous section, so it is a good idea to make sure that you have a sufficient understanding of it before proceeding. Even though it differs in a number of ways, the goals of the Concurrent Marking are very similar.  G1 Concurrent Marking uses the Snapshot-At-The-Beginning approach that marks all the objects that were live at the beginning of the marking cycle, even if they have turned into garbage meanwhile. The information on which objects are live allows to build up the liveness stats for each region so that the collection set could be efficiently chosen afterwards.



This information is then used to perform garbage collection in the Old regions. It can happen fully concurrently, if the marking determines that a region contains only garbage, or during a stop-the-world evacuation pause for Old regions that contain both garbage and live objects.



Concurrent Marking starts when the overall occupancy of the heap is large enough. By default, it is 45%, but this can be changed by the InitiatingHeapOccupancyPercent JVM option. Like in CMS, Concurrent Marking in G1 consists of a number of phases, some of them fully concurrent, and some of them requiring the application threads to be stopped.


**Phase 1: Initial Mark.** This phase marks all the objects directly reachable from the GC roots. In CMS, it required a separate stop-the world pause, but in G1 it is typically piggy-backed on an Evacuation Pause, so its overhead is minimal. You can notice this pause in GC logs by the “(initial-mark)” addition in the first line of an Evacuation Pause:


	1.631: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0062656 secs]

**Phase 2: Root Region Scan.** This phase marks all the live objects reachable from the so-called root regions, i.e. the ones that are not empty and that we might end up having to collect in the middle of the marking cycle. Since moving stuff around in the middle of concurrent marking will cause trouble, this phase has to complete before the next evacuation pause starts. If it has to start earlier, it will request an early abort of root region scan, and then wait for it to finish. In the current implementation, the root regions are the survivor regions: they are the bits of Young Generation that will definitely be collected in the next Evacuation Pause.

	1.362: [GC concurrent-root-region-scan-start]
	1.364: [GC concurrent-root-region-scan-end, 0.0028513 secs]


**Phase 3. Concurrent Mark.** This phase is very much similar to that of CMS: it simply walks the object graph and marks the visited objects in a special bitmap. To ensure that the semantics of snapshot-at-the beginning are met, G1 GC requires that all the concurrent updates to the object graph made by the application threads leave the previous reference known for marking purposes.



This is achieved by the use of the Pre-Write barriers (not to be confused with Post-Write barriers discussed later and memory barriers that relate to multithreaded programming). Their function is to, whenever you write to a field while G1 Concurrent Marking is active, store the previous referee in the so-called log buffers, to be processed by the concurrent marking threads.


	1.364: [GC concurrent-mark-start]
	1.645: [GC concurrent-mark-end, 0.2803470 secs]


**Phase 4. Remark.** This is a stop-the-world pause that, like previously seen in CMS, completes the marking process. For G1, it briefly stops the application threads to stop the inflow of the concurrent update logs and processes the little amount of them that is left over, and marks whatever still-unmarked objects that were live when the concurrent marking cycle was initiated. This phase also performs some additional cleaning, e.g. reference processing (see the Evacuation Pause log) or class unloading.


	1.645: [GC remark 1.645: [Finalize Marking, 0.0009461 secs] 1.646: [GC ref-proc, 0.0000417 secs] 1.646: [Unloading, 0.0011301 secs], 0.0074056 secs]
	[Times: user=0.01 sys=0.00, real=0.01 secs]


**Phase 5. Cleanup.** This final phase prepares the ground for the upcoming evacuation phase, counting all the live objects in the heap regions, and sorting these regions by expected GC efficiency. It also performs all the house-keeping activities required to maintain the internal state for the next iteration of concurrent marking.



Last but not least, the regions that contain no live objects at all are reclaimed in this phase. Some parts of this phase are concurrent, such as the empty region reclamation and most of the liveness calculation, but it also requires a short stop-the-world pause to finalize the picture while the application threads are not interfering. The logs for such stop-the-world pauses would be similar to:

	1.652: [GC cleanup 1213M->1213M(1885M), 0.0030492 secs]
	[Times: user=0.01 sys=0.00, real=0.00 secs]





In case when some heap regions that only contain garbage were discovered, the pause format can look a bit different, similar to:
	
	1.872: [GC cleanup 1357M->173M(1996M), 0.0015664 secs]
	[Times: user=0.01 sys=0.00, real=0.01 secs]
	1.874: [GC concurrent-cleanup-start]
	1.876: [GC concurrent-cleanup-end, 0.0014846 secs]						
### Evacuation Pause: Mixed




It’s a pleasant case when concurrent cleanup can free up entire regions in Old Generation, but it may not always be the case. After Concurrent Marking has successfully completed, G1 will schedule a mixed collection that will not only get the garbage away from the young regions, but also throw in a bunch of Old regions to the collection set.



A mixed Evacuation pause does not always immediately follow the end of the concurrent marking phase. There is a number of rules and heuristics that affect this. For instance, if it was possible to free up a large portion of the Old regions concurrently, then there is no need to do it.



There may, therefore, easily be a number of fully-young evacuation pauses between the end of concurrent marking and a mixed evacuation pause.



The exact number of Old regions to be added to the collection set, and the order in which they are added, is also selected based on a number of rules. These include the soft real-time performance goals specified for the application, the liveness and gc efficiency data collected during concurrent marking, and a number of configurable JVM options. The process of a mixed collection is largely the same as we have already reviewed earlier for fully-young gc, but this time we will also cover the subject of remembered sets.



Remembered sets are what allows the independent collection of different heap regions. For instance, when collecting region A,B and C, we have to know whether or not there are references to them from regions D and E to determine their liveness. But traversing the whole heap graph would take quite a while and ruin the whole point of incremental collection, therefore an optimization is employed. Much like we have the Card Table for independently collecting Young regions in other GC algorithms, we have Remembered Sets in G1.



As shown in the illustration below, each region has a remembered set that lists the references pointing to this region from the outside. These references will then be regarded as additional GC roots. Note that objects in Old regions that were determined to be garbage during concurrent marking will be ignored even if there are outside references to them: the referents are also garbage in that case.


![](04_13_g1-03.png)




What happens next is the same as what other collectors do: multiple parallel GC threads figure out what objects are live and which ones are garbage:

![](04_14_g1-04.png)




And, finally, the live objects are moved to survivor regions, creating new if necessary. The now empty regions are freed and can be used for storing objects in again.


![](04_15_g1-05-v2.png)





To maintain the remembered sets, during the runtime of the application, a Post-Write Barrier is issued whenever a write to a field is performed. If the resulting reference is cross-region, i.e. pointing from one region to another, a corresponding entry will appear in the Remembered Set of the target region. To reduce the overhead that the Write Barrier introduces, the process of putting the cards into the Remembered Set is asynchronous and features quite a number of optimizations. But basically it boils down to the Write Barrier putting the dirty card information into a local buffer, and a specialized GC thread picking it up and propagating the information to the remembered set of the referred region.



In the mixed mode, the logs publish certain new interesting aspects when compared to the fully young mode:



>
	[Update RS (ms)1: Min: 0.7, Avg: 0.8, Max: 0.9, Diff: 0.2, Sum: 6.1]
	[Processed Buffers2: Min: 0, Avg: 2.2, Max: 5, Diff: 5, Sum: 18]
	[Scan RS (ms)3: Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.8]
	[Clear CT: 0.2 ms]4
	[Redirty Cards: 0.1 ms]5

>
1. [Update RS (ms) – Since the Remembered Sets are processed concurrently, we have to make sure that the still-buffered cards are processed before the actual collection begins. If this number is high, then the concurrent GC threads are unable to handle the load. It may be, e.g., because of an overwhelming number of incoming field modifications, or insufficient CPU resources.
1. [Processed Buffers – How many local buffers each worker thread has processed.
1. [Scan RS (ms) – How long it took to scan the references coming in from remembered sets.
1. [Clear CT: 0.2 ms] – Time to clean the cards in the card table. Cleaning simply removes the “dirty” status that was put there to signify that a field was updated, to be used for Remembered Sets.
1. [Redirty Cards: 0.1 ms] – The time it takes to mark the appropriate locations in the card table as dirty. Appropriate locations are defined by the mutations to the heap that GC does itself, e.g. while enqueuing references.


### Summary



This should give one a sufficient basic understanding of how G1 functions. There are, of course, still quite some implementation details that we have left out for brevity, like dealing with humongous objects. All things considered, G1 is the most technologically advanced production-ready collector available in HotSpot. On top of that, it is being relentlessly improved by the HotSpot Engineers, with new optimizations or features coming in with newer java versions.



As we have seen, G1 addressed a wide range of problems that CMS has, starting from pause predictability and ending with heap fragmentation. Given an application not constrained by CPU utilization, but very sensitive to the latency of individual operations, G1 is very likely to be the best available choice for HotSpot users, especially when running the latest versions of Java. However, these latency improvements do not come for free: throughput overhead of G1 is larger thanks to the additional write barriers and more active background threads. So, if the application is throughput-bound or is consuming 100% of CPU, and does not care as much about individual pause durations, then CMS or even Parallel may be better choices.



The only viable way to select the right GC algorithm and settings is through trial and errors, but we do give the general guidelines in the next chapter.



Note that G1 will probably be the default GC for Java 9: http://openjdk.java.net/jeps/248


## Shenandoah



We have outlined all of the production-ready algorithms in HotSpot that you can just take and use right away. There is another one in the making, a so-called Ultra-Low-Pause-Time Garbage Collector. It is aimed for large multi-core machines with large heaps, the goal is to manage heaps of 100GB and larger with pauses of 10ms or shorter. This is traded off against throughput: the implementers are aiming at a no more than 10% of a performance penalty for applications with no GC pauses.



We are not going to go into the implementation details before the new algorithm is released as production-ready, but it also builds upon many of the ideas already covered in earlier chapters, such as concurrent marking and incremental collecting. It does a lot of things differently, however. It does not split the heap into multiple generations, instead having a single space only. That’s right, Shenandoah is not a generational garbage collector. This allows it to get rid of card tables and remembered sets. It also uses forwarding pointers and a Brooks style read barrier to allow for concurrent copying of live objects, thus reducing the number and duration of pauses.



A lot of more detailed and up-to-date information about Shenandoah is available on the Internet, for instance in this blog: https://rkennke.wordpress.com/













原文链接: [GC Algorithms: Implementations](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations)




