## 存储结构

### HBase构架

![HBase存储结构](/images/hbase-framework.png)

如上图所示，一个HBase集群是由Zookeeper、HMaster和HRegionServer构成的

**HRegionServer**

HBase集群上的各个节点，一个数据量很大的表可能被保存在不同RegionServer上

**HLog**

HBase将数据存储在各个HRegionServer上，每个HRegionServer都有一个HLog文件记录该节点上数据的CRUD操作记录
图中错误的地方在于HLog是和HRegionServer一个级别的，而不是里面的HRegion
HLog是Hadoop上的sequence file文件，key为HLogKey对象，记录了hbase读写操作的日志，value是hbase的key-value对象

**HRegion**

按照Rowkey横向切分的多个块，保存着表中某段连续的数据
当某个RegionServer上的HRegion过大时会进行水平切分
如果HRegion数过多，HBase会选择一个比较空闲的RegionServer来保存一部分的HRegion
这样一来一个表的数据就被保存在不同RegionServer上的不同HRegion中

**Store**

HRegion中包含多个Store，每个Store对应HBase表中的一个列族，有几个列族就有几个Store
Store分为两种：

> 1.MemStore
> 2.StoreFile

向表中写数据的时候先写入内存的MemStore中，MemStore达到一个阈值的时候会产生溢写，写数据到磁盘中形成StoreFile
读表数据时也是先查MemStore中有没有该数据，没有才去找StoreFile

**HFile**

HBase数据最后存储在HDFS上的形式，StoreFile其实是HFile的一个轻量级封装，底层就是HFile

## HBase在HDFS上的目录结构

### 根级文件

**.logs目录**

HDFS上的HBase目录默认为/hbase，进入/hbase目录之后可以看到/hbase/.logs文件夹
这是被HLog管理的WAL文件，对于每个RegionServer，该目录中都会包含有一个对应的子目录
因为日志滚动的原因每个子目录中有多个HLog文件
日志滚动将在后面详细说明

**hbase.id和hbase.version**

顾名思义，分别包含集群的唯一ID和文件格式的版本信息，它们在内部使用，且携带的信息不多

**.splitlog和.corrupt**

日志拆分过程中产生的文件夹，详情见数据恢复小节

### 表级文件

每个表在HBase中都会以一个单独的目录存储，每个表目录包含一个名为table.info的顶层文件
该文件存储表对应的序列化之后的HTableDescriptor，其中包含表和列族的定义
.tmp目录中包含一些临时数据，例如当更新.tableinfo文件时生成的临时数据就会被存放到该目录中

### region级文件

每个表目录中，表模式的每个列族都有一个单独的目录，这些目录的名称为一串MD5值
每个region目录中都有一个.regioninfo文件包含了对应region的HRegionInfo实例序列化之后的信息

**Region的拆分**
当一个region里的存储文件增长到大于配置的hbase.hregion.max.filesize或者列族层面配置的大小时region就需要拆分
其会创建一个对应的splits目录用来临时存放两个子region相关的数据，包括region目录和参考文件
然后关闭原始region不再接受请求
一旦拆分成功之后子region的数据会被移动到表目录中并形成两个新的region，分别为原来region的一半
此时.META.表中原始region的状态会被更新以表示其现在拆分的节点和子节点是什么（.META.表的信息说明）

当两个region都准备好之后将会被同一个服务器并行打开，打开的过程包括更新.META.表的新region信息
使得两个新的region可以上线接受请求
最终原始region会被清理掉，也就是说它在.META.表中的表项会被移除，并且磁盘上的文件将被删除
最后master被告知关于拆分的情况，并且可以由于负载均衡而把新的region移动到其他region机器上

## 合并
之前说过，Store中的MemStore会溢写产生StoreFile
当MemStore溢写次数过多的时候会产生多个StoreFile，导致HDFS上有大量HFile小文件
所以当StoreFile数量大小达到一个阈值的时候HBase会对其进行合并（Compaction）
这个过程持续到这些文件中最大的文件超过配置的最大存储文件大小，此时会触发一个region拆分

Compaction分为两种：

> 1.major compaction
> 2.mirror compaction

