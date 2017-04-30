## MapReduce原理

要知道怎么对MapReduce作业进行调优前提条件是需要对Map-Reduce的过程了然于胸。  
Map-Reduce运行原理图：   
![Map-Reduce运行原理图](/images/mr-shuffle.jpg)

### Map Side

**1.从磁盘读取数据并分片**

默认每个block对应一个分片，一个map task

**2.进行map处理**

运行自定义的map业务过程

**3.输出数据到缓冲区中**

map输出的数据并不是直接写入磁盘的，而是会先存储在一个预定义的buffer中

**4、分区、排序分组的过程**

对map输出的数据进行分区，按照key进行排序和分组

**5、归约（可选）**

相当于本地端的reduce过程

**6、合并写入磁盘**

对map的最终数据进行merge之后输出到磁盘中等待shuffle过程

### Reduce side

**1.从map端复制数据**

**2.对数据进行合并**

以上两个步骤即为shuffle过程

**3.对数据进行排序**

**4.进行reduce操作**

**5.输出到磁盘**

详细的过程将会在调优技巧中体现出来

## 最简单的调优方式

### 设置Combiner

Combiner在Map端提前进行了一次Reduce处理。   
可减少Map Task中间输出的结果，从而减少各个Reduce Task的远程拷贝数据量，最终表现为Map Task和Reduce Task执行时间缩短。

### 选择合理的Writable类型

为应用程序处理的数据选择合适的Writable类型可大大提升性能。   
比如处理整数类型数据时，直接采用IntWritable比先以Text类型读入在转换为整数类型要高效。   
如果输出整数的大部分可用一个或两个字节保存，那么直接采用VIntWritable或者VLongWritable，它们采用了变长整型的编码方式，可以大大减少输出数据量。

## 作业级别调优

### 增加输入文件的副本数

假设集群有1个Namenode+8个Datanode节点，HDFS默认的副本数为3   
那么map端读取数据的时候，在启动map task的机器上读取本地的数据为3/8，一部分数据是通过网络从其他节点拿到的   
那么如果副本数设置为8会是什么情况？   
相当于每个子节点上都会有一份完整的数据，map读取的时候直接从本地拿，不需要通过网络这一层了

但是在实际情况中设置副本数为8是不可行的，因为数据本身非常庞大，副本数超过5对集群的磁盘就非常有压力了，**所以这项设置需要酌情处理**

该配置在hdfs-side.xml的dfs.replication项中设置

## Map side tuning

### InputFormat

这是map阶段的第一步，从磁盘读取数据并切片，每个分片由一个map task处理

