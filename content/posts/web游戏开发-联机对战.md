---
title: Web游戏开发
date: 2022-05-16T22:01:27+08:00
lastmod: 2022-05-16T22:01:27+08:00
author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - web游戏开发
tags:
  - 异步编程
  - django_channel
# nolastmod: true
draft: false
---

关于异步编程和django_channel的总结整理

<!--more-->

# 一、Python的异步编程

## 1、产生背景

历史上，Python 并不支持专门的异步编程语法，因为不需要。

有了多线程（threading）和多进程（multiprocessing），就没必要一定支持异步了。如果一个线程（或进程）阻塞，新建其他线程（或进程）就可以了，程序不会卡死。

但是，多线程有"线程竞争"的问题，处理起来很复杂，还涉及加锁。对于简单的异步任务来说（比如与网页互动），写起来很麻烦。

Python 3.4 引入了 `asyncio` 模块，增加了异步编程，跟 JavaScript 的`async/await` 极为类似，大大方便了异步任务的处理。它受到了开发者的欢迎，成为从 Python 2 升级到 Python 3 的主要理由之一。

## 2、进程、线程与协程

举个例子：假设有1个洗衣房，里面有10台洗衣机，有一个洗衣工在负责这10台洗衣机。那么洗衣房就相当于1个进程，洗衣工就相当1个线程。如果有10个洗衣工，就相当于10个线程，1个进程是可以开多线程的。这就是多线程！

那么协程呢？先不急。大家都知道，洗衣机洗衣服是需要等待时间的，如果10个洗衣工，1人负责1台洗衣机，这样效率肯定会提高，但是不觉得浪费资源吗？明明1 个人能做的事，却要10个人来做。只是把衣服放进去，打开开关，就没事做了，等衣服洗好再拿出来就可以了。就算很多人来洗衣服，1个人也足以应付了，开好第一台洗衣机，在等待的时候去开第二台洗衣机，再开第三台，……直到有衣服洗好了，就回来把衣服取出来，接着再取另一台的（哪台洗好先就取哪台，所以协程是无序的）。这就是计算机的协程！洗衣机就是执行的方法。

在python中，async与await就是协程的一种实现方式。

## 3、async和await的使用

正常的函数在执行时是不会中断的，所以你要写一个能够中断的函数，就需要添加async关键字。

async 用来声明一个函数为异步函数，异步函数的特点是能在函数执行过程中挂起，去执行其他异步函数，等到挂起条件（假设挂起条件是sleep(5)）消失后，也就是5秒到了再回来执行。

await 用来用来声明程序挂起，比如异步程序执行到某一步时需要等待的时间很长，就将此挂起，去执行其他的异步程序。await 后面只能跟异步程序或有__await__属性的对象，因为异步程序与一般程序不同。假设有两个异步函数async a，async b，a中的某一步有await，当程序碰到关键字await b()后，异步程序挂起后去执行另一个异步b程序，就是从函数内部跳出去执行其他函数，**当挂起条件消失后，不管b是否执行完，要马上从b程序中跳出来**，回到原程序执行原来的操作。如果await后面跟的b函数不是异步函数，那么操作就只能等b执行完再返回，无法在b执行的过程中返回。如果要在b执行完才返回，也就不需要用await关键字了，直接调用b函数就行。所以这就需要await后面跟的是异步函数了。在一个异步函数中，可以不止一次挂起，也就是可以用多个await。

# 二、Django Channels

## 1、概念

django的channels在官方文档里有着很详细的介绍，这里只总结一下项目用到的部分。

首先，官方表面整个channels遵循“turtles all the way down” 的原则，搜了一下，发现这背后还有一个典故，大概就是我们的世界是在一个乌龟的背上，而乌龟的下面是一个又一个乌龟驮着的乌龟群。作者这里在后面解释了一下，所有的channels的应用都是一整个有效的，可以自己运行的ASGI（ASGI is the name for the asynchronous server specification that Channels is built on.）应用。

## 2、channels中的消费者

消费者是 Channels 代码的基本单位。我们称它为*消费者*，因为它 *消费事件*，但您可以将其视为它自己的小应用程序。当一个请求或新套接字进来时，Channels 将遵循它的路由表——我们稍后会看一下——为那个传入连接找到正确的消费者，并启动它的副本。

这意味着，与 Django 视图不同，消费者是长期运行的。它们也可以是短期运行的——毕竟，HTTP 请求也可以由消费者提供服务——但它们是围绕生存一段时间的想法构建的（正如我们上面所描述的，它们在一个*范围内生存）。*

官网中提供了消费者的基本代码：

