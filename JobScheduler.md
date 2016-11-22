SparkCore
  
   当RDD/DataSet/DataSet遇到Action的算子就会触发Job提交，提交后由DAGScheduler和TaskScheduler进行调度。这两种调度针对的维度不同，DAGScheduler是Stage维度，TaskScheduler是TaskSet维度。如下图所示：
   
   ![scheduler](file:///Users/songjun/Documents/工作/EMR/spark/pic/scheduler.jpeg)
   
1. DAGScheduler
   
     * 负责将Job拆解成Stage(ShuffleMapStage/ResultStage)  
       
       RDD的action算子触发submitJob操作，action算子作用的当前RDD记为finalRDD，Spark会对finalRDD的dependency进行递归遍历，遇到ShuffleDependency宽依赖则生成一个新的Stage
       
     * 负责调度Stage  
       
       一个Job生成多个Stage，DAGScheduler会先执行父Stage，父Stage执行完了才会执行子Stage，直到所有Stage执行完，作业结束
              
     * 负责获取Stage中Task的位置信息 
     
       Task在提交给TaskScheduler之前会获取task的位置信息，即task对应的partition的位置信息
              
     * 负责根据partition生成Task(ShuffleMapTask/ResultTask)并提交给TaskScheduler
        
     
     * 负责跟踪Cache，防止RDD重复计算
     * 负责失败重试，如shuffle中间文件丢失会重新提交stage
        
      当一个executor挂了，会重新调度失败的Stage
    
    DAGScheduler类图关系如下：
    ![class_dagscheduler](file:///Users/songjun/Documents/工作/EMR/spark/pic/dagscheduler_class.jpeg)
    
    其中:
    - TaskScheduler
      
      DAGScheduler调度的Stage生成TaskSet，提交到TaskScheduler进行Task的调度。
    
    - BlockManagerMaster
      
      存储管理，如获取task的location
       
    - MapOutputTrackerMaster
       
       记录ShuffleMapTask的输出数据位置，在ResultTask中从MapOutputTrackerMaster获取位置信息，然后通过BlockManager获取真正的数据
    
    - LiveListenerBus
      
      异步处理事件
      
   
    
2. TaskScheduler

   TaskScheduler会对DAGScheduler提交的taskset进程调度，调度的算法有两种(FAIR/FIFO)，默认是FIFO.
   * 负责将Stage提交的TaskSet调度到集群的executor上面执行
   * 负责task失败重试
   * 负责将event汇报给DAGScheduler
   
   2.1 调度TaskSet
   
      调度会涉及两方，一个是执行的资源(Executor->offer)，一个是被执行的资源(Task->resource),即如何将Task分配到Executor上执行的过程，Executor的资源(cpu)由`CoarseGrainedSchedulerBackend`管理，Task的选择由SchedulableBuilder来进行管理，这两个类都被组合进TaskScheduler中。
   
      * 如何触发调度
      
         有两个地方可以触发调度，
         
         - driver端触发
           
           `CoarseGrainedSchedulerBackend`启动的时候会创建一个定时器，每隔1s给自己Driver发消息，触发一次调度,Driver会调用makeOffers(executorId)方法，makeOffers会调用TaskSchedulerImpl的resourceOffers方法(按照调度策略获取taskset
           
         - executor端触发
         
           `CoarseGrainedExecutorBackend` 执行完task或者状态的变化会发送StatusUpdate给Driver，Driver会执行上面同样的调度逻辑。
         
      
      * 如何调度
      
         - FIFO
         	
         	DAGScheduler将一个Job拆解成多个Stage，同一个Job内有血缘关系的Stage由DAGScheduler来进行调度，而不存在血缘关系的多个Stage会同时存在TaskScheduler中，TaskScheduler会优先执行StageId小的TaskSet；
         	
         	DAGScheduler提交的多个Job，生成的TaskSet也会同时存在于TaskScheduler中，TaskScheduler会优先执行JobId小的TaskSet
         
            总体来说，TaskScheduler会按照(JobId,StageId)来进行调度TaskSet集，JobId小(先提交的作业)的优先调度，同一个JobId内优先调度StageId小的。
         
         - FAIR
   
   2.2 预测执行
      
     TaskSetManager会对其中每个Task进行监控，如      果满足一定的条件则会执行影子任务。
      有两个参数来控制是否需要对Task启动影子任务：
      
      - spark.speculation.quantile
          
          运行成功的task的个数是否超过TaskSet中总task个数的比例，如设置为0.75，则表示TaskSet中超过75%的task执行成功了，它是触发影子任务的前提条件
        
      - spark.speculation.multiplier
          
          当前运行成功的Task的运行时间中位数的倍数，如果运行时间超过这个时间的task需要进行启动影子任务，如设置成1.5倍。
   
   2.3 失败重试	
      
      - 如何判断Task失败
        
        超过120s没心跳
        
      


