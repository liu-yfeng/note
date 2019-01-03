[TOC]

# Hadoop 集群部署

## 一、环境说明

### 1、服务器环境

| 主机名 | IP           | 用户名 | 密码   | 安装目录          | 说明      |
| :----- | :----------- | :----- | :----- | :---------------- | --------- |
| nn01   | 192.168.2.75 | hadoop | hadoop | /usr/local/hadoop | name node |
| nn02   | 192.168.2.76 | hadoop | hadoop | /usr/local/hadoop | name node |
| dn01   | 192.168.2.77 | hadoop | hadoop | /usr/local/hadoop | data node |
| dn02   | 192.168.2.78 | hadoop | hadoop | /usr/local/hadoop | data node |
| dn03   | 192.168.2.79 | hadoop | hadoop | /usr/local/hadoop | data node |

### 2、软件版本

| 软件名称  | 版本号                     |
| --------- | -------------------------- |
| jdk       | jdk-8u191-linux-x64.tar.gz |
| hadoop    | hadoop-3.1.1.tar.gz        |
| zookeeper | zookeeper-3.4.13.tar.gz    |

### 3、集群规划

|                         | nn01        | nn02        | dn01        | dn02        | dn03        |
| ----------------------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| NameNode                | **&radic;** | **&radic;** |             |             |             |
| DataNode                |             |             | **&radic;** | **&radic;** | **&radic;** |
| ResourceManager         | **&radic;** | **&radic;** |             |             |             |
| NodeManager             | **&radic;** | **&radic;** | **&radic;** | **&radic;** | **&radic;** |
| Zookeeper               |             |             | **&radic;** | **&radic;** | **&radic;** |
| JournalNode             | **&radic;** | **&radic;** | **&radic;** | **&radic;** | **&radic;** |
| ZKFC                    | **&radic;** | **&radic;** |             |             |             |
| JobHistory              | **&radic;** | **&radic;** |             |             |             |
| DFSZKFailoverController | **&radic;** | **&radic;** |             |             |             |
| QuorumPeerMain          |             |             | **&radic;** | **&radic;** | **&radic;** |

## 二、环境初始化 (每台机器)

### 1、设置 `/etc/hosts` 文件

可以先在一台机器上改好，再复制到其它机器

```shell
# cat /etc/hosts
192.168.2.75	nn01
192.168.2.76	nn02
192.168.2.77	dn01
192.168.2.78	dn02
192.168.2.79	dn03
```

### 2、关闭防火墙

```shell
# systemctl stop firewalld
# systemctl disable firewalld
```

### 3、禁用 SELinux

```shell
# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
# sed -i 's/SELINUXTYPE=targeted/# SELINUXTYPE=targeted/g' /etc/selinux/config
# setenforce 0
```

### 4、时间同步设置

使用 chrony 进行时间同步

```shell
# yum install chrony

编辑 chrony 配置文件
# vim /etc/chrony.conf
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony

# systemctl start chronyd && systemctl enable chronyd
```

## 三、安装 JDK (每台机器)

### 1、解压 JDK

```shell
[root@nn01 ~]# cd /usr/local/src/
[root@nn01 src]# tar xf jdk-8u191-linux-x64.tar.gz -C /usr/local
[root@nn01 src]# cd ../
[root@nn01 local]# ln -snf jdk1.8.0_191 jdk
```

### 2、设置 java 环境变量

```shell
# vim /etc/profile.d/java.sh
写入如下内容:
# Java Environment
export JAVA_HOME=/usr/local/jdk
export PATH=$JAVA_HOME/bin:$PATH
```

### 3、载入配置文件

```shell
# . /etc/profile.d/java.sh
```

### 4、验证是否安装成功

```shell
# java -version
java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)
```

## 四、设置免密登录 (每台机器)

