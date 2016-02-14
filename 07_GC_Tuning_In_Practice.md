# 7. GC 调优(实战篇)






This chapter covers several typical performance problems that one may encounter with garbage collection. The examples given here are derived from real applications, but are simplified for the sake of clarity.



High Allocation Rate



Allocation rate is a term used when communicating the amount of memory allocated per time unit. Often it is expressed in MB/sec, but you can use PB/year if you feel like it. So that is all there is – no magic, just the amount of memory you allocate in your Java code measured over a period of time.



An excessively high allocation rate can mean trouble for your application’s performance. When running on a JVM, the problem will be revealed by garbage collection posing a large overhead.



How to Measure Allocation Rate?



One way to measure the allocation rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses similar to the following:
0.291: [GC (Allocation Failure) [PSYoungGen: 33280K->5088K(38400K)] 33280K->24360K(125952K), 0.0365286 secs] [Times: user=0.11 sys=0.02, real=0.04 secs] 
0.446: [GC (Allocation Failure) [PSYoungGen: 38368K->5120K(71680K)] 57640K->46240K(159232K), 0.0456796 secs] [Times: user=0.15 sys=0.02, real=0.04 secs] 
0.829: [GC (Allocation Failure) [PSYoungGen: 71680K->5120K(71680K)] 112800K->81912K(159232K), 0.0861795 secs] [Times: user=0.23 sys=0.03, real=0.09 secs]



From the GC log above, we can calculate the allocation rate as the difference between the sizes of the young generation after the completion of the last collection and before the start of the next one. Using the example above, we can extract the following information:



At 291 ms after the JVM was launched, 33,280 K of objects were created. The first minor GC event cleaned the young generation, after which there were 5,088 K of objects in the young generation left.



At 446 ms after launch, the young generation occupancy had grown to 38,368 K, triggering the next GC, which managed to reduce the young generation occupancy to 5,120 K.



At 829 ms after the launch, the size of the young generation was 71,680 K and the GC reduced it again to 5,120 K.



This data can then be expressed in the following table calculating the allocation rate as deltas of the young occupancy:



Event	Time	Young before	Young after	Allocated during	Allocation rate
1st GC	291ms	33,280KB	5,088KB	33,280KB	114MB/sec
2nd GC	446ms	38,368KB	5,120KB	33,280KB	215MB/sec
3rd GC	829ms	71,680KB	5,120KB	66,560KB	174MB/sec



Total	829ms	N/A	N/A	133,120KB	161MB/sec



Having this information allows us to say that this particular piece of software had the allocation rate of 161 MB/sec during the period of measurement.



Why Should I Care?



After measuring the allocation rate we can understand how the changes in allocation rate affect application throughput by increasing or reducing the frequency of GC pauses. First and foremost, you should notice that only minor GC pauses cleaning the young generation are affected. Neither the frequency nor duration of the GC pauses cleaning the old generation are directly impacted by the allocation rate, but instead by the promotion rate, a term that we will cover separately in the next section.



Knowing that we can focus only on Minor GC pauses, we should next look into the different memory pools inside the young generation. As the allocation takes place in Eden, we can immediately look into how sizing Eden can impact the allocation rate. So we can hypothesize that increasing the size of Eden will reduce the frequency of minor GC pauses and thus allow the application to sustain faster allocation rates.



And indeed, when running the same application with different Eden sizes using -XX:NewSize -XX:MaxNewSize & -XX:SurvivorRatio parameters, we can see a two-fold difference in allocation rates.



Re-running with 100 M of Eden reduces the allocation rate to below 100 MB/sec.



Increasing Eden size to 1 GB increases the allocation rate to just below 200 MB/sec.



If you are still wondering how this can be true – if you stop your application threads for GC less frequently you can do more useful work. More useful work also happens to create more objects, thus supporting the increased allocation rate.



Now, before you jump to the conclusion that “bigger Eden is better”, you should notice that the allocation rate might and probably does not directly correlate with the actual throughput of your application. It is a technical measurement contributing to throughput. The allocation rate can and will have an impact on how frequently your minor GC pauses stop application threads, but to see the overall impact, you also need to take into account major GC pauses and measure throughput not in MB/sec but in the business operations your application provides.



Give me an Example



