## HDFS中的数据完整性

HDFSZ在写入数据的时候会**计算数据的校验和**，针对每个由**dfs.bytes.per.checksum**指定字节的数据计算校验和，默认为512个字节   
当客户端读取数据的时候，会对数据的**校验和**进行检查，如果发现数据出现损坏，则会执行以下步骤：

> 1.向Namenode报告其正在读取的数据块和所在的Datanode，之后会抛出ChecksumException异常   
> 2.Namenode会将高数据块标记为损坏，让其不再处理请求，或者将该数据块复制到其他节点上   
> 3.Namenode安排该数据块的其他完整的副本复制一份到其他完好的节点上，如此系统中的副本数恢复到期望值

在使用FileSystem的open方法之前，可以通过**setVerifyChecksum(false)**方法将校验过程停用

## 压缩

在Hadoop中使用压缩可以带来许多好处，例如：**减少存储空间和降低网络传输的消耗**   
在选择压缩算法的时候通常要权衡时间和空间之间的平衡度，例如，压缩速度越快的往往节约的空间会比较少

### Hadoop中压缩API的使用

如果需要对输出的数据进行压缩，可以通过createOutputStream(OutputStream out)方法来获得一个**CompressionOutputStream**   
反之，也可以通过createInputStream(InputStream in)方法来创建一个**CompressionInputStream**

#### 压缩数据

使用示例：

```java
public class StreamCompressor{
	public static void main(String[] args) throw Exception{
		String codecClassName = args[0];
		//获得压缩类的全名称
		Class<?> codecClass = Class.forName(codecClassName);
		Configuration conf = new Configuration();
		//通过ReflectionUtil来创建一个CompressionCodec实例
		CompressionCodec codec = (CompressionCodec)ReflectionUtil.newInstance(codecClass,conf);
		//由CompressionOutputStream对System.out进行包装，对数据进行压缩
		CompressionOutputStream out = codec.createOutputStream(System.out);
		IOUtils.copyBytes(System.in,out,4096,false);
		out.finish();
	}
}
```

进行测试：

```shell
echo "hello" | hadoop StreamCompressor org.apache.hadoop.io.compress.GzipCodec | gunzip
# 将会输出
hello
```

#### 解压缩数据

使用示例：

```java
public class FileDecompressor{
	public static void main(String[] args) throw Exception{
		String uri = args[0];
		Configuration conf = new Configuration();
		FileSystem fs = FileSystem.get(URI.create(uri),conf);
		Path inputPath = new Path(uri);
		//CompressionCodecFactory根据文件后缀名来获取对应的codec
		CompressionCodecFactory factory = new CompressionCodecFactory(conf);
		CompressionCodec codec = factory.getCodec(inputPath);
		if(codec == null){
			System.out.println("No codec found for " + uri);
			System.exit(1);
		}
		//将压缩的后缀去除，形成普通的文件名
		String outputUri = CompressionCodecFactory.removeSuffix(uri,codec.getDefaultExtension);
		InputStream in = null;
		OutputSteam out = null;
		try{
			in = codec.createInputStream(fs.open(inputPath));
			out = fs.create(new Path(outputUri));
			IOUtils.copyBytes(in,out,conf);
		}finally{
			IOUtils.closeStream(in);
			IOUtils.closeStream(out);
		}
	}
}
```

进行测试：

```shell
hadoop FileDecompressior file.gz
```

### CodecPool

压缩格式的实现有**原生类库**和**Java实现的类库**两种，在gzip格式中，使用原生类库会比使用Java类库提高很多效率（10%的压缩时间和50%的解压缩时间）   
所以最好使用原生类库进行操作（注意，并非所有压缩格式都有原生类库）   
可以通过设置**java.library.path**属性指定原生代码库路径（可以在应用中单独设置，或者在bin目录下的脚本中设置）   
Hadoop中自带了32和64位Linux构架的原生代码库，位于**$HADOOP_HOME/lib/native**目录   
默认情况下Hadoop运行的时候会自动搜索原生代码库路径，如果不需要，可以通过将**hadoop.natice.lib**设置为false

如果使用了原生代码库，并且在应用中需要反复压缩和解压缩，可以通过CodecPool来减少创建这些对象的开销

使用示例：

