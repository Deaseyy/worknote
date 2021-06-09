# [Django QuerySet API文档](https://www.cnblogs.com/linxiyue/p/4040262.html)



### **在查询时发生了什么(When QuerySets are evaluated)**

QuerySet 可以被构造，过滤，切片，做为参数传递，这些行为都不会对数据库进行操作。只要你查询的时候才真正的操作数据库。

下面的 QuerySet 行为会导致执行查询的操作：

**循环(Iteration)**：QuerySet 是可迭代的，在你遍历对象时就会执行数据库操作。例如，打印出所有博文的大标题：

```
`for` `e ``in` `Entry.objects.``all``():``    ``print``(e.headline)`
```

**切片(Slicing)**： QuerySet 是可以用 Python 的数组切片语法完成切片。一般来说对一个未查询的 QuerySet 切片就返回另一个未查询的 QuerySet (新 QuerySet 不会被执行)。不过如果你在切片时使用了 "step" 参数，Django 仍会执行数据库操作并且返回列表对象。对一个查询过的QuerySet执行切片也会返回列表对象。

**序列化／缓存化(Pickling/Caching)**： 详情请查看 pickling QuerySets。 这一节所强调的一点是查询结果是从数据库中读取的。

**repr()**. 调用 QuerySet 的 repr() 方法时，查询就会被运行。这对于 Python 命令行来说非常方便，你可以使用 API 立即看到查询结果。

**len()**. 调用 QuerySet 的 len() 方法，查询就会被运行。这正如你所料，会返回查询结果列表的长度。

注意：如果你想得到集合中记录的数量，就不要使用 QuerySet 的 len() 方法。因为直接在数据库层面使用 SQL 的 SELECT COUNT(*) 会更加高效，Django 提供了 count() 方法就是这个原因。详情参阅下面的 count() 方法。

**list()**. 对 QuerySet 应用 list() 方法，就会运行查询。例如：

```
`entry_list ``=` `list``(Entry.objects.``all``())`
```

要注意地是：使用这个方法会占用大量内存，因为 Django 将列表内容都载入到内存中。做为对比，遍历 QuerySet 是从数据库读取数据，仅在使用某个对象时才将其载入到内容中。

**Pickling QuerySets**

如果你要 序列化(pickle) 一个 QuerySet，Django 首先就会将查询对象载入到内存中以完成序列化，这样你就可以第一时间使用对象(直接从数据库中读取数据需要一定时间，这正是缓存所想避免的)。而序列化是缓存化的先行工作，所以在缓存查询时，首先就会进行序列化工作。这意味着当你反序列化 QuerySet 时，第一时间就会从内存中获得查询的结果，而不是从数据库中查找。

如果你只是想序列化部分必要的信息以便晚些时候可以从数据库中重建 Queryset ，那只序列化 QuerySet 的 query 属性即可。接下来你就可以使用下面的代码重建原来的 QuerySet （这当中没有数据库读取）：

