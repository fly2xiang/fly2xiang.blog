title: "HBase 安装使用"
date: 2018-04-15 22:46:17
tags:
- Hadoop
- 大数据
- HBase
- HDFS
categories: 
- 大数据

## 概述

HBase 是 Hadoop 家族中分布式、可伸缩的大数据存储。使用 HBase 可以达到随机、实时读写大数据。HBase 支持数十亿的行与数百万的列。HBase 是根据 Google BigTable 论文实现的。

## 下载安装

从 [HBase 官网](https://hbase.apache.org/) 下载并解压。

```bash
tar zxvf hbase-1.2.6-bin.tar.gz
ln -s hbase-1.2.6 hbase
mkdir -p /opt/hbase/zookeeper/property/dataDir
cd hbase
```

编辑 `bin/hbase-site.xml` 配置文件

```xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://127.0.0.1:9000/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/opt/hbase/zookeeper/property/dataDir</value>
  </property>
  <property>
    <name>hbase.unsafe.stream.capability.enforce</name>
    <value>false</value>
    <description>
      Controls whether HBase will check for stream capabilities (hflush/hsync).

      Disable this if you intend to run on LocalFileSystem, denoted by a rootdir
      with the 'file://' scheme, but be mindful of the NOTE below.

      WARNING: Setting this to false blinds you to potential data loss and
      inconsistent system state in the event of process and/or node failures. If
      HBase is complaining of an inability to use hsync or hflush it's most
      likely not a false positive.
    </description>
  </property>
</configuration>
```

启动 HBase 服务

```bash
./bin/start-hbase.sh
```

启动后可通过 Web 访问 16010 端口访问管理界面。

## 基本概念

在使用 HBase 之前，需要理解一些基本概念

* 表(table)：列式存储，支持高表&宽表(上亿行，上百万列)
* 行(row)：每一行由唯一的行键确定
* 列族(columnFamily)：每一行包含一个或多个列族，是列的集合
* 列(column)：列式存储，列是最基本单位，可能有多个版本的值
* 时间戳(Timestamp)：列的不同版本之间用时间戳区分
* 单元格(cell)：列的每一个版本是一个单元格，是存储的基本单位

这里可以使用传统关系数据库的概念理解这些概念，但底层的数据存储结构与关系型数据库是不同的。

* 行键可以理解为主键，唯一确定一行。
* 列族的概念可以理解为纵向切分，访问频率不同或关系较近的列可以放在同一个列族。
例如文章，在传统数据库设计中为了性能考虑通常将文章内容这一较大的列拆分出去，与文章的其他数据（标题、作者等）放在不同的表中，这里 HBase 使用列族来区分。
* 同一张表的列族最好控制位 2 ~ 3 个。
* 值可以有多个版本，以时间戳区分。

HBase 实质上是一张多维的 Map，可以理解为以 rowKey + columnFamily + qualifier + timestamp 为 Key， 以 value 为 Value 的 K-V 存储。
带来的优势是列可以随时指定，不存在的列不会存储，但针对每个带来的优势是列可以随时指定，不存在的列不会存储。
但针对每个 cell 都存储了 Key，因此 columnFamily 名称不要太长。

## 基本操作

启动 HBase Shell，定义 user 表，进行增删改查

```bash
./bin/hbase shell
hbase> create 'user', 'account'	// 创建表，指定表名称与列族名称
hbase> put 'user', 'row_1', 'account:username' 'jack'	//添加一条数据，指定表名称，rowKey，列族与列名称，值
hbase> put 'user', 'row_2', 'account:username', 'alex'
hbase> put 'user', 'row_3', 'account:username', 'peter'
hbase> get 'user', 'row_2'	//以表名称，rowKey 查询所有列
hbase> put 'user', 'row_3', 'account:password' '123456'	//添加一个不同的列
hbase> get 'user', 'row_3', 'account'	//指定表名称，rowKey，columnFamily 查询，会查询出列族中的所有的列
hbase> get 'user', 'row_3', 'account:username'	//指定表名称，rowKey，columnFamily，列名查询值
```

## 基础架构

HBase 中的数据是分片存储的，每个分区中的 rowKey 是连续的。分区可能被存储在相同的机器上，也有可能是不同的机器。

HBase 在写入数据时会先写入 HLog，HLog 起到数据恢复的作用，同一个 Server 上的不同分区公用一个 HLog 实例。在写入数据时会先写入到内存中，待缓存写满后再刷入磁盘。

HBase 在文件的存储上完全依赖与 HDFS，称之为 HFile，在文件存储上，同一列族的数据会被写入到同一文件中，因此，在定义列族时最好将会被一同访问的数据放在同一列族中。

对于读缓存，HBase 是在同一服务器上的所有分区使用一个读缓存，当读取某一条数据时，HBase 会将整个 HFile block 读取到 cache 中。另外需要注意的是*在做全表扫描时应当关闭度缓存，防止将其他热数据从缓存中刷出*。

## 数据查找的步骤

对于查询一个 rowKey 的数据

1. 在 `hbase.meta` 表中查找数据对应的分区 (region id)，客户端会缓存 `hbase.meta` 表，以加快查询速度。
2. 在对应分区所在服务器上查找相应记录，有三种方式： 扫描缓存、块索引、布隆过滤器

* 扫描缓存，扫描缓存是在内存中进行，速度比较快
* 块索引，块索引在 HFile 结尾，包含了这个文件存储的起始 key，查找块索引可以很快找到此文件是否存在要找的数据
* 布隆过滤器，




