# 常用操作

## 常用sql

### 1.根据某字段删除所有重复行，只保留一行（取id最大那行）

```mysql
delete from fatie where id not in(select t.id1 from ( (select max(a.id) id1 from fatie a group by a.name)as t))
```

### 2.一行查询为多行

```mysql
select SUBSTRING_INDEX(SUBSTRING_INDEX(lower_Veneer,'/',help_topic_id+1),'/',-1) AS lower_Veneer
from t_cboardproductinfo a 
join mysql.help_topic  b ON b.help_topic_id < ( length( a.lower_Veneer ) - length( REPLACE ( a.lower_Veneer, '/', '') ) + 1 ) 
where lower_Veneer is not null and lower_Veneer!=''
```

### 3.插入数据时避开重复数据

三种方法如下：

使用以下方法的前提是表中有一个 PRIMARY KEY 或 UNIQUE 约束/索引，否则，使用以上三个语句没有特殊意义，与使用单纯的 INSERT INTO 效果相同。

```mysql
A、insert ignore into(若没有则插入，若存在则忽略)
作用：
insert ignore 会根据主键或者唯一键判断，忽略数据库中已经存在的数据
若数据库没有该条数据，就插入为新的数据，跟普通的 insert into 一样
若数据库有该条数据，就忽略这条插入语句，不执行插入操作。

B、insert into ... on duplicate key update(若没有则正常插入，若存在则更新)
作用：
在 insert into 语句末尾指定 on duplicate key update，会根据主键或者唯一键判断：
若数据库有该条数据，则直接更新原数据，相当于 update
若数据库没有该条数据，则插入为新的数据，跟普通的 insert into 一样。

C、replace into(若没有则正常插入，若存在则先删除后插入)
作用：
replace into 会根据主键或者唯一键判断：
若表中已存在该数据，则先删除此行数据，然后插入新的数据，相当于 delete + insert
可能会丢失数据、主从服务器的 AUTO_INCREMENT 不一致。
若表中不存在该数据，则直接插入新数据，跟普通的 insert into 一样。
```

### 4.MySQL 将查询出来的一列数据拼装成一个字符串

```mysql
select GROUP_CONCAT(字段a) from tb   // 默认逗号分割
select GROUP_CONCAT(字段a separator ';') from tb
```

### 5.in 子查询优化 为 join连接查询

```mysql
select * from table1 where id in ( select uid from table2 );
优化为：
select table1.* from table1 inner join table2 on table1.id=table2.uid;
```

### 6. in 子查询 和 exists 子查询

```python
# in
select * from t_key_personnel where job_num in (select key_person from t_platform_acquaintances)
# exists
select * from t_key_personnel t1 where exists (select id from t_platform_acquaintances t2 where t1.job_num=t2.key_person)
```

**exists 执行顺序如下：**

```
1.首先执行一次外部查询
2.对于外部查询中的每一行分别执行一次子查询，而且每次执行子查询时都会引用外部查询中当前行的值。
3.使用子查询的结果来确定外部查询的结果集。
如果外部查询返回100行，SQL   就将执行101次查询，一次执行外部查询，然后为外部查询返回的每一行执行一次子查询。
从执行情况来看：外表的大小控制着整条sql的效率，外表越小，执行次数越少，最终查询结果就越快
```

**EXISTS与IN的使用效率：**

**in适用外表大而内表小的情况；EXISTS适用外表小而内表大的情况**

### 7.利用GROUP BY 的WITH ROLLUP 子句做统计

使用GROUP BY的WITH ROLLUP字句可以检索出更多的分组聚合信息，它不仅仅能像一般的GROUP BY语句那样检索出各组的聚合信息，在根据多个字段分组时，还能检索出每个字段分组的整体聚合信息。

```mysql
select year, country, product, sum(profit) from sales group by year, country, product with rollup;
```

