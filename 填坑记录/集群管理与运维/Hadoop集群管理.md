## HDFS进程目录结构

对于一个集群管理员来说，理解HDFS各个进程存储在磁盘上的数据含义是十分有用的，可以帮助你诊断和排查一些集群问题

### Namenode的目录结构

HDFS进行初次格式化之后将会在$dfs.namenode.name.dir/current目录下生成一些列文件：   

```
${dfs.namenode.name.dir}/
├── current  
│ ├── VERSION  
│ ├── edits_0000000000000000001-0000000000000000007  
│ ├── edits_0000000000000000008-0000000000000000015  
│ ├── edits_0000000000000000016-0000000000000000022  
│ ├── edits_0000000000000000023-0000000000000000029  
│ ├── edits_0000000000000000030-0000000000000000030  
│ ├── edits_0000000000000000031-0000000000000000031  
│ ├── edits_inprogress_0000000000000000032  
│ ├── fsimage_0000000000000000030  
│ ├── fsimage_0000000000000000030.md5  
│ ├── fsimage_0000000000000000031  
│ ├── fsimage_0000000000000000031.md5  
│ └── seen_txid  
└── in_use.lock
```

VERSION文件的内容是一些HDFS的ID信息，比较重要的是**fsimage、edits和seen_txid**   

fsimage中存储的是HDFS的**元数据信息**，包括文件系统上的文件目录和相关序列化的信息。   
edits文件保存的是HDFS上的更新操作信息。   
seen_txid文件保存的是一个数字，就是最后一个edits_的数字。

每次Namenode启动的时候都会将fsimage文件读入内存，并从00001开始到seen_txid中记录的数字依次执行每个edits里面的更新操作，
保证内存中的元数据信息是最新的、同步的，可以看成Namenode启动的时候就将fsimage和edits文件进行了合并。   

集群启动的之后，每次对HDFS的更新操作都会写入edits文件。   
对于客户端来说，**每次写操作都会先记录到edits文件中**，之后更新Namenode内存中的元数据信息。

对于一个长期运行中的集群来说，edits文件会一直增大，**SecondaryNamenode会在运行的时候定期将edits文件合并到fsimage中**   
否则一旦集群重新启动，Namenode合并fsimage和edits使用的时间将会非常久

SecondaryNamenode会运行一个checkpoint进程来进行合并操作：

> 1.SecondaryNamenode告知Namenode停止当前edits文件的读写，并将操作写入一个新的edits文件，同时更新seen_txid文件   
> 2.通过HTTP Get请求从Namenode中获得fsimage和edits文件   
> 3.加载fsimage文件到内存（和Namenode内存一致的原因），并执行edits文件中的每个操作，产生一个新的fsimage   
> 4.通过HTTP Put将新的fsimage文件发送到Namenode中，保存为一个.ckpt文件   
> 5.Namenode重命名.ckpt文件使其生效

默认情况下，SecondaryNamenode每隔一个小时就会进行以上的操作，通过**dfs.namenode.checkpoint.period**以秒为单位来配置   
另外，如果未到达配置的时间而edits文件大小达到**dfs.namenode.checkpoint.txns**配置的值时也会进行合并操作   
每隔**dfs.namenode.checkpoint.check.period**指定的时间检查一个edits文件的大小

### SecondaryNamenode的目录结构

SecondaryNamenode的current目录和Namenode是类似的，但是其还保留着一个previous.checkpoint目录   
数据存储的目录由**dfs.namenode.checkpoint.dir**指定
顾名思义，就是上一个检查点的目录，其结构是和current目录一致的   
如此一来当Namenode挂了之后可以从SecondaryNamenode上启动一个新的Namenode（即使可能不是最新的元数据）   
有两种方式可以实现：

> 1.将相关的存储目录复制到新Namenode指定目录中   
> 2.在SecondaryNamenode上使用-importCheckpoint选项来启动Namenode进程

第二种方式会将指定目录下的元数据加载到内存中从而成为一个新的Namenode   
该目录由**dfs.namenode.checkpoint.dir**指定（前提是**dfs.namenode.name.dir**目录下找不到元数据信息）