> 要求能通过免密登录(包括使用 IP 和主机名)都能免密登录：
>
> 1. NameNode 能免密登录所有的 DataNode。
> 2. 各 NameNode 能免密登录自己。
> 3. 各 NameNode 间能免密互登录。
> 4. DataNode 能免密登录自己。
> 5. DataNode 不需要配置免密登录 NameNode 和其它 DataNode。

### 1、创建用于运行 Hadoop 的用户`hadoop`

```shell
# useradd hadoop
# echo 'hadoop' | passwd hadoop --stdin
Changing password for user hadoop.
passwd: all authentication tokens updated successfully.
```

### 2、以 hadoop 用户登录，生成公钥和私钥

```shell
# 生成密钥,输入之后一直选择 enter 即可。生成的秘钥位于 ~/.ssh 文件夹下
$ ssh-keygen -t rsa
$ ssh-copy-id -i nn01
$ ssh-copy-id -i nn02
$ ssh-copy-id -i dn01
$ ssh-copy-id -i dn02
$ ssh-copy-id -i dn03
# nn02 操作同上
# 各 DataNode 操作如下：
$ ssh-keygen -t rsa
$ ssh-copy-id -i dn01(dn02,dn03)
```

## 五、安装 zookeeper (dn01,dn02,dn03)

### 1、解压 zookeeper

```shell
# tar xf zookeeper-3.4.13.tar.gz -C /usr/local/
# cd ..
# ln -s zookeeper-3.4.13 zookeeper
```

### 2、编辑 `zoo.cfg` 文件

```shell
# cd /usr/local/zookeeper/conf/
复制 zoo_sample.cfg 为 zoo.cfg
# cp zoo_sample.cfg zoo.cfg

# vim zoo.cfg
dataDir=/data/zk 
在最后添加，指定 zookeeper 集群主机及端口，机器数必须为奇数
server.1=dn01:2888:3888
server.2=dn02:2888:3888
server.3=dn03:2888:3888
```

### 3、创建并编辑 `myid` 文件

```shell
在 zookeeper dataDir 目录
# mkdir /data/zk
创建 myid 文件
# echo '1' > /data/zk/myid (dn02,dn03 设置为 2,3)
```

### 4、设置 zk 环境变量

```shell
# vim /etc/profile.d/zk.sh
# Zookeeper Environment
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
```

### 5、载入配置文件

```shell
# . /etc/profile.d/zk.sh
```

### 6、启动 zookeeper

```shell
# zkServer.sh start

查看状态
# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: follower
```

三台机器的 zookeeper 状态必须只有一个 `leader` ，其它都是 `follower`。

## 六、安装 Hadoop

> 只需要在 nn01 上操作，然后再分发到各机器。

### 1、解压 hadoop

```shell
# tar xf hadoop-3.1.1.tar.gz -C /usr/local/
# cd ..
# ln -s hadoop-3.1.1 hadoop
```

### 2、更改 hadoop 文件的属主及属组

```sehll
# chown -R hadoop:hadoop /usr/local/hadoop-3.1.1
```

### 3、设置 hadoop 环境变量

```shell
# Hadoop Environment
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```

### 4、创建 hadoop 数据存放目录

```shell
# mkdir -pv /data/hadoop/{logs,hdfs/{nn,dn,jn}}
mkdir: created directory ‘/data/hadoop’
mkdir: created directory ‘/data/hadoop/logs’
mkdir: created directory ‘/data/hadoop/hdfs’
mkdir: created directory ‘/data/hadoop/hdfs/nn’
mkdir: created directory ‘/data/hadoop/hdfs/dn’
mkdir: created directory ‘/data/hadoop/hdfs/jn’

更改属主及属组
# chown -R hadoop:hadoop /data/hadoop
```

### 5、修改配置文件

配置文件路径 `/usr/local/hadoop/etc/hadoop`

切换到 hadoop 用户

```shell
# su - hadoop
# cd /usr/local/hadoop/etc/hadoop
```

#### 1. 修改 `hadoop-env.sh` 文件