```
|2004 | china | tnt1 | 2001 |
| 2004 | china | tnt2 | 2002 |
| 2004 | china | tnt3 | 2003 |  # 2004 china tnt3的总量
| 2004 | china |      | 6006 |  # 2004 china总量
| 2004 |       |      | 6006 |  # 2004 总量
| 2005 | china | tnt1 | 4011 |
| 2005 | china | tnt2 | 4013 |
| 2005 | china | tnt3 | 4015 |
| 2005 | china |      | 12039 |
| 2005 |       |      | 12039 |
```

注意：

- 1、当使用 ROLLUP 时, 不能同时使用ORDER BY 子句进行结果排序。换言之， **ROLLUP和ORDER BY 是互相排斥的** 
- 2、LIMIT 用在 ROLLUP 后面。

### 8.根据已有表结构创建临时表

```sql
create TEMPORARY table `t_module_tmp` like `t_module`
```



## 常用命令

```mysql
# 查看优化器优化后的sql
explain extended sql语句，然后show warnings查看

# 查看表的索引
show index from tableA；
# 查看变量
show variables like '%character%'; # 查看字符集
# 查询SQL线程
select * from information_schema.innodb_trx;
# 杀死线程
kill trx_mysql_thread_id
KILL 938943;
# 查看数据库连接数
show status like 'Threads%';

-- 锁超时问题
SELECT * FROM information_schema.INNODB_TRX
kill 302226;

# 获取当前数据库的所有连接的相关信息
show processlist
# 查看死锁的展示信息
show engine innodb status
# 查看mysql的最大连接数
show variables like '%max_connections%'
# 查看服务器响应（实际使用）的最大连接数
show global status like 'Max_used_connections'

# 创建索引  -- 此方法只能添加普通索引和唯一索引
CREATE INDEX index_name ON table_name (column_list)
CREATE UNIQUE INDEX index_name ON table_name (column_list)
# 删除索引
drop index index_name on table_name
alter table table_name drop index index_name
# 查看索引
show index from table

```

### 1. 修改数据库最大连接数

```
修改方式一：set GLOBAL max_connections=2000;（一次生效方法）

修改方式二：vi /etc/my.cnf  修改配置文件
[mysqld]
set-variable=max_connections=250  #加入这些内容
:wq

/etc/init.d/mysqld restart
```







# 一，函数

调用语法：

select 函数名(参数) 【from table】（需要用到表中字段就加）

## 单行函数 

### 字符函数：

- length()  获取参数值的字节个数；         SQLserver 中为 datalength

- char_length() 获取参数值字符个数；     SQLserver 中为 len

- concat()  拼接字符串

- upper()，lower()

- left(str,2), right(str,3)  从左(或右)边截取指定个数字符串

- substr，substring  截取字符串 （两个意思一样）

  - sql语言中索引都是从1开始
  - select substr('一二三四再来一次'，5)  output；从第5个字符开始取到最后，输出为‘再来一次’。
  - select substr('一二三四再来一次'，5，2)  output；从第五个字符开始取，取两个字符。

- position(substr in str) 返回子串第一次出现的索引，若找不到返回0

- instr(str,substr)  返回子串第一次出现的索引，若找不到返回0

- locate(substr, str)  返回子串第一次出现的索引，若找不到返回0

  - 以上三个函数都可以用来判断字段中是否包含某字符

    selelct  * from table where locate('张', name)>0

- trim('  str  ' )  去除字符串两边空格

  - trim('a' from 'str')  去除两边的 ‘a’

- lpad('str', 10, '*')，rpad()  用指定的字符实现左(右)填充至指定长度

- replace('str', 'oldstr', 'newstr') 替换

- rand() 产生随机数，可配合order by提取随机行

  原理就是order by rand()能够把数据随机排序；

  select * from table order by rand() limit 5;  #  随机抽取了5 个样本



### 数学函数

- round(1.45)  四舍五入
  - round(1.456, 2)  保留两位小数
- ceil(1.52) ，floor() 向上(下)取整
- truncate(1.69, 1) 直接截断，保留1位小数-->1.6
- mod(10，3) 取余，被除数为正数，取余结果就是正数



### 日期函数

- now() 返回当前系统日期+时间
- curdate()  返回当前系统日期，不包含时间
  - current_date 变量 返回也一样
