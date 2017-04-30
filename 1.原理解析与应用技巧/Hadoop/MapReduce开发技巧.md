## 数据类型的选择

### 自定义数据类型

参考：[Hadoop提交作业自定义排序和分组](http://www.xiaohei.info/2015/02/15/mapreduce-custom-sort-group/)

### MapWritable/SortedMapWritable

Hadoop中可传输的Map集合，和Java中的Map用法差不多，但是可以用与mapper和reducer之间的数据传输

### Map输出不同类型的Value

使用自定义的数据类型继承自GenericWritable可以实现在mapper中输出多个不同类型的value

```java
//使用这个数据类型将可以输出IntWritable和Text两种类型的value
public class MultiValueWritable extends GenericWritable{
	private static Class[] CLASSES = new Class{
		IntWritable.class,
		Text.class
	}
	
	public MultiValueWritable(){
	}

	public MultiValueWritable(Writable value){
		set(value);
	}

	protected Class[] getTypes(){
		return CLASSES;
	}
}
```

mapper中context.write的时候可以使用如下的格式：

```java
context.write(key,new MultiValueWritable(new Text("1")));
context.write(key,new MultiValueWritable(IntWritable Text(1)));
```

reducer的Values迭代器中可以通过这种方式来判断value是那种数据类型：

```java
Writable value = value.get();
if(value instanceof Text){
	...
}
```

## 选择合适的InputFormat/OutputFormat

基本上每个InputFormat都会有一个对应的OutputFormat

### TextInputFormat

默认的输入格式，按行读取，key为每行偏移量，value为行的内容

### NLineInputFormat

可以指定一次数据文件多少行的内容：   
```java
//设置一次读取50行的内容
NLineInputFormat.setNumLinesPerSplit(job,50);
```

### SequenceFileInputFormat

输入的格式为keylen,key,valuelen,value，适合用于多个job之间的数据连接

### DBInputFormat

处理数据库输入，待使用测试

### 自定义的InputFormat

参考：[自定义分片策略解决大量小文件问题](http://www.xiaohei.info/2016/03/01/mapreduce-custom-split/)

## 同时处理不同类型的输入

参考：[多个Mapper和Reducer处理多个输入](http://www.xiaohei.info/2016/02/22/mapreduce-multi-input-mapper-reducer/)

## Partitioner的选择

### TotalOrderPartitioner

对所有reducer中的结果进行排序，默认情况下每个reducer中的内容都是各自排序互不影响的

### 自定义partitioner

参考：[Hadoop作业中自定义分区和归约](http://www.xiaohei.info/2015/02/15/mapreduce-custom-partitioner-combiner/)

### KeyFieldBasedPartitioner

在分区的时候mapper的key部分会参与计算   
配合参数   
```
map.output.key.field.separator
num.key.fields.for.partition
```
指定分隔符和要参与分区的字符索引

例如：key="name-price"，指定map.output.key.field.separator="-",num.key.fields.for.partition=1表示key的price部分参与分区计算

## 二次排序

### setSortComparatorClass

map中每个分区调用进行排序，reduce中shuffle之后再次调用

### setGroupingComparatorClass

第二次排序，属于同一组的顺序记录并放入同一个value迭代器

## 分布式缓存的使用

参考：[MapReduce中的DistributedCache](http://www.xiaohei.info/2016/02/26/mapreduce-distributed-cache/)

作者：[@小黑](http://www.xiaohei.info)


