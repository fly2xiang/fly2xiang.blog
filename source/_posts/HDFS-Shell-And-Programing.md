title: "HDFS 命令行与编程访问"
date: 2018-04-10 16:20:17
tags:
- Hadoop
- HDFS
- 大数据
- 分布式存储
categories: 
- 大数据


## 摘要

借助于文件系统 Shell，可以像 Linux Shell 一样方便的访问 HDFS。另外的，通过连接 namenode 可以编程远程访问 HDFS。

## 命令行访问

HDFS 可以使用 `./bin/hdfs` 访问，也可在 Hadoop 配置为使用 HDFS 作为存储时 (core-site.xml 中 fs.defaultFS) 使用 `./bin/hadoop fs` 访问。
这里以 `./bin/hadoop fs` 为例。

创建工作目录

```bash
./bin/hadoop fs -mkdir /user
```

同样的，这里可以加参数 -p 创建一个多层目录

```bash
./bin/hadoop fs -mkdir -p /user/hadoop/app1/sample/input/demo
```

列出目录内容

```bash
./bin/hadoop fs -ls /user/hadoop/app1/sample
```

复制本地文件到 HDFS

```bash
./bin/hadoop fs -put README.txt /user/hadoop/app1/sample
```

复制 HDFS 文件到本地

```bash
./bin/hadoop fs -get /user/hadoop/app1/sample/README.txt README.txt.from_HDFS
```

直接查看 HDFS 文件内容

```bash
./bin/hadoop fs -cat /user/hadoop/app1/sample/README.txt
···

查看最后几行

```bash
./bin/hadoop fs -tail -f /user/hadoop/app1/sample/README.txt
```

删除文件

```bash
./bin/hadoop fs -rm /user/hadoop/app1/sample/README.txt
```

追加文件内容

```bash
# 将本地 README.txt 文件的内容追加到指定的 HDFS 文件，这里可以看到可以访问远程 HDFS
./bin/hadoop fs -appendToFile README.txt hdfs://hadoop-host:9000/user/hadoop/app1/sample/README.txt
# 将标准输入追加到 HDFS 文件
./bin/hadoop fs -appendToFile - hdfs://hadoop-host:9000/user/hadoop/app1/sample/README.txt
```

使用 `./bin/hadoop fs` 不仅可以访问 HDFS，还可以使用本地文件、AWS S3等

```bash
# 访问本地文件系统
./bin/hadoop fs -cat file:///etc/redhat-release
```

## 编程访问



## 可能遇到的问题

* 写入 hdfs 时报 `0 datanode running` 之类

可能是由于多次格式化了 namenode，可检查 `logs/hadoop-***-datanode-***.log` 看到异常: 

```
java.io.IOException: Incompatible clusterIDs in /tmp/hadoop-hadoop/dfs/data: namenode clusterID = CID-d5386456-16fe-45c9-9a17-afd5e3d853b0; datanode clusterID = CID-fdd3a3c8-3724-4b80-9e0e-ba7969fc4ebf
```

解决办法是指定与 datanode 相同的 clusterId 格式化 namenode

```bash
./bin/hdfs namenode -format -clusterId CID-fdd3a3c8-3724-4b80-9e0e-ba7969fc4ebf
```


## 参考链接

[FileSystem Shell](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/FileSystemShell.html)


