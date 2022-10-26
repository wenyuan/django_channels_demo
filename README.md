# Django 集成 Channels 使用案例

该项目出自[官方文档](https://channels.readthedocs.io/en/stable/tutorial/index.html)的案例，增加了一些更详细的代码解读和注释。实现步骤可以通过官方文档进行学习。

## 依赖环境

* Linux
* Python 3.8
* Django 4.1.2（兼容 2.x 和 3.x）
* Channels 3.0.5

## 运行方式

下载、安装、运行代码：

```bash
git clone https://github.com/wenyuan/django_channels_demo.git
pip install -r requirements.txt
cd django_channels_demo/
python manage.py runserver
```

浏览器访问 `ip:port/chat`，输入任意聊天室的名字，进入聊天室开始聊天（WebSocket 测试）。

## 主要文件

下面对这个小案例涉及到的主要文件/文件夹进行标注，省略跟 Channels 无关的文件。

```
django_channels_demo/
├── chat
│   │── templates/      # 模板文件（即 html）
│   │── urls.py         # 映射到视图的路由，处理 http 请求
│   │── views.py        # 视图文件，接收并处理 http 链接
│   │── routing.py      # 到消费者的路由，专门处理 websocket，类似普通的处理 http 请求的 urls.py
│   │── consumers.py    # 消费者，接收并处理 websocket 链接
│   └── ...（其它省略）
├── django_channels_demo
│   │── settings.py/    # Django 配置文件
│   │── urls.py         # 项目根路由
│   │── asgi.py         # 配置 ASGI 协议，让它替代并兼容原本的 WSGI 协议
│   └── ...（其它省略）
└── ...
```

## 知识点

### WebSocket

WebSocket 是一种在单个 TCP 连接上进行全双工通讯的协议。它允许服务端主动向客户端推送数据。客户端浏览器和服务器只需要完成一次握手就可以创建持久性的连接，并在浏览器和服务器之间进行双向的数据传输。

使用场景：

* 在 Django 中遇到一些耗时较长的任务我们通常会使用 Celery 来异步执行，那么浏览器如果想要获取这个任务的执行状态，在 HTTP 协议中只能通过轮询的方式由浏览器不断的发送请求给服务器来获取最新状态，这样发送很多无用的请求不仅浪费资源，还不够优雅，这时候可以考虑使用 WebSocket 来实现。
* 对于告警或通知的需求，虽然使用 HTTP 轮询也能实现，但借助 WebSocket 能创建持久性的连接并可以让服务端主动发起消息的特性，实现起来会更优雅。
* 还有诸如本案例的聊天室场景，一个用户（浏览器）发送的消息需要实时的让其他用户（浏览器）接收，这在 HTTP 协议下是很难实现的。

### Channels

Django 本身不支持 WebSocket，但可以通过集成 Channels 框架来实现 WebSocket。

Channels 是针对 Django 项目的一个增强框架，可以使 Django 不仅支持 HTTP 协议，还能支持 WebSocket，MQTT 等多种协议，同时 Channels 还将 Django 自带的认证系统以及 session 集成到了模块中，扩展性非常强。

### Daphne

普通的 Django 应用部署方式都采用的是 Nginx + UWSGI 进行部署，当 Django 集成 Channels 时候，由于 UWSGI 不能处理 WebSocket 请求，所以我们需要 ASGI 服务器来处理 WebSocket 请求，官方推荐使用 daphne。

配置入口就是 `asgi.py`：

```python
... 省略

application = ProtocolTypeRouter(
    {
        "http": django_asgi_app,
        "websocket": AllowedHostsOriginValidator(
            AuthMiddlewareStack(URLRouter(chat.routing.websocket_urlpatterns))
        ),
    }
)
```

这样一来，Daphne 服务就接管了 Django 原先的服务，能同时支持 `http` 和 `websocket` 请求：

* 当 Django 接收到 HTTP 请求时，它会查询根路由配置（urls.py）来查找视图函数，然后调用视图函数来处理请求。
* 当 Channels 接收到 WebSocket 连接时，它会查询根路由配置（routing.py）来查找使用者，然后调用使用者上的各种函数来处理来自连接的事件。

> 注意：Daphne 库必须位于 `INSTALLED_APPS` 的顶部，这样可以确保 Daphne 模式的 runserver 命令正常运行。因为有可能你使用的其他第三方应用也要修改 runserver 命令，从而会产生冲突。

### Consumer

Consumer 类似 Django 中的 View 或者 API，用于编写具体业务逻辑的代码。

### Channel Layer

Channel Layer 是一个中间通道层，它允许多个使用者实例之间以及与 Django 的其他部分进行通信。

举个例子，如果不引入 Channel Layer，那么本案例实现的聊天室只能作为单机聊天室，也就是先打开第一个聊天窗口，然后当打开第二个浏览器选项卡到同一个房间页面并键入一条消息，该消息将不会出现在第一个选项卡中。

因此为了实现同一个房间中多个 ChatConsumer 实例相互通信的需求，Channels 引入了一个 Layer 的概念，它的大致原理是：

* 让每个 ChatConsumer 将其 Channel 添加到一个组（Group），这里我们把组就命名为房间名称。
* 使用 Redis 作为后台存储，当用户发布消息时，JavaScript 函数将通过 WebSocket 将消息传输到 ChatConsumer。ChatConsumer 接收到该消息并将其转发到对应的组（Group）。同一组中（即同一个房间中）的每个 ChatConsumer 将接收来自该组的消息，并通过 WebSocket 将其转发回 JavaScript，然后将其添加到聊天日志中。

因此 Layer 有两个关键的概念：

* channel name：Channel 实际上就是一个发送消息的通道，每个 Channel 都有一个名称，每一个拥有这个名称的人都可以往 Channel 里面发送消息。
* group：多个 channel 可以组成一个 Group，每个 Group 都有一个名称，每一个拥有这个名称的人都可以往 Group 里面添加/删除 Channel，也可以往 Group 里发送消息，Group 内的所有 Channel 都可以收到，但是无法发送给 Group 内的具体某个 Channel。

然后的话，官方推荐使用 Redis 作为 Channel Layer，与之配对的库是 `channels_redis`。

## 关键代码解释

### settings.py

```python
# 指定 ASGI 的路由地址
ASGI_APPLICATION = 'django_channels_demo.asgi.application'
```

Channels 需要运行于 ASGI 协议上，它是区别于 Django 使用的 WSGI 协议的一种异步服务网关接口协议，`ASGI_APPLICATION` 指定主路由的位置（`application` 的位置）。官方文档是把 `application` 写在 `settings.py` 同级的 `asgi.py` 里面的，看到很多人写的文章会在 `settings.py` 同级新建一个 `routing.py`，然后将 `application` 写在里面。

### asgi.py 或 routing.py

里面的内容就类似于 Django 中的 url.py，指明 WebSocket 协议的路由。

```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing

application = ProtocolTypeRouter({
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

* ProtocolTypeRouter：ASIG 支持多种不同的协议，在这里可以指定特定协议的路由信息，如果只使用了 WebSocket 协议，这里只配置 `websocket` 即可。
* AuthMiddlewareStack：Django 的 Channels 封装了 Django 的 auth 模块，使用这个配置我们就可以在 consumer 中通过类似下面的代码获取到用户的信息。
  ```python
  def connect(self):
      # self.scope 类似于 Django 中的 request，包含了请求的 type、path、header、cookie、session、user 等等有用的信息
      self.user = self.scope["user"]
  ```
* URLRouter：指定路由文件的路径，也可以直接将路由信息写在这里，代码中配置了路由文件的路径，会去 `chat` 下的 routeing.py 文件中查找 `websocket_urlpatterns`。

### chat/routing.py

routing.py 路由文件跟 Django 的 url.py 功能类似，语法也一样，下面代码的意思就是访问 `ws/chat/` 都交给 ChatConsumer 处理。

```python
from django.urls import re_path
from chat.consumers import ChatConsumer

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", ChatConsumer.as_asgi()),
]
```

调用 `as_asgi()` 类方法是为了获得一个 ASGI 应用程序，该应用程序将为每个用户连接提供 ChatConsumer 的实例。这类似于 Django 的 `as_view()`，它起的作用跟每个对 Django 视图的请求一样。

### consumers.py

consumer 类似 Django 中的 view，一个简单的 consumer 类如下：

```python
from channels.generic.websocket import WebsocketConsumer
import json


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]
        # send message to websocket
        self.send(text_data=json.dumps({
            "message": message
        }))