```shell
修改如下几项：
根据自己服务器配置设置 jvm 内存大小
export JAVA_HOME=/usr/local/jdk
export HDFS_NAMENODE_OPTS="-Xms1024m -Xmx1024m -XX:+UseParallelGC"
export HDFS_DATANODE_OPTS="-Xms512m -Xmx512m"
export HADOOP_LOG_DIR=/data/hadoop/logs
```

#### 2. 修改 `core-site.xml` 文件

```xml
<configuration>
    <property>
    	<!-- 指定 hdfs 的地址，ha 模式中是连接到 nameservice -->
        <name>fs.defaultFS</name>
        <value>hdfs://mycluster</value>
    </property>
    <property>
    	<!-- NameNode、DataNode、JournalNode 等存放数据的公共目录，也可单独指定 -->
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop/tmp</value>
    </property>
    <property>
    	<!-- 指定 zookeeper 地址 -->
        <name>ha.zookeeper.quorum</name>
        <value>dn01:2181,dn02:2181,dn03:2181</value>
    </property>
</configuration>
```

#### 3. 修改`hdfs-site.xml` 文件

```xml
<configuration>
    <property>
        <!-- 指定 hdfs 的 nameservice 为 mycluster，需要和 core-site.xml 中的保持一致
         dfs.ha.namenodes.[nameservice id] 为在 nameservice 中的每一个 NameNode 设置唯一标示符。配置一个逗号分隔的 NameNode ID 列表。这将是被 DataNode 识别为所有的 NameNode。
        例如，如果使用 "mycluster" 作为 nameservice ID，并且使用 "nn01" 和 "nn02" 作为NameNodes 标示符 -->
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>   
    <property>
        <!-- mycluster下面有两个 NameNode，分别是 nn01，nn02 -->
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn01,nn02</value>
    </property>  
    <property>
        <!-- nn01 的 RPC 通信地址,rpc 用来和 datanode 通讯 -->
        <name>dfs.namenode.rpc-address.mycluster.nn01</name>
        <value>nn01:8020</value>
    </property> 
    <property>
        <!-- nn02 的 RPC 通信地址,rpc 用来和 datanode 通讯 -->
        <name>dfs.namenode.rpc-address.mycluster.nn02</name>
        <value>nn02:8020</value>
    </property>   
    <property>
        <!-- nn01 的 http 通信地址,用来和 web 客户端通讯 -->
        <name>dfs.namenode.http-address.mycluster.nn01</name>
        <value>nn01:50070</value>
    </property> 
    <property>
        <!-- nn02 的 http 通信地址,用来和 web 客户端通讯 -->
        <name>dfs.namenode.http-address.mycluster.nn02</name>
        <value>nn02:50070</value>
    </property>
    <property>
        <!-- 指定 NameNode 的 edits 元数据的共享存储位置。也就是 JournalNode 列表
         该 url 的配置格式：qjournal://host1:port1;host2:port2;host3:port3/journalId
        journalId 推荐使用 nameservice，默认端口号是：8485 -->
        <name>dfs.namenode.shared.edits.dir</name>
  <value>qjournal://nn01:8485;nn02:8485;dn01:8485;dn02:8485;dn03:8485/mycluster</value>
    </property> 
    <property>
        <!-- 开启 NameNode 失败自动切换 -->
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <!-- 配置失败自动切换实现方式 -->
        <name>dfs.client.failover.proxy.provider.mycluster</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <!-- 配置隔离机制方法，多个机制用换行分割，即每个机制暂用一行 -->
        <name>dfs.ha.fencing.methods</name>
        <value>
            sshfence
	    	shell(/bin/true)
        </value>
    </property>
    <property>
        <!-- 是否在 HDFS 中开启权限检查 -->
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
    <property>
        <!-- 是否允许 append -->
        <name>dfs.support.append</name>
        <value>true</value>
    </property>
    <property>
        <!-- 使用 sshfence 隔离机制时需要 ssh 免登陆 -->
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/hadoop/.ssh/id_rsa</value>
    </property>
    <property>
        <!-- 指定副本数 -->
        <name>dfs.replication</name>
        <value>2</value>
    </property>
    <property>
        <!-- 本地磁盘目录，NN 存储 fsimage 文件的地方。可以是按逗号分隔的目录列表，fsimage 文件会存储在全部目录，冗余安全。这里多个目录设定，最好在多个磁盘，另外，如果其中一个磁盘故障，不会导致系统故障，会跳过坏磁盘。由于使用了 HA，建议仅设置一个。如果特别在意安全，可以设置 2 个 -->
        <name>dfs.namenode.name.dir</name>
        <value>/data/hadoop/hdfs/nn</value>
    </property>
    <property>
        <!-- 本地磁盘目录，HDFS 数据应该存储 Block 的地方。可以是逗号分隔的目录列表（典型的，每个目录在不同的磁盘）。这些目录被轮流使用，一个块存储在这个目录，下一个块存储在下一个目录，依次循环。每个块在同一个机器上仅存储一份。不存在的目录被忽略。必须创建文件夹，否则被视为不存在。 -->
        <name>dfs.datanode.data.dir</name>
        <value>/data/hadoop/hdfs/dn</value>
    </property>
    <!-- 指定 JournalNode 在本地磁盘存放数据的位置 -->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/data/hadoop/hdfs/jn</value>
    </property>  
    <property>
        <!-- 启用 webhdfs -->
        <name>dfs.webhdfs.enabled</name>
        <value>true</value>
    </property>
    <property>
        <!-- 配置 sshfence 隔离机制超时时间 -->
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>30000</value>
    </property>
    <property>
        <!-- 手动运行的 FC 功能（从CLI）等待健康检查、服务状态的超时时间 -->
        <name>ha.failover-controller.cli-check.rpc-timeout.ms</name>
        <value>60000</value>
    </property>
</configuration>
```