```java
public class PooledStreamCompressor{
	public static void main(String[] args) throw Exception{
		String codecClassName = args[0];
		//获得压缩类的全名称
		Class<?> codecClass = Class.forName(codecClassName);
		Configuration conf = new Configuration();
		//通过ReflectionUtil来创建一个CompressionCodec实例
		CompressionCodec codec = (CompressionCodec)ReflectionUtil.newInstance(codecClass,conf);
		Compressor compressor = null;
		try{
			compressor = CodecPool.getCompressor(codec);
			//CompressionOutputStream通过CodecPool中获得，而不是重新创建一个
			CompressionOutputStream out = codec.createOutputStream(System.out,compressor);
			IOUtils.copyBytes(System.in,out,4096,false);
			out.finish();
		}finally{
			CodecPool.returnCompressor(compressor);
		}
	}
}
```

### 选择合适的压缩格式

HDFS中，数据被划分为多个分片进行存储，由于在MapReduce中，每个分片有一个map任务来处理   
如果这些数据使用了不支持切分的压缩格式，那么**整个数据文件将会由一个map来处理，失去了数据本地化的特性**   
因为在不支持切分的压缩文件中，**无法从任意的数据流中读取数据**   
所以在对大数据量的文件进行压缩的时候**要选择支持切分的压缩格式**

下面是一些建议，效率由高到低进行排序：

> * 使用容器文件格式，例如sequence file,RCFile或者Avro数据文件，这些文件格式同时支持**压缩和切分**，所以可以选择快速压缩工具，如LZO,LZ4或者Snappy   
> * 使用**支持切分的压缩格式**，如bzip2（尽管其很慢），或者通过索引来实现切分的LZO   
> * 在应用中手动将文件切片，每个数据块单独进行压缩，**确保压缩之后的数据块大小和HDFS块大小相当**   
> * 直接使用未压缩的文件

### 在MapReduce中使用压缩

如果能够根据文件的后缀名推断出使用的压缩格式，MapReduce会在输入的时候自动对数据进行解压缩

要对MapReduce作业的输出进行压缩可以使用两种方式：

> 1.在应用中设置**mapreduce.output.fileoutputformat.compress**属性为true、设置**mapreduce.output.fileoutputformat.codec**为压缩格式的全名   
> 2.在应用中通过**FileOutputFormat.setCompressOutput(job,true)、FileOutputFormat.setOutputCompressorClass(job,GzipCodec.class)**进行设置

此外，在map任务产生的中间数据输出到磁盘的时候使用压缩也可以带来很大的效率提升，仍然可以通过两种方式进行设置

|属性名|类型|默认值|
|---|---|---|
|mapreduce.output.map.output.compress|boolean|false|
|mapreduce.output.map.output.codec|CLass|

```java
conf.setBoolean(Job.MAP_OUTPUT_COMPRESS,true);
conf.setClass(Job.MAP_OUTPUT_COMPRESS_CODEC,GzipCOdec.class,CompressionCodec.class);
```

## 序列化

序列化是指将**结构化对象转为字节流数组**以便在网络上进行传输或者永久存储的行为，反序列化则是序列化的一个逆过程   
序列化常被用在进程间的通讯和永久存储，Hadoop系统各个节点之间通过**RPC**进行进程间的通信，交换数据的过程就设计到序列化和反序列化

通常，RPC序列化格式标准包括以下几点：

> * 紧凑：不仅节约磁盘空间，同时减少数据传输所消耗的网络开销   
> * 快速：使序列化和反序列化的开销达到最小   
> * 可扩展：支持新老系统的兼容   
> * 互操作：不同语言的客户端和服务端通信不会产生障碍

### Writable接口

Hadoop中提供了一系列的基本数据类型和Java的数据类型一一对应，不同的是，Hadoop中的数据类型都**实现Writable接口**   
Writable接口提供了两个方法，一个将数据序列化写入二进制流，另一个从二进制流中反序列化读取数据   
该接口使得Hadoop的数据类型能够以高效的序列化方式进行运作

```java
public interface Writable{
	void write(DataOutput out) throw IOException;
	void readFields(DataInput in) throw IOException;
}
```

大部分Hadoop的基本数据类型并不是直接实现Writable接口，而是通过实现一个**WritableComparable接口**来实现，如下图：

![Hadoop数据类型继承图](/images/hadoop-base-datatype.png)

WritableComparable接口的定义如下：

```java
public interface WritableComparable<T> extends Writable,Comparable<T>{
}
```

因为Hadoop中数据除了需要进行序列化，在MapReduce过程中还需要**对Key进行排序**的阶段   
所以需要数据类型有可以比较的方法   
Comparable来自java.lang.Comparable，其**提供了compareTo方法对转换为对象的字节流数据进行比较**

### 自定义Writable数据类型

