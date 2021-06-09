



# celery的使用

### 1.django中使用celery

celery 实例 初始化文件 djcelery.init.py

```python
import os
from celery import Celery
from celery.schedules import crontab
from datetime import timedelta
from . import celeryConfig

from PyVbord.settings import dev

# 设置 Django 的配置文件
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'PyVbord.settings.dev')

# 创建 celery 实例
celery_app = Celery('PyVbord')
# 加载 celery 配置文件
celery_app.config_from_object(celeryConfig)
# 检测异步任务
celery_app.autodiscover_tasks(lambda: dev.INSTALLED_APPS)

""" 配置定时任务 """
celery_app.conf.update(
    CELERYBEAT_SCHEDULE={
        'email_alert': {
            'task': 'PyVbord.apps.WebPage.send_email.tasks.email_alert',
            'schedule': crontab(minute='0', hour='10, 17', day_of_week='*'),
        },
    }
）
```

celery配置文件: djcelery.celeryconfig.py

```python
BROKER_URL = f'redis://{REDIS_HOST}:{REDIS_PORT}/2'  # 任务调度队列,接收任务,使用redis
BROKER_POOL_LIMIT = 100  # Borker 连接池, 默认是10

CELERY_TIMEZONE = 'Asia/Shanghai'  # 时区
CELERY_ACCEPT_CONTENT = ['json']
TASK_SERIALIZER = 'json'

CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/1'  # 任务结果存储
CELERY_RESULT_SERIALIZER = 'json'  # 任务执行结果的序列化方式
CELERY_MAX_CACHED_RESULTS = 1000  # 任务结果最大缓存数量

CELERY_IGNORE_RESULT = True  # 丢弃任务的结果

# 日志等级
WORKER_REDIRECT_STDOUTS_LEVEL = 'INFO'

# 其他配置，重命名默认队列名，（默认为celery）
CELERY_DEFAULT_QUEUE = "default"   
CELERY_QUEUES = {
    "default": { # 这是上面指定的默认队列
        "exchange": "default",
        "exchange_type": "direct",
        "routing_key": "default"
    }
}
```

### 2.flask中使用celery

celery 初始化文件: flask_celery.init.py

```python
from celery import Celery

def make_celery(app):
    """创建celery实例"""
    celery_app = Celery(__name__)
    celery_app.config_from_object('libs.flask_celery.celery_config')
    # 自动检测指定包下的tasks文件中的任务
    celery_app.autodiscover_tasks(['app.api_1_0.ToMaintain'])

    class ContextTask(celery_app.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)

    celery_app.Task = ContextTask
    return celery_app


from manage import app
celery_app = make_celery(app)
```





# 使用celery工作

## 小知识

使用redis作为boroker和backend

### 1.broker

##### 默认队列名

默认队列名为 `celery`，使用redis的List类型

##### 消息格式

队列中的消息格式为：

```
{\"body\": \"W1syMDIwMTJdLCB7fSwgeyJjYWxsYmFja3MiOiBudWxsLCAiZXJyYmFja3MiOiBudWxsLCAiY2hhaW4iOiBudWxsLCAiY2hvcmQiOiBudWxsfV0=\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"PyVbord.apps.StockManagement.CodeManagementTool.tasks.match_self_table\", \"id\": \"6e3b00ea-37f1-4bdf-8a55-a779999fbbac\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"6e3b00ea-37f1-4bdf-8a55-a779999fbbac\", \"parent_id\": null, \"argsrepr\": \"(202012,)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen18552@dggy5ywx8001901\"}, \"properties\": {\"correlation_id\": \"6e3b00ea-37f1-4bdf-8a55-a779999fbbac\", \"reply_to\": \"3fb91bdc-3267-36ac-b16e-b77a8634fe01\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"asynchronous_task\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"47e32074-fc35-4308-9313-b37b24dbe467\"}}
```



##### 2.结果存储

使用string类型进行任务结果存储，

key为:`celery-task-meta-181002b5-bd28-4c1e-b4e4-3936d813a744`

value为：一个对象字典，包括 如下：

```
{\"status\": \"FAILURE\", \"result\": {\"exc_type\": \"NotRegistered\", \"exc_message\": [\"app.api_1_0.ToMaintain.task.adds\"], \"exc_module\": \"celery.exceptions\"}, \"traceback\": null, \"children\": [], \"date_done\": \"2020-11-05T02:35:18.402983\", \"task_id\": \"181002b5-bd28-4c1e-b4e4-3936d813a744\"}
```

##### 3.worker

默认启动4个worker进程

##### 4.获取任务结果和状态

```python
r = task.apply_async()
r.ready()     # 查看任务状态，返回布尔值,  任务执行完成, 返回 True, 否则返回 False.
r.wait()      # 会阻塞等待任务完成, 返回任务执行结果，很少使用；
r.get(timeout=1)       # 获取任务执行结果，可以设置等待时间，如果超时但任务未完成返回None；
r.result      # 任务执行结果，未完成返回None；
r.state       # PENDING, START, SUCCESS，任务当前的状态
r.status      # PENDING, START, SUCCESS，任务当前的状态
r.successful  # 任务成功返回true
r.traceback  # 如果任务抛出了一个异常，可以获取原始的回溯信息
```



## 使用示例

1.根据任务id获取任务执行的结果实例：

```python
res = AsyncResult(task_id)
```

2.使用示例：

一般都是一个接口推送了任务之后，过一段时间通过查询接口去查看任务执行状态或结果：

```python
# api1: 推送任务到broker消息对列
res = refresh_onboard_table.delay(add_time)   # AsyncResult实例
# print('res', res)  # 一个异步任务结果对象：AsyncResult实例
# print('task_id', res.id)  # 任务id
# print('status', res.status)  # 任务状态
# print('r', res.get())  # 任务返回值，或失败时的异常实例
# res.result # 任务返回值，或失败时的异常实例
# res.traceback  # 异常抛出时的堆栈回溯信息
# res.date_done  # 任务完成时间（仅当状态为成功和失败时有值）（和中国区时间对不上）
cache.set('task_id:{用户唯一id}:substitution_system:manual_sync_data', res.id, 60*60)

# api2：根据任务id获取它的任务执行结果（状态或者返回值）
state_map = {'PENDING': '等待执行(或状态未知)', 'STARTED': '正在执行', 'SUCCESS': '执行成功', 'FAILURE': '执行失败'}  # 记录STARTED状态 需要开启track_started=True
task_id = cache.get('task_id:{用户唯一id}:substitution_system:manual_sync_data')
res = AsyncResult(task_id)  # 根据任务id获取任务执行结果对象 AsyncResult实例
status = res.status  # 任务状态
# 状态结果可能不在map中（还有其它可能结果），直接取status
state_map.get(res.status, res.status) if res.status else ''
value = res.get()  # 任务返回值

```

- @celery_app.task 参数
  - track_started=True  记录任务开始状态STARTED，默认不记录



- res.get()      获取任务执行结果，包括失败异常信息

  在下面不同状态使用包含不通信息：

  - PENDING 状态： 会一直阻塞程序

  - STARTED  结果包含执行当前任务的工作单元的进程 ID 和主机名

  - SUCCESS  结果包含任务的返回值（res.result）

  - FAILURE   结果包含抛出的异常，以及异常抛出时的堆栈回溯信息

    propagate参数默认为True，抛出异常；设为False时，不会直接向外抛出异常信息

    ```python
    res.get(propagate=False)  # 不会直接向外抛出异常信息,会返回异常实例(res.result)
    res.traceback # 异常抛出时的堆栈回溯信息
    res.result  # 结果，失败时就是异常实例
    ```





# celery 常用命令

#### 启动worker

```
celery -A libs.djcelery worker -P eventlet

# flask使用时若报错：AttributeError: 'Flask' object has no attribute 'user_options'
# 原因：Celery会把叫做app或celery的对象认为是celery实例对象，所以不要把flask实例命名为app
# 解决：在后面加上 `:celery_app`(指定celery的app实例变量名)
celery -A libs.djcelery:celery_app worker -P eventlet

```

- -A：指定celery的app实例
- --loglevel=info：打印日志到终端

#### 启动定时任务推送beat

```
# 启动beat： 该进程到指定时间点会将任务推送到队列
celery -A libs.djcelery beat -l info

# flask中可能报错，改使用如下
celery -A libs.flask_celery:celery_app beat -l info
```

#### 启动flower --任务监控插件

```
# 先安装 pip install flower
celery -A PyVbord.libs.djcelery flower --port=5555 --broker=redis://10.43.246.55:6379/2
```





# celery常用配置介绍

**在django项目中使用时，都需要在配置名前加上 `CELERY_`的前缀**

设置时区
`CELERY_TIMEZONE = 'Asia/Shanghai'`
启动时区设置
`CELERY_ENABLE_UTC = True`
限制任务的执行频率
下面这个就是限制tasks模块下的add函数，每秒钟只能执行10次
`CELERY_ANNOTATIONS = {'tasks.add':{'rate_limit':'10/s'}}`
或者限制所有的任务的刷新频率
`CELERY_ANNOTATIONS = {'*':{'rate_limit':'10/s'}}`
也可以设置如果任务执行失败后调用的函数

```
def my_on_failure(self,exc,task_id,args,kwargs,einfo):
    print('task failed')

CELERY_ANNOTATIONS = {'*':{'on_failure':my_on_failure}}
```



并发的worker数量，也是命令行-c指定的数目
事实上并不是worker数量越多越好，保证任务不堆积，加上一些新增任务的预留就可以了
`CELERYD_CONCURRENCY = 20`

celery worker每次去redis取任务的数量，默认值就是4
`CELERYD_PREFETCH_MULTIPLIER = 4`

每个worker执行了多少次任务后就会死掉，建议数量大一些
`CELERYD_MAX_TASKS_PER_CHILD = 200`

使用redis作为任务队列
组成: <db+scheme://user:password@host:port/dbname>
`BROKER_URL = 'redis://127.0.0.1:6379/0'`

celery任务执行结果的超时时间
`CELERY_TASK_RESULT_EXPIRES = 1200`
单个任务的运行时间限制，否则会被杀死
`CELERYD_TASK_TIME_LIMIT = 60`

使用redis存储任务执行结果，默认不使用
`CELERY_RESULT_BACKEND = 'redis://127.0.0.1:6379/1'`

将任务结果使用'pickle'序列化成'json'格式
任务序列化方式
`CELERY_TASK_SERIALIZER = 'pickle'`
任务执行结果序列化方式
`CELERY_RESULT_SERIALIZER = 'json'`
也可以直接在Celery对象中设置序列化方式
`app = Celery('tasks', broker='...', task_serializer='yaml')`

关闭限速
`CELERY_DISABLE_RATE_LIMITS = True`

### 一份比较常用的配置文件:

在celery4.x以后，就是BROKER_URL，如果是以前，需要写成CELERY_BROKER_URL
`BROKER_URL = 'redis://127.0.0.1:6379/0'`
指定结果的接收地址
`CELERY_RESULT_BACKEND = 'redis://127.0.0.1:6379/1'`

指定任务序列化方式
`CELERY_TASK_SERIALIZER = 'msgpack'`
指定结果序列化方式
`CELERY_RESULT_SERIALIZER = 'msgpack'`
指定任务接受的序列化类型.
`CELERY_ACCEPT_CONTENT = ['msgpack']`

任务过期时间，celery任务执行结果的超时时间
`CELERY_TASK_RESULT_EXPIRES = 24 * 60 * 60`

任务发送完成是否需要确认，对性能会稍有影响
`CELERY_ACKS_LATE = True`

压缩方案选择，可以是zlib, bzip2，默认是发送没有压缩的数据
`CELERY_MESSAGE_COMPRESSION = 'zlib'`

规定完成任务的时间
在5s内完成任务，否则执行该任务的worker将被杀死，任务移交给父进程
`CELERYD_TASK_TIME_LIMIT = 5`

celery worker的并发数，默认是服务器的内核数目,也是命令行-c参数指定的数目
`CELERYD_CONCURRENCY = 4`

celery worker 每次去BROKER中预取任务的数量
`CELERYD_PREFETCH_MULTIPLIER = 4`

每个worker执行了多少任务就会死掉，默认是无限的
`CELERYD_MAX_TASKS_PER_CHILD = 40`

设置默认的队列名称，如果一个消息不符合其他的队列就会放在默认队列里面，如果什么都不设置的话，数据都会发送到默认的队列中
`CELERY_DEFAULT_QUEUE = "default"`
队列的详细设置

```
CELERY_QUEUES = {
    "default": { # 这是上面指定的默认队列
        "exchange": "default",
        "exchange_type": "direct",
        "routing_key": "default"
    },
    "topicqueue": { # 这是一个topic队列 凡是topictest开头的routing key都会被放到这个队列
        "routing_key": "topic.#",
        "exchange": "topic_exchange",
        "exchange_type": "topic",
    },
    "task_eeg": { # 设置扇形交换机
        "exchange": "tasks",
        "exchange_type": "fanout",
        "binding_key": "tasks",
    },
```

或者配置成下面两种方式:

```
# 配置队列（settings.py）
CELERY_QUEUES = (
    Queue('default', 
        Exchange('default'), 
        routing_key='default'),
    Queue('for_task_collect', 
        Exchange('for_task_collect'), 
        routing_key='for_task_collect'),
    Queue('for_task_compute', 
        Exchange('for_task_compute'), 
        routing_key='for_task_compute'),
)
# 路由（哪个任务放入哪个队列）
CELERY_ROUTES = {
    'umonitor.tasks.multiple_thread_metric_collector': 
    {
        'queue': 'for_task_collect', 
        'routing_key': 'for_task_collect'
    },
    'compute.tasks.multiple_thread_metric_aggregate': 
    {
        'queue': 'for_task_compute', 
        'routing_key': 'for_task_compute'
    },
    'compute.tasks.test': 
    {
         'queue': 'for_task_compute',
         'routing_key': 'for_task_compute'
    },
}
```



# 需要注意的问题

**1.提示任务未注册： NotRegistered**

在本地启动woker，使用delay发送任务到celery后，提示任务未注册NotRegistered，原因：

broker中的任务消息被线上服务器的woker获取到，而线上的任务函数没有使用app.task注册, 所以根据任务名映射查找任务函数时会找不到，提示NotRegistered。