#### 4. 修改 `mapred-site.xml` 文件

```xml
<configuration>
	<property>
        <!-- 指定 mr 框架为 yarn 方式 -->
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <!-- 指定 mapreduce jobhistory 地址 -->
        <name>mapreduce.jobhistory.address</name>
        <value>nn01:10020</value>
    </property>
    <property>
        <!-- 任务历史服务器的 web 地址 -->
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>nn01:19888</value>
    </property>
    <!-- property>
        <name>mapreduce.application.classpath</name>
        <value>
          /usr/local/hadoop/hadoop/etc/hadoop,
          /usr/local/hadoop/hadoop/share/hadoop/common/*,
          /usr/local/hadoop/hadoop/share/hadoop/common/lib/*,
          /usr/local/hadoop/hadoop/share/hadoop/hdfs/*,
          /usr/local/hadoop/hadoop/share/hadoop/hdfs/lib/*,
          /usr/local/hadoop/hadoop/share/hadoop/mapreduce/*,
          /usr/local/hadoop/hadoop/share/hadoop/mapreduce/lib/*,
          /usr/local/hadoop/hadoop/share/hadoop/yarn/*,
          /usr/local/hadoop/hadoop/share/hadoop/yarn/lib/*
        </value>
    </property -->
    <property>
        <!-- 用户为 MR AM 添加环境变量 -->
  		<name>yarn.app.mapreduce.am.env</name>
  		<value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
	</property>
	<property>
  		<name>mapreduce.map.env</name>
  		<value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
	</property>
	<property>
  		<name>mapreduce.reduce.env</name>
  		<value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
	</property>
</configuration>
```

#### 5. 修改 `yarn-site.xml` 文件