当输入的是海量的小文件的时候，会启动大量的map task，效率及其之慢，有效的解决方式是使用CombineInputFormat自定义分片策略**对小文件进行合并处理**   
从而减少map task的数量，减少map过程使用的时间   
详情请看：[自定义分片策略解决大量小文件问题](http://www.xiaohei.info/2016/03/01/%E8%87%AA%E5%AE%9A%E4%B9%89%E5%88%86%E7%89%87%E7%AD%96%E7%95%A5%E8%A7%A3%E5%86%B3%E5%A4%A7%E9%87%8F%E5%B0%8F%E6%96%87%E4%BB%B6%E9%97%AE%E9%A2%98/)

另外，map task的启动数量也和下面这几个参数有关系：

> * mapred.min.split.size：Input Split的最小值 默认值1
> * mapred.max.split.size：Input Split的最大值
> * dfs.block.size：HDFS 中一个block大小，默认值128MB

当mapred.min.split.size小于dfs.block.size的时候，一个block会被分为多个分片，也就是对应多个map task
当mapred.min.split.size大于dfs.block.size的时候，一个分片可能对应多个block，也就是一个map task读取多个block数据

集群的网络、IO等性能很好的时候，建议**调高dfs.block.size**
根据数据源的特性，主要调整mapred.min.split.size来控制map task的数量

### Buffer

该阶段是map side中将结果输出到磁盘之前的一个处理方式，通过对其进行设置的话可以**减少map任务的IO开销，从而提高性能**

由于map任务运行时中间结果首先存储在buffer中,默认当缓存的使用量达到80%的时候就开始写入磁盘,这个过程叫做spill(溢出)   
这个buffer默认的大小是100M可以通过设定**io.sort.mb**的值来进行调整

当map产生的数据**非常大**时，如果默认的buffer大小不够看，那么势必会**进行非常多次的spill，进行spill就意味着要写磁盘，产生IO开销**   
这时候就可以**把io.sort.mb调大**，那么map在整个计算过程中spill的次数就势必会降低，map task对磁盘的操作就会变少   
如果map tasks的瓶颈在磁盘上，这样调整就会大大提高map的计算性能

**但是如果将io.sort.mb调的非常大的时候，对机器的配置要求就非常高，因为占用内存过大，所以需要根据情况进行配置**

map并不是要等到buffer全部写满时才进行spill，因为如果全部写满了再去写spill，势必会造成map的计算部分等待buffer释放空间的情况。   
所以，map其实是当buffer被写满到一定程度（比如80%）时，才开始进行spill   
可以通过设置**io.sort.spill.percent**的值来调整这个阈值   
这个参数同样也是影响spill频繁程度，进而影响map task运行周期对磁盘的读写频率

**但是通常情况下只需要对io.sort.mb进行调整即可**

### Merge

该阶段是map产生spill之后，对spill进行处理的过程，通过对其进行配置也可以达到**优化IO开销的目的**

map产生spill之后必须将些spill进行合并,这个过程叫做merge   
merge过程是并行处理spill的,每次并行多少个spill是由参数**io.sort.factor**指定的,默认为10个

如果产生的spill非常多，merge的时候每次只能处理10个spill，那么还是会造成频繁的IO处理   
适当的**调大每次并行处理的spill数**有利于减少merge数因此可以影响map的性能

**但是如果调整的数值过大，并行处理spill的进程过多会对机器造成很大压力**

### Combine

我们知道如果map side设置了Combiner，那么会根据设定的函数对map输出的数据进行一次类reduce的预处理   
但是和分组、排序分组不一样的是，combine发生的阶段**可能是在merge之前，也可能是在merge之后**

这个时机可以由一个参数控制：**min.num.spill.for.combine**，默认值为3   
当job中设定了combiner，并且spill数最少有3个的时候，那么combiner函数就会在merge产生结果文件之前运行

例如，产生的spill非常多，虽然我们可以通过merge阶段的io.sort.factor进行优化配置，但是在此之前我们还可以通过**先执行combine对结果进行处理**之后再对数据进行merge   
这样一来，到merge阶段的数据量将会进一步减少，IO开销也会被降到最低

### 输出中间数据到磁盘

这个阶段是map side的最后一个步骤，在这个步骤中也可以通过**压缩选项**的配置来得到任务的优化

其实无论是spill的时候，还是最后merge产生的结果文件，都是可以压缩的   
压缩的好处在于，通过压缩减少写入读出磁盘的数据量。对中间结果非常大，磁盘速度成为map执行瓶颈的job，尤其有用

控制输出是否使用压缩的参数是**mapred.compress.map.output**，值为true或者false   
启用压缩之后，会牺牲CPU的一些计算资源，但是可以节省IO开销，非常适合IO密集型的作业（如果是CPU密集型的作业不建议设置）

设置压缩的时候，我们可以选择不同的压缩算法   
Hadoop默认提供了GzipCodec，LzoCodec，BZip2Codec，LzmaCodec等压缩格式

通常来说，想要达到比较平衡的cpu和磁盘压缩比，LzoCodec比较合适，但也要取决于job的具体情况   
如果想要自行选择中间结果的压缩算法，可以设置配置参数：   
```java
mapred.map.output.compression.codec=org.apache.hadoop.io.compress.DefaultCodec
//或者其他用户自行选择的压缩方式
```

使用示例：

```java
LzopCodec c = new LzopCodec();
c.setConf(conf);
OutputStream os = c.createOutputStream(fs.create(new Path(“file.txt”))); 
InputStream is = c.createInputStream(fs.open(new Path(“in.lzo”)));
```

在core-site.xml中需要添加压缩选项的支持:

```xml
<property>
<name>io.compression.codecs</name> 
<value>org.apache.hadoop.io.compress.GzipCodec,
com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.com press.SnappyCodec
</value> 
</property>
```

### Map side tuning总结

从上面提到的几点可以看到，map端的性能瓶颈都是频繁的IO操作造成的，所有的优化也都是针对IO进行的，而优化的瓶颈又很大程度上被机器的配置等外部因素所限制

map端调优的相关参数：

|选项|类型|默认值|描述|
|---|---|---|---|
|mapred.min.split.size|int|1|Input Split的最小值|
|mapred.max.split.size|int|.|Input Split的最大值|
|io.sort.mb|int|100|map缓冲区大小|
|io.sort.spill.percent|float|0.8|缓冲区阈值|
|io.sort.factor|int|10|并行处理spill的个数|
|min.num.spill.for.combine|int|3|最少有多少个spill的时候combine在merge之前进行|
|mapred.compress.map.output|boolean|false|map中间数据是否采用压缩|
|mapred.map.output.compression.codec|String|.|压缩算法|

## Reduce side tuning

### Shuffle

**1.Copy**

由于job的每一个map都会根据reduce(n)数将数据分成map 输出结果分成n个partition，所以map的**中间结果中是有可能包含每一个reduce需要处理的部分数据的**   
为了优化reduce的执行时间，hadoop中**等第一个map结束后，所有的reduce就开始尝试从完成的map中下载该reduce对应的partition部分数据**

在这个shuffle过程中，由于map的数量通常是很多个的，而每个map中又都有可能包含每个reduce所需要的数据   
所以对于每个reduce来说，**去各个map中拿数据也是并行的**，可以通过**mapred.reduce.parallel.copies**这个参数来调整，默认为5   
当map数量很多的时候，就可以适当调大这个值，**减少shuffle过程使用的时间**

还有一种情况是：reduce从map中拿数据的时候，有可能因为中间结果丢失、网络等其他原因**导致map任务失败**   
而reduce不会因为map失败就永无止境的等待下去，它会尝试去别的地方获得自己的数据（这段时间失败的map可能会被重跑）   
所以设置reduce获取数据的超时时间可以避免一些因为网络不好导致无法获得数据的情况   
**mapred.reduce.copy.backoff**，默认300s   
**一般情况下不用调整这个值，因为生产环境的网络都是很流畅的**

**2.Merge**

由于reduce是并行将map结果下载到本地，所以也是需要进行merge的，**所以io.sort.factor的配置选项同样会影响reduce进行merge时的行为**

和map一样，reduce下载过来的数据也是存入一个buffer中而不是马上写入磁盘的，所以我们同样可以控制这个值来减少IO开销   
控制该值的参数为：   
**mapred.job.shuffle.input.buffer.percent**，默认0.7，这是一个百分比，意思是reduce的**可用内存中拿出70%作为buffer存放数据**

reduce的可用内存通过**mapred.child.java.opts来设置**，比如置为-Xmx1024m，该参数是同时设定map和reduce task的可用内存，**一般为map buffer大小的两倍左右**

设置了reduce端的buffer大小，我们同样可以通过一个参数来控制buffer中的数据达到一个阈值的时候开始往磁盘写数据：**mapred.job.shuffle.merge.percent**，默认为0.66   

### Sort

sort的过程一般非常短，因为是边copy边merge边sort的，后面就直接进入真正的reduce计算阶段了

### Reduce

之前我们说过reduc端的buffer，默认情况下，数据达到一个阈值的时候，buffer中的数据就会写入磁盘，然后**reduce会从磁盘中获得所有的数据**   
也就是说，buffer和reduce是没有直接关联的，中间多个一个写磁盘->读磁盘的过程，既然有这个弊端，那么就可以通过参数来配置   
**使得buffer中的一部分数据可以直接输送到reduce**，从而减少IO开销：**mapred.job.reduce.input.buffer.percent**，默认为0.0

当值大于0的时候，会保留指定比例的内存读buffer中的数据直接拿给reduce使用   
这样一来，设置buffer需要内存，读取数据需要内存，reduce计算也要内存，所以要根据作业的运行情况进行调整

### Reduce side tuning总结

和map阶段差不多，reduce节点的调优也是主要集中在加大内存使用量，减少IO，增大并行数

reduce调优主要参数：

|选项|类型|默认值|描述|
|---|---|---|---|
|mapred.reduce.parallel.copies|int|5|每个reduce去map中拿数据的并行数|
|mapred.reduce.copy.backoff|int|300|获取map数据最大超时时间|
|mapred.job.shuffle.input.buffer.percent|float|0.7|buffer大小占reduce可用内存的比例|
|mapred.child.java.opts|String|.|-Xmx1024m设置reduce可用内存为1g|
|mapred.job.shuffle.merge.percent|float|0.66|buffer中的数据达到多少比例开始写入磁盘|
|mapred.job.reduce.input.buffer.percent|float|0.0|指定多少比例的内存用来存放buffer中的数据|

## MapReduce tuning总结

Map Task和Reduce Task调优的一个原则就是   
**减少数据的传输量**   
**尽量使用内存**   
**减少磁盘IO的次数**   
**增大任务并行数**   
除此之外还有根据自己集群及网络的实际情况来调优

### Map task和Reduce task的启动数

在集群部署完毕之后，根据机器的配置情况，我们就可以通过一定的公式知道**每个节点上container的大小和数量**

**1.mapper数量**

每个作业启动的mapper由输入的分片数决定，**每个节点启动的mapper数应该是在10-100之间，且最好每个map的执行时间至少一分钟**   
如果输入的文件巨大，会产生无数个mapper的情况，应该使用**mapred.tasktracker.map.tasks.maximum**参数确定每个tasktracker能够启动的最大mapper数，默认只有2   
以免同时启动过多的mapper

**2.reducer数量**

reducer的启动数量官方建议是0.95或者1.75*节点数*每个节点的container数   
使用0.95的时候reduce只需要一轮就可以完成   
使用1.75的时候完成较快的reducer会进行第二轮计算，并进行负载均衡   
**增加reducer的数量会增加集群的负担，但是会得到较好的负载均衡结果和减低失败成本**

一些详细的参数：

|选项|类型|默认值|描述|
|---|---|---|---|
|mapred.reduce.tasks|int|1|reduce task数量|
|mapred.tasktracker.map.tasks.maximum|int|2|每个节点上能够启动map task的最大数量|
|mapred.tasktracker.reduce.tasks.maximum|int|2|每个节点上能够启动reduce task的最大数量|
|mapred.reduce.slowstart.completed.maps|float|0.05|map阶段完成5%的时候开始进行reduce计算|

map和reduce task是同时启动的，很长一段时间是并存的   
共存的时间取决于**mapred.reduce.slowstart.completed.maps**的设置   
如果设置为0.6.那么reduce将在map完成60%后进入运行态

如果设置的map和reduce参数都很大，势必造成map和reduce争抢资源，造成有些进程饥饿，超时出错，最大的可能就是socket.timeout的出错

reduce是在33%的时候完成shuffle过程，所以**确保reduce进行到33%的时候map任务全部完成**，可以通过观察任务界面的完成度进行调整   
当reduce到达33%的时候，map恰好达到100%设置最佳的比例，可以让map先完成，但是**不要让reduce等待计算资源**

作者：[@小黑](http://www.xiaohei.info)
