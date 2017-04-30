对于复杂的mr任务来说，只有一个map和reduce往往是不能够满足任务需求的，有可能是需要n个map之后进行reduce，reduce之后又要进行m个map。

在hadoop的mr编程中可以使用ChainMapper和ChainReducer来实现链式的Map-Reduce任务。

### ChainMapper

以下为官方API文档翻译：   
ChainMapper类允许在单一的Map任务中使用多个Mapper来执行任务，这些Mapper将会以链式或者管道的形式来调用。   
第一个Mapper的输出即为第二个Mapper的输入，以此类推，直到最后一个Mapper则为任务的输出。   
这个特性的关键功能在于，在链中的Mappers不必知道他们是否已经被执行，这可以在一个单一的任务中让一些Mapper进行重用，组合在一起完成复杂的操作。   
使用的时候需要注意，每个Mapper的输出都会在下一个Mapper的输入中进行验证，这里假设所有的Mapper和Reduce都使用相匹配的key和value作为输入和输出，因为在链式执行的代码中并没有对其进行转换。   
使用ChainMapper和ChainReducer可以将Map-Reduce任务组合成[MAP+ / REDUCE MAP*]的形式，这个模式最直接的好处就是可以大大减少磁盘的IO开销。   
**注意：**没有必要为ChainMapper指定输出的key和value的类型，使用addMapper方法添加最后一个Mapper的时候回自动完成。   

**使用的格式：**

```
Job = new Job(conf);
//mapA的配置，如果不是特殊配置可传入null或者共用一个conf
Configuration mapAConf = new Configuration(false);
//将Mapper加入执行链中
ChainMapper.addMapper(job, AMap.class, LongWritable.class, Text.class,
   Text.class, Text.class, true, mapAConf);
Configuration mapBConf = new Configuration(false);
ChainMapper.addMapper(job, BMap.class, Text.class, Text.class,
   LongWritable.class, Text.class, false, mapBConf);

 job.waitForComplettion(true);
```

addMapper函数的定义如下：

```
public static void addMapper(Job job,
             Class<? extends Mapper> klass,
             Class<?> inputKeyClass,
             Class<?> inputValueClass,
             Class<?> outputKeyClass,
             Class<?> outputValueClass,
             Configuration mapperConf)
                      throws IOException
```

### ChainReducer

基本描述同ChainMapper。   
对于每条reduce输出的数据，Mappers将会以链或者管道的形式调用。    ？
ChainReducer有两个基本函数可以调用，使用格式：

```
Job = new Job(conf);
Configuration reduceConf = new Configuration(false);
//这里是在setReducer之后才调用addMapper
ChainReducer.setReducer(job, XReduce.class, LongWritable.class, Text.class,
   Text.class, Text.class, true, reduceConf);
ChainReducer.addMapper(job, CMap.class, Text.class, Text.class,
   LongWritable.class, Text.class, false, null);
ChainReducer.addMapper(job, DMap.class, LongWritable.class, Text.class,
   LongWritable.class, LongWritable.class, true, null);
job.waitForCompletion(true);
```

setReducer定义：

```
public static void setReducer(Job job,
              Class<? extends Reducer> klass,
              Class<?> inputKeyClass,
              Class<?> inputValueClass,
              Class<?> outputKeyClass,
              Class<?> outputValueClass,
              Configuration reducerConf)
```

addMapper定义同ChainMapper。   

## 实际的测试过程

在demo程序测试中观察结果得到两条比较有用的结论：

> 1. 对于reduce之后添加的Mapper，每条reduce的输出都会马上调用一次该map函数，而不是等待reduce全部完成之后再调用map，如果是有多个map，应该是一样的道理。   
> 2. reduce之后的Mapper只执行map过程，并不会有一个完整map阶段（如，map之后的排序，分组，分区等等都没有了）。

**另注：**reduce之前设置多个Mapper使用ChainMapper的addMapper，reduce之后设置多个Mapper使用ChainReducer的addMapper。   

## 多个job连续运行

有时候链式的设置多个Mapper仍然无法满足需求，例如，有时候我们需要多个reduce过程，或者map之后的分组排序等，这就需要多个job协同进行工作。   
使用的方法很简单，直接在第一个job.waitForCompletion之后再次实例化一个Job对象，按照八股文的格式进行设置即可，注意输入和输出的路径信息。

example：

```
Job newJob = Job.getInstance(conf, jobName + "-sort");
        newJob.setJarByClass(jarClass);

        FileInputFormat.setInputPaths(newJob, new Path(outPath + "/part-*"));
        newJob.setInputFormatClass(TextInputFormat.class);

        newJob.setMapperClass(SortMapper.class);
        newJob.setMapOutputKeyClass(SortKey.class);
        newJob.setMapOutputValueClass(NullWritable.class);

        FileOutputFormat.setOutputPath(newJob, new Path(outPath + "/sort"));
        newJob.setOutputFormatClass(TextOutputFormat.class);

        newJob.waitForCompletion(true);
```