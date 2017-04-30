## 运行流程

当你在MapReduce程序中调用了Job实例的Submit()或者waitForCompletion()方法，该程序将会被提交到Yarn中运行   
其中的过程大部分被Hadoop隐藏起来了，对开发者来说是透明的   
程序运行的过程涉及到个概念：

> 1.Client：提交程序的客户端   
> 2.ResourceManager：集群中的资源分配管理   
> 3.NodeManager：启动和监管各自节点上的计算资源   
> 4.ApplicationMaster：每个程序对应一个AM，负责程序的任务调度，本身也是运行在NM的Container中   
> 5.HDFS：分布式文件系统，和以上各个实体进行作业数据的共享

MapReduce作业在Yarn中的运行流程如图所示：

![MapReduce运行流程图](/images/mapreduce-works.png)

这张图和[Hadoop核心组件之Yarn](http://www.xiaohei.info/2016/05/02/hadoop-yarn-summary/)中提到的流程图很相似   
因为MapReduce作业也是属于Yarn管理的一部分，只是这张图针对MapReduce的运行更加细化了

### 作业提交

> 1.客户端调用Job实例的Submit()或者waitForCompletion()方法提交作业   
> 2.客户端向RM请求分配一个Application ID

进行步骤2的时候，**客户端会对程序的输出路径进行检查，如果没有问题，进行作业输入分片的计算**

> 3.将作业运行所需要的资源拷贝到HDFS中，包括jar包、配置文件和计算出来的输入分片信息等   
> 4.调用RM的submitApplication方法将作业提交到RM

### 作业初始化

> 5a.RM收到submitApplication方法的调用之后会命令一个NM启动一个Container   
> 5b.在该NM的Container上启动管理该作业的ApplicationMaster进程   
> 6.AM对作业进行初始化操作，并将会接收作业的处理和完成情况报告   
> 7.AM从HDFS中获得输入数据的分片信息  

在步骤7中，AM将会根据分片信息确定要启动的map任务数，reduce任务数则根据mapreduce.job.reduces属性或者Job实例的setNumReduceTasks方法来决定

### 任务分配

> 8.AM为其每个map和reduce任务向RM请求计算资源

步骤8中，map任务的资源分配是要优先于reduce任务的，因为在reduce的排序阶段开始之前，map任务必须全部完成   
因此，reduce任务的资源请求只有当map任务完成了至少5%的时候才会进行

reduce任务是可以在集群上的任意一个节点运行的，所以进行计算资源分配的时候RM不需要为reduce任务考虑分配哪个节点的资源给它   
但是map任务不一样，**map任务有一个数据本地化的优化特性**   

数据本地优化是指map任务中处理的数据存储在各个运行map本身的节点上，这能够使得作业以最好的状态运行，因为不需要跨界点消耗网络带宽进行数据传输   
移动计算而不移动数据   
Yarn会优先给map任务分配本地数据，如果不存在，则在同一机架内的不同节点上搜寻数据，最差的情况是跨机架之间的数据传输   
map每个split大小默认和hdfs的block块大小一致的原因就是：   
太大，会导致map读取的数据可能跨越不同的节点，没有了数据本地化的优势   
太小，会导致map数量过多，任务启动和切换开销太大，并行度过高

RM在map任务要处理的数据块的那个节点上为map分配计算资源，如此一来，map任务就不需要跨网络进行数据传输了   
因为AM中有输入数据的分片信息和要启动的map任务的信息，所以在为map任务请求资源的时候，RM会根据这些信息为map分配计算资源

这里的计算资源指的是map/reduce任务运行所需要的内存和CPU等，默认每个任务分配1024M的内存和一个CPU虚拟核心   
可以通过修改以下选项来修改这个配置：

> * mapreduce.map.memory.mb   
> * mapreduce.reduce.memory.mb   
> * mapreduce.map.cpu.vcores   
> * mapreduce.reduce.cpu.vcores 

### 任务执行

> 9a.AM在RM指定的NM上启动Container   
> 9b.在Container上启动任务（通过YarnChild进行来运行）   
> 10.在真正执行任务之前，从HDFS从将任务运行需要的资源拷贝到本地，包括jar包、配置文件信息和分布式缓存文件等   
> 11.执行map/reduce任务

### 作业完成

作业执行过程中，我们可以通过Yarn Web UI界面的AM页面中查看作业的运行信息    
当在客户端调用waitForCompletion方法，每隔5秒钟客户端会检查一次作业的运行情况

作业执行完毕之后将调用OutputCommitter方法对作业的信息进行最后的清理工作

## 失败处理

在实际场景中，用户的代码总是会有bug、程序异常和节点失效等问题，Hadoop提供了失败处理的机制尽可能的保证用户的作业可以被顺利完成   
其中的过程需要考虑以下几个实体：

### Task的失败

#### 任务失败标记

当由于用户的代码导致map/reduce任务产生运行时异常时   
在该任务退出之前，JVM会发送报告给AM，并且将错误信息写入用户日志中   
AM将该任务标记为失败，释放Container资源

除了用户代码之外，还有很多其他原因会导致任务失败，如JVM的bug等   
AM会间隔性的接收来自各个任务的汇报，当一段时间过后AM没有接收到某个任务的报告   
AM将会判断**该任务超时**，将该任务标记为失败并让节点上的任务退出

任务超时的时间可以在作业中通过**mapreduce.task.timeout**选项来为每个作业单独配置   
设置为0表示无任务超时时间，此时任务运行再久也不会被标记为失败，其资源也无法释放，会导致集群效率降低

#### 失败任务的重启

当AM注意到一个任务失败了之后，将会尝试重新调度该任务   
任务的重试不会在之前失败了的节点上运行，并且失败四次之后AM将不会继续重启该任务   
这个值同样是可以配置的：

> * mapreduce.map.maxattempts   
> * mapreduce.reduce.maxattempts

默认的，一旦作业中有任何一个任务失败超过4次，**那么整个作业将会标记为失败**   
但是很多情况下，即使作业中的某些任务失败了，其他任务的执行结果还是有价值的   
所以我们可以配置一个作业中允许任务失败的最大比例：

> * mapreduce.map.failures.maxpercent   
> * mapreduce.reduce.failures.maxpercent

### ApplicationMater的失败

和任务失败一样，AM也可能由于各种原因（如网络问题或者硬件故障）失效，Yarn同样会尝试重启AM   
可以为每个作业单独配置AM的尝试重启次数：**mapreduce.am.max-attempts**，默认值为2   
需要注意的是，Yarn中限制了每个AM重启的最大限制，默认也为2，如果为单个作业设置重启次数为3，超过了这个上限也不会起到作用   
所以还需要注意将Yarn中的上限一起提高：**yarn.resourcemanager.am.nax-attempts**

由于AM会通过心跳机制向RM信息，当RM注意到AM失败了之后，会在另外一个节点的Container上重启，并将恢复已经执行的任务进度（心跳机制保留）   
使得重启的AM不用重头执行任务，任务进度恢复默认是开启的，可以通过**yarn.app.mapreduce.am.job.recovery.enable**为false来禁用

客户端通过AM来获得作业的执行情况，当AM失效的时候，客户端会重新向RM请求新的AM地址来更新信息

### NodeManager的失败

NM也通过心跳机制向RM汇报情况，当一个NM失效，或者运行缓慢的时候，RM将收不到该NM的心跳，或者心跳时间超时   
此时RM会认为该NM失败并**移出可用NM管理池**，心跳超时的时间通过**yarn.resourcemanager.nm.liveness-monitor.expiry-interval-ms**来配置   
默认为10分钟

在失败的NM上运行的AM或者任务都会按照之前讨论的机制进行恢复   

Yarn对于NM的管理还有一个类似黑名单的功能，当该NM上的任务失败次数超过3次之后（默认），该NM会被拉入黑名单   
此时，即使该NM没有失效，AM也不会在该NM上运行任务了   
NM上的最大任务失败次数可以通过**mapreduce.job.maxtaskfailures.per.tracker**来配置

### ResourceManager的失败

RM是Yarn中最终要的一个角色，没有了RM集群将无法使用   
RM类似于HDFS中的Namenode，需要一种高可用的机制来保障   
RM一开始就使用一种检查点的机制来将集群信息持久化到磁盘中   
当RM失效了之后，管理员可以手动启动一个备用的RM读取持久化的信息

## Shuffle和Sort

Shuffle是MapReduce中的核心概念，也是代码优化中很重要的一部分   
理解Shuffle过程可以帮助你写出更高级的MapReduce程序

![Map-Reduce运行原理图](/images/mr-shuffle.jpg)

### Map Side

如上图所示，map函数在产生数据的时候并不是直接写入磁盘的，而是先写入一个内存中的环形缓冲区   
理由是在内存中对数据进行分区、分组排序等操作对比在磁盘上快很多

每个map都有一个缓冲区，默认大小为100M
当缓冲区的数据达到一个阈值的时候将会产生spill操作写入到磁盘中，阈值默认为80%

缓冲区中的数据spill的时候，map产生的数据会源源不断的写入到缓冲区中空出来的空间   
当缓冲区占满的时候，map任务将会阻塞直到缓冲区可以写入数据

在map的数据写入磁盘之前，在内存的缓冲区中会根据程序中设置的分区数对数据进行分区   
并在每个分区内对数据进行分组和排序   
如果设置了combiner函数，其将会在排序之后的数据上运行，以减少写入到磁盘的数据量

缓冲区的每次spill操作都会在磁盘上产生一个spill文件，所以一个map可能产生多个spill文件   
任务完成之前，这些spill文件会被合并为一个已分区且已排序的输出文件

如果至少存在三个spill文件（默认）且设置了combiner函数，那么在合并spill文件再次写入到磁盘的时候会再次调用combiner

此时Map端的任务完成

### Reduce Side

Map端的任务完成之后，Reduce端将会启动会多个线程通过HTTP的方式到Map端获取数据   
为了优化reduce的执行时间，hadoop中等第一个map结束后，所有的reduce就开始尝试从完成的map中下载该reduce对应的partition部分数据

在这个shuffle过程中，由于map的数量通常是很多个的，而每个map中又都有可能包含每个reduce所需要的数据
所以对于每个reduce来说，去各个map中拿数据也是并行的

Reduce端会启动一个线程周期性的向AM请求Map端产生数据所在的位置（map任务和AM之间有心跳机制）   
Map端产生的数据并不会在Reduce端获取之后马上删除，因为reduce任务可能会因为失败而重启

Reduce端将Map端的数据拷贝过来之后也会放入一个内存缓冲区中，数据达到缓冲区的指定阈值之后h合并写入到磁盘   

随着磁盘数据文件的增多，和Map端一样，Reduce端也会对溢出文件进行合并   
**mapreduce.task.io.sort.factor**可以控制Map和Reduce端的文件数量达到多少个时进行合并   
和Map端的合并不同，假设上述选项采用默认值10，共有40个溢出文件   
Map端最终会合并形成4个文件   
而Reduce端第一次只会合并4个文件，随后三次各合并10个文件，还剩余6个文件   
此时Reduce端中的文件还有10个，最后一次合并为1个文件输入到reduce函数中   
由此可以看出，Reduce端的合并**目标是合并最小的文件数量以满足最后一次合并刚好达到设置的文件合并系数**   
其目的是为了reduce读取减少磁盘的开销

如果指定了combiner函数则会在合并期间运行

随后进入reduce函数的执行阶段，并产生数据输出到HDFS   
由于大部分情况下，运行NM的节点往往还运行着Datanode，所以输出数据的第一个副本通常是保存在本地

## 性能调优

参考：[MapReduce性能调优记录](http://www.xiaohei.info/2016/03/15/mapreduce-tunning/)

作者：[@小黑](http://www.xiaohei.info)
