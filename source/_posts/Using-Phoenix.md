title: "Phoenix 使用"
date: 2018-05-12 17:20:17
tags:
- Hadoop
- 大数据
- MapReduce
- Phoenix
categories: 
- 大数据

---


## 概述

Phoenix 是 SQL on Hadoop 的一个方案，可以使用 Phoenix 来查询 HBase 中的数据。

## 下载安装

从 [Phoenix 官网](http://phoenix.apache.org/) 下载对应的压缩包，要根据对应的 HBase 版本选择，这里选择 `apache-phoenix-4.13.1-HBase-1.2-bin.tar.gz`。

下载完成后解压，将 `phoenix-4.13.1-HBase-1.2-server.jar` 拷贝到对应的 HBase `lib` 目录下，重启 HBase 。

## 简单操作

执行 Phoenix 目录下的 `./bin/sqlline.py`，可以进入 SQL 命令行。

```
./bin/sqlline.py
0: jdbc:phoenix:localhost:2181:/hbase> !table
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-----------------+--------------+-----------------+---+
| TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE_NAME  | SELF_REFERENCING_COL_NAME  | REF_GENERATION  | INDEX_STATE  | IMMUTABLE_ROWS  | S |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-----------------+--------------+-----------------+---+
|            | SYSTEM       | CATALOG     | SYSTEM TABLE  |          |            |                            |                 |              | false           | n |
|            | SYSTEM       | FUNCTION    | SYSTEM TABLE  |          |            |                            |                 |              | false           | n |
|            | SYSTEM       | SEQUENCE    | SYSTEM TABLE  |          |            |                            |                 |              | false           | n |
|            | SYSTEM       | STATS       | SYSTEM TABLE  |          |            |                            |                 |              | false           | n |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-----------------+--------------+-----------------+---+
0: jdbc:phoenix:localhost:2181:/hbase> CREATE TABLE tbl_book (id integer PRIMARY KEY, name varchar, author varchar);
No rows affected (1.57 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> UPSERT INTO tbl_book(id, name, author) VALUES(1, 'Hadoop: The Definitive Guide', 'Tom White');
1 row affected (0.103 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> SELECT * FROM tbl_book;
+-----+-------------------------------+------------+
| ID  |             NAME              |   AUTHOR   |
+-----+-------------------------------+------------+
| 1   | Hadoop: The Definitive Guide  | Tom White  |
+-----+-------------------------------+------------+
1 row selected (0.103 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> DELETE FROM tbl_book WHERE id = 1;
1 row affected (0.042 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> SELECT * FROM tbl_book;
+-----+-------+---------+
| ID  | NAME  | AUTHOR  |
+-----+-------+---------+
+-----+-------+---------+
No rows selected (0.073 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> 
```

可以看到这里与关系型数据库的操作基本相同。

## 与 HBase 互操作

### 在 HBase 中读写 Phoenix 写入的数据

先在 Phoenix 添加两条数据

```
0: jdbc:phoenix:localhost:2181:/hbase> UPSERT INTO tbl_book(id, name, author) VALUES(2, 'Learning Spark: Lightning-fast Data Analysis', 'Holden Karau');
1 row affected (0.023 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> SELECT * FROM tbl_book;
+-----+-----------------------------------------------+---------------+
| ID  |                     NAME                      |    AUTHOR     |
+-----+-----------------------------------------------+---------------+
| 1   | Hadoop: The Definitive Guide                  | Tom White     |
| 2   | Learning Spark: Lightning-fast Data Analysis  | Holden Karau  |
+-----+-----------------------------------------------+---------------+
2 rows selected (0.094 seconds)
```

然后来到 HBase 的命令行

```
hbase(main):004:0> scan 'TBL_BOOK'
ROW                                        COLUMN+CELL                                                                                                                 
 \x80\x00\x00\x01                          column=0:\x00\x00\x00\x00, timestamp=1523929914637, value=x                                                                 
 \x80\x00\x00\x01                          column=0:\x80\x0B, timestamp=1523929914637, value=Hadoop: The Definitive Guide                                              
 \x80\x00\x00\x01                          column=0:\x80\x0C, timestamp=1523929914637, value=Tom White                                                                 
 \x80\x00\x00\x02                          column=0:\x00\x00\x00\x00, timestamp=1523930135150, value=x                                                                 
 \x80\x00\x00\x02                          column=0:\x80\x0B, timestamp=1523930135150, value=Learning Spark: Lightning-fast Data Analysis                              
 \x80\x00\x00\x02                          column=0:\x80\x0C, timestamp=1523930135150, value=Holden Karau                                                              
2 row(s) in 0.4890 seconds

hbase(main):005:0> 
```

这里可以看出，表明被 Phoenix 默认存储为了大写，在 Phoenix 中定义的主键被存储为四字节整数，默认的列族名是 `0` 。

### 在 Phoenix 中读写 Hbase 写入的数据

先在 HBase 中写入几条数据

```
hbase(main):002:0> scan 'user'
ROW                                        COLUMN+CELL                                                                                                                 
 row_1                                     column=account:username, timestamp=1523610454413, value=jack                                                                
 row_2                                     column=account:username, timestamp=1523610497089, value=alex                                                                
 row_3                                     column=account:password, timestamp=1523610604420, value=123456                                                              
 row_3                                     column=account:username, timestamp=1523610515752, value=peter                                                               
 row_4                                     column=account:username, timestamp=1523759722311, value=tom                                                                 
4 row(s) in 0.4740 seconds
```

然后来到 Phoenix 命令行

```
0: jdbc:phoenix:localhost:2181:/hbase> CREATE TABLE "user"(id varchar primary key, "account"."username" varchar, "account"."password" varchar);
4 rows affected (5.972 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> SELECT * FROM "user";
+--------+-----------+-----------+
|   ID   | username  | password  |
+--------+-----------+-----------+
| row_1  | jack      |           |
| row_2  | alex      |           |
| row_3  | peter     | 123456    |
| row_4  | tom       |           |
+--------+-----------+-----------+
4 rows selected (0.083 seconds)
```

由于 Phoenix 默认会将表名、列族名、列名转换为大写，要保持小写需要添加双引号。可以看到通过创建与 HBase 同名的表，Phoenix 可以查询出通过 HBase 写入的数据。

修改 peter 的密码

```
0: jdbc:phoenix:localhost:2181:/hbase> UPSERT INTO "user"(id, "account"."password") VALUES('row_3', '888888');
1 row affected (0.013 seconds)
0: jdbc:phoenix:localhost:2181:/hbase> SELECT * FROM "user";
+--------+-----------+-----------+
|   ID   | username  | password  |
+--------+-----------+-----------+
| row_1  | jack      |           |
| row_2  | alex      |           |
| row_3  | peter     | 888888    |
| row_4  | tom       |           |
+--------+-----------+-----------+
4 rows selected (0.073 seconds)
```

回到 HBase 读取

```
hbase(main):014:0> get 'user','row_3','account:password'
COLUMN                                     CELL                                                                                                                        
 account:password                          timestamp=1523931290681, value=888888                                                                                       
1 row(s) in 0.0520 seconds
```

## 参考链接

[Phoenix F.A.Q.](https://phoenix.apache.org/faq.html)
[Phoenix Quick Start](https://phoenix.apache.org/Phoenix-in-15-minutes-or-less.html)
