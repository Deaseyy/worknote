[TOC]

# 一，Python

## 1，python基础

### 基础

#### 1.1，new 和 init 的区别

#### 1.2，生成器和迭代器

#### 1.3，闭包的概念，装饰器

#### 1.4，range 和 xrange的区别

### 重点

#### 1.1，GIL锁

#### 1.2，python垃圾回收机制

#### 1.3，进程，线程，协程；多线程与多进程区别。

```
进程：
1、操作系统进行资源分配和调度的基本单位，多个进程之间相互独立
2、稳定性好，如果一个进程崩溃，不影响其他进程，但是进程消耗资源大，开启的进程数量有限制

线程：
1、CPU进行资源分配和调度的基本单位，线程是进程的一部分，是比进程更小的能独立运行的基本单位，一个进程下的多个线程可以共享该进程的所有资源
2、如果IO操作密集，则可以多线程运行效率高，缺点是如果一个线程崩溃，都会造成进程的崩溃
```

### 其他

#### 1.1，什么是猴子补丁

在程序运行期间动态的修改一个函数，类或模块。

#### 1.2，python自省

运行时能够获得对象的类型。比如：**type()、dir()、getattr()、hasattr()、isinstance()**

#### 1.3，提高python运行效率的方法

```
1、使用生成器，因为可以节约大量内存
2、循环代码优化，避免过多重复代码的执行
3、核心模块用Cython  PyPy等，提高效率
4、多进程、多线程、协程
5、多个if elif条件判断，把最可能先发生的条件放到前面写，可以减少程序判断次数，提高效率
```

## 2.web知识

#### 2.1，Web应用的本质

```
1. 浏览器发送一个HTTP请求；
2. 服务器收到请求，生成一个HTML文档；
3. 服务器把HTML文档作为HTTP响应的Body发送给浏览器；
4. 浏览器收到HTTP响应，从HTTP Body取出HTML文档并显示。
```

#### 2.2，cookie和session的区别

```
1，session 保存在服务器端，cookie 保存在客户端（浏览器），所以cookie安全性比session差，但性能更高。

2、session 的运行依赖 session id，而 session id 是存在 cookie 中的，也就是说，如果浏览器禁用了 cookie ，同时 session 也会失效。服务端存储Session时，键与Cookie中的sessionid相同，值是开发人员设置的键值对信息，进行了base64编码，过期时间由开发人员设置。
```

#### 2.3，同源策略

需要同时满足以下三点要求： 1.协议相同  2.域名相同  3.端口相同 。

#### 2.4，Nginx是什么及作用？

1、静态托管文件
首先，Nginx是一个HTTP服务器，可以将服务器上的静态文件（如HTML、图片）通过HTTP协议展现给客户端。

2、反向代理
客户端本来可以直接通过HTTP协议访问某网站应用服务器，网站管理员可以在中间加上一个Nginx，客户端请求Nginx，Nginx请求应用服务器，然后将结果返回给客户端，此时Nginx就是反向代理服务器。

3、负载均衡
当网站访问量非常大，网站越来越慢，一台服务器已经不够用了。于是将同一个应用部署在多台服务器上，将大量用户的请求分配给多台机器处理。同时带来的好处是，其中一台服务器万一挂了，只要还有其他服务器正常运行，就不会影响用户使用。



## 3.Django

#### 3.1，ORM

ORM，全拼Object-Relation Mapping，意为对象-关系映射。实现了数据模型与数据库的解耦，通过简单的配置就可以轻松更换数据库，而不需要修改代码只需要面向对象编程,orm操作本质上会根据对接的数据库引擎，翻译成对应的sql语句,所有使用Django开发的项目无需关心程序底层使用的是MySQL、Oracle、sqlite....，如果数据库迁移，只需要更换Django的数据库引擎即可。

## 4.Flask

## 5, 其他

#### 





# 二，MySQL

## 基础

#### 1，CHAR 和 VARCHAR 的区别？

```
> char的特点
1.定长字符串；插入数据小于char的固定长度时，则用空格填充；
2.因长度固定，所以存取速度要比varchar快很多，但正因为其长度固定，所以会占据多余的空间，是空间换时间的做法；
3.适用情况：
- 很短的字符串，存储效率更高。
- 定长值，或所有值都接近同一个长度。如：用char(1)存储Y/N值。再比如身份证，密码散   列等
- 经常变更的列，因为定长类型不易产生碎片。
  
> varchar的特点
1.可变长字符串；比char定长类型更节省空间，因为仅使用必要的空间。
2.存储时会多用一到两个字节来记录可变列的数据长度。当列的最大长度大于255字节，则使用两个字节表示，否则使用一个字节。
3.适用情况：
- 字符串列的最大长度比平均长度大很多。
- 列的更新很少，所以碎片不是问题。
4.varchar(5) 和 varchar(200) 存储‘hello’所占空间一样。但更长的列在使用内存临时表做排序等操作时会消耗更多的内存。

总结：最好的策略是只分配真正需要的空间； 结合性能角度（char更快）和节省磁盘空间角度（varchar更小）。
```

#### 2.若一张表中只有一个字段 VARCHAR(N)类型，utf8 编码，则 N 最大值为多少(精确到数量级即可)

由于 utf8 的每个字符最多占用 3 个字节。而 MySQL 定义行的长度不能超过65535，因此 N 的最大值计算方法为：(65535-1-2)/3。

```
减去 1 的原因是实际存储从第二个字节开始，需要一个字节存储空标志；
减去 2 的原因是因为要在列表长度存储实际的字符长度；
除以 3 是因为 utf8 每个字符最多占用 3 个字节。
```

#### 3，数据库的三范式