major默认24小时执行一次，会将全部的StoreFile合并起来，消耗大量的io，最好手动写程序控制执行的时间，放在闲时处理

mirror可以通过配置文件设定当StoreFile文件大小、数量达到设定的值的时候，对指定的多少个StoreFile文件进行合并
相关配置项如下：

hbase.hstore.compaction.min :默认值为 3，表示至少需要三个满足条件的StoreFile时，minor compaction才会启动
hbase.hstore.compaction.max :默认值为10，表示一次minor compaction中最多选取10个StoreFile
hbase.hstore.compaction.min.size :表示文件大小小于该值的StoreFile 一定会加入到minor compaction的StoreFile中
hbase.hstore.compaction.max.size :表示文件大小大于该值的StoreFile 一定会被minor compaction排除
hbase.hstore.compaction.ratio :将StoreFile 按照文件年龄排序（older to younger），minor compaction总是从older StoreFile开始选择，如果该文件的size 小于它后面hbase.hstore.compaction.max 个StoreFile size 之和乘以 该ratio，则该StoreFile 也将加入到minor compaction 中。

## 日志滚动

日志的写入大小是有限制的，所谓的日志滚动就是，日志文件在到达指定的时间时会被关闭，并启用新的日志文件
该时间通过hbase.regionserver.logroll.period属性来控制，默认是一小时

经过一段时间后，系统会积攒一系列不断增长的日志文件，LogRoller类会调用HLog.closeOldLogs来检查写入到存储文件中的最大序列号是多少
因为到这个序列号为止，所有之前的修改都被持久化了，然后它会检查是不是所有日志文件的序列号都小于这个数字
如果是的话，就将这些文件移动到.lodlogs文件夹中，留下其余没有被持久化的数据更改日志

另外也可以通过hbase.regionserver.hlog.blocksize和hbase.regionserver.logroll.multiplier来控制日志滚动的时机
前者是日志块的大小（默认32M），后者表示当日志达到块大小的95%（默认）的时候就会滚动日志

当条件满足阈值时就会触发日志滚动，在10分钟后（默认）.lodlogs下的旧日志文件将会被master删除
该时间可以通过hbase.master.logleaner.ttl属性控制
master每隔一分钟将会检查一次这些文件，这是通过hbase.master.cleaner.interval属性设置的

那么为什么需要日志滚动呢？为什么不直接在一个日志文件中写入即可？
因为每次向HBase写入数据的时候都会先写入WAL中，之后才写入对应region的memstore
而memstore会产生溢写数据持久化到HDFS，已经持久化了的数据记录就不需要继续保存在WAL中了
过程如下：

> 1.将数据记录在提交日志中（WAL预写日志）
> 2.之后将数据写入内存中的memstore
> 3.memstore数据量达到阈值之后移出内存作为HFile写入磁盘
> 4.删除已经持久化到磁盘的WAL内容

所以需要一种机制可以按时删除WAL中已经持久化的数据记录，即前面所描述的日志滚动和日志清除

向HBase写入的数据都会先写入到WAL中，当HRegion被实例化的时候
HLog实例会被当做一个参数传入到HRegion的构造器中，当一个region接收到一个更新操作的时候
它可以直接将数据保存到一个共享的WAL中

## 数据恢复 

如同之前讨论的同一个region服务器的数据更新都会被写入到同一个HLog中
那么为什么不分开将每个region的所有数据更改都写入到一个单独的日志文件中去呢？
因为HBase的底层存储依赖于HDFS，同时写入太多的文件且需要保持滚动的日志会影响系统的可扩展性
这些写入会导致大量的硬盘寻道来向不同的物理日志文件中写入数据

正是因为这个原因，导致了多个region的数据更改记录混合在同一个文件中

如果用户的数据被及时的、安全的持久化了，那么所有的事情就非常简单了
但是只要遇到服务器崩溃，系统就需要拆分日志
现在的问题是，所有的数据更改日志都混在同一个日志文件中，并且没有任何索引
由于这个原因，master不可能立即把一个崩溃的服务器上的region部署到其他服务器上
它需要等待对应的region日志被拆分出来

