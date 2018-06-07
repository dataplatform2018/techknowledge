# spark

## Hadoop MRv1的局限

​    早在Hadoop1.0版本，当时采用的是MRv1版本的MapReduce编程模型。MRv1版本的实现都封装在org.apache.hadoop.mapred包中，MRv1的Map和Reduce是通过接口实现的。MRv1包括三个部分：

- 运行时环境（JobTracker和TaskTracker）；
- 编程模型（MapReduce）；
- 数据处理引擎（Map任务和Reduce任务）。

​        MRv1存在以下不足：

- **可扩展性差**：在运行时，JobTracker既负责资源管理又负责任务调度，当集群繁忙时，JobTracker很容易成为瓶颈，最终导致它的可扩展性问题。
- **可用性差**：采用了单节点的Master，没有备用Master及选举操作，这导致一旦Master出现故障，整个集群将不可用。
- **资源利用率低**：TaskTracker 使用slot等量划分本节点上的资源量。slot代表计算资源（CPU、内存等）。一个Task 获取到一个slot 后才有机会运行，Hadoop 调度器负责将各个TaskTracker 上的空闲slot 分配给Task 使用。一些Task并不能充分利用slot，而其他Task也无法使用这些空闲的资源。slot 分为Map slot 和Reduce slot 两种，分别供MapTask 和Reduce Task 使用。有时会因为作业刚刚启动等原因导致MapTask很多，而Reduce Task任务还没有调度的情况，这时Reduce slot也会被闲置。
- **不能支持多种MapReduce框架**：无法通过可插拔方式将自身的MapReduce框架替换为其他实现，如Spark、Storm等。

![1528297414298](img/1528297414298.png)

​	Apache为了解决以上问题，对Hadoop升级改造，MRv2最终诞生了。MRv2中，重用了MRv1中的编程模型和数据处理引擎。但是运行时环境被重构了。JobTracker被拆分成了通用的资源调度平台（ResourceManager，简称RM）、节点管理器（NodeManager）和负责各个计算框架的任务调度模型（ApplicationMaster，简称AM）。ResourceManager依然负责对整个集群的资源管理，但是在任务资源的调度方面只负责将资源封装为Container分配给ApplicationMaster 的一级调度，二级调度的细节将交给ApplicationMaster去完成，这大大减轻了ResourceManager 的压力，使得ResourceManager 更加轻量。NodeManager负责对单个节点的资源管理，并将资源信息、Container运行状态、健康状况等信息上报给ResourceManager。ResourceManager 为了保证Container的利用率，会监控Container，如果Container未在有限的时间内使用，ResourceManager将命令NodeManager杀死Container，以便于将资源分配给其他任务。MRv2的核心不再是MapReduce框架，而是Yarn。在以Yarn为核心的MRv2中，MapReduce框架是可插拔的，完全可以替换为其他MapReduce实现，比如Spark、Storm等。MRv2的示意如图2所示。 

![1528297448361](img/1528297448361.png)

​	Hadoop MRv2虽然解决了MRv1中的一些问题，但是由于对HDFS的频繁操作（包括计算结果持久化、数据备份、资源下载及Shuffle等）导致磁盘I/O成为系统性能的瓶颈，因此只适用于离线数据处理或批处理，而不能支持对迭代式、流式数据的处理。 

## Spark的特点

​	spark看到MRv2的问题，对MapReduce做了大量优化，总结如下：