```
`>>> ``import` `pickle``>>> query ``=` `pickle.loads(s)     ``# Assuming 's' is the pickled string.``>>> qs ``=` `MyModel.objects.``all``()``>>> qs.query ``=` `query            ``# Restore the original 'query'.`
```

query 属性是一个不透明的对象。这就意味着它的内部结构并不是公开的。即便如此，对于本节提到的序列化和反序列化来说，它仍是安全和被完全支持的。

### 查询API(QuerySet API)

**filter(\**kwargs)**

返回一个新的 QuerySet ，它包含了与所给的筛选条件相匹配的对象。

这些筛选条件(**kwargs)在下面的字段筛选(Field lookups) 中有详细介绍。多个条件之间在 SQL 语句中是 AND 关系。

**exclude(\**kwargs)**

返回一个新的 QuerySet，它包含那些与所给筛选条件不匹配的对象。

这些筛选条件(**kwargs)也在下面的 字段筛选(Field lookups) 中有详细描述。多个条件之间在 SQL 语句中也是 AND 关系，但是整体又是一个 NOT() 关系。

下面的例子剔除了出版日期 pub_date 晚于 2005-1-3 并且大标题 headline 是 "Hello" 的所有博文(entry)：

```
`Entry.objects.exclude(pub_date__gt``=``datetime.date(``2005``, ``1``, ``3``), headline``=``'Hello'``)`
```

在 SQL 语句中，这等价于：

```
`SELECT ...``WHERE NOT (pub_date > ``'2005-1-3'` `AND headline ``=` `'Hello'``)`
```

下面的例子剔除出版日期 pub_date 晚于 2005-1-3 或者大标题是 "Hello" 的所有博文：

```
`Entry.objects.exclude(pub_date__gt``=``datetime.date(``2005``, ``1``, ``3``)).exclude(headline``=``'Hello'``)`
```

在 SQL 语句中，这等价于：

```
`SELECT ...``WHERE NOT pub_date > ``'2005-1-3'``OR NOT headline ``=` `'Hello'`
```

要注意第二个例子是有很多限制的。

**annotate(\*args, **kwargs)**

我们可以为 QuerySet 中的每个对象添加注解。可以通过计算查询结果中每个对象所关联的对象集合，从而得出总计值(也可以是平均值或总和，等等)，做为 QuerySet 中对象的注解。annotate() 中的每个参数都会被做为注解添加到 QuerySet 中返回的对象。

Django 提供的注解函式在下面的 (注解函式Aggregation Functions) 有详细介绍。

注解会使用关键字参数来做为注解的别名。其中任何参数都会生成一个别名，它与注解函式的名称和被注解的 model 相关。

例如，你正在操作一个博客列表，你想知道一个博客究竟有多少篇博文：

```
`>>> ``from` `django.db.models ``import` `Count``>>> q ``=` `Blog.objects.annotate(Count(``'entry'``))``# The name of the first blog``>>> q[``0``].name``'Blogasaurus'``# The number of entries on the first blog``>>> q[``0``].entry__count``42`
```

Blog model 类本身并没有定义 entry__count 属性，但可以使用注解函式的关系字参数，从而改变注解的命名：

```
`>>> q ``=` `Blog.objects.annotate(number_of_entries``=``Count(``'entry'``))``# The number of entries on the first blog, using the name provided``>>> q[``0``].number_of_entries``42`
```

**order_by(\*fields)**

默认情况下， QuerySet 返回的查询结果是根据 model 类的 Meta 设置所提供的 ordering 项中定义的排序元组来进行对象排序的。你可以使用 order_by 方法覆盖之前 QuerySet 中的排序设置。

例如：

```
`Entry.objects.``filter``(pub_date__year``=``2005``).order_by(``'-pub_date'``, ``'headline'``)`
```

返回结果就会先按照 pub_date 进行升序排序，再按照 headline 进行降序排序。 "-pub_date" 前面的负号"-"表示降序排序。默认是采用升序排序。要随机排序，就使用 "?"，例如：

```
`Entry.objects.order_by(``'?'``)`
```

注意：order_by('?') 可能会非常缓慢和消耗过多资源，这取决于你所使用的数据库。

要根据其他 model 字段排序，所用语法和跨关系查询的语法相同。就是说，用两个连续的下划线(__)连接关联 model 和 要排序的字段名称, 而且可以一直延伸。例如：

```
`Entry.objects.order_by(``'blog__name'``, ``'headline'``)`
```

如果你想对关联字段排序，在没有指定 Meta.ordering 的情况下，Django 会采用默认排序设置，就是按照关联 model 的主键进行排序。例如：

```
`Entry.objects.order_by(``'blog'``)`
```

等价于：

```
`Entry.objects.order_by(``'blog__id'``)`
```

这是因为 Blog model 没有声明排序项的原故。

**Django 1.7新添加：**

```
`# No Join``Entry.objects.order_by(``'blog_id'``)     ``#可以避免JOIN的代价` `# Join``Entry.objects.order_by(``'blog__id'``)`
```

如果你使用了 distinct() 方法，那么在对关联字段排序时要格外谨慎。

在 Django 当中是可以按照多值字段（例如 ManyToMany 字段）进行排序的。不过，这个特性虽然先进，但是并不实用。除非是你已经很清楚过滤结果或可用数据中的每个对象，都只有一个相关联的对象时（就是相当于只是一对一关系时），排序才会符合你预期的结果，所以对多值字段排序时要格外注意。

如果你不想对任何字段排序，也不想使用 model 中原有的排序设置，那么可以调用无参数的 order_by() 方法。

对于排序项是否应该大小写敏感，Django 并没有提供设置方法，这完全取决于后端的数据库对排序大小写如何处理。

你可以令某个查询结果是可排序的，也可以是不可排序的，这取决于 QuerySet.ordered 属性。如果它的值是 True ，那么 QuerySet 就是可排序的。

**reverse()**

使用 reverse() 方法会对查询结果进行反向排序。调用两次 reverse() 方法相当于排序没发生改过。

要得到查询结果中最后五个对象，可以这样写：

```
`my_queryset.reverse()[:``5``]`
```

要注意这种方式与 Python 语法中的从尾部切片是完全不一样的。在上面的例子中，是先得到最后一个元素，然后是倒数第二个，依次处理。但是如果我们有一个 Python 队列，使用 seq[-5:]时，却是先得到倒数第五个元素。Django 之所以采用 reverse 来获取倒数的记录，而不支持切片的方法，原因就是后者在 SQL 中难以做好。

还有一点要注意，就是 reverse() 方法应该只作用于已定义了排序项 QuerySet (例如，在查询时使用了order_by()方法，或是在 model 类当中直接定义了排序项). 如果并没有明确定义排序项，那么调用 QuerySet, calling reverse() 就没什么实际意义（因为在调用 reverse() 之前，数据没有定义排序，所以在这之后也不会进行排序。)

**distinct()**

返回一个新的 QuerySet ，它会在执行 SQL 查询时使用 SELECT DISTINCT。这意味着返回结果中的重复记录将被剔除。

默认情况下， QuerySet 并会剔除重复的记录。在实际当中，这不是什么问题，因为象 Blog.objects.all() 这样的查询并不会产生重复的记录。但是，如果你使用 QuerySet 做多表查询时，就很可能会产生重复记录。这时，就可以使用 distinct() 方法。

**Note**

在 order_by(*fields) 中出现的字段也会包含在 SQL SELECT 列中。如果和 distinct() 同时使用，有时返回的结果却与预想的不同。这是因为：如果你对跨关系的关联字段进行排序，这些字段就会被添加到被选取的列中，这就可能产生重复数据（比如，其他的列数据都相同，只是关联字段的值不同）。但由于 order_by 中的关联字段并不会出现在返回结果中（他们仅仅是用来实现order），所以有时返回的数据看上去就象是并没有进行过 distinct 处理一样。

同样的原因，如果你用 values() 方法获得被选取的列，就会发现包含在 order_by() (或是 model 类的 Meta 中设置的排序项)中的字段也包含在里面，就会对返回的结果产生影响。

这章节强调的就是在你使用 distinct() 时，要谨慎对待关联字段排序。同样的，在同时使用 distinct() 和 values() 时，如果排序字段并没有出现在 values() 返回的结果中，那么也要引起注意。

**values(\*fields)**

返回一个 ValuesQuerySet ----一个特殊的 QuerySet ，运行后得到的并不是一系列 model 的实例化对象，而是一个可迭代的字典序列。

每个字典都表示一个对象，而键名就是 model 对象的属性名称。

下面的例子就对 values() 得到的字典与传统的 model 对象进行了对比：

```
`# This list contains a Blog object.``>>> Blog.objects.``filter``(name__startswith``=``'Beatles'``)``[<Blog: Beatles Blog>]` `# This list contains a dictionary.``>>> Blog.objects.``filter``(name__startswith``=``'Beatles'``).values()``[{``'id'``: ``1``, ``'name'``: ``'Beatles Blog'``, ``'tagline'``: ``'All the latest Beatles news.'``}]`
```

values() 可以接收可选的位置参数，*fields，就是字段的名称，用来限制 SELECT 选取的数据。如果你指定了字段参数，每个字典就会以 Key-Value 的形式保存你所指定的字段信息；如果没有指定，每个字典就会包含当前数据表当中的所有字段信息。

例如：

```
`>>> Blog.objects.values()``[{``'id'``: ``1``, ``'name'``: ``'Beatles Blog'``, ``'tagline'``: ``'All the latest Beatles news.'``}],``>>> Blog.objects.values(``'id'``, ``'name'``)``[{``'id'``: ``1``, ``'name'``: ``'Beatles Blog'``}]`
```

下面这些细节值得注意：

```
如果你有一个名为 foo 的ForeignKey 字段，默认情况下调用 values() 返回的字典中包含键名为 foo_id 的字典项，因为它是一个隐含的 model 字段，用来保存关联对象的主键值( foo 属性用来联系相关联的 model )。当你使用 values() 并传递字段名称时， 传递foo 或 foo_id 都会得到相同的结果 (字典中的键名会自动换成你传递的字段名称)。
```

例如：

```
`>>> Entry.objects.values()``[{``'blog_id'``: ``1``, ``'headline'``: u``'First Entry'``, ...}, ...]` `>>> Entry.objects.values(``'blog'``)``[{``'blog'``: ``1``}, ...]` `>>> Entry.objects.values(``'blog_id'``)``[{``'blog_id'``: ``1``}, ...]`
```

```
在 values() 和 distinct() 同时使用时，要注意排序项会影响返回的结果，详情请查看上面 distinct() 一节。