Meet the demo application. Suppose that it works with an external sensor that provides a number. The application continuously updates the value of the sensor in a dedicated thread (to a random value, in this example), and from other threads sometimes uses the most recent value to do something meaningful with it in the processSensorValue() method:
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



The demo application is impacted by the GC not keeping up with the allocation rate. The ways to verify and solve the issue are given in the next sections.



Could my JVMs be Affected?



First and foremost, you should only be worried if the throughput of your application starts to decrease. As the application is creating too much objects that are almost immediately discarded, the frequency of minor GC pauses surges. Under enough of a load this can result in GC having a significant impact on throughput.



When you run into a situation like this, you would be facing a log file similar to the following short snippet extracted from the GC logs of the demo application introduced in the previous section. The application was launched as with the -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m command line arguments:
2.808: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003076 secs]
2.819: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003079 secs]
2.830: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0002968 secs]
2.842: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
2.853: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0004672 secs]
2.864: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003371 secs]
2.875: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003214 secs]
2.886: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
2.896: [GC (Allocation Failure) [PSYoungGen: 9760K->32K(10240K)], 0.0003588 secs]



What should immediately grab your attention is the frequency of minor GC events. This indicates that there are lots and lots of objects being allocated. Additionally, the post-GC occupancy of the young generation remains low, and no full collections are happening. These symptoms indicate that the GC is having significant impact to the throughput of the application at hand.



What is the Solution?



In some cases, reducing the impact of high allocation rates can be as easy as increasing the size of the young generation. Doing so will not reduce the allocation rate itself, but will result in less frequent collections. The benefit of the approach kicks in when there will be only a few survivors every time. As the duration of a minor GC pause is impacted by the number of surviving objects, they will not noticeably increase here.



The result is visible when we run the very same demo application with increased heap size and, with it, the young generation size, by using the -Xmx64m parameter:
2.808: [GC (Allocation Failure) [PSYoungGen: 20512K->32K(20992K)], 0.0003748 secs]
2.831: [GC (Allocation Failure) [PSYoungGen: 20512K->32K(20992K)], 0.0004538 secs]
2.855: [GC (Allocation Failure) [PSYoungGen: 20512K->32K(20992K)], 0.0003355 secs]
2.879: [GC (Allocation Failure) [PSYoungGen: 20512K->32K(20992K)], 0.0005592 secs]



However, just throwing more memory at it is not always a viable solution. Equipped with the knowledge on allocation profilers from the previous chapter, we may find out where most of the garbage is produced. Specifically, in this case, 99% are Doubles that are created with the readSensor method. As a simple optimization, the object can be replaced with a primitive double, and the null can be replaced with Double.NaN. Since primitive values are not actually objects, no garbage is produced, and there is nothing to collect. Instead of allocating a new object on the heap, a field in an existing object is directly overwritten.



The simple change (diff) will, in the demo application, almost completely remove GC pauses. In some cases, the JVM may be clever enough to remove excessive allocations itself by using the escape analysis technique. To cut a long story short, the JIT compiler may in some cases prove that a created object never “escapes” the scope it is created in. In such cases, there is no actual need to allocate it on the heap and produce garbage this way, so the JIT compiler does just that: it eliminates the allocation. See this benchmark for an example.



Premature Promotion



Before explaining the concept of premature promotion, we should familiarize ourselves with the concept it builds upon – the promotion rate. The promotion rate is measured in the amount of data propagated from the young generation to the old generation per time unit. It is often measured in MB/sec, similarly to the allocation rate.



Promoting long-lived objects from the young generation to the old is how JVM is expected to behave. Recalling the generation hypothesis we can now construct a situation where not only long-lived objects end up in the old generation. Such a situation, where objects with a short life expectancy are not collected in the young generation and get promoted to the old generation, is called premature promotion.



Cleaning these short-lived objects now becomes a job for major GC, which is not designed for frequent runs and results in longer GC pauses. This significantly affects the throughput of the application.



How to Measure Promotion Rate



