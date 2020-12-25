# django多线程并发请求

settings配置中配置了 default 和 user_db 两个数据库。

### 测试代码

```python
class ConnectionHandler:
    def __getitem__(self, alias):
        if hasattr(self._connections, alias):
            return getattr(self._connections, alias)

        print('新建数据库连接对象', alias)        # 打印测试
        self.ensure_defaults(alias)
        self.prepare_test_settings(alias)
        db = self.databases[alias]
        backend = load_backend(db['ENGINE'])
        conn = backend.DatabaseWrapper(db, alias)
        setattr(self._connections, alias, conn)
        return conn

```

以上为建立数据库连接对象的django源代码，新开启一个线程时，都会重新建立数据库连接对象保存到local对象上（即当前线程上），下面会测试。



```python
print(threading.current_thread()) # django主线程

def test_view1(request):
    # print(request)
    print(111)
    print(connections['user_db '])
    print(threading.current_thread())
    time.sleep(5)
    return HttpResponse('ok')


def test_view2(request):
    print(222)
    print(connections['user_db '])
    print(threading.current_thread())
    time.sleep(5)
    return HttpResponse('ok')


def test_view3(request):
    print(333)
    print(connections['user_db '])
    print(threading.current_thread())
    time.sleep(5)
    return HttpResponse('ok')
```



### 测试1

测试过程：

runserver启动django程序，输出如下

```
新建数据库连接对象 default
新建数据库连接对象 default
Performing system checks...

新建数据库连接对象 default
<_DummyThread(Dummy-1, started daemon 9748)>         # 主线程号
System check identified no issues (0 silenced).
December 04, 2020 - 10:09:15
Django version 2.1.15, using settings 'DjangoDbTest.settings'
Starting development server at http://127.0.0.1:8500/
Quit the server with CTRL-BREAK.
```

结论：

从上可以看到，程序(主线程)启动，会新建数据库连接对象，主线程为  `<_DummyThread(Dummy-1, started daemon 9748)> `



### 测试2

测试过程：

1.请求`test_view1` 等待响应返回，

2.再请求`test_view2`等待响应返回，

3.再请求`test_view3`

输出如下：

```
新建连接对象 default
新建连接对象 user_db

111
<django.db.backends.mysql.base.DatabaseWrapper object at 0x00000215A12965F8>
<Thread(Thread-3, started daemon 22156)>
[04/Dec/2020 10:15:13] "GET /test/ HTTP/1.1" 200 2
222
<django.db.backends.mysql.base.DatabaseWrapper object at 0x00000215A12965F8>
<Thread(Thread-3, started daemon 22156)>
[04/Dec/2020 10:15:22] "GET /test2/ HTTP/1.1" 200 2
333
<django.db.backends.mysql.base.DatabaseWrapper object at 0x00000215A12965F8>
<Thread(Thread-3, started daemon 22156)>
[04/Dec/2020 10:15:30] "GET /test3/ HTTP/1.1" 200 2
```

结果：

可以看到，每次请求都会使用已经存在(第一次创建)的线程，和该线程上创建的数据库连接对象。

说明django请求会复用已有的线程来处理请求。



### 测试3

测试过程（重启django再测试）：

1.请求`test_view1` ，

2.在其返回前，同时请求`test_view2`，

3.在它们返回前，同时请求`test_view3`

即同时发出三个不同接口的请求，输出如下：

```
新建数据库连接对象 default
新建数据库连接对象 user_db
111
<django.db.backends.mysql.base.DatabaseWrapper object at 0x000001E3E51F9B38>
<Thread(Thread-5, started daemon 5044)>       # 1
新建数据库连接对象 default
新建数据库连接对象 user_db
222
<django.db.backends.mysql.base.DatabaseWrapper object at 0x000001E3E520DB00>
<Thread(Thread-4, started daemon 13036)>      # 2
新建数据库连接对象 default
新建数据库连接对象 user_db               
333
<django.db.backends.mysql.base.DatabaseWrapper object at 0x000001E3E5214518>
<Thread(Thread-6, started daemon 23544)>       # 3
[04/Dec/2020 10:49:56] "GET /test/ HTTP/1.1" 200 2
[04/Dec/2020 10:49:57] "GET /test2/ HTTP/1.1" 200 2
[04/Dec/2020 10:49:59] "GET /test3/ HTTP/1.1" 200 2
```

结果：

可以看到，每次请求都新开启了线程来处理请求，同时每个新线程新建数据库连接对象



### 综合测试2和测试3

可以看出Django请求时，前面创建的线程正在处理请求，所以会新开启线程处理新来的请求

- **如果存在可用的线程，那么就复用该线程(上个请求释放的线程)；**
- **如果没有可用的线程，那么会新开启一个线程来处理**



### 测试4

隔一段时间后，再去请求三个接口，发现每个接口都会开启新的线程，说明创建的线程是有寿命限制的。



### 遗留问题

如果同时多次请求同一个接口，发现该接口有几个请求有时是串行执行(有一些又是并行)请求的，必须等待上个请求返回，才开始执行下一个请求。

问题：同时多次请求同一个接口，何种情况才会开启新的线程来处理？