```
第一范式（1NF）
字段具有原子性,不可再分。(所有关系型数据库系统都满足第一范式数据库表中的字段都是单一属性的，不可再分)

第二范式（2NF）
要满足第二范式（2NF）必须先满足第一范式（1NF）。要求数据库表中的每个实例或行必须可以被唯一地区分。通常需要为表加上一个列，以存储各个实例的唯一标识。这个唯一属性列被称为主关键字或主键。

第三范式（3NF）
满足第三范式（3NF）必须先满足第二范式（2NF）。第三范式（3NF）要求一个数据库表中不包含已在其它表中已包含的非主关键字信息。
第三范式具有如下特征：
  - 每一列只有一个值 ；
  - 每一行都能区分；
  - 每一个表都不包含其他表已经包含的非主关键字信息。
```

#### 4.MqSQL中 in 和 exists 区别

in 语句是把外表和内表作hash 连接；（MySQL5.5 会改写为exists语句，MySQL5.6中，会将独立in子查询改写为join查询）

exists 语句是对外表作loop循环，每次loop循环再对内表进行查询；所以外表需全表扫描，当外表非常大时，性能会很差。

```
1.如果查询的两个表大小相当，那么用in和exists差别不大。
2.如果两个表中一个较小，一个是大表，则子查询表大的用exists，子查询表小的用in。
```



## 重点

#### 1，锁的划分

```
锁的粒度取决于具体的存储引擎，InnoDB实现了行级锁，页级锁，表级锁。加锁开销从大到小，并发能力也是从大到小
InnoDB锁类型：读锁，写锁、意向锁、MDL锁。
InnoDB行锁种类：
  - 单个行记录上的锁（record lock）
  - 间隙锁（GAP lock）；
  - 记录锁和间隙锁的组合（next-key-lock），锁定一个范围，包含记录本身。
  注：普通索引默认的就是next-key lock模式。
```

**相关知识点：**

```
1.innodb对于行的查询使用next-key lock
2.当查询的索引含有唯一属性时，将next-key-lock降级为record key
3.RR隔离级别下，为了避免幻读现象，引入GAP lock。
4.Gap锁设计的目的是为了阻止多个事务将记录插入到同一范围内，导致幻读问题的产生。
  - 有两种方式显式关闭gap锁：（除了外键约束和唯一性检查外，其余情况仅使用record lock） 
	A. 将事务隔离级别设置为RC 
	B. 将参数innodb_locks_unsafe_for_binlog设置为1
```

#### 2，InnoDB引擎的行锁是怎么实现的？

InnoDB 行锁是**通过给索引上的索引项加锁**来实现的，这一点 MySQL 与 Oracle 不同，Oracle是通过在数据块中对相应数据行加锁来实现的。

InnoDB 这种行锁实现特点意味着：只有**通过索引条件检索数据，InnoDB 才使用行级锁，否则将使用表锁！**

#### 3，数据库什么情况下会产生死锁？怎么解决？

死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方的资源，从而导致恶性循环的现象

**常见的死锁解决办法：**

```
1、如果不同程序会并发存取多个表，尽量约定以相同的顺序访问表，可以大大降低死锁机会。
2、对于并发修改同一个表的多条记录，也需要以相同顺序访问数据行；（因为InnoDB每访问一行，就会    将其锁住，直到修改完所有行）
3、在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率；
4、对于非常容易产生死锁的业务部分，可尝试使用升级锁粒度，通过表级锁定来减少死锁产生的概率；
```

#### 4，乐观锁，悲观锁，各自使用场景

为了确保多个事务同时存取数据库中同一数据时不破坏事务的隔离性和统一性以及数据库的统一性。乐观并发控制（乐观锁）和悲观并发控制（悲观锁）是并发控制主要采用的技术手段。

```
1.悲观锁：
假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。在查询数据的时候就把事务锁起来，直到提交事务。实现方式：使用数据库中的锁机制
2.乐观锁：
假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。在修改数据的时候把事务锁起来，通过version的方式来进行锁定。实现方式：一般会使用版本号机制或CAS算法实现。

3.两种锁的使用场景
乐观锁适用于写比较少的情况下（多读场景），即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。
但如果是多写的情况，一般会经常产生冲突，这就会导致上层应用会不断的进行retry，这样反倒是降低了性能，所以一般多写的场景下用悲观锁就比较合适。
```

#### 5，事务的四大特性

```
原子性：事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么全部回滚；
一致性：执行事务前后，数据状态保持一致。
隔离性：并发访问数据库时，一个事务不被其他事务所干扰，防止交叉执行导致数据不一致；
持久性：事务提交之后，对数据的改变是永久的，即使系统故障也不会丢失。
```

#### 6，数据库的事务隔离级别

```
1.读未提交(RU)：在一个事务中，可以读取到其他事务未提交的数据变化。
2.读已提交(RC)：在一个事务中，可以读取到其他事务已经提交的数据变化。
3.可重复读(RR)：MySQL默认；在一个事务开始到结束前，多次读取同样的目标数据，不会发生变化。避免了脏读、不可重复读、幻读现象的发生。
4.串行：在每个读的数据行上加表级共享锁，每次写数据时加表级排他锁。并发能力下降，大量超时和锁竞争会发生。
```

#### 7，MySQL的复制原理以及流程

复制过程中工作的线程：主服务器有一个工作线程I/O dump thread，从服务器有两个工作线程 (I/O thread 和 SQL thread)。

**7.1 主要复制过程如下：**

```
1.主库把接收的SQL请求记录到自己的binlog文件中；
2.从库的 I/O thread 去请求主库（主库通过IO dump thread 给从库IO thread传送binlog日志），将得到的binlog日志写入自己的Relay log(中继日志)文件中；
3.在从库上重做应用中继日志中的SQL语句，应用到自己的数据库上。
```