One of the ways you can measure the promotion rate is to turn on GC logging by specifying -XX:+PrintGCDetails -XX:+PrintGCTimeStamps flags for the JVM. The JVM now starts logging the GC pauses just like in the following snippet:
0.291: [GC (Allocation Failure) [PSYoungGen: 33280K->5088K(38400K)] 33280K->24360K(125952K), 0.0365286 secs] [Times: user=0.11 sys=0.02, real=0.04 secs] 
0.446: [GC (Allocation Failure) [PSYoungGen: 38368K->5120K(71680K)] 57640K->46240K(159232K), 0.0456796 secs] [Times: user=0.15 sys=0.02, real=0.04 secs] 
0.829: [GC (Allocation Failure) [PSYoungGen: 71680K->5120K(71680K)] 112800K->81912K(159232K), 0.0861795 secs] [Times: user=0.23 sys=0.03, real=0.09 secs]



From the above we can extract the size of the young Generation and the total heap both before and after the collection event. Knowing the consumption of the young generation and the total heap, it is easy to calculate the consumption of the old generation as just the delta between the two. Expressing the information in GC logs as:



Event	Time	Young decreased	Total decreased	Promoted	Promotion rate
1st GC	291ms	28,192K	8,920K	19,272K	66.2 MB/sec
2nd GC	446ms	33,248K	11,400K	21,848K	140.95 MB/sec
3rd GC	829ms	66,560K	30,888K	35,672K	93.14 MB/sec



Total	829ms			76,792K	92.63 MB/sec
will allow us to extract the promotion rate for the measured period. We can see that on average the promotion rate was 92 MB/sec, peaking at 140.95 MB/sec for a while.



Notice that you can extract this information only from minor GC pauses. Full GC pauses do not expose the promotion rate as the change in the old generation usage in GC logs also includes objects cleaned by the major GC.



Why Should I Care?



Similarly to the allocation rate, the main impact of the promotion rate is the change of frequency in GC pauses. But as opposed to the allocation rate that affects the frequency of minor GC events, the promotion rate affects the frequency of major GC events. Let me explain – the more stuff you promote to the old generation the faster you will fill it up. Filling the old generation faster means that the frequency of the GC events cleaning the old generation will increase.


![](07_01_how-java-garbage-collection-works.png)






As we have shown in earlier chapters, full garbage collections typically require much more time, as they have to interact with many more objects, and perform additional complex activities such as defragmentation.



Give me an Example



Let us look at a demo application suffering from premature promotion. This app obtains chunks of data, accumulates them, and, when a sufficient number is reached, processes the whole batch at once:
public class PrematurePromotion {

   private static final Collection<byte[]> accumulatedChunks = new ArrayList<>();

   private static void onNewChunk(byte[] bytes) {
       accumulatedChunks.add(bytes);

       if(accumulatedChunks.size() > MAX_CHUNKS) {
           processBatch(accumulatedChunks);
           accumulatedChunks.clear();
       }
   }
}



The demo application is impacted by premature promotion by the GC. The ways to verify and solve the issue are given in the next sections.



Could my JVMs be Affected?



In general, the symptoms of premature promotion can take any of the following forms:



The application goes through frequent full GC runs over a short period of time.



The old generation consumption after each full GC is low, often under 10-20% of the total size of the old generation.



Facing the promotion rate approaching the allocation rate.



Showcasing this in a short and easy-to-understand demo application is a bit tricky, so we will cheat a little by making the objects tenure to the old generation a bit earlier than it happens by default. If we ran the demo with a specific set of GC parameters (-Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1), we would see this in the garbage collection logs:
2.176: [Full GC (Ergonomics) [PSYoungGen: 9216K->0K(10752K)] [ParOldGen: 10020K->9042K(12288K)] 19236K->9042K(23040K), 0.0036840 secs]
2.394: [Full GC (Ergonomics) [PSYoungGen: 9216K->0K(10752K)] [ParOldGen: 9042K->8064K(12288K)] 18258K->8064K(23040K), 0.0032855 secs]
2.611: [Full GC (Ergonomics) [PSYoungGen: 9216K->0K(10752K)] [ParOldGen: 8064K->7085K(12288K)] 17280K->7085K(23040K), 0.0031675 secs]
2.817: [Full GC (Ergonomics) [PSYoungGen: 9216K->0K(10752K)] [ParOldGen: 7085K->6107K(12288K)] 16301K->6107K(23040K), 0.0030652 secs]



At first glance it may seem that premature promotion is not the issue here. Indeed, the occupancy of the old generation seems to be decreasing on each cycle. However, if few or no objects were promoted, we would not be seeing a lot of full garbage collections.



