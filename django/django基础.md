## 本地安装库文件 

将下载的库解压，进入到setup.py所在目录，运行：

python setup.py insatll   (可能会提示需要先安装依赖包)





# 在django代码中使用settings

1.通过引入 `django.conf.settings` 使用配置，settings是一个对象

例如：settings.DEBUG

2.不建议运行时更改配置,应该只在settings文件中更改设置.:

例如：视图函数中使用： settings.DEBUG=True 不建议

3.python manage.py diffsettings  查看当前配置与默认配置的不同



# 有用技巧

### 1.查看一段代码中，orm执行的所有SQL代码及耗时

```python
from django.db import connection,connections
u = User(name=u'河南', code='0371')
u.save()
print(connection.queries)
# 指定数据库
print(connections['StockManagement_db'].queries)

>>> 将打印执行该段代码过程中，执行过的所有sql语句，和耗时；

```

### 2. 更新时使用模型中的字符串和其他字符串拼接

```python
from django.db.models import Value
from django.db.models.functions import Concat

Model.objects.update(aaa=Concat('aaa', Value('bbb')))
```





# 一，模型

#### obj.save() 方法

该记录存在时，更新该记录字段属性

不存在则创建一个新记录到数据库

```python
obj = Snippet.objects.create(id=1, title='预警1')
obj.id=132  # 此操作会切换一个对象（id为1的对象切换为id为132的对象）
obj.save() # 此时若存在id为132的记录，则更新这条记录，title会变为'预警1'，若不存在，则创建一条新记录，title为'预警1'
```

**注意，通过obj.save() 更新时包含主键字段，如id，则不会更新该记录的id，而是查看该id的记录是否存在，存在则更新其他字段，否则重新生成一条记录**



## 1.根据已经存在数据库中的表自动生成模型

```
步骤1：设置setting中数据库的连接
步骤2：在终端（只能是终端）执行命令 python manage.py inspectdb > model.py

	1.直接将打印的代码直接导入到指定的Model文件中
	python manage.py inspectdb > student/models.py  # 前提是创建了app(student)并且在			setting.py文件中注册过
	
	2.配置了多个数据库，则还可以配置数据库别名来指定根据哪个库中的表来生成Model
	python manage.py inspectdb --database default >student/models.py  # default是默认的		别名
	
	3.将指定的表生成对应的Model
	python manage.py inspectdb --database default table1 table2 >student/models.py
	注意：导出model文件路径要正确
	注意：inspectdb反导入模型完成后，不要急着更改模型，先将版本号映射到数据库，下面一切操作完后再回头正常操作

步骤3：修正模型：
　　模型名；
　　模型所属APP；
　　模型外键；
　　让django管理模型：Meta下的managed=False删除；
　　多对多模型使用ManytoManyField；中间表删除
步骤4：执行命令 
   python manage.py makemigrations  生成初始化脚本
　 python manage.py migrate --fake-initial  将版本号映射到数据库中，而不用执行sql命令
步骤5：将Django的核心表映射到数据库中，比如auth表，session表等
若出现已存在表错误，可将数据库migration表和项目的migrations文件删除后重新迁移


```



## 2.建立联合主键

```python
class Stu(models.MOdel):
	......
	class Meta:
		unique_together = ('name', 'addr') # 由姓名，地址一起决定一条学生记录
```



## 3.关联关系

不使用外键是指数据库层面的外键约束，而Django ORM所使用的是查询引擎的查询逻辑。

所以要关闭数据库层面的外键约束，只需要使用`db_constraint`参数设置为`False`即可。

### 一对一

例：stu 表和 info表：

```python
# stu表中（一）
class Stu（model.Models）：
	name =...
	......
	info = models.OneToOneField(Info,related_name='stu',
                                ondelete=models.CASCADE,null=True)
	
# info表中（一）
class Info（model.Models）：
	addr = ...
    ...... 
```

**通过对象查询：**

正向：addr = Stu.objects.get(id=1).info.addr

反向：name = Info.objects.get(id=1).stu.name

**通过queryset：**

正向：queryset = Stu.objects.all().values('info')   默认只取info表id字段

```
   queryset = Stu.objects.all().values('info__addr') 
```

反向：queryset = Info.objects.all().values('stu__name')

### 一对多

例：author表和article表：

```python
# author表中（一）
class Author（model.Models）：
	name = ......

# article表中（多）
class Article（model.Models）:
    title = ......
	author = models.ForeignKey(Author,related_name='article',
	on_delete=models.CASCADE,null=True)
```

