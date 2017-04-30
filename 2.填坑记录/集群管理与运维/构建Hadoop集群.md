Hadoop的安装包可以从以下渠道获取：

> * Apache tarballs：Hadoop官网提供的tar包，包括二进制和源码文件，使用这种方式部署Hadoop集群灵活性比较高，但是要自己进行很多额外的操作   
> * Packages：Hadoop也提供RPM和Debian包，先对比tar包，rpm可以简化部署时候的配置路径等繁琐的操作，并且和Hadoop生态圈中的各个组件版本都兼容对应   
> * Hadoop cluster management tools：例如Cloudera Manager和Apache Ambari，这些工具将部署的整个证明周期封装起来，通过可视化的界面简单操作和配置

## 集群配置规范

Hadoop被设计为运行在大量廉价的商用机器上，各个公司所使用的机器配置各不相同   
举例说明，2014年运行Datanode和NodeManager的机器配置如下：

> * 两个8或者16核，3GHz的CPU   
> * 64-512GB的ECC内存   
> * 12-24块1-4TB的SATA硬盘

## 集群规模

到底要部署多大的集群才适合我的场景呢？   
这个问题其实可以更好的换成：**集群数据的增长量是多少呢？**   
因为Hadoop可以动态的添加和删除节点，所以当当前的集群资源不够时，只要将新节点上线即可

当集群规模很小的时候，将Namenode、SecondaryNamenode和ResourceManager等进程运行在一个节点是可以接受的   
但是当集群内的文件或者数据量变得很大的时候，这么做显然不行了   
因为这三个进程都十分耗内存（即使SecondaryNamenode平时是闲置的，但是当checkpoint工作的时候也会非常吃内存）   
所以这个时候应该将这三个进程分开在独立的机器上

## 网络拓扑

Hadoop的网络拓扑有两级：节点与节点之间，机架与机架之间   
当集群的机器处理同一个机架下的时候用户无需任务配置，因为这是默认的   
但是当集群的机器分散在不同的机架中时，Hadoop需要知道哪些机器在哪个机架中   
**由此才能智能的判断节点间的距离，存放副本和分配任务**