### Datanode的目录结构

和Namenode不一样的是，Datanode的目录并不需要进行格式化来创建，如下：

```
${dfs.datanode.data.dir}/
├── current  
│ ├── BP-1079595417-192.168.2.45-1412613236271  
│ │ ├── current  
│ │ │ ├── VERSION  
│ │ │ ├── finalized  
│ │ │ │ └── subdir0  
│ │ │ │ └── subdir1  
│ │ │ │ ├── blk_1073741825  
│ │ │ │ └── blk_1073741825_1001.meta  
│ │ │ │── lazyPersist  
│ │ │ └── rbw  
│ │ ├── dncp_block_verification.log.curr  
│ │ ├── dncp_block_verification.log.prev  
│ │ └── tmp  
│ └── VERSION  
└── in_use.lock  
```

blk开头的文件就是存储HDFS block的数据块

> * VERSION：与Namenode类似   
> * finalized/rbw目录：于实际存储HDFS BLOCK的数据，里面包含许多block_xx文件以及相应的.meta文件，.meta文件包含了checksum信息   
> * dncp_block_verification.log：用于追踪每个block最后修改后的checksum值   
> * in_use.lock：防止一台机器同时启动多个Datanode进程导致目录数据不一致

## 安全模式

安全模式是指Namenode在刚启动的时候，加载fsimage合并edits文件的过程，**一旦加载完毕将会在磁盘中为自己创建一个新的fsimage和空的edits文件**   
该过程中无法对文件系统进行写操作

关于安全模式的一些命令：

> * hadoop dfsadmin -safemode get：查询是够处于安全模式   
> * hadoop dfsadmin -safemode wait：执行某命令以前先退出安全模式   
> * hadoop dfsadmin -safemode enter：进入安全模式   
> * hadoop dfsadmin -safemode leave：退出安全模式  

## 管理工具

### dfsadmin

hadoop dfsadmin既可以查询HDFS状态信息，也可以进行HDFS管理操作，用途较广   
该命令参数较多，可以使用-help选项来查看使用帮助

### fsck

即为Filesystem check，文件系统检查，可以得到HDFS指定目录的健康状态信息   
该工具可以找出HDFS上缺失、过多、过少或者损坏的数据块   
fsck将从Namenode中获得所有信息，而不会去Datanode获取真实的block数据

```
hadoop fsck /user/heybiiiiii/part-00007 -files -blocks -racks
```

以上命令演示了如何通过fsck工具来为一个文件查找其block信息

> * -files：显示该文件的文件名、大小、block数量和健康状态   
> * -blocks：显示该文件每个block信息，每行一条信息   
> * -racks：显示每个block所在的Datanode和机架位置信息

### Datanode的块扫描器

每个Datanode都运行着一个块扫描器定期检测节点上的所有块   
从而在客户端读到损坏的数据块之前及时的检车和修复   
默认情况下每个504个小时（三周）会自动检查一个，该值可以通过**dfs.datanode.scan.period.hours**选项来配置   
用户可以通过访问   
http://datanode:50075/blockScanerReport   
来查看该Datanode的块检测报告

### Balancer

随着时间的推移，各个Datanode上的数据的分布会越来越不均衡，这样会降低MapReduce的数据本地性导致性能降低，部分Datanode相对繁忙   
balancer能够将数据从繁忙的Datanode中迁移到较为空闲的Datanode，从而从新分配数据块，**在迁移数据的过程中也会始终坚持HDFS的副本存放策略**

balancer一旦启动，将会不断地移动数据，直至集群中的数据达到一个平衡的状态   
该状态通过**该节点已用空间和整个集群可用空间之比**和**集群中已用空间和整个集群可用空间之比**来判断   
一个集群中只能运行一个balancer程序，命令如下：

```
start-balancer.sh
```

通过-threshold选型来指定一个阈值以判断集群是否平衡，默认为10%

## 日常运维

### 元数据备份

一旦元数据出现损坏，整个集群将无法工作，所以元数据的备份是非常重要的   
可以在系统中分别保存不同时间点的备份（一小时、一天、一周或者一个月）以保护元数据