- curtime()  返回当前时间，不包含日期
- date(now())  获取指定时间的指定的部分(日期，时间，年，月，日，时，分, 秒)
  - time(now()),  year(now()), month()
  - monthname()  获取月份的英文名(默认为数字)  
- str_to_date 将字符串通过指定格式转换为日期
  - str_to_date('1998-3-2', '%Y-%c-%d')  -->1998-03-02
  - str_to_date('4-3 1992',  '%c-%d %Y')
- date_format(now(), '%y年%m月%d日') 将日期转化为指定格式字符串。

![1566699738933](C:\Users\12395\AppData\Local\Temp\1566699738933.png)

- 计算两个日期时间差函数

  - timestampdiff(day, 远的时间，近的时间)      精确到秒

    第一个参数可以为week，day，month，year，hour，minute，second

    结果为第二个参数（近的时间） - 第一个参数（远的时间）

  - datediff()   只能比较DAY天数  精确到天， 参数顺序和上面相反

    ```mysql
    # 区别
    ## 比较精确到每一秒，返回第一个参数指定类型的值；
    ## 相差1天
    select TIMESTAMPDIFF(DAY, '2018-03-20 23:59:00', '2018-03-22 00:00:00');  
    ## 相差2天，仅比较日期天数差（精确到天）,返回相差天数
    SELECT DATEDIFF('2018-03-20 23:59:00', '2018-03-22 00:00:00');  -- -2
    
    # 注意：sqlserver中datediff():
    ## 根据第一个参数（年月日时分秒）求差，精确到第一个参数指定的类型,返回第一个参数指定类型的值；（即指定哪个类型，精确到该类型，返回该类型的差值）
    SELECT DATEDIFF(day,'2018-03-20 23:59:00', '2018-03-22 00:00:00'); # 相差2天
    
    SQLserver：datediff（day,小日期，大日期）
    # 等同
    mysql:datediff(大日期，小日期)
    ```

- to_days()  将日期时间戳转化为为天数

- adddate(now(),5)  返回指定时间上增加多少天后的日期时间 

### 其他函数

- select version();  查看版本号
- select database();  查看当前数据库
- select user();  查看当前用户



### 流程控制函数

##### 1. if函数： if else的效果 

- select if(10>5,'大'，'小')；
- select if(字段a is null, '22', '33')

##### 2. if  then else

```mysql
if 条件 then
	语句1；
elseif 条件 then
	语句2；
else
	语句3；
end if；
```

函数中使用 if 语句：

```mysql
DELIMITER $$
DROP FUNCTION IF EXISTS GetCMESWarningCount $$
CREATE FUNCTION GetCMESWarningCount(ParentType VARCHAR(50),PartsCode VARCHAR(50),DefectSrcCode VARCHAR(50),UploadID VARCHAR(50)) RETURNS VARCHAR(50)
begin
   DECLARE count_id INT;
   if DefectSrcCode='' then
		SELECT count(ID) INTO count_id from T_CMES_PROJECT_COOPERATE_USE AS t1 WHERE t1.Parent_Type=ParentType AND t1.PARTS_CODE=PartsCode AND t1.UPLOAD_ID=UploadID;
   ELSE
		SELECT count(ID) INTO count_id from T_CMES_PROJECT_COOPERATE_USE AS t2 WHERE t2.Parent_Type=ParentType AND t2.PARTS_CODE=PartsCode AND t2.UPLOAD_ID=UploadID AND  t2.DEFECT_SRC_CODE=DefectSrcCode;
   END if;
   return count_id;
END $$
```



##### 3. case函数

