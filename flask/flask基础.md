- request.get_json()：将传入的文本数据转换为字典或列表。



# orm

#### 表映射模型

```
flask-sqlacodegen mysql://root:Wuxian@123@10.43.246.48/pyvboard?charset=utf8 --tables t_smt_remaining --outfile "app/models/ToMaintain/model2.py" --flask
```



#### 两种查询方式：

> 1.模型名.query.filter()

单表查询：obj = A.query.filter(A.name=='zs').first()

连接查询：obj = A.query.join(B, A.id==B.id).filter(A.name=='zs').first()

- 会返回一个Model_A的对象，包含该表中所有字段，不包含其他表中的任何字段
- 需要取指定字段或另一个表字段：使用with_entities(A.name,B.code)
- 后面接`all()`或`first()` ,结果为model对象，不接为`BaseQuery`对象
- 打印执行的原始sql：
  - print(obj.query)  或 str(obj.query)  :一定要调用str方法，print自动调用`__str__`



> 2.db.session.query(模型).filter()

单表查询：obj= db.session.query(A).filter(A.name=='zs').first()

连接查询：obj = db.session.query(A).join(B, A.id==B.id).filter(A.name=='zs').first()

- 后面接`all()`或`first()` ,结果为model对象，不接为`BaseQuery`对象
- 需要取指定字段或另一个表字段：query(A.name,B.code)
- 打印执行的原始sql (后面不能接`all`或`first`)：
  - print(obj)  或 str(obj)  :一定要调用str方法，print自动调用`__str__`

**打印原始sql的总结：**

- 如果是`BaseQuery`对象，直接print打印该对象，或str(obj)
- 如果是模型对象，需调用`query`属性，print(obj.query) 或 str(obj.query)



### 模型方法

- with_entities()   查询指定字段





# 分页

为了显示某页中的记录，要把all()换成flask-SQLALchemy提供的paginate()方法。页数为paginate的第一个参数，第二个参数是每页显示的条数，如果没有指出，默认为20个记录

```
paginate = Students.query.paginate(1,10)
```

paginate 为一个Pagination类对象：

- paginate.items 当前页的记录对象 ( 模型对象）
- paginate.total 显示一共有多少条记录
- paginate.pages 显示一共有多少页
- paginate.has_prev 是否有上一页
- paginate.prev_num 上一页的id
- paginate.has_next 是否有下一页
- paginate.next_num 下一页的id
- iter_page:一个迭代器，返回一个在分页导航中显示的页数列表













# 数据库事务

```
try:
            group = CaseGroup()
            group.name = form.name.data
            group.info = form.info.data
            db.session.add(group)
            db.session.flush()
            if form.users.data:
                current_app.logger.info(group.id)
                for user in form.users.data:
                    user_auth = UserAuth()
                    user_auth.user_id = user
                    user_auth.auth_id = group.id
                    user_auth.type = UserAuthEnum.GROUP
                    db.session.add(user_auth)
            db.session.commit()
        except Exception as e:
            db.session.rollback()
            raise UnknownException(msg='新增异常 数据已回滚')

```

这个例子中，新增关联表的数据需要已新增分组数据的id，此时未commit所以自增id为None，需要在新增分组后使用`db.session.flush()`刷新获取分组id。

首先要明白flush和commit区别:

```
  \> flush: 写数据库，但不提交，也就是事务未结束

  \> commit: sqlalchemy会自动创建隐私的事务，先调用flush写数据库，然后提交，结束事务，并开始新的事务
```
