## 先决条件

本教程假定RabbitMQ [已](https://www.rabbitmq.com/download.html)在标准端口（5672）的本地主机上[安装](https://www.rabbitmq.com/download.html)并运行。如果您使用其他主机，端口或凭据，则连接设置需要进行调整

**(使用Pika Python客户端)**

```
pip install pika
```



## 核心概念



一个主机上不可能有两个相同名称的交换器或队列：

```
queue_declare 创建队列是幂等的，可以根据需要多次使用该命令， 但是同名队列只会创建一个。
exchange_declare 多次创建同名称交换器也只会创建一个
```



## 常用命令

```
1.查看RabbitMQ的队列以及队列中有多少条消息
# linux: 使用命令sudo rabbitmqctl；windows使用命令rabbitmqctl.bat
sudo rabbitmqctl list_queues
rabbitmqctl.bat list_queues

2.查看RabbitMQ的队列中 待消费的消息和未确认的消息
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged 

3.查看服务器上的所有交换器及类型
sudo rabbitmqctl list_exchanges

4.列出现有的绑定（交换器和队列的bind关系）
rabbitmqctl list_bindings
```



## 1.简单的示例: hello world

**生产者程序：`send.py`：**

```python
from pika import BlockingConnection, ConnectionParameters

# rabbitMQ连接参数 ，基本有默认值
params = ConnectionParameters(host='localhost',
    # port=_DEFAULT,
    # virtual_host=_DEFAULT,
    # credentials=_DEFAULT,
    # channel_max=_DEFAULT,
    # frame_max=_DEFAULT,
    # heartbeat=_DEFAULT,
    # ssl_options=_DEFAULT,
    # connection_attempts=_DEFAULT,
    # retry_delay=_DEFAULT,
    # socket_timeout=_DEFAULT,
    # stack_timeout=_DEFAULT,
    # locale=_DEFAULT,
    # blocked_connection_timeout=_DEFAULT,
    # client_properties=_DEFAULT,
    # tcp_options=_DEFAULT,
)

connection = BlockingConnection(parameters=params)  # 创建连接
channel = connection.channel()   # 创建信道

# 在发送之前， 我们需要确保收件人队列存在， 如果将消息发送到不存在的位置，RabbitMQ只会删除该消息
# 创建一个hello队列
channel.queue_declare(queue='hello')

# 开始发送消息到交换器，根据路由键路由到指定队列 (在RabbitMQ中， 永远无法将消息直接发送到队列， 它始终需要进行交换)
msg = 'hello world 你好'
# exchange='' 空串代表默认消息交换器，routing_key 指定路由的队列
channel.basic_publish(exchange='', routing_key='hello', body=msg) # 发布
print(f'已发送: {msg}!')

# 通过关闭连接来刷新网络缓存区:
# 在退出程序之前，我们需要确保网络缓冲区被刷新，且我们的消息被实际传送到RabbitMQ。我们可以通过轻轻关闭连接来完成。
connection.close()
```

**消费者程序: `receive.py`:**

```python
from pika import BlockingConnection, ConnectionParameters

# 同样需要先连接RabbitMQ服务器,连接代码一样

# RabbitMQ连接参数，基本有默认值
parameters = ConnectionParameters()
connection = BlockingConnection(parameters=parameters)

channel = connection.channel()

# 像send.py一样，需要确保队列是否存在， 使用queue_declare创建队列是幂等的， 可以根据需要多次使用该命令， 但是只会创建一个。
channel.queue_declare(queue='hello')
# 说明: 在send.py中已经声明了该队列且已经运行，这里其实可以不用声明，但是考虑到不确定哪个程序先执行, 所以最好在两个程序中重复声明。
# 查看消息队列中有多少消息，linux下可以通过:rabbitmqctl list_queues命令查看，windows通过rabbitmqctl.bat list_queues查看。如果安装了web控制台，登录之后可以切换到Queues标签下查看。

# 定义回调函数作为消息处理函数; 将回调函数订阅到队列，每当接收到消息时，都会调用此回调函数
def callback(ch, method, properties, body):
    """回调函数：消息处理器"""
    print(f"收到消息：{body.decode()}")

# 指定回调函数从哪个队列接收消息
channel.basic_consume(queue='hello', auto_ack=True,on_message_callback=callback)

# 最后进入一个无限循环， 该循环等待数据并在必要的时候运行回调。
print('等待接收消息, 按CTRL+C退出....')
channel.start_consuming()

```



## 2.工作队列 Work queues

稍微修改上面的示例：

**生产者程序：`new_task.py`:**

```python
# TODO: 通过命令行运行该脚本，并传递参数
import sys
from pika import BlockingConnection, ConnectionParameters


# rabbitMQ连接参数 ，基本有默认值
params = ConnectionParameters()

connection = BlockingConnection(parameters=params)  # 创建连接
channel = connection.channel()   # 创建信道

# 在发送之前， 我们需要确保收件人队列存在， 如果将消息发送到不存在的位置，RabbitMQ只会删除该消息
# 创建一个hello队列
channel.queue_declare(queue='hello')

# 开始发送消息到交换器，根据路由键路由到指定队列 (在RabbitMQ中， 永远无法将消息直接发送到队列， 它始终需要进行交换)
msg = ''.join(sys.argv[1:]) or 'Hello World!'
# exchange='' 空串代表默认消息交换器，routing_key 指定路由的队列
channel.basic_publish(exchange='', routing_key='hello', body=msg) # 发布
print(f'已发送: {msg}!')

# 通过关闭连接来刷新网络缓存区:
connection.close()

```

**消费者程序：`worker.py`:**

```python
# TODO: 当开启多个消费者时：
# TODO: 默认情况下，RabbitMQ将每个消息按顺序发送给下一个使用者。平均而言，每个消费者都会收到相同数量的消息。这种分发消息的方式称为循环


# 同样需要先连接RabbitMQ服务器,连接代码一样
import time

from pika import BlockingConnection, ConnectionParameters

# RabbitMQ连接参数，基本有默认值
parameters = ConnectionParameters()
connection = BlockingConnection(parameters=parameters)

channel = connection.channel()

# 像send.py一样，需要确保队列是否存在， 使用queue_declare创建队列是幂等的， 可以根据需要多次使用该命令， 但是只会创建一个。
channel.queue_declare(queue='hello')
# 说明: 在send.py中已经声明了该队列且已经运行，这里其实可以不用声明，但是考虑到不确定哪个程序先执行, 所以最好在两个程序中重复声明。
# 查看消息队列中有多少消息，linux下可以通过:rabbitmqctl list_queues命令查看，windows通过rabbitmqctl.bat list_queues查看。如果安装了web控制台，登录之后可以切换到Queues标签下查看。

# 定义回调函数作为消息处理函数; 将回调函数订阅到队列，每当接收到消息时，都会调用此回调函数
def callback(ch, method, properties, body):
    """回调函数：消息处理器"""
    print(f"收到消息：{body.decode()},正在处理...")
    time.sleep(body.count(b'.'))
    print(f"完成,耗{body.count(b'.')}")
    # 完成任务后，手动确认消息，并让rabbitmq删除该消息标记； 如果未确认就挂掉，消息将重新排列到队列
    ch.basic_ack(delivery_tag=method.delivery_tag) # 绝不能漏，否则rabbitmq消耗内存会越来越多

# auto_ack: 完成任务后，需手动消息确认(默认False 打开， True 关闭)，然后rabbitmq会从队列删除该消息的标记
# channel.basic_consume(queue='hello', auto_ack=True,on_message_callback=callback) # 若出错和挂掉，则丢失分配给该消费者的所有消息
channel.basic_consume(queue='hello',on_message_callback=callback) #  默认开启手动确认

# 最后进入一个无限循环， 该循环等待数据并在必要的时候运行回调。
print('等待接收消息, 按CTRL+C退出....')
channel.start_consuming()

```

### 循环调度

```
默认情况下，RabbitMQ将每个消息按顺序发送给下一个使用者。平均而言，**每个消费者都会收到相同数量的消息**。这种分发消息的方式称为循环。
```

**注意：默认情况，rabbitmq会将队列中的所有消息，一次性平均分派给订阅它的多个消费者，且不论它是否已完成正在执行的任务，都会接收到多个消息**



### 消息确认

```
如果其中一个使用者开始一项漫长的任务，而仅部分完成而死掉，会发生什么情况。使用我们当前的代码，RabbitMQ一旦将消息传递给使用者，便立即将其标记为删除。在这种情况下，如果您杀死一个工人，我们将丢失正在处理的消息。我们还将丢失所有发送给该特定工作人员但尚未处理的消息。

为了确保消息永不丢失，RabbitMQ支持 [消息*确认*](https://www.rabbitmq.com/confirms.html)。消费者发送回一个确认（acknowledgement）以告知RabbitMQ已经接收，处理了特定的消息，并且RabbitMQ可以自由删除它。

如果使用者在**不发送确认的情况下死亡（其通道已关闭，连接已关闭或TCP连接丢失）**，RabbitMQ将了解消息未得到充分处理，并**将重新排队**。如果同时有其他消费者在线，它将很快**将其重新分发给另一个消费者**。这样，您可以确保即使工人偶尔死亡也不会丢失任何消息。
```

```python
def callback(ch, method, properties, body):
    print(f"收到消息：{body.decode()},正在处理...")
    time.sleep(body.count(b'.'))
    print(f"完成,耗{body.count(b'.')}")
    # 完成任务后，手动确认消息，并让rabbitmq删除该消息标记； 如果未确认就挂掉，消息将重新排列到队列
    ch.basic_ack(delivery_tag=method.delivery_tag) # 绝不能漏，否则rabbitmq消耗内存会越来越多

# auto_ack: 完成任务后，需手动消息确认(默认False 打开， True 关闭)，然后rabbitmq会从队列删除该消息的标记
channel.basic_consume(queue='hello',on_message_callback=callback) #  默认开启手动确认
```



### 讯息持久性

```
如果RabbitMQ服务器停止，我们的任务仍然会丢失；要确保重启后消息不会丢失，需要做两件事：**我们需要将队列和消息都标记为持久性。**
```

**a.将队列声明为持久性：**

```python
channel.queue_declare(queue='hello', durable=True)
```

```
尽管此命令本身是正确的，但在我们的设置中将不起作用。那是因为我们已经定义了一个叫hello的队列 ，它并不持久。RabbitMQ不允许您使用不同的参数重新定义现有队列，并且将向尝试执行此操作的任何程序返回错误。但是有一个快速的解决方法-让我们声明一个名称不同的队列，例如task_queue：
```

```python
channel.queue_declare(queue='task_queue', durable=True)
```

**此`queue_declare`更改需要同时应用于生产者代码和使用者代码。**此时即使RabbitMQ重新启动，task_queue队列也不会丢失。

**b.将消息标记为持久性**——通过提供值为 2 的`delivery_mode`属性。

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
```



[^有关消息持久性的说明]: 将消息标记为持久性并不能完全保证不会丢失消息。尽管它告诉RabbitMQ将消息保存到磁盘，但是RabbitMQ接受消息但尚未将其保存时，仍有很短的时间。另外，RabbitMQ不会对每条消息都执行fsync（2）－它可能只是保存到缓存中，而没有真正写入磁盘。持久性保证并不强，但是对于我们的简单任务队列而言，这已经绰绰有余了。如果您需要更强有力的保证，则可以使用 [发布者确认](https://www.rabbitmq.com/confirms.html)。



### 公平派遣

```
当部分消息执行时间特别长时，而有一些又特别短时，一位工人将一直忙碌而另一位工人将几乎不做任何工作。RabbitMQ对此一无所知，并且仍将平均分配消息。

发生这种情况是因为RabbitMQ在消息进入队列时才调度消息。它不会查看使用者的未确认消息数。它只是盲目地将每第n条消息发送给第n个使用者。

为了解决这个问题，可以**将`Channel＃basic_qos`通道方法与 `prefetch_count = 1`设置一起使用**。这使用`basic.qos`协议方法来**告诉RabbitMQ一次不向工作人员发送多条消息**。换句话说，**在处理并确认上一条消息之前，不要将新消息发送给工作人员。而是将其分派给不忙的下一个工作程序。**
```

```python
channel.basic_qos（prefetch_count = 1）
```



[^关于队列大小的注意事项]: v如果所有工作人员都忙，您的队列就满了。您将需要注意这一点，并可能增加更多的工作人员，或使用[消息TTL](https://www.rabbitmq.com/ttl.html)。



### 将代码整合到一起

**`new_task.py` 代码：**

```python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

# 声明持久化队列
channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=2,  # 持久化消息
    ))
print(" [x] Sent %r" % message)
connection.close()
```

**`worker.py`代码**

```python
import pika
import time

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print(' [*] Waiting for messages. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag=method.delivery_tag)

# 在处理并确认上一条消息之前，不要将新消息发送给工作人员。而是将其分派给不忙的下一个工作程序。
channel.basic_qos(prefetch_count=1)

channel.basic_consume(queue='task_queue', on_message_callback=callback)

channel.start_consuming()
```



**使用`消息确认`和`prefetch_count`，您可以设置工作队列。`持久性`选项即使重新启动RabbitMQ也可以使任务继续存在。**



## 3.发布/订阅 Publish/Subscribe

定义：以广播的形式一次向多个消费者发送消息

```
工作队列背后的假设是，每个任务都恰好交付给一个工人。在这一部分中，我们将一个消息传达给多个消费者；本质上，已发布的日志消息将广播到所有接收者；订阅了指定队列的消费者都可以收到，这种模式称为“发布/订阅”。

我们将构建一个简单的日志记录系统。它由两个程序组成-第一个程序将发出日志消息，第二个程序将接收并打印它们。接收器程序运行的每个副本都将获得消息。

RabbitMQ消息传递模型的核心思想是**生产者从不将任何消息直接发送到队列**，相反，**生产者只能将消息发送到交换器**，一方面，它接收来自生产者的消息，另一方面，将它们推入队列。**交换器必须确切知道如何处理收到的消息。是否应将其附加到特定队列？是否应该将其附加到许多队列中？还是应该丢弃它。规则由*交换类型*定义** 。

有几种交换类型可用：**direct，topic，headers 和fanout**。我们将集中讨论最后一个-扇出。
```

```python
# 创建一个名为logs的交换机，类型为扇出fanout
channel.exchange_declare(exchange='logs', exchange_type='fanout')
```

### fanout 交换器

- **fanout(扇出)：将接收到的所有消息广播到它知道的所有队列中**

之前发布消息：

```python
channel.basic_publish(exchange='',routing_key='hello',body=message)
```

现在发布到我们创建的交换器logs：

```
channel.basic_publish(exchange='logs',routing_key='',body=message)
```

**注意：默认的消息交换器，发送时需提供routing_key,但是对于fanout交换器，它的值将被忽略。**

### 临时队列

##### a.多个消费者之间共享队列

```
之前当我们想在生产者和多个消费者之间共享队列时，必须给队列自命名，让其指向同一个队列；(即：当多次创建名称相同的队列时是无效的，都将使用一个队列)
```

##### b.每个消费者使用单独的队列

```
现在需要每个消费者程序都获取所有的消息，就不能使用同一个队列取值，而是每个消费者都将有单独的队列获取消息

无论何时连接到Rabbit，都需要一个全新的空队列。为此，可以创建一个具有随机名称的队列，或者甚至更好-让服务器为我们选择一个随机队列名称。可以通过`queue_declare`提供空的`queue`参数来做到这一点：
```

```python
result = channel.queue_declare(queue = '')
print(result.method.queue) # 打印随机命名队列的名称，看起来像amq.gen-JzTY20BRgKO-HjmUJj0wLg。
```

一旦消费者连接关闭，则应删除队列，可以创建队列时使用`exclusive`参数

```python
result = channel.queue_declare(queue='', exclusive=True)
```

### 绑定

定义：现在需要告诉交换机将消息发布到我们的队列。**交换器和队列之间的关系称为*绑定***

```python
# 将交换机绑定到服务器随机创建的队列
channel.queue_bind(exchange='logs',queue=result.method.queue)
```



### 代码成合到一起

![img](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)



实际上，本次的程序和之前的区别就在于：**生产者将消息发布到指定的`logs`交换器，然后由交换器将消息推送到和它绑定的所有队列中，每个消费者都能从一个队列获取所有消息**

**默认状态下（即自命名队列，例如hello），所有消费者都订阅到同一个队列 hello；现在因为每次连接到rabbitmq服务器，随机生成队列名，所以每次新连接上的消费者都使用单独的队列**

**`emit_log.py` 生产者程序：**

```
import sys
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

# 创建一个名为logs的交换机，类型为扇出fanout
channel.exchange_declare(exchange='logs', exchange_type='fanout')

# queue='':随机创建一个队列，exclusive=True: 连接关闭时,自动删除该队列
# 所以生产者程序无需创建队列，创建了连接关闭时也会被删除
# result = channel.queue_declare(queue='', exclusive=True)

msg = ''.join(sys.argv[1:]) or 'Hello World!'
# 默认的消息交换器，发送时需提供routing_key,但是对于fanout交换器，它的值将被忽略。
channel.basic_publish(exchange='logs', routing_key='',body=msg)
print(f'已发送: {msg}!')

connection.close()
```

建立连接后，我们声明了交换器。由于禁止发布到不存在的交换器，因此此步骤是必需的

如果没有队列绑定到交换机，则消息将丢失，但这对我们来说是可以的。如果没有消费者在听，我们可以放心地丢弃该消息。

**`receive_log.py` 消费者程序：**

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

# 像生产者程序一样创建一个名为logs的交换机，类型为扇出fanout
channel.exchange_declare(exchange='logs', exchange_type='fanout')

# queue='' 创建随机名称的队列，exclusive=True:关闭连接自动删除
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue  # 获取队列名
print(f'随机队列名:{queue_name}')

# 将交换机绑定到服务器随机创建的队列
channel.queue_bind(exchange='logs',queue=queue_name)

print('等待日志打印')

def callback(ch, method, properties, body):
    print(f"收到消息：{body.decode()}")

channel.basic_consume(queue=queue_name,on_message_callback=callback,auto_ack=True)

channel.start_consuming()
```

执行脚本：

```
python receive_logs.py > logs_from_rabbit.log
python receive_logs.py
Python emit_log.py

```

最终的结果就是:多个消费者都执行了相同的消息任务， 运行的两个消费者程序都将打印一样消息，第一个会输出所有的消息到文件



## 4.路由 Routing

有选择的接收消息

```
我们将向之前代码添加功能-使仅订阅消息的子集成为可能。例如，我们将只能将严重错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

```

### 绑定

绑定是交换和队列之间的关系。可以简单地理解为：队列对来自此交换的消息感兴趣。

```
绑定可以使用额外的`routing_key`参数。为了避免与`basic_publish`的参数`路由键`混淆，我们将其称为 `绑定键`。这是我们如何使用键创建绑定的方法：

```

```python
# 此处的 routing_key 为绑定键，
channel.queue_bind(exchange=exchange_name,queue=queue_name,routing_key='black')
```

**绑定键的含义取决于交换器类型。我们之前使用的`fanout`交换器只是忽略了它的值。**

### direct 交换器

之前使用的是`fanout`交换器，它并没有给我们太大的灵活性-只能进行无意识的广播。

- **direct: 消息进入其 `绑定键` 与消息的 `路由键` 完全匹配的队列**

思考一下设置：

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

可以看到绑定了两个队列的`direct交换器X`。第一个队列由绑定键orange绑定，第二个队列有两个绑定，一个绑定键为black，另一个绑定为green。

这样的设置中，**使用`路由键orange`发布到交换机的消息 将被路由到队列Q1。`路由键black或green`的消息将转到Q2。所有其他消息将被丢弃。**



### 多重绑定

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

**用相同的绑定键绑定多个队列是完全合法的**。示例中，可以使用绑定键black在X和Q1及Q2之间添加绑定。**在这种情况下，直接交换的行为类似于fanout扇出**，并将消息广播到所有匹配的队列。带有black路由键的消息将同时传递到 Q1和Q2。



### 在日志系统使用此模型

发送者：

将发送消息到direct交换机。使用日志严重性作为路由键。这样，接收脚本将能够选择其想要接收的严重性消息

```python
# 创建日志交换器，类型为direct
channel.exchange_declare(exchange='direct_logs',exchange_type='direct')
# 发送消息，routing_key='严重性'：严重性具体值可以是“信息”，“警告”，“错误”之一
channel.basic_publish(exchange='direct_logs',routing_key='严重性',body=message)
                         
```

接收者：

```python
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue
# severities：[“信息”，“警告”，“错误”]
for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity) # 指定路由键
```



### 代码整合到一起

![img](https://www.rabbitmq.com/img/tutorials/python-four.png)

**`emit_log_direct.py`生产者程序：**

```python
import sys
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

# 创建一个名为logs_direct的交换机，类型为direct
channel.exchange_declare(exchange='logs_direct', exchange_type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
msg = ''.join(sys.argv[2:]) or 'Hello World!'
# 默认的消息交换器，发送时需提供routing_key,但是对于fanout交换器，它的值将被忽略。
channel.basic_publish(exchange='logs_direct', routing_key=severity,body=msg)

print(" [x] Sent %r:%r" % (severity, msg))

connection.close()
```

**`receive_logs_direct.py`消费者程序：**

```python
import sys
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

# # 创建一个名为logs_direct的交换机，类型为direct
channel.exchange_declare(exchange='logs_direct', exchange_type='direct')

# queue='' 创建随机名称的队列，exclusive=True:关闭连接自动删除
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue  # 获取队列名
# print(f'随机队列名:{queue_name}')

severities = sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

# 将交换机绑定到服务器随机创建的队列,设置绑定键
for severity in severities:
    channel.queue_bind(exchange='logs_direct',queue=queue_name,routing_key=severity)

# 发布的消息将进入路由键与交换器绑定键完全匹配的队列

print('等待日志打印')

def callback(ch, method, properties, body):
    print(f"收到消息：{method.routing_key}, {body.decode()}")

channel.basic_consume(queue=queue_name,on_message_callback=callback,auto_ack=True)

channel.start_consuming()
```

执行脚本：

```
只想将“警告”和“错误”（而不是“信息”）日志消息保存到文件中
python receive_log_direct.py warning error > logs_from_rabbit.log

想在屏幕上查看所有日志消息
python receive_logs_direct.py info warning error

发送不同严重级别的消息，例如 ：
error级别：
python emit_log_direct.py error  # 结果是：屏幕打印和文件中都出现该消息
info级别：
python emit_log_direct.py info  # 结果是：只有屏幕打印该消息


```



## 5.主题 topics

direct 交换器 仍然存在局限性-它不能基于多个条件进行路由

日志记录系统中，我们可能不仅要根据严重性订阅日志，还要根据发出日志的源订阅日志。

### topic 交换器

- **topic：逻辑类似于direct交换的逻辑**
  - topic交换的消息必须是单词列表，以点分隔。这些词可以是任何东西，但通常它们指定与消息相关的某些特性，一些有效路由键示例：`quick.orange.rabbit`
  - 路由键中可以包含任意多个单词，最多255个字节。
  - 绑定键也必须采用相同的形式--使用特定路由键发送的消息将传递到使用匹配绑定键绑定的所有队列
  - 绑定键有两个重要的特殊情况
    - *（星号）可以代替一个单词
    - ＃（井号）可以替代零个或多个单词。



![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

此示例，将发送所有描述动物的消息。使用包含三个词（两个点）的路由键发送消息。路由密钥中的第一个单词将描述一个速度，第二个描述颜色，第三个描述物种：“ <celerity>.<colour>.<species> ”。

示例创建了三个绑定：Q1与绑定键`*.orange.*`  绑定，Q2与`*.*.rabbit `和`lazy.＃` 绑定。

这些绑定可以总结为：

- Q1对所有橙色动物都感兴趣。
- Q2想接收有关兔子的一切，以及有关慢速动物的一切。

路由键设置为“ quick.orange.rabbit ”的消息将传递到两个队列。消息“ lazy.orange.elephant ”也将发送给他们两个。另一方面，“ quick.orange.fox ”只会进入第一个队列，而“ lazy.brown.fox ”只会进入第二个队列。“ lazy.pink.rabbit ”将被传递到第二队只有一次，即使两个绑定匹配。“ quick.brown.fox ”与任何绑定都不匹配，因此将被丢弃。

如果我们违反规则并发送一个或四个单词的消息，例如“ 橙色 ”或“ quick.orange.male.rabbit ”，这些消息将不匹配任何绑定，并且将会丢失。

另一方面，“ lazy.orange.male.rabbit ”即使有四个单词，也将匹配最后一个绑定，并将其传送到第二个队列。

```
topic交换器功能强大，可以像其他交换器一样进行。

当队列用“ ＃ ”绑定键绑定时，它将接收所有消息，而与路由键无关，就像在扇出fanout交换中一样。

当在绑定中不使用特殊字符“ * ”和“ ＃ ”时，主题交换的行为就像直接direct交换器的一样。
```



### 代码整合到一起

`emit_log_topic.py`

```python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(
    exchange='topic_logs', routing_key=routing_key, body=message)
print(" [x] Sent %r:%r" % (routing_key, message))
connection.close()
```

`receive_logs_topic.py`

```python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

result = channel.queue_declare('', exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(
        exchange='topic_logs', queue=queue_name, routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
   print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(
   queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

执行脚本：

```cmd
要接收所有日志，请运行：
python receive_logs_topic.py "#"
接收“ kern ” 发出的所有日志：
python receive_logs_topic.py "kern.*"
只想接收“ critical ”日志：
python receive_logs_topic.py "*.critical"
创建多个绑定：
python receive_logs_topic.py "kern.*" "*.critical"
发出带有路由键“ kern.critical ” 的日志类型：
python emit_log_topic.py "kern.critical" "A critical kernel error"
```







