```

`connect` 方法在连接建立时触发，`disconnect` 在连接关闭时触发，`receive` 方法会在收到消息后触发。

### room.html

这里主要是对页面添加 WebSocket 支持（属于前端开发的知识）。

```javascript
const chatSocket = new WebSocket(
  'ws://'
  + window.location.host
  + '/ws/chat/'
  + roomName
  + '/'
);

chatSocket.onmessage = function (e) {
  const data = JSON.parse(e.data);
  document.querySelector('#chat-log').value += (data.message + '\n');
};

chatSocket.onclose = function (e) {
  console.error('Chat socket closed unexpectedly');
};
```

WebSocket 对象一个支持四个消息：`onopen`，`onmessage`，`onclose` 和 `onerror`，我们这里用了两个 `onmessage` 和 `onclose`。

* `onopen`：当浏览器和 WebSocket 服务端连接成功后会触发 `onopen` 消息。
* `onerror`：如果连接失败，或者发送、接收数据失败，或者数据处理出错都会触发 `onerror` 消息。
* `onmessage`：当浏览器接收到 WebSocket 服务器发送过来的数据时，就会触发 `onmessage` 消息，参数 `e` 包含了服务端发送过来的数据。
* `onclose`：当浏览器接收到 WebSocket 服务器发送过来的关闭连接请求时，会触发 `onclose` 消息。

所以，在浏览器中输入消息就会通过 websocket --> rouging.py --> consumer.py 处理后返回给前端。