在values()之后使用defer()和only()是无用的。
```

ValuesQuerySet 是非常有用的。利用它，你就可以只获得你所需的那部分数据，而不必同时读取其他的无用数据。

最后，要提醒的是，ValuesQuerySet 是 QuerySet 的一个子类，所以它拥有 QuerySet 所有的方法。你可以对它调用 filter() 或是 order_by() 以及其他方法。所以下面俩种写法是等价的：

```
`Blog.objects.values().order_by(``'id'``)``Blog.objects.order_by(``'id'``).values()`
```

Django 的编写者们更喜欢第二种写法，就是先写影响 SQL 的方法，再写影响输出的方法（比如例中先写 order，再写values ），但这些都无关紧要，完全视你个人喜好而定。

也可以指向一对一、多对多、外键关系对象的域：

```
`Blog.objects.values(``'name'``, ``'entry__headline'``)``[{``'name'``: ``'My blog'``, ``'entry__headline'``: ``'An entry'``},``     ``{``'name'``: ``'My blog'``, ``'entry__headline'``: ``'Another entry'``}, ...]`
```

当指向多对多关系时，因为关联对象可能有很多，所以同一个对象会根据不同的多对多关系返回多次。

**values_list(\*fields)**

它与 values() 非常相似，只不过后者返回的结果是字典序列，而 values() 返回的结果是元组序列。每个元组都包含传递给 values_list() 的字段名称和内容。比如第一项就对应着第一个字段，例如：

```
`>>> Entry.objects.values_list(``'id'``, ``'headline'``)``[(``1``, u``'First entry'``), ...]`
```

如果你传递了一个字段做为参数，那么你可以使用 flat 参数。如果它的值是 True，就意味着返回结果都是单独的值，而不是元组。下面的例子会讲得更清楚：

```
`>>> Entry.objects.values_list(``'id'``).order_by(``'id'``)``[(``1``,), (``2``,), (``3``,), ...]` `>>> Entry.objects.values_list(``'id'``, flat``=``True``).order_by(``'id'``)``[``1``, ``2``, ``3``, ...]`
```

如果传递的字段不止一个，使用 flat 就会导致错误。

如果你没给 values_list() 传递参数，它就会按照字段在 model 类中定义的顺序返回所有的字段。

注意这个方法返回的是 ValuesListQuerySet对象，和列表相似但并不是列表，需要列表操作时需list()转为列表。

**dates(field, kind, order='ASC')**

返回一个 DateQuerySet ，就是提取 QuerySet 查询中所包含的日期，将其组成一个新的 datetime.date 对象的列表。

field 是你的 model 中的 DateField 字段名称。

kind 是 "year", "month" 或 "day" 之一。 每个 datetime.date对象都会根据所给的 type 进行截减。

"year" 返回所有时间值中非重复的年分列表。
"month" 返回所有时间值中非重复的年／月列表。
"day" 返回所有时间值中非重复的年／月／日列表。
order, 默认是 'ASC'，只有两个取值 'ASC' 或 'DESC'。它决定结果如何排序。

例子：

```
`>>> Entry.objects.dates(``'pub_date'``, ``'year'``)``[datetime.date(``2005``, ``1``, ``1``)]``>>> Entry.objects.dates(``'pub_date'``, ``'month'``)``[datetime.date(``2005``, ``2``, ``1``), datetime.date(``2005``, ``3``, ``1``)]``>>> Entry.objects.dates(``'pub_date'``, ``'day'``)``[datetime.date(``2005``, ``2``, ``20``), datetime.date(``2005``, ``3``, ``20``)]``>>> Entry.objects.dates(``'pub_date'``, ``'day'``, order``=``'DESC'``)``[datetime.date(``2005``, ``3``, ``20``), datetime.date(``2005``, ``2``, ``20``)]``>>> Entry.objects.``filter``(headline__contains``=``'Lennon'``).dates(``'pub_date'``, ``'day'``)``[datetime.date(``2005``, ``3``, ``20``)]`
```

**datetimes(field, kind, order='ASC')**

返回一个 DateTimeQuerySet ，就是提取 QuerySet 查询中所包含的日期，将其组成一个新的 datetime.datetime 对象的列表。

**none()**

返回一个 EmptyQuerySet -- 它是一个运行时只返回空列表的 QuerySet。它经常用在这种场合：你要返回一个空列表，但是调用者却需要接收一个 QuerySet 对象。（这时，就可以用它代替空列表）

例如：

```
`>>> Entry.objects.none()``[]``>>> ``from` `django.db.models.query ``import` `EmptyQuerySet``>>> ``isinstance``(Entry.objects.none(), EmptyQuerySet)``True`
```

**all()**

返回当前 QuerySet (或者是传递的 QuerySet 子类)的一分拷贝。 这在某些场合是很用的，比如，你想对一个 model manager 或是一个 QuerySet 的查询结果做进一步的过滤。你就可以调用 all() 获得一分拷贝以继续操作，从而保证原 QuerySet 的安全。

当一个QuerySet查询后，它会缓存查询结果。如果数据库发生了改变，就可以调用all()来更新查询过的QuerySet。

**select_related()**

返回一个 QuerySet ，它会在执行查询时自动跟踪外键关系，从而选取所关联的对象数据。它是一个增效器，虽然会导致较大的数据查询（有时会非常大），但是接下来再使用外键关系获得关联对象时，就会不再次读取数据库了。

下面的例子展示在获得关联对象时，使用 select_related() 和不使用的区别，首先是不使用的例子：

```
`# Hits the database.``e ``=` `Entry.objects.get(``id``=``5``)` `# Hits the database again to get the related Blog object.``b ``=` `e.blog`
```

接下来是使用 select_related 的例子：

```
`# Hits the database.``e ``=` `Entry.objects.select_related().get(``id``=``5``)` `# Doesn't hit the database, because e.blog has been prepopulated``# in the previous query.``b ``=` `e.blog`
```

select_related() 会尽可能地深入遍历外键连接。例如：

```
`from` `django.db ``import` `models` `class` `City(models.Model):``    ``# ...``    ``pass` `class` `Person(models.Model):``    ``# ...``    ``hometown ``=` `models.ForeignKey(City)` `class` `Book(models.Model):``    ``# ...``    ``author ``=` `models.ForeignKey(Person)`
```

 接下来调用 Book.objects.select_related().get(id=4) 将缓存关联的 Person 和 City：

```
`b ``=` `Book.objects.select_related(``'person__city'``).get(``id``=``4``)``p ``=` `b.author         ``# Doesn't hit the database.``c ``=` `p.hometown       ``# Doesn't hit the database.` `b ``=` `Book.objects.get(``id``=``4``) ``# No select_related() in this example.``p ``=` `b.author         ``# Hits the database.``c ``=` `p.hometown       ``# Hits the database.`