- 使用一：java中switch case 效果

  java中：

  switch(变量或表达式){

  ```
  case 常量1：语句1；break；
  
  case 常量2：语句2；break；
  
  default： 语句n；
  ```

  }

  mysql中：

  case 要判断的字段或表达式

  when 常量1 then 要显示的值1或语句1；  (then后面为值，不需要分号)

  when 常量2 then 要显示的值2或语句2；

  ......

  else 要显示的值n或语句n；

  end

  例：查询员工的工资，要求: 

  ```
  部门号30, 显示为原工资1.1倍
  
  部门号40, 显示为原工资1.2倍							
  
  部门号50, 显示为原工资1.3倍
  
  其他部门，显示为原工资
  ```

  ```mysql
  select salary 原始工资，department_id,
  case department_id
  when 30 then salary*1.1
  when 40 then salary*1.2
  when 50 then salary*1.3
  else salary
  end as 新工资 from employees；
  ```

- 使用二：多重if  else if 效果

  case

  when 条件1 then 要显示的值或语句1

  when 条件2 then 要显示的值或语句2

  ......

  else 要显示的值n或语句n

  end

  例：查询员工的工资情况

  ```
  如果工资>20000, 显示A级别
  
  如果工资>15000,显示B级别
  
  如果工资>10000,显示C级别
  
  否则，显示D级别
  ```

  ```mysql
  select salary，case
  when salary>20000 then 'A'
  when salary>15000 then 'B'
  when salary>10000 then 'C'
  else 'D'
  end as 工资级别 from employees;
  ```

## 分组函数

用作统计使用， 又称为聚合函数或统计函数或组函数

分类：sum，avg，max，min，count

特点：

```
1.sum， avg一般用于处理数值型；max，min，count可处理任何类型

2.以上分组函数都忽略null值，（null值加任何数都为null）

3.和distinct搭配

sum(distinct salary)  去重后求和，其他函数都支持 

4.和分组函数一同查询的字段要求是group by后的字段
```

- ##### count() 函数	

  count(字段名)  统计该字段不为空的记录行数

  count(*)  统计表中所有记录行

  count(1)  统计表中所有记录行（实质插入一列值都为1的字段，统计1的个数，也可填其他常量）

  **效率**：myisam存储引擎下，count(*) 效率最高(myisam中有个内部计数器，直接返回记录个数)；

  innodb存储引擎下，count(\*)和count(1)效率差不多，比count(字段名)高一些(**有一个判断不为null的过程**)。



# 二，增删改查

## 查询

### 1.连接查询

sql92标准：mysql中仅支持内连接，orecle支持外连接

sql99标准：mysql支持内连接+外连接(左，右)+交叉连接

不支持全外连接； 连接使用 join on 关键字

#### 等值连接

```mysql
select salary from A，B where A.id=B.id （sql92）

select salary from A join B on A.id=B.id （sql99）
```

#### 非等值连接

```sql
select salary, level from A, C where salary between C.low_level and C.high_level;  
```

#### 自连接

```mysql
# sql92
select e.employee_id, e.last_name, m.employee_id, m.last_name
from employees e, employees m
where e.manager_id = m.employee_id;

# sql99
select e.employee_id, e.last_name, m.employee_id, m.last_name
from employees e
join employees m
on e.manager_id = m.employee_id;
```

#### 全外连接(mysql不支持)

```mysql
select A.*, B.*
from A
full outer join B
on A.id = B.id;
```

### 交叉连接(就是实现了笛卡尔乘积)

```mysql
select A.*, B.*
from A
cross join B;
```

### 2.子查询

#### 行子查询

用的较少，有局限性，要求所有筛选条件都用相同符号=，> 或<

```mysql
# 查询员工编号最小且工资最高的员工信息
select * from employees
where (employees_id,salary)=(
	select min(employee_id),max(salary)
	from employees
)
```

#### select后面

```mysql
# 查询每个部门的员工个数
## 先从员工表查出总数，帅选条件为员工表的部门id=部门表的部门id，再从部门表查询所有信息
select d.*,(
	select count(*)
	from employees e
	where e.department_id = d.department_id
) as 个数
from departments d；
```

#### exists后面  (exists查询又叫相关子查询)

select exists(select employee_id from employees);   返回0或1

```mysql
# 查询有员工的部门名
##方法一
select department_name
from departments d
where exists(
	select *
	from employees e
	where d.department_id=e.department_id
);
##方法二  能用exists查询一般也能用in查询
select department_name
from departments d
where d.department_id in (
	select department_id 
    from employees
)
```

