# 5. GC 调优(基础篇)


Tuning garbage collection is no different from any other performance-tuning activity. It is easy to fall into the trap of randomly tweaking one of the 200 GC-related JVM parameters or to start changing random parts of your application code. Instead, following a simple process will guarantee that you are actually moving towards the right target while being transparent about the progress:


GC调优(Tuning Garbage Collection)和其他性能调优行为并没有什么不同。一般人可能会陷入 200 多个 GC相关的JVM参数中, 漫无目的随便调整某个参数或者随便改变程序的一部分代码。但只要遵循一些简单的步骤，就能保证你朝着正确的目标前进:

1. State your performance goals
1. Run tests
1. Measure the results
1. Compare the results with the goals
1. If goals are not met, make a change and go back to running tests

<br/>

1. 列出性能目标(State your performance goals)
2. 执行测试(Run tests)
3. 衡量结果(Measure the results)
4. 与目标进行对比(Compare the results with the goals)
5. 如果没有达到目标, 修改配置, 然后继续测试(go back to running tests)





So, as the first step we need to set clear performance goals in regards of Garbage Collection. The goals fall into three categories, which are common to all performance monitoring and management topics:


所以,第一步我们需要制定明确的性能目标(关于GC方面的)。目标分为三类, 对所有性能监控和管理主题都是通用的:



- 延迟(Latency)
- 吞吐量(Throughput)
- 处理能力(Capacity)



After explaining the concepts in general, we will demonstrate how to apply these goals in the context of Garbage Collection. If you are already familiar with the concepts of latency, throughput and capacity, you may decide to skip the next section.


在解释概念之后,我们将演示如何在垃圾收集的情景下应用这些目标。如果您延迟、吞吐量和处理能力的概念很熟悉, 则可以决定跳过下一小节。



## 核心概念




##
##



Let us start with an example from manufacturing by observing an assembly line in a factory. The line is assembling bikes from ready-made components. Bikes are built on this line in a sequential manner. Monitoring the line in action, we measure that it takes four hours to complete a bike from the moment the frame enters the line until the assembled bike leaves the line at the other end.


让我们从一个示例开始从制造业通过观察在一家工厂装配线。线路从现成的组件组装自行车。自行车是建在这条线的顺序。监测在行动,我们测量,需要四个小时完成一个自行车从帧输入线,直到组装自行车离开线的另一端。



![](05_01_assembly-line01-compact.jpg)





Continuing our observations we can also see that one bike is assembled after each minute, 24 hours a day, every day. Simplifying the example and ignoring maintenance windows, we can forecast that in any given hour such an assembly line assembles 60 bikes.


继续我们的观察,我们还可以看到,一个自行车组装每一分钟后,一天24小时,每一天。简化和忽略维护窗口的例子中,我们可以预测,在任何给定的60小时这样的流水线组装自行车。



Equipped with these two measurements, we now possess crucial information about the current performance of the assembly line in regards of latency and throughput:


配备这两个测量值,我们现在拥有关键信息的当前性能装配线的延迟和吞吐量:



- Latency of the assembly line: 4 hours
- Throughput of the assembly line: 60 bikes/hour


——组装线的延迟:4小时
——组装线的吞吐量:60自行车/小时



Notice that latency is measured in time units suitable for the task at hand – anything from nanoseconds to millennia can be a good choice. Throughput of a system is measured in completed operations per time unit. Operations can be anything relevant to the specific system. In this example the chosen time unit was hours and operations were the assembled bikes.


注意,延迟时间测量单位适合手头的任务——从纳秒到几千年可以是一个不错的选择。系统的吞吐量是每单位时间以完成操作。操作可以是任何相关的特定系统。在本例中,选择时间单位是小时和操作组装自行车。



Having been equipped with the definitions of latency and throughput, let us carry out a performance tuning exercise in the very same factory. The demand for the bikes has been steady for a while and the assembly line has kept producing bikes with the latency of four hours and the throughput of 60 bikes/hour for months. Let us now imagine a situation where the sales team has been successful and the demand for the bikes suddenly doubles. Instead of the usual 60*24 = 1,440 bikes/day the customers demand twice as many. The performance of the factory is no longer satisfactory and something needs to be done.