在集群启动或者某个服务失效的时候，master会检查WAL中的数据以恢复没有持久化到硬盘的数据，这个过程叫做日志回放
在日志如回放之前，如同我们之前讨论的，日志需要被单独放在每个region对应的单独的日志文件中，这个过程叫做日志拆分：
读取混在一起的日志，并且所有的条目都按照它所归属的region来分组，这些分组的修改记录被存放在一个紧挨着目标region的文件中以供接下来的数据恢复过程使用

日志的拆分模式有两种，分别是早期的master线程处理和现在的分布式日志拆分
早期版本会直接在master上启动一个线程来读取文件，在有很多region服务器的集群上
master只能串行地恢复每个日志文件，这样master才不会在IO和内存方面使用过载
这意味着每个被挂起的数据更改的region都会被阻塞知道日志拆分以及恢复完成之后才能被打开

最新的分布式日志拆分中使用ZK来将每个被对其的日志文件分发给一个region服务器，他们通过检测ZK来发现需要执行的任务
一旦master指出某个日志是可以被处理的，那么它们就会竞争这个任务，获胜的region服务区就会在一个线程中读取并且拆分这个日志
如此一来切分日志的实际工作就从master转移到了region服务器上

可以通过hbase.master.distributed.log.splitting设置为false来关闭分布式日志拆分

拆分的过程首先会将数据改动写入到HBase根目录下的splitlog暂存文件夹，他们会被放置在对应region相同的路径下
为了与其他日志区分开能够执行并发操作，路径中包含了日志文件名、表名、region名和recovered.edits文件夹

.corrupt文件夹包含所有不能被解析的日志文件，任何不能从日志文件中解析出来的数据更改都会使整个日志文件被移动到该文件夹中
hbase.hlog.split.skip.errors属性为false的时候回直接抛出IO异常，日志拆分过程将被打断

日志拆分成功之后就是数据恢复的过程了

当集群启动，或者一个region移动到另外一个region服务器，region都会被打开
此时region会先检查recovered.edits目录是否存在，如果存在那么久开始读取文件所包含的数据更改记录
由于文件是按照含序列ID的文件名排序的，任何序列ID小于或者等于保存在磁盘中存储文件序列ID的记录都会被忽略，因为这些记录之前已经被刷写了
其他的数据会被添加到region的memstore中恢复之前的数据状态，最后一次强制的memstore刷写将数据写入磁盘

## 读路径

由于存储文件都是不可变的，从这些文件中删除一个特定的值是做不到的，通过重写存储文件将已经被删除的单元格移除也是毫无意义的且要做很多操作
所以HBase删除数据的过程是使用“墓碑标记”，如下过程：

> 1.在要删除的数据上设置删除标志位
> 2.查询数据的时候忽略删除标志位的数据
> 3.在文件合并的时候跳过删除标志位的数据
> 4.最后形成的存储文件中没有要删除的数据，删除此时生效

当用户向行中的不同列写入数据的时候，逻辑上一整行的数据到底存储在哪里呢？
作为客户端，当我们发出get命令的时候我们希望一整行都被返回（也就是多个列），就好像跟关系型数据库一样他们是被存储在一个实体中的
但是实际上不同的列族是分开存储的，一行的数据可能横跨任意数目个存储文件
实际上HBase查找数据的时候主要是通过两个类来实现的：QueryMatcher和ColumnTracker
前者会精确匹配用户要取出的列，后者则追踪所有改行包含的列

region第一次找到的时候缓存是空的，为了找到对应的数据，客户端需要发出三次查询：

> 1.连接到ZK查询-ROOT-表所在的位置
> 2.在对应的服务器上查询.META.表的位置
> 3.查询对应服务器上的.META.表找到对应的region服务器
> 4.在该服务器上打开region读取数据
> 5.HBase将该次请求缓存，下次直接访问Region服务器而不去找到-ROOT-

## Region的生命周期

如下表：

|状态|描述|
|---|---|
|Offline	|region下线|
|Pending Open	|打开region的请求已经发送到了服务器|
|Opening	|服务器打开了reign|
|Open	|region已经打开完全可以使用|
|Pending Close	|关闭region的请求发送到服务器|
|Closing	|服务器正在处理关闭的region|
|Splitting	|服务器开始切分region|
|Split	|region已经被切分|


作者：[@小黑](xiaohei.info)