```
class ChatConsumer(WebsocketConsumer):

    def connect(self):
        self.username = "Anonymous"
        self.accept()
        self.send(text_data="[Welcome %s!]" % self.username)

    def receive(self, *, text_data):
        if text_data.startswith("/name"):
            self.username = text_data[5:].strip()
            self.send(text_data="[set your username to %s]" % self.username)
        else:
            self.send(text_data=self.username + ": " + text_data)

    def disconnect(self, message):
        pass
```

也可以用它来编写完全异步的函数：

```
class PingConsumer(AsyncConsumer):
    async def websocket_connect(self, message):
        await self.send({
            "type": "websocket.accept",
        })

    async def websocket_receive(self, message):
        await asyncio.sleep(1)
        await self.send({
            "type": "websocket.send",
            "text": "pong",
        })
```

## 3、路由

您可以将多个使用者（记住，他们自己的 ASGI 应用程序）组合成一个更大的应用程序，使用路由来代表您的项目：

```
application = URLRouter([
    path("chat/admin/", AdminChatConsumer.as_asgi()),
    path("chat/", PublicChatConsumer.as_asgi(),
])
```

Channels 不仅仅是围绕 HTTP 和 WebSockets 构建的——它还允许您将任何协议构建到 Django 环境中，方法是构建一个将这些协议映射到一组类似事件的服务器。例如，您可以构建一个类似风格的聊天机器人：

```
class ChattyBotConsumer(SyncConsumer):

    def telegram_message(self, message):
        """
        Simple echo handler for telegram messages in any chat.
        """
        self.send({
            "type": "telegram.message",
            "text": "You said: %s" % message["text"],
        })
```

然后使用另一个路由器让一个项目能够同时服务 WebSocket 和聊天请求：

```
application = ProtocolTypeRouter({

    "websocket": URLRouter([
        path("chat/admin/", AdminChatConsumer.as_asgi()),
        path("chat/", PublicChatConsumer.as_asgi()),
    ]),

    "telegram": ChattyBotConsumer.as_asgi(),
})
```

Channels 的目标是让您构建您的 Django 项目以跨现代网络中可能遇到的任何协议或传输方式工作，同时让您使用熟悉的组件和编码风格。

## 4、进程间通信

通道层允许您在应用程序的不同实例之间进行对话。如果您不想让所有消息或事件通过数据库传送，它们是制作分布式实时应用程序的有用部分。

#### 1）Redis 通道层

