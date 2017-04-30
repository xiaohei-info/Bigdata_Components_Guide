当遇到有特殊的业务需求时，需要对hadoop的作业进行分区处理
那么我们可以通过自定义的分区类来实现。
还是通过单词计数的例子，JMapper和JReducer的代码不变，只是在JSubmit中改变了设置默认分区的代码，见代码：

```java
//1.3分区  
//设置自定义分区类  
job.setPartitionerClass(JPartitioner.class);  
//设置分区个数--这里设置成2，代表输出分为2个区，由两个reducer输出  
job.setNumReduceTasks(2);
```

自定义的JPartitioner代码如下：

```java
import org.apache.hadoop.io.LongWritable;  
import org.apache.hadoop.io.Text;  
import org.apache.hadoop.mapreduce.lib.partition.HashPartitioner;  
  
//自定义的分区类必须继承Partitioner类，这里只要继承默认的HashPartitioner，并重写getPartition方法即可  
public class JPartitioner extends HashPartitioner<Text, LongWritable> {  
    @Override  
    public int getPartition(Text key, LongWritable value, int numReduceTasks) {  
        //由于之前在代码中设置了分区的个数为2,  
        //getPartition方法的返回值就是分区的下标，如：第一个分区return 0，第二个return 1  
        //如果key的长度小于4，那么将这些键值对分入第一个区  
        //否则就分入第二个区，numReduceTasks是设置的分区数量  
        return key.toString().length() < 4 ? 1 % numReduceTasks:2 % numReduceTasks;  
    }  
}
```

自定义分区就完成了

如果在海量数据的情况下，可能要设置归约（combiner）来减轻网络和reducer的压力
那么可以再JSubmit中通过代码设置combiner的类来启动
代码很简单，就一句话

```java
//1.5归约  
job.setCombinerClass(JReducer.class);
```

其实combiner和reducer都是设置的JReducer
侧面反映了combiner的角色作就是本地的reducer

作者：[@小黑](http://www.xiaohei.info)