```xml
<configuration> 
    <property>
        <!-- 开启 RM 高可用 -->
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <property>
        <!-- 指定 RM 的 cluster id -->
        <name>yarn.resourcemanager.cluster-id</name>
        <value>yrc</value>
    </property>
    <property>
        <!-- 指定 RM 的名字 -->
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <property>
        <!-- 分别指定 RM 的地址 -->
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>nn01</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>nn02</value>
    </property>
    <property>
        <!-- 指定 zk 集群地址 -->
        <name>hadoop.zk.address</name>
        <value>dn01:2181,dn02:2181,dn03:2181</value>
    </property>
    <property>
        <!-- 请配置为：mapreduce.shuffle，在 Yarn 上开启 MR 的必须项 -->
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <!-- 是否允许日志汇聚功能。建议开启 -->
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <property>
        <!-- 保存汇聚日志时间，秒，超过会删除，-1 表示不删除。注意，设置的过小，将导致 NN 垃圾碎片。建议 3-7 天 = 7 * 86400 = 604800 -->
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property> 
    <property>
        <!-- 启用自动恢复 -->
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
    <property>
        <!-- 指定 resource manager 的状态信息存储在 zookeeper 集群上 -->
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address.rm1</name>
	<value>nn01:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address.rm2</name>
	<value>nn02:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address.rm1</name>
	<value>nn01:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address.rm2</name>
	<value>nn02:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm1</name>
	<value>nn01:8088</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm2</name>
	<value>nn02:8088</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
	<value>nn01:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address.rm2</name>
	<value>nn02:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address.rm1</name>
	<value>nn01:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address.rm2</name>
	<value>nn02:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.admin.address.rm1</name>
	<value>nn01:23142</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.admin.address.rm2</name>
	<value>nn02:23142</value>
    </property>
</configuration>
```

#### 6. 修改 `workers` 文件

```shell
将 所有节点 的主机名写入到文件中
$ cat workers
nn01
nn02
dn01
dn02
dn03
```

#### 7. 修改 `start-dfs.sh` 、`stop-dfs.sh` 文件

```shell
添加如下内容：
HDFS_DATANODE_USER=hadoop
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_ZKFC_USER=hadoop
HDFS_JOURNALNODE_USER=hadoop
HDFS_NAMENODE_USER=hadoop
HDFS_SECONDARYNAMENODE_USER=hadoop
```

#### 8. 修改 `start-yarn.sh`、`stop-yarn.sh` 文件

```shell
添加如下内容：
YARN_RESOURCEMANAGER_USER=hadoop
HDFS_DATANODE_SECURE_USER=yarn
YARN_NODEMANAGER_USER=hadoop
```

### 6、将 hadoop 目录分发到其它机器

使用 scp 命令将 hadoop 目录复制到其它机器，复制之前可以先将 `/usr/local/hadoop/share/doc` 文档目录删除，以减小文件体积。

以 root 用户操作，复制成功后修改权限为 hadoop。

```shell
# scp -r /usr/local/hadoop-3.1.1 nn02:/usr/local/
# scp -r /usr/local/hadoop-3.1.1 dn01:/usr/local/
# scp -r /usr/local/hadoop-3.1.1 dn02:/usr/local/
# scp -r /usr/local/hadoop-3.1.1 dn03:/usr/local/
```

复制 hadoop 环境变量配置文件到各机器

```shell
# scp /etc/profile.d/hadoop.sh nn02:/etc/profile.d/
# scp /etc/profile.d/hadoop.sh dn01:/etc/profile.d/
# scp /etc/profile.d/hadoop.sh dn02:/etc/profile.d/
# scp /etc/profile.d/hadoop.sh dn03:/etc/profile.d/
```

所有文件复制完成后，创建好 hadoop 所需要的目录并设置好目录权限，

```shell
# mkdir -pv /data/hadoop/{logs,hdfs/{nn,dn,jn}}
# chown -R hadoop:hadoop /data/hadoop
```

### 7、初始化并启动 hadoop

> 启动顺序：
>
> zookeeper --> JournalNode --> 格式化 NameNode --> 创建命名空间 (zkfc) --> NameNode --> DataNode --> ResourceManager --> NodeManager