已经装备了延迟和吞吐量的定义,让我们进行性能调优在同一工厂锻炼。自行车的需求一直在稳定一段时间,生产线一直生产自行车四小时的延迟和吞吐量60自行车/小时数月。现在让我们想象一个销售团队取得成功,自行车突然双打的需求。而不是通常的60 * 24 = 1440辆自行车/天,客户需求的两倍。工厂的性能不再是令人满意的和需要做的事情。



The factory manager seemingly correctly concludes that the latency of the system is not a concern – instead he should focus on the total number of bikes produced per day. Coming to this conclusion and assuming he is well funded, the hypothetical manager would immediately take the necessary steps to improve throughput by adding capacity.


工厂经理看似正确地得出结论,系统的延迟并不担心——他应该关注每天的自行车生产总数。这个结论和假设他是资金充足,假设经理将立即采取必要的措施改善吞吐量增加产能。



As a result we would now be observing not one but two identical assembly lines in the same factory. Both of these assembly lines would be assembling the very same bike every minute of every day. By doing this, our imaginary factory has doubled the number of bikes produced per day. Instead of the 1,440 bikes the factory is now capable of shipping 2,880 bikes each day. It is important to note that we have not reduced the time to complete an individual bike by even a millisecond – it still takes four hours to complete a bike from start to finish.


因此我们将观察两个相同的生产线在同一个工厂。这两个生产线将装配同一自行车每天的每一分钟。通过这样做,我们的虚拟工厂每天生产的自行车数量增加了一倍。而不是1440辆自行车的工厂现在每天能运送2880辆自行车。重要的是要注意,我们没有时间来完成一个人骑自行车减少了甚至一毫秒——它仍然需要四个小时完成一个自行车从开始到结束。



![](05_02_assembly-line02-compact.jpg)





In the example above a performance optimization task was carried out, coincidentally impacting both throughput and capacity. As in any good example we started by measuring the system’s current performance, then set a new target and optimized the system only in the aspects required to meet the target.


在上面的例子中进行了性能优化任务,巧合的是影响吞吐量和能力。在任何好的例子我们开始通过测量系统的当前性能,然后设定一个新目标,优化方面的系统只需要满足目标。



In this example an important decision was made – the focus was on increasing throughput, not on reducing latency. While increasing the throughput, we also needed to increase the capacity of the system. Instead of a single assembly line we now needed two assembly lines to produce the required quantity. So in this case the added throughput was not free, the solution needed to be scaled out in order to meet the increased throughput requirement.



在这个例子中一个重要的决定——重点是增加吞吐量,而不是减少延迟。在增加吞吐量的同时,我们还需要增加系统的容量。而非单一装配线我们现在需要两个流水线生产所需的数量。所以在这种情况下增加的吞吐量并不是免费的,需要向外扩展的解决方案以满足增加的吞吐量要求。



An important alternative should also be considered for the performance problem at hand. The seemingly non-related latency of the system actually hides a different solution to the problem. If the latency of the assembly line could have been reduced from 1 minute to 30 seconds, the very same increase of throughput would suddenly be possible without any additional capacity.



一个重要的选择也应考虑手头的性能问题。看似不相关的系统延迟实际上隐藏了一个不同的解决问题的办法。如果延迟的组装线可以从1分钟减少到30秒,同一增加吞吐量会突然有可能没有任何额外的能力。



Whether or not reducing latency was possible or economical in this case is not relevant. What is important is a concept very similar to software engineering – you can almost always choose between two solutions to a performance problem. You can either throw more hardware towards the problem or spend time improving the poorly performing code.


减少延迟是否可能或经济在这种情况下是不相关的。重要的是一个概念非常类似于软件工程——你几乎总是可以选择两个性能问题的解决方案。你可以把更多的硬件对问题或花时间改善表现糟糕的代码。


### Latency




Latency goals for the GC have to be derived from generic latency requirements. Generic latency requirements are typically expressed in a form similar to the following:


延迟目标GC必须来自通用的延迟需求。通用的延迟需求通常表达形式类似如下:



- All user transactions must respond in less than 10 seconds
- 90% of the invoice payments must be carried out in under 3 seconds
- Recommended products must be rendered to a purchase screen in less than 100 ms



——所有用户事务必须响应在不到10秒
- 90%的发票付款必须在3秒
——推荐产品必须购买屏幕呈现在不到100 ms



When facing performance goals similar to the above, we would need to make sure that the duration of GC pauses during the transaction does not contribute too much to violating the requirements. “Too much” is application-specific and needs to take into account other factors contributing to latency including round-trips to external data sources, lock contention issues and other safe points among these.


当面对性能目标与上述类似,我们需要确保GC暂停时间的事务不贡献太多违反要求。“太多”是特定于应用程序的,需要考虑到其他因素导致延迟包括往返外部数据源,锁争用在这些问题和其他安全点。




Let us assume our performance requirements state that 90% of the transactions to the application need to complete under 1,000 ms and no transaction can exceed 10,000 ms. Out of those generic latency requirements let us again assume that GC pauses cannot contribute more than 10%. From this, we can conclude that 90% of GC pauses have to complete under 100 ms, and no GC pause can exceed 1,000 ms. For simplicity’s sake let us ignore in this example multiple pauses that can occur during the same transaction.


让我们假设我们的性能需求状态,90%的应用程序需要完成的事务在1000 ms和事务不能超过10000的女士一般延迟需求让我们再次假设GC暂停不能贡献超过10%。由此,我们可以得出结论,90%的GC暂停完成在100 ms,女士没有GC暂停能超过1000为简单起见我们忽略可能发生在这个例子中多个停顿在同一事务。



Having formalized the requirement, the next step is to measure pause durations. There are many tools for the job, covered in more detail in the chapter on Tooling, but in this section, let us use GC logs, namely for the duration of GC pauses. The information required is present in different log snippets so let us take a look which parts of date/time data are actually relevant, using the following example:



正式的要求,下一步是衡量暂停时间。有许多工具工作,覆盖这一章更详细地工具,但在本节中,我们使用GC日志,即GC暂停期间。所需的信息存在于不同的日志片段让我们看一看哪些部分的日期/时间数据实际上是相关的,使用下面的例子:




	2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
			[PSYoungGen: 93677K->70109K(254976K)] 
			[ParOldGen: 499597K->511230K(761856K)] 
			593275K->581339K(1016832K),
			[Metaspace: 2936K->2936K(1056768K)]
		, 0.0713174 secs]
		[Times: user=0.21 sys=0.02, real=0.07 secs





The example above expresses a single GC pause triggered at 13:34:16 on June 4, 2015, just 2,578 ms after the JVM was started.


上面的例子表示一个GC暂停触发13:34:16 6月4日,2015年,后2578毫秒JVM启动。



The event stopped the application threads for 0.0713174 seconds. Even though it took 210 ms of CPU times on multiple cores, the important number for us to measure is the total stop time for application threads, which in this case, where parallel GC was used on a multi-core machine, is equal to a bit more than 70 ms. This specific GC pause is thus well under the required 100 ms threshold and fulfils both requirements.


0.0713174秒的事件停止应用程序线程。虽然花了210 ms的多核CPU时间,我们测量的重要的数字是总应用程序线程停止时间,在这种情况下,在多核机器上并行使用GC,等于70多一点这个特定的GC暂停女士因此远低于所需的100 ms阈值和满足需求。


Extracting information similar to the example above from all GC pauses, we can aggregate the numbers and see whether or not we are violating the set requirements for any of the pause events triggered.


提取信息从GC暂停所有类似于上面的例子,我们可以总数量,看看我们是否违反任何的设置要求暂停事件触发。



### Throughput




Throughput requirements are different from latency requirements. The only similarity that the throughput requirements share with latency is the fact that again, these requirements need to be derived from generic throughput requirements. Generic requirements for throughput can be similar to the following:



从延迟需求吞吐量需求是不同的。唯一相似的吞吐量需求与延迟的事实是,这些需求需要来自通用的吞吐量要求。通用要求吞吐量可以类似如下:



- The solution must be able to process 1,000,000 invoices/day
- The solution must support 1,000 authenticated users each invoking one of the functions A, B or C every five to ten seconds
- Weekly statistics for all customers have to be composed in no more than six hours each Sunday night between 12 PM and 6 AM


- 解决方案必须能够处理1000000发票/天
- 解决方案必须支持1000个经过身份验证的用户每调用一个函数,每五到十秒B或C
- 每周统计所有客户必须由不超过6个小时每个周日晚上12点和6点之间




So, instead of setting requirements for a single operation, the requirements for throughput specify how many operations the system must process in a given time unit. Similar to the latency requirements, the GC tuning part now requires determining the total time that can be spent on GC during the time measured. How much is tolerable for the particular system is again application-specific, but as a rule of thumb, anything over 10% would look suspicious.


因此,而不是单个操作的设置要求,吞吐量的要求指定多少操作系统必须处理在一个给定的时间单位。GC调优部分类似于延迟需求,现在需要确定的总时间,可以花在GC期间测量。可以承受多少的特定系统是特定于应用程序的,但作为一个经验法则,任何看起来可疑的10%以上。



Let us now assume that the requirement at hand foresees that the system processes 1,000 transactions per minute. Let us also assume that the total duration of GC pauses during any minute cannot exceed six seconds (or 10%) of this time.


现在让我们假设手头的需求预测,系统流程每分钟1000个事务。让我们还假设在任何一分钟的GC暂停的总持续时间不能超过6秒(或10%)。



Having formalized the requirements, the next step would be to harvest the information we need. The source to be used in the example is again GC logs, from which we would get information similar to the following:


有正式的需求,下一步将是获取我们需要的信息。源使用的例子是GC日志,从中我们可以得到类似于下面的信息:



	2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
			[PSYoungGen: 93677K->70109K(254976K)] 
			[ParOldGen: 499597K->511230K(761856K)] 
			593275K->581339K(1016832K), 
			[Metaspace: 2936K->2936K(1056768K)], 
		 0.0713174 secs] 
		 [Times: user=0.21 sys=0.02, real=0.07 secs





This time we are interested in user and system times instead of real time. In this case we should focus on 23 milliseconds (21 + 2 ms in user and system times) during which the particular GC pause kept CPUs busy. Even more important is the fact that the system was running on a multi-core machine, translating to the actual stop-the-world pause of 0.0713174 seconds, which is the number to be used in the following calculations.


这一次我们感兴趣的用户和系统时间而不是实时的。在这种情况下我们应该专注于23日毫秒(21 + 2女士在用户和系统时间)在特定的GC暂停让cpu繁忙。更重要的是,系统是运行在多核机器上,翻译的实际停止一切停顿0.0713174秒,这是用于下列数量计算。


Extracting the information similar to the above from the GC logs across the test period, all that is left to be done is to verify the total duration of the stop-the-world pauses during each minute. In case the total duration of the pauses does not exceed 6,000ms or six seconds in any of these one-minute periods, we have fulfilled our requirement.


提取从GC日志类似于上面的信息在测试期间,剩下要做的是验证的总持续时间每分钟期间停止一切停顿。以防的总持续时间停顿不超过6000毫秒或在任何一分钟6秒时间,满足我们的要求。



### 处理能力(Capacity)




Capacity requirements put additional constraints on the environment where the throughput and latency goals can be met. These requirements might be expressed either in terms of computing resources or in cold hard cash. The ways in which such requirements can be described can, for example, take the following form:

容量需求把额外的约束环境中实现目标的吞吐量和延迟。这些需求可能表达的计算资源或寒冷的现金。这样的要求可以被描述的方式,例如,采取以下形式:



- The system must be deployed on Android devices with less than 512 MB of memory
- The system must be deployed on Amazon EC2 The maximum required instance size must not exceed the configuration c3.xlarge (8 G, 4 cores)
- The monthly invoice from Amazon EC2 for running the system must not exceed $12,000



——系统必须部署在Android设备少于512 MB的内存
——系统必须部署在Amazon EC2实例所需的最大尺寸不得超过配置c3。超大(8 G,4芯)
——运行系统的月度发票从Amazon EC2不得超过12000美元



Thus, capacity has to be taken into account when fulfilling the latency and throughput requirements. With unlimited computing power, any kind of latency and throughput targets could be met, yet in the real world the budget and other constraints have a tendency to set limits on the resources one can use.


因此,能力时必须考虑满足延迟和吞吐量需求。与无限的计算能力,任何延迟和吞吐量目标能够实现,然而在现实世界中预算和其他约束倾向于限制一个可以使用的资源。


## 相关示例(Example)




Now that we have covered the three dimensions of performance tuning, we can start investigating setting and hitting GC performance goals in practice.


现在我们已经介绍了性能调优的三维空间,我们可以开始调查设置和GC性能目标在实践中。


For this purpose, let us take a look at an example code:


为此,让我们看一个示例代码:



	//imports skipped for brevity
	public class Producer implements Runnable {
	
	  private static ScheduledExecutorService executorService
			 = Executors.newScheduledThreadPool(2);
	
	  private Deque<byte[]> deque;
	  private int objectSize;
	  private int queueSize;
	
	  public Producer(int objectSize, int ttl) {
	    this.deque = new ArrayDeque<byte[]>();
	    this.objectSize = objectSize;
	    this.queueSize = ttl * 1000;
	  }
	
	  @Override
	  public void run() {
	    for (int i = 0; i < 100; i++) { 
			deque.add(new byte[objectSize]); 
			if (deque.size() > queueSize) {
		        deque.poll();
			}
	    }
	  }
	
	  public static void main(String[] args) 
			throws InterruptedException {
	    executorService.scheduleAtFixedRate(
			new Producer(200 * 1024 * 1024 / 1000, 5), 
			0, 100, TimeUnit.MILLISECONDS
		);
	    executorService.scheduleAtFixedRate(
			new Producer(50 * 1024 * 1024 / 1000, 120), 
			0, 100, TimeUnit.MILLISECONDS);
	    TimeUnit.MINUTES.sleep(10);
	    executorService.shutdownNow();
	  }
	}





The code submits two jobs to run every 100 ms. Each job emulates objects with a specific lifespan: it creates objects, lets them leave for a predetermined amount of time and then forgets about them, allowing GC to reclaim the memory.


代码提交两份工作运行每100每个工作模拟对象与一个特定的寿命女士:它创建对象,让他们去预定的时间,然后忘记,允许GC回收内存。



When running the example with GC logging turned on with the following parameters:


当运行这个例子与GC日志记录打开以下参数:



	-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps


we immediately see the impact of GC in the log files, similarly to the following:


我们立即看到的影响GC日志文件,类似于以下几点:




	2015-06-04T13:34:16.119-0200: 1.723: [GC (Allocation Failure) 
			[PSYoungGen: 114016K->73191K(234496K)] 
		421540K->421269K(745984K), 
		0.0858176 secs] 
		[Times: user=0.04 sys=0.06, real=0.09 secs] 

	2015-06-04T13:34:16.738-0200: 2.342: [GC (Allocation Failure) 
			[PSYoungGen: 234462K->93677K(254976K)] 
		582540K->593275K(766464K), 
		0.2357086 secs] 
		[Times: user=0.11 sys=0.14, real=0.24 secs] 

	2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics) 
			[PSYoungGen: 93677K->70109K(254976K)] 
			[ParOldGen: 499597K->511230K(761856K)] 
		593275K->581339K(1016832K), 
			[Metaspace: 2936K->2936K(1056768K)], 
		0.0713174 secs] 
		[Times: user=0.21 sys=0.02, real=0.07 secs]





Based on the information in the log we can start improving the situation with three different goals in mind:


基于日志中的信息我们可以改善这种情况有三个不同的目标:



1. Making sure the worst-case GC pause does not exceed a predetermined threshold
1. Making sure the total time during which application threads are stopped does not exceed a predetermined threshold
1. Reducing infrastructure costs while making sure we can still achieve reasonable latency and/or throughput targets



1。确保最坏的GC暂停不超过预定的阈值
1。确保在哪些应用程序线程停止的总时间不超过预定的阈值
1。降低基础设施成本,同时确保我们仍然可以实现合理的延迟和/或吞吐量目标




For this, the code above was run for 10 minutes on three different configurations providing three very different results summarized in the following table:


为此,上面的代码运行10分钟在三个不同的配置提供三个非常不同的结果总结在下表中:



<table class="data compact">
	<thead>
		<tr>
			<th><b>Heap</b></th>
			<th><b>GC Algorithm</b></th>
			<th><b>Useful work</b></th>
			<th><b>Longest pause</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>89.8%</td>
			<td><strong>560 ms</strong></td>
		</tr>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseParallelGC</td>
			<td>91.5%</td>
			<td>1,104 ms</td>
		</tr>
		<tr>
			<td>-Xmx8g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>66.3%</td>
			<td>1,610 ms</td>
		</tr>
	</tbody>
</table>






The experiment ran the same code with different GC algorithms and different heap sizes to measure the duration of Garbage Collection pauses with regards to latency and throughput. Details of the experiments and an interpretation of the results are given in the following chapters.



实验运行相同的代码不同的GC算法和不同的堆大小来衡量垃圾收集暂停期间关于延迟和吞吐量。实验的细节和结果的解释在以下章节。




Note that in order to keep the example as simple as possible only a limited amount of input parameters were changed, for example the experiments do not test on different number of cores or with a different heap layout.



注意,为了保持尽可能简单的例子只有数量有限的输入参数改变,例如实验不测试在不同的内核数或不同堆布局。


### Tuning for Latency

### 调优的延迟




Let us assume we have a requirement stating that all jobs must be processed in under 1,000 ms. Knowing that the actual job processing takes just 100 ms we can simplify and deduct the latency requirement for individual GC pauses. Our requirement now states that no GC pause can stop the application threads for longer than 900 ms. Answering this question is easy, one would just need to parse the GC log files and find the maximum pause time for an individual GC pause.



让我们假设我们有一个需求说明所有工作必须在处理1000年女士知道实际的工作处理只需要100 ms可以简化和扣除延迟要求个人GC暂停。我们现在的需求状态,没有GC暂停可以停止超过900的应用程序线程女士回答这个问题是很容易的,一个只需要解析GC日志文件并找到最大暂停时间为一个单独的GC暂停。



Looking again at the three configuration options used in the test:


再看这三个配置选项用于测试:



<table class="data compact">
	<thead>
		<tr>
			<th><b>Heap</b></th>
			<th><b>GC Algorithm</b></th>
			<th><b>Useful work</b></th>
			<th><b>Longest pause</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>89.8%</td>
			<td><strong>560 ms</strong></td>
		</tr>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseParallelGC</td>
			<td>91.5%</td>
			<td>1,104 ms</td>
		</tr>
		<tr>
			<td>-Xmx8g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>66.3%</td>
			<td>1,610 ms</td>
		</tr>
	</tbody>
</table>


we can see that there is one configuration that already matches this requirement. Running the code with:


我们可以看到,有一个配置已经匹配的要求。运行代码:




	java -Xmx12g -XX:+UseConcMarkSweepGC Producer


results in a maximum GC pause of 560 ms, which nicely passes the 900 ms threshold set for satisfying the latency requirement. If neither the throughput nor the capacity requirements are violated, we can conclude that we have fulfilled our GC tuning task and can finish the tuning exercise.


导致的最大GC暂停560 ms,这很好地通过了900 ms阈值设置为满足延迟的要求。如果违反吞吐量和能力需求,我们可以得出结论,我们完成GC调优任务并能完成调优运动。



### Tuning for Throughput

### 吞吐量调优



Let us assume that we have a throughput goal to process 13,000,000 jobs/hour. The example configurations used again give us a configuration where the requirement is fulfilled:


让我们假定我们有一个吞吐量目标/小时处理13000000个工作岗位。再次使用的示例配置给我们配置需求的实现:



<table class="data compact">
	<thead>
		<tr>
			<th><b>Heap</b></th>
			<th><b>GC Algorithm</b></th>
			<th><b>Useful work</b></th>
			<th><b>Longest pause</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>89.8%</td>
			<td>560 ms</td>
		</tr>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseParallelGC</td>
			<td><strong>91.5%</strong></td>
			<td>1,104 ms</td>
		</tr>
		<tr>
			<td>-Xmx8g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>66.3%</td>
			<td>1,610 ms</td>
		</tr>
	</tbody>
</table>





Running this configuration as:


运行此配置为:



	java -Xmx12g -XX:+UseParallelGC Producer


we can see that the CPUs are blocked by GC for 8.5% of the time, leaving 91.5% of the computing power for useful work. For simplicity’s sake we will ignore other safe points in the example. Now we have to take into account that:


我们可以看到,cpu是8.5%的时间被GC,留下91.5%的计算能力有用的工作。为简单起见,我们将忽略其他安全点的例子。现在我们必须考虑:



1. One job is processed in 100 ms by a single core
1. Thus, in one minute, 60,000 jobs could be processed by one core
1. In one hour, a single core could thus process 3.6 M jobs
1. We have four cores available, which could thus process 4 x 3.6 M = 14.4 M jobs in an hour



1。100年的一个工作是处理由一个核心
1。因此,在一分钟,可以处理60000个工作岗位的核心之一
1。在一个小时,一个核心可能因此流程3.6个就业岗位
1。我们有四个核心可用,因此过程4 x 3.6米= 14.4米的工作一个小时



With this amount of theoretical processing power we can make a simple calculation and conclude that during one hour we can in reality process 91.5% of the 14.4 M theoretical maximum resulting in 13,176,000 processed jobs/hour, fulfilling our requirement.


这个数量的理论我们可以做一个简单的计算处理能力和得出结论,一个小时我们可以在现实过程中91.5%的理论最大导致13176000 14.4 /小时处理工作,实现我们的需求。


It is important to note that if we simultaneously needed to fulfill the latency requirements set in the previous section, we would be in trouble, as the worst-case latency for this case is close to two times of the previous configuration. This time the longest GC pause on record was blocking the application threads for 1,104 ms.


同时需要注意的是,如果我们需要满足延迟需求设置在前一节中,我们就有麻烦了,最坏延迟对于这种情况以前配置的接近两倍。这一次GC暂停时间最长的记录是阻止应用程序线程为1104 ms。


### Tuning for Capacity

### 调优的能力




Let us assume we have to deploy our solution to the commodity-class hardware with up to four cores and 10 G RAM available. From this we can derive our capacity requirement that the maximum heap space for the application cannot exceed 8 GB. Having this requirement in place, we would need to turn to the third configuration on which the test was run:


让我们假设我们必须将我们的解决方案部署到商品硬件与四核和10 G RAM可用。从这一点来看,我们可以得出我们的能力要求应用程序的最大的堆空间不能超过8 GB。有这个需求,我们需要求助于第三配置的测试运行:



<table class="data compact">
	<thead>
		<tr>
			<th><b>Heap</b></th>
			<th><b>GC Algorithm</b></th>
			<th><b>Useful work</b></th>
			<th><b>Longest pause</b></th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>89.8%</td>
			<td>560 ms</td>
		</tr>
		<tr>
			<td>-Xmx12g</td>
			<td>-XX:+UseParallelGC</td>
			<td>91.5%</td>
			<td>1,104 ms</td>
		</tr>
		<tr>
			<td><strong>-Xmx8g</strong></td>
			<td>-XX:+UseConcMarkSweepGC</td>
			<td>66.3%</td>
			<td>1,610 ms</td>
		</tr>
	</tbody>
</table>





The application is able to run on this configuration as


应用程序能够运行在这个配置


	java -Xmx8g -XX:+UseConcMarkSweepGC Producer


but both the latency and especially throughput numbers fall drastically:



但是延迟和特别是吞吐量数字大幅下跌:



- GC now blocks CPUs from doing useful work a lot more, as this configuration only leaves 66.3% of the CPUs for useful work. As a result, this configuration would drop the throughput from the best-case-scenario of 13,176,000 jobs/hour to a meager 9,547,200 jobs/hour
- Instead of 560 ms we are now facing 1,610 ms of added latency in the worst case



现在- GC块cpu做有用的工作很多,这个配置只有66.3%的cpu有用的工作。因此,这个配置将吞吐量从13176000人/小时的最好的情况不足9547200人/小时
——而不是560 ms我们现在面临1610毫秒的延迟在最坏的情况下补充道





Walking through the three dimensions it is indeed obvious that you cannot just optimize for “performance” but instead need to think in three different dimensions, measuring and tuning both latency and throughput, and taking the capacity constraints into account.



走在三维空间中确实明显,你不能优化的“业绩”,而是需要考虑三个不同维度,测量和调优延时和吞吐量,并考虑能力约束。




原文链接:  [GC Tuning: Basics](https://plumbr.eu/handbook/gc-tuning)



