## 程序在Yarn上的运行流程

![Yarn运行流程图](/images/hadoop-yarn.png)

如图所示，Yarn上的应用程序运行会经过如下步骤：

> 1.客户端提交应用程序   
> 2.RM找到一个NM启动第一个container来运行AM   
> 3.AM会向RM请求资源的分配并通过心跳机制来汇报作业的运行状态   
> 4.AM在分配的NM上启动container来运行作业

## Yarn和MRv.1的对比

MRv.1中由两个进程控制作业的执行：**JobTracker和TaskTracker**   
JobTracker协调系统中运行的所有作业，并分配任务给TaskTracker   
TaskTracker负责运行任务并发送执行报告给JobTracker   
如果任务执行失败，JobTracker将会在另外一个TaskTracker上重新分配该任务

在MRv.1中，JobTracker管理着系统的资源调度、任务分配   
在Yarn中，JobTracker被了另外两个角色所代替：ResourceManager和ApplicationMaster（每个作业单独一个）

下表列出了MRv.1和Yarn中的各个角色的对比

|MRv.1|Yarn|
|---|---|
|JobTracker|ResourceManager,ApplicationMaster,TimelineServer|
|TaskTracker|NodeManager|
|slot|container|

Yarn的设计突破了MRv.1中的众多限制，例如：

### 可扩展性

因为JobTracker集资源调度和任务分配两大核心功能于一身，是个相当重量级的组件   
当集群规模变大的时候，JobTracker所在的节点负荷会变得很大，这也就限制了其集群规模只能为4000个节点或者调度40000个任务

而Yarn将JobTracker划分为系统级的ResourceManager和应用级的ApplicationMaster，角色细分，可以管理超过10000个节点和100000个任务的超大集群   

### 高可用性

JobTracker存在单点故障问题，因为JobTracker本身功能的复杂性，所以很难做到HA

反之Yarn则不同，ResourceManager和ApplicationMaster都做到了高可用的机制   

### 资源利用率

MRv.1中的资源是以slot形式来描述的，每个slot是一个固定大小的资源槽（在配置的时候就固定下来了）   
并且还划分为map slot和reduce slot，分别只能运行map和reduce task   
无法实现资源共享，并且每个task并不一定能够完全使用一个slot的资源，这就造成了资源浪费

Yarn中将资源描述为container，map和reduce task都使用同样概念的container，并且按需分配

### 多用户

从某个角度看，使用Yarn的最大好处就是可以将MapReduce和其他并行计算框架完美的结合起来   
Yarn将资源调度和任务分配拆分开来，使得例如Spark、Storm等实时计算框架可以和MapReduce运行在同一集群上并进行统一的资源调度   
而MRv.1则是专门为离线计算的MapReduce框架而生的

## Container模型

当前Yarn支持内存和cpu两种资源类型的管理和分配，yarn采用了动态资源分配机制   
NodeManager启动时会向ResourceManager注册，注册信息中包含该节点可分配的cpu和内存总量

> * yarn.nodemanager.resource.memory-mb:可分配的物理内存总量，默认是8mb*1024，即8G   
> * yarn.nodemanager.vmem-pmem-ratio:任务使用单位物理内存量对应最多可使用的虚拟内存量，默认值是2.1，表示每1mb的物理内存，最多可以使用2.1mb虚拟内存总量   
> * yarn.nodemanager.resource.cpu-vcores:可分配的虚拟cpu个数，默认是8，yarn允许管理员根据实际需要和cpu性能将每个物理cpu划分成若干个虚拟cpu

## Yarn中的调度器

Yarn支持三种调度器：FIFO,Capacity和Fair调度器   
默认为FIFO

### FIFO Scheduler

顾名思义，该调度器将作业按照提交的顺序排成一个队列，先进先出，先提交的先运行   
这就带来一个问题：如果先运行了一个非常大的作业，该作业会占用集群的所有资源，而一些只需要一小部分资源就能完成的小作业将会等到大作业完成之后才能进行