**以下操作以 hadoop 用户进行**

#### 1. 启动 zookeeper

前面已经启动

#### 2. 启动 JournalNode

nn01,nn02,dn01,dn02,dn03

```shell
$ hdfs --daemon start journalnode
$ jps
14480 Jps
14461 JournalNode
```

#### 3. 格式化 NameNode

nn01

```shell
$ hdfs --daemon start journalnode
...
2019-01-03 17:04:05,974 INFO namenode.FSImage: Allocated new BlockPoolId: BP-2014040957-192.168.2.75-1546506245974
2019-01-03 17:04:06,001 INFO common.Storage: Storage directory /data/hadoop/hdfs/nn has been successfully formatted.
2019-01-03 17:04:06,653 INFO namenode.FSImageFormatProtobuf: Saving image file /data/hadoop/hdfs/nn/current/fsimage.ckpt_0000000000000000000 using no compression
2019-01-03 17:04:07,026 INFO namenode.FSImageFormatProtobuf: Image file /data/hadoop/hdfs/nn/current/fsimage.ckpt_0000000000000000000 of size 391 bytes saved in 0 seconds .
2019-01-03 17:04:07,038 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
2019-01-03 17:04:07,909 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at nn01/192.168.2.75
************************************************************/
...
```

有看到`INFO common.Storage: Storage directory /data/hadoop/hdfs/nn has been successfully formatted.` 输出，说明格式化成功

**把 nn01 节点上生成的元数据复制到其它节点上**

```shell
$ scp -r /data/hadoop/hdfs/nn/* nn02:/data/hadoop/hdfs/nn/
$ scp -r /data/hadoop/hdfs/nn/* dn01:/data/hadoop/hdfs/nn/
$ scp -r /data/hadoop/hdfs/nn/* dn02:/data/hadoop/hdfs/nn/
$ scp -r /data/hadoop/hdfs/nn/* dn03:/data/hadoop/hdfs/nn/
```

#### 4. 格式化 zkfc

**只能在 NameNode 节点执行(nn01)**

```shell
$ hdfs zkfc -formatZK
...
2019-01-03 17:12:02,391 INFO zookeeper.ClientCnxn: Opening socket connection to server dn02/192.168.2.78:2181. Will not attempt to authenticate using SASL (unknown error)
2019-01-03 17:12:02,398 INFO zookeeper.ClientCnxn: Socket connection established to dn02/192.168.2.78:2181, initiating session
2019-01-03 17:12:02,447 INFO zookeeper.ClientCnxn: Session establishment complete on server dn02/192.168.2.78:2181, sessionid = 0x2002e6835960000, negotiated timeout = 10000
2019-01-03 17:12:02,486 INFO ha.ActiveStandbyElector: Successfully created /hadoop-ha/mycluster in ZK.
2019-01-03 17:12:02,491 INFO zookeeper.ZooKeeper: Session: 0x2002e6835960000 closed
...
```

看到 `INFO ha.ActiveStandbyElector: Successfully created /hadoop-ha/mycluster in ZK.` 输出，说明格式化 zkfc 成功。

#### 5. 启动 HDFS

**只能在 NameNode 节点执行 (nn01)**

```shell
$ start-dfs.sh
```

#### 6. 启动 YARN

在主备 ResourceManager 中随便选择一台进行启动

nn02

```shell
$ start-yarn.sh
```

若备用节点的 ResourceManager 没有启动起来，则手动启动起来: `yarn --daemon start resourcemanager`

#### 7. 启动 mapreduce 任务历史服务器

```shell
$ mapred --daemon start historyserver
```

#### 8. 状态查看

查看各主节点的状态

```shell
[hadoop@nn01 ~]$ hdfs haadmin -getServiceState nn01
active
[hadoop@nn01 ~]$ hdfs haadmin -getServiceState nn02
standby
[hadoop@nn01 ~]$ yarn rmadmin -getServiceState rm1
standby
[hadoop@nn01 ~]$ yarn rmadmin -getServiceState rm2
active
```