**7.2 复制的作用：**

```
1.主数据库出现问题，可以切换到从数据库。
2.可以进行数据库层面的读写分离。
3.可以在从数据库上进行日常备份。
```

**7.3 复制解决的问题：**

```
1.数据分布：随意开始或停止复制，并在不同地理位置分布数据备份
2.负载均衡：降低单个服务器的压力
3.高可用和故障切换：帮助应用程序避免单点失败
4.升级测试：可以用更高版本的MySQL作为从库
```

#### 8，Innodb的索引实现，为什么用B+树，B树和B+树的区别，各自优势

**为什么用B+树 而不是用B树**

1. B树只适合随机检索，而B+树同时支持随机检索和顺序检索；

1. B+树空间利用率更高，可减少I/O次数，磁盘读写代价更低。一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗。B+树的内部结点并没有指向关键字具体信息的指针，只是作为索引使用，其内部结点比B树小，盘块能容纳的结点中关键字数量更多，一次性读入内存中可以查找的关键字也就越多，相对的，IO读写次数也就降低了。而IO读写次数是影响索引检索效率的最大因素；

1. B+树的查询效率更加稳定。B树搜索有可能会在非叶子结点结束，越靠近根节点的记录查找时间越短，只要找到关键字即可确定记录的存在，其性能等价于在关键字全集内做一次二分查找。而在B+树中，顺序检索比较明显，随机检索时，任何关键字的查找都必须走一条从根节点到叶节点的路，所有关键字的查找路径长度相同，导致每一个关键字的查询效率相当。

1. B-树在提高了磁盘IO性能的同时并没有解决元素遍历的效率低下的问题。B+树的叶子节点使用指针顺序连接在一起，只要遍历叶子节点就可以实现整棵树的遍历。而且在数据库中基于范围的查询是非常频繁的，而B树不支持这样的操作。

1. 增删文件（节点）时，效率更高。因为B+树的叶子节点包含所有关键字，并以有序的链表结构存储，这样可很好提高增删效率

**B树和B+树的区别：**

```
B-Tree：键和值存放在内部节点和叶子节点，叶子节点间各自独立。
B+Tree：仅叶子节点存储键值数据，非叶子节点只存储键，叶子节点间通过双向链表连接。
```

**B树和B+树各自的优势**

- B树
  B树可以在内部节点同时存储键和值，因此，把频繁访问的数据放在靠近根节点的地方将会大大提高热点数据的查询效率。这种特性使得B树在特定数据重复多次查询的场景中更加高效。
- B+树
  由于B+树的内部节点只存放键，不存放值，因此，一次读取，可以在内存页中获取更多的键，有利于更快地缩小查找范围。 B+树的叶节点由一条链相连，因此，当需要进行一次全数据遍历的时候，B+树只需要使用O(logN)时间找到最小的一个节点，然后通过链进行O(N)的顺序遍历即可。而B树则需要对树的每一层进行遍历，这会需要更多的内存置换次数，因此也就需要花费更多的时间

## 进阶

#### 1，MySQL存储引擎MyISAM与InnoDB区别

```
事务支持：InnoDB支持事务，MyISAM不支持事务
锁：InnoDB 可支持行级锁，MyISAM 只支持表机锁，并发能力不如InnoDB
MVVC：InnoDB 支持 MVVC，而 MyISAM 不支持
外键：InnoDB 支持外键，而MyISAM 不支持
全文索引：InnoDB 不支持全文索引，而 MyISAM 支持
表主键：MyISAM允许没有任何索引和主键的表存在，索引都是保存行的地址；如果没有设定主键或者非空唯一索引，就会自动生成一个 6 字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。

索引：
索引存储方式不同：MyISAM使用前缀压缩技术是索引更小；InnoDB 则按照原数据格式进行存储。
索引方式不同：MyISAM索引通过数据的物理地址引用被索引的行；InnoDB根据主键引用被索引的行。
  - MyISAM：索引保存的是数据的物理地址；
  - InnoDB：索引保存真实的数据，（主索引保存完整行，普通索引保存索引列和主键值）
```

#### 2，InnoDB引擎的4大特性

```
1.插入缓冲（insert buffer)
2.二次写(double write)
3.自适应哈希索引(ahi)
4.预读(read ahead)
```

#### 3，索引的优缺点

```
- 优点：
1.加快检索速度，减少了服务器需要扫描的数据量。
2.帮助服务器避免排序和临时表。
3.将随机I/O变为顺序I/O

- 缺点：
1.创建索引和维护索引要耗费时间，增、删、改时，索引也要动态更新，会降低增/改/删的执行效率；
2.索引需要占物理空间
```

#### 4，索引设计的原则，注意事项。

	不要过度索引。索引需要额外的磁盘空间，并降低写操作的性能。在修改表内容的时候，索引会进行更新甚至重构，索引列越多，这个时间就会越长。所以只保持需要的索引有利于查询即可。

```
1.适合索引的列：where子句中的列，连接子句on指定的列，或排序分组列。
2.使用选择性高(基数大）的列创建索引；
3.为经常查询的列创建索引；
4.使用短索引，如果对长字符串列进行索引，应该指定一个前缀长度，这样能够节省大量索引空间
5.更新频繁字段不适合创建索引；
6.尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可；
7.创建复合索引需遵循最左前缀原则。
8.定义有外键的数据列一定要建立索引；
9.对于定义为text、image和bit的数据类型的列尽量不要建立索引。
```

**创建索引时注意事项：**