查询方式同一对一

### 多对多

例：stu和course表

会默认生成一个中间表stu_course,仅包含两表的id字段

正向查询：stu_obj.course.all()

### 注意： 反向查询时：

一对一：若无指定related_name字段，则反向查询字段默认使用模型名的小写（表名）

一对多：若无指定related_name字段，则反向查询字段默认使用模型名的小写_set；  例：foo_set

多对多：若无指定related_name字段，则反向查询字段默认使用模型名的小写_set；  例：foo_set



## 4.字段

1.**null** 默认false，true将NULL在数据库存储空值

2.**blank **默认False，True，该字段允许为空		

注意， [`null`](../../ref/models/fields.html#django.db.models.Field.null)纯粹与数据库相关，而 [`blank`](../../ref/models/fields.html#django.db.models.Field.blank)与验证相关。如果字段有 [`blank=True`](../../ref/models/fields.html#django.db.models.Field.blank)，则表单验证将允许输入空值。如果字段有[`blank=False`](../../ref/models/fields.html#django.db.models.Field.blank)，则需要该字段。

3.**choice** 每个元组中的第一个元素是将存储在数据库中的值。第二个元素由字段的窗体小部件显示。

获取设置了choice属性字段的显示值

```
obj.get_字段名_display()
```

4.**primary_key**  若需指定自定义主键，只需添加primary_key=True到指定字段 。如果Django看到你明确设置[`Field.primary_key`](../../ref/models/fields.html#django.db.models.Field.primary_key)，它将不会添加自动 `id`列

主键字段是只读的。如果更改现有对象上的主键值，然后保存它，则将创建一个与旧对象并排的新对象

## 5.查询集queryset

**django的queryset在每次for循环时是取缓存的，也就是第一次for时会发出sql请求，以后的for循环会使用已请求的缓存。但是在queryset里面再次get(filter)会再次发起sql查询。**

#### 以下情况会计算queryset，去操作数据库

- 打印，比如print() 

- 迭代：第一次迭代它时会去数据库查询

  - 若只想确定是否存在至少一个结果，使用exists().

- 切片：当带有 step 参数时，将执行数据库查询

- len()：对其使用len方法，将执行数据库查询

- list()：对其使用list方法强制转换，将执行数据库查询

- bool()：布尔类操作 bool(), or, and, not，if语句

  - if ： 使用if对其判断时，将执行数据库查询

    例如：if queryset：pass   ;  可使用queryset.esists()

- pickle



## 6.查询方法

**ORM查询对应sql使用.query. __ str__ ()**

- 对时间的查询：

  - table.objects.filter(create_date__range=(start_time,end_time))
  - table.objects.filter(create_date__year='2019')

- **get(): 会直接查询数据库(包括first等);将缓存普通的模型字段，使用外键字段每次都会导致DB查找**

- **filter(): query_set是惰性获取的，创建 query_set 的操作，不会进行任何数据库查询**

- **latest('create_date')**:根据给定的字段返回表中的最新对象。比如： 获取时间最新的一条记录（且必须存在）

  - table.objects.filter().latest('create_date')
  - 注意：空的queryset调用该方法会报错；

- **range()** : 范围查找,类似MySQL的between and

  - table.objects.filter(create_date__range=(start_time,end_time))

- **isnull()** : 值为True或False，相当于SQL is null和is not null

  - table.objects.filter(create_date__isnull=True)

- **startswith，istartswith(不区分大小写)，endswith，iendswith(不区分大小写)，contains，icontains**

- **only('id')**：  仅查找出所需要的字段，还是返回一个包含所有对象的queryset，取出的对象只含有指定字段，但依然可以用‘.’取没有的字段,但需要重新去查询数据库。

- **defer()**：与only方法相反

- **union()**: 合并多个queryset；

  - 返回第一个QuerySet类型的模型实例, 操作都是基于第一个queryset类型
  - 只允许对结果QuerySet执行切片、count(）、order_by()和values()/values_list()操作
  - q1.union(q2,q3)

- **annotate()** ：使用聚合函数，(适用于分组查询等，默认按id分组)

  **单表查询**

  ```mysql
  # 默认按id分组， 分组后返回queryset对象，包含分组后每一个模型对象
  queryset.annotate(numm=Count('*')) # orm  <QuerySet [<SummarizeWarning: SummarizeWarning object (1)>, <SummarizeWarning: SummarizeWarning object (2)>...]
  SELECT id,w_type,product,...,COUNT(*) AS numm FROM t_s GROUP BY id ORDER BY NULL;  #sql
  
  # 使用其他字段分组 使用values() # 返回queryset对象， 包含values指定分组字段和聚合字段组成的字典，可以继续使用values()取指定字段
  ##指定id分组
  queryset.values('id').annotate(numm=Count('*')) # <QuerySet [{'id': 1, 'numm': 1}, {'id': 2, 'numm': 1}]>
  SELECT id, COUNT(*) AS numm FROM t_s GROUP BY id ORDER BY NULL # sql
  
  ##指定w_type分组
  queryset.values('w_type').annotate(numm=Count('*')) # <QuerySet [{'w_type': '类型一', 'numm': 5497}, {'w_type': '类型二', 'numm': 1}]>
  SELECT w_type, COUNT(*) AS numm FROM t_s GROUP BY w_type ORDER BY NULL;  #sql
  
  ##指定多个字段分组
  queryset.values('w_type','product').annotate(numm=Count('*')) # <QuerySet [{'w_type': '类型一', 'product': 'HERT MPE', 'numm': 95}, {'w_type': '类型二', 'product': 'RRU', 'numm': 10}]>
  SELECT w_type, product, COUNT(*) AS numm FROM t_s GROUP BY w_type, product ORDER BY NULL  # sql
  
  ##使用values()继续获取指定字段
  queryset.values('w_type').annotate(numm=Count('*')).values('numm') #<QuerySet [{'numm': 5497}, {'numm': 1}]>
  SELECT COUNT(*) AS numm FROM t_s GROUP BY w_type ORDER BY NULL;
  ```

    **连表查询**

  ```
  
  ```

- **`aggregate()`**  使用聚合函数，对queryset中所有对象进行计算，返回一个字典。

  ```python
  ## Sum，Avg会返回一个Decimal类型结果
  queryset.aggregate(numm=Avg('id')) # numm不指定默认使用‘字段名__聚合函数名’
  # 返回  {'numm': Decimal('15134663')}
  queryset.aggregate(numm=Max('id'))
  # 返回 {'numm': 11111}
  ```

- **`values()`**  返回一个`QuerySet`在用作iterable时返回字典而不是模型实例,每个字典代表一个对象，其中的键对应于模型对象的属性名称

  ```python
  # 使用可选关键字参数，被传递给annotate()
  queryset.values(lower_name=Lower('product'))
  # <QuerySet [{'lower_name': 'hert mpe'}, {'lower_name': 'bbu'}
  ```

- **`values_list()`**  类似于values，它在迭代时返回元组，每个元组代表一个对象，只包含字段值

  - 若只传单个字段，可以传flat 参数。flat=True返回的结果是单个值，而不是一个元组；

    `flat`当有多个字段时传入是错误的；

    queryset.values_list('product',flat=True).get(pk=1)

  - 传递`named=True`以获得结果 [`namedtuple()`](https://docs.python.org/3/library/collections.html#collections.namedtuple)命名元组。

    <QuerySet [Row(id=1, headline='First entry'), ...]>

    使用命名元组可能会使结果更具可读性，但代价是将结果转换为命名元组会降低性能。

  **注意**：`values()`和`values_list()`它们都是针对特定用例的优化：**检索数据子集而无需创建模型实例的开销**。当处理多对多和其他多值关系（例如反向外键的一对多关系）时，这个比喻就会崩溃，因为“一行，一个对象”假设不成立。

- **`select_related`（* fields）** 返回一个外键关联的对象,这可以提高性能，意味着之后用外间关系将不需要查询数据库。 相当于inner join；

  **对于OneToOneField和外键字段(ForeignKey)可使用其来优化查询**。

  **【1】普通查找和`select_related()`查找之间的区别：**

  ```python
  # 普通查找：
  e = Entry.objects.get(id=5)  # 查询数据库
  b = e.blog  # 再查询数据库
  
  #select_related查找：
  e = Entry.objects.select_related('blog').get(id=5) # 查询数据库
  b = e.blog  # 不再需要查询数据库，之前已经缓存
  ```

  `select_related()`与任何查询集一起使用；`filter()`和`select_related()`链接的顺序并不重要。这些查询集是等效的：

  ```
  Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog')
  Entry.objects.select_related('blog').filter(pub_date__gt=timezone.now())
  ```

  **【2】接受可变长参数**

  select_related() 接受可变长参数，每个参数是需要获取的外键（父表的内容）的字段名，以及外键的外键的字段名、外键的外键的外键…。若要选择外键的外键需要使用两个下划线“__”来连接。

```python
# 使用两个join 连查三个表，未指定的外键（hometown）则不会被添加到结果中
zhangs = Person.objects.select_related('liveTown__province').get(id=1) 
# 同时指定两个外键使用（django1.7以后）
zhangs = Person.objects.select_related('hometown__province').select_related('living__province').get(id=1)
```

```
**【3】depth参数**

select_related() 接受depth参数，depth参数可以确定select_related的深度。Django会递归遍历指定深度内	的所有的OneToOneField和ForeignKey.
```

```python
zhangs = Person.objects.select_related(depth = d)
# d=1  相当于 select_related(‘hometown’,'living’)
# d=2  相当于 select_related(‘hometown__province’,'living__province’)

```

```
**【4】**select_related() 也可以不加参数，这样表示要求Django尽可能深的select_related
```

- **`prefetch_related()`**  返回一个`QuerySet`，它将自动为每个指定的查询单批检索相关对象。

  **对于多对多字段(ManyToManyField)和一对多(ForeignKey)字段，使用prefetch_related()来进行优化.**

```python
zhangs = Person.objects.prefetch_related('visitTown').get(id=1)
for city in zhangs.visitTown.all() :  # 也就相当与上面e.blog，多对多关系 所以需要加 .all()
	print(city)  # 上面查询已经缓存，将不再查询数据库
```

**【注意】在使用QuerySet时，一旦在链式操作中改变了数据库请求，之前用prefetch_related缓存的数据将会被忽略掉。导 致Django重新请求数据库，从而造成性能问题。这里提到的改变数据库请求指各种filter()、exclude()等等最终会改变 SQL代码的操作。而all()并不会改变最终的数据库请求，因此不会导致重新请求数据库**



#### F查询

Django使用该`F()`对象生成一个SQL表达式，该表达式描述数据库级别所需的操作。

可以用于同一记录中多个字段间的操作

**`F()` 可以通过以下方式提供性能优势：**

- 获取数据库而不是Python来工作
- 减少某些操作所需的查询数量

```python
# 下面代码将数据库的值拉到内存中
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed += 1
reporter.save()

# 通常使用下面更好
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed = F('stories_filed') + 1
reporter.save()
## 要访问以此方式保存的新值，必须重新加载该对象：
reporter = Reporters.objects.get(pk=reporter.pk)
## 或者使用一下更简洁：
reporter.refresh_from_db()
```

`F()`可以在单个实例上进行上述操作外，还可以在`QuerySets`对象实例上使用`update()`。这将我们上面使用的两个查询- `get()`和 [`save()`](instances.html#django.db.models.Model.save)-简化为一个：

```python
reporter = Reporters.objects.filter(name='Tintin') # 不会操作数据库
reporter.update(stories_filed=F('stories_filed') + 1)
```

还可以[`update()`](querysets.html#django.db.models.query.QuerySet.update)用于增加多个对象的字段值-这可能比将它们全部从数据库中拉入Python，循环遍历，增加每个对象的字段值并将每个对象保存回数据库的速度要快得多

```python
Reporter.objects.all().update(stories_filed=F('stories_filed') + 1)
```

**使用F()避免竞争条件：**

让数据库（而不是Python）更新字段的值可以避免*争用情况*(详情见文档)

上面示例有俩线程执行时，数据库负责更新字段，则过程将更加健壮：它将仅根据执行[`save()`](instances.html#django.db.models.Model.save)or或时`update()`执行时数据库中字段的值来更新字段，而不是根据实例检索时的值来更新字段。

**分配给模型字段的F()对象在保存模型实例后将保持，并将应用于每次 save()**

```python
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed = F('stories_filed') + 1
reporter.save()

reporter.name = 'Tintin Jr.'
reporter.save()
# stories_filed在这种情况下将更新两次。如果是初始1值，则最终值为3。
```

- **Subquery()表达式**

  限制一个子查询到单个列

  ```
  Comment.objects.filter(post__in=Subquery(queryset.values('pk')))
  ```

  将子查询限制为单行

  ```
  subquery = Subquery(newest.values('email')[:1])
  Post.objects.annotate(newest_commenter_email=subquery)
  ```

## 7.自定义管理器

**作用：可以用来新增模型的方法，和重写模型中已有的方法**

- **使用自定义的manager**

增加额外的manager是为模块添加**表级功能**的首选办法.(至于**行级功能**,就是只作用于模型实例对象的函数,则通过自定义模型方法实现【即属性方法】)

```python
# 自定义模型管理器类
class BookManager(models.Manager):
    #自定义模型管理器中的方法
    def title_count(self, keyword):
        return self.filter(title_icountains=keyword).count()

class Book(models.Model):
    title = models.CharField(max_length=100)
    authors = models.ManyToManyField(Author)
    # 命名objects 将覆盖默认模型管理器objects，但继承了默认管理器的所有属性和方法
    objects = BookManager()  
```

- **添加额外的manager**

  ```python
  #首先,定义一个Manager的子类
  class ZsBookManager(models.Manager):
  	# 覆盖了默认管理器的返回所有对象queryset的方法
      def get_queryset(self):
          return super(DahlBookManager, self).get_queryset().filter(author='张三')
  
  # 然后,将它显式地插入到Book模型中
  class Book(models.Model):
      title = models.CharField(max_length=100)
      author = models.CharField(max_length=50)
      ...
      objects = models.Manager()    # 默认Manager
      dahl_objects = ZsBookManager() 
  ```

## 直接操作数据库pymysql

##### 将fetchall()返回结果改为字典形式

fetchall()将结果放在二维数组里面，每一行的结果在元组里面，想返回字典格式，只需要在建立游标的时候加个参数，cursor=pymysql.cursors.DictCursor。这样每行返回的值放在字典里面，然后整体放在一个list里面。

```python
cur = conn.cursor(cursor=pymysql.cursors.DictCursor)
```



# 数据库

### 1.配置多数据库

```python
DATABASES = {
    'default': {
        'NAME': 'app_data',
        'ENGINE': 'django.db.backends.postgresql',
        'USER': 'postgres_user',
        'PASSWORD': 's3krit'
    },
    'users': {
        'NAME': 'user_data',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'priv4te'
    }
}
```

同步数据库：

```
python manage.py migrate  // 默认使用default数据库
python manage.py migrate --database=users

```



### 数据库路由器

数据库路由器是一个提供四个方法的类：

- db_for_read   指定读操作使用的数据库
- db_for_write  指定写操作使用的数据库
- allow_relation  允许使用相同数据库的应用程序之间存在任何关系
- allow_migrate  是否允许在别名为`db`的数据库上运行迁移操作

```python
from django.conf import settings

DATABASE_MAPPING = settings.DATABASE_APPS_MAPPING

class DatabaseAppsRouter(object):
    """
    A router to control all database operations on models for different
    databases.

    In case an app is not set in settings.DATABASE_APPS_MAPPING, the router
    will fallback to the `default` database.

    Settings example:

    DATABASE_APPS_MAPPING = {'app1': 'db1', 'app2': 'db2'}
    """

    def db_for_read(self, model, **hints):
        """"Point all read operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None

    def db_for_write(self, model, **hints):
        """Point all write operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """Allow any relation between apps that use the same database."""
        db_obj1 = DATABASE_MAPPING.get(obj1._meta.app_label)
        db_obj2 = DATABASE_MAPPING.get(obj2._meta.app_label)
        if db_obj1 and db_obj2:
            if db_obj1 == db_obj2:
                return True
            else:
                return False
        return None

    # for Django 1.4 - Django 1.6
    def allow_syncdb(self, db, model):
        """Make sure that apps only appear in the related database."""
        if db in DATABASE_MAPPING.values():
            return DATABASE_MAPPING.get(model._meta.app_label) == db
        elif model._meta.app_label in DATABASE_MAPPING:
            return False
        return None

    # Django 1.7 - Django 1.11
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if db in DATABASE_MAPPING.values():
            return DATABASE_MAPPING.get(app_label) == db
        elif app_label in DATABASE_MAPPING:
            return False
        return None
```

settings配置：

```python
DATABASE_ROUTERS = ['MyProject.DatabaseAppsRouter']
DATABASE_APPS_MAPPING = {
    # example:
    # 'app_name':'database_name',
    'user_app': 'user',
    'other_app': 'other_db',
}
```



# 事务

### django中会使用默认事务行为

1.默认事务：默认情况下，在Django中事务是自动提交的,

- 每条对数据库操作的语句会被自动提交（无论使用orm还是原生SQL），**使用pymysql必须主动提交**

2.使用事务装饰器`@transaction.atomic()` 装饰函数或者视图：

- 接收一个using参数，指定数据库，默认使用django默认数据库default连接
- 事务内的每条orm语句或使用django的connection执行的sql语句将不再自动提交，函数返回时自动提交事务内所有操作
- 可以使用 `transaction.savepoint()` 设置保存点，提交部分事务。

3.with 语句用法（控制局部范围事务）：

```python
 def func()
 	# 这部分代码不在事务中，会被 Django 自动提交
 	......
    try:
        with transaction.atomic():  # using='default' 指定数据库
            # 这部分代码会在事务中执行
            # raise 抛异常 将自动回滚，不需要任何操作
    except: 
        pass
 # transaction不需要在代码中手动commit和rollback的。因为只有当一个transaction正常退出的时候，才会对数据库层面进行操作。
```

- 为了保证原子性，**atomic**还禁止了一些API。像试图提交、回滚事务，以及改变数据库连接的自动提交状态这些操作，在atomic代码块中都是不予许的，否则就会抛出异常。

- **atomic**使用一个参数来指定数据库的名字。如果该参数没有设置值，Django就会使用系统默认的数据库。

- Django的事务管理代码：

  - 进入最外层atomic代码块时开启一个事务；
  - 进入内部atomic代码块时创建保存点；
  - 退出内部atomic时释放或回滚事务；
  - 退出最外层atomic代码块时提交或者回滚事务；

- **on_commit()**

  - 有时候你想在事务成功提交后执行一些与当前数据库事务相关的操作。比如说Celery任务，邮件通知，或者缓存失效。

  - 提供了**on_commit()**方法来注册一些只有在事务成功提交后才执行的回调函数。

    ```
    on_commit(lambda: expand_abbreviations.delay(article.pk))
    
    ```

  - 注意： `on_commit` 函数只在 Django 1.9 以上版本才可用，如果你使用以前的版本，那么可以使用 `django-transaction-hooks` 库添加相关支持。



### 在HTTP请求上加事务

1.使用中件间TransactionMiddleware来处理请求和响应中的事务。它的工作原理是这样的：当一个请求到来时，Django开始一个事务，如果响应没有出错，Django提交这期间所有的事务，如果view中的函数抛出异常，那么Django会回滚这之间的事务。

2.顺序很重要，TransactionMiddleware中间件会将置于其后的中间件都包含在事务的范围之中（用于缓存的中间件除外，他们不受影响，例如CacheMiddleware，UpdateCacheMiddleware和FetchFromCacheMiddleware，另外，TransactionMiddleware只会影响[DATABASES](http://blog.sina.com.cn/s/blog_3fe961ae010167ah.html#std:setting-DATABASES)设置中的默认的数据库



# 信号

### 1. Django内置的信号量

```
Model signals
    pre_init                    # django的modal执行其构造方法前，自动触发
    post_init                   # django的modal执行其构造方法后，自动触发
    pre_save                    # django的modal对象保存前，自动触发
    post_save                   # django的modal对象保存后，自动触发
    pre_delete                  # django的modal对象删除前，自动触发
    post_delete                 # django的modal对象删除后，自动触发
    m2m_changed                 # django的modal中使用m2m字段操作第三张表（add,remove,clear）前后，自动触发
    class_prepared              # 程序启动时，检测已注册的app中modal类，对于每一个类，自动触发
Management signals
    pre_migrate                 # 执行migrate命令前，自动触发
    post_migrate                # 执行migrate命令后，自动触发
Request/response signals
    request_started             # 请求到来前，自动触发
    request_finished            # 请求结束后，自动触发
    got_request_exception       # 请求异常后，自动触发
Test signals
    setting_changed             # 使用test测试修改配置文件时，自动触发
    template_rendered           # 使用test测试渲染模板时，自动触发
Database Wrappers
    connection_created          # 创建数据库连接时，自动触发

```



### 2. 使用

【注意】: 需在apps.py中导入信号量，注册。

【1】使用方式一：

**需将以下代码放入init文件中，方可触发：**

```python
from django.db.models.signals import post_delete, post_save

def callback(sender, **kwargs): 
	# sender：接收 信号发送者（send()中指定的sender）， **kwargs 接受其他回调参数
    pass
    
post_save.connect(callback，sender=‘aa’)   # 该代码应在apps.py中 或项目init文件中 sender接收特定发送者的信号
```

post_save.connect(callback)：将信号post_save与函数callback绑定在一起，一旦检测到model的save()方法执行完，那么就执行callback函数。kwargs中的created为True,表示是新增一条记录。

**信号量回调参数：**

```
{'signal': <django.db.models.signals.ModelSignal object at 0x0000025C9B99AD30>,
'instance': <Book: Book object>, 'created': True, 'update_fields': None, 'raw': False, 'using': 'default'}

sender            模型类 
instance          保存的实际实例 
created           布尔值; True如果创建了新记录（True表示数据创建） 
raw               布尔值; True如果模型完全按照提供的方式保存。不应该查询/修改数据库中的其他记录，因为数据库可能尚未处于一致状态 
using             正在使用的数据库别名 
update_fields     要传递给更新的字段集model.save（），或者None 如果update_fields未传给它save() 

```

【2】使用方式二：

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Book) # 接收仅由特定发送者发送的信号
def my_callback(sender, **kwargs):
    print("Request finished!")
    
# apps.py 加入以下代码注册信号
def ready(self):
    import app目录名.signals  # 导入定义的signals文件
```



### 3.防止重复信号

在某些情况下，将接收器连接到信号的代码可能会运行多次。这可能会导致您的接收器功能被多次注册，因此对于单个信号事件将被多次调用。

如果此行为有问题（例如在保存模型时使用信号发送电子邮件），请传递一个唯一标识符作为`dispatch_uid`参数，以识别您的接收方功能。该标识符通常是一个字符串，尽管任何可哈希对象都足够。最终结果是，对于每个唯一`dispatch_uid`值，您的接收器功能将仅与信号绑定一次：

```
from django.core.signals import request_finished

request_finished.connect(my_callback, dispatch_uid="my_unique_identifier")

```

### 4.自定义信号使用

##### 【1】定义信号

```
from django.dispatch import Signal

test_signal = Signal(providing_args=["name", "age"])  # 声明一个test_signal的信号，提供给接收器name跟age两个参数（可自定义参数）

```

##### 【2】注册信号

```
def my_callback(sender, **kwargs):
    print(sender)
    print("信号已接收")

test_signal.connect（my_callback）  # 注册信号，指定接收器为my_callback

```

##### 【3】触发信号

```python
from xxx import test_signal

# 触发信号，发送name，age参数信息
test_signal.send(sender='test', name='zzq', age='18')  #test 为发送者（当有多个地方发送此定义的信号test_signal时，接收端可指定接收哪个发送者的信号）； 
```

当然这样在选择发送信号的方式有两种一种使用Signal.send，还有一种是Signal.send_robut。

send()与send_robust()处理接收器功能引起的异常的方式不同。

send()并不能捕获由接收器提出的任何异常; 它只是允许错误传播。因此，在面对错误时，不是所有接收器都可以被通知信号。

send_robust()捕获从Python Exception类派生的所有错误，并确保所有接收器都收到信号通知。如果发生错误，则会在引发错误的接收器的元组对中返回错误实例。



# 缓存

#### 1. 模型属性方法缓存

`cached_property`:

使用[`cached_property`](../ref/utils.html#django.utils.functional.cached_property)装饰器可以保存属性返回的值。下次在该实例上调用该函数时，它将返回保存的值，而不是重新计算它。请注意，这仅适用于将其`self`作为唯一参数的方法，并将方法更改为属性

#### 2. 缓存单个视图结果

```python
@cache_page(60 * 15, cache="special_cache") # cache指定缓存后端
def my_view(request):
	。。。
```

**或在URLconf中指定每个视图的缓存：**

```python
urlpatterns = [
    path('foo/<int:code>/', cache_page(60 * 15)(my_view)),
]
```

**注意：cache_page装饰器会触发浏览器缓存（浏览器缓存时间与设置的后端缓存时间一致），即使删除后端缓存，浏览器也会优先读取浏览器缓存的数据，不会向后端发送请求**

#### 3. 整个站点缓存

#### 4. 低级别的缓存API

`django.core.cache.cache`:

不想缓存整个结果（因为某些数据经常更改），但仍想缓存很少更改的结果：

**基本接口是**：set(key, value, timeout)  和  get(key)

```python
# 基本存取接口
cache.set('my_key', 'hello, world!', 30)
cache.get('my_key') # key不存在，也可已设置返回default值

# 若要只在尚不存在的情况下添加key，使用add(),它采用与set()相同的参数，如果指定的键已经存在，它将不尝试更新缓存，若需要知道是否缓存了该key，可以检查其返回值：已存返回True，反之 False
cache.add('my_key','value',100)

# 键不存在则设置该键的缓存，而不是简单的返回
cache.get_or_set('my_key','value',100) 

# get_many()接口只命中一次缓存。返回一个字典，包含传入的所有键，这些键要在缓存中存在（且尚未过期）
cache.get_many(['a', 'b', 'c'])

# 设置多个值，请使用set_many()传递键值对字典
cache.set_many({'a': 1, 'b': 2, 'c': 3})

# 使用delete()显式删除密钥。这是清除特定对象缓存的简单方法
cache.delete('a')
# 一次清除一堆键，delete_many()可以列出要清除的键列表
cache.delete_many(['a', 'b', 'c'])
# 若要删除缓存中的所有键，使用 cache.clear()。对此要小心；clear()将从缓存中删除所有内容，而不仅仅是您的应用程序设置的键
cache.clear()

# 您也可以分别使用incr()或decr()方法递增或递减已存在的键，默认现有的高速缓存值将递增或递减1。可以指定递增或递减的值
cache.incr('num', 10)
cache.decr('num', 10)

# cache.close() 如果由缓存后端实现，则可以关闭与缓存的连接
```

### 清除特定视图的缓存

方法：找到指定视图缓存的key值，通过delete接口去删除指定键

1.根据生成缓存键的源码，自己构造缓存键

```python
def get_cache_key_(request,path,key_prefix=None, method='GET', cache=None):
    if key_prefix is None:
        key_prefix = settings.CACHE_MIDDLEWARE_KEY_PREFIX

    url = hashlib.md5(force_bytes(iri_to_uri(path)))
    cache_key = 'views.decorators.cache.cache_header.%s.%s' % (
        key_prefix, url.hexdigest())
    cache_key_header  = _i18n_cache_key_suffix(request, cache_key)
    if cache is None:
        cache = caches[settings.CACHE_MIDDLEWARE_ALIAS]
    headerlist = cache.get(cache_key_header)
    if headerlist is not None:
        ctx = hashlib.md5()
        for header in headerlist:
            value = request.META.get(header)
            if value is not None:
                ctx.update(force_bytes(value))
        # url = hashlib.md5(force_bytes(iri_to_uri(path)))
        cache_key = 'views.decorators.cache.cache_page.%s.%s.%s.%s' % (
            key_prefix, method, url.hexdigest(), ctx.hexdigest())
        return _i18n_cache_key_suffix(request, cache_key), cache_key_header
    else:
        return None, None
    
def expire_page(path):
    request = HttpRequest()
    request.path = path
    request.META['HTTP_HOST'] = '127.0.0.1:8000'
    url = "http://127.0.0.1:8000/commons/distinct_cache?tableName=t_devicecode_basicline&colNames=manufacture,asl,country,parts_type_1d5,parts_type_2d0,small_model"
    key,key_header = get_cache_key_(request,url,cache=cache)
    if key and cache.has_key(key):
        cache.delete(key)
        cache.delete(key_header)
```



```python
def expire_view_cache(path, servername, serverport, key_prefix=None):
    '''
    刷新视图缓存
    :param path:url路径
    :param servername:host
    :param serverport:端口
    :param key_prefix:前缀
    :return:是否成功
    '''
    from django.http import HttpRequest
    from django.utils.cache import get_cache_key

    request = HttpRequest()
    request.META = {'SERVER_NAME': servername, 'SERVER_PORT': serverport}
    request.path = path

    key = get_cache_key(request, key_prefix=key_prefix, cache=cache)
    if key:
        logger.info('expire_view_cache:get key:{path}'.format(path=path))
        if cache.get(key):
            cache.delete(key)
        return True
    return False
```



### Cache-Control头部

`cache_control` 可以控制浏览器缓存；使用cache_control装饰器来设定Cache-Control头部。

**设置对特定的用户提供缓存服务**：

```python
from django.views.decorators.cache import cache_control
 
@cache_control(private=True)
def my_view(request):
    ...
```

**设置时间**（一旦设置max_age,浏览器将进行缓存，从浏览器缓存取数据 ，直到缓存过期；否则不会访问后端）：

```python
from django.views.decorators.cache import cache_control
 
@cache_control(max_age=3600)
def my_view(request):
    ...
```

等等，可用的Cache-Control指令([IANA registry](http://www.iana.org/assignments/http-cache-directives/http-cache-directives.xhtml))都可使用。

**列举cache_control接收到的参数：**

- public=True
- private=True
- no_cache=True
- no_transform=True
- must_revalidate=True
- proxy_revalidate=True
- max_age=num_seconds
- s_maxage=num_seconds

