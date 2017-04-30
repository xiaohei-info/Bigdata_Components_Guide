## **简介**

DistributedCache是Hadoop为MapReduce框架提供的一种分布式缓存机制，它会将需要缓存的文件分发到各个执行任务的子节点的机器中，各个节点可以自行读取本地文件系统上的数据进行处理。

## **符号链接**

可以同在原本HDFS文件路径上+"#somename"来设置符号连接（相当于一个快捷方式）    

这样在MapReduce程序中可以直接通通过：

```
File file = new File("somename");
```

来获得这个文件

## **缓存在本地的目录设置**

以下为默认值：
```
<property>
  <name>mapred.local.dir</name>
  <value>${hadoop.tmp.dir}/mapred/localdir/filecache</value>
</property>
 
<property>
  <name>local.cache.size</name>
  <value>10737418240</value> 
</property>
```

## **应用场景**

> 1.分发第三方库(jar,so等)    
> 2.共享一些可以装载进内存的文件    
> 3.进行类似join连接时，小表的分发

## **使用方式**

旧版本的DistributedCache已经被注解为过时，以下为Hadoop-2.2.0以上的新API接口，测试的Hadoop版本为2.7.2。

```
Job job = Job.getInstance(conf);
//将hdfs上的文件加入分布式缓存
job.addCacheFile(new URI("hdfs://url:port/filename#symlink"));
```

由于新版API中已经默认创建符号连接，所以不需要再调用setSymlink(true)方法了，可以通过

```
System.out.println(context.getSymlink());
```

来查看是否开启了创建符号连接。

之后在map/reduce函数中可以通过context来访问到缓存的文件，一般是重写setup方法来进行初始化：

```
@Override
    protected void setup(Context context) throws IOException, InterruptedException {
        super.setup(context);
        if (context.getCacheFiles() != null && context.getCacheFiles().length > 0) {
	    String path = context.getLocalCacheFiles()[0].getName();
        File itermOccurrenceMatrix = new File(path);
        FileReader fileReader = new FileReader(itermOccurrenceMatrix);
        BufferedReader bufferedReader = new BufferedReader(fileReader);
        String s;
        while ((s = bufferedReader.readLine()) != null) {
	        //TODO:读取每行内容进行相关的操作
        }
        bufferedReader.close();
        fileReader.close();
    }
}
```

得到的path为本地文件系统上的路径。

这里的getLocalCacheFiles方法也被注解为过时了，只能使用context.getCacheFiles方法，和getLocalCacheFiles不同的是，getCacheFiles得到的路径是HDFS上的文件路径，如果使用这个方法，那么程序中读取的就不再试缓存在各个节点上的数据了，相当于共同访问HDFS上的同一个文件。    

可以直接通过符号连接来跳过getLocalCacheFiles获得本地的文件。

**单机安装的hadoop没有通过，提示找不到该文件，待在集群上进行测试。**

## **注意事项**

> 1.需要分发的文件必须是存储在HDFS上了    
> 2.文件只读    
> 3.不缓存太大的文件，执行task之前对进行文件的分发，影响task的启动速度    

作者：[@小黑][1]

[1]:http://www.xiaohei.info