There is a simple explanation for this GC behavior: while many objects are being promoted to the old generation, some existing objects are collected. This gives the impression that the old generation usage is decreasing, while in fact, there are objects that are constantly being promoted, triggering full GC.



What is the Solution?



In a nutshell, to fix this problem, we would need to make the buffered data fit into the young generation. There are two simple approaches for doing this. The first is to increase the young generation size by using -Xmx64m -XX:NewSize=32m parameters at JVM startup. Running the application with this change in configuration will make Full GC events much less frequent, while barely affecting the duration of minor collections:
2.251: [GC (Allocation Failure) [PSYoungGen: 28672K->3872K(28672K)] 37126K->12358K(61440K), 0.0008543 secs]
2.776: [GC (Allocation Failure) [PSYoungGen: 28448K->4096K(28672K)] 36934K->16974K(61440K), 0.0033022 secs]



Another approach in this case would be to simply decrease the batch size, which would also give a similar result. Picking the right solution heavily depends on what is really happening in the application. In some cases, business logic does not permit decreasing batch size. In this case, increasing available memory or redistributing in favor of the young generation might be possible.



If neither is a viable option, then perhaps data structures can be optimized to consume less memory. But the general goal in this case remains the same: make transient data fit into the young generation.



Weak, Soft and Phantom References



Another class of issues affecting GC is linked to the use of non-strong references in the application. While this may help to avoid an unwanted OutOfMemoryError in many cases, heavy usage of such references may significantly impact the way garbage collection affects the performance of your application.



Why Should I Care?



When using weak references, you should be aware of the way the weak references are garbage-collected. Whenever GC discovers that an object is weakly reachable, that is, the last remaining reference to the object is a weak reference, it is put onto the corresponding ReferenceQueue, and becomes eligible for finalization. One may then poll this reference queue and perform the associated cleanup activities. A typical example for such cleanup would be the removal of the now missing key from the cache.



The trick here is that at this point you can still create new strong references to the object, so before it can be, at last, finalized and reclaimed, GC has to check again that it really is okay to do this. Thus, the weakly referenced objects are not reclaimed for an extra GC cycle.



Weak references are actually a lot more common than you might think. Many caching solutions build the implementations using weak referencing, so even if you are not directly creating any in your code, there is a strong chance your application is still using weakly referenced objects in large quantities.



When using soft references, you should bear in mind that soft references are collected much less eagerly than the weak ones. The exact point at which it happens is not specified and depends on the implementation of the JVM. Typically the collection of soft references happens only as a last ditch effort before running out of memory. What it implies is that you might find yourself in situations where you face either more frequent or longer full GC pauses than expected, since there are more objects resting in the old generation.



When using phantom references, you have to literally do manual memory management in regards of flagging such references eligible for garbage collection. It is dangerous, as a superficial glance at the javadoc may lead one to believe they are the completely safe to use:



In order to ensure that a reclaimable object remains so, the referent of a phantom reference may not be retrieved: The get method of a phantom reference always returns null.



Surprisingly, many developers skip the very next paragraph in the same javadoc (emphasis added):



Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.



That is right, we have to manually clear() up phantom references or risk facing a situation where the JVM starts dying with an OutOfMemoryError. The reason why the Phantom references are there in the first place is that this is the only way to find out when an object has actually become unreachable via the usual means. Unlike with soft or weak references, you cannot resurrect a phantom-reachable object.
 



Give me an Example



Let us take a look at another demo application that allocates a lot of objects, which are successfully reclaimed during minor garbage collections. Bearing in mind the trick of altering the tenuring threshold from the previous section on promotion rate, we could run this application with -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1 and see this in GC logs:
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
2.059: [Full GC (Ergonomics)  20365K->19611K(22528K), 0.0654090 secs]
2.125: [Full GC (Ergonomics)  20365K->19711K(22528K), 0.0707499 secs]
2.196: [Full GC (Ergonomics)  20365K->19798K(22528K), 0.0717052 secs]
2.268: [Full GC (Ergonomics)  20365K->19873K(22528K), 0.0686290 secs]
2.337: [Full GC (Ergonomics)  20365K->19939K(22528K), 0.0702009 secs]
2.407: [Full GC (Ergonomics)  20365K->19995K(22528K), 0.0694095 secs]