```

**prefetch_related()**

对于多对多字段（ManyToManyField）和一对多字段，可以使用prefetch_related()来进行优化。或许你会说，没有一个叫OneToManyField的东西啊。实际上 ，ForeignKey就是一个多对一的字段，而被ForeignKey关联的字段就是一对多字段了。

prefetch_related()和select_related()的设计目的很相似，都是为了减少SQL查询的数量，但是实现的方式不一样。后者是通过JOIN语句，在SQL查询内解决问题。但是对于多对多关系，使用SQL语句解决就显得有些不太明智，因为JOIN得到的表将会很长，会导致SQL语句运行时间的增加和内存占用的增加。若有n个对象，每个对象的多对多字段对应Mi条，就会生成Σ(n)Mi 行的结果表。

prefetch_related()的解决方法是，分别查询每个表，然后用Python处理他们之间的关系。

 例如：

```
`from` `django.db ``import` `models` `class` `Topping(models.Model):``    ``name ``=` `models.CharField(max_length``=``30``)` `class` `Pizza(models.Model):``    ``name ``=` `models.CharField(max_length``=``50``)``    ``toppings ``=` `models.ManyToManyField(Topping)` `    ``def` `__str__(``self``):              ``# __unicode__ on Python 2``        ``return` `"%s (%s)"` `%` `(``self``.name, ``", "``.join([topping.name``                                                  ``for` `topping ``in` `self``.toppings.``all``()]))`