### 3.联合查询

```mysql
# 语法
查询语句1
union
查询语句2
union
......;
```

**应用场景: **要查询的结果来自多个表，且多个表没有直接的连接关系，单查询的信息一致。

**特点：**

1.要求多条查询语句的查询列数是一致的

2.要求多条查询语句查询的每一列的类型和顺序最好一致

3.union关键字默认去重，可使用union all 包含重复项。

```mysql
SELECT 'aa' as ab, 'MES+生产质量问题' AS '预警类型',sum(CASE product when 'AAS' then 1 end) as AAS,sum(CASE product when 'CDMA' then 1 end) as CDMA from t_summarizewarning where w_type='MES+生产质量问题'
UNION all
SELECT 'bb','MES+生产质量问题2' as '预警类型',sum(CASE product when 'AAS' then 1 end) as AAS,sum(CASE product when 'CDMA' then 1 end) as CDMA from t_summarizewarning where w_type='MES+生产质量问题'
```

![1566782188572](C:\Users\YWX800~1\AppData\Local\Temp\1566782188572.png)

## 插入

#### 1.插入方式

**方式一：insert into table(字段1, 字段2,...) values(值1,值2...);**

```
列名可不写，默认为所有字段，顺序与数据库列名一致；
字段个数必须与值的个数保持一致。
```

**方式二：insert into table set 字段1=值1，字段2=值2，... **

**两方式对比：**

1.方式一支持插入多行，方式二不支持

2.方式一支持子查询，方式二不支持

```mysql
insert into table(id,name,phone)
select 26,'张三'，'1388888' from tb_A;
```

## 更新

#### 1.一条SQL同时更新多条记录的同一个字段为不同的值

```mysql
UPDATE categories
    SET display_order = CASE id
        WHEN 1 THEN 3
        WHEN 2 THEN 4
        WHEN 3 THEN 5
    END,
    title = CASE id
        WHEN 1 THEN 'New Title 1'
        WHEN 2 THEN 'New Title 2'
        WHEN 3 THEN 'New Title 3'
    END
WHERE id IN (1,2,3)
```



# 三，锁

### 查询加锁

必须要开启事务查询

```sql
BEGIN;
-- 加读锁S（共享锁）
select * from tt  where id='123' lock in share mode;
-- 加写锁X（排他锁）
select * from tt  where id='123' for update;
COMMIT;
```

- 查询加锁若用到索引，innodb默认加行锁，否则锁整个表。
- 行锁时，若查询记录为空，则不会加锁；
- 表锁时，即使查询记录为空，也会锁表；



# 四，事务

语法：

```mysql
# 1.开始事务
set autocommit=0； # 关闭自动提交，开启显示事务。
start transaction；# 可不写
语句。。。
# 2.结束事务
commit；  # 成功，则提交
rollback； # 失败，回滚

# 演示savepoint 的使用--回滚点，搭配rollback
set autocommit=0；
start transaction；
delete from table where id=2；
savepoint a；# 设置保存点
delete from table where id=4；
rollback to a； # 回滚到保存点
```

## 数据库的隔离级别

1.对于同时运行的多个事务，若不采取隔离机制，访问相同数据时会出现并发问题。

2.同一个事务中对同一条数据的多次读取，不能因为其他事务改变了该数据而发生改变。

**脏读：**T1修改未提交，此时T2读取该数据，若之后T2回滚，则T2读取的是临时且无效的数据。

**不可重复读：**T1读取了一个字段，之后T2更新了该字段，之后T1再次读取同一字段，值就不同了。

**幻读：**T1读取了一个字段，然后T2再在该表插入了新行。如果T1再次读取该表就会多出几行。

### mysql数据库提供四种事务隔离级别

1. **READ UNCOMMITTED  (读未提交数据)**

   允许事务读取未被其它事务提交的变更：脏读，不可重复读和幻读的问题都会出现。

2. **READ COMMITTED  (读已提交数据)**  (oracle默认事务隔离级别)

   只允许事务读取已经被其它事务提交的变更，可以避免脏读，但不可重复读，幻读问题仍可能出现。