```java
//要实现自定义的排序规则必须实现WritableComparable接口，泛型参数为类本身  
public class NewK2 implements WritableComparable<NewK2> {  
    
    //代表第一列和第二列的数据  
    Long first;  
    Long second;  
	
	public NewK2() {  
    }  

    public NewK2(long first, long second) {  
        this.first = first;  
        this.second = second;  
    }  
      
    //重写序列化和反序列化方法  
    @Override  
    public void readFields(DataInput in) throws IOException {  
        this.first = in.readLong();  
        this.second = in.readLong();  
    }  

    @Override  
    public void write(DataOutput out) throws IOException {  
        out.writeLong(first);  
        out.writeLong(second);  
    }  

    //当k2进行排序时，会自动调用该方法. 当第一列不同时，升序；当第一列相同时，第二列升序  
    //如果希望降序排列，那么只需要对调this对象和o对象的顺序  
    @Override  
    public int compareTo(NewK2 o) {  
        if(this.first != o.first)  
        {  
        return (int)(this.first - o.first);  
        }  
        else  
        {  
            return (int) (this.second - o.second);  
        }  
    }  
      
    //重写hashCode和equals方法  
    //hashCode会在分区阶段被HashPartitioner调用来确定该数据所属的reduce分区
    @Override  
    public int hashCode() {  
        return this.first.hashCode() + this.second.hashCode();  
    }  

    @Override  
    public boolean equals(Object obj) {  
        if (!(obj instanceof NewK2)) {  
            return false;  
        }  
        NewK2 oK2 = (NewK2) obj;  
        return (this.first == oK2.first) && (this.second == oK2.second);  
    }
    //toString方法会TextOutputFormat输出数据的时候调用
    @Override
    public String toString(){
    	return this.first + '\t' + this.second;
    }
}
```