```

运行

```
`>>> Pizza.objects.``all``().prefetch_related(``'toppings'``)`

```

可以联合查询：

```
`class` `Restaurant(models.Model):``    ``pizzas ``=` `models.ManyToMany(Pizza, related_name``=``'restaurants'``)``    ``best_pizza ``=` `models.ForeignKey(Pizza, related_name``=``'championed_by'``)`

```

下面的例子都可以

```
`>>> Restaurant.objects.prefetch_related(``'pizzas__toppings'``)``>>> Restaurant.objects.prefetch_related(``'best_pizza__toppings'``)``>>> Restaurant.objects.select_related(``'best_pizza'``).prefetch_related(``'best_pizza__toppings'``)　　`

```

**extra**(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)

有些情况下，Django 的查询语法难以简练地表达复杂的 WHERE 子句。对于这种情况，Django 提供了 extra() QuerySet 修改机制，它能在QuerySet 生成的 SQL 从句中注入新子句。

由于产品差异的原因，这些自定义的查询难以保障在不同的数据库之间兼容(因为你手写 SQL 代码的原因)，而且违背了 DRY 原则，所以如非必要，还是尽量避免写 extra。

在 extra 可以指定一个或多个 params 参数，如 select，where 或 tables。所有参数都是可选的，但你至少要使用一个。

   **select**
select 参数可以让你在 SELECT 从句中添加其他字段信息。它应该是一个字典，存放着属性名到 SQL 从句的映射。

例如：

```
`Entry.objects.extra(select``=``{``'is_recent'``: ``"pub_date > '2006-01-01'"``})`

