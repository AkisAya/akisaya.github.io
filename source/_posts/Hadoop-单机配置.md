---
title: Hadoop 单机配置
date: 2017-12-02 21:51:01
updated: 2017-12-02 21:51:01
categories: 大数据
tags:
 - Hadoop
---


## 1 前置安装程序
### 1.1 安装 JDK
参考 [安装 JDK 的两种方式](https://www.cnblogs.com/a2211009/p/4265225.html)
### 1.2 安装 SSH
在 Ubuntu 环境下，首先安装 SSH
<!-- more -->
```
$ sudo apt-get install ssh
```
然后配置免密码登录本机节点
```
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keyss
  $ ssh localhost
```

## 2 安装配置 Hadoop
在官网下载 Hadoop 安装包并解压，本文以 Hadoop-2.8.2 为例，Hadoop 从 1.x 到 2.x 后解压后的包结构有所变化，1.x 中 `conf` 文件夹在 2.x 中已不在，在 2.x 中配置文件存放在了 `etc/hadoop` 中。


为了后续方便调用 Hadoop 的命令，添加相应的环境变量，在 `~/.bashrc` 中最后添加上
```
export HADOOP_HOME=~/program/hadoop2   # hadoop 的解压后存放路径
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

在单节点的 Hadoop 配置中包含默认**单机模式(Standalone Operation)**和**伪分布式模式(Pseudo-Distributed Operation)**

### 2.1 单机模式 - Standalone Operation
hadoop 压缩包解压后无需任何配置，该方式可以用来调试
hadoop 安装包中自带了 example 可以用来测试，在 2.x 版本中该示例存放在 `hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.x.x.jar`
在单机模式下，没有使用到 hadoop 的 hdfs 系统，输入数据和输出数据都是存放在本机上，可以直接用本机上的数据进行测试。
```
$ mkdir input
$ echo "hello world hello python hello java bye java bye bye" > file
$ mv file input 
$ hadoop jar  program/hadoop2/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.2.jar wordcount input output
```
output 是输出文件夹，不要提前自行该创建该文件夹。该文件夹中包含了输出结果和成功信息，如下所示：
```
$ ls output
part-r-00000  _SUCCESS
akis@ubuntu:~$ cat output/part-r-00000 
bye    3
hello    3
java    2
python    1
world    1
```

### 2.2 伪分布式模式 - Pseudo-Distributed Operation
伪分布式模式用到了 hadoop 的 hdfs，输入输出文件由 hdfs 管理，相比单机模式更接近生产环境。伪分布式模式可以看做只有一个节点的集群，配置过程需要修改 `etc/hadoop` 中的一些文件。

在 `hadoop-evn.sh` 中配置 JAVA_HOME:
```
# The java implementation to use.
export JAVA_HOME=/opt/jdk8  # 此处需要使用绝对路径
```

修改 `core-site.xml`，此处配置的是 HDFS 的地址及端口号：
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

修改 `hdfs-site.xml`，此处配置备份数量，默认是 3，修改为 1
```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

此时已经可以运行 Hadoop 了.
首先初始化系统，然后启动 NameNode 和 DataNode:
```
$ hdfs namenode -format
$ start-dfs.sh
```

然后可以运行 `jps` 查看运行的进程，有如下 `NameNode`, `DataNode`, `SecondaryNameNode` 则运行成功。
```
akis@ubuntu:~$ jps
12241 NameNode
14070 Jps
12589 SecondaryNameNode
12366 DataNode
```

此时输入输出的文件都由 hdfs 来管理，使用 `FileSystem Shell` 中的 `put` 可以将本地文件上传到 hdfs 中，然后可运行示例程序。
```
$ hadoop fs -mkidr -p /wordcount/input    # 在 hdfs 中创建文件夹
$ hadoop fs -put input/* /wordcount/input   # 将本地的 input 文件夹内容拷贝至 hdfs 中
$ hadoop jar  program/hadoop2/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.2.jar wordcount /wordcount/input /wordcount/output
$ hadoop fs -cat /wordcount/output/part-r-00000 # 运行成功后 output 文件夹中文件同上文的单机模式时一致，使用 cat 查看如下
bye    3
hello    3
java    2
python    1
world    1
```

如果要使程序运行在 yarn 上，需要再配置。
修改 `mapred-site.xml`:
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

同时修改 `yarn-site.xml`:
```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

启动 yarn：
```
$ start-yarn.sh
```

如果要停止服务：
```
$stop-yarn.sh
$stop-dfs.sh
```

在单机模式、伪分布模式和基于 yarn 的伪分布模式三种模式中，用小数据量测试时运行速度依次变慢，三种模式耗费的硬件资源依次增多。在笔者的测试中，在虚拟机中配置好后，yarn 模式运行很慢。**如果出现了再 yarn 模式下运行出错的情况，可以尝试增大虚拟机内存，笔者虚拟机内存在 2G 时，运行自带的 wordcount 示例也会出现 connection refused 和 namenode 断掉的情况，将内存增加至 4G 后问题解决了!**

## 3 可能出现的问题
如果遇到 namenode 无法启动的情况，可能是默认的缓存文件夹的问题，则此时在 `core-site.xml` 更改缓存的位置后重新格式化即可
```
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/akis/hadoop_tmp</value>  <!-- 指定本机上的一个目录, 默认目录在本机的 /tmp 下-->
    <description>A base for other temporary directories.</description>
</property>
```

如果遇到端口占用无法启动则杀掉该端口或者换端口即可。如果控制台没有输出错误信息，可以查看 hadoop 安装路径下的 log 文件夹中查看对应的 log 信息。

## 4 Reference
- [Setting up a Single Node Cluster](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html)
- [解决Hadoop namenode无法启动以及修改hdfs的存放位置](http://blog.csdn.net/scgaliguodong123_/article/details/44498173)
- [hadoop启动遇到的各种问题](https://segmentfault.com/a/1190000006838239)