所以该调度器不适合用于一个共享的、多用户的集群环境

### Capacity Scheduler

其为FIFO的升级版，该调度器会将集群中的一小部分资源划分出来单独作为一个队列来运行小作业   
虽然在处理大作业的同时也能够处理小作业，但是由于大作业的资源被占用了一部分（FIFO中其是独占整个集群资源的），所以大作业的执行时间会比FIFO的更久   
大小作业队列内部采用FIFO的方式进行调度

**配置方式**

下面是一个简单的Capacity Scheduler的配置文件（在classpath中的capacity-scheduler.xml）

```xml
<?xml version="1.0"?>
<configuration>
	<!--将root划分为两个列队-->
	<property>
		<name>yarn.scheduler.capacity.root.queues</name>
		<value>prod,dev</value>
	</property>
	<!--dev再次划分为两个列队-->
	<property>
		<name>yarn.scheduler.capacity.root.dev.queues</name>
		<value>eng,science</value>
	</property>
	<!--root中两个队列的容量分别为40%和60%-->
	<property>
		<name>yarn.scheduler.capacity.root.prod.capacity</name>
		<value>40</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.capacity</name>
		<value>60</value>
	</property>
	<!--dev队列所能用的最大集群资源量-->
	<property>
		<name>yarn.scheduler.capacity.root.dev.maxinum-capacity</name>
		<value>75</value>
	</property>
	<!--dev中两个队列的容量都为50%-->
	<property>
		<name>yarn.scheduler.capacity.root.dev.eng.capacity</name>
		<value>50</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.science.capacity</name>
		<value>50</value>
	</property>
</configuration>
```

该配置文件会产生如下所示的调度队列：

root
~ prod
~ dev
~~~ eng
~~~ science

### Fair Scheduler

公平调度器会根据运行的作业来动态分配资源   
当集群中只有一个大作业在运行的时候，其会占用集群中的所有资源

当第二个作业（小作业）提交之后，将会划分集群中一般的资源来运行这个作业，这样集群中的各个作业得到的资源都是相同的   
当小作业运行完毕之后，大作业会继续占用那一半的资源，所以称为公平调度器

这是在单用户多任务的情况下的资源分配，当出现多用户多任务的时候   
**每个用户都将得到集群中相同比例的资源数**，每个用户中的每个作业将会得到相同比例的该用户**所得资源数**

**配置方式**

在yarn-site.xml中需要配置   
yarn.resourcemanager.scheduler.class   
为   
org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler   
（Hadoop中该选项默认为Capacity，而在CDH中默认为Fair）

下面是一个简单的Fair Scheduler的配置文件   
（在classpath中的fair-scheduler.xml，可以通过yarn-site.xml中的yarn.scheduler.fair.allocation.file配置项修改）

```xml
<?xml version="1.0"?>
<allocations>
	<!--设置各个队列中默认的调度器为fair-->
	<defaultQueueSchedulingPolicy>fair</defaultQueueSchedulingPolicy>
	<queue name="prod">
		<weight>40</weight>
		<!--特殊指定该队列的调度器为fifo-->
		<schedulingPolicy>fifo</schedulingPolicy>
	</queue>
	<queue name="dev">
		<weight>60</weight>
		<!--没有指定weight默认平分-->
		<<queue name="eng" />
		<<queue name="sicence" />
	</queue>
	<queuePlacementPolicy>
		<!--队列中没有specefied则进入下个rule-->
		<rule name="specefied" create="false" />
		<!--队列中没有用户组则进入下个rule-->
		<rule name="primaryGroup" create="false" />
		<!--默认进入dev.eng-->
		<rule name="default" create="dev.eng" />
	</queuePlacementPolicy>
</allocations>
```

上面的配置文件将会产生和之前Capacity配置的一样的队列

作者：[@小黑](http://www.xiaohei.info)
