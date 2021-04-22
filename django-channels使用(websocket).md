# 介绍

#### Channels

Django本身不支持WebSocket，但可以通过集成Channels框架来实现WebSocket

Channels是针对Django项目的一个增强框架，可以使Django不仅支持HTTP协议，还能支持WebSocket，MQTT等多种协议，同时Channels还整合了Django的auth以及session系统方便进行用户管理及认证。

- 安装相关库

```
pip install channels
pip install channels-redis  # 支持channel layers
```



# 后端

## 聊天室功能

### 1.修改`settings.py`文件

```python
# APPS中添加channels
INSTALLED_APPS = [
    'channels',
]

WSGI_APPLICATION = 'mychat.wsgi.application'  # django默认使用的wsgi
# 增加ASGI的支持
ASGI_APPLICATION = "mychat.routing.application"

# Channel相关配置
# 使用channels_layers服务
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [os.environ.get('REDIS_URL', 'redis://localhost:6379')],
        },
        # "ROUTING": "chat.routing.channel_routing", # 告诉 Channel去哪里找通道路由
    },
}
# 跨域等必要其他配置
```

```
channels运行于ASGI协议上，ASGI的全名是Asynchronous Server Gateway Interface。它是区别于Django使用的WSGI协议 的一种异步服务网关接口协议，正是因为它才实现了websocket.
```



### 2.setting.py的同级目录下增加routing.py

**ASGI_APPLICATION** 指定为routing.py文件中的application，表示由django使用的WSGI协议转换Channels使用的ASGI协议

```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing

application = ProtocolTypeRouter({
    # (http->django views is added by default)
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

**配置说明：**

**ProtocolTypeRouter**:  ASGI支持多种不同的协议，在这里可以指定特定协议的路由信息，我们只使用了websocket协议，这里只配置websocket即可

**AuthMiddlewareStack**: django的channels封装了django的auth模块，使用这个配置我们就可以在consumer中通过下边的代码获取到用户的信息:

```python
def connect(self):
	self.user = self.scope["user"]
    # self.scope类似于django中的request，包含了请求的type、path、header、cookie、session、user等等有用的信息
```

**URLRouter**： 指定路由文件的路径，也可以直接将路由信息写在这里，代码中配置了路由文件的路径，会去chat下的routeing.py文件中查找websocket_urlpatterns

```
**启动项目后发现启动过程由 原来的`Starting development server`  变为 `Starting ASGI/Channels version 2.1.6 development server`**
```



### 创建chat app

### 3.chat 目录下增加routing文件

**websocket请求访问路由文件，相当于django的urls**

```python
from django.urls import path
from chat.consumers import ChatConsumer

# 定义websocket的两条连接路径
websocket_urlpatterns = [
    path('ws/chat/', ChatConsumer), # 访问ws/chat/都交给ChatConsumer(相当于视图)处理
]
```



### 4.chat 目录下增加consumers文件

**websocket请求处理文件，相当于django的view**

```python
from channels.generic.websocket import WebsocketConsumer
import json

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_dict = json.loads(text_data)
        message = 'xxx：' + text_data_dict['message']
        self.send(text_data=json.dumps({
            'message': message
        }))
```

至此配合前端界面已经能实现消息发送接收



### 5.启用Channel Layer

```
上面已经实现了消息发送和接受，当打开多个浏览器分别输入消息后发现只有自己收到消息，其他浏览器端收不到，如何解决这个问题，让所有客户端都能一起聊天：
```

Channels引入了一个layer的概念，channel layer是一种通信系统，允许多个consumer实例之间互相通信，以及与外部Django程序实现互通。

channel layer主要实现了两种概念抽象：

**channel name：** channel实际上就是一个发送消息的通道，每个Channel都有一个名称，每一个拥有这个名称的人都可以往Channel里边发送消息

**group：** 多个channel可以组成一个Group，每个Group都有一个名称，每一个拥有这个名称的人都可以往Group里添加/删除Channel，也可以往Group里发送消息，Group内的所有channel都可以收到，但是无法发送给Group内的具体某个Channel

1. **官方推荐使用redis作为channel layer，所以先安装channels_redis**

```
pip install channels_redis
```

1. **修改setting对layer的支持**

```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [os.environ.get('REDIS_URL', 'redis://localhost:6379')],
        },
    },
}
```

添加channel之后我们可以通过python控制台检查通道层是否能够正常工作：

```python
>>> import channels.layers
>>> channel_layer = channels.layers.get_channel_layer()
>>>
>>> from asgiref.sync import async_to_sync
>>> async_to_sync(channel_layer.send)('test_channel',{'site':'redis://localhost:6379'})
>>> async_to_sync(channel_layer.receive)('test_channel')
{'site': 'redis://localhost:6379'}
```

1. **consumer做如下修改引入channel layer**

   self.scope： 相当于django的request

```python
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer
import json