```

结果中每个 Entry 对象都有一个额外的 is_recent 属性，它是一个布尔值，表示 pub_date 是否晚于2006年1月1号。

Django 会直接在 SELECT 中加入对应的 SQL 片断，所以转换后的 SQL 如下：

```
`SELECT blog_entry.``*``, (pub_date > ``'2006-01-01'``)``FROM blog_entry;`

```

下面这个例子更复杂一些；它会在每个 Blog 对象中添加一个 entry_count 属性，它会运行一个子查询，得到相关联的 Entry 对象的数量：

```
`Blog.objects.extra(``select``=``{``'entry_count'``: ``'SELECT COUNT(*) FROM blog_entry WHERE blog_entry.blog_id = blog_blog.id'``},``)`

```

(在上面这个特例中，我们要了解这个事实，就是 blog_blog 表已经存在于 FROM 从句中。)

翻译成 SQL 如下：

```
`SELECT blog_blog.``*``, (SELECT COUNT(``*``) FROM blog_entry WHERE blog_entry.blog_id ``=` `blog_blog.``id``) AS entry_count``FROM blog_blog;`

```

要注意的是，大多数数据库需要在子句两端添加括号，而在 Django 的 select 从句中却无须这样。同样要引起注意的是，在某些数据库中，比如某些 MySQL 版本，是不支持子查询的。

某些时候，你可能想给 extra(select=...) 中的 SQL 语句传递参数，这时就可以使用 select_params 参数。因为 select_params 是一个队列，而 select 属性是一个字典，所以两者在匹配时应正确地一一对应。在这种情况下中，你应该使用 django.utils.datastructures.SortedDict 匹配 select 的值，而不是使用一般的 Python 队列。

例如：

```
`Blog.objects.extra(``select``=``SortedDict([(``'a'``, ``'%s'``), (``'b'``, ``'%s'``)]),``select_params``=``(``'one'``, ``'two'``))`

```

在使用 extra() 时要避免在 select 字串含有 "%%s" 子串， 这是因为在 Django 中，处理 select 字串时查找的是 %s 而并非转义后的 % 字符。所以如果对 % 进行了转义，反而得不到正确的结果。

   **where / tables**
你可以使用 where 参数显示定义 SQL 中的 WHERE 从句，有时也可以运行非显式地连接。你还可以使用 tables 手动地给 SQL FROM 从句添加其他表。

where 和 tables 都接受字符串列表做为参数。所有的 where 参数彼此之间都是 "AND" 关系。

例如：

```
`Entry.objects.extra(where``=``[``'id IN (3, 4, 5, 20)'``])`

```

大致可以翻译为如下的 SQL:

```
`SELECT ``*` `FROM blog_entry WHERE ``id` `IN (``3``, ``4``, ``5``, ``20``);`

```

在使用 tables 时，如果你指定的表在查询中已出现过，那么要格外小心。当你通过 tables 参数添加其他数据表时，如果这个表已经被包含在查询中，那么 Django 就会认为你想再一次包含这个表。这就导致了一个问题：由于重复出现多次的表会被赋予一个别名，所以除了第一次之外，每个重复的表名都会分别由 Django 分配一个别名。所以，如果你同时使用了 where 参数，在其中用到了某个重复表，却不知它的别名，那么就会导致错误。

一般情况下，你只会添加一个未在查询中出现的新表。但是如果上面所提到的特殊情况发生了，那么可以采用如下措施解决。首先，判断是否有必要要出现重复的表，能否将重复的表去掉。如果这点行不通，就试着把 extra() 调用放在查询结构的起始处，因为首次出现的表名不会被重命名，所以可能能解决问题。如果这也不行，那就查看生成的 SQL 语句，从中找出各个数据库的别名，然后依此重写 where 参数，因为只要你每次都用同样的方式调用查询(queryset)，表的别名都不会发生变化。所以你可以直接使用表的别名来构造 where。

   **order_by**
如果你已通过 extra() 添加了新字段或是数据库，此时若想对新字段进行排序，就可以给 extra() 中的 order_by 参数传递一个排序字符串序列。字符串可以是 model 原生的字段名(与使用普通的 order_by() 方法一样)，也可以是 table_name.column_name 这种形式，或者是你在 extra() 的 select 中所定义的字段。

例如：

```
`q ``=` `Entry.objects.extra(select``=``{``'is_recent'``: ``"pub_date > '2006-01-01'"``})``q ``=` `q.extra(order_by ``=` `[``'-is_recent'``])`

```

这段代码按照 is_recent 对记录进行排序，字段值是 True 的排在前面，False 的排在后面。(True 在降序排序时是排在 False 的前面)。

顺便说一下，上面这段代码同时也展示出，可以依你所愿的那样多次调用 extra() 操作(每次添加新的语句结构即可)。

```
**params**

```

上面提到的 where 参数还可以用标准的 Python 占位符 -- '%s' ，它可以根据数据库引擎自动决定是否添加引号。 params 参数是用来替换占位符的字符串列表。

例如：

```
`Entry.objects.extra(where``=``[``'headline=%s'``], params``=``[``'Lennon'``])`

```

使用 params 替换 where 的中嵌入值是一个非常好的做法，这是因为 params 可以根据你的数据库判断要不要给传入值添加引号（例如，传入值中的引号会被自动转义）。

不好的用法：

```
`Entry.objects.extra(where``=``[``"headline='Lennon'"``])`

```

优雅的用法：

```
`Entry.objects.extra(where``=``[``'headline=%s'``], params``=``[``'Lennon'``])`

```

**defer(\*fields)**

在某些数据复杂的环境下，你的 model 可能包含非常多的字段，可能某些字段包含非常多的数据(比如，文档字段)，或者将其转化为 Python 对象会消耗非常多的资源。在这种情况下，有时你可能并不需要这种字段的信息，那么你可以让 Django 不读取它们的数据。

将不想载入的字段的名称传给 defer() 方法，就可以做到延后载入：

```
`Entry.objects.defer(``"lede"``, ``"body"``)`

```

延后截入字段的查询返回的仍是 model 类的实例。在你访问延后载入字段时，你仍可以获得字段的内容，所不同的是，内容是在你访问延后字段时才读取数据库的，而普通字段是在运行查询(queryset)时就一次性从数据库中读取数据的。

你可以多次调用 defer() 方法。每个调用都可以添加新的延后载入字段：

```
`# Defers both the body and lede fields.``Entry.objects.defer(``"body"``).``filter``(headline``=``"Lennon"``).defer(``"lede"``)`

```

对延后载入字段进行排序是不会起作用的；重复添加延后载入字段也不会有何不良影响。

你也可以延后载入关联 model 中的字段(前提是你使用 select_related() 载入了关联 model)，用法也是用双下划线连接关联字段：

```
`Blog.objects.select_related().defer(``"entry__lede"``, ``"entry__body"``)`

```

如果你想清除延后载入的设置，只要使用将 None 做为参数传给 defer() 即可：

```
`# Load all fields immediately.``my_queryset.defer(``None``)`

```

有些字段无论你如何指定，都不会被延后加载。比如，你永远不能延后加载主键字段。如果你使用 select_related() 获得关联 model 字段信息，那么你就不能延后载入关联 model 的主键。（如果这样做了，虽然不会抛出错误，事实上却不完成延后加载）

**Note**

defer() 方法(和随后提到的 only() 方法) 都只适用于特定情况下的高级属性。它们可以提供性能上的优化，不过前提是你已经对你用到的查询有过很深入细致的分析，非常清楚你需要的究竟是哪些信息，而且已经对你所需要的数据和默认情况下返回的所有数据进行比对，清楚两者之间的差异。这完成了上述工作之后，再使用这两种方法进行优化才是有意义的。所以当你刚开始构建你的应用时，先不要急着使用 defer() 方法，等你已经写完查询并且分析成哪些方面是热点应用以后，再用也不迟。

**only(\*fields)**

only() 方法或多或少与 defer() 的作用相反。如果你在提取数据时希望某个字段不应该被延后载入，而应该立即载入，那么你就可以做使用 only() 方法。如果你一个 model ，你希望它所有的字段都延后加载，只有某几个字段是立即载入的，那么你就应该使用 only() 方法。

如果你有一个 model，它有 name, age 和 biography 三个字段，那么下面两种写法效果一样的：

```
`Person.objects.defer(``"age"``, ``"biography"``)``Person.objects.only(``"name"``)`

```

你无论何时调用 only()，它都会立刻更改载入设置。这与它的命名非常相符：只有 only 中的字段会立即载入，而其他的则都是延后载入的。因此，连续调用 only() 时，只有最后一个 only 方法才会生效：

```
`# This will defer all fields except the headline.``Entry.objects.only(``"body"``, ``"lede"``).only(``"headline"``)`

```

由于 defer() 可以递增（每次都添加字段到延后载入的列表中），所以你可以将 only() 和 defer() 结合在一起使用，请注意使用顺序，先 only 而后 defer：

```
`# Final result is that everything except "headline" is deferred.``Entry.objects.only(``"headline"``, ``"body"``).defer(``"body"``)` `# Final result loads headline and body immediately (only() replaces any``# existing set of fields).``Entry.objects.defer(``"body"``).only(``"headline"``, ``"body"``)`

```

**using(alias)**

使用多个数据库时使用，参数是数据库的alias

```
`# queries the database with the 'default' alias.``>>> Entry.objects.``all``()` `# queries the database with the 'backup' alias``>>> Entry.objects.using(``'backup'``)`

```

**select_for_update(nowait=False)**

返回queryset，并将需要更新的行锁定，类似于SELECT ... FOR UPDATE的操作。

```
`entries ``=` `Entry.objects.select_for_update().``filter``(author``=``request.user)`

```

所有匹配的entries都会被锁定直到此次事务结束。

**raw(raw_query, params=None, translations=None)**

执行raw SQL queries


# 续接2