3. **REPEATABLE READ  (可重复读)  (mysql默认事务隔离级别) **

   确保事务可以多次从一个字段中读取相同的值，在这个事务持续期间，禁止其他事务对这个字段进行更新，可以避免脏读和不可重复读，但幻读的问题仍然存在。

4. **SERIALIZABLE(串行化)**    （oracle支持）

   确保事务可以从一个表中读取相同的行，在这个事务持续期间，禁止其他事务对该表执行插入，更新和删除操作，所有并发问题都可以避免，但性能十分低下。

- 每启动一个mysql程序，会获得一个单独数据库连接，该连接会有一个全局变量@@tx_isolation,表示当前事务隔离级别
- 查看当前的隔离级别：select @@tx_isolation;
- 设置当前mysql连接的隔离级别：set session transaction isolation level 级别名;
- 设置数据库系统的全局隔离级别：set global transaction isolation level 级别名；（可能需要重启生效）



# 五，视图

特点：视图相当于一个虚拟表，临时的；可以直接当作表来使用。**创建的视图只保存sql逻辑，不保存查询结果，**是使用视图时动态生成的。

- 重用sql语句
- 简化复杂的sql操作，不必知道它的查询细节
- 保护数据，提高安全性。(没法知道基表的内容)

### 创建视图：

```mysql
create view myview1
as
select * from table；
```

### 修改视图（结构）：

```mysql
# 方式一
create or replace view myview1
as
查询语句；

# 方式二
alter view myview1
as
查询语句；
```

### 删除视图

drop view 视图名，视图名，......;

### 查看试图

使用desc： desc myview；

show create view myview3；

### 视图的更新（数据）（基本都不行，没啥应用场景，视图基本用作查询）

和表的增删改查操作一样，更新视图数据时，原表数据也会更新；

具备以下特点的视图不允许更新（基本都不行，没啥应用场景）：

- ...省略



# 六，变量

## 系统变量

**说明**：由系统提供，分为 **全局系统变量 和 会话系统变量**

**作用域**：

全局变量：服务器每次启动将为所有全局变量赋初始值，针对所有会话连接接，但不能跨重启（需修改配置文件）；

会话变量：仅仅针对于当前会话连接有效；

- 查看所有的系统变量

  show global |【session】 variables；

- 查看满足条件的部分系统变量

  show global |【session】variables like '%char%';

- 查看指定的某个系统变量的值

  select @@global |【session】.系统变量名；

- 为某个系统变量赋值

  - 方式一：set global |【session】系统变量名=值；
  - 方式二：set @@global|【session】.系统变量名=值；

**注意**：如果是全局级别，使用global；若是会话级别，则使用session；若不写，默认session；



## 自定义变量

**说明**：是由用户自定义，分为 用户变量 和 局部变量

**用户变量 **：针对当前会话连接有效，等同于会话变量的作用域

赋值的操作符：= 或 :=

1.声明并初始化

```
set @用户变量名=值；  或：

set @用户变量名:=;  或：

select @用户变量名:=值；

```

2.赋值（更新用户变量的值）

方式一：通过set和select

```
	set @用户变量名=值；  或：

	set @用户变量名:=;  或：

	select @用户变量名:=值；

```

方式二：通过select into

```
	select 字段 into 变量名 from table；

```

3.使用（查看用户变量的值）

```
select @用户变量名；

```

**局部变量：**仅仅在定义它的begin end中有效；应用在begin end中的第一句话；

1.声明

declare 变量名 类型；

declare 变量名 类型 default 值；

2.赋值

和用户变量一样，变量名前无需@符号；

3.使用

select 变量名；



# 七，存储过程

### 创建语法

```mysql
delimiter $
create procedure myp1()
begin
	语句；
end $
```

- 1.参数列表包含三部分： 参数模式 参数名 参数类型

  举例：in stuname vachar(20)

  参数模式：

  in：该参数可以作为输入，也就是该参数需要调用方传入值

  out：该参数可以作为输出，也就是该参数可以作为返回值

  inout：该参数既可以作为输入又可以作为输出，也就是该参数既需要传入值，又可以返回值

