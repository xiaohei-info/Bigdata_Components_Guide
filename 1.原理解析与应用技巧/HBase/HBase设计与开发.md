### **适合HBase应用的场景**

> * **成熟的数据分析主题，查询模式已经确定且不会轻易改变。**
> * 传统数据库无法承受负载。
> * 简单的查询模式。

### **基本概念**

**行健：**是hbase表自带的，每个行健对应一条数据。
**列族：**是创建表时指定的，为列的集合，每个列族作为一个文件单独存储，存储的数据都是字节数组，其中的数据可以有很多，通过时间戳来区分。
**物理模型：**整个hbase表会拆分为多个region，每个region记录着行健的起始点保存在不同的节点上，查询时就是对各个节点的并行查询，当region很大时使用.META表存储各个region的起始点，-ROOT又可以存储.META的起始点。

**rowkey的设计原则：**长度原则、相邻原则，**各个列簇数据平衡**，创建表的时候设置表放入regionserver缓存中，**避免自动增长和时间**，使用字节数组代替string，最大长度64kb，最好16字节以内。
**列族的设计原则：**尽可能少（按照列族进行存储，按照region进行读取，不必要的io操作），经常和不经常使用的两类数据放入不同列族中，列族名字尽可能短，注意列族的数量级，**避免数据倾斜**。

rowkey设计可以参考：http://blog.chedushi.com/archives/9720

### **HBase开发**

**写入HBase**

maper程序不需要改动，还是继承Mapper类读取文件。

```
//reducer要extends Tablereducer，使用方式和reducer一样，泛型的最后一个参数是输出的key类型，value类型固定为Put
public class ExtractReducer extends TableReducer<Text, Text, ImmutableBytesWritable> {
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) {
    	//Put对象new的时候参数为Bytes.toBytes(行健的值)，一个put就是一行
        Put put = new Put(key.getBytes());
        for (Text v : values) {
            String[] vs = v.toString().split("\t");
            for (int i = 1; i < vs.length; i++) {
            	//put.add(Bytes.toBytes(列簇),Bytes.toBytes(列),Bytes.toBytes(值));
                put.add(Bytes.toBytes("trade"), Bytes.toBytes(""), Bytes.toBytes(vs[i]));
            }
        }
        try {
            context.write(new ImmutableBytesWritable(key.getBytes()), put);
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

提交mr程序的main方法和普通的mr程序差不多，只是conf中要多加几个hbase的设置：
```
conf.set("hbase.rootdir","/hbase");
conf.set("hbase.zookeeper.quorum","zookeeper地址");
conf.set("dfs.socket.timeout", "18000");
```

main程序示例：

```
public class ExtractDriver extends BaseDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Job job = new Job(config, "Extract");
        job.setJarByClass(ExtractDriver.class);
        FileInputFormat.setInputPaths(job, "/oracle/TYZXJS/T_CUSTOMER_AGREEMENT");
        job.setInputFormatClass(TextInputFormat.class);

        job.setMapperClass(ExtractMapper.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(Text.class);
        //设置reduce类和操作的hbase表名
        TableMapReduceUtil.initTableReducerJob("CUSTOMER", ExtractReducer.class, job);

        job.waitForCompletion(true);
    }
}

```



**读取hbase数据**

map程序示例：

```
//需要使用到TableMapper这个类
 public static class MyMapper extends TableMapper<Text, IntWritable>  {  
  
        private final IntWritable ONE = new IntWritable(1);  
        private Text text = new Text();  
  
        public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {  
            String ip = Bytes.toString(row.get()).split("-")[0];  
            //列族，列名
            String url = new String(value.getValue(Bytes.toBytes("info"), Bytes.toBytes("url")));  
            text.set(ip+"&"+url);  
            context.write(text, ONE);  
        }  
    }  
```

main函数示例：

```
 public static void main(String[] args) {  
        try{  
            Configuration config = HBaseConfiguration.create();  
            Job job = new Job(config,"ExampleSummary");  
            job.setJarByClass(MapReduce.class);
  			//设置scan读取hbase
            Scan scan = new Scan();  
            scan.setCaching(500);
            scan.setCacheBlocks(false);
            TableMapReduceUtil.initTableMapperJob(  
                    "access-log",        // 读取的表
                    scan,               // scan对象
                    MyMapper.class,     // mapper类
                    Text.class,         // mapper输出key类型
                    IntWritable.class,  // mapper输出value类型
                    job);
            job.setNumReduceTasks(1);

            //reduce代码和设置不变

            job.waitForCompletion(true);  
        } catch(Exception e){  
            e.printStackTrace();  
        }  
    }  
```

**读写hbase示例**

此处代码来自[CSDN][2]

```
public class ExampleTotalMapReduce{  
    public static void main(String[] args) {  
        try{  
            Configuration config = HBaseConfiguration.create();  
            Job job = new Job(config,"ExampleSummary");  
            job.setJarByClass(ExampleTotalMapReduce.class);     // class that contains mapper and reducer  
  
            Scan scan = new Scan();  
            scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs  
            scan.setCacheBlocks(false);  // don't set to true for MR jobs  
            // set other scan attrs  
            //scan.addColumn(family, qualifier);  
            TableMapReduceUtil.initTableMapperJob(  
                    "access-log",        // input table  
                    scan,               // Scan instance to control CF and attribute selection  
                    MyMapper.class,     // mapper class  
                    Text.class,         // mapper output key  
                    IntWritable.class,  // mapper output value  
                    job);  
            TableMapReduceUtil.initTableReducerJob(  
                    "total-access",        // output table  
                    MyTableReducer.class,    // reducer class  
                    job);  
            job.setNumReduceTasks(1);   // at least one, adjust as required  
  
            boolean b = job.waitForCompletion(true);  
            if (!b) {  
                throw new IOException("error with job!");  
            }   
        } catch(Exception e){  
            e.printStackTrace();  
        }  
    }  
  
    public static class MyMapper extends TableMapper<Text, IntWritable>  {  
  
        private final IntWritable ONE = new IntWritable(1);  
        private Text text = new Text();  
  
        public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {  
            String ip = Bytes.toString(row.get()).split("-")[0];  
            String url = new String(value.getValue(Bytes.toBytes("info"), Bytes.toBytes("url")));  
            text.set(ip+"&"+url);  
            context.write(text, ONE);  
        }  
    }  
  
    public static class MyTableReducer extends TableReducer<Text, IntWritable, ImmutableBytesWritable>  {  
        public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {  
            int sum = 0;  
            for (IntWritable val : values) {  
                sum += val.get();  
            }  
  
            Put put = new Put(key.getBytes());  
            put.add(Bytes.toBytes("info"), Bytes.toBytes("count"), Bytes.toBytes(String.valueOf(sum)));  
  
            context.write(null, put);  
        }  
    }  
}  
```

### **操作HBase过程中遇到的异常信息**

**1、hbase测试程序运行失败，情况一。**
     **症状：**提示zookeeper无法连接。
     **问题所在：**zk地址错误。
     **解决方法：**使用正确的zk地址。

**2、hbase测试程序运行失败，情况二**
     **症状：**java程序访问hbase 卡住不动 不报异常。
     **问题插排：**使用shell创建表失败，转异常3。
     **解决方式：**异常3解决之后即可正常运行。

**3、使用hbase shell创建表失败。**
     **症状：**提示ERROR: java.io.IOException: Table Namespace Manager not ready yet, try again later。
     **问题插排：**hbase集群没有启动，转异常4。
     **解决方式：**异常4解决之后正常启动集群即可。
     
**4、hbase集群无法正常启动。**
     **症状：**hbase无法正常启动，子节点进程自动退出。
     **问题插排：**查看节点日志发现异常，集群时间没有同步，待解决。
     **日志如下：** org.apache.hadoop.hbase.ClockOutOfSyncException: org.apache.hadoop.hbase.ClockOutOfSyncException: Server zx-hadoop-210-27,60020,1451550407914 has been rejected; Reported time is too far out of sync with master.  Time difference of 158391ms > max allowed of 30000ms
     **解决方式：**同步集群时间。

作者：[@小黑][1]

[1]:http://www.xiaohei.info
[2]:http://blog.csdn.net/woshiwanxin102213/article/details/17914083