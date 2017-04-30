## 输入格式

### 输入分片与记录

之前讨论过，输入数据的每个分片对应一个map任务来处理   
在MapReduce中输入分片被表示为InputSplit类，原型如下：

```java
public abstract class InputSplit{
	//该分片的长度，用于排序分片，有限处理大分片
	public abstract long getLength() throw IOException,InterruptedException;
	//该分片所在的存储位置（主机），不直接存储数据，而是指向数据的引用
	public abstract String[] getLocations() throw IOException,InterruptedException;
}
```

开发者不用直接操作InputSplit，**InputFormat根据输入的数据来创建计算的InputSplit并将其分割为一各个Record**   
该类的原型如下：

```java
public abstract class InputFormat<K,V>{
	public abstract List<InputSplit> getSplits(JobContext context) throw IOException,InterruptedException;
	public abstract RecordReader<K,V> createRecordReader(InputSplit inputSplit,TaskAttemptContext context) throw IOException,InterruptedException;
}
```

正如[MapReduce工作机制](http://www.xiaohei.info/2016/05/07/mapreduce-works/)中讨论到的一样   
客户端运行作业的时候会对输入的数据计算分片信息，调用的就是InputFormat的getSplits方法   
并将分片分析交给AM由此可以知道分片在集群上的存储位置并申请资源

在map任务中，将各自的输入分片传给InputFormat的createRecordReader方法来获得这个分片的RecordReader   
RecordReader是一个迭代器，map任务通过其遍历该分片内的数据，可以从Mapper类的run方法中看到：

```java
public void run(Context context) throw IOException,InterruptedException{
	super(context);
	//context的nextKeyValue将会委托给RecordReader的同名函数
	while(context.nextKeyValue()){
		//将当前的key和value传给map函数
		map(context.getCurrentKey(),context.getCurrentValue(),context);
	}
	cleanup(context);
}
```

### FileInputFormat

FileInputFormat是文件类的InputFormat的默认实现，其不能直接使用，但是提供了两个功能：

> 1.指定输入数据的路径   
> 2.输入文件生成分片的代码实现，即实现了InputFormat抽象类的getSplits方法

而如何把分片分割为记录（即InputFormat的createRecordReader方法）则由其具体的子类来完成   
常见的有TextInputFormat等，类图如下：

![InputFormat类层次图](/images/inputformat-class.png)

需要注意的是FileInputFormat设置输入路径的时候，如果包含子路径默认是**当做文件处理的**   
如果需要递归的读取子目录文件，那么需要设置**mapreduce.input.fileinputformat.input.dir.recursive**属性为true

### FileInputFormat的输入分片

FileInputFormat的getSplit方法是如何将数据切分成一个个分片呢？   
FileInputFormat只切分大文件，大文件是指文件超过HDFS块的大小，因为通常分片大小是和HDFS块大小相同的   

但是输入分片的大小是可以通过属性来控制的，如下：

|属性名|类型|默认值|描述|
|---|---|---|---|
|mapreduce.input.fileinputformat.split.minsize|int|1(字节)|Input Split的最小值|
|mapreduce.input.fileinputformat.split.maxsize|long|Long.MAX_VALUE|Input Split的最大值|
|dfs.blocksize|long|128M|HDFS块大小|

通过这三个属性来控制输入分片的大小的**同时也可以控制作业的map任务数**，定义分片大小的时候要注意：   
太大，会导致map读取的数据可能跨越不同的节点，没有了数据本地化的优势   
太小，会导致map数量过多，任务启动和切换开销太大，并行度过高

那么分片的大小是如何从这三个属性中得到的呢？计算公式如下：   
max(minimumSize,min(maximumSzie,blockSize))   
首先从最大分片大小和block大小之间选出一个比较小的，再和最小分片大小相比选出一个较大的   
由于默认情况下，**minimumSize小于blockSize小于maximumSzie，所以分片的默认大小和blockSize一致**

### 小文件与CombineFileInputFormat

在实例场景中介绍CombineFileInputFormat：[自定义分片策略解决大量小文件问题](http://www.xiaohei.info/2016/03/01/mapreduce-custom-split/)

### 其他InputFormat的子类们

Hadoop提供了各个场景下使用的InputFormat，具体介绍可以参考Hadoop权威指南或者网上的资料~

## 输出格式

和前一节讲述的输入格式一样，Hadoop也有一套输出格式，如下图：

![OutputFomat类层次图](/images/outputformat-class.jpg)

### 文本输出

TextFileOutputFormat是默认的输出格式，其键值可以是任意的类型   
将会调用键值的toString方法，并以制表符分割，键值之间的分隔符可以通过**mapreduce.output.textoutputformat.separator**属性来设置   

### 二进制输出

可以将数据内容以SequenceFile或者MapFile的形式存储在HDFS中

### 多个输出

默认情况下，一个reducer会以特定的文件名产生一个文件，这个动作是自动完成的   
有时候我们可能需要对输出文件的文件名和数量进行控制   
所以Hadoop为我们提供了MultipleOutputs类

MultipleOutputs类可以在一个mapper或者reducer中生成多个文件   
文件名的格式为name-m-nnnnn或者name-r-nnnnn，分别代表mapper和reducer的输出   
其中name是程序中指定的字符串，nnnnn是一个指明的快号整数（从0开始）

MultipleOutputs的使用示例如下：

```java
static class MultipleOutputReducer{
	//MultipleOutputs类的实例
	private MultipleOutputs<NullWritable,Text> multipleOutput;

	@Override
	protected void setup(Context context) throw IOException,InterruptedException{
		//在setup方法中通过context进行初始化
		multipleOutput = new MultipleOutputs<NullWritable,Text>(context);
	}

	@Override
	protected void reduce(Text key,Iterable<Text> values,Context context)IOException,InterruptedException{
		for(Text value:values){
			//使用multipleOutput代替context调用write方法
			//参数分别为：key，value，文件名中的name
			multipleOutput.write(NullWritable.get(),value,key.toString());
		}
	}

	@Override
	protected void cleanup(Context context){
		//在cleanup方法中关闭该对象
		multipleOutput.close();
	}
}
```

如此一来，key相同的数据会被输出到同一个文件中，并且以该key的值作为文件名的开头

在这个过程中，我们设置可以设置多层的输出路径：

```java
@Override
protected void reduce(Text key,Iterable<Text> values,Context context)IOException,InterruptedException{
	for(Text value:values){
		//getYear和getMonth为伪代码，表示获取当前年份和月份，输出的时候将按照年月的路径输出
		String basePath = String.format("%s/%s/part",getYear(),getMonth())
		//使用multipleOutput代替context调用write方法
		//参数分别为：key，value，文件名中的name
		multipleOutput.write(NullWritable.get(),value,basePath);
	}
}
```

### 延迟输出

FileOutputFormat即使没有数据也会产生一个空文件，有时候我们并不想这样子   
这时候就可以使用LazyOutputFormat，该类只有在第一条数据真正输出的时候才会创建文件


作者：[@小黑](http://www.xiaohei.info)
