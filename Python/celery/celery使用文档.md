中文翻译文档：<https://blog.csdn.net/weixin_40475396/article/details/80439781>

相关函数：

```
任务函数名.name : 查看任务名称
app.conf: 查看或设置celery配置
app.tasks: 查看celery示例注册的所有任务（包括celery内建任务）
```

@app.task() 的参数：

```
typing=False  通过设置任务的 typing 属性为 False 来禁用参数检查
```



# 一，Application

## 1. 主模块名称

```python
# app = Celery()
app = Celery('projectname')

@app.task
def add(x,y):
    return x + y

当Celery不能探查到这个任务函数属于哪个模块时，它将使用主模块名称来产生任务名称的前缀。    
初始化app时：
未给主模块命名：add.name 为 __main__.add
主模块命名projectname：add.name 为 projectname.add
```

## 2. 配置

可以设置一些选项来改变 Celery 的工作方式。这些选项可以直接在 app 实例上进行设置，或者也可以使用一个指定的配置模块

**配置使用 `app.conf` 变量保存：**

```python
# 查看配置
app.conf.timezone  # 'Europe/London'
# 直接设置配置值:
app.conf.enable_utc = True
# 使用 update 方法同时更新多个配置项
app.conf.update(enable_utc=True,timezone='Europe/London')
```

**实际的配置对象由多个字典决定，配置项按以下顺序查询：** 

1. 运行时的配置修改 
2. 配置模块（如果声明） 
3. 默认配置（celery.app.defaults）



#### add_defaults

 `app.add_defaults()` 方法添加新的默认配置源

所有可用配置的完整列表及其默认值请参照 Configuration reference



#### config_from_object

`app.config_from_object()` 方法从一个配置对象加载配置。

它可以是一个配置模块，或者任意包含配置属性的对象

调用 `config_from_object` 方法后配置被重置。设置附加的配置应该在调用这个方法之后。

**示例1： 使用模块名 /  使用模块对象**

```python
"""`app.config_from_object()` 方法的参数可以是一个 python 模块的全限定名称，或者一个 python 属性名，例如：”celeryconfig”，”myproj.config.celery”，或者 “myproj.config:CeleryConfig”：
"""
from celery import Celery
# import celeryconfig  # 使用模块对象

app = Celery()
app.config_from_object('celeryconfig')
# app.config_from_object(celeryconfig)  # 填入导入的celeryconfig模块对象
```

celeryconfig.py:

```
enable_utc = True
timezone = 'Europe/London'
```

提示： 建议使用模块名的方式加载，因为这种情况下当prefork池使用时，配置模块不必序列化。如果遇到配置问题或者序列化错误，可以尝试使用模块名的方式加载配置。

**示例2：使用配置类/对象**

```python
from celery import Celery

app = Celery()

class Config:
    enable_utc = True
    timezone = 'Europe/London'

app.config_from_object(Config)
# or using the fully qualified name of the object:
#   app.config_from_object('module:Config')
```



#### config_from_envvar

`app.config_from_envvar()` 从环境变量中获取配置模块名称。

#### 敏感配置

你想打印配置信息，作为调试信息或者类似，你也许不想暴露密码和API秘钥这类信息。

app.conf.humanize(with_defaults=False, censored=True)

如果一个配置项键名包含以下字符串，它将被看作是敏感的： `API，TOKEN，KEY，SECRET，PASS，SIGNATURE，DATABASE`

## 3.延迟加载

一个应用实例是延迟加载的，意味着它只有在实际调用的时候才会被求值。 

```
创建一个celery 实例只会做如下事情：
1. 创建一个用于事件的逻辑时钟实例
2. 创建一个任务注册表
3. 将自己设置为当前应用实例（如果 set_as_current 参数被禁用将不会做此设置）
4. 调用 app.on_init() 回调函数(默认不做任何事情)
```

`app.task()` 装饰器在**任务定义时不会创建任务，而是延迟任务的创建 到任务使用时(包括访问任务对象的属性)，或者应用被终止时**。

应用的终止有两种情况：显示调用 `app.finalize()` 终止，或者通过访问 `app.tasks` 属性隐示终止。



## 4.打破链式操作



## 5.抽象任务

所有使用 task() 装饰器创建的任务都会继承应用的基础 `Task` 类。

可以使用装饰器的 base 参数给任务声明一个不同的基类



# 二，Task

### 概念

**任务只有在定义他们的模块被导入时才会被注册。当工作单元接收到任务，它会根据任务名称从任务注册表中找到实际要执行的代码。**

1.每个任务都有不同的名称，发给 celery 的任务消息中会引用这个名称，工作单元就是根据这个名称找到正确的执行函数

2.**任务消息只有在被工作单元确认后才会从队列中删除**。工作单元会预先保存许多任务消息，如果工作单元被杀死-由于断电或者其他原因-任务消息将会重新传递给其他工作单元。

3.默认的行为是在任务即将执行之前预先确认任务消息，这使得已经开始的任务不会再被执行。

   如果你的任务函数是幂等的，你可以设置 `acks_late` 选项让工作单元在任务执行返回之后再确认任务消息

告警：
一个无限期阻塞的任务会使得工作单元无法再做其他事情。

如果你的任务里有 I/O 操作，请确保给这些操作加上超时时间，例如使用 `requests` 库时给网络请求添加一个超时时间



### 1.基础

使用 `task()` 装饰器，你可以很容易**创建一个任务**，任务装饰器可以从 Celery 应用实例上获取: app.task

任务上可以设置很多选项，这些选项作为参数传递给装饰器

```
@app.task(serializer='json')
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

当使用多个装饰器装饰任务函数时，确保 `task` 装饰器最后应用（在python中，这意味它必须在第一个位置）

#### 绑定任务

`bind=True`: 任务函数的第一个参数是任务实例本身(`self`)

```
@task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)
```

**绑定任务在这些情况下是必须的**：任务重试（使用 `app.Task.retry()` )，访问当前任务请求的信息，以及你添加到自定义任务基类的附加功能。

#### 任务继承

 `base` :  任务装饰器的 `base` 参数可以声明任务的基类

`@task(base=MyTask)`



### 2.任务名称

每个任务必须有不同的名称。如果没有显示提供名称，任务装饰器将会自动产生一个，产生的名称会基于这些信息： 1）任务定义所在的模块， 2）任务函数的名称

显示设置任务名称:  `@app.task(name='sum-of-two-numbers')`

最好是使用模块名称作为命名空间： `@app.task(name='tasks.add')` (不提供时 默认就是这种)



### 3.任务请求

`app.Task.request` 包含当前执行任务相关的信息与状态。

```
任务请求定义了以下属性：
id: 执行任务的唯一 id
group: 如果任务属于一个组，这个属性表示组 id
chord: 任务所属 chord 的 id(如果任务是header的一部分)
correlation_id: 自定义 ID，用来处理类似重复删除操作
args: 位置参数
kwargs: 关键字参数
origin: 发送任务消息的主机名
retries: 当前任务已经重试的次数。它是一个从0开始的整数
is_eager: 如果任务是在客户端本地执行而不是通过工作单元执行，那么这个属性设置为 True
eta: 任务的原始 ETA（如果存在）。用 UTC 时间表示（依赖于 enable_utc 设置）
expires: 原始的过期时间（如果存在）。用 UTC 时间（依赖于 enable_utc 设置）
hostname: 执行任务的工作单元的节点名称
delivery_info: 附加的消息传递消息。它是一个映射，包含用来递送任务消息的路由规则以及路由键。例如， app.Task.retry() 函数可以根据它来重新发送消息到相同的目标队列。该映射中键的可用性取决于使用的消息中间件
reply-to: 回复发送的目的队列的名称（例如在RPC 存储后端中使用）
called_directly: 如果任务不是由执行单元执行，这个属性设置为 True
timelimit: 它是一个元组，表示任务上当前激活的（软性，硬性）时间限制（如果存在）
callbacks: 如果任务执行成功，将被调用的函数签名的列表
errback: 如果任务还行失败，将被调用的函数签名的列表
utc: 如果调用者使能了 UTC（enable_utc），这个属性为True
```



### 4.日志



### 5.重试

`app.Task.retry()` 函数可以用来重新执行任务，例如在可恢复错误的事件中

当你调用 `retry` 函数，它将发送一个新的消息，使**用相同的任务 id**，而且它会小心**确保该消息投递到原始任务相同的队列。**

一个任务被重试将记录为一个任务状态，因此你可以使用结果实例跟踪任务的进度（查看状态这一节）

```python
@app.task(bind=True)
def send_twitter_status(self, oauth, tweet):
    try:
        twitter = Twitter(oauth)
        twitter.update_status(tweet)
    except (Twitter.FailWhaleError, Twitter.LoginError) as exc:
        raise self.retry(exc=exc, max_retries=5)
```

注意： 

`app.Task.retry()` 调用将会抛出一个异常使得任意 `retry` 后面的代码都不会被执行

`exc` 参数是用来传递在日志中使用或者在后端结果中存储的异常信息

当任务带有 `max_retries` 值，如果已经达到最大尝试次数，当前异常会被重新被抛出，除了以下情况：

- 没有给定 exc 参数 这种情况下，`MaxRetriesExceededError` 异常将会被抛出。 
- - 当前没有异常 如果没有原始异常被重新抛出，`exc` 参数将会被使用

```
self.retry(exc=Twitter.LoginError())
```

将会抛出 `exc` 给定的异常。



#### 5.1 使用自定义重试延迟

**重试前会等待给定的时间**，默认的延迟是由 `default_retry_delay` 属性定义。默认设置为 3 分钟。注意延迟设置的单位是秒（int 或者 float）

你可以通过提供 **`countdown` 参数覆盖这个默认值**

```python
@app.task(bind=True, default_retry_delay=30 * 60)  # retry in 30 minutes.
def add(self, x, y):
    try:
        something_raising()
    except Exception as exc:
        # overrides the default delay to retry after 1 minute
        raise self.retry(exc=exc, countdown=60)
```



#### 5.2 对已知异常的自动尝试

想在特定异常抛出时重试任务，通过使用任务装饰器中的 `autoretry_for` 参数让 Celery 自动尝试一个任务：

```python
@app.task(autoretry_for=(FailWhaleError,))
```

给内部调用的 `Task.retry` 函数传递自定义的参数，可以传递 `retry_kwargs` 参数给任务装饰器：

```
@app.task(autoretry_for=(FailWhaleError,),retry_kwargs={'max_retries': 5})
```

如果你想在发生任意错误时重试，可以这样：

@app.task(autoretry_for=(Exception,))



### 6. 选项列表

任务装饰器有一系列选项可以用来修改任务的行为，例如可以通过 `rate_limit` 选项来设置任务的速率限制

传递给任务装饰器的关键字参数将会设置为结果任务类的一个属性。

下面是内建属性的列表：

 `Task.name`  ：任务注册的名称

`Task.request`：如果任务将被执行，这个属性会包含当前请求的信息。会使用Thread local 存储（另见任务请求这一节）

`Task.max_retries` ：只有在任务函数中调用了 `self.retry` 函数或者给任务装饰器传递了 `autoretry_for` 参数时才有用。最大尝试次数。如果尝试次数超过这个值，`MaxRetriesExceededError` 异常将会被抛出

`Task.throws`： 一个可选的预知错误类元组，不认为是真正的错误。

`Task.default_retry_delay` ：任务重试前延迟的默认时间值，以秒为单位。可以是 int 或者 float。默认值是 3 分钟

`Task.rate_limit`：设置任务类型的速率限制（限制给定时间内可以运行的任务数量）。当设置了速率限制后，已经开始的任务仍然会继续完成，但是任务开始前将等待一些时间。

如果这个属性设置成 `None`，速率限制将不生效。如果设置为整数或者浮点数，它将解释成”没秒任务数”。

速率限制可以以秒、分钟或者小时声明，只要值后面附加 “/s”, “/m”, “/h”。任务将在给定时间内平均分布。

这里指的是每个任务工作单元的限制，不是全局速率限制。要实现全局速率限制（例如，对一个 API 的每秒请求数限制），你必须限定指定的任务队列。

`Task.time_limit` ：任务的硬性时间限制，以秒为单位。如果没有设置，那么将使用任务工作单元的默认值。

`Task.soft_time_limit`： 任务的软性时间限制。如果没有设置，那么将使用任务工作单元的默认值。

`Task.ignore_result` ：不存储任务状态。注意这意味着你不能使用 `AsyncResult` 来检测任务是否完成，或者获取任务返回值。

`Task.store_errors_even_if_ignored` ：如果设置为 `True`，即使任务被设置成忽略结果，错误也会也存储记录

`Task.serializer` ：表示默认使用的序列化方法的一个字符串。默认值是 `task_serializer` 设置。可以是 `pickle,json,yaml` 或者任意通过 `kombu.serialization.registry` 注册过的自定义序列化方法。

`ignore_result`：ignore_result=True 忽略任务的返回结果，因为存储结果耗费时间和资源。

可以通过 `task_ignore_result` 设置全局忽略任务结果



### 7. 状态

#### 内建状态

- PENDING  任务等待被执行或者状态未知。任何不知道的任务 id 都被认为在 pending 状态
  - res.get() 取结果时将会阻塞程序
- STARTED  任务已经开始。默认不记录此状态，如果要启用:  `@app.task(track_started=True)`
  - 结果包含执行当前任务的工作单元的进程 ID 和主机名。
- SUCCESS  任务已经被成功执行
  - 结果包含任务的返回值
- FAILURE   任务执行失败,（也可能是任务未注册NotRegistered）
  - 结果包含抛出的异常，以及异常抛出时的堆栈回溯信息，propagates: True
- RETRY  任务被重试   
  - 结果包含导致重试的异常，以及异常抛出时的堆栈回溯信息  propagates: True
- REVOKED  任务被取消  
  - propagates: True

### 8. 后端存储

### 9. 自定义任务类

#### Handler

`after_return(self, status, retval, task_id, args, kwargs, einfo)` 任务返回后调用的处理函数。

`on_retry(self, exc, task_id, args, kwargs, einfo)` 任务重试时由工作单元调用

`on_success(self, retval, task_id, args, kwargs)` 任务成功执行完由工作单元调用的处理函数。



# 三，调用任务

API 中定义了一个执行选项的标准集，以及三个方法：

- `apply_async(args[, kwargs[, ...]])`   发送任务消息 
- `delay(*args, **kwargs)` 发送任务消息的简写，不支持执行选项 
- `calling`(`__call__`) 直接调用任务对象，意味着任务不会被工作单元执行，而是在当前进程中执行（不会发送任务消息）

### 1. 使用

- T.delay(arg, kwarg=value)是 `.apply_async` 方法的参数简写方式。（`.delay(*args, **kwargs)` 会调用 `.apply_async(args, kwargs)`） 
- `T.apply_async((arg,), {'kwarg': value})`
- `T.apply_async(countdown=10)` 从现在开始10秒后执行任务。 - 
- `T.apply_async(countdown=60, expires=120)` 从现在开始1分钟后执行任务，任务过期时间为2分钟 

使用 `apply_async()` 方法，你必须这样写：

```
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
```



### 2. Linking 链式任务

Celery 支持链接任务，这使得执行一个任务之后接着执行另一个任务。回调任务会将父任务的结果作为本任务函数的部分参数。

```python
add.apply_async((2, 2), link=add.s(16))
# 这里第一个任务 (4) 将会发送到另一个任务将 16 与前面结果相加，形成表达式 (2 + 2) + 16 = 20
```



### 3. On message

Celery 通过设置 `setting_on_message` 回调支持捕获所有状态变更

例如，对于长时间任务，你可以通过如下类似操作更新任务进度：

```python
@app.task(bind=True)
def hello(self, a, b):
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 50})
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 90})
    time.sleep(1)
    return 'hello world: %i' % (a+b)

def on_raw_message(body):
    print(body)

r = hello.apply_async()
print(r.get(on_message=on_raw_message, propagate=False))
```



### 4. 其他

#### 定时执行

`countdown`  ETA（估计到达时间）声明任务将被执行的时间（只能保证在其后，因为可能有多个此时执行的任务）。`countdown` 是设置 ETA 的快捷方式

```
result = add.apply_async((2, 2), countdown=3)
result.get()    # this takes at least 3 seconds to return
```



#### 任务过期时间

`expires` 参数定义了一个可选的过期时间，可以是任务发布后的秒数，或者使用 `datetime` 声明一个日期和时间

当工作单元接收到一个过期任务，它会将任务标记为 `REVOKED`



#### 消息发送重试

当链接失败，celery 会重试发送任务消息，并且重试行为可以设置 - 比如重试的频率，或者最大重试次数 - 或者禁用所有。

禁用消息发送重试，你可以设置重试的执行选项为 `False`:
`add.apply_async((2, 2), retry=False)`