[channels_redis](https://pypi.org/project/channels-redis/)是唯一支持生产使用的官方 Django 维护的通道层。该层使用 Redis 作为其后备存储，它支持单服务器和分片配置以及组支持。要使用这一层，您需要安装[channels_redis](https://pypi.org/project/channels-redis/) 包。

在此示例中，Redis 在 localhost (127.0.0.1) 端口 6379 上运行：

```
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

同时还支持内存通道层，但是官方建议不要在生产中使用内存通道层的配置，因此一般只会使用Redis通道层。

#### 2）同步函数

默认情况下`send()`，`group_send()`、`group_add()`和其他函数是异步函数，这意味着您必须使用`await`它们。如果您需要从同步代码中调用它们，则需要使用方便的 `asgiref.sync.async_to_sync`包装器：

```
from asgiref.sync import async_to_sync

async_to_sync(channel_layer.send)("channel_name", {...})
```

#### 3）发送消息

通道层用于高级应用程序到应用程序的通信。当您发送消息时，它会被另一端收听组或频道的消费者接收。这意味着您应该通过通道层发送高级事件，然后让消费者处理这些事件，并对其连接的客户端进行适当的低级联网。

例如，聊天应用程序可以通过通道层发送如下事件：

```
await self.channel_layer.group_send(
    room.group_name,
    {
        "type": "chat.message",
        "room_id": room_id,
        "username": self.scope["user"].username,
        "message": message,
    }
)
```

然后消费者定义一个处理函数来接收这些事件并将它们转换为 WebSocket 帧：

```
async def chat_message(self, event):
    """
    Called when someone has messaged our chat.
    """
    # Send a message down to the client
    await self.send_json(
        {
            "msg_type": settings.MSG_TYPE_MESSAGE,
            "room": event["room_id"],
            "username": event["username"],
            "message": event["message"],
        },
    )
```

任何基于 Channels 的消费者`SyncConsumer`或`AsyncConsumer`将自动为您提供一个`self.channel_layer`and`self.channel_name` 属性，其中包含指向通道层实例的指针和将分别到达消费者的通道名称。

发送到该通道名称（或通道名称添加到的组）的任何消息都将被消费者接收，就像来自其连接客户端的事件一样，并分派到消费者上的命名方法。该方法的名称将是`type`事件的名称，其中句点替换为下划线 - 因此，例如，通过通道层传入的事件`type`将由`chat.join`该方法处理`chat_join`。

笔记

如果您从`AsyncConsumer`类树继承，您的所有事件处理程序，包括通道层上的事件处理程序，都必须是异步的 ( )。如果您在类树中，则它们都必须是同步的 ( )。`async def``SyncConsumer``def`

#### 4）单通道

每个应用程序实例 - 例如，每个长时间运行的 HTTP 请求或打开的 WebSocket - 都会产生一个消费者实例，如果您启用了通道层，消费者将为自己生成一个唯一的*通道名称*，并开始监听它事件。

这意味着您可以从流程外部（可能来自其他消费者，或来自管理命令）发送这些消费者事件，他们将对它们做出反应并运行代码，就像他们从客户端连接中发送事件一样。

频道名称在消费者上可用`self.channel_name`. 这是一个在连接时将通道名称写入数据库的示例，然后为其上的事件指定处理方法：

```
class ChatConsumer(WebsocketConsumer):

    def connect(self):
        # Make a database row with our channel name
        Clients.objects.create(channel_name=self.channel_name)

    def disconnect(self, close_code):
        # Note that in some rare cases (power loss, etc) disconnect may fail
        # to run; this naive example would leave zombie channel names around.
        Clients.objects.filter(channel_name=self.channel_name).delete()

    def chat_message(self, event):
        # Handles the "chat.message" event when it's sent to us.
        self.send(text_data=event["text"])
```

请注意，因为您正在混合来自通道层和协议连接的事件处理，所以您需要确保您的类型名称不会发生冲突。建议您为类型名称添加前缀（就像我们在此处使用 所做的那样`chat.`）以避免冲突。

要发送到单个频道，只需找到其频道名称（对于上面的示例，我们可以爬取数据库），然后使用`channel_layer.send`：

```
from channels.layers import get_channel_layer

channel_layer = get_channel_layer()
await channel_layer.send("channel_name", {
    "type": "chat.message",
    "text": "Hello there!",
})
```

#### 5）Groups，通道中的广播机制

显然，发送到单个channels并不是特别有用 - 在大多数情况下，您希望一次发送到多个channels/消费者作为广播。不仅适用于聊天等情况，您希望将传入消息发送给房间中的每个人，甚至可以发送给可能连接了多个浏览器选项卡或设备的个人用户。

如果您喜欢使用现有数据存储，您可以为此构建自己的解决方案，或者您可以使用某些通道层内置的组系统。Groups 是一个广播系统，它：

- 允许您在命名组中添加和删除频道名称，并发送到这些命名组。
- 提供组到期以清理其断开处理程序未运行的连接（例如电源故障）

它们不允许您枚举或列出组中的频道；这是一个纯粹的广播系统。如果您需要更精确的控制或需要知道谁连接，您应该构建自己的系统或使用合适的第三方系统。

您可以通过在连接期间向组添加通道并在断开连接期间将其删除来使用组，如 WebSocket 通用使用者上所示：

```
# This example uses WebSocket consumer, which is synchronous, and so
# needs the async channel layer functions to be converted.
from asgiref.sync import async_to_sync

class ChatConsumer(WebsocketConsumer):

    def connect(self):
        async_to_sync(self.channel_layer.group_add)("chat", self.channel_name)

    def disconnect(self, close_code):
        async_to_sync(self.channel_layer.group_discard)("chat", self.channel_name)
```

然后，要发送到一个组，请使用`group_send`，就像在这个小示例中一样，当与上面的代码结合使用时，它会将聊天消息广播到每个连接的套接字：

```
class ChatConsumer(WebsocketConsumer):

    ...

    def receive(self, text_data):
        async_to_sync(self.channel_layer.group_send)(
            "chat",
            {
                "type": "chat.message",
                "text": text_data,
            },
        )

    def chat_message(self, event):
        self.send(text_data=event["text"])
```

#### 6) 在消费者之外调用通道函数

您通常希望从消费者范围之外发送到通道层，因此您不会有`self.channel_layer`. 在这种情况下，您应该使用该`get_channel_layer`函数来检索它：

```
from channels.layers import get_channel_layer
channel_layer = get_channel_layer()
```

然后，一旦你拥有它，你就可以调用它的方法。请记住， **通道层仅支持异步方法**，因此您可以从自己的异步上下文中调用它：

```
for chat_name in chats:
    await channel_layer.group_send(
        chat_name,
        {"type": "chat.system_message", "text": announcement_text},
    )
```

或者您需要使用 async_to_sync：

```
from asgiref.sync import async_to_sync

async_to_sync(channel_layer.group_send)("chat", {"type": "chat.force_disconnect"})
```

## 6、部署配置



# 参考资料

1、django channels官方文档： https://channels.readthedocs.io/en/latest/