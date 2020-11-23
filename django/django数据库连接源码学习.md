# ConnectionHandler

数据库的连接配置处理类，管理不同别名数据库的连接

connections = ConnectionHandler()  #  一个可选的数据库定义字典

可以通过connections ['别名'] 获取指定别名的数据库连接对象

```python
class ConnectionHandler:
	def __init__(self, databases=None):
	    """一个可选的数据库定义字典"""
	    self._databases = databases # 数据库配置
	    self._connections = local() # 从当前线程变量获取所有数据库连接对象
		
	def databases(self):
	    """获取所有数据库配置字典"""
	    # 如果databases不传，则从配置文件取
	
	def ensure_defaults(self, alias):
	    """将没有提供任何设置的给定连接的默认值放入设置字典中"""
	    # 对于已设置好的连接，加入一些默认配置参数：诸如，AUTOCOMMIT，ENGINE，CONN_MAX_AGE，OPTIONS
		
	def __getitem__(self, alias):
     	    """获取指定别名的数据库连接对象
            当使用connections['alias']时，就会调用该方法获取连接
            """
            # 从当前线程local中取该别名的连接对象
            if hasattr(self._connections, alias):
        	return getattr(self._connections, alias)
            ......
            # 使用配置信息新建数据库连接
            db = self.databases[alias]
            backend = load_backend(db['ENGINE']) # 使用指定数据库后端
            conn = backend.DatabaseWrapper(db, alias) # 新建连接对象（DatabaseWrapper实例）
            setattr(self._connections, alias, conn) # 保存到线程变量中
            return conn
    
	def close_all(self):
            """关闭当前线程中的所有数据库连接"""
            for alias in self:
                try:
                    connection = getattr(self._connections, alias)
                except AttributeError:
                    continue
                connection.close()
    
```



**django项目启动时，会为配置字典中的每个数据库生成一个连接对象(DatabaseWrapper实例)**

`django.db.models.base` 中的Model类

```python
class Model(metaclass=ModelBase):
	@classmethod
    def _check_long_column_names(cls)：
    	......
    	for db in settings.DATABASES:
    		if not router.allow_migrate_model(db, cls):
                continue
            connection = connections[db]  # 获取每个别名的连接对象，没有则新建连接对象
            ......
```





# BaseDatabaseWrapper

数据库连接类，其实例对象表示一个数据库连接

```python
class BaseDatabaseWrapper:
    """Represent a database connection."""
    def __init__(self, settings_dict, alias=DEFAULT_DB_ALIAS,
                 allow_thread_sharing=False):
        # connection 连接相关的的属性.
        # 底层数据库连接.（MYSQLdb包中的Connection对象）
        self.connection = None
        # 该数据库的配置字典
        self.settings_dict = settings_dict
        self.alias = alias
        # 在调试模式或显式启用时进行 查询日志 记录
        self.queries_log = deque(maxlen=self.queries_limit)
        self.force_debug_cursor = False

        # Transaction 事务相关的属性
        # 跟踪连接是否处于自动提交模式。默认情况下，它不是
        self.autocommit = False
        # 跟踪连接是否在由“atomic”管理的事务中。
        self.in_atomic_block = False
        # 递增以生成唯一的保存点id。
        self.savepoint_state = 0
        # 'atomic'创建的保存点列表.
        self.savepoint_ids = []
        # 跟踪最外层的“原子”块是否应该在退出时提交，如果自动提交在进入时处于活动状态
        self.commit_on_exit = True
        # 跟踪由于内部块中的异常，事务是否应该回滚到下一个可用的保存点。
        self.needs_rollback = False

        # Connection termination 连接终止相关的属性.
        self.close_at = None
        self.closed_in_transaction = False
        self.errors_occurred = False

        # Thread-safety 线程安全相关的属性.
        self.allow_thread_sharing = allow_thread_sharing
        self._thread_ident = _thread.get_ident()

        # 事务提交时要运行的无参数函数列表。
		# 每个条目都是一个(sids, func)元组，其中sids是注册此函数时的一组活动保存点id。
        self.run_on_commit = []

        # 我们应该在下一次调用set_autocommit(True)时运行on-commit钩子吗?
        self.run_commit_hooks_on_set_autocommit_on = False

        # 围绕execute()/executemany()调用调用的包装器堆栈。
        # 每个条目都是一个函数，具有5个参数:execute、sql、params、many和context。
        # 函数负责调用execute(sql、params、many、context)。
        self.execute_wrappers = []

        self.client = self.client_class(self)
        self.creation = self.creation_class(self)
        self.features = self.features_class(self)
        self.introspection = self.introspection_class(self)
        self.ops = self.ops_class(self)
        self.validation = self.validation_class(self)
    
    @property
    def queries(self):
        """返回该连接的所有查询日志"""
    
    def connect(self):
        """连接到数据库。假设连接已关闭。"""
        # 检查无效配置
        self.check_settings()
        # 以防前一个连接在原子块中被关闭
        self.in_atomic_block = False
        self.savepoint_ids = []
        self.needs_rollback = False
        
        # 定义何时关闭连接的重置参数；
        # 数据库配置字典中设置CONN_MAX_AGE，表示数据库连接保持的时间
        # 默认为0，表示每次请求进来新建连接，请求结束就关闭连接
        # 如果为None，表示永久保持连接状态
        max_age = self.settings_dict['CONN_MAX_AGE']
        self.close_at = None if max_age is None else time.time() + max_age
        self.closed_in_transaction = False
        self.errors_occurred = False
        
        # 建立底层连接
        conn_params = self.get_connection_params()
        self.connection = self.get_new_connection(conn_params)
        self.set_autocommit(self.settings_dict['AUTOCOMMIT'])
        self.init_connection_state()
        # 发送底层连接创建的信号
        connection_created.send(sender=self.__class__, connection=self)

        self.run_on_commit = []
        
    def get_connection_params(self):
        """返回建立新连接所需的相关参数"""
    
    def get_new_connection(self, conn_params):
        """打开一个到数据库的连接。
        返回一个MYSQLdb包中的Connection对象
        """
        # 使用MYSQLdb的connect方法连接数据库
        return Database.connect(**conn_params) 
    
    def close(self):
        """关闭与数据库的连接。"""
        # ......
        try:
            self._close()
        finally:
            if self.in_atomic_block:
                self.closed_in_transaction = True
                self.needs_rollback = True
            else:
                self.connection = None
    
    def close_if_unusable_or_obsolete(self):
        """如果出现不可恢复的错误，请关闭当前连接; 或者它是否超过了它的最大寿命。"""
        # ......
        # 发生错误，关闭连接
        if self.errors_occurred:
            if self.is_usable(): # self.connection.ping() 测试连接是否可用
                self.errors_occurred = False
            else:
                self.close()
                return
		# 寿命过期，即当前时间 > self.close_at (connect方法中根据CONN_MAX_AGE算得)
        if self.close_at is not None and time.time() >= self.close_at:
            self.close()
            return
        
    def ensure_connection(self):
        """确保已建立到数据库的连接（创建游标时需要判断该connection是否存在）"""
        if self.connection is None:
            with self.wrap_database_errors:
                self.connect()  # 底层连接为None，新建连接
                
    def cursor(self):
        """创建一个游标."""
        return self._cursor() # _cursor() 继续调用create_cursor(name)
    
    def create_cursor(self, name=None):
        cursor = self.connection.cursor() # MYSQLdb中Connection对象的cursor方法
        return CursorWrapper(cursor)
```

其他方法：

- all()  获取所有连接对象

  


# django的数据库行为

### 1.通过简单测试发现，默认情况下，**django项目启动时**：

- 会为配置字典中的每个数据库生成一个连接对象(DatabaseWrapper实例)，但不会真正建立底层数据库连接。

- 但会给默认'default'数据库，建立一个底层数据库连接（真正连接到数据库）
  - 测试时看到，它调用了DatabaseWrapper中的connect方法，进而调用MYSQLdb的connect方法建立连接。

**注意**：除非程序中显示执行了connect方法或调用了connect方法的函数；比如，提前创建连接池.


  

### 2.何时会调用connect方法，连接数据库？

使用连接对象connections['alias'].cursor() 创建游标对象时，将调用connect。

- 前提是该数据库底层连接未创建或已关闭，即该连接对象的connection属性为None

  创建游标cursor前，会先判断底层连接connection是否为空；若底层连接有效，将复用该连接；比如设置了CONN_MAX_AGE，有效期内，不会再调用connect

  ```python
  def ensure_connection(self):
      """确保已建立到数据库的连接"""
      if self.connection is None:
      	with self.wrap_database_errors:
      		self.connect()  # 底层连接为None，新建连接
  ```


- connect方法中设置close_at ，即关闭该连接的时间，根据CONN_MAX_AGE算得。

  ```
  close_at = None if max_age is None else time.time() + max_age
  ```

  

### 3.何时调用close方法，关闭数据库连接？

1.出现不可恢复的错误，将调用close关闭当前连接

2.请求结束(响应返回)后：若 当前时间戳 > close_at ，会调用close关闭连接

```
if self.close_at is not None and time.time() >= self.close_at:
	self.close()
```


  

### 4. django请求开始和请求完成的数据库行为

- 请求开始之前：重置该连接的查询日志，关闭不可用或过期的连接

- 请求结束之后：关闭不可用或过期的连接

  通过调用`close_if_unusable_or_obsolete`方法

```python
"""django.db.__init__目录下"""
# 注册一个事件，在Django请求开始时重置保存的查询日志。
def reset_queries(**kwargs):
    for conn in connections.all():
        conn.queries_log.clear()

signals.request_started.connect(reset_queries)


# 注册一个事件来重置事务状态并在其生命周期结束后关闭连接。
def close_old_connections(**kwargs):
    for conn in connections.all():
        # 关闭不可用或过期的连接
        conn.close_if_unusable_or_obsolete()

signals.request_started.connect(close_old_connections)
signals.request_finished.connect(close_old_connections)

```







