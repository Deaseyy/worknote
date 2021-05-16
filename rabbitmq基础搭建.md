# windows下rabbitmq命令管理

- 本地rabbitmq关闭了自启动，开机需要手动启动

```
rabbitmq启动方式有2种

1、以应用方式启动
rabbitmq-server -detached 后台启动

rabbitmq-server 直接启动，如果你关闭窗口或者需要在改窗口使用其他命令时应用就会停止

关闭:rabbitmqctl stop

// 试了下以下命令 没啥效果
2、以服务方式启动（安装完之后在任务管理器中服务一栏能看到RabbtiMq）
rabbitmq-service install 安装服务

rabbitmq-service start 开始服务

Rabbitmq-service stop  停止服务

Rabbitmq-service enable 使服务有效

Rabbitmq-service disable 使服务无效

rabbitmq-service help 帮助

当rabbitmq-service install之后默认服务是enable的，如果这时设置服务为disable的话，rabbitmq-service start就会报错。

当rabbitmq-service start正常启动服务之后，使用disable是没有效果的

  关闭:rabbitmqctl stop

3、Rabbitmq 管理插件启动，可视化界面

rabbitmq-plugins enable rabbitmq_management 启动

rabbitmq-plugins disable rabbitmq_management 关闭
```



Rabbitmq节点管理方式

```
Rabbitmqctl
```



## 远程访问其他服务器的rabbitmq

#### 1.先添加用户并分配角色

```
rabbitmqctl add_user lisi lisi123  // 设置用户名和密码
rabbitmqctl set_user_tags name administrator
```

#### 2.用户授权

```
rabbitmqctl set_permissions -p / lisi ".*" ".*" ".*"
```

不然会引发异常：pika.exceptions.ProbaleAccessDeniedError



## 相关操作命令

#### 用户操作

```
1. 查看用户列表：rabbitmqctl list_users

2. 新增一个用户：rabbitmqctl add_user bruce 123456

3. 删除一个用户：rabbitmqctl delete_user bruce

4.修改用户密码：rabbitmqctl change_password bruce 654321

5.授予管理员角色：rabbitmqctl set_user_tags bruce administrator

6.授予用户权限：rabbitmqctl set_permissions -p / bruce ".*" ".*" ".*"
```

#### mq操作

```
rabbitmqctl list_queues  // 查看队列中的消息

```





# 典型应用场景

### 1.异步处理

> 场景说明：用户注册后，需要发注册邮件和注册短信，传统的方法有两种 串行和并行

- **串行方式**：将注册信息写入数据库后，发送注册邮件，再发送注册短信，以上三个任务全部完成后才返回给客户端。这有一个问题是：邮件，短信不是必须的，只是一个通知，这会让客户端等待没有必要等待的东西。

  ![1598109644113](C:\Users\12395\AppData\Local\Temp\1598109644113.png)



- **并行方式**：将注册信息写入数据库后 ，在发送邮件的同时发送短信，以上三个任务完成后，返回给客户端，并行的方式能缩短处理的时间。

  ![1598109738467](C:\Users\12395\AppData\Local\Temp\1598109738467.png)

- **消息队列**：假设三个业务节点分别使用50ms，则串行方式使用150ms，并行使用100ms。虽然并行已经缩短了处理的时间，但是前面说过，邮件和短信对我正常使用网站没有任何影响，客户端不必要等待其发送完成才显示注册成功，应该是写入数据库后就返回。引入消息队列后，把不是必须的业务逻辑异步处理。

  ![1598109978869](C:\Users\12395\AppData\Local\Temp\1598109978869.png)

由此可以看出，引入消息队列后，用户的响应时间就等于写入数据库的时间 + 写入消息队列的时间(可忽略)。

这里应该使用mq的广播模型，可以被多个消费者订阅，不同消费者实现各自的业务。



### 2.应用解耦

> 场景: 双11是购物狂欢节，用户下单后，订单系统需要通知库存系统，传统的做法就是订单系统调用库存系统的接口。

这种做法有个缺点：

当库存系统出现故障时，订单就会失败。订单系统和库存系统高耦合。

引入消息队列：

- 订单系统：用户下单后，订单系统完成持久化处理，再将消息写入消息队列，返回用户下单成功。
- 库存系统：订阅下单的消息，进行库操作，就算库存系统出现故障，消息队列也能保证消息的可靠投递，不会导致消息丢失。



### 3.流量削峰

> 场景：秒杀活动，一般会因为瞬间流量过大，导致应用挂掉，为了解决该问题，一般在业务模块前加入消息队列。

**作用：**

- 1.可以控制秒杀活动人数，超过此一定阈值的订单直接丢弃

- 2.可以缓解短时间的高流量压垮应用（应用能按自己最大处理能力获取订单处理）

![1598111091673](C:\Users\12395\AppData\Local\Temp\1598111091673.png)

**过程：**

1.用户的请求服务器收到之后，首先写入消息队列，加入消息队列的消息长度超过最大值，则直接抛弃用户请求或者跳转到错误页面。

2.秒杀业务根据消息队列中的请求信息，再做后续逻辑处理



# rabbitmq的集群

## 1.集群架构

### 1.1普通集群（副本集群）

架构图（主备结构）：

![1598144805464](C:\Users\12395\AppData\Local\Temp\1598144805464.png)

默认情况下：rabbitmq代理操作所需的所有数据/状态都将跨所有节点复制，消息队列quene例外，默认情况下消息队列位于一个节点上，尽管它们可以从所有节点服务看到和访问(访问是有限制的)

通俗来说就是，只有主节点存储了队列信息，slave从节点无法同步消息队列quene，意味着当**主节点宕机后，slave从节点也无法切换为主节点继续提供服务**，只有等到主节点服务恢复后，然后从slave节点读取原始数据恢复。虽然不能复制队列，但是从节点是可以看到主节点上的的队列信息的

**核心解决问题**：<font color="orange">当集群中某一时候主节点宕机后，可以对Qunene中源信息进行备份，但是无法备份队列消息</font>

slave节点可以减轻主节点上来自消费者的并发压力

**主要缺点：**<font color="orange">当主节点宕机后，slave从节点无法切换为主节点继续提供服务，不能故障转移；如果消息没有做持久化，那么也会丢失消息</font>





# rabbitmq集群搭建

### 1.使用docker启动mq

首先拉取镜像：rabbitmq:management  镜像id：771d109b8bde

创建容器并运行（15672是管理界面的端口，5672是服务的端口。可将管理系统的用户名和密码设置为admin admin），例如：

```
docker run -dit --name Myrabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 rabbitmq:managemen
```

下面开始运行集群的rabbitmq节点(这里使用三个节点)：

node1:  mq1  master 主节点
node2:  mq2  slave1 副本节点
node3:  mq3  slave2 副本节点

启动mq1：

```
docker run -id --name mq1 -p 15600:15672 -p 5600:5672 --hostname mq1_host -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' 771d109b8bde
```

启动mq2：

```
docker run -id --name mq2 -p 15601:15672 -p 5601:5672 --hostname mq2_host -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' --link mq1:mq1_host 771d109b8bde
```

启动mq3：

```
docker run -id --name mq3 -p 15602:15672 -p 5602:5672 --hostname mq3_host -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' --link mq1:mq1_host --link mq2:mq2_host 771d109b8bde
```

参数解释：

- -p ：宿主机端口映射容器端口，分别为web插件端口，rbbitmq服务端口
- --hostname：主机名称，也是rabbitmq节点名称
- -e：设置rabbitmq的cookie，一个集群中的rabbitmq节点必须使用相同cookie文件；
  - rabbitmq服务首次启动时,都会生成一个cookie，在`/var/lib/rabbitmq/.erlang.cookie`文件中
  - rabbitmq是用Erlang实现的，Erlang Cookie 相当于不同节点之间通讯的密钥，Erlang节点通过交换 Erlang Cookie 获得认证

- --link：容器之间进行相互连接



### 2.检查各节点的cookie是否一致

```
node1: cat /var/lib/rabbitmq/.erlang.cookie 
node2: cat /var/lib/rabbitmq/.erlang.cookie 
node3: cat /var/lib/rabbitmq/.erlang.cookie
```

### 3.加入节点到集群

在node2和node3执行加入集群命令：

```
docker exec -it mq1 /bin/bash     // 进入容器
rabbitmqctl stop_app        // 关闭应用（关闭当前启动的节点）
// 非必须
rabbitmqctl reset        // 从管理数据库中移除所有数据，如配置过的用户和虚拟宿主, 删除所有持久化的消息（该命令要在rabbitmqctl stop_app之后使用）
rabbitmqctl join_cluster rabbit@mq1_host   // 加入集群，成功会返回：Clustering node rabbit@mq2_host with rabbit@mq1_host
rabbitmqctl start_app    // 启动应用
```

注意：必须rabbit@开头，后面接主机名(不能是ip)，所以前面的主机名和ip映射很重要

**查看集群状态,任意节点执行：**

```
rabbitmqctl cluster_status
```

**集群测试：**

1.在node1上创建队列hello

2.查看node2和node3节点

2.关闭node1节点,执行命令 rabbitmqctl stop_app,查看node2和node3: 

从结果看到：普通集群当主节点宕机后，从节点状态state变为down，无法继续提供服务



**集群搭建的关键**：

- ip和主机名的映射

- 各节点的cookie需保证一致



有将来的提示：RABBITMQ_ERLANG_COOKIE 会被移除

```
RABBITMQ_ERLANG_COOKIE env variable support is deprecated and will be REMOVED in a future version. Use the $HOME/.erlang.cookie file or the --erlang-cookie switch instead
```



# 镜像集群

### 1.策略说明

**策略，简单来说就是规则：**

RabbitMQ镜像功能，需要基于RabbitMQ策略来实现，策略policy是用来控制和修改群集范围的某个vhost的队列行为和Exchange行为。设置哪些Exchange或者queue的数据需要复制、同步，以及同步的规则

<mark>普通集群下，从节点无法复制队列数据，虽然可以从slave节点看到queue；</mark>

**现在对队列做镜像，需要添加一个策略：**

```
rabbitmqctl set_policy [-p <vhost>] [--priority <priority>] [--apply-to <apply-to>] <name> <pattern>  <definition>
```

主要参数:

- -p  vhost ： 可选，针对指定vhost下的queue进行镜像
- name：策略名称(自定义)
- pattern：queue的匹配模式(正则)，对匹配到的queue进行镜像
- definition：镜像定义，主要包括三部分：ha-mode，ha-params，ha-sync-mode
  - ha-mode：指明镜像队列的模式，有效值为 all/exactly/nodes
    - all：表示在集群中所有的节点上进行镜像
    - exactly：表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
    - nodes：表示在指定的节点上进行镜像，节点名称通过ha-params指定
  - ha-params：ha-mode模式需要用到的参数
  - ha-sync-mode：进行队列中消息的同步方式，有效值为automatic和manual
  - priority：可选参数，policy的优先级（当有多个策略作用时，该策略的优先级）



### 2.查看当前策略

```
rabbitmqctl list_policies
```



### 3.添加策略

现添加一个策略：

- 在集群中所有的节点上进行镜像，  ha-mode: all

- 只对 hello 开头的队列做镜像， '^hello'
- 队列中消息的同步方式为自动，ha-sync-mode: automatic

```
rabbitmqctl set_policy ha-all '^hello' '{"ha-mode":"all","ha-sync-mode":"automatic"}' 
// 说明:策略正则表达式为 “^” 表示匹配所有队列名称  “^hello”:匹配hello开头队列
```

我在阿里云服务器上mq集群添加的策略：

```
 rabbitmqctl set_policy ha-all '^' '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```



### 4.删除策略

```
rabbitmqctl clear_policy ha-all
```