class ChatConsumer(WebsocketConsumer):
    # 连接请求时触发
    def connect(self):
        # 可通过参数将房间名传进来作为组名，从而建立多个Group，这样可以实现仅同房间内的消息互通
       self.room_group_name = self.scope['url_route']['kwargs']['room_name'] # 设置组名
        # join the room
        # group_add(): 一个连接创建时：将该channel（也就相当于该连接）添加到该group
        async_to_sync(self.channel_layer.group_add)( 
            self.room_group_name ,
            self.channel_name
        )
        self.accept()

    # 连接关闭时触发
    def disconnect(self, close_code):
        # Leave room group
        # group_discard(): 一个连接关闭时：将该channel移出该group
        async_to_sync(self.channel_layer.group_discard)( 
            self.room_group_name,
            self.channel_name
        )

    # 收到消息后触发
    # Receive message from WebSocket
    def receive(self, text_data=None, bytes_data=None):
        text_data_dict = json.loads(text_data)
        message = '预警信息1：' + text_data_dict['msg']
        # Send message to room group
        # group_send(): 收到消息时:将消息发送到group，该Group内所有的channel都可以收到
        async_to_sync(self.channel_layer.group_send)(  
            self.room_group_name,
            {'type': 'chat_message','message': message}   # 参数type 指定消息处理的函数
        )
	
    # 处理器:消息处理函数
    # Receive message from room group
    def chat_message(self, event):
        message = '预警信息2：' + event['message']
        # Send message to WebSocket
        self.send(text_data=json.dumps({  # send(): ;通过WebSocket发送一个回复
            'message': message
        }))
```

```
当我们启用了channel layer之后，所有与consumer之间的通信将会变成异步的，所以必须使用`async_to_sync`。

经过以上的修改，再次在多个浏览器上打开聊天页面输入消息，发现彼此已经能够看到了。
```



### 6.修改为异步

上面实现的consumer是同步的，为了能有更好的性能，官方支持异步的写法，只需要修改consumer.py即可

```python
from asgiref.sync import async_to_sync
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class ChatConsumer(AsyncWebsocketConsumer):
    # 连接请求时触发
   async def connect(self):
        # 可通过参数将房间名传进来作为组名，从而建立多个Group，这样可以实现仅同房间内的消息互通
        self.room_group_name = self.scope['url_route']['kwargs']['room_name'] # 设置组名
        # join the room
        # group_add(): 一个连接创建时：将该channel（也就相当于该连接）添加到该group
       await self.channel_layer.group_add( 
            self.room_group_name ,
            self.channel_name
        )
       await self.accept()

    # 连接关闭时触发
   async def disconnect(self, close_code):
        # Leave room group
        # group_discard(): 一个连接关闭时：将该channel移出该group
       await self.channel_layer.group_discard( 
            self.room_group_name,
            self.channel_name
        )

    # 收到消息后触发
    # Receive message from WebSocket
   async def receive(self, text_data=None, bytes_data=None):
        text_data_dict = json.loads(text_data)
        message = '预警信息1：' + text_data_dict['msg']
        # Send message to room group
        # group_send(): 收到消息时:将消息发送到group，该Group内所有的channel都可以收到
       await self.channel_layer.group_send(  
            self.room_group_name,
            {'type': 'chat_message','message': message}   # 参数type 指定消息处理的函数
        )
    
	# 处理器:消息处理函数
    # Receive message from room group
   async def chat_message(self, event):
        message = '预警信息2：' + event['message']
        # Send message to WebSocket
       await self.send(text_data=json.dumps({  # send(): ;通过WebSocket发送一个回复
            'message': message
        }))
```

**异步的代码和之前只有几个小区别：**

ChatConsumer由`WebsocketConsumer`修改为了`AsyncWebsocketConsumer`

所有的方法都修改为了异步def`async def`， 用`await`来实现异步I/O的调用

channel layer也不再需要使用`async_to_sync`了

**consumer的几个基类：**

```
WebsocketConsumer ：基础类的通用封装

JsonWebsocketConsumer ：WebsocketConsumer的变体，可自动进行JSON编码和解码； 消息进出时,期望一切都是文字；否则报将二进制数据错误。

AsyncWebsocketConsumer ： WebsocketConsumer 的异步版本