```
1.非空字段：
应该指定列为NOT NULL，除非你想存储NULL。在mysql中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。应该用0、一个特殊的值或者一个空串代替空值。
2.取值离散大的字段：
(列值基数大)的列放到联合索引的前面，可用count()查看字段差异值。
3.索引字段越小越好：
数据库的数据存储以页为单位一页存储的数据越多一次IO操作获取的数据越大效率越高。
```



#### 5，什么是聚簇索引

聚簇索引：将数据存储与索引放到了一块，找到索引也就找到了数据。

```
innodb中，只有主键索引是聚簇索引，如果没有主键，则挑选一个唯一键建立聚簇索引。如果没有唯一键，则隐式的生成一个键来建立聚簇索引。
其他索引都称为辅助索引，辅助索引访问数据总是需要二次查找，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引，辅助索引叶子节点存储的不再是行的物理位置，而是主键值
```

#### 6，binlog 和 redo log 的区别

binlog：二进制日志文件 ，redo log：重做日志文件(InnoDB)

```markdown
# 1.记录的内容不同
- binlog 是逻辑日志，记录所有数据的改变信息
- redo log 是物理日志，记录所有InnoDB表数据的变化
# 2.记录内容的时间不同
- binlog 记录commit完毕之后的 DML和DDL SQL语句
- redo log 记录事务发起，数据修改之后的值，无论事务是否提交
# 3.写入文件的方式不同
- binlog 不是循环使用，在写满或实例重启之后，会生成新的binlog文件
- redo log 顺序写，循环写，最后一个文件写满后，会重新写第一个文件。写满日志文件切换时，会	触发脏页的刷新。
# 4.作用不用
- binlog 可作为恢复数据使用，主从复制搭建。
- redo log 作为故障宕机后的数据恢复使用，保证数据完整性。
```

#### 7，主键使用自增ID还是UUID

推荐使用自增ID，不要使用UUID。

```markdown
因为在InnoDB存储引擎中，主键索引是作为聚簇索引存在的，也就是说，主键索引的B+树叶子节点上存储了主键索引以及全部的数据(按照顺序)，
> 如果主键索引是自增ID，那么只需要不断向后排列即可，如果是UUID，由于到来的ID与原来的大小不确定，会造成非常多的数据插入，数据移动，页分裂，然后导致产生很多的内存碎片，进而造成插入性能的下降。

> 且UUID占用空间很大，所以每页能存储的数据就会变少，每次IO能读取的数据变少，IO次数会更多，降低查询效率。
```

#### 8，如何对大表进行优化？

**8.1 单表不拆分下的优化**

- 限定数据查询范围： 禁止不带任何限制条件的查询语句。比如：查询历史记录时限制访问一个月。
- 读/写分离： 经典的数据库拆分方案，主库负责写，从库负责读；
- 缓存： 使用MySQL的缓存，另外对重量级、更新少的数据考虑使用应用级别的缓存

**8.2 分库分表**

- 垂直拆分：
  根据数据库里面数据表的相关性进行拆分。 例如，用户表中既有用户的登录信息又有用户的基本信息，可以将用户表拆分成两个单独的表，甚至放到单独的库做分库。

  ```markdown
  # 1.垂直拆分的优点
  - 行数据变小；
  - 减少读取的Block数；
  - 减少I/O次数；
  - 简化表的结构；
  - 易于维护。
  # 2.垂直拆分的缺点
  - 主键会出现冗余；
  - 需要管理冗余列；
  - 会引起Join操作；
  - 让事务变得更加复杂。
  ```

- 水平拆分

  保持数据表结构不变，通过某种策略存储数据分片。这样每一片数据分散到不同的表或者库中，达到了分布式的目的。 

  水品拆分可以支持非常大的数据量。但分表仅仅是解决了单一表数据过大的问题，由于表的数据还是在同一台机器上，其实对于提升MySQL并发能力没有什么意义，所以 **水平拆分最好分库** 。



#### 9，分库分表后将面临那些棘手的问题？

```markdown
# 1.事务支持
分库分表后，就成了分布式事务了。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价； 如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。

# 2.跨库join
只要是进行切分，跨节点Join的问题是不可避免的。但是良好的设计和切分却可以减少此类情况的发生。解决这一问题的普遍做法是分两次查询实现。在第一次查询的结果集中找出关联数据的id,根据这些id发起第二次请求得到关联数据。

# 3.跨节点的count,order by,group by以及聚合函数问题
这些是一类问题，因为它们都需要基于全部数据集合进行计算。多数的代理都不会自动处理合并工作。解决方案：与解决跨节点join问题的类似，分别在各个节点上得到结果后在应用程序端进行合并。和join不同的是每个结点的查询可以并行执行，因此很多时候它的速度要比单一大表快很多。但如果结果集很大，对应用程序内存的消耗是一个问题。

# 4.数据迁移，容量规划，扩容等问题
来自淘宝综合业务平台团队，它利用对2的倍数取余具有向前兼容的特性（如对4取余得1的数对2取余也是1）来分配数据，避免了行级别的数据迁移，但是依然需要进行表级别的迁移，同时对扩容规模和分表数量都有限制。总得来说，这些方案都不是十分的理想，多多少少都存在一些缺点，这也从一个侧面反映出了Sharding扩容的难度。

# 5.ID问题
一旦数据库被切分到多个物理结点上，我们将不能再依赖数据库自身的主键生成机制。一方面，某个分区数据库自生成的ID无法保证在全局上是唯一的；另一方面，应用程序在插入数据之前需要先获得ID,以便进行SQL路由。
常见的主键生成策略：
- UUID
  使用UUID作主键是最简单的方案，但是缺点也是非常明显的。由于UUID非常的长，除占用大量存储空间外，最主要的问题是在索引上，在建立索引和基于索引进行查询时都存在性能问题。
- Twitter的分布式自增ID算法Snowflake
  在分布式系统中，需要生成全局UID的场合还是比较多的，twitter的snowflake解决了这种需求，实现也还是很简单的，除去配置信息，核心代码就是毫秒级时间41位 机器ID 10位 毫秒内序列12位。

# 6.跨分片的排序分页
一般来讲，分页时需要按照指定字段进行排序。当排序字段就是分片字段的时候，我们通过分片规则可以比较容易定位到指定的分片，而当排序字段非分片字段的时候，情况就会变得比较复杂了。为了最终结果的准确性，我们需要在不同的分片节点中将数据进行排序并返回，并将不同分片返回的结果集进行汇总和再次排序，最后再返回给用户。
```



