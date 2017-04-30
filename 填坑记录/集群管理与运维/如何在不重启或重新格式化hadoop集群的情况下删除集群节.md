在master节点上的hadoop安装目录下
进入conf目录
配置hdfs-site.xml文件
添加节点如下：

```xml
<property>
<name>dfs.hosts.exclude</name>
<value>home/hadoop/hadoop-0.20.2/conf/excludes</value>
</property>
```

节点的值为excludes文件的路径
该文件的内容为要下架的节点的ip地址或者主机名，一行一个
完成配置之后执行命令
./hadoop dfsadmin -refreshNodes
重新加载配置
成功之后可以将已经移除的节点从excludes文件中删除

作者：[@小黑](http://www.xiaohei.info)