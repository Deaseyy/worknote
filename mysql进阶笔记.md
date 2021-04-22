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