As we can see, there are now many full collections, and the duration of the collections is an order of magnitude longer! Another case of premature promotion, but this time a tad trickier. The root cause, of course, lies with the weak references. Before we added them, the objects created by the application were dying just before being promoted to the old generation. But with the addition, they are now sticking around for an extra GC round so that the appropriate cleanup can be done on them. Like before, a simple solution would be to increase the size of the young generation by specifying -Xmx64m -XX:NewSize=32m:
2.328: [GC (Allocation Failure)  38940K->13596K(61440K), 0.0012818 secs]
2.332: [GC (Allocation Failure)  38172K->14812K(61440K), 0.0060333 secs]
2.341: [GC (Allocation Failure)  39388K->13948K(61440K), 0.0029427 secs]
2.347: [GC (Allocation Failure)  38524K->15228K(61440K), 0.0101199 secs]
2.361: [GC (Allocation Failure)  39804K->14428K(61440K), 0.0040940 secs]
2.368: [GC (Allocation Failure)  39004K->13532K(61440K), 0.0012451 secs]



The objects are now once again reclaimed during minor garbage collection.



The situation is even worse when soft references are used as seen in the next demo application. The softly-reachable objects are not reclaimed until the application risks getting an OutOfMemoryError. Replacing weak references with soft references in the demo application immediately surfaces many more Full GC events:
2.162: [Full GC (Ergonomics)  31561K->12865K(61440K), 0.0181392 secs]
2.184: [GC (Allocation Failure)  37441K->17585K(61440K), 0.0024479 secs]
2.189: [GC (Allocation Failure)  42161K->27033K(61440K), 0.0061485 secs]
2.195: [Full GC (Ergonomics)  27033K->14385K(61440K), 0.0228773 secs]
2.221: [GC (Allocation Failure)  38961K->20633K(61440K), 0.0030729 secs]
2.227: [GC (Allocation Failure)  45209K->31609K(61440K), 0.0069772 secs]
2.234: [Full GC (Ergonomics)  31609K->15905K(61440K), 0.0257689 secs]



And the king here is the phantom reference as seen in the third demo application. Running the demo with the same sets of parameters as before would give us results that are pretty similar as the results in the case with weak references. The number of full GC pauses would, in fact, be much smaller because of the difference in the finalization described in the beginning of this section.



However, adding one flag that disables phantom reference clearing (-Dno.ref.clearing=true) would quickly give us this:
4.180: [Full GC (Ergonomics)  57343K->57087K(61440K), 0.0879851 secs]
4.269: [Full GC (Ergonomics)  57089K->57088K(61440K), 0.0973912 secs]
4.366: [Full GC (Ergonomics)  57091K->57089K(61440K), 0.0948099 secs]



Exception in thread "main" java.lang.OutOfMemoryError: Java heap space



One must exercise extreme caution when using phantom references and always clear up the phantom reachable objects in a timely manner. Failing to do so will likely end up with an OutOfMemoryError. And trust us when we say that it is quite easy to fail at this: one unexpected exception in the thread that processes the reference queue, and you will have a dead application at your hand.



Could my JVMs be Affected?



As a general recommendation, consider enabling the -XX:+PrintReferenceGC JVM option to see the impact that different references have on garbage collection. If we add this to the application from the WeakReference example, we will see this:
2.173: [Full GC (Ergonomics) 2.234: [SoftReference, 0 refs, 0.0000151 secs]2.234: [WeakReference, 2648 refs, 0.0001714 secs]2.234: [FinalReference, 1 refs, 0.0000037 secs]2.234: [PhantomReference, 0 refs, 0 refs, 0.0000039 secs]2.234: [JNI Weak Reference, 0.0000027 secs][PSYoungGen: 9216K->8676K(10752K)] [ParOldGen: 12115K->12115K(12288K)] 21331K->20792K(23040K), [Metaspace: 3725K->3725K(1056768K)], 0.0766685 secs] [Times: user=0.49 sys=0.01, real=0.08 secs] 
2.250: [Full GC (Ergonomics) 2.307: [SoftReference, 0 refs, 0.0000173 secs]2.307: [WeakReference, 2298 refs, 0.0001535 secs]2.307: [FinalReference, 3 refs, 0.0000043 secs]2.307: [PhantomReference, 0 refs, 0 refs, 0.0000042 secs]2.307: [JNI Weak Reference, 0.0000029 secs][PSYoungGen: 9215K->8747K(10752K)] [ParOldGen: 12115K->12115K(12288K)] 21331K->20863K(23040K), [Metaspace: 3725K->3725K(1056768K)], 0.0734832 secs] [Times: user=0.52 sys=0.01, real=0.07 secs] 
2.323: [Full GC (Ergonomics) 2.383: [SoftReference, 0 refs, 0.0000161 secs]2.383: [WeakReference, 1981 refs, 0.0001292 secs]2.383: [FinalReference, 16 refs, 0.0000049 secs]2.383: [PhantomReference, 0 refs, 0 refs, 0.0000040 secs]2.383: [JNI Weak Reference, 0.0000027 secs][PSYoungGen: 9216K->8809K(10752K)] [ParOldGen: 12115K->12115K(12288K)] 21331K->20925K(23040K), [Metaspace: 3725K->3725K(1056768K)], 0.0738414 secs] [Times: user=0.52 sys=0.01, real=0.08 secs]