#### 10，数据库结构优化可以从那些方面入手？

一个好的数据库设计方案，需要考虑数据冗余、查询和更新的速度、字段的数据类型是否合理等多方面的内容。

```markdown
# 1.将字段很多的表分解成多个表
对于字段较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。
因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。

# 2.增加中间表
对于需要经常联合查询的表，可以建立中间表以提高查询效率。
通过建立中间表，将需要通过联合查询的数据插入到中间表中，然后将原来的联合查询改为对中间表的查询。

# 3.增加冗余字段
设计数据表时应尽量遵循范式理论的规约，尽可能的减少冗余字段，让数据库设计看起来精致、优雅。但是，合理的加入冗余字段可以提高查询速度。
表的规范化程度越高，表和表之间的关系越多，需要连接查询的情况也就越多，性能也就越差。
```



# 三，Redis

## 基础

#### 1，为什么说redis是单线程模型？单线程有什么好处？

单线程模型指的是：一个线程处理所有网络请求，即执行redis命令的模块是单线程的；其他模块仍用了多个线程。

好处：1.不需要考虑并发安全性； 2.省去了很多上下文切换线程的时间

#### 2，redis分布式锁的实现
##### 2.1 简单分布式锁-同步锁

1.使用setnx设置一个公共锁

   锁名key可以为：lock:product_id

```
# 基于string类型setnx的特性：仅当key不存在时设置
setnx lock-key value
```

利用setnx命令的返回特征，有值则返回设置失败0，无值则返回设置成功1

- 对于返回设置成功的，拥有控制权，进行下一步的具体业务操作
- 对于返回设置失败的，不具有控制权，排队或等待

2.操作完毕通过del操作释放锁

```
del lock-key
```

注意：这是一种设计概念，依赖规范保障（必须锁同一把锁），高并发下还会有很多各种问题出现，下面分析。



##### 2.2. 分布式锁改良--死锁

死锁的各种出现场景及解决方案，需要一步一步优化，考虑全面，如下：

**1.业务程序中途抛错，导致锁永不释放**

解决方案：业务逻辑应当放入try代码块，使用finally无论如何都会及时释放锁

**2.程序中途宕机，来不及释放锁，导致死锁**

解决方案：使用expire为锁key添加时间限定, 到限定时间不释放，放弃锁

```
expire lock-key 秒
pexpire lock-key 毫秒
// 或设置锁key时直接设置过期时间：
set lock-key true ex 10 nx  //更好，防止设置锁后来不及设置过期时间就宕机
```

由于操作通常都是微妙或毫秒级，因此过期时间不宜设置过大，具体需测试后确认：

- 例如：持有锁的操作最长执行时间127ms，最短7ms
- 测试百万次最长执行时间对应命令的最大耗时，测试百万次网络延迟平均耗时

**3.场景2中，如果使用先设置锁，再设置过期时间，则有可能导致没来得及设置过期时间程序就已宕机**

解决方案：设置锁key时直接设置过期时间

```
set lock-key true ex 10 nx  // 底层分为设置锁和过期时间两部，但做了原子操作
```



##### 2.3. 分布式锁改良--锁失效

业务逻辑可能会执行超时，高并发下会出现各种锁问题

**4.在上面的场景下，如果执行超时后锁释放，将会导致多个线程同时在执行，还会出现一个线程删除另一个线程设置的锁的情况，可能导致锁永久失效**

```
场景分析：
有三个线程：A 执行需要15s；B 执行需要8s；C 执行需要8s；锁过期时间设为10s.
1.线程A先执行，当在10s时，A设的锁被释放，此时线程A还在执行，同时线程B拿到锁也开始执行；
2.当B执行到5s时，线程A执行完将会释放锁，但此时删除的是B线程设置的锁；
3.接着线程B还在执行，线程C就已经拿到锁开始执行；如此锁将永久失效。
```

**4.1，一个线程删除另一个线程设置的锁，导致锁失效**

解决方案：

- 思路：每个线程只能删除自己设置的锁

- 措施：锁的value值设置一个随机uuid，该线程在删除锁前，判断该锁的value值是否等于该uuid

  ```
  clientId = 设一个随机UUid
  try:
  	......
  finally:
  	if clientId == 锁lockkey的value值
  		del lockkey 删除锁
  ```

**4.2，线程执行超时，锁已释放，导致多个线程同时在执行** （又会同时执行一个原子代码块）

解决方案：在线程开始时，开启一个分线程，在分线程内部，启动一个定时器，每隔一段时间（如锁过期时间的三分之一）给锁续时，直到主线程执行完删除锁，分线程也随之结束

​		一般自己去写这种代码逻辑，会出很多难以预料的bug，都会采用已有成熟的框架方案去实行，例如java中的redisson框架。

##### 2.4 主从架构中的问题

​		如果是redis单机架构，以上方案的解决已经相当完善，出现bug几率微乎其微；但如果是主从架构，还会出现一些其他问题

##### 执行框架图