用户可以自定义一个脚本文件，并通过net.topology.script.file.name属性设置该脚本的路径   
该脚本的内容就是定义机器和机架之间的映射关系   
一个示例脚本可以在[这里](http://wiki.apache.org/hadoop/topology_rack_awareness_scripts)查看

## 配置文件管理

搭建过Hadoop集群的人都知道，每个节点上都有自己的一份配置文件    
如果进行集群间的配置文件同步是一个很头疼的问题，尽管我们可以通过rsync这类工具来实现

Hadoop的环境变量设置在**hadoop-env.sh**文件中，另外还单独提供了**mapred-env.sh**和**yarn-env.sh**文件   
前者是全局的，后两者是专门为MapReduce和Yarn来配置的，后两者会覆盖前者的设置

### Java环境设置

JAV_HOME可以设置在用户的shell环境中，但是最好在hadoop-env.sh文件中也设置以确保集群的Java环境是相同的

### 进程内存设置

默认情况下，**Hadoop会为每个进程分配1GB的内存**    
对应hadoop-env.sh文件中的**HADOOP_HEAPSIZE**选项   
用户也可以单独为某个进程指定内存，例如ResourceManager的设置在yarn-env.sh中的**YARN_RESOURCEMANAGER_HEAPSIZE**

### Namenode的内存分配

对于默认的1GB内存，Namenode很可能会全部吃光   
因为该进程的内存需求量是取决于**HDFS中的目录数、块数量和文件名长度等因素**   
根据经验来看，1GB的内存足够Namenode来管理100万个Block 

以一个200节点的集群为例，每个节点的硬盘位24T，块大小为128M，副本数量为3   
那么该集群最多能够存放的Block数量为：   
**200 * 24 * 1000 * 1000 / 128 / 3 = 12500000**   
1250万个Block，根据之前1G管理100万个Block   
**那么此时Namenode的内存最好一开始就设置为12-13G**

用户可以在hadoop-env.sh中通过**HADOOP_NAMENODE_OPTS**选项来控制Namenode的内存大小   
例如-Xmx12000m将会按照之前讨论的内存分配给Namenode

当你改变了Namenode分配的内存时，不要忘记SecondaryNamenode的设置：   
**HADOOP_SECONDARYNAMENODE_OPTS**   
因为SecondaryNamenode的内存使用量和Namenode基本是持平的

### 作业内存分配相关

由于container在工作节点启动，分配container的内存时需要考虑到该节点上有多少进程在运行

如果Datanode和NodeManager同时运行在工作节点上，默认情况下他们将会占据2G的内存    
再加上该机器上其他任务的进程，**剩下的内存才能专门分配给container**

确定了节点上有多少内存可以给container使用之后，就要确定如何为每个单独的作业分配内存   
控制该因素主要有两点：

> 1.Yarn分配的container内存大小   
> 2.container中运行的Java程序的内存

第一个通过**mapreduce.map.memory.mb**和**mapreduce.reduce.memory.mb**来控制，默认为1024M   
第二个通过**mapred.child.java.opts**来控制，默认为-Xmx200m，也可以使用**mapred.map.java.opts**和**mapred.reduce.java.opts**控制更细粒度的map和reduce任务内存的分配

需要注意的是，map和reduce任务的内存不可以超出container分配的内存   
这些参数**通常在客户端通过Configuration来设置**，因为每个作业的需求量都不一样

具体如何确定作业的内存，可以借助计数器功能   
**PHYSICAL_MEMORY_BYTES,VIRTUAL_MEMORY_BYTES和COMMITTED_HEAP_BYTES**这三个指标可以帮助用户决定分配作业的内存

此处参见《Hadoop权威指南》第十章第三节中的关键属性配置

### CPU配置

在Yarn中，CPU也被作为一种可以分配的资源来呗container请求   
一个节点上最多可用的CPU核心数通过**yarn.nodemanager.resource.cpu-vcores**控制   
每个作业也可以在客户端通过   
**mapreduce.map.cpu.vcores**    
**mapreduce.reduce.cpu.vcores**   
两个属性来单独设置，设置的规则和之前讨论的内存分配相似

### 系统日志文件

Hadoop的日志文件默认保存在$HADOOP_HOME/logs目录下   
用户可以通过hadoop-env.sh中的**HADOOP_LOG_DIR**选项来修改   
最好将日志目录设置在Hadoop的安装目录之外   
这样一来即使Hadoop升级时候**目录结构发生变化也不会影响到日志文件**

各个Hadoop进程会产生两种日志：

> 1.通过log4j产生的.log后缀文件：该文件记录了大部分的程序信息，是**排错的首选**   
> 2.记录标准输出和错误的.out文件：由于使用log4j输出日志，这类文件通常内容很少甚至为空

第一类日志系统不会自动删除，需要**用户手动归档或者删除处理**，以免磁盘空间不够用   
第二类日志系统只会保留最新的5个文件，1-5依次表示从新到老文件

日志文件的命名格式采用用户名-进程名-机器名的形式   
其中可以通过hadoop-env.sh中的**HADOOP_IDENT_STRING**选项来修改

### 其他属性配置

#### 集群成员管理

为了方便对集群中的节点进行上下线等操作，Hadoop在配置文件中也提供了对应的属性：

> * dfs.hosts：允许作为Datanode节点加入集群的主机列表   
> * dfs.hosts.exclude：不允许作为Datanode节点加入集群的主机列表
> * yarn.resourcemanager.nodes.include-path：允许作为NodeManager节点加入集群的主机列表   
> * yarn.resourcemanager.nodes.exclude-path：不允许作为NodeManager节点加入集群的主机列表

以上选项的值可以是一个文件路径，该文件中每行一个主机名，具体的操作可以参考：   
[Hadoop 添加删除Slave][http://www.xiaohei.info/2016/01/14/hadoop-delete-slave/]

#### 文件操作的缓冲区大小

Hadoop使用一个4k的缓冲区来辅助I/O操作，对于现代的硬件和操作系统来说，这个值太过保守   
适合当的提高可以优化性能，通过core-site.xml文件中的**io.file.buffer.size**来设置（通常选择为128k）

#### HDFS的块大小

HDFS的Block大小默认为128MB，但是为了降低Namenode的内存压力，并**提高mapper任务数据输入量**   
很多集群将其设置为256MB，根据用户自己的集群配置来设置

#### 磁盘空间保留

默认情况下，Datanode将会使用其安装目录下的所有磁盘空间   
如果希望保留一些空间给其他非HDFS文件存放可以通过**dfs.datanode.du.reservrd**来设定（以字节为单位）

#### 回收站设置

Hadoop也有回收站的设置，通过core-site.xml中的**fs.trash.interval**选项来控制回收站中的文件保留多久   
默认为0表示不启用回收站机制   

启用了回收站，通过shell删除的文件都会先移动到回收站的文件夹中   
但是如果通过程序，**文件将直接删除**，除非使用了Trash类的moveToTrash方法

## 集群构建和安装

详细步骤参考：   
[大数据平台生产环境部署指南](http://www.xiaohei.info/2016/03/14/hadoop-dataplatform-deploy-guide/)

### 格式化文件系统

使用一个新的Hadoop集群之前，需要将HDFS文件系统格式化，命令为：

```shell
hdfs namenode -format
```

格式化程序将会创建存储目录和初始的Namenode持久化数据的结构版本

### 启动和停止进程

启动HDFS的命令为：

```shell
start-dfs.sh
```

该脚本将会进行一下操作：

> * 在hdfs getconf -namenodes命令得到的ip机器上启动Namenode进程   
> * 在slaves文件中记录的所有机器上启动Datanode进程   
> * 在hdfs getconf -secondarynamenodes命令得到的ip机器上启动SecondaryNamenode进程

其中hdfs getconf -namenodes命令获得的机器ip是根据配置文件中的**fs.defaultFS**选项来获得的   

启动Yarn的命令为：

```shell
start-yarn.sh
```

和启动HDFS类似，该脚本会进行一下操作：

> * 在本地启动ResourceManager进程   
> * 在slaves文件中记录的所有机器上启动NodeManager进程   