As always, this information should only be analyzed when you have identified that GC is having impact to either the throughput or latency of your application. In such case you may want to check these sections of the logs. Normally, the number of references cleared during each GC cycle is quite low, in many cases exactly zero. If this is not the case, however, and the application is spending a significant period of time clearing references, or just a lot of them are being cleared, then further investigation is required.



What is the Solution?



When you have verified the application actually is suffering from the mis-, ab- or overuse of either weak, soft or phantom references, the solution often involves changing the application’s intrinsic logic. This is very application specific and generic guidelines are thus hard to offer. However, some generic solutions to bear in mind are:



Weak references – if the problem is triggered by increased consumption of a specific memory pool, an increase in the corresponding pool (and possibly the total heap along with it) can help you out. As seen in the example section, increasing the total heap and young generation sizes alleviated the pain.



Phantom references – make sure you are actually clearing the references. It is easy to dismiss certain corner cases and have the clearing thread to not being able to keep up with the pace the queue is filled or to stop clearing the queue altogether, putting a lot of pressure to GC and creating a risk of ending up with an OutOfMemoryError.



Soft references – when soft references are identified as the source of the problem, the only real way to alleviate the pressure is to change the application’s intrinsic logic.



Other Examples



Previous chapters covered the most common problems related to poorly behaving GC. Unfortunately, there is a long list of more specific cases, where you cannot apply the knowledge from previous chapters. This section describes a few of the more unusual problems that you may face.



RMI & GC



When your application is publishing or consuming services over RMI, the JVM periodically launches full GC to make sure that locally unused objects are also not taking up space on the other end. Bear in mind that even if you are not explicitly publishing anything over RMI in your code, third party libraries or utilities can still open RMI endpoints. One such common culprit is for example JMX, which, if attached to remotely, will use RMI underneath to publish the data.



The problem is exposed by seemingly unnecessary and periodic full GC pauses. When you check the old generation consumption, there is often no pressure to the memory as there is plenty of free space in the old generation, but full GC is triggered, stopping the application threads.



This behavior of removing remote references via System.gc() is triggered by the sun.rmi.transport.ObjectTable class requesting garbage collection to be run periodically as specified in the sun.misc.GC.requestLatency() method.



For many applications, this is not necessary or outright harmful. To disable such periodic GC runs, you can set up the following for your JVM startup scripts:
java -Dsun.rmi.dgc.server.gcInterval=9223372036854775807L -Dsun.rmi.dgc.client.gcInterval=9223372036854775807L com.yourcompany.YourApplication



This sets the period after which System.gc() is run to Long.MAX_VALUE; for all practical matters, this equals eternity.



An alternative solution for the problem would to disable explicit calls to System.gc() by specifying -XX:+DisableExplicitGC in the JVM startup parameters. We do not however recommend this solution as it can have other side effects.



JVMTI tagging & GC



Whenever the application is run alongside with a Java Agent (-javaagent), there is a chance that the agent can tag the objects in the heap using JVMTI tagging. Agents can use tagging for various reasons that are not in the scope of this handbook, but there is a GC-related performance issue that can start affecting the latency and throughput of your application if tagging is applied to a large subset of objects inside the heap.



The problem is hidden in the native code where JvmtiTagMap::do_weak_oops iterates over all the tags during each garbage collection event and performs a number of not-so-cheap operations for all of them. To make things worse, this operation is performed sequentially and is not parallelized.