一个最直接的方法就是使用dfsadmin命令来下载一个Namenode最新的元数据副本：

```
hdfs dfsadmin -fetchImage fsimage.backup
```

可以通过一个脚本定时在异地站点存储元数据信息

### 数据备份

尽管HDFS的数据容错性做的很好，但是我们仍然不能够保证其永远不会出错   
HDFS上的数据备份可以通过**distcp**工具来实现集群数据的复制   
具体使用方法请参考：

### 文件系统检查

建议定期使用fsck工具对HDFS上的数据块进行检查，主动查找丢失或者损坏的数据块

### 文件系统均衡

定期运行balancer程序以保证各个Datanode的数据平衡

## 添加和删除节点

hdfs-site.xml和yarn-site.xml文件中分别有以下属性：

> * dfs.hosts：运行连接的Datanode列表   
> * dfs.hosts.exclude：不允许连接的Datanode列表
> * yarn.resourcemanager.nodes.include-path：允许连接的NodeManager列表   
> * yarn.resourcemanager.nodes.exclude-path：不允许连接的NodeManager列表

### 添加节点

过程如下：

> 1.将新机器的地址加入以上**允许连接**的列表中   
> 2.使用hdfs dfsadmin -refreshNodes/yarn rmadmin -refreshNodes命令重新读取配置   
> 3.在slaves文件中添加该节点信息，**使得hadoop脚本文件可以读取到**   
> 4.在新节点配置完之后启动Datanode和NodeManager进程   
> 5.可以在We界面查看该节点是否上线

**注意：HDFS不会自动将数据迁移到新节点，需要手动启动balancer进程**

### 删除节点

过程如下：

> 1.将新机器的地址加入以上**不允许连接**的列表中   
> 2.使用hdfs dfsadmin -refreshNodes/yarn rmadmin -refreshNodes命令重新读取配置   
> 3.在Web界面可以看到该节点处于Decommission in progress状态，节点上的数据将会被迁移到其他节点中   
> 4.当节点处于Decommissioned状态时表示数据块复制完毕，关闭该节点上的进程
> 5.从**允许连接**的列表中移除该节点并重新执行命令刷新配置
> 6.在slaves文件中移除该节点

**注意，副本数量要小于或者等于正常节点的数量，否则删除失败**

**【已解决】删除节点时，该节点长期处于Decommission Status : Decommission in progress状态，由于数据量太大，导致复制的时间很久，使用新集群测试时瞬间下线该节点**

## 集群升级

在升级集群之前可以现在一个测试集群上进行测试确保升级无误之后再在生产集群上进行升级

仅当文件系统健康时才能进行升级，所以在升级之前使用fsck工具对文件系统进行检查   
最好保留检查结果，对比升级前和升级后的信息

升级之前最好清空临时文件，包括HDFS、MapReduce系统目录和本地临时文件

升级步骤如下：

> 1.升级之前确保前一升级已经定妥   
> 2.关闭Yarn和MapReduce进程   
> 3.关闭HDFS进程并备份元数据目录   
> 4.在集群中安装新版本的Hadoop    
> 5.使用-upgrade选项来启动HDFS    
> 6.等待等级完成   
> 7.检查HDFS是否运行正常   
> 8.启动Yarn和MapReduce进程   
> 9.回滚（可选操作）

步骤5中使用的命令如下：

```
${NEW_HADOOP}/bin/start-dfs.sh -upgrade
```

该命令的结果是让Namenode升级元数据，并将前一版本放在名为previous目录下，Datanode的操作类似

步骤6中，可以使用dfsadmin工具来查看升级进度：

```
${NEW_HADOOP}/bin/hdfs dfsadmin -upgradeProgress status
```

如果需要对升级进行回滚，则进行步骤9的操作：

首先关闭HDFS进程   
使用-rollback选项来启动旧版本的Hadoop进程：

```
${OLD_HADOOP}/in/start-dfs.sh -rollback
```

该命令会让Namenode和Datanode使用升级之前的副本替换当前存储目录，文件系统恢复之前的状态ghe

作者：[@小黑](http://www.xiaohei.info)