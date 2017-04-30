当集群数量很大时，修改配置文件和节点之间的文件同步是一件很麻烦且浪费时间的事情。

rsync是linux上实现不同机器之间文件同步、备份的工具，centos系统中默认已经安装，使用

```
rsync -h
```

检查是否已经安装rsync。

### **使用前提**

确保各个节点部署的目录结构是一致的，不然同步起来很麻烦。

## **使用过程**

在网上找到一大堆rsync的配置资料，然而使用起来不尽人意，对于初次使用rsync的人来说，各种配置显然太过复杂，需要一步步来熟悉。

所以这里不会对rsync的配置文件进行任何修改，仅仅使用rsync的命令进行同步操作。

**需求**

需要同步各个节点上的hadoop、hbase和spark的配置文件，其余目录/文件不需要同步。

**exclude文件**

在部署hadoop等父目录下，新建一个rsync-exclude.list文件，内容为不需要同步的目录/文件，每个目录/文件为一行

**每行的内容不要留有空格或者制表符等，不然会导致无效！**

在该目录下执行rsync命令：

```
sudo rsync -avz --exclude-from rsync-exclude.list ${需要同步的目录} ${目标节点ip/host}:${目标节点的目录}
```

-avz 表示传输过程中递归同步、显示详情并使用压缩。
--exclude-from 的参数为exclude文件目录。

**注意，exclude文件中要使用相对路径，测试时使用绝对路径失败。**

## **脚本实现多服务器同步**

只需要一个简单的for循环即可实现：

```
#!/bin/bash
hosts=(host1 host2 host3 ...)
for host in ${hosts[@]}
do
sudo rsync -avz --exclude-from rsync-exclude.list ${需要同步的目录} $host:${目标节点的目录}
done
```