- 2.如果存储过程体仅仅只有一句话，begin end可以省略;

  存储过程体中的每条sql语句的结尾要求必须加分号；

  ```
  **存储过程的结尾符号可以使用 delimiter 重新设置**
  
  ```

```
语法：delimiter 结束标记
举例：delimiter $

```

### 调用语法

call 存储过程名(实参列表)；

#### 1.空参列表

```mysql
# 案例：插入到admin表中五条记录
delimiter $
create procedure myp1()
begin
	inser into admin(username,password) values('john1', '0000'),	('lily',0000),('rose','0000'),('jack','0000'),('tom','0000');
end $

# 调用
call myp1()$
```

#### 2.带in模式参数

```mysql
# 案例1：创建存储过程，根据女神名，查询对应的男神信息
create procedure myp2(in beautyName varchar(20))
begin
	select bo.*
	from boys bo
	right join beauty b on bo.id = b.boyfriend_id
	where b.name=beautyName;
end $
## 调用
call myp2('柳岩')$

# 案例2：创建存储过程实现，用户是否登录成功(多个in模式参数)
create procedure myp3(in username varchar(20),in password varchar(20))
begin
	declare result int default 0; # 声明变量并初始化
	
	select count(*) into result  # 赋值给变量
	from admin
	where admin username = username
	and admin.password = password;

	select if(result>0,'成功'，'失败');# 使用变量
end $
## 调用
call myp3('张飞'，'8888')$
```

#### 3.带out模式参数

```mysql
# 案例1：根据女神名，返回对应的男神名

create procedure myp5(in beautyName varchar(20),out boyName varchar(20))
begin
	select bo.boyName into boyName  # 将结果赋给返回值变量
	from boys bo
	inner join beauty b on bo.id = b.noyfriend_id
	where b.name=beautyName;
end $
## 调用
set @bName$ #可不写,定义用户变量（不在begin end内不能定义局部变量）
call myp5('小明'，@bName)$
select @bName$  

# 案例2：根据女神名，返回对应的难神名和男神魅力值(多个out参数)
create procedure myp6(in beautyName varchar(20),out boyName varchar(20),out userCP int)
begin
	select bo.boyName,bo.userCP into boyName,userCP  # 赋值
	from boys bo
	inner join beauty b on bo.id = b.noyfriend_id
	where b.name=beautyName;
end $
## 调用
call myp6('小昭',@bName,@usercp)$
select @bName,@usercp$
```

#### 4.带inout模式参数

```mysql
# 案例：传入a和b两个值，最终a和b都翻倍返回
create procedure myp7(inout a int,inout b int)
begin
	set a=a*2；
	set b=b*2；
end $
## 调用
set @m=10$  # 此时必须先声明变量并初始化
set @n=(20)$
call myp7(@m,@n)$ # inout模式需要传入值，还要传入返回值的变量
select @m,@n$
```



# 八，自定义函数

### 基本语法

```mysql
DELIMITER $$  # 定义结束符，函数中的语句需要使用‘;’会冲突
DROP FUNCTION IF EXISTS GetDPRatePTByCodingID $$  # 函数存在则先删除
CREATE FUNCTION GetDPRatePTByCodingID(CodingId VARCHAR(50)) RETURNS VARCHAR(50)
BEGIN
	DECLARE dpratept VARCHAR(50); # 声明变量
	SELECT T_ProductInfoNew.dpratept INTO dpratept from T_ProductInfoNew WHERE T_ProductInfoNew.CodingId=CodingId LIMIT 1; # 得到的结果赋值给声明的变量
	RETURN dpratept; # 返回值
END $$
```



# 九，其他知识点

### any 和 all

```sql
-- any 任意一个  
select * from A where id in(1,2,3)  等同于：
select * from A where id=any(1,2,3)

-- all  所有
select * from A where salary>all(1000,2000,3000) 相当于大于最大值
select * from A where salary <> all(1000,2000,3000)
```


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