![image-20201024120518199](C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20201024120518199.png)

**5.线程1设一把锁到主节点master，主节点还没来得及将锁同步至从节点，就已经宕机；此时线程2将访问新的主节点（slave结点），发现没有锁，将设置一把新锁，此时，又出现了两个线程同时执行一个超卖原子代码块的问题。**

方案参考：可以使用zookeeper解决

**6.由于redis是单线程的，所以内部都是串行处理；但redis的QPS很快，对于一般中小企业业务已经够用**

需要高并发处理请求，方案参考：将库存总数分片存入redis多个节点，分开线程扣减不同的库存，如：

100个库存，分为key1 = 30， key2=30， key3=40


#### 3，数据删除策略

**3.1 删除策略**

```markdown
主要针对具有时效性的key的删除管理
# 1.定时删除
创建一个定时器，当key设置有过期时间，且过期时间到达时，由定时器任务立即执行对key的删除操作。
优缺点：节约内存，但cpu压力大——以时间换空间
# 2.惰性删除
数据到达过期时间，不做处理，等到下次访问该数据时：
    - 如果未过期，返回该数据；
    - 发现已过期，删除，返回不存在；
具体实现：每次获取数据时都会调用 expireIfNeeded() 函数，检查key是否过期。
优缺点：节约cpu性能，发现必须删除时才删除，但内存压力大，出现长期占用内存情况——以空间换时间
# 3.定期删除
- 内存定期随机清理
- 每秒花费固定的cpu资源维护内存
- 随机抽查，重点抽查
```

**3.2 逐出算法（淘汰策略）**

```markdown
> 检测有过期时间的数据：
1.lru 挑选最近最少使用的数据淘汰
2.ttl 挑选将要过期的数据淘汰
3.random 任意选择数据淘汰
> 检测全库数据：
4.lru 挑选最近最少使用的数据淘汰
5.random 任意选择数据淘汰
> 放弃数据驱逐：
6.no-enviction：禁止驱逐数据，会引发错误OOM(Out Of Memory)
```

#### 4，数据持久化

**RDB**

使用bgsave指令，它不会加入到任务执行序列中，而是创建子进程去执行；

```markdown
# 优点
- RDB是一个紧凑压缩的二进制文件，存储效率较高
- RDB内部存储的是某个时间点的数据快照，非常适用于数据备份，全量复制等场景
- RDB恢复数据的速度要比AOF快很多
应用：服务器中每X小时执行bgsave备份，并将RDB文件拷贝到远程机器中，用于灾难恢复

# RDB缺点
- 无法做到实时持久化，具有较大的可能性丢失数据
- bgsave指令每次运行要fork操作创建子进程，要牺牲掉一些性能
- redis的众多版本中未进行RDB文件格式版本统一，有可能出现各版本服务器之间数据无法兼容现象
```

**AOF**

```markdown
# AOF写数据过程 
1.先将执行过的命令写入缓存区。
2.将命令写入AOF文件，AOF写命令会刷新缓存区.

# AOF写数据三种策略
- always(每次)：每次操作均同步到AOF文件中，零误差，性能低，不建议使用
- everysec(每秒)：每秒将缓冲区的指令同步到AOF文件中，最多丢失1秒内的数据，默认配置。
- no(系统控制)：由操作系统控制每次同步到AOF文件的周期，整体过程不可控
```

**总结：**

RDB文件占用存储空间小，存储慢，恢复快，阶段性丢失数据；AOF文件占用空间大，存储块，恢复慢，数据安全依赖策略决定。

若配置了两种，数据恢复会优先加载AOF。

#### 5，Redis 的同步机制了解么？

	Redis 可以使用主从同步，从从同步。第一次同步时，主节点做一次 bgsave， 并同时将后续修改操作记录到内存 buffffer， 待完成后将 rdb 文件全量同步到复制节点，复制节点接受完成后将 rdb 镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。

#### 6，Redis 常见性能问题和解决方案？

```
(1) Master 最好不要做任何持久化工作，如 RDB 内存快照和 AOF 日志文件

(2) 如果数据比较重要，某个 Slave 开启 AOF 备份数据，策略设置为每秒同步一次

(3) 为了主从复制的速度和连接的稳定性，Master 和 Slave 最好在同一个局域网内

(4) 尽量避免在压力很大的主库上增加从库

(5) 主从复制不要用图状结构，用单向链表结构更为稳定，即：Master <- Slave1 <- Slave2 <- Slave3...

这样的结构方便解决单点故障问题，实现 Slave 对 Master 的替换。如果 Master 挂了，可以立刻启用 Slave1 做 Master，其他不变。
```

#### 7.缓存雪崩，缓存击穿，缓存穿透，以及它们的解决方案

**7.1 缓存雪崩**

```markdown
# 概念
设置缓存时采用了相同的过期时间，导致大量缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩。

# 解决方案
给缓存设置随机过期时间，我们可以在原有的失效时间基础上增加一个随机值，比如1-5分钟随机，这样每个缓存的过期时间的重复率就会降低，很难引发集体失效事件。
```

**7.2 缓存击穿**

```markdown
# 概念
单个key高热数据过期的瞬间，恰好数据访问量较大，未命中redis，发起大量对同一数据的请求到数据库。

# 解决方案：
可以使用锁来进行并发控制，如redis的setnx命令实现分布式锁。当一个请求检测到缓存失效时，先使用setnx获取锁，如果能成功获取(返回值为1)，则请求数据库，然后写入缓存，否则重新进行整个请求读取缓存的过程。

```

**7.3 缓存穿透**

