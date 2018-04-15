title: "Hadoop 3.1.0 伪分布式搭建"
date: 2018-04-09 11:20:17
tags:
- Hadoop
- 大数据
- 分布式
categories: 
- 大数据

---

## 概述

### Hadoop 介绍

Hadoop 主要包含两个部分：

* HDFS，即 Hadoop Distributed File System，一个分布式的文件系统，为大量数据的存储提供了解决方案
* YARN，即 Yet Another Resource Negotiator，一个资源管理器，负责管理分配全局资源与任务生命周期内的所有事宜

### 基础环境

* 硬件 VMware Workstation 虚拟机， 4核CPU 6G RAM
* 操作系统： CentOS 7.4.1708

## 下载安装

### 环境搭建：

按照自己的喜好建立一个用户名，这里使用 hadoop 作为用户名

* 新建用户 hadoop 与用户组

建立新用户可以方便的进行权限管理并避免使用 root 用户带来的安全问题

```bash
groupadd hadoop
useradd hadoop -g hadoop
```

建立用户后，以 hadoop 用户登录

* 配置 SSH 密钥免密码登录

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

配置完成后执行 `ssh localhost` 验证是否配置成功

* 下载配置 hadoop

从[Hadoop官网](http://hadoop.apache.org)下载 hadoop 压缩包之后解压

```bash
tar zxvf hadoop-3.1.0.tar.gz
cd hadoop-3.1.0
```

编辑几个配置文件：

etc/hadoop/core-site.xml

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

etc/hadoop/hdfs-site.xml:

这里添加 `dfs.namenode.rpc-address` IP 为 0.0.0.0 以支持远程访问 HDFS

```xml
<configuration>
    <property>
        <name>dfs.namenode.rpc-address</name>
        <value>0.0.0.0:9000</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

etc/hadoop/yarn-site.xml

```xml
<configuration>
   <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

etc/hadoop/maperd-site.xml

这里配置内存参数防止 oom

```xml
<configuration>
    <property>
        <name>mapreduce.map.memory.mb</name>
        <value>4096</value>
    </property>
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>4096</value>
    </property>
    <property>
        <name>mapreduce.map.java.opts</name>
        <value>-Xmx3072m</value>
    </property>
    <property>
        <name>mapreduce.reduce.java.opts</name>
        <value>-Xmx3072m</value>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

修改主机名

```bash
hostname hadoop-host
```

配置 hosts

如需在局域网内其他机器上访问，还需在其他机器上也配置 hosts

```bash
vim /etc/hosts
```

格式化 hdfs

···bash
./bin/hdfs namenode -format
···

启动 hdfs

```bash
./sbin/start-dfs.sh
```

启动 yarn

```
./sbin/start-yarn.sh
```

至此 hadoop 伪分布式已经搭建成功


## 参考链接

[Hadoop: Setting up a Single Node Cluster](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed_Operation)













