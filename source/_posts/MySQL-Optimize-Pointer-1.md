title: "MySQL 优化要点整理（一）"
date: 2015-07-10 10：32：00
tags:
- MySQL
- 优化
categories: 
---

## MySQL使用误区

1. 将MySQL作为文件存储。MySQL存储大量数据时会造成数据库性能下降，建议将文件交由文件系统处理，MySQL只存储文件的存储位置。若需要将文章内容等大量文本内容存储进数据库时建议与文章的元数据（作者、发布时间等）分表存储。
2. 使用MySQL存储中间数据，如游戏的中间数据。MySQL作为一个持久存储的数据库，不建议将一些频繁修改的中间数据存入MySQL，会造成MySQL压力过大。
3. 将MySQL作为搜索引擎。MySQL的全文索引性能并不好，建议使用 `Sphinx` (`Coreseek`) 、`Lucene` 或 `Solr` 等专业的全文索引引擎提供全文检索功能。

## MySQL版本的选择

### 官方版本的选择

MySQL5.1之前的版本对多核支持较差（可使用 `top` 命令后按 `1` 查看CPU核心的是用情况）；
5.1支持4个CPU核心；
5.5支持24个核心；
5.6支持64个核心。

建议使用最新的稳定版。

MySQL对与每一个连接使用一个线程来处理，每个并发query使用一个核心，所以应尽量让每一个query快速完成，减少后续query等待。

### 分支版本的选择

官方版本锁竞争严重，建议使用 `MariaDB` 或者 `Precona` 分支，当前建议使用的顺序为：

Percona > MariaDB > 官方MySQL

## MySQL的内存利用特点

MySQL对于内存的使用分为全局内存块和session（会话）内存块。

PGA（程序全局区，Program Global Area，每个会话分配的内存）不宜过大，会造成OOM（out of memory）。

可通过增加物理内存降低系统IO，提高并发性能。
一些存储引擎也会将锁放入物理内存中。

query cache (查询缓存) 管理机制糟糕，不建议使用，直接关闭。原因是query cache存在全局锁，任何数据变更都会导致cache失效。

与Oracle不同的是，MySQL没有执行计划缓存，原因是MySQL的查询解析器非常精简、高效，无需缓存。

硬件方面的内存分配大概为热点数据的 15%~20%，MySQL单机专用实例建议分配的内存为全部物理内存的 50%~70%（可由50%开始分配，逐步加大看内存使用）。

K-V（key-value）简单数据使用 NoSQL 存储，例如 redis、memcached、mongoDB 等。

redis 与 memcached 的选择建议 redis，速度更快功能更多。

## MySQL的磁盘使用特点

binlog、redolog、undolog 主要是顺序IO。

datafile 主要是随机IO，也有顺序IO。

OLTP（Online Transactions Processing，联机事务处理）已随机IO为主，建议加大物理内存（合并随机IO为顺序IO）。

OLAP（Online Analysis Processing，联机分析）已顺序IO为主，建议加大内存、增加硬盘数量提高顺序IO性能。

MyISAM是堆组织表（Heap Organize Table，HOT），存储时直接写在数据尾部。

> 堆组织表中，数据以堆的方式管理。增加数据时，会使用段中找到的第一个能放下此数据的自由空间。从表中删除数据后，允许以后的INSERT和UPDATE重用这部分空间。堆（heap）是一组空间，以一种随机的方式使用。因此，无法保证按照放入表中的顺序取得数据​。

InnoDB是索引组织表（Index Organize Table，IOT），使用B+树进行数据组织。

>  索引组织表(IOT)不仅可以存储数据，还可以存储为表建立的索引。索引组织表的数据是根据主键排序后的顺序进行排列的，这样就提高了访问的速度。但是这是由牺牲插入和更新性能为代价的(每次写入和更新后都要重新进行重新排序)。

InnoDB会比MyISAM消耗更多的磁盘控件一般会消耗1.5~2）​倍。

InnoDB基于主键的查找会比MyISAM快。MyISAM的索引与数据是分开的，所以要读取两次。