web 界面

HDFS：

`http://192.168.2.75:50070`

`http://192.168.2.76:50070`

YARN:

`http://192.168.2.76:8088`

## 七、高可用测试

### 1、NameNode 高可用测试

> 测试 NameNode 之前，需要在各 NameNode 上安装 `psmisc` 软件包，不然 `fuser` 命令将不可用，导致故障转移失败。

上传一个文件到 HDFS，然后将 NameNode 给 kill 掉，再看看上传的文件还是否存在

```shell
$ cat word.txt 
hadoop mapreduce hive
hbase spark storm
sqoop hadoop hive
spark hadoop
```

将这个文件上传到 DFS

```shell
$ hdfs dfs -put ./word.txt /
$ hdfs dfs -ls /
Found 2 items
drwxrwx---   - hadoop supergroup          0 2019-01-03 17:33 /tmp
-rw-r--r--   2 hadoop supergroup         71 2019-01-03 17:59 /word.txt
$ hdfs dfs -cat /word.txt
hadoop mapreduce hive
hbase spark storm
sqoop hadoop hive
spark hadoop
```

查看当前哪个节点是 active 的

```
$ hdfs haadmin -getServiceState nn01
active
$ hdfs haadmin -getServiceState nn02
standby
```

kill 掉 nn01 上的 NameNode 进程

```
$ jps
18513 Jps
16517 JournalNode
16713 DFSZKFailoverController
16841 ResourceManager
16268 NameNode
17180 JobHistoryServer

$ kill -9 16268
```

查看 active 节点有没有转移到 nn02  上

```shell
$ hdfs haadmin -getServiceState nn02
```

检查 word.txt 还是否在 dfs 上

```shell
$ hdfs dfs -ls /
Found 2 items
drwxrwx---   - hadoop supergroup          0 2019-01-03 17:33 /tmp
-rw-r--r--   2 hadoop supergroup         71 2019-01-03 17:59 /word.txt
```

重新启动 nn01 上的 NameNode 服务

```shell
$ hdfs --daemon start namenode
```

查看当前 active 状态的节点是哪一个

```shell
$ hdfs haadmin -getServiceState nn01
standby
$ hdfs haadmin -getServiceState nn02
active
```

可以看到，即使 nn01 重新启动了，但 active 节点仍然是 nn02，nn01 变成了 standby。

### 2、 ResourceManager 高可用测试

查看当前哪个节点是 active

```shell
$ yarn rmadmin -getServiceState rm1
standby
$ yarn rmadmin -getServiceState rm2
active
```

运行 wordcount 测试用例，运行过程中，将 nn02 上的 ResourceManager 进程 kill 掉，看 wordcount 用例能否继续成功执行

```shell
$ yarn jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.1.jar wordcount /word.txt /output

$ jps 
4352 NameNode
4544 DFSZKFailoverController
4436 JournalNode
8420 Jps
7982 ResourceManager
$ kill -9 7982
```

查看 dfs 上是否有成功执行后生成的文件

```shell
$ hdfs dfs -ls /output
Found 2 items
-rw-r--r--   2 hadoop supergroup          0 2019-01-03 20:11 /output/_SUCCESS
-rw-r--r--   2 hadoop supergroup         60 2019-01-03 20:11 /output/part-r-00000

$ hdfs dfs -cat /output/part-r-00000
hadoop	3
hbase	1
hive	2
mapreduce	1
spark	2
sqoop	1
storm	1
```

可以看到 wordcount 用例运行是成功的。

重新启动 nn02 上 ResourceManager 进程

```shell
$ yarn --daemon start resourcemanager
```

查看当前节点状态

```shell
$ yarn rmadmin -getServiceState rm1
active
$ yarn rmadmin -getServiceState rm2
standby
```

可见，即使 nn02 重启之后，active 节点依然还是在 nn01 上。