                                          # 收录一些常用的数据处理操作：包括numpy，pandas。

## 一，numpy

> np.where(pd.isna(x), y, x)：条件判断，表达式为True返回 y 反之 x

> np.argsort()：返回排序后的下标

> arr.tolist()：将array转为列表



#### DataFrame 和 numpy函数之间操作

> log,  exp,  sqrt

```
np.log(df)
np.sqrt(df)
```

> np.asarray(df)：df转为ndarray



## 二，series

> s.shift(1) : 将index对应的值移位

> s.value_counts() : 统计每个值的个数，返回一个Series

> s.str.lower() : 转小写，NaN不作处理

> s.dtype：查看数据类型

> s.to_numpy()  和  np.asarray(s) ： Series 转化为 ndarray

> s.to_dict()：Series 转化为字典

```
ser.to_numpy(dtype="datetime64[ns]")
ser.to_numpy(dtype=object)
```

> s.sort_values() ：排序

- na_position：NAN值排在前面

> searchsorted() : 返回指定所有元素的索引位置，组成一个ndarray，
>
> 元素不存在：若小于最小值 返回0， 大于最大值返回最大索引加1

> s.nsmallest(n)：返回最小的n个值
>
> s.nlargest(n)：返回最大的n个值



> s.isin([2, 4]) ：是否满足指定条件（True 或 False），返回一个series
>
> 逆函数not in 表示：~(df.x.isin(['a', 'b'])

> s.rank() : 对每个值进行排名 <https://blog.csdn.net/starter_____/article/details/79183595>

- 默认升序排名，相同大小的值会排在同名次
- method值为：
  - first  值相同时，根据值在原数据中出现的顺序排名
- ascending=False  降序  （method='max'）
- 可用于df分组后。对指定列（可多列）排名



## 三，pd

待添加。。。



## 四，DataFrame

> df.to_numpy() : df转ndarry

> df.to_dict(orient='records')：df转字典（列表套字典）

> df.nsmallest(n, 'c')：以c列为标准，取值最小的n行数据
>
> df.nlargest(n, 'a')：以a列为标准，取值最大的n行数据

> df.select_dtypes('bool')：根据数据类型查找数据

> df.pop('A') ：从原df删除并返回该列
>
> df.insert(1, 'bar', df['one'])： 在指定位置插入新的列

> **df.query('two>2') ：查询满足条件的结果**

```
dfdf.query('a <= b')  等价于：df[df.a <= df.b]
```

> df.eval('a + b')：可以用于列之间的计算

> df.groupby(['by1', 'by2']) ： 分组

```
df.groupby(['Animal', 'FeedType'])['Amount'].sum()
```



> df.set_index(['keys'])：将指定列(可多列)设置为索引
>
> df.reset_index()：将索引还原为列

#### 类型转换

> astype()：转换类型

```
df.astype({'a': np.bool, 'c': np.float64})
```

> pd.to_numeric(arr)
>
> pd.to_datetime(arr)
>
> pd.to_timedelta(arr)



#### 排序

> reindex()：重排索引，将数据按指定索引顺序显示, 若索引不存在，填充NaN

series:  s.reindex()

dataframe:  df.redindex()

- axis: index 或 columns
- method：填充方式

df.reindex(['c', 'b', 'd', 'a','f'], axis='index')



> df.sort_index(axis=0, ascending=False) ： 将列或行按索引值大小排序

- axis： 0 行，1 列
- ascending：升序或降序
- inplace

s.sort_index()

df.sort_index()



> df.sort_values(by='B') : 根据某一列进行排序

- by：列字段
- ascending：升序或降序
- inplace
- na_position：空值排在最前面

```
df.sort_values(by='B')
df.sort_values(by=['A', 'B'])
s.sort_values()
```



> df['E'].isin(['two', 'four']) :  检查该列，返回一个series，值在列表中为True，反之False

```
df[df['E'].isin(['two', 'four'])]
```



> s1.idxmin() 
>
> s1.idxmax()：求最大或最小值对应的索引，值为多个时取第一个



> pd.date_range()  : 生成日期列表(DatetimeIndex类型)

- start

- end 

- periods: 日期个数

- freq：时间间隔，'D' 日期(默认)   'S' 日期时间  '5D' (5天)   '5S'(5秒)  '5T'(5分)

  ```
  pd.date_range('2020-05-11', periods=6)
  ```



> drop()：删除行或列

- axis：0 行，1 列

  df.drop(['a', 'd'], axis=0)

  df.drop(['one'], axis=1)



> rename() : 索引重命名

- inplace ：是否在原df上修改
- axis：index 或 columns

全改：得全写上
df.columns = ['a','b','c']  
部分改:
df.rename(columns={'A':'a', 'B':'b', 'C':'c'}, inplace = True)

df.rename({'A': 'apple', 'B': 'banana', 'C': 'durian'}, axis='index')

#### 函数应用

> df.apply(func)：将列或行 作用于函数，返回作用后的结果

```python
# 返回每列的平均值
df.apply('mean', axis=0) 
df.apply(np.mean, axis=0) 
# 自定义函数
def subtract_and_divide(x, sub, divide=1):
	return (x - sub) / divide
df.apply(subtract_and_divide, args=(5,), divide=3)
```

> df.agg(func)：聚合API, 用法和apply相同

```
- Aggregating with single functions
  df.agg('mean') 等价 df.apply('mean') 等价 df.mean()

- Aggregating with multiple functions
  df.agg(['sum', 'mean'])  返回dataframe

- Aggregating with a dict
  df.agg({'A': 'mean', 'B': 'sum'}) 不同列分别使用不同聚合函数
  df.agg({'A': ['mean', 'min'], 'B': 'sum'})
```



> df.transform(func)：转换API, 用法和agg相似，但不能用于聚合函数

- df.A.transform(np.abs)
- df.transform({'A': np.abs, 'B': [lambda x: x + 1, 'sqrt']})

> map()：用法和agg，transform类似，series的方法，只能作用于单列数据

```
df.A.map(np.abs)
```



> applymap()：用法同map，dataframe的方法，作用于整个dataframe

```
df.applymap(np.abs)
```



> df.assign(new_col=df['old1'] / df['old2']：通过已有列快速添加新的列，返回一个copy副本

```
df.assign(new_col=lambda x: (x['old1'] / x['old2']))

df.assign(C=lambda x: x['A'] + x['B'],D=lambda x: x['A'] + x['C'])
```

> df.replace({a: b} , inplace = True)：修改某个值



#### 处理空值

> df.dropna(how='any') ： 删除行

- how：any 删除包含空值的行，  all 删除所有列为空值的行



> df.fillna(value=5) ： 填充空值



> pd.isna(df): 对每项做空值判断



> df.groupby(['A', 'B']).sum() ： 分组并求和



#### 运算

> df.sub(df.iloc[1], axis='columns') : 减法操作

- axis： columns 或 1,  index 或 0



> df.add(df2, fill_value=0) ：加法操作 



> df.mean(axis=0)



> df.sum(axis=0, skipna=False)

- skipna：跳过空值



> eq, ne, lt, gt, le,  ge 对df每个值做比较运算
>
> df.gt(df2)



> df.equals(df2)：比较两个df是否相等

```
(df + df).equals(df * 2)
```



> all() ：全True则True
>
> any()  ：任意一个True则True

```
(df > 0).all()
(df > 0).any().any()
```

> df.empty：判断df是否为空

- 当需要条件判断时可以使用 if df.empty 或 df.any() or df.all()



| Function | Description                                |
| -------- | ------------------------------------------ |
| count    | Number of non-NA observations              |
| sum      | Sum of values                              |
| mean     | Mean of values                             |
| mad      | Mean absolute deviation                    |
| median   | Arithmetic median of values                |
| min      | Minimum                                    |
| max      | Maximum                                    |
| mode     | Mode                                       |
| abs      | Absolute Value                             |
| prod     | Product of values                          |
| std      | Bessel-corrected sample standard deviation |
| var      | Unbiased variance                          |
| sem      | Standard error of the mean                 |
| skew     | Sample skewness (3rd moment)               |
| kurt     | Sample kurtosis (4th moment)               |
| quantile | Sample quantile (value at %)               |
| cumsum   | Cumulative sum                             |
| cumprod  | Cumulative product                         |
| cummax   | Cumulative maximum                         |
| cummin   | Cumulative minimum                         |

#### 连接合并操作

> combine_first()

```
# 用df2的值去填补df1中不存在或为NaN的值
df1.combine_first(df2)
```





#### 其他

> align()  最快速的将两个对象排成一行

- join：inner,  left,  right
- axis

> df.rename_axis()：重命名索引列或行的name名称

```
df.rename_axis(index={'let': 'abc'})

df.rename_axis(index=str.upper)
```



> df.iterrows() : 返回一个df行内容(Series对象)的可迭代对象

- 注意：迭代对象返回的是一个copy副本，不是引用视图，不能试图去改变它的值

```python
# 这是无效的
for index, row in df.iterrows():
	row['a'] = 10
```

> df.items()：返回一个df列内容(Series对象)的可迭代对象



> .dt：日期存取器

```
s.dt.hour
s.dt.year
s.dt.date
s.dt.strftime('%Y/%m/%d')
```

> .str：字符串存取器，只能作用于字符串；

```
s.str.lower()
s.str.len()
s.str.rstrip()
s.str.find("a")
s.str[0:1]
s.str.split(" ", expand=True)[0]
```



## 五，对比SQL

#### 1.SELECT

```
SELECT total_bill, tip, smoker, time FROM tips LIMIT 5;

tips[['total_bill', 'tip', 'smoker', 'time']].head(5)
```



#### 2.WHERE

```python
SELECT * FROM tips WHERE time = 'Dinner' LIMIT 5;
tips[tips['time'] == 'Dinner'].head(5)

# 多条件连接：OR(|) , AND(&)
SELECT * FROM tips WHERE time = 'Dinner' AND tip > 5.00;
tips[(tips['time'] == 'Dinner') & (tips['tip'] > 5.00)]

# NULL值检测： notna() ， isna() 
SELECT * FROM frame WHERE col2 IS NULL;
frame[frame['col2'].isna()]
```



#### 3.GROUP BY

first()：分组后取每组第一行

```python
# size()统计每组的记录数，返回series
SELECT sex, count(*) FROM tips GROUP BY sex;
tips.groupby('sex').size()

# count() 分别统计每组的每一列不为空的记录数，返回dataframe
tips.groupby('sex').count()
# count() 统计每组特定列不为空的记录数，返回series
tips.groupby('sex')['total_bill'].count()

# 一次多个函数
SELECT day, AVG(tip), COUNT(*) FROM tips GROUP BY day;
tips.groupby('day').agg({'tip': np.mean, 'day': np.size})

# 根据多列分组
SELECT smoker, day, COUNT(*), AVG(tip) FROM tips GROUP BY smoker, day;
tips.groupby(['smoker', 'day']).agg({'tip': [np.size, np.mean]})
```



#### 4.JOIN

join() 和 merge() 都能实现 SQL的join

```python
# 可以用参数how 指定连接方式：inner(默认),left,right,outer
SELECT * FROM df1 INNER JOIN df2 ON df1.key = df2.key;
pd.merge(df1, df2, on='key',how='inner')

# 用一个df的列去连接另一个df的索引
pd.merge(df1, df2, left_on='key', right_index=True)

参数： suffixes=['', '_sum']
```



#### 5.UNION

用concat() 可实现SQL的 union all，

union会移除重复记录，可以concat后使用去重方法 drop_duplicates()

```python
# UNION ALL
SELECT city, rank FROM df1 UNION ALL SELECT city, rank FROM df2;
pd.concat([df1, df2])

# UNION
SELECT city, rank FROM df1 UNION SELECT city, rank FROM df2;
pd.concat([df1, df2]).drop_duplicates()
```



#### 6.limit n offset m

```python
# 取10个， 从第五个开始取
SELECT * FROM tips ORDER BY tip DESC LIMIT 10 OFFSET 5;
tips.nlargest(10 + 5, columns='tip').tail(10) # 先取15，再取后10个
```



#### 7.UPDATE

```
UPDATE tips SET tip = tip*2 WHERE tip < 2;
tips.loc[tips['tip'] < 2, 'tip'] *= 2
```

#### 8.DELETE

pandas中，应该反过来选择要保留下来的值，排除不要的，而不是删除

```
DELETE FROM tips WHERE tip > 9;
tips = tips.loc[tips['tip'] <= 9]
```



## 六，dataframe条件筛选

#### 1，按条件取值有两种方法

```
df[df.A > 5]['B']
1    3
2    6
Name: B, dtype: int64
```

或者

```
df.loc[df.A > 5, 'B']
1    3
2    6
Name: B, dtype: int64
```

**loc操作可以返回view，直接进行修改赋值**



#### 2，根据多条件取值

1.使用query()方法

2.使用链式切片或索引

df[(df.c1==1) & (df.c2==1)]   # 普通，直接赋值会报SettingWithCopyWarning警告，可使用loc

df.A.isin(['foo']) & df.B=='two'  # 使用loc ，返回view 可直接操作原dataframe



#### 3，关于条件筛选取值时注意事项

##### 3.1，& ，| ，and，or的使用

- 在 .loc[] 中  多条件联合只能使用 &,  | 操作符
- 在query()方法中，条件联合查询 可使用 and(&), or(|) ，效果一样

##### 3.2，多条件联合时，每个条件必须使用括号括起来

形如：df[(df.c1==1) & (df.c2==1)] 

不然会报警告，并且筛选失败：

```
FutureWarning: elementwise comparison failed; returning scalar instead, but in the future will perform elementwise comparison
  res_values = method(rvalues)

```































