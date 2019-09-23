## SQL优化
    在应用的的开发过程中，由于初期数据量小，开发人员写SQL 语句时更重视功能上的实现，
但是当应用系统正式上线后，随着生产数据量的急剧增长，很多SQL 语句开始逐渐显露出
性能问题，对生产的影响也越来越大，此时这些有问题的SQL 语句就成为整个系统性能的
瓶颈，因此我们必须要对它们进行优化，本章将详细介绍在MySQL 中优化SQL 语句的方法。
## 18.1 优化SQL 语句的一般步骤
当面对一个有SQL 性能问题的数据库时，我们应该从何处入手来进行系统的分析，使得能
够尽快定位问题SQL 并尽快解决问题，本节将向读者介绍这个过程。
### 18.1.1 通过show status 命令了解各种SQL 的执行频率
MySQL 客户端连接成功后，通过show [session|global]status 命令可以提供服务器状态信
息，也可以在操作系统上使用mysqladmin extended-status 命令获得这些消息。show
[session|global] status 可以根据需要加上参数“session”或者“global”来显示session 级（当
前连接）的统计结果和global 级（自数据库上次启动至今）的统计结果。如果不写，默认使
用参数是“session”。
下面的命令显示了当前session 中所有统计参数的值：
mysql> show status like 'Com_%';
+--------------------------+-------+
| Variable_name | Value |
+--------------------------+-------+
| Com_admin_commands | 0 |
| Com_alter_db | 0 |
Linux公社 www.linuxidc.com
205
| Com_alter_event | 0 |
| Com_alter_table | 0 |
| Com_analyze | 0 |
| Com_backup_table | 0 |
| Com_begin | 0 |
| Com_change_db | 1 |
| Com_change_master | 0 |
| Com_check | 0 |
| Com_checksum | 0 |
| Com_commit | 0 |
……
Com_xxx 表示每个xxx 语句执行的次数，我们通常比较关心的是以下几个统计参数。
 Com_select：执行select 操作的次数，一次查询只累加1。
 Com_insert：执行INSERT 操作的次数，对于批量插入的INSERT 操作，只累加一次。
 Com_update：执行UPDATE 操作的次数。
 Com_delete：执行DELETE 操作的次数。
上面这些参数对于所有存储引擎的表操作都会进行累计。下面这几个参数只是针对
InnoDB 存储引擎的，累加的算法也略有不同。
 Innodb_rows_read：select 查询返回的行数。
 Innodb_rows_inserted：执行INSERT 操作插入的行数。
 Innodb_rows_updated：执行UPDATE 操作更新的行数。
 Innodb_rows_deleted：执行DELETE 操作删除的行数。
通过以上几个参数，可以很容易地了解当前数据库的应用是以插入更新为主还是以查询
操作为主，以及各种类型的SQL 大致的执行比例是多少。对于更新操作的计数，是对执行
次数的计数，不论提交还是回滚都会进行累加。
对于事务型的应用，通过Com_commit 和Com_rollback 可以了解事务提交和回滚的情况，
对于回滚操作非常频繁的数据库，可能意味着应用编写存在问题。
此外，以下几个参数便于用户了解数据库的基本情况。
 Connections：试图连接MySQL 服务器的次数。
 Uptime：服务器工作时间。
 Slow_queries：慢查询的次数。
18.1.2 定位执行效率较低的SQL 语句
可以通过以下两种方式定位执行效率较低的SQL 语句。
 通过慢查询日志定位那些执行效率较低的SQL 语句，用--log-slow-queries[=file_name]选
项启动时，mysqld 写一个包含所有执行时间超过long_query_time 秒的SQL 语句的日志
文件。具体可以查看本书第26 章中日志管理的相关部分。
 慢查询日志在查询结束以后才纪录，所以在应用反映执行效率出现问题的时候查询慢查
询日志并不能定位问题，可以使用show processlist 命令查看当前MySQL 在进行的线程，
包括线程的状态、是否锁表等，可以实时地查看SQL 的执行情况，同时对一些锁表操
作进行优化。
18.1.3 通过EXPLAIN 分析低效SQL 的执行计划
Linux公社 www.linuxidc.com
206
通过以上步骤查询到效率低的SQL 语句后，可以通过EXPLAIN 或者DESC 命令获取MySQL
如何执行SELECT 语句的信息，包括在SELECT 语句执行过程中表如何连接和连接的顺序，比
如想计算2006 年所有公司的销售额，需要关联sales 表和company 表，并且对moneys 字段
做求和（sum）操作，相应SQL 的执行计划如下：
mysql> explain select sum(moneys) from sales a,company b where a.company_id = b.id and a.year
= 2006\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: a
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
*************************** 2. row ***************************
id: 1
select_type: SIMPLE
table: b
type: ref
possible_keys: ind_company_id
key: ind_company_id
key_len: 5
ref: sakila.a.company_id
rows: 1
Extra: Using where; Using index
2 rows in set (0.00 sec)
每个列的简单解释如下：
 select_type：表示SELECT 的类型，常见的取值有SIMPLE（简单表，即不使用表连接
或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或
者后面的查询语句）、SUBQUERY（子查询中的第一个SELECT）等。
 table：输出结果集的表。
 type：表示表的连接类型，性能由好到差的连接类型为system（表中仅有一行，即
常量表）、const（单表中最多有一个匹配行，例如primary key 或者unique index）、
eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接
中使用primary key或者unique index）、re（f 与eq_ref类似，区别在于不是使用primary
key 或者unique index，而是使用普通的索引）、ref_or_null（与ref 类似，区别在于
条件中包含对NULL 的查询）、index_merge(索引合并优化)、unique_subquery（in
的后面是一个查询主键字段的子查询）、index_subquery（与unique_subquery 类似，
区别在于in 的后面是查询非唯一索引字段的子查询）、range（单表中的范围查询）、
index（对于前面的每一行，都通过查询索引来得到数据）、all（对于前面的每一行，
Linux公社 www.linuxidc.com
207
都通过全表扫描来得到数据）。
 possible_keys：表示查询时，可能使用的索引。
 key：表示实际使用的索引。
 key_len：索引字段的长度。
 rows：扫描行的数量。
 Extra：执行情况的说明和描述。
18.1.4 确定问题并采取相应的优化措施
经过以上步骤，基本就可以确认问题出现的原因。此时用户可以根据情况采取相应的措
施，进行优化提高执行的效率。
在上面的例子中，已经可以确认是对a 表的全表扫描导致效率的不理想，那么对a 表的
year 字段创建索引，具体如下：
mysql> create index ind_sales2_year on sales2(year);
Query OK, 1000 rows affected (0.03 sec)
Records: 1000 Duplicates: 0 Warnings: 0
创建索引后，再看一下这条语句的执行计划，具体如下：
mysql> explain select sum(moneys) from sales2 a,company2 b where a.company_id = b.id and
a.year = 2006\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: a
type: ref
possible_keys: ind_sales2_year
key: ind_sales2_year
key_len: 2
ref: const
rows: 1
Extra: Using where
*************************** 2. row ***************************
id: 1
select_type: SIMPLE
table: b
type: ref
possible_keys: ind_company2_id
key: ind_company2_id
key_len: 5
ref: sakila.a.company_id
rows: 1
Extra: Using where; Using index
2 rows in set (0.00 sec)
可以发现建立索引后对a 表需要扫描的行数明显减少（从1000 行减少到1 行），可见索引的
使用可以大大提高数据库的访问速度，尤其在表很庞大的时候这种优势更为明显。
Linux公社 www.linuxidc.com
208
## 18.2 索引问题
索引是数据库优化中最常用也是最重要的手段之一，通过索引通常可以帮助用户解决大多数
的SQL 性能问题。本节将对MySQL 中的索引的分类、存储、使用方法做详细的介绍。
### 18.2.1 索引的存储分类
MyISAM 存储引擎的表的数据和索引是自动分开存储的，各自是独立的一个文件；InnoDB
存储引擎的表的数据和索引是存储在同一个表空间里面，但可以有多个文件组成。
MySQL 中索引的存储类型目前只有两种（BTREE 和HASH），具体和表的存储引擎相关：
MyISAM 和InnoDB 存储引擎都只支持BTREE 索引；MEMORY/HEAP 存储引擎可以支持HASH
和BTREE 索引。
MySQL 目前不支持函数索引，但是能对列的前面某一部分进索引，例如name 字段，可
以只取name 的前4 个字符进行索引，这个特性可以大大缩小索引文件的大小，用户在设计
表结构的时候也可以对文本列根据此特性进行灵活设计。下面是创建前缀索引的一个例子：
mysql> create index ind_company2_name on company2(name(4));
Query OK, 1000 rows affected (0.03 sec)
Records: 1000 Duplicates: 0 Warnings: 0
### 18.2.2 MySQL 如何使用索引
索引用于快速找出在某个列中有一特定值的行。对相关列使用索引是提高SELECT 操作
性能的最佳途径。
查询要使用索引最主要的条件是查询条件中需要使用索引关键字，如果是多列索引，那
么只有查询条件使用了多列关键字最左边的前缀时，才可以使用索引，否则将不能使用索引。
1．使用索引
在MySQL 中，下列几种情况下有可能使用到索引。
（1）对于创建的多列索引，只要查询的条件中用到了最左边的列，索引一般就会被使用，
举例说明如下。
首先按company_id，moneys 的顺序创建一个复合索引，具体如下：
mysql> create index ind_sales2_companyid_moneys on sales2(company_id,moneys);
Query OK, 1000 rows affected (0.03 sec)
Records: 1000 Duplicates: 0 Warnings: 0
然后按company_id 进行表查询，具体如下：
mysql> explain select * from sales2 where company_id = 2006\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ref
possible_keys: ind_sales2_companyid_moneys
Linux公社 www.linuxidc.com
209
key: ind_sales2_companyid_moneys
key_len: 5
ref: const
rows: 1
Extra: Using where
1 row in set (0.00 sec)
可以发现即便where 条件中不是用的company_id 与moneys 的组合条件，索引仍然能
用到，这就是索引的前缀特性。但是如果只按moneys 条件查询表，那么索引就不会
被用到，具体如下：
mysql> explain select * from sales2 where moneys = 1\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
（2）对于使用like 的查询，后面如果是常量并且只有％号不在第一个字符，索引才可能会
被使用，来看下面两个执行计划：
mysql> explain select * from company2 where name like '%3'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
mysql> explain select * from company2 where name like '3%'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: range
Linux公社 www.linuxidc.com
210
possible_keys: ind_company2_name
key: ind_company2_name
key_len: 11
ref: NULL
rows: 103
Extra: Using where
1 row in set (0.00 sec)
可以发现第一个例子没有使用索引，而第二例子就能够使用索引，区别就在于“%”的位置
不同，前者把“%”放到第一位就不能用到索引，而后者没有放到第一位就使用了索引。
另外，如果如果like 后面跟的是一个列的名字，那么索引也不会被使用。
（3）如果对大的文本进行搜索，使用全文索引而不用使用like ‘%…%’。
（4）如果列名是索引，使用column_name is null 将使用索引。如下例中查询name 为null
的记录就用到了索引：
mysql> explain select * from company2 where name is null\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ref
possible_keys: ind_company2_name
key: ind_company2_name
key_len: 11
ref: const
rows: 1
Extra: Using where
1 row in set (0.00 sec)
2．存在索引但不使用索引
在下列情况下，虽然存在索引，但是MySQL 并不会使用相应的索引。
（1） 如果MySQL 估计使用索引比全表扫描更慢，则不使用索引。例如如果列
key_part1 均匀分布在1 和100 之间，下列查询中使用索引就不是很好：
SELECT * FROM table_name where key_part1 > 1 and key_part1 < 90;
（2） 如果使用MEMORY/HEAP 表并且where 条件中不使用“=”进行索引列，那么
不会用到索引。heap 表只有在“=”的条件下才会使用索引。
（3） 用or 分割开的条件，如果or 前的条件中的列有索引，而后面的列中没有索引，
那么涉及到的索引都不会被用到，例如：
mysql> show index from sales\G;
*************************** 1. row ***************************
Table: sales
Non_unique: 1
Key_name: ind_sales_year
Seq_in_index: 1
Column_name: year
Linux公社 www.linuxidc.com
211
Collation: A
Cardinality: NULL
Sub_part: NULL
Packed: NULL
Null:
Index_type: BTREE
Comment:
1 row in set (0.00 sec)
从上面可以发现只有year 列上面有索引，来看如下的执行计划：
mysql> explain select * from sales where year = 2001 or country = 'China'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales
type: ALL
possible_keys: ind_sales_year
key: NULL
key_len: NULL
ref: NULL
rows: 12
Extra: Using where
1 row in set (0.00 sec)
可见虽然在year 这个列上存在索引ind_sales_year，但是这个SQL 语句并没有用到这个索引，
原因就是or 中有一个条件中的列没有索引。
（4） 如果不是索引列的第一部分，如下例子：
mysql> explain select * from sales2 where moneys = 1\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
可见虽然在money 上面建有复合索引，但是由于money 不是索引的第一列，那么在查询中
这个索引也不会被MySQL 采用。
（5） 如果like 是以％开始，例如：
mysql> explain select * from company2 where name like '%3'\G;
*************************** 1. row ***************************
id: 1
Linux公社 www.linuxidc.com
212
select_type: SIMPLE
table: company2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
可见虽然在name 上建有索引，但是由于where 条件中like 的值的“%”在第一位了，那么
MySQL 也不会采用这个索引。
（6） 如果列类型是字符串，那么一定记得在where 条件中把字符常量值用引号引
起来，否则的话即便这个列上有索引，MySQL 也不会用到的，因为，MySQL 默认把输入的
常量值进行转换以后才进行检索。如下面的例子中company2 表中的name字段是字符型的，
但是SQL 语句中的条件值294 是一个数值型值，因此即便在name 上有索引，MySQL 也不能
正确地用上索引，而是继续进行全表扫描。
mysql> explain select * from company2 where name = 294\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ALL
possible_keys: ind_company2_name
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
mysql> explain select * from company2 where name = '294'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ref
possible_keys: ind_company2_name
key: ind_company2_name
key_len: 23
ref: const
rows: 1
Extra: Using where
1 row in set (0.00 sec)
Linux公社 www.linuxidc.com
213
从上面的例子中可以看到，第一个SQL 语句中把一个数值型常量赋值给了一个字符型的列
name，那么虽然在name 列上有索引，但是也没有用到；而第二个SQL 语句就可以正确使
用索引。
### 18.2.3 查看索引使用情况
如果索引正在工作，Handler_read_key 的值将很高，这个值代表了一个行被索引值读的
次数，很低的值表明增加索引得到的性能改善不高，因为索引并不经常使用。
Handler_read_rnd_next 的值高则意味着查询运行低效，并且应该建立索引补救。这个值
的含义是在数据文件中读下一行的请求数。如果正进行大量的表扫描，
Handler_read_rnd_next 的值较高，则通常说明表索引不正确或写入的查询没有利用索引，具
体如下。
mysql> show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name | Value |
+-----------------------+-------+
| Handler_read_first | 0 |
| Handler_read_key | 5 |
| Handler_read_next | 0 |
| Handler_read_prev | 0 |
| Handler_read_rnd | 0 |
| Handler_read_rnd_next | 2055 |
+-----------------------+-------+
6 rows in set (0.00 sec)
从上面的例子中可以看出，目前使用的MySQL 数据库的索引情况并不理想。
## 18.3 两个简单实用的优化方法
对于大多数开发人员来说，可能只希望掌握一些简单实用的优化方法，对于更多更复杂的优
化，更倾向于交给专业DBA 来做。本节将向大家介绍两个简单适用的优化方法。
### 18.3.1 定期分析表和检查表
分析表的语法如下：
ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
本语句用于分析和存储表的关键字分布，分析的结果将可以使得系统得到准确的统计信
息，使得SQL 能够生成正确的执行计划。如果用户感觉实际执行计划并不是预期的执行计
划，执行一次分析表可能会解决问题。在分析期间，使用一个读取锁定对表进行锁定。这对
于MyISAM, BDB 和InnoDB 表有作用。对于MyISAM 表，本语句与使用myisamchk -a 相当，
下例中对表sales 做了表分析：
mysql> analyze table sales;
+--------------+---------+----------+----------+
| Table | Op | Msg_type | Msg_text |
+--------------+---------+----------+----------+
Linux公社 www.linuxidc.com
214
| sakila.sales | analyze | status | OK |
+--------------+---------+----------+----------+
1 row in set (0.00 sec)
检查表的语法如下：
CHECK TABLE tbl_name [, tbl_name] ... [option] ... option = {QUICK | FAST | MEDIUM | EXTENDED
| CHANGED}
检查表的作用是检查一个或多个表是否有错误。CHECK TABLE 对MyISAM 和InnoDB 表有作用。
对于MyISAM 表，关键字统计数据被更新，例如：
mysql> check table sales;
+--------------+-------+----------+----------+
| Table | Op | Msg_type | Msg_text |
+--------------+-------+----------+----------+
| sakila.sales | check | status | OK |
+--------------+-------+----------+----------+
1 row in set (0.00 sec)
CHECK TABLE 也可以检查视图是否有错误，比如在视图定义中被引用的表已不存在，举例如
下。
（1）首先我们创建一个视图。
mysql> create view sales_view3 as select * from sales3;
Query OK, 0 rows affected (0.00 sec)
（2）然后CHECK 一下该视图，发现没有问题。
mysql> check table sales_view3;
+--------------------+-------+----------+----------+
| Table | Op | Msg_type | Msg_text |
+--------------------+-------+----------+----------+
| sakila.sales_view3 | check | status | OK |
+--------------------+-------+----------+----------+
1 row in set (0.00 sec)
（3）现在删除掉视图依赖的表。
mysql> drop table sales3;
Query OK, 0 rows affected (0.00 sec)
 （4）再来CHECK 一下刚才的视图，发现报错了。
mysql> check table sales_view3\G;
*************************** 1. row ***************************
Table: sakila.sales_view3
Op: check
Msg_type: error
Msg_text: View 'sakila.sales_view3' references invalid table(s) or column(s) or function(s)
or definer/invoker of view lack rights to use them
1 row in set (0.00 sec)
### 18.3.2 定期优化表
Linux公社 www.linuxidc.com
215
优化表的语法如下：
OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
如果已经删除了表的一大部分，或者如果已经对含有可变长度行的表（含有VARCHAR、
BLOB 或TEXT 列的表）进行了很多更改，则应使用OPTIMIZE TABLE 命令来进行表优化。这个
命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费，但
OPTIMIZE TABLE 命令只对MyISAM、BDB 和InnoDB 表起作用。
以下例子显示了优化表sales 的过程：
mysql> optimize table sales;
+--------------+----------+----------+----------+
| Table | Op | Msg_type | Msg_text |
+--------------+----------+----------+----------+
| sakila.sales | optimize | status | OK |
+--------------+----------+----------+----------+
1 row in set (0.00 sec)
注意：ANALYZE、CHECK、OPTIMIZE 执行期间将对表进行锁定，因此一定注意要在数据库不
繁忙的时候执行相关的操作。
## 18.4 常用SQL 的优化
前面我们介绍了MySQL 中怎么样通过索引来优化查询。日常开发中，除了使用查询外，我
们还会使用一些其他的常用SQL，比如INSERT、GROUP BY 等。对于这些SQL 语句，我们该
怎么样进行优化呢？本节将针对这些SQL 语句介绍一些优化的方法。
### 18.4.1 大批量插入数据
当用load 命令导入数据的时候，适当的设置可以提高导入的速度。
对于MyISAM 存储引擎的表，可以通过以下方式快速的导入大量的数据。
ALTER TABLE tbl_name DISABLE KEYS;
loading the data
ALTER TABLE tbl_name ENABLE KEYS;
DISABLE KEYS 和ENABLE KEYS 用来打开或者关闭MyISAM 表非唯一索引的更新。在导入
大量的数据到一个非空的MyISAM 表时，通过设置这两个命令，可以提高导入的效率。对于
导入大量数据到一个空的MyISAM 表，默认就是先导入数据然后才创建索引的，所以不用进
行设置。
下面例子中，用LOAD 语句导入数据耗时115.12 秒：
mysql> load data infile '/home/mysql/film_test.txt' into table film_test2;
Query OK, 529056 rows affected (1 min 55.12 sec)
Records: 529056 Deleted: 0 Skipped: 0 Warnings: 0
而用alter table tbl_name disable keys 方式总耗时6.34 + 12.25 = 18.59 秒，提高了6 倍多。
mysql> alter table film_test2 disable keys;
Query OK, 0 rows affected (0.00 sec)
Linux公社 www.linuxidc.com
216
mysql> load data infile '/home/mysql/film_test.txt' into table film_test2;
Query OK, 529056 rows affected (6.34 sec)
Records: 529056 Deleted: 0 Skipped: 0 Warnings: 0
mysql> alter table film_test2 enable keys;
Query OK, 0 rows affected (12.25 sec)
上面是对MyISAM表进行数据导入时的优化措施，对于InnoDB类型的表，这种方式并不
能提高导入数据的效率，可以有以下几种方式提高InnoDB表的导入效率。
（1）因为InnoDB 类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺
序排列，可以有效地提高导入数据的效率。
例如，下面文本film_test3.txt是按表film_test4的主键存储的，那么导入的时候共耗时
27.92秒。
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (22.92 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
而下面的film_test4.txt 是没有任何顺序的文本，那么导入的时候共耗时31.16 秒。
mysql> load data infile '/home/mysql/film_test4.txt' into table film_test4;
Query OK, 1587168 rows affected (31.16 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
从上面例子可以看出当被导入的文件按表主键顺序存储的时候比不按主键顺序存储的时候
快1.12 倍。
（2）在导入数据前执行SET UNIQUE_CHECKS=0，关闭唯一性校验，在导入结束后执行
SET UNIQUE_CHECKS=1，恢复唯一性校验，可以提高导入的效率。
例如，当UNIQUE_CHECKS=1 时：
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (22.92 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
当SET UNIQUE_CHECKS=0 时：
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (19.92 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
可见比UNIQUE_CHECKS=0 的时候比SET UNIQUE_CHECKS=1 的时候要快一些。
（3）如果应用使用自动提交的方式，建议在导入前执行SET AUTOCOMMIT=0，关闭自
动提交，导入结束后再执行SET AUTOCOMMIT=1，打开自动提交，也可以提高导入的效率。
例如，当AUTOCOMMIT=1 时：
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (22.92 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
当AUTOCOMMIT=0 时：
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (20.87 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
对比一下可以知道，当AUTOCOMMIT=0 时比AUTOCOMMIT=1 时导入数据要快一些。
Linux公社 www.linuxidc.com
欢迎点击这里的链接进入精彩的Linux公社 网站
Linux公社（www.Linuxidc.com）于2006年9月25日注册并开通网站，Linux现在已经成
为一种广受关注和支持的一种操作系统，IDC是互联网数据中心，LinuxIDC就是关于
Linux的数据中心。
Linux公社是专业的Linux系统门户网站，实时发布最新Linux资讯，包括Linux、
Ubuntu、Fedora、RedHat、红旗Linux、Linux教程、Linux认证、SUSE Linux、
Android、Oracle、Hadoop、CentOS、MySQL、Apache、Nginx、Tomcat、Python、
Java、C语言、OpenStack、集群等技术。
Linux公社（LinuxIDC.com）设置了有一定影响力的Linux专题栏目。
Linux公社 主站网址：www.linuxidc.com 旗下网站：www.linuxidc.net
包括：Ubuntu 专题 Fedora 专题 Android 专题 Oracle 专题 Hadoop 专题
RedHat 专题 SUSE 专题 红旗Linux 专题 CentOS 专题
Linux 公社微信公众号：linuxidc_com
217
### 18.4.2 优化INSERT 语句
当进行数据INSERT 的时候，可以考虑采用以下几种优化方式。
 如果同时从同一客户插入很多行，尽量使用多个值表的INSERT 语句，这种方式将大大
缩减客户端与数据库之间的连接、关闭等消耗，使得效率比分开执行的单个INSERT 语
句快(在一些情况中几倍)。下面是一次插入多值的一个例子：
insert into test values(1,2),(1,3),(1,4)…
 如果从不同客户插入很多行，能通过使用INSERT DELAYED 语句得到更高的速度。
DELAYED 的含义是让INSERT 语句马上执行，其实数据都被放在内存的队列中，并没有
真正写入磁盘，这比每条语句分别插入要快的多；LOW_PRIORITY 刚好相反，在所有其
他用户对表的读写完后才进行插入；
 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）；
 如果进行批量插入，可以增加bulk_insert_buffer_size 变量值的方法来提高速度，但是，
这只能对MyISAM 表使用；
 当从一个文本文件装载一个表时，使用LOAD DATA INFILE。这通常比使用很多INSERT 语
句快20 倍。
### 18.4.3 优化GROUP BY 语句
默认情况下，MySQL 对所有GROUP BY col1，col2....的字段进行排序。这与在查询中指定
ORDER BY col1，col2...类似。因此，如果显式包括一个包含相同的列的ORDER BY 子句，则
对MySQL 的实际执行性能没有什么影响。
如果查询包括GROUP BY 但用户想要避免排序结果的消耗，则可以指定ORDER BY NULL
禁止排序，如下面的例子：
mysql> explain select id,sum(moneys) from sales2 group by id\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using temporary; Using filesort
1 row in set (0.00 sec)
mysql> explain select id,sum(moneys) from sales2 group by id order by null\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: NULL
Linux公社 www.linuxidc.com
218
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using temporary
1 row in set (0.00 sec)
从上面的例子可以看出第一个SQL 语句需要进行“filesort”，而第二个SQL 由于ORDER BY NULL
不需要进行“filesort”，而filesort 往往非常耗费时间。
### 18.4.4 优化ORDER BY 语句：
在某些情况中，MySQL 可以使用一个索引来满足ORDER BY 子句，而不需要额外的排序。
WHERE 条件和ORDER BY 使用相同的索引，并且ORDER BY 的顺序和索引顺序相同，并且
ORDER BY 的字段都是升序或者都是降序。
例如，下列SQL 可以使用索引。
SELECT * FROM t1 ORDER BY key_part1,key_part2,... ;
SELECT * FROM t1 WHERE key_part1=1 ORDER BY key_part1 DESC, key_part2 DESC;
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 DESC;
但是在以下几种情况下则不使用索引：
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC；
--order by 的字段混合ASC 和DESC
SELECT * FROM t1 WHERE key2=constant ORDER BY key1；
--用于查询行的关键字与ORDER BY 中所使用的不相同
SELECT * FROM t1 ORDER BY key1, key2；
--对不同的关键字使用ORDER BY：
### 18.4.5 优化嵌套查询
MySQL 4.1 开始支持SQL 的子查询。这个技术可以使用SELECT 语句来创建一个单列的查
询结果，然后把这个结果作为过滤条件用在另一个查询中。使用子查询可以一次性地完成很
多逻辑上需要多个步骤才能完成的SQL 操作，同时也可以避免事务或者表锁死，并且写起
来也很容易。但是，有些情况下，子查询可以被更有效率的连接（JOIN）替代。
在下面的例子中，要从sales2 表中找到那些在company2 表中不存在的所有公司的信息：
mysql> explain select * from sales2 where company_id not in ( select id from
company2 )\G;
*************************** 1. row ***************************
id: 1
select_type: PRIMARY
table: sales2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
Linux公社 www.linuxidc.com
219
rows: 1000
Extra: Using where
*************************** 2. row ***************************
id: 2
select_type: DEPENDENT SUBQUERY
table: company2
type: index_subquery
possible_keys: ind_company2_id
key: ind_company2_id
key_len: 5
ref: func
rows: 2
Extra: Using index
2 rows in set (0.00 sec)
如果使用连接（JOIN）来完成这个查询工作，速度将会快很多。尤其是当company2 表
中对id 建有索引的话，性能将会更好，具体查询如下：
mysql> explain select * from sales2 left join company2 on sales2.company_id =
company2.id where sales2.company_id is null\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ref
possible_keys: ind_sales2_companyid_moneys
key: ind_sales2_companyid_moneys
key_len: 5
ref: const
rows: 1
Extra: Using where
*************************** 2. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ref
possible_keys: ind_company2_id
key: ind_company2_id
key_len: 5
ref: sakila.sales2.company_id
rows: 1
Extra:
2 rows in set (0.00 sec)
从执行计划中可以明显看出查询扫描的记录范围和使用索引的情况都有了很大的改善。
连接（JOIN）之所以更有效率一些，是因为MySQL 不需要在内存中创建临时表来完成这
个逻辑上的需要两个步骤的查询工作。
Linux公社 www.linuxidc.com
220
### 18.4.6 MySQL 如何优化OR 条件
对于含有OR 的查询子句，如果要利用索引，则OR 之间的每个条件列都必须用到索引；
如果没有索引，则应该考虑增加索引。
例如，首先使用show index 命令查看表sales2 的索引，可知它有3 个索引，在id、year
两个字段上分别有1 个独立的索引，在company_id 和year 字段上有1 个复合索引。
mysql> show index from sales2\G;
*************************** 1. row ***************************
Table: sales2
Non_unique: 1
Key_name: ind_sales2_id
Seq_in_index: 1
Column_name: id
Collation: A
Cardinality: 1000
Sub_part: NULL
Packed: NULL
Null: YES
Index_type: BTREE
Comment:
*************************** 2. row ***************************
Table: sales2
Non_unique: 1
Key_name: ind_sales2_year
Seq_in_index: 1
Column_name: year
Collation: A
Cardinality: 250
Sub_part: NULL
Packed: NULL
Null: YES
Index_type: BTREE
Comment:
*************************** 3. row ***************************
Table: sales2
Non_unique: 1
Key_name: ind_sales2_companyid_moneys
Seq_in_index: 1
Column_name: company_id
Collation: A
Cardinality: 1000
Sub_part: NULL
Packed: NULL
Null: YES
Linux公社 www.linuxidc.com
221
Index_type: BTREE
Comment:
*************************** 4. row ***************************
Table: sales2
Non_unique: 1
Key_name: ind_sales2_companyid_moneys
Seq_in_index: 2
Column_name: year
Collation: A
Cardinality: 1000
Sub_part: NULL
Packed: NULL
Null: YES
Index_type: BTREE
Comment:
4 rows in set (0.00 sec)
然后在两个独立索引上面做OR 操作，具体如下：
mysql> explain select * from sales2 where id = 2 or year = 1998\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: index_merge
possible_keys: ind_sales2_id,ind_sales2_year
key: ind_sales2_id,ind_sales2_year
key_len: 5,2
ref: NULL
rows: 2
Extra: Using union(ind_sales2_id,ind_sales2_year); Using where
1 row in set (0.00 sec)
可以发现查询正确的用到了索引，并且从执行计划的描述中，发现MySQL 在处理含有OR
字句的查询时，实际是对OR 的各个字段分别查询后的结果进行了UNION。
但是当在建有复合索引的列company_id 和moneys 上面做OR 操作的时候，却不能用到索引，
具体结果如下：
mysql> explain select * from sales2 where company_id = 3 or moneys = 100\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: ind_sales2_companyid_moneys
key: NULL
key_len: NULL
ref: NULL
Linux公社 www.linuxidc.com
222
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
18.4.7 使用SQL 提示
SQL 提示（SQL HINT）是优化数据库的一个重要手段，简单来说就是在SQL 语句中加入一些
人为的提示来达到优化操作的目的。
下面是一个使用SQL 提示的例子：
SELECT SQL_BUFFER_RESULTS * FROM...
这个语句将强制MySQL 生成一个临时结果集。只要临时结果集生成后，所有表上的锁
定均被释放。这能在遇到表锁定问题时或要花很长时间将结果传给客户端时有所帮助，因为
可以尽快释放锁资源。
下面是一些在MySQL 中常用的SQL 提示。
1．USE INDEX
在查询语句中表名的后面，添加USE INDEX 来提供希望MySQL 去参考的索引列表，就可
以让MySQL 不再考虑其他可用的索引。
mysql> explain select * from sales2 use index (ind_sales2_id) where id = 3\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ref
possible_keys: ind_sales2_id
key: ind_sales2_id
key_len: 5
ref: const
rows: 1
Extra: Using where
1 row in set (0.00 sec).
2．IGNORE INDEX
如果用户只是单纯地想让MySQL 忽略一个或者多个索引，则可以使用IGNORE INDEX 作
为HINT。同样是上面的例子，这次来看一下查询过程忽略索引ind_sales2_id 的情况：
mysql> explain select * from sales2 ignore index (ind_sales2_id) where id = 3\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
Linux公社 www.linuxidc.com
223
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec).
从执行计划可以看出，系统忽略了指定的索引，而使用了全表扫描。
3．FORCE INDEX
为强制MySQL 使用一个特定的索引，可在查询中使用FORCE INDEX 作为HINT。例如，
当不强制使用索引的时候，因为id 的值都是大于0 的，因此MySQL 会默认进行全表扫描，
而不使用索引，如下所示：
mysql> explain select * from sales2 where id > 0 \G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ALL
possible_keys: ind_sales2_id
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
但是，当使用FORCE INDEX 进行提示时，即便使用索引的效率不是最高，MySQL 还是选择使
用了索引，这是MySQL 留给用户的一个自行选择执行计划的权力。加入FORCE INDEX 提示
后再次执行上面的SQL：
mysql> explain select * from sales2 force index (ind_sales2_id) where id > 0
\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: range
possible_keys: ind_sales2_id
key: ind_sales2_id
key_len: 5
ref: NULL
rows: 1000
Linux公社 www.linuxidc.com
224
Extra: Using where
1 row in set (0.00 sec).
果然，执行计划中使用了FORCE INDEX 后的索引。
18.5 小结
SQL 优化问题是数据库性能优化最基础也是最重要的一个问题，实践表明很多数据库性能问
题都是由不合适的SQL 语句造成。本章通过实例描述了SQL 优化的一般过程，从定位一个
有性能问题的SQL 语句到分析产生性能问题的原因，最后到采取什么措施优化SQL 语句的
性能。另外还介绍了优化SQL 语句经常需要考虑的几个方面，比如索引、表分析、排序等。
