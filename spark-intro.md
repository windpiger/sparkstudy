##Spark介绍

###什么是Spark？
  
  Spark是一个通用的大数据处理框架，2009年诞生于UC伯克利的AMPLab，2010年开源并于2013年成为Apache顶级项目。
  
  Spark具有如下特点:
  
  - **快速**
   
     Spark计算速度相对于Hadoop的MapReduce计算框架有10x~100x的提升。
    
    * 从框架上看，MapReduce执行一个job的流程包含map阶段(从存储读取数据，如HDFS)和reduce两个阶段(将结果写入存储，如HDFS)。如果要实现一个复杂的DAG，会生成多个job，job之间通过HDFS存储作为衔接，这样会导致整个DAG执行的非常慢，特别对于一些有迭代计算的场景，MapReduce会频繁的进行HDFS读写操作，很难满足需求。而Spark可以将数据存储在内存中，并且支持cyclic data flow，大大减少了DAG中间数据的读写，从而大大减少了运行时间；
    
    * MapReduce是多进程模型，虽然可以更细粒度控制task占用的资源，但是JVM启动会消耗更多的时间，Spark则采用的是多线程模型，task启动快，不同的task可以共享内存;
        
    * 目前Spark 2.0很多功能都是基于SQL，充分利用了SQL优化策略对DAG进行全局优化;
    
    * Tungsten Project专门针对Spark的CPU/内存利用率方面做了很多优化，比如whole stage codegen
   
  - **易用**
  	
  	* Spark支持Scala/Java/Python/R语言
 	
	* 提供超过80多个高级算子，用户可以很方便的对算子进行组合很快完成应用的实现
	
	   如wordcount
	   
	   ```
	    val rdd = spark.sparkContext.textFile("/README.md")
	    val counts = rdd.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)
	    counts.saveAsTextFile("/results")
	   ```
	
	* 可以通过Scala/Python/R的shell进行交互式的使用
	
	* Spark 2.0中StructStreaming/MLlib等都基于SparkSQL，接口基本统一到DataSet/DataFrame，API简单，使得编程更容易
  
  - **通用**
    
    * Spark包含SparkSQL/StructStreaming/MLlib/GraphX，能够处理各种大数据处理需求，如ETL离线处理、流式计算、机器学习、图计算等。
      
  - **融合**
  
    * 不仅有自己的standalone模式，也可以运行在Yarn/Mesos等资源调度框架之上
    
    * 可以读写HBase/HDFS/Cassandra/OSS/S3/Hive/Alluxio等存储
    

###Spark框架