详细的使用案例请看：[Hadoop提交作业自定义排序和分组](http://www.xiaohei.info/2015/02/15/mapreduce-custom-sort-group/)

但是这个方法有一个缺陷，WritableComparable提供的compareTo方法是针对对象的   
也就是说，要对比两个数据，需要**先进行反序列化成对象之后才能调用compare进行比较**，如果能够直接**在字节流中对数据进行对比**，将会减少反序列化的开销   
Hadoop中提供了一个RawComparator接口，该接口的compare方法可以直接比较字节流中的数据   
而在实际使用中，通常不直接实现RawComparator接口，而是继承RawComparator接口的一个通用实现WritableComparator   
因为它提供了一些很好的默认实现

```java
public class MyComparator implements WritableComparator {  
    //重写两个比较方法  
    //按对象进行比较，规定只要两个NewK2对象的first属性相同就视为相等  
    @Override  
    public int compare(WritableComparable o1, WritableComparable o2) {  
    	if(o1 instanceof NewK2 && o2 instanceof NewK2){
    		return (NewK2)o1.fitst - (NewK2)o2.first;
    	}
        return super.compare(o1,o2);  
    }  
    /** 
     * @param arg0 
     *            表示第一个参与比较的字节数组 
     * @param arg1 
     *            表示第一个参与比较的字节数组的起始位置 
     * @param arg2 
     *            表示第一个参与比较的字节数组的偏移量 
     *  
     * @param arg3 
     *            表示第二个参与比较的字节数组 
     * @param arg4 
     *            表示第二个参与比较的字节数组的起始位置 
     * @param arg5 
     *            表示第二个参与比较的字节数组的偏移量 
     */  
    @Override  
    public int compare(byte[] arg0, int arg1, int arg2, byte[] arg3,  
            int arg4, int arg5) {  
        //其他字节比较方法
        return WritableComparator  
                .compareBytes(arg0, arg1, 8, arg3, arg4, 8);  
    }  
}
```

直接使用RawComparator的情况

```java
public class MyGroupingComparator implements RawComparator<NewK2> {  
    //重写两个比较方法  
    //按对象进行比较，规定只要两个NewK2对象的first属性相同就视为相等  
    @Override  
    public int compare(NewK2 o1, NewK2 o2) {  
        return (int) (o1.first - o2.first);  
    }  

    /** 
     * @param arg0 
     *            表示第一个参与比较的字节数组 
     * @param arg1 
     *            表示第一个参与比较的字节数组的起始位置 
     * @param arg2 
     *            表示第一个参与比较的字节数组的偏移量 
     *  
     * @param arg3 
     *            表示第二个参与比较的字节数组 
     * @param arg4 
     *            表示第二个参与比较的字节数组的起始位置 
     * @param arg5 
     *            表示第二个参与比较的字节数组的偏移量 
     */  
    @Override  
    public int compare(byte[] arg0, int arg1, int arg2, byte[] arg3,  
            int arg4, int arg5) {  
        return WritableComparator  
                .compareBytes(arg0, arg1, 8, arg3, arg4, 8);  
    }
}
```

## 基于文件的数据结构

因为直接将二进制数据的大对象存入单独的文件中不容易扩展，所以Hadoop提供了很多高层次的容器，也就是一些特殊的数据结构

### SequenceFile

SequenceFile提供了一种**二进制的键值对**存储格式，通常有以下的使用场景：

> 1.作为中间mr任务的输出结果：即作为下一个mr任务的输入   
> 2.适当解决小文件的问题：作为一个**容器**，可以将其当做一堆小文件的集合，文件名作为key，内容作为值一起存储在一个大文件中   
> 3.二进制的数据对象存储

SequenceFile的键值类型只要能够进行序列化都可以，通常使用Hadoop的框架中的数据类型（或者自定义的Writable类型）

SequenceFile实际的存储内容也是二进制格式的，所以无法直接查看   
但是Hadoop CLI的-text选项可以自动判别SequenceFile文件进行输出（如果是自定义的Writable类型，会使用到toString方法，类需要在classpath中）

```shell
hadoop fs -text data.seq
```

#### 写入SequenceFile文件

```java
public static void main(String[] args) throw Exception{
	String uri = args[0];
	Configuration conf = new Configuration();
	FileSystem fs = FileSystem.get(URI.create(uri),conf);
	Path inputPath = new Path(uri);
	IntWritable key = new IntWritable();
	Text value = new Text();
	SequenceFile.Writer writer = null;
	try{
		//通过SequenceFile.createWriter得到写入对象
		writer = SequenceFile.createWriter(fs,conf,path,key.getClass(),value.getClass());
		//设置键值内容
		key.set(0);
		value.set("test");
		//调用append方法写入
		writer.append(key,value);
	}finally{
		IOUtils.closeStream(writer);
	}
}
```

#### 读取SequenceFile文件

```java
public static void main(String[] args) throw Exception{
	String uri = args[0];
	Configuration conf = new Configuration();
	FileSystem fs = FileSystem.get(URI.create(uri),conf);
	Path inputPath = new Path(uri);
	SequenceFile.Reader reader = null;
	try{
		//创建一个读取对象
		reader = new SequenceFile.Reader(fs,path,conf);
		//获得键值类型
		Writable key = (Writable)ReflectionUtils.newInstance(reader.getKeyClass().conf);
		Writable value = (Writable)ReflectionUtils.newInstance(reader.getValueClass().conf);
		//获得当前读取数据的位置
		long position = reader.getPosition();
		//如果有记录，返回true，并将数据填入key，value参数中
		while(reader.next(key,value)){
			System.out.println(key + ":" + value + "position in file:" + position);
			position = reader.getPosition();
		}
	}finally{
		IOUtils.closeStream(reader);
	}
}
```

如果使用的是Writable接口的数据类型，则可以直接使用next方法得到下一个键值对的值   
如果不是，则需要使用以下的方法：

```java
public Object next(Object key) throw IOException;
public Object getCurrentValue(Object val) throw IOException;
```

### MapFile

MapFile相当于**加了索引和排过序**的SequenceFile   
和SequenceFile不同的是，MapFile存储在HDFS上是一个**包含两个SequenceFile文件的目录**   
例如写入一个MapFile文件，生成test.map，其是一个目录，包含data和index两个SequenceFile文件：

> * data文件：和之前讨论的单独的SequenceFile完全一致   
> * index文件：也是一个key-value型的SequenceFile

可以看到，MapFile最主要的特点就是多了个index文件，那么这个index文件有什么用呢？   
顾名思义，其是一个充当索引作用的文件，将会被载入内存中以提高数据的访问速度

index文件中，key的内容就是data文件中的key，data文件中每个128个key（默认）**便在index文件中记录一次**   
index文件中该key对应的value**是data文件中，该key对应的value的偏移量**

也就是说，index中存储着data中的**一部分一模一样的key**和**该key对应value的偏移量**   
当index被载入内存中，程序可以根据index的内容**快速定位key在data文件中所在的位置**   
由于index中只记录部分key，所以对于**随机读**来说，可以提高很高的效率

MapFile文件的读写流程和SequenceFile类似，只要将SequenceFile.Writer/Reader替换为**MapFile.Writer/Reader**即可   
如果需要改变默认的每隔128个key记录一次，可以通过SequenceFile.Writer实例的**setIndexInterval()**设置**io.map.index.interval**属性即可