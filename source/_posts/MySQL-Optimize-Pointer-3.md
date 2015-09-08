title: "MySQL 优化要点整理（三）"
date: 2015-09-08 10：44：10
tags:
- MySQL
- 优化
categories: 
---

##设计优化

### 使用InnoDB引擎，适用于99%的应用场景

* 支持并发

* 数据一致性

* Crash_Recovery

* 更高的存取效率

* MyISAM只能缓存索引，InnoDB可以缓存索引 + 数据

### Schema设计

* 必须有自增主键（特别是MyISAM引擎下），不用UUID做主键（性能差）

* 日期时间使用 int 或 DateTime （推荐）

* IPv4地址使用int

* 对性别、是否等数据使用 enum、tinyint，不使用 varchar / char

* 杜绝Text/BLOB，必须使用时不要与其他数据存储在一个表中，可以做垂直拆分，放入MyISAM表

* 对于特定长度的字符串（如username）等，使用varchar(30)，不使用varchar(255)、char(30)

* 所有字段显式定义 NOT NULL 并制定默认值，使用 NULL 会有性能问题

### 索引的设计

* 基数很低的字段不创建索引

* MySQL不支持bitmap索引

* 采用第三方系统实现 Text/Blob 的全文索引（Sphinx、Coreseek、Lucene、ElashSearch）

* 常用的 ORDER BY 、GROUP BY 、DISTINCT 字段要建立索引

* 索引不能太多，会有负作用

* 多使用联合索引、少使用独立索引

* 字符型可创建前缀索引（如 username 字段 80% 的数据都小于18个字符，那么可以创建18个字符的前缀索引）

### 无法使用索引的场景

* 通过索引扫描的记录数超过30%会进行全表扫描

* 第一个索引列使用范围查询不能使用索引

* 内存表使用Hash进行全表扫描

* ORDER BY 、GROUP BY Hash索引只能进行等于/不等于的检索

* SELECT ... WHERE key1 = ? ORDER BY key2 ASC 对于key1和key2上的索引，查询优化器会自己判断用哪个（只能用到一个）

* 表关联字段类型要一样（包括长度），否则会有类型转换

* 使用函数时不能用到索引( WHERE func(key1) = ? 不能用到)( WHERE key1 + 1 = ? 不能用到)(WHERE key1 = ? + ? 可以用到)

### 常见的杀手级SQL

* SELECT *

* ORDER BY RAND()

* LIMIT ?, ?

* SELECT COUNT(*) FROM Some_One_InnoDB_Table

对于分页查询：

SELECT ... FORM table1 WHERE ... ORDER BY key1 LIMIT 9999999, 10

应使用子查询优化：

SELECT ... FROM table1 WHERE table1.id(自增主键) > ( SELECT id FROM table1 WHERE id > 9999999 ORDER BY key1) LIMIT 10

## 架构优化

### 架构优化要点

* 减少物理IO，让MySQL闲下来，使用前端Cache

* 转变随机IO为顺序IO，写队列最后合并

* 减小活跃数据，冷热数据分离

* 分库分表（水平、垂直、分布集群）

* 主从分离

### 优化工具

* pt-ioprofile

* mysqldumpslow

* pt-mysql-digest

## 优化误区

* 给MySQL分配的内存过大，会引起内存SWAP、OOM（out of memory）

* session 分配过大，会引起OOM

* 索引越多越好，会引发更多的IO

* Query Cache，效果差

* 认为MyISAM只读效率高于InnoDB

* 过度优化，改业务、改SQL、购买昂贵的IO设备等