- **减少磁盘I/O**：随着实时大数据应用越来越多，Hadoop作为离线的高吞吐、低响应框架已不能满足这类需求。HadoopMapReduce的map端将中间输出和结果存储在磁盘中，reduce端又需要从磁盘读写中间结果，势必造成磁盘IO成为瓶颈。Spark允许将map端的中间输出和结果存储在内存中，reduce端在拉取中间结果时避免了大量的磁盘I/O。Hadoop Yarn中的ApplicationMaster申请到Container后，具体的任务需要利用NodeManager从HDFS的不同节点下载任务所需的资源（如Jar包），这也增加了磁盘I/O。Spark将应用程序上传的资源文件缓冲到Driver本地文件服务的内存中，当Executor执行任务时直接从Driver的内存中读取，也节省了大量的磁盘I/O。
- **增加并行度**：由于将中间结果写到磁盘与从磁盘读取中间结果属于不同的环节，Hadoop将它们简单的通过串行执行衔接起来。Spark把不同的环节抽象为Stage，允许多个Stage既可以串行执行，又可以并行执行。
- **避免重新计算**：当Stage中某个分区的Task执行失败后，会重新对此Stage调度，但在重新调度的时候会过滤已经执行成功的分区任务，所以不会造成重复计算和资源浪费。
- **可选的Shuffle排序**：HadoopMapReduce在Shuffle之前有着固定的排序操作，而Spark则可以根据不同场景选择在map端排序或者reduce端排序。
- **灵活的内存管理策略**：Spark将内存分为堆上的存储内存、堆外的存储内存、堆上的执行内存、堆外的执行内存4个部分。Spark既提供了执行内存和存储内存之间是固定边界的实现，又提供了执行内存和存储内存之间是“软”边界的实现。Spark默认使用“软”边界的实现，执行内存或存储内存中的任意一方在资源不足时都可以借用另一方的内存，最大限度的提高资源的利用率，减少对资源的浪费。Spark由于对内存使用的偏好，内存资源的多寡和使用率就显得尤为重要，为此Spark的内存管理器提供的Tungsten实现了一种与操作系统的内存Page非常相似的数据结构，用于直接操作操作系统内存，节省了创建的Java对象在堆中占用的内存，使得Spark对内存的使用效率更加接近硬件。Spark会给每个Task分配一个配套的任务内存管理器，对Task粒度的内存进行管理。Task的内存可以被多个内部的消费者消费，任务内存管理器对每个消费者进行Task内存的分配与管理，因此Spark对内存有着更细粒度的管理。
- **检查点支持**：Spark的RDD之间维护了血缘关系（lineage），一旦某个RDD失败了，则可以由父RDD重建。虽然lineage可用于错误后RDD的恢复，但对于很长的lineage来说，恢复过程非常耗时。如果应用启用了检查点，那么在Stage中的Task都执行成功后，SparkContext将把RDD计算的结果保存到检查点，这样当某个RDD执行失败后，在由父RDD重建时就不需要重新计算，而直接从检查点恢复数据。
- **易于使用**。Spark现在支持Java、Scala、Python和R等语言编写应用程序，大大降低了使用者的门槛。自带了80多个高等级操作符，允许在Scala，Python，R的shell中进行交互式查询。
- **支持交互式**：Spark使用Scala开发，并借助于Scala类库中的Iloop实现交互式shell，提供对REPL（Read-eval-print-loop）的实现。
- **支持SQL查询**。在数据查询方面，Spark支持SQL及Hive SQL，这极大的方便了传统SQL开发和数据仓库的使用者。
- **支持流式计算**：与MapReduce只能处理离线数据相比，Spark还支持实时的流计算。Spark依赖SparkStreaming对数据进行实时的处理，其流式处理能力还要强于Storm。
- **可用性高**。Spark自身实现了Standalone部署模式，此模式下的Master可以有多个，解决了单点故障问题。Spark也完全支持使用外部的部署模式，比如YARN、Mesos、EC2等。
- **丰富的数据源支持**：Spark除了可以访问操作系统自身的文件系统和HDFS，还可以访问Kafka、Socket、Cassandra、HBase、Hive、Alluxio（Tachyon）以及任何Hadoop的数据源。这极大地方便了已经使用HDFS、HBase的用户顺利迁移到Spark。
- **丰富的文件格式支持**：Spark支持文本文件格式、Csv文件格式、Json文件格式、Orc文件格式、Parquet文件格式、Libsvm文件格式，也有利于Spark与其他数据处理平台的对接。