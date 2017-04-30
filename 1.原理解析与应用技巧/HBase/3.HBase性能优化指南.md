## 垃圾回收优化

当region服务器处理大量的写入负载时，繁重的任务会迫使JRE默认的内存分配策略无法保证程序的稳定性   
所以我们可能需要对region服务器的垃圾回收机制进行一些参数调整（因为master并不处理实际任务，所以没有优化的必要）

首先来了解JAVA内存中的几个概念

在[HBase构架](http://www.xiaohei.info/2016/07/10/hbase-framework/)中我们可以知道   
数据会被写入到memstore内存中直到达到一个阈值之后刷写持久化到磁盘   
但是由于数据是客户端在不同时间写入的，这些数据占据的JAVA内存中的堆空间很可能是不连续的，所以JAVA虚拟机的内存会出现“孔洞”

在JAVA的内存空间中，数据在内存停留的时间决定了该数据在内存中的位置分配   
被快速插入并刷写到磁盘的数据会被分配到**年轻代**的堆中，**这种空间可以被迅速回收，对内存管理没有太大影响，年轻代通常只占用128-512M的内存空间**   
在内存中国停留时间过长（例如向一个列族中插入数据过慢的时候），该数据有可能被放在**老生代**的堆中，**老生代会占用几乎可以占用的内存空间，通常是好几个G**

region服务器的默认配置中，年轻代的空间大小对于大多数负载来说都太小   
所以可以将其适当的调大，如果使用默认值的话，**频繁的从年轻代中收集对象对消耗大量CPU**，所以某些情况下用户可能会发现服务器CPU的使用量会急剧上升   
用户可以通过hbase-env.sh中的HBASE_REGIONSERVER_OPT选项设置为-Xmn128m来修改默认的配置，一般来说128M可以满足需求，具体的配置还需要参考实际的JVM指标

所谓的垃圾回收指的是：**为了重复使用由于数据刷写到磁盘产生的内存孔洞，JRE将压缩内存碎片使其再次被使用，同时将符合条件的年轻代数据提升到老生代等过程**   
推荐的垃圾回收策略是：**-XX:+UseParNewGC -XX:UseConcMarkSweepGC**

-XX:+UseParNewGC：   
该选项设置年轻代使用Parallel New Conllector垃圾回收策略   
效果是，清空年轻代堆的时候将停止运行的JAVA进程   
难道停止进程不会对集群有影响吗？   
对于年轻代来说基本是不会的，因为年轻代很小，这个过程花费的时间很短，通常只需要几百毫秒的时间   
但是对于老生代来说就不是这么回事了，如果在回收老生代空间的时候停止了JAVA进程   
**一旦清理的时间超出了ZK的会话超时限制，当这个region回过神来的时候就会发现已经被master抛弃了，然后自行关闭。。**   
基于这个情况，**年轻代的堆大小也不能设置得太大，以免垃圾回收的时候影响服务器的延迟**

所以对于老生代来说应该使用第二个选项的配置    
-XX:UseConcMarkSweepGC：   
该策略会视图在不停止运行JAVA进程的情况下尽可能的异步并行完成垃圾回收的工作，避免因为垃圾回收造成的region停顿   
但是缺点是将会增加CPU的负担

基于以上几点和gc日志相关，可以使用以下列内容作为集群的初始配置：

```shell
export HBASE_REGIONSERVER_OPT="-Xmx8g -Xms8g -Xmn128m -XX:+UseParNewGC -XX:UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 -verbose:gc -XX:+printGCDetails -XX:+PrintGCTimeStamps -Xloggc:${HBASE_HOME}/logs/gc-${hostname}-hbase.log"
```

## 使用压缩

HBase支持大量的压缩算法，可以从嫘祖级别上进行压缩   
因为CPU压缩和解压缩消耗的时间往往比从磁盘读取和写入数据要快得多   
所以使用压缩通常会带来很可观的性能提升

### 可用的编解码器

几个压缩算法的比较：

|算法|压缩比%|压缩MB/s|解压MB/s|
|---|---|---|---|
|GZIP|13.4|21|118|
|LZO|20.5|135|410|
|Zippy/Snappy|22.2|172|409|

关于压缩算法在HBase中的集成安装请Google之~

### 验证安装

安装完压缩算法之后我们可以使用一些工具对算法进行测试

**压缩测试工具**

HBase包含一个能够测试压缩设置是否正常的工具，我们可以输入

```shell
./bin/hbase org.apache.hadoop.hbase.util.CompresstionTest
```

用户需要制定一个测试文件和压缩算法，例如：

```shell
./bin/hbase org.apache.hadoop.hbase.util.CompresstionTest /user-home/test.gz gz
```

对于成功的安装，将会返回success信息   
反之则会得到一个异常

### 启用检查

及时测试工具报告成功了，由于JNI需要先安装好本地库，如果缺失这一步将会在添加新服务器的时候出现问题   
导致新的服务器使用本地库打开含有压缩列族的region失败

我们可以在服务器启动的时候检查压缩库是否已经正确安装，如果没有则不会启动服务器：

```xml
<property>
	<name>hbase.regionserver.codecs</name>
	<value>snappy,lzo</value>
</property>
```

这样一来region服务器在启动的时候将会检查Snappy和LZO压缩库是否已经正确安装

### 启用压缩

我们可以通过shell创建表的时候指定列族的压缩格式：

```shell
create 'testtable',{NAME => 'colfam1',COMPRESSION => 'GZ'}
```

需要注意的是，如果用户要更改一个已经存在的表的压缩格式，要先将该表disable才能修改之后再enable重新上线   
并且**更改后的region只有在刷写存储文件的时候才会使用新的压缩格式，没有刷写之前保持原样**    
用户可以通过shell的major_compact <tablename>来强制格式重写，但是此操作会占用大量资源

## 优化拆分和合并

默认的拆分和合并机制是合理的，但是在某些情况下我们仍然可以根据需求对这部分功能过呢进行优化以获得额外的性能

### 管理拆分

通常HBase是自动处理region拆分的：一旦它们增长到了既定的阈值region将被拆分为两个   
之后它们可以接受新的数据并继续增长

但是有没有这样一种情况：**拆分之后的两个子region都已恒定的速率增大，导致在同一时刻进行拆分**   
这种情况是很有可能出现的对吧，但是两个region同时拆分会怎么样吗？   
只有两个的话确实不会有什么影响，但是如果两个region拆分之后继续以恒定的速率增长导致子子region又一起拆分，甚至子子子region，无限循环...

这样一来问题就大条了吧，这种情况被称为**拆分/合并风暴**，这将导致磁盘IO的飙升

这种情况下，与其依赖HBase的自动拆分，用户不如手动使用split和major_compact命令来管理   
因为手动管理的话可以**将这些region的拆分/合并时机分割开来，尽量分散IO负载**   
碰到这种情况的时候不要忘记把配置文件中的**hbase.hregion.max.filesize**设置为非常大（例如100G？但是触发的时候回导致一小时的major合并...）   
来防止自定拆分/合并的发生

### region热点问题

在[HBase高级用法](http://www.xiaohei.info/2016/07/14/hbase-advenced-usage/)中我们提到过行健的设计避免数据热点问题的产生   
但是在生产环境中，即使使用了随机行健，某一台的机器负载仍然要大于其他机器   
对于很多region表来说，**大部分region分布并不均匀，即大多数region位于同一服务器上**

唯一可以缓解这种现象的途径是**手动将热点region按特定的便捷拆分出一个或者多个新region并负载分布到多个region服务器上**   
用户可以为region指定一个拆分行健，即region被拆分为两部分的位置   
这个行健可以任意指定，可以生成大小完全不同的region

### 预拆分region

HBase建表时默认只在一个HRegionServer上建一个HRegion，写数据全部往该HRegion写   
当这个HRegion达到一定大小的时候进行水平切割为多个HRegion，自动进行负载均衡   

在这个环节，我们可以通过预先创建一个空的HRegion，并规定好每个HRegion存储的Rowkey范围   
这样一来，**指定范围内的数据就会被写入到指定的HRegion中，可以省略很多IO操作**   

**通过对数据的特性进行分析预先创建分区可以有效的解决HBase中的数据倾斜问题**

以存储日志访问的记录为例   
根据数据量和每个HRegion的大小，预先创建空的HRegion，每个HRegion都设定了其RowKey的范围，插入数据时直接写入指定的HRegion中
或者根据数据的访问特性（28原则，进行抽样调查，得到数据频繁的情况），将经常访问的ip断分割为几个HRegion，其他为一个或者几个HRegion
根据具体需求确定预分区是预估还是抽样，预分区的时候尽可能的想到各种各样的情况

用户可以通过HBase提供的工具在创建表的时候指定region的个数：

```shell
/bin/hbase org.apache.hadoop.hbase.util.RegionSplitter
```

关于预拆分region的数量，可以先按照每个服务器10个region并随着时间的推移观察数据的增长情况   
先设置较少的region在稍后滚动拆分是一种很好的方式，因为一开始就设置很多region的话会影响集群的性能

## 执行负载均衡

master中有一个内置的负载均衡器，默认每五分钟执行一次（通过**hbase.balancer.period**属性控制）   
其功能是**均匀分配region到所有region服务器**

其实这个自动的均衡器所起到的效果和之前的手动管理region拆分是一样的   
都是为了将region的负载平衡到整个集群中

当用户想控制某张表特定region的确切位置的时候使用手动管理的方式是比较方便的   
我们可以通过HBase的balancer命令来显示的启动均衡器，或者通过balance_switch开启和关闭均衡器

## 合并region

在某些特殊的情况下，用户可能需要对服务器上的region进行合并（比如删除大量数据并且想减少服务器region数量的情况下？）   
HBase集成了一个工具能够让用户在集群没有工作的时候合并两个相邻的region：

```shell
/bin/hbase org.apache.hadoop.hbase.util.Merge
```

参数分别是表名要合并的两个region名，region的信息可以在WebUI中得到，也可以从scan .META表中获得

## 客户端API优化


写代码使用HBase客户端访问HBase的时候是有很大优化空间的   
下面给出一个列表参考：

### 禁用自动刷写

当有大量的写入操作时，使用setAutoFlush(false)关闭自动刷写特性    
这样一来，数据将会被写入缓存中达到指定值的时候一起发送到服务器   
也可以通过flushCommits()来强制刷写   
调用HTable的close方法会隐式的调用flushCommits()

### 使用扫描缓存

如果HBase作为一个MapReduce作业的而输入源，最好将MapReduce作业的输入扫描器实例的缓存用setCaching()设置为比1大的多的值   
例如设置为500的时候则一次可以传送500行数据到客户端进行处理

### 限定扫描范围

如果只处理少数列，则应当只有这些列被添加到Scan的输入中   
因为如果没有做筛选，则会扫描其他的数据存储文件

### 关闭ResultScanner

这不会带来性能提升，但是会避免一些性能问题   
所以一定要在try/catch中关闭ResultScanner

### 关闭Put的WAL

使用Put的writeToWAL(false)关闭WAL可以提高吞吐量   
但是带来的影响是在服务器故障的时候将会丢失数据   
而且关闭日志所带来的性能提升也不是很明显，所以不推荐使用

## 配置优化

启动集群的时候，配置文件中同样有很多可以优化的选项

### 减少Zookeeper超时的发生

默认region服务器和ZK的通信超时时间是3分钟   
这意味着**当服务器出现故障的时候，master会在3分钟之后发现并进行处理**

可以通过**zookeeper.session.timeout**将这个值设置为1分钟或者其他情况   
如此一来服务器当掉的时候master可以尽早发现

但是要注意的是：之前我们提到过，**在垃圾回收的时候JAVA进程将会被停止一段时间**    
如果这个值太小的话有可能造成ZK的误判

### 增加处理线程

默认情况下HBase响应外部用户访问数据表请求的线程数为10   
设置的有点小了，这是为了防止用户在客户端高并发使用较大写缓冲区的情况使得服务器过载

我们可以通过**hbase.regionserver.handler.count**属性将这个值设置为最大客户端数目   
但是也不宜太大，因为并发的写请求涉及到的数据累加起来之后可能会对一个region服务器的内存造成压力

### 增加堆大小

HBase使用默认一组合理并且保守的配置以满足大多数不同机型的测试   
如果用户有更好的服务器则可以分配给HBase8G甚至更大的空间

可以通过hbase-env.sh中的HBASE_HEAPSIZE来进行全局的设置   
如果只想更改region服务器的内存，则可以单独设置HBASE_REGIONSERVER_OPTS选项，master将会以默认的1G继续运行

### 增加region大小

更大的region可以减少集群总的region数量，一般来说**管理较少的region可以让集群更加平稳的运行**   
默认情况下region的大小是256MB，我们可以配置1GB或者更大的region   
region的大小也是需要评估的，因为**太大的region也以为这在高负载的情况下合并的停顿时间更长**

更多的配置调优可以参考《HBase权威指南》中性能优化的配置小节

作者：[@小黑](http://www.xiaohei.info)