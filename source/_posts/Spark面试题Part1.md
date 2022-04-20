---
title: Spark面试题Part1
date: 2021-09-08 21:20:16
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "面试题" #分类
tags:   #标签
    - Spark
    - 面试题
---
spark core部分的重点
RDD的理解
shuffle部分的理解
spark作业的提交流程(DagScheduler,TaskScheduler,stage划分等)

#### RDD
```scala
 * A Resilient Distributed Dataset (RDD), the basic abstraction in Spark. Represents an immutable,
 * partitioned collection of elements that can be operated on in parallel. This class contains the
 * basic operations available on all RDDs, such as `map`, `filter`, and `persist`. In addition,
 * [[org.apache.spark.rdd.PairRDDFunctions]] contains operations available only on RDDs of key-value
 * pairs, such as `groupByKey` and `join`;
 * [[org.apache.spark.rdd.DoubleRDDFunctions]] contains operations available only on RDDs of
 * Doubles; and
 * [[org.apache.spark.rdd.SequenceFileRDDFunctions]] contains operations available on RDDs that
 * can be saved as SequenceFiles.
 * All operations are automatically available on any RDD of the right type (e.g. RDD[(Int, Int)])
 * through implicit.

 * Internally, each RDD is characterized by five main properties:
 *
 *  - A list of partitions
 *  - A function for computing each split
 *  - A list of dependencies on other RDDs
 *  - Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
 *  - Optionally, a list of preferred locations to compute each split on (e.g. block locations for
 *    an HDFS file)
```
#### Spark作业的提交流程
spark-submit
-> 构据提交程序的主类,通过反射构造这个类实例
-> 主类中的main静态方法开始运行
在这里面你构造的 SparkContext对象
它需要用到dagScheduler, taskScheudler, SparkEnv.
main代码里面 构造一个初始rdd,或多个rdd,
然后再做一些转换，最后触发一个或多个action.
由action开始构建一个job,分成stages,然后去运行。
sparkContext.submitjob -> dagScheduler.submitJob
DAGSchedulerEventProcessLoop
eventProcessLoop.post(JobSubmitted(
jobId, rdd, func2, partitions.toArray, callSite, waiter,
Utils.cloneProperties(properties)))
dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions,
callSite, listener, properties)
finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
val parents = getOrCreateParentStages(shuffleDeps, jobId)
val stage = new ResultStage(id, rdd, func, partitions, parents, jobId,
val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
/** Submits stage, but first recursively submits any missing parents. */
val missing = getMissingParentStages(stage).sortBy(_.id)
for (parent <- missing) {
submitStage(parent)
}
如果父RDD的一个Partition被一个子RDD的Partition所使用就是窄依赖，否则的话就是宽依赖。

// For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
// For ResultTask, serialize and broadcast (rdd, func).
taskBinary = sc.broadcast(taskBinaryBytes)
根据stage的属性和partitions来构造tasks[]
taskScheduler.submitTasks(new TaskSet(
tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties,
stage.resourceProfileId))
taskScheduler,taskset,maxTaskFailures构建成TasksetManager
设置任务集调度策略，调度模式有FAIR,FIFO两种，默认的是FIFO,将tasksetManager添加到
FIFOSchedulableBuilder中。

任务构造完成之后，分配到哪些节点？哪些节点有足够的资源。
资源分配，使用LocalBackend的reviveOffer方法，它的处理步骤如下：
1.使用ExecutorId,ExecutorHostName,freeCores创建WorkerOffer
2.调用TaskSchedulerImpl的resourceOffers方法分配资源
3.调用Executor的launchTask方法运行任务
Executor的launchTask方法，标志着任务执行阶段的开始。
1.创建TaskRunner,并将其与taskId,taskName以及serializedTask添加到runningTasks中。
2.TaskRunner实现了Runnable接口，最`1    后使用线程池来执行TaskRunner。
run方法的处理动作包含状态更新，任务反序列化和任务运行。
将driver提交的task在Executor上通过反序列化，更新依赖达到还原效果。
val(taskFiles, taskJars, taskBytes) = Task.deserializeWithDependencies(sd)
val value=task.run(taskId.toInt)

#### Shuffle过程分析
shuffle过程用于连接map任务的输出和reduce任务的输入，map任务的中间输出结果按照key值
哈希后分配给某一个reduce任务。
shuffle包含map任务端的shuffle write和reduce任务端的shuffle read两部分。
writer部分的任务个数由finalRdd的partition个数决定。
reader部分的任务个数由spark.sql.shuffle.partitions个数决定。
write阶段会把状态与shuffle输出的数据数量位置信息封装到MapStatus对象，然后发送到driver.
reader阶段会从driver请求mapstatus,然后读取数据。

spark1.2之前使用hashsuffleManager来处理shuffle过程，现更新为sortShuffleManager来处理。
因为早期的HashshuffleManager在shuffle write阶段生成的中间结果文件过多，会给IO造成过多的压力。
1.map任务会为每一个reduce task创建一个bucket,map阶段最终会创建M*R个bucket。
2.reduce任务从本地或者远端的map任务所在的BlockManager获取相应的bucket作为输入。
早期shuffle过程存在的问题：
1.map任务的中间结果先存入内存，后写入磁盘，对内存的开销很大，当一个节点上的map输出很大时，
容易造成OOM.
2.生成的中间文件过多，磁盘IO将成为性能瓶颈。
spark shuffle做的优化如下：
1.将map任务给每个partition的reduce任务输出的bucket合并到同一个文件当中。
2.map任务逐条输出计算结果，使用appendOnlyMap缓存与聚合算法对中间结果进行聚合，减少中间结果
占用的内存大小。
3.对SizeTrackingAppendOnlyMap和SizeTrackingPairBuffer等缓存进行溢出判断，当超出
threshold时将数据写入磁盘，防止内存溢出。
4.reduce任务对拉取到的map任务中间结果逐条读是不是一次性读取到内存，并在内存中进行聚合和排序，
减少了对内存的使用。
shuffleSortManager 有两种模式，一种是bypass模式，一种是普通模式。
bypass模式启动的动件是shuffle read task个数小于200（bypassMergeThreshold)，同时mapsideCombine
是false的情况下，才会开启bypass,bypass模式基本可以理解为和之前shuffle模式一样，只是最后会合并生成的临时文件，
生成一个数据文件和一个索引文件。
普通模式的sortShuffleSortManager的工作流程：
在该模式下，数据会先写入一个内存数据结构中，此时根据不同的shuffle算子，可能选用不同的数据结构。
如果是reduceByKey这种聚合类的shuffle算子，那么会选用Map数据结构，一边通过Map进行聚合，一边写入内存；
如果是join这种普通的shuffle算子，那么会选用Array数据结构，直接写入内存。接着每写一条数据进入内存数据结构之后，
就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，
然后清空内存数据结构。

在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。
默认的batch数量是10000条，排序好的数据，会以每批1万条数据的形式分批写入磁盘文件。
写入磁盘文件是通过Java的BufferedOutputStream实现的。首先会将数据缓冲在内存中，
当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘IO次数，提升性能。

一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。
最后会将之前所有的临时磁盘文件都进行合并，这就是merge过程，此时会将之前所有临时磁盘文件中的数据读取出来，
然后依次写入最终的磁盘文件之中。此外，由于一个task就只对应一个磁盘文件，
也就意味着该task为下游stage的task准备的数据都在这一个文件中，因此还会单独写一份索引文件，
其中标识了下游各个task的数据在文件中的start offset与end offset。

SortShuffleManager由于有一个磁盘文件merge的过程，因此大大减少了文件数量。

```scala
 * Sort-based shuffle has two different write paths for producing its map output files:
 *  - Serialized sorting: used when all three of the following conditions hold:
 *    1. The shuffle dependency specifies no map-side combine.
 *    2. The shuffle serializer supports relocation of serialized values (this is currently
 *       supported by KryoSerializer and Spark SQL's custom serializers).
 *    3. The shuffle produces fewer than or equal to 16777216 output partitions.
 *  - Deserialized sorting: used to handle all other cases.
 ExternalSorter
  *
 * @param aggregator optional Aggregator with combine functions to use for merging data
 * @param partitioner optional Partitioner; if given, sort by partition ID and then key
 * @param ordering optional Ordering to sort keys within each partition; should be a total ordering
 * @param serializer serializer to use when spilling to disk
```

#### Spark OOM分析
RDD已经可以序列化到磁盘，为什么我们Spark作业还会OOM?
合理地调整spark有关mergesort spill相关的参数