AsyncJsonWebsocketConsumer ：JsonWebsocketConsumer 的变体，可自动进行JSON编码和解码。。。
```



## 消息推送功能

### 7.外部消息发送到Channels

**consumers.py文件中 定义外部可调用的消息推送函数**

**可以在 channels 外部将消息推送到指定group组里面，再推送给group中的各个channel**

```python
def push_msg(room_name,message):   # 可不用定义在
    # 从channels外部发送消息到channel
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        room_name,
        {
            "type": "chat_push_message",  # consumers类中的消息处理方法
            "message": message,
        }
    )

# 在python shell中测试：
push_msg('myroom', '预警信息推送')
```

**`channel_layer`实例获取：**调用 `group_send` 方法需要有 `channel_layer`实例。但在脱离了 consumer类情况下， Channels 提供了 `get_channel_layer` 函数接口来获取它。

**channel layers仅支持异步方法：**包括send()、group_send()，group_add()等，若是异步上下文环境，使用await 去异步调用其方法，若是同步上下文环境，使用 `async_to_sync` 来转换调用。



## 一对一功能

### 8.发送消息到指定channel

**channel_name可以在consumer中调用self.channel_name属性获取**

```python
from channels.layers import get_channel_layer

def send_to_channel(channel_name, msg)
    channel_layer = get_channel_layer()
    await channel_layer.send(channel_name, {
        "type": "chat_message",
        "text": "Hello there!",
    })
```



# 前端

## 使用vue构建前端简单聊天界面：

```
<template>
  <div>
    <div>nihao</div>
    <div style="text-align:center">
      <div class="chat-content">
        <span  v-for="item in info"> {{item}}<br/> </span>
      </div>
      <el-input v-model="input">
        <el-button slot="append" @click="wssend(input)">发送</el-button>
      </el-input>
    </div>
</div>
</template>

<script>
  export default {
    name : 'test',
    data() {
      return {
        input:'',
        websock: null,
        info:[]
      }
    },
    created() {
      this.initWebSocket();
    },
    // destroyed() {
    //   this.websock.close() //离开路由之后断开websocket连接
    // },
    methods: {
      initWebSocket(){ //初始化weosocket
      	let room = 'myroom'
      	// const path = "ws://127.0.0.1:8801/ws/chat/"; // websocket后端访问url
        const path = `ws://127.0.0.1:8801/ws/chat/${room}/`;
        this.websock = new WebSocket(path);
        // 监听websocket消息
        this.websock.onmessage = this.wsmessage;
        // 监听websocket连接
        this.websock.onopen = this.wsopen;
        // 监听websocket错误信息
        this.websock.onerror = this.wserror;
        // 监听ws关闭
        this.websock.onclose = this.wsclose;
      },
      wsopen(){ //连接建立之后执行send方法发送数据
        console.log('WS连接成功')
        this.wssend('连接成功!');
      },
      wserror(){//连接建立失败重连
        // console.log(`连接失败的信息：${e}`);
        console.log('连接失败');
        this.initWebSocket();
      },
      wsmessage(e){ //数据接收  回调e：事件对象
        // console.log(e) // messageEvent 对象
        const redata = JSON.parse(e.data);
        console.log('数据接收:', redata)
        this.info.push(redata.message)
      },
      wssend(message){//数据发送
        const data = {msg:message}
        this.websock.send(JSON.stringify(data)); // 转为json字符串发送
        console.log('数据发送:', data)
      },
      wsclose(e){  //关闭
        console.log('断开连接',e);
      },

    },
  //  destroyed () {
  //       // 销毁监听
  //       // this.socket.onclose = this.close
  //       this.websock.close()
  //   }
  }
</script>

<style scoped>
  * {
    margin:0 auto;
  }
  .chat-content {
    width:300px;
    height:300px;
    border:solid rgb(151, 175, 187) 10px;
    color:red;
    text-align:left
  }
  .el-input {
    width: 350px;
  }
</style>>

```

WebSocket对象一个支持四个消息：onopen，onmessage，oncluse和onerror，我们这里用了两个onmessage和onclose

**onopen：** 当浏览器和websocket服务端连接成功后会触发onopen消息

**onerror：** 如果连接失败，或者发送、接收数据失败，或者数据处理出错都会触发onerror消息

**onmessage：** 当浏览器接收到websocket服务器发送过来的数据时，就会触发onmessage消息，参数`e`包含了服务端发送过来的数据

**onclose：** 当浏览器接收到websocket服务器发送过来的关闭连接请求时，会触发onclose消息