```markdown
# 概念：
查询了一定不存在的数据，跳过了合法数据的redis数据缓存阶段，导致每次请求这个不存在的数据都要访问数据库。

# 解决方案：
最常见的则是采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被 这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。

另外也有一个更为简单粗暴的方法，如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。
```



#### 8.如何保证缓存与数据库的双写一致性

**先更新数据库再删除缓存：最经典的缓存+数据库读写的模式，就是 Cache Aside Pattern。**

- 读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。
- 更新的时候，**先更新数据库，然后再删除缓存**。

**为什么是删除缓存，而不是更新缓存？**

```
某些缓存是经过大量计算后的结果，如果每次修改数据后都去更新缓存，那么当写操作很多，但却很少读取时，就会出

现大量冷数据。比如每分钟更新数据库100次，则需要更新100次缓存，但实际只读取了一次或很少次，那么就有很多

次的缓存更新都是没有使用到的，浪费大量计算资源。

```

所以很少采用更新缓存的做法，一般都是先删除缓存，用到时再去数据库取来计算再写入缓存。

**a.简单的缓存不一致问题及解决方案**

```markdown
# 问题分析：
先修改数据库，再删除缓存。若删除缓存失败了，那么会导致数据库中是新数据，缓存中是旧数据，数据就出现了不一致。

# 解决思路：
解决思路：**先删除缓存，再修改数据库**。若数据库修改失败，那么数据库中是旧数据，缓存中是空的，那么不会出现数据不一致问题。因为再读的时候没有缓存，则读数据库中旧数据，然后更新到缓存中。
```

**b.高并发下的缓存不一致问题分析**

```markdown
# 问题分析：
数据发生了变更，先删除了缓存，然后要去修改数据库。此时还没来得及修改，另一个请求过来，去读缓存，发现缓存

空了，去查询数据库，此时查到的是即将被修改的旧数据，放到了缓存中。随后上一个请求完成了数据库的修改。这

时，数据库和缓存中的数据不一样了。

# 解决思路：
方案1: 利用队列将读和写请求中间的部分操作异步串行化执行
- 写操作：删除缓存 --> 修改数据库
- 读操作：读缓存(若为空) --> 读数据库 --> 更新缓存
需要串行执行： 修改数据库--> 读数据库 --> 更新缓存

方案2：双删加超时，在写库前后都进行删缓存redis.del(key)操作，并且设定合理的超时时间。这样最差的情况是在超时时间内存在不一致，当然这种情况极其少见(*即同时出现并发问题和写库后删缓存失败的情况*)，可能的原因就是服务宕机。此种情况可以满足绝大多数需求。
- a、先淘汰缓存，更新数据库
- b、延时200ms
- c、再次淘汰缓存，从数据库更新最新的数据缓存
这个200ms具体该休眠多久根据自己的项目的读写数据业务逻辑的耗时而定。

方案3：使用canal解析binlog Mysql通过binlog同步redis。

```


## 进阶

#### 1，redis扩容机制（渐进式单线程扩容）

#### 2，Pipeline 有什么好处，为什么要用 pipeline？

	可以将多次 IO 往返的时间缩减为一次，前提是 pipeline 执行的指令之间没有因果相关性。使用

redis-benchmark 进行压测的时候可以发现影响 redis 的 QPS 峰值的一个重要因素是 pipeline 批次指

令的数目。

#### 3，Redis 哈希槽的概念？

	Redis 集群没有使用一致性 hash,而是引入了哈希槽的概念， Redis 集群有16384 个哈希槽，每个key 通过 CRC16 校验后对 16384 取模来决定放置哪个槽， 集群的每个节点负责一部分 hash 槽。

#### 4，Redis 集群的主从复制模型是怎样的？

	为了使在部分节点失败或者大部分节点无法通信的情况下集群仍然可用， 所以集群使用了主从复制模型,每个节点都会有 N-1 个复制品.

#### 5，Redis 集群会有写操作丢失吗？

Redis 并不能保证数据的强一致性，这意味这在实际中集群在特定的条件下可能会丢失写操作。



# 四，网络

#### 1, 各种网络协议

- HTTP 超文本传输协议 / HTTPS 超文本传输安全协议  (应用层)
- FTP 文件传输协议  (应用层)
- SMTP 简单邮件传输协议  (应用层)
- DNS  (应用层)
- TCP 传输控制协议 / UDP 用户数据报协议  (传输层)
- IP  (网络层)

#### 2, TCP三次握手，四次挥手

**三次握手：**