With a large number of tags, this implies that a large part of the GC process is now carried out in a single thread and all the benefits of parallelism disappear, potentially increasing the duration of GC pauses by an order of magnitude.



To check whether or not a particular agent can be the reason for extended GC pauses, you would need to turn on the diagnostic option of –XX:+TraceJVMTIObjectTagging. Enabling the trace will allow you to get an estimate of how much native memory the tag map consumes and how much time the heap walks take.



If you are not the author of the agent yourself, fixing the problem is often out of your reach. Apart from contacting the vendor of a particular agent you cannot do much. In case you do end up in a situation like this, recommend that the vendor clean up the tags that are no longer needed.



Humongous Allocations



Whenever your application is using the G1 garbage collection algorithm, a phenomenon called humongous allocations can impact your application performance in regards of GC. To recap, humongous allocations are allocations that are larger than 50% of the region size in G1.



Having frequent humongous allocations can trigger GC performance issues, considering the way that G1 handles such allocations:



If the regions contain humongous objects, space between the last humongous object in the region and the end of the region will be unused. If all the humongous objects are just a bit larger than a factor of the region size, this unused space can cause the heap to become fragmented.



Collection of the humongous objects is not as optimized by the G1 as with regular objects. It was especially troublesome with early Java 8 releases – until Java 1.8u40 the reclamation of humongous regions was only done during full GC events. More recent releases of the Hotspot JVM free the humongous regions at the end of the marking cycle during the cleanup phase, so the impact of the issue has been reduced significantly for newer JVMs.



To check whether or not your application is allocating objects in humongous regions, the first step would be to turn on GC logs similar to the following:
java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintReferenceGC -XX:+UseG1GC -XX:+PrintAdaptiveSizePolicy -Xmx128m MyClass



Now, when you check the logs and discover sections like these:
 0.106: [G1Ergonomics (Concurrent Cycles) request concurrent cycle initiation, reason: occupancy higher than threshold, occupancy: 60817408 bytes, allocation request: 1048592 bytes, threshold: 60397965 bytes (45.00 %), source: concurrent humongous allocation]
 0.106: [G1Ergonomics (Concurrent Cycles) request concurrent cycle initiation, reason: requested by GC cause, GC cause: G1 Humongous Allocation]
 0.106: [G1Ergonomics (Concurrent Cycles) initiate concurrent cycle, reason: concurrent cycle initiation requested]
 0.106: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 0.106: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 0, predicted base time: 10.00 ms, remaining time: 190.00 ms, target pause time: 200.00 ms]
you have evidence that the application is indeed allocating humongous objects. The evidence is visible in the cause for a GC pause being identified as G1 Humongous Allocation and in the “allocation request: 1048592 bytes” section, where we can see that the application is trying to allocate an object with the size of 1,048,592 bytes, which is 16 bytes larger than the 50% of the 2 MB size of the humongous region specified for the JVM.



The first solution for humongous allocation is to change the region size so that (most) of the allocations would not exceed the 50% limit triggering allocations in the humongous regions. The region size is calculated by the JVM during startup based on the size of the heap. You can override the size by specifying -XX:G1HeapRegionSize=XX in the startup script. The specified region size must be between 1 and 32 megabytes and has to be a power of two.



This solution can have side effects – increasing the region size reduces the number of regions available so you need to be careful and run extra set of tests to see whether or not you actually improved the throughput or latency of the application.



A more time-consuming but potentially better solution would be to understand whether or not the application can limit the size of allocations. The best tools for the job in this case are profilers. They can give you information about humongous objects by showing their allocation sources with stack traces.



Conclusion



With the enormous number of possible applications that one may run on the JVM, coupled with the hundreds of JVM configuration parameters that one may tweak for GC, there are astoundingly many ways in which the GC may impact your application’s performance.



Therefore, there is no real silver bullet approach to tuning the JVM to match the performance goals you have to fulfill. What we have tried to do here is walk you through some common (and not so common) examples to give you a general idea of how problems like these can be approached. Coupled with the tooling overview and with a solid understanding of how the GC works, you have all the chances of successfully tuning garbage collection to boost the performance of your application.


















原文链接:  [GC Tuning: In Practice](https://plumbr.eu/handbook/gc-tuning-in-practice)