#### 框架图
   
   Spark框架是由一个核心执行引擎SparkCore，以及上层组件SparkSQL/StructStreaming/MLlib/GraphX,如下图所示：
   
   ![based Spark 2.0](https://github.com/windpiger/sparkstudy/blob/master/pic/sparkgit.jpeg) 
   
   上图是Spark 2.0的框架，从图上可以看出Spark可以读写各种存储系统，它可以基于Yarn/Mesos做资源调度，也可以有自己的资源管理standalone模式，所以Spark可以很容易的融入Hadoop的生态体系中。
   
   上层组件共用同一套执行引擎SparkCore，有很多优点：
   
   * 当SparkCore做了优化(Tungsten Project)会立即体现在上层的每个组件中，Spark 2.0 很多组件基于SparkSQL，SparkSQL的Catalyst优化也会惠及上层组件;
   
   * 用户只需维护一套系统，就可以进行ETL、流式计算、机器学习等不同的处理模型。
   
#### 组件

   
   |名称|描述| 
   |--|--|-- 
   |SparkCore|核心执行引擎，负责Job/Task调度、存储管理、网络传输、容错处理等，对外提供RDD级别 API|
   |SparkSQL|处理结构化数据，提供sql语句以及DataFrame/DataSet两种使用方式，支持Hive/json/parquet等数据源，一般做离线ETL处理/交互式查询|
   |StructStreaming|基于SparkSQL的流处理引擎，充分利用SparkSQL的Catalyst来优化性能，并且使用DataSet/DataFrame的api来实现流处理逻辑，简单易用|
   |MLlib|提供机器学习相关的算法，Spark在迭代计算方面有很大优势，运行速度很快|
   |GrapX|图计算，支持很多图计算的操作|
   
### Spark编程模型

Spark有两种编程模型，一种是基于RDD的，另一种是基于DataSet/DataFrame.

#### RDD

   RDD(Resilient Distributed Dataset-弹性分布式数据集)是Spark中非常核心的数据结构，它是对各种数据集的一种封装抽象，可以将HDFS/本地文件/HBase等数据封装成RDD，也可以通过对一些RDD进行相关操作来生成新的RDD，Spark的执行引擎简单理解为将数据集封装成RDD，然后对RDD做一系列的操作处理，最后产生结果输出。

##### RDD属性

   RDD是对数据集的抽象描述，它是从以下几个方面来描述数据集:
   
   - partitions:如何对数据集进行分区，一个RDD包含多个分区 
     
     如HadoopRDD的分区即对应Hadoop的split，Spark会根据分区的个数来生成/调度task.
     
     ![partition](https://github.com/windpiger/sparkstudy/blob/master/pic/rdd.jpeg)
   
   - compute function: 如何读取数据集的分区中的每条数据
   
   - dependencies: 如何从父RDD生成本RDD，即依赖关系
      
      dependency描述了RDD的血缘关系(lineage)，即当前的RDD如何由父RDD生成，例如当某个RDD丢失可以通过dependency从父RDD重新计算得到，即lineage实现了数据的容错机制。
      
      dependency有两种类型：
     
   		
   	* 窄依赖
   	   
   	   父RDD的每个分区至多对应一个子RDD分区，则为窄依赖。
   	   
   	   当通过lineage对某个子RDD分区进行数据恢复时，只需要计算与之对应的一个父RDD分区即可，不需要计算其它父RDD的分区，减少了额外的开销。
   	   
   	   窄依赖的RDD可以进行pipeline操作.
   	   
   	   |类名|描述|
   	   |--|--|
   	   |OneToOneDependency|一个子RDD分区对应一个父RDD分区|
   	   |RangeDependency| a set of child partitions depend on a set of parent partitions|
   	   |PruneDependency|the child RDD contains a subset of partitions of the parents|
   	   
   	   ![narrowDep](https://github.com/windpiger/sparkstudy/blob/master/pic/narrowdep.jpeg)
   	   
   	   上图中两表的数据具有同样的分区，即相同key的数据分区在同一台机器上面，这种join场景不需要shuffle。
   	
   	* 宽依赖
   		
   	    父RDD至少有一个分区对应多个子RDD分区，则为宽依赖
   	    
   	    当通过lineage对某个子RDD分区进行数据恢复时，需要计算该子RDD分区对应的多个父RDD分区，所以相对窄依赖，宽依赖的数据恢复消耗会大点，所以一般对于ShuffleRDD最好还是使用cache/checkpoint等方式。
   	    
   	    宽依赖是Spark进行Job Stage划分的依据.
   	    
   	   |类名|描述|
   	   |--|--|
   	   |ShuffleDependency|shuffle操作中ShuffleRDD中使用|
   
   ![narrowDep](https://github.com/windpiger/sparkstudy/blob/master/pic/shuffledep.jpeg)
   
   - preferred locations: 分区所在的位置，Spark可以通过分区的位置来进行调度达到本地计算的目的
   
   - partitioner: shuffle的时候如何对RDD进行分区，可以决定reducer的个数
   
#####RDD算子
   
   &nbsp;&nbsp;&nbsp;&nbsp;Spark提供了很多对RDD进行处理的算子，用户可以通过各种不同的算子进行组合来完成数据处理工作。
   RDD算子可以分为两大类，transform和action.transform算子不会触发Spark Job的提交，它作用于RDD后会生成一个新的RDD，只有action算子才能触发，最终输出结果，常用的算子如下：
   
   |类型|算子|
   |--|--|
   |transform|map/filter/flatMap/groupByKey/reduceByKey/join等|
   |action|reduce/collect/count/take/saveAsTextFile等|
     
   如:
   
   ```
   	  // transform 生成新的RDD
      val rdd1 = spark.sparkContext.textFile("/README.md")
      val rdd2 = rdd1.flaMap(line => line.split(" "))
      val rdd3 = rdd2.map(word => (word,1))
      val rdd4 = rdd3.reduceByKey(_ + _)
      
      // action 触发作业提交执行
      val result = rdd4.saveAsTextFile("/results")
   ```
   
   了解了RDD的属性和算子，SparkCore的运行都是围绕RDD来进行的，流程如下：
   
   ![spark-core-logic](https://github.com/windpiger/sparkstudy/blob/master/pic/spark-core-logic.jpeg)
   
   如上图所示，当用户代码中有RDD的action操作，会触发作业的提交，通过DAGScheduler和TaskScheduler，最终到task的真正执行。
   
   - Task
   
     Task是一个执行单元，Spark会根据RDD的分区个数来确定Task个数，有两种Task类型:
     
     |类型|描述|
     |-|-|
     |ShuffleMapTask|输入RDD，经过一系列的transform算子的处理，输出按照partitioner进行shuffle|
     |ResultTask|输入RDD(如ShuffleMapTask的输出)，经过一系列的transform算子的处理，输出最终结果|
    
     `Spark的执行并不是对每个transform/action算子生成一个task，而是根据ShuffleDependency(宽依赖)为边界，边界之内的所有窄依赖的算子在一个ShuffleMapTask内以pipeline的方式执行(如图所示)`
     
   - DAGScheduler
   
    action算子触发作业提交(DAGScheduler.submitJob)后，DAGScheduler会根据RDD的血缘关系(lineage/dependency),逆向寻找(子->父)shuffleDependency，一旦遇到shuffleDependency就会生成一个Stage(两种类型ResultStage/ShuffleMapStage，如图所示)，所有Stage都划分完成后，Spark会
按顺序执行各个Stage(父Stage->子Stage,父Stage跑完才能跑子Stage)里面的task
    
   - TaskScheduler
   
   Stage里面的Task通过TaskScheduler进行调度，Spark包含两种调度策略(FAIR/FIFO,默认FIFO)
   
#### DataSet/DataFrame    
 
  Spark 2.x版本弱化了RDD的概念，使用更高级的DataSet/DataFrame编程接口，DataSet/DataFrame的编程模型跟RDD类似。
  
  ![rdd_df](https://github.com/windpiger/sparkstudy/blob/master/pic/rdd_df.jpeg)
   
  ![rdd_df2](https://github.com/windpiger/sparkstudy/blob/master/pic/rdd_df2.jpeg)
   
  - DataSet
    
     * DataSet是一个强类型的数据集，对数据集内部结构(schema)更清晰有利于优化，且在编译时检查类型
     * 跟RDD类似的算子(transform/action)，还提供了其它算子如select等
     * 使用SparkSQL的执行引擎，可以充分利用SparkSQL的优化策略
     * 支持从parquet/json/csv/jdbc等创建DataSet，也可以从其它DataSet转换过来
     
     
    ```
        val ds1 = spark.read.text("README.md").as[String]
        val ds2 = ds1.flaMap(line => line.split(" "))
        val ds3 = ds2.groupByKey(_)
        val count = ds3.count()
        
    ```
    
      
  - DataFrame
    
     * DataFrame = DataSet[Row]，它是一种特殊的DataSet
    
  Spark提供了RDD和DataSet/DataFrame的转换接口，可以将RDD自动转换为DataSet/DataFrame
  
  ```
     import spark.implicits._
     val ds = spark.sparkContext.makeRDD(Seq("a","b","c")).toDS
     val df = spark.sparkContext.makeRDD(Seq("a","b","c")).toDF("col")
     
  ``` 
  
  Spark也提供了Seq和DataSet/DataFrame的转换，如
  
  ```
     import spark.implicits._
     val ds = Seq("a","b","c").toDS
     val df = Seq("a","b","c").toDF("col")
     
     val ds1 = Seq(1,2,3).toDS
     val df1 = Seq(1,2,3).toDF("col")
     
  ```
  
  `SparkSQL会将scala对象进行encode成SparkSQL内部的InternalRow,InternalRow的存储结构耗费的内存更少，目前支持primitive type(如long/int/float/double等)，Product，Seq，Array等类型的encoder，Set等类型不支持会报错。`
   
      