![img](https://p-blog.csdn.net/images/p_blog_csdn_net/phunxm/EntryImages/20091227/TCPConnect.JPG)

```
第一次：客户端要和服务端进行通信，首先请求服务端想要建立连接，此时发出一个（SYN=1，seq=200）的请求信号。

第二次：当服务端收到客户端的请求时，要回给客户端一个确认响应（ACK=1，seq=200+1），表示接收到请求，然后在给客户端一个请求，询问现在是否能建立连接（SYN=1，seq=500）。

第三次：当客户端收到确认信息后，回一个响应（ACK=1，seq=500+1）,完成三次握手开始建立连接。
```

**四次挥手：**

```
第一次：客户端请求断开通信连接，向服务端发送（FIN = 1，seq=u）。

第二次：服务端收到请求后，回复（ACK=1， seq=u+1），表示接收到请求

第三次：服务端再次发送（FIN=1，seq=w），表示中断请求，同意断开连接

第四次：客户端收到中断请求，发送回复（ACK=1，seq=w+1），断开与服务器的连接。
```

#### 3，TCP和UDP的区别

| tcp -- 传输控制协议        | udp -- 用户数据报协议    |
| -------------------------- | ------------------------ |
| 面向连接                   | 面向无连接               |
| 首部20字节                 | 首部8字节                |
| 只能点对点通信             | 一对一，一对多，多对一   |
| 字节流传输                 | 报文传输                 |
| 可靠稳定的传输（三次握手） | 不可靠不稳定，易丢失数据 |
| 传输效率低，速度慢         | 传输效率高，速度快       |

#### 4，**为何基于tcp协议的通信比基于udp协议的通信更可靠？** 

TCP的可靠保证，是它的三次握手双向机制，这一机制保证校验了数据，保证了他的可靠性。 UDP就没有了，udp信息发出后，不验证是否到达对方，所以不可靠。

#### 5，什么是socket？简述基于tcp协议的套接字通信流程。

Socket 又称”套接字”,是系统提供的用于网络通信的方法.

TCP编程的客户端一般步骤是：

```
1、创建一个socket，用函数socket()；
2、设置socket属性，用函数setsockopt();* 可选
3、绑定IP地址、端口等信息到socket上，用函数bind();* 可选
4、设置要连接的对方的IP地址和端口等属性；
5、连接服务器，用函数connect()；
6、收发数据，用函数send()和recv()，或者read()和write();
7、关闭网络连接；
```

#### 6，简述 进程、线程、协程的区别 以及应用场景？

| 区别   | 进程                                 | 线程                               |
| ------ | ------------------------------------ | ---------------------------------- |
| 定义   | 进程是资源分配的最小单位             | 线程是系统调度的最小单位           |
| 功能   | 如一台电脑上同时运行多个QQ           | 如一个QQ开多个聊天窗口             |
| 优缺点 | 进程执行开销大，但利于资源的管理维护 | 线程执行开销小，不利于资源管理维护 |
| 共享   | 进程间数据不共享                     | 线程间共享进程数据                 |

**协程的适用场景**： 当程序中存在大量IO操作时，适用于协程。

**python中使用线程池和进程池**：Threadpool 和ProcessPool模块。

#### 7，进程之间如何进行通信

进程间通信（IPC）方法包括：管道（PIPE）、消息排队、共用内存以及套接字（Socket）。

#### 8，并发和并行的区别

```
并发是指一个处理器同时处理多个任务。
并行是指多个处理器或者是多核的处理器同时处理多个不同的任务。
并发是逻辑上的同时发生，而并行是物理上的同时发生。
```

#### 9，Http与Https的区别

```markdown
- HTTP 的URL 以http:// 开头，而HTTPS 的URL 以https:// 开头
- HTTP 是不安全的，而 HTTPS 是安全的
- HTTP 标准端口是80 ，而 HTTPS 的标准端口是443
- 在OSI 网络模型中，HTTP工作于应用层，而HTTPS 的安全传输机制工作在传输层
- HTTP 无法加密，而HTTPS 对传输的数据进行加密
- HTTP无需证书，而HTTPS 需要CA机构颁发的SSL证书
```

#### 10, 介绍HTTPS协议，SSL协议，SSL建立连接的过程。  

**HTTPS：**

```
HTTP + 加密 + 认证 + 完整性保护 = HTTPS
HTTPS 就是身披SSL协议外壳的HTTP，只是 HTTP 通信接口部分用SSL 和 TLS 协议替代。
通常，HTTP直接和TCP进行通信。当使用SSL时，则演变成先和SSL通信，再由SSL和TCP通信。
HTTPS 采用混合加密机制：
- 使用公开密钥加密方式交换共享密钥加密的密钥
- 使用共享密钥加密方式进行之后的通信建立，交换报文
```

**HTTP的缺点：**

```
1.通信使用明文，内容可能会被窃听。       解决方案：SSL加密
2.不验证通信方的身份，有可能遭遇伪装。    解决方案：SSL证书
3.无法证明报文的完整性，有可能已遭篡改    解决方案：SSL完整性保护
（ps：其他未加密的协议中也会存在这类问题）
```

**SSL（安全套接层）：**

```markdown
1.SSL提供：加密，认证，完整性保护
- 将HTTP通信线路进行加密。
- 提供证书认证手段，用于确定通信方。
- 提供报文完整性保护。

2.SSL采用 公开密钥加密（公私钥）的加密处理方式
```

**加密算法：**

```
- 对称加密（共享密钥加密）算法：指加密和解密使用相同的密钥，速度高，可加密内容较大；典型的有：DES、AES、RC5、IDEA（分组加密）RC4。

- 非对称加密（公开密钥加密）算法：是指加密和解密使用不同的密钥，加密速度较慢，但能提供更好的身份认证技术，（公钥用于加密，私钥用于解密）；典型的算法RSA、DSA、DH。
```

#### 11，HTTP优化方案

1，TCP复用：TCP连接复用是将多个客户端的HTTP请求复用到一个服务器端TCP连接上，而HTTP复用则是一个客户端的多个HTTP请求通过一个TCP连接进行处理。前者是负载均衡设备的独特功能；而后者是HTTP 1.1协议所支持的新功能。

2，内容缓存：将经常用到的内容进行缓存起来，那么客户端就可以直接在内存中获取相应的数据了。

3，压缩：将文本数据进行压缩，减少带宽。

4，SSL加速（SSL Acceleration）：使用SSL协议对HTTP协议进行加密，在通道内加密并加速。

5，TCP缓冲：通过采用TCP缓冲技术，可以提高服务器端响应时间和处理效率，减少由于通信链路问题给服务器造成的连接负担。

# 五，Linux

#### 1.linux常用命令

```
ls  pwd  cd  vi  rm  mkdir  tree  cp  mv  cat  more  grep  shutdown 

重定向命令:
>  : 表示输出，会覆盖文件原有的内容
>> : 表示追加，会将内容追加到已有文件的末尾
```



# 六，设计模式

。。。。。。
