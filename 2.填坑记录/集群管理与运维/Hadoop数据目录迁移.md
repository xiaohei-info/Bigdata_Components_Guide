随着数据的不断导入和增大，原本集群部署的目录磁盘空间不足了，所以要把hadoop存储数据的位置迁移到另外一个巨大的磁盘上，另外的一个用意是将数据和程序分离开，以免互相影响。

以下是迁移过程和需要注意的一些地方：

**动手之前先把集群停止，如果有hbase也一起停了，因为hbase的存储是依赖于hdfs的，如果没有停止就进行目录迁移hbase会出现错误。**

### **修改配置文件**

hadoop最重要的存储数据的配置在core-site.xml文件中设置，修改core-site.xml的hadoop.tmp.dir值为新磁盘的路径即可。

考虑到数据和程序的分离，决定将那些会不断增长的文件都迁移出去，包括：日志文件，pid目录，journal目录。

日志文件和pid目录在hadoop-env.sh中配置，export HADOOP\_PID\_DIR，HADOOP\_LOG_DIR为对应磁盘路径即可。

journal目录在hdfs-site.xml中配置dfs.journalnode.edits.dir

同理，yarn和hbase的log和pid文件路径都可在*_env.sh文件中export设置

**改完Hadoop的配置文件之后将其拷贝到hbase/conf目录下**

hbase的日志文件和pid目录配置在hbase-daemon.sh的HBASE\_PID\_DIR，HBASE\_LOG_DIR

spark日志文件的pid目录在spark-env.sh的SPARK\_PID\_DIR，SPARK\_LOG_DIR

修改完之后拷贝配置文件到各个子节点。

并将原始数据目录、日志目录和pid目录移动至新磁盘中，重新启动集群，查看输出信息是否正确。

#**更新**

**hdfs-site.xml**中更新的配置：
```
<property>
<name>dfs.name.dir</name>  
<value>/data2/hadoop/hdfs/name</value>  
</property>
<property>
<name>dfs.data.dir</name>  
<value>/data2/hadoop/hdfs/data</value>  
</property>
```

分别是存储hdfs元数据信息和数据的目录，如果没有配置则默认存储到hadoop.tmp.dir中。

**格式化hdfs系统之后，hbase启动异常，HMaster自动退出。**

日志信息：

```
2016-01-15 14:01:38,231 DEBUG [MASTER_SERVER_OPERATIONS-zx-hadoop-210-11:60000-4] master.DeadServer: Finished processing zx-hadoop-210-24,60020,1452828414814
2016-01-15 14:01:38,231 ERROR [MASTER_SERVER_OPERATIONS-zx-hadoop-210-11:60000-4] executor.EventHandler: Caught throwable while processing event M_SERVER_SHUTDOWN
java.io.IOException: failed log splitting for zx-hadoop-210-24,60020,1452828414814, will retry
        at org.apache.hadoop.hbase.master.handler.ServerShutdownHandler.resubmit(ServerShutdownHandler.java:322)
        at org.apache.hadoop.hbase.master.handler.ServerShutdownHandler.process(ServerShutdownHandler.java:202)
        at org.apache.hadoop.hbase.executor.EventHandler.run(EventHandler.java:128)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.io.IOException: error or interrupted while splitting logs in [hdfs://ns1/hbase/WALs/zx-hadoop-210-24,60020,1452828414814-splitting] Task = installed =
 1 done = 0 error = 0
        at org.apache.hadoop.hbase.master.SplitLogManager.splitLogDistributed(SplitLogManager.java:362)
        at org.apache.hadoop.hbase.master.MasterFileSystem.splitLog(MasterFileSystem.java:410)
        at org.apache.hadoop.hbase.master.MasterFileSystem.splitLog(MasterFileSystem.java:384)
        at org.apache.hadoop.hbase.master.MasterFileSystem.splitLog(MasterFileSystem.java:282)
        at org.apache.hadoop.hbase.master.handler.ServerShutdownHandler.process(ServerShutdownHandler.java:195)
        ... 4 more
2016-01-15 14:01:38,232 INFO  [master:zx-hadoop-210-11:60000-EventThread] zookeeper.ClientCnxn: EventThread shut down
2016-01-15 14:01:38,232 INFO  [master:zx-hadoop-210-11:60000.oldLogCleaner] zookeeper.ZooKeeper: Session: 0x25243ddd648000a closed
2016-01-15 14:01:38,232 DEBUG [MASTER_SERVER_OPERATIONS-zx-hadoop-210-11:60000-4] master.DeadServer: Finished processing zx-hadoop-210-22,60020,1452828414925
2016-01-15 14:01:38,233 ERROR [MASTER_SERVER_OPERATIONS-zx-hadoop-210-11:60000-4] executor.EventHandler: Caught throwable while processing event M_SERVER_SHUTDOWN
java.io.IOException: Server is stopped
        at org.apache.hadoop.hbase.master.handler.ServerShutdownHandler.process(ServerShutdownHandler.java:183)
        at org.apache.hadoop.hbase.executor.EventHandler.run(EventHandler.java:128)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at java.lang.Thread.run(Thread.java:745)
2016-01-15 14:01:38,338 DEBUG [master:zx-hadoop-210-11:60000] catalog.CatalogTracker: Stopping catalog tracker org.apache.hadoop.hbase.catalog.CatalogTracker@6c4b58f0
2016-01-15 14:01:38,338 INFO  [master:zx-hadoop-210-11:60000] client.HConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x15243ddd6340004
2016-01-15 14:01:38,343 INFO  [master:zx-hadoop-210-11:60000] zookeeper.ZooKeeper: Session: 0x15243ddd6340004 closed
2016-01-15 14:01:38,343 INFO  [master:zx-hadoop-210-11:60000-EventThread] zookeeper.ClientCnxn: EventThread shut down
2016-01-15 14:01:38,343 INFO  [zx-hadoop-210-11,60000,1452837685871.splitLogManagerTimeoutMonitor] master.SplitLogManager$TimeoutMonitor: zx-hadoop-210-11,60000,14528
37685871.splitLogManagerTimeoutMonitor exiting
2016-01-15 14:01:38,347 INFO  [master:zx-hadoop-210-11:60000] zookeeper.ZooKeeper: Session: 0x35243ddd73b0001 closed
2016-01-15 14:01:38,347 INFO  [main-EventThread] zookeeper.ClientCnxn: EventThread shut down
2016-01-15 14:01:38,347 INFO  [master:zx-hadoop-210-11:60000] master.HMaster: HMaster main thread exiting
2016-01-15 14:01:38,350 ERROR [main] master.HMasterCommandLine: Master exiting
java.lang.RuntimeException: HMaster Aborted
        at org.apache.hadoop.hbase.master.HMasterCommandLine.startMaster(HMasterCommandLine.java:192)
        at org.apache.hadoop.hbase.master.HMasterCommandLine.run(HMasterCommandLine.java:134)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
        at org.apache.hadoop.hbase.util.ServerCommandLine.doMain(ServerCommandLine.java:126)
        at org.apache.hadoop.hbase.master.HMaster.main(HMaster.java:2785)
Fri Jan 15 14:15:02 CST 2016 Starting master on zx-hadoop-210-11
```

**解决方法**

> * 1.切换到zookeeper的bin目录
> * 2.执行$sh zkCli.sh

```
ls /
rmr /hbase
quit
```

重启hbase。

作者：[@小黑][1]

[1]:http://www.xiaohei.info