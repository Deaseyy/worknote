# 十，SQL优化

## 10.1 优化SQL 语句的一般步骤

#### 1.通过show status 命令了解各种sql的执行频率

```mysql
# 查看global级（自数据库上次启动至今），session级（当前连接）
show [session|global] status；  # 获取服务器状态信息
show status like 'Com_%'; # Com_xxx表示查询每个xxx语句执行的次数
```

- Com_select：执行select 操作的次数，一次查询只累加1。
- Com_insert：执行INSERT 操作的次数，对于批量插入的INSERT 操作，只累加一次。
- Com_update：执行UPDATE 操作的次数。
- Com_delete：执行DELETE 操作的次数。

#### 2.定位执行效率较低的SQL语句

#### 3.通过explain分析低效SQL的执行计划

```
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

```

每个列的简单解释如下：

- select_type：表示SELECT 的类型，常见的取值有SIMPLE（简单表，即不使用表连接
  或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或
  者后面的查询语句）、SUBQUERY（子查询中的第一个SELECT）等。
- table：输出结果集的表。
- type：表示表的连接类型，性能由好到差的连接类型为system（表中仅有一行，即
  常量表）、const（单表中最多有一个匹配行，例如primary key 或者unique index）、
  eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接
  中使用primary key或者unique index）、re（f 与eq_ref类似，区别在于不是使用primary
  key 或者unique index，而是使用普通的索引）、ref_or_null（与ref 类似，区别在于
  条件中包含对NULL 的查询）、index_merge(索引合并优化)、unique_subquery（in
  的后面是一个查询主键字段的子查询）、index_subquery（与unique_subquery 类似，
  区别在于in 的后面是查询非唯一索引字段的子查询）、range（单表中的范围查询）、
  index（对于前面的每一行，都通过查询索引来得到数据）、all（对于前面的每一行，都通过全表扫描来得到数据）。
- possible_keys：表示查询时，可能使用的索引。
- key：表示实际使用的索引。
- key_len：索引字段的长度。
- rows：扫描行的数量。
- Extra：执行情况的说明和描述。

#### 4.确定问题并采取相应的优化措施

```
比如发现type值为all，说明是对表的全表扫描导致效率不理想，对筛选字段建立索引:

```

```
create index ind_sales2_year on sales2(year);

```

```
可以发现建立索引后需要扫描的行数（rows）明显减少（从1000 行减少到1 行）;

```

## 10.2 索引问题

```
索引用于快速找出在某个列中有一特定值的行。对相关列使用索引是提高SELECT 操作

```

性能的最佳途径。
	查询要使用索引最主要的条件是查询条件中需要使用索引关键字，如果是多列索引，那
么只有查询条件使用了多列关键字最左边的前缀时，才可以使用索引，否则将不能使用索引。

### 使用索引

```
在MySQL 中，下列几种情况下有可能使用到索引。

```

**1.对于创建的多列索引，只要查询的条件中用到了最左边的列，索引一般就会被使用**

```mysql
# 先按company_id，moneys 的顺序创建一个复合索引
create index ind_sales2_companyid_moneys on sales2(company_id,moneys);
# 然后按company_id 进行表查询
explain select * from sales2 where company_id = 2006\G;
# 可以发现即便where 条件中不是用的company_id 与moneys 的组合条件，索引仍然能用到，这就是索引的前缀特性。但是如果只按moneys 条件查询表，那么索引就不会被用到,如下：
explain select * from sales2 where moneys = 1\G;
```

**2.对于like 的查询，后面如果是常量并且只有％号不在第一个字符，索引才可能会被使用**

```mysql
# 会使用索引
explain select * from company2 where name like '%3'\G;
# 不会使用索引
explain select * from company2 where name like '3%'\G;
```

```
另外，如果如果like 后面跟的是一个列的名字，那么索引也不会被使用

```

**3.如果对大的文本进行搜索，使用全文索引而不用使用like ‘%…%’**

**4.如果列名是索引，使用column_name is null 将使用索引。**

### 存在索引但不使用索引

**1.如果MySQL 估计使用索引比全表扫描更慢，则不使用索引**

```mysql
# 如果列key_part1 均匀分布在1 和100 之间，下列查询中使用索引就不是很好：
SELECT * FROM table_name where key_part1 > 1 and key_part1 < 90;
```

**2.如果使用MEMORY/HEAP 表并且where 条件中不使用“=”进行索引列，那么不会用到索引。heap 表只有在“=”的条件下才会使用索引。**

**3.用or 分割开的条件，如果or 前的条件中的列有索引，而后面的列中没有索引，那么涉及到的索引都不会被用到**。

**4.虽然列上建立了复合索引，但若该列不是复合索引列的第一列，这个索引也不会被使用。**

**5.如果like 的值是以％开始，虽然在name 上建有索引，也不会被使用。**

**6.如果列类型是字符串，必须在where 条件中把字符常量值用引号引起来，否则即便这个列上有索引也不会用到的，因为MySQL 默认把输入的常量值进行转换以后才进行检索。**



## 10.3两个简单实用的优化方法

### 1.定期分析表和检查表

分析表的语法如下：

```
ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...

```

检查表的语法如下：

```
CHECK TABLE tbl_name [, tbl_name] ... [option] ... option = {QUICK | FAST | MEDIUM | EXTENDED| CHANGED}

```

### 2.定期优化表

优化表的语法如下：

```
OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...

```

```
这个命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费，但OPTIMIZE TABLE 命令只对MyISAM、BDB 和InnoDB 表起作用。

```



## 10.4 常用SQL 的优化

### 1.大批量插入数据

```
当用load 命令导入数据的时候，适当的设置可以提高导入的速度。

```

对于MyISAM 存储引擎的表，可以通过以下方式快速的导入大量的数据:

```mysql
ALTER TABLE tbl_name DISABLE KEYS;
load data infile '文件地址' into table B;
ALTER TABLE tbl_name ENABLE KEYS;
```

```
DISABLE KEYS 和ENABLE KEYS 用来打开或者关闭MyISAM 表非唯一索引的更新。在导入大量的数据到一个非空的MyISAM 表时，通过设置这两个命令，可以提高导入的效率。对于导入大量数据到一个空的MyISAM 表，默认就是先导入数据然后才创建索引的，所以不用进行设置。

```

对于InnoDB类型的表:

- 因为InnoDB 类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺序排列，可以有效地提高导入数据的效率。
- 2）在导入数据前执行SET UNIQUE_CHECKS=0，关闭唯一性校验，在导入结束后执行
  SET UNIQUE_CHECKS=1，恢复唯一性校验，可以提高导入的效率。
- 如果应用使用自动提交的方式，建议在导入前执行SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行SET AUTOCOMMIT=1，打开自动提交，也可以提高导入的效率

### 2.优化INSERT 语句

- 如果同时从同一客户插入很多行，尽量使用多个值INSERT 语句，能缩减客户端与数据库之间的连接、关闭等消耗，使得效率比分开执行的单个INSERT 语句快(在一些情况中几倍)。下面是一次插入多值的一个例子：
  insert into test values(1,2),(1,3),(1,4)…
- 如果从不同客户插入很多行，能通过使用INSERT DELAYED 语句得到更高的速度。
  DELAYED 的含义是让INSERT 语句马上执行，其实数据都被放在内存的队列中，并没有
  真正写入磁盘，这比每条语句分别插入要快的多；LOW_PRIORITY 刚好相反，在所有其
  他用户对表的读写完后才进行插入；
- 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）；
- 如果进行批量插入，可以增加bulk_insert_buffer_size 变量值的方法来提高速度，但是，
  这只能对MyISAM 表使用；
- 当从一个文本文件装载一个表时，使用LOAD DATA INFILE。这通常比使用很多INSERT 语
  句快20 倍。

### 3 .优化GROUP BY 语句

```
MySQL 默认对所有GROUP BY col1，col2....的字段进行排序。这与在查询中指定ORDER BY col1，col2...类似。因此，如果显式包括一个包含相同的列的ORDER BY 子句，则对MySQL 的实际执行性能没有什么影响。

```

如果查询包括GROUP BY 但用户想要避免排序结果的消耗，则可以指定ORDER BY NULL禁止排序；

例： explain select id,sum(moneys) from sales2 group by id order by null \G；

### 4.优化ORDER BY 语句

```
在某些情况中，MySQL 可以使用一个索引来满足ORDER BY 子句，而不需要额外的排序。WHERE 条件和ORDER BY 使用相同的索引，并且ORDER BY 的顺序和索引顺序相同，并且ORDER BY 的字段都是升序或者都是降序。

```

例如，下列SQL 可以使用索引。

```mysql
SELECT * FROM t1 ORDER BY key_part1,key_part2,... ;
SELECT * FROM t1 WHERE key_part1=1 ORDER BY key_part1 DESC, key_part2 DESC;
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 DESC;
```

但是在以下几种情况下则不使用索引：

```mysql
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC；
-- order by 的字段混合ASC 和DESC
SELECT * FROM t1 WHERE key2=constant ORDER BY key1；
-- 用于查询行的关键字与ORDER BY 中所使用的不相同
SELECT * FROM t1 ORDER BY key1, key2；
-- 对不同的关键字使用ORDER BY
```

### 5.优化嵌套查询

```
使用子查询可以一次性地完成很多逻辑上需要多个步骤才能完成的SQL 操作，同时也可以避免事务或者表锁死。有些情况下，子查询可以被更有效率的连接（JOIN）替代:

```

```mysql
# not in 转化为 left join
select * from A where A.id not in (select id from B);
优化为：
select * from A left join B on A.id=B.id where B.id is null；
```

```
连接（JOIN）之所以更有效率一些，是因为MySQL 不需要在内存中创建临时表来完成这个逻辑上的需要两个步骤的查询工作。

```

### 6.优化OR 条件

```
对于含有OR 的查询子句，如果要利用索引，则OR 的每个条件列都必须用到索引；如果没有索引，则应该考虑增加索引。

```

例：表sales2 上有3 个索引，在id、year两个字段上分别有1 个独立的索引，在company_id 和year 字段上有1 个复合索引。

```mysql
# 正确使用到了索引
explain select * from sales2 where id = 2 or year = 1998\G;
# 未使用到索引
explain select * from sales2 where company_id = 3 or moneys = 100\G;
```

### 7 .使用SQL 提示

```
简单来说就是在SQL 语句中加入一些人为的提示来达到优化操作的目的。例如：

```

```
SELECT SQL_BUFFER_RESULTS * FROM...

```

```
这个语句将强制MySQL 生成一个临时结果集。只要临时结果集生成后，所有表上的锁定均被释放。这能在遇到表锁定问题时或要花很长时间将结果传给客户端时有所帮助，因为可以尽快释放锁资源。

```

下面是一些在MySQL 中常用的SQL 提示。

- USE INDEX

  在查询语句中表名的后面，添加USE INDEX 来提供希望MySQL 去参考的索引列表，就可以让MySQL 不再考虑其他可用的索引。

```mysql
explain select * from sales2 use index (ind_sales2_id) where id = 3\G;
```

- IGNORE INDEX

  如果只是单纯地想让MySQL 忽略一个或者多个索引，则可以使用IGNORE INDEX 作为HINT

```mysql
explain select * from sales2 ignore index (ind_sales2_id) where id = 3\G;
```

- FORCE INDEX

  为强制MySQL 使用一个特定的索引，可在查询中使用FORCE INDEX 作为HINT。例如，当不强制使用索引的时候，因为id 的值都是大于0 的，因此MySQL 会默认进行全表扫描，而不使用索引.

```mysql
explain select * from sales2 where id > 0 \G;
```

## 10.5 常用SQL优化2

- 对查询进行优化，尽量避免全表扫描，首先考虑在 where 及 order by 涉及的列上建立索引。
- 尽量避免在 where 子句中对字段进行 null 判断，否则将导致引擎放弃索引而全表扫描。

```mysql
select id from t where num is null;
# 若是可以，在 num 上设置默认值 0,确保表中 num 列没有 null 值，然后查询：
select id from t where num=0;
```

- 尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃索引而全表扫描。

```mysql
select id from t where num=10 or num=20;
# 可使用 union ：
select id from t where num=10 union all select id from t where num=20;
```

- in 和 not in 也要慎用，否则会导致全表扫描，对于连续的数值，能用 between 就不要用 in 了，还可以使用 join 连接查询。
- 很多时候用 exists 代替 in 是一个好的选择。

```mysql
select num from a where num in(select num from b);
# 可替换为：
select num from a where exists(select 1 from b where num=a.num);
```

- 如果 where 子句中使用参数，也会导致全表扫描，因为 SQL 只有在运行时才会解析局部变量。

```mysql
select id from t where num=@num ;
# 可以改为强制查询使用索引：
select id from t with(index(索引名)) where num=@num ;
```

- 尽量避免在 where 子句中对字段进行表达式操作， 这将导致引擎放弃索引而全表扫描。

```mysql
select id from t where num/2=100;
# 可以这样查：
select id from t where num=100*2;
```

- 尽量避免在 where 子句中对字段进行函数操作，这将导致引擎放弃索引而全表扫描。

```mysql
select id from t where substring(name,1,3)='abc';#name 以 abc 开头的 id
# 可用：
select id from t where name like 'abc%';
```

- 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用 索引。
- 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件 时才能保证系统使用该索引， 否则该索引将不会 被使用， 并且应尽可能的让字段顺序与索引顺序相一致。
- 并不是所有索引对查询都有效，SQL 是根据表中数据来进行查询优化的，当索引列有大量数据重复时， SQL 查询可能不会去利用索引，如一表中有字段 male、female 几乎各一半，那么即使在上建 了索引也对查询效率起不了作用。
- 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过 6 个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。
- 应尽可能的避免更新 clustered 索引数据列， 因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。
- 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并 会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言 只需要比较一次就够了。
- 尽可能的使用 varchar/nvarchar 代替 char/nchar , 因为首先变长字段存储空间小， 可以节省存储空间， 其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。
- 任何地方都不要使用 select * from t ,用具体的字段列表代替“*”,不要返回用不到的任何字段。
- 尽量使用表变量(mysql没有)来代替临时表。如果表变量包含大量数据，请注意索引非常有限(只有主键索引)。
- 避免频繁创建和删除临时表，以减少系统表资源的消耗。
- 临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件， 最好使用导出表。
- 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table,避免造成大量 log ,以提高速度;如果数据量不大，为了缓和系统表的资源，应先 create table,然后 insert.
- 如果使用到了临时表， 在存储过程的最后务必将所有的临时表显式删除， 先 truncate table ,然后 drop table ,这样可以避免系统表的较长时间锁定。
- 尽量避免使用游标，因游标的效率较差，如果游标操作的数据超过 1 万行，那么就应该考虑改写。
- 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更 有效。
- 与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，尤其是在必须引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。
- 在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ,在结束时设置 SET NOCOUNT OFF .无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。
- 尽量避免大事务操作，提高系统并发能力。 sql 优化方法使用索引来更快地遍历表。 缺省情况下建立的索引是非群集索引，但有时它并不是最佳的。在非群集索引下，数据在物理上随机存放在数据页上。合理的索引设计要建立在对各种查询的分析和预测上。一般来说：

a.有大量重复值、且经常有范围查询( > ,< ,> =,< =)和 order by、group by 发生的列，可考虑建立集群索引;

b.经常同时存取多列，且每列都含有重复值可考虑建立组合索引;

c.组合索引要尽量使关键查询形成索引覆盖，其前导列一定是使用最频繁的列。索引虽有助于提高性能但 不是索引越多越好，恰好相反过多的索引会导致系统低效。用户在表中每加进一个索引，维护索引集合就 要做相应的更新工作。

30.定期分析表和检查表。

```
分析表的语法：ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tb1_name[, tbl_name]...

```

以上语句用于分析和存储表的关键字分布，分析的结果将可以使得系统得到准确的统计信息，使得SQL能够生成正确的执行计划。如果用户感觉实际执行计划并不是预期的执行计划，执行一次分析表可能会解决问题。在分析期间，使用一个读取锁定对表进行锁定。这对于MyISAM，DBD和InnoDB表有作用。

```
例如分析一个数据表：analyze table table_name
检查表的语法：CHECK TABLE tb1_name[,tbl_name]...[option]...option = {QUICK | FAST | MEDIUM | EXTENDED | CHANGED}

```

检查表的作用是检查一个或多个表是否有错误，CHECK TABLE 对MyISAM 和 InnoDB表有作用，对于MyISAM表，关键字统计数据被更新

CHECK TABLE 也可以检查视图是否有错误，比如在视图定义中被引用的表不存在。

31.定期优化表。

```
优化表的语法：OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tb1_name [,tbl_name]...

```

如果删除了表的一大部分，或者如果已经对含有可变长度行的表(含有 VARCHAR、BLOB或TEXT列的表)进行更多更改，则应使用OPTIMIZE TABLE命令来进行表优化。这个命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费，但OPTIMIZE TABLE 命令只对MyISAM、 BDB 和InnoDB表起作用。

```
例如： optimize table table_name

```

注意： analyze、check、optimize执行期间将对表进行锁定，因此一定注意要在MySQL数据库不繁忙的时候执行相关的操作。

> 补充：
>
> 1、在海量查询时尽量少用格式转换。
>
> 2、ORDER BY 和 GROPU BY:使用 ORDER BY 和 GROUP BY 短语，任何一种索引都有助于 SELECT 的性能提高。
>
> 3、任何对列的操作都将导致表扫描，它包括数据库教程函数、计算表达式等等，查询时要尽可能将操作移 至等号右边。
>
> 4、IN、OR 子句常会使用工作表，使索引失效。如果不产生大量重复值，可以考虑把子句拆开。拆开的子 句中应该包含索引。
>
> 5、只要能满足你的需求，应尽可能使用更小的数据类型：例如使用 MEDIUMINT 代替 INT
>
> 6、尽量把所有的列设置为 NOT NULL,如果你要保存 NULL,手动去设置它，而不是把它设为默认值。
>
> 7、尽量少用 VARCHAR、TEXT、BLOB 类型
>
> 8、如果你的数据只有你所知的少量的几个。最好使用 ENUM 类型
>
> 9、正如 graymice 所讲的那样，建立索引。
>
> 10、合理用运分表与分区表提高数据存放和提取速度。



# 十一,优化数据库对象

## 11.1 优化表的数据类型

可以使用函数PROCEDURE ANALYSE()对当前应用的表进行分析，该函数可以对数据表中列的数据类型提出优化建议。以下是函数PROCEDURE ANALYSE()的使用方法：

```mysql
SELECT * FROM tbl_name PROCEDURE ANALYSE();
# 不要为那些包含的值多于16 个或者256 字节的ENUM 类型提出建议
SELECT * FROM tbl_name PROCEDURE ANALYSE(16,256);
```

## 11.2 通过拆分提高表的访问效率

## 11.3使用中间表提高统计查询速度

例：有一客户消费记录表，需求：：希望了解最近一周客户的消费总金额和近一周每天不同时段用户的消费总金额。

```mysql
# 使用中间表
insert into tmp_session select * from session where cust_date > adddate(now(),-7);
```

另外，针对业务部门想了解“近一周每天不同时段用户的消费总金额”这一需求，在中间表上给出统计结果更为合适,原因是源数据表(session 表)cust_date 字段没有索引并且源表的数据量较大，所以在按时间进行分时段统计时效率很低，这时可以在中间表上对cust_date 字段创建单独的索引来提高统计查询的速度。
中间表在统计查询中经常会用到，其优点如下:

- 中间表复制源表部分数据，并且与源表相“隔离”，在中间表上做统计查询不
  会对在线应用产生负面影响。
- 中间表上可以灵活的添加索引或增加临时用的新字段,从而达到提高统计查询
  效率和辅助统计查询作用。





# 二十，其他问题

### 1. windows MySQL的cmd黑屏终端显示中文乱码问题

Windows cmd写入显示的编码是GBK，与mysql配置不一样，解决方法：

```mysql
mysql命令行下输入：
set character_set_client=gbk;
set character_set_results=gbk;
```

### 2. Mysql删除数据什么情况下会释放空间

1、drop table table_name 立刻释放磁盘空间 ，不管是 Innodb和MyISAM

2、truncate table table_name 立刻释放磁盘空间 ，不管是 Innodb和MyISAM 

3、delete from table_name  对于MyISAM 会立刻释放磁盘空间，InnoDB 不会释放磁盘空间 

4、对于delete from table_name where xxx带条件的删除，innodb和MyISAM都不会释放磁盘空间

5、**delete操作以后 使用optimize table table_name 会立刻释放磁盘空间**。不管是innodb还是myisam 。

6、delete from表以后虽然未释放磁盘空间，但是下次插入数据的时候，仍然可以使用这部分空间。

**注意：由于命令optimize会进行锁表操作，所以进行优化时要避开表数据操作时间，避免影响正常业务的进行。**

### 3.常用相关信息查询语句

```mysql
-- 1.查询指定数据库指定表的相关信息
show table status from pyvboard where name = 't_summarizewarning'
```



### 4.mysql何时使用行锁，何时使用表锁

- mysql的行锁其实锁的不是记录，而是给索引加锁
- 当操作表中的数据时，如果条件中未使用到索引，则会锁住整个表
- 如果条件使用到索引，则会使用行锁
- 所以需要提高并发，则必须给修改等操作的筛选条件加上索引



### 5.EXPLAIN命令查询结果的字段含义

```
通过以上步骤查询到效率低的 SQL 语句后，可以通过 EXPLAIN 或者 DESC 命令获取 MySQL
如何执行 SELECT 语句的信息，包括在 SELECT 语句执行过程中表如何连接和连接的顺序，比
如想计算 2006 年所有公司的销售额，需要关联 sales 表和 company 表，并且对 moneys 字段
做求和（sum）操作，相应 SQL 的执行计划如下：
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


```

**每个列的简单解释如下：**

- select_type：表示 SELECT 的类型，常见的取值有：
  - SIMPLE（简单表，即不使用表连接或者子查询）
  - PRIMARY（主查询，即外层的查询）
  - UNION（UNION 中的第二个或者后面的查询语句）
  - SUBQUERY（子查询中的第一个 SELECT）等。
- table：输出结果集的表。
- type：表示表的连接类型，性能由好到差的连接类型为：
  - system（表中仅有一行，即常量表）
  - const（单表中最多有一个匹配行，例如 primary key 或者 unique index）
  - eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接中使用primary key或者unique index）
  - ref（与eq_ref类似，区别在于不是使用primary key 或者 unique index，而是使用普通的索引）
  - ref_or_null（与 ref 类似，区别在于条件中包含对 NULL 的查询）
  - index_merge(索引合并优化)
  - unique_subquery（in的后面是一个查询主键字段的子查询）
  - index_subquery（与 unique_subquery 类似，区别在于 in 的后面是查询非唯一索引字段的子查询）
  - range（单表中的范围查询）
  - index（对于前面的每一行，都通过查询索引来得到数据）
  - all（对于前面的每一行，都通过全表扫描来得到数据）
- possible_keys：表示查询时，可能使用的索引。
- key：表示实际使用的索引。
- key_len：索引字段的长度。
- rows：扫描行的数量。
- Extra：执行情况的说明和描述。

