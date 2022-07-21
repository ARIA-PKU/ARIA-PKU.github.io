---
title: Web游戏开发 基于Thrift的游戏匹配系统
date: 2022-05-20T16:53:05+08:00
lastmod: 2022-05-20T16:53:05+08:00
cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/reverse_world.jpg
categories:
  - web游戏开发
tags:
  - RPC
  - Thrift
# nolastmod: true
draft: false
---

总结整理一下用到的RPC框架——Thrift

<!--more-->

# Thrift

## 一、特性介绍

我们写一个应用时，这个应用程序并不止一个服务，而且不同的服务分配到不同服务器(或者进程)上，也就是我们常说的微服务 。通过Thrift这样的RPC框架，可以实现服务的调用。

### 关于http与rpc

在实习期间，当时做业务的时候，既会使用http直接调用，也会通过公司自研的tRPC调用服务，在使用过程中没有感觉到明显的差别，在自己接触了thrift的rpc框架后发现，rpc的直观优势在于不需要自己写两端的接口，只需要定义好需要实现的函数，框架能自动生产服务端与客户端的协议，而不用管底层的协议，接口，封装等问题，只需要在服务端写好函数即可，当然还需要熟悉框架的使用。

在看过网上一些总结之后，发现还有其他的很多区别：

首先，这都是两种服务调用的方法，但是又不是明确的并列关系。从**通信协议**来看，如果是基于http1.1的话，http的调用缺点很多，比如报头信息过长导致无用信息过多，但是http2.0就没有这个问题了（在另一篇博客总结了各个http协议的不同）。有些rpc框架，例如gRPC就是采用了http2.0作为传输协议的，当然还有自定义tcp协议的（例如dubbo）。

但是，成熟的rpc库相对http容器，更多的是封装了**“服务发现”**，**"负载均衡"**，**“熔断降级”**一类面向服务的高级特性。可以这么理解，rpc框架是面向服务的更高级的封装。如果把一个http servlet容器上封装一层服务发现和函数代理调用，那它就已经可以做一个rpc框架了。

因此，rpc的优势在于良好的rpc调用是面向服务的封装，针对服务的可用性和效率等都做了优化。单纯使用http调用则缺少了这些特性。

### thrift的简介

**Apache Thrift**软件框架用于可伸缩的跨语言服务开发，它将**软件栈**和**代码生成引擎**结合在一起，以构建在C++、Java、Python、PHP、Ruby、Erlang、Perl、Haskell、C#、Cocoa、JavaScript、Node.js、Smalltalk、OCaml和Delphi等语言之间高效、无缝地工作的服务。

**Thrift使用C++进行编写**，在安装使用的时候需要安装依赖，安装方式见官网：https://thrift.apache.org/

Thrift 采用**IDL**（Interface Definition Language）来定义通用的服务接口，然后通过Thrift提供的编译器，可以将服务接口编译成不同语言编写的代码，通过这个方式来实现跨语言的功能。

通过命令调用Thrift提供的编译器将服务接口编译成不同语言编写的代码。

这些代码又分为服务端和客户端，将所在不同进程(或服务器)的功能连接起来。

### 网络栈

关于其网络栈，在网上找到的图片：

![image.png](http://oss.surfaroundtheworld.top/blog-pictures/6_15/thrift.jpg)

下面两层为数据传输层：

**TTransport层**

代表thrift的数据传输方式，thrift定义了如下几种常用数据传输方式

- `TSocket`: 阻塞式socket；
- `TFramedTransport`: 以frame为单位进行传输，非阻塞式服务中使用；
- `TFileTransport`: 以文件形式进行传输；

**TProtocol层**

代表thrift客户端和服务端之间传输数据的协议，通俗来讲就是客户端和服务端之间传输数据的格式(例如json等)，thrift定义了如下几种常见的格式

- `TBinaryProtocol`: 二进制格式；
- `TCompactProtocol`: 压缩格式；
- `TJSONProtocol`: JSON格式；
- `TSimpleJSONProtocol`: 提供只写的JSON协议；

**thrift支持的Server模型**

thrift主要支持以下几种服务模型

- `TSimpleServer`: 简单的单线程服务模型，常用于测试；
- `TThreadPoolServer`: 多线程服务模型，使用标准的阻塞式IO；
- `TNonBlockingServer`: 多线程服务模型，使用非阻塞式IO(需要使用`TFramedTransport`数据传输方式);
- `THsHaServer`: `THsHa`引入了线程池去处理，其模型读写任务放到线程池去处理，`Half-sync/Half-async`处理模式，`Half-async`是在处理IO事件上(accept/read/write io)，`Half-sync`用于handler对rpc的同步处理；

`TThreadPoolServer`与`TNonBlockingServer`都可以在实际生产中拿来用。在客户端不多的情况下，可以选用TThreadPoolServer，但是要注意TThreadPoolServer的客户端只要不从服务器上断开连接，就会一直占据服务器的一个线程，当服务器线程池所有线程都在被使用时，新到来的客户端将排在队列里等待，直到有客户端断开连接，使服务器端线程池出现空闲线程方可继续被提供服务，所以使用这种模型时，一定要注意客户端不使用时不要长时间连接服务器，如果确实有这种需求，请使用TNonblockingServer。

### thrift 接口文件

以下是官网中给出的thrift基础数据类型：

```
//The first thing to know about are types. The available types in Thrift are:
bool        //Boolean, one byte
i8 (byte)   //Signed 8-bit integer
i16         //Signed 16-bit integer
i32         //Signed 32-bit integer
i64         //Signed 64-bit integer
double      //64-bit floating point value
string      //String
binary      //Blob (byte array)
map<t1,t2>  //Map from one type to another
list<t1>    //Ordered list of one type
set<t1>     //Set of unique elements of one type
```

## 二、在项目中的使用

使用rpc的原因在于，匹配系统的匹配时间可能会随策略的不同而变得很长，长期占用当前服务的进程会造成资源的浪费，因此可以通过其他进程来实现，利用rpc可以实现进程间的通信。除了游戏的匹配系统，像是外卖的骑手匹配，打车的匹配，以及OJ的在线测评都可以通过rpc实现。根据服务的需求，可以将rpc的服务端放到其他的服务器上，对于更大型的业务还需要负载均衡等。对于thrift而言，服务的连接是通过IP地址加端口号确定，rpc易于在不同服务器上通信。

在实现的过程中，一般可以通过异步返回结果的方式或者在服务端和客户端再分别设置服务端和客户端的方法，获得匹配结果，但是django_channels提供了一个可以从外部调用的方法，因此可以直接在thrift的服务端直接调用channel中的方法，实现进程间通信。

但是这种方式需要客户端与服务端在同一台服务器上，并且需要同时使用python语言实现才可以。查阅资料发现，thrift开源代码本身不支持这种直接的异步返回结果，但是可以通过其他开源代码实现。但是，也可以采用在两端同时设置服务端和客户端两套thrift来实现，日后尝试用后者实现，暂时埋个坑。

### thrift的使用

根据官网教程，首先需要确定服务端需要被调用的函数，在项目中我们需要调用的只有`add_player`这一个函数，将其写到`match.thrift`文件中，在官方的`tutorial.thrift`(https://github.com/apache/thrift/blob/master/tutorial/tutorial.thrift)中，还给出了如何定义结构体等：

```
namespace py match_service  # py代表语言，namespace是命名域，c++中语法


service Match {
    i32 add_player(  # 这里声明类型需要是给定的类型，声明方式也比较特别
    1: i32 score, 
    2: string uuid, 
    3: string username, 
    4: string photo, 
    5: string channel_name),
}
```

然后创建`src`文件夹存放thrift自动生成的服务端文件，通过命令：`thrift -r --gen py ../thrift/match.thrift`，就能在src文件夹下生成对应的服务端文件，产生的文件结构如下：

```
.
|-- src
|   `-- gen-py
|       |-- __init__.py
|       `-- match_service
|           |-- Match-remote
|           |-- Match.py
|           |-- __init__.py
|           |-- constants.py
|           `-- ttypes.py  # 如果在.thrift文件中定义了struct会在这里生成
`-- thrift
    `-- match.thrift  # 定义函数的
```

注意这里的`ttypes.py`，如果定义了struct类型，需要在客户端和服务端引入该`.py`文件。

接下来，在官网的tutorial中找到教程的服务端模板，编写服务端代码，在教程中代码其实是很通俗易懂的，下面是教程中给出的**服务端**示例：

```
import glob
import sys
# 系统路径定向
sys.path.append('gen-py')
sys.path.insert(0, glob.glob('../../lib/py/build/lib*')[0])

# 引入生成的用于通信的类
from tutorial import Calculator
from tutorial.ttypes import InvalidOperation, Operation

from shared.ttypes import SharedStruct

# 各个传输协议，上面介绍过了
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol
from thrift.server import TServer

# Handler类，这里与下面对应
class CalculatorHandler:
    def __init__(self):
        self.log = {}

    def ping(self):
        print('ping()')

    def add(self, n1, n2):
        print('add(%d,%d)' % (n1, n2))
        return n1 + n2

    def calculate(self, logid, work):
        print('calculate(%d, %r)' % (logid, work))

        if work.op == Operation.ADD:
            val = work.num1 + work.num2
        elif work.op == Operation.SUBTRACT:
            val = work.num1 - work.num2
        elif work.op == Operation.MULTIPLY:
            val = work.num1 * work.num2
        elif work.op == Operation.DIVIDE:
            if work.num2 == 0:
                raise InvalidOperation(work.op, 'Cannot divide by 0')
            val = work.num1 / work.num2
        else:
            raise InvalidOperation(work.op, 'Invalid operation')

        log = SharedStruct()
        log.key = logid
        log.value = '%d' % (val)
        self.log[logid] = log

        return val

    def getStruct(self, key):
        print('getStruct(%d)' % (key))
        return self.log[key]

    def zip(self):
        print('zip()')


if __name__ == '__main__':
    handler = CalculatorHandler()  # 这里确定handler的类
    processor = Calculator.Processor(handler)  
    transport = TSocket.TServerSocket(host='127.0.0.1', port=9090)  # 传输端口
    tfactory = TTransport.TBufferedTransportFactory()
    pfactory = TBinaryProtocol.TBinaryProtocolFactory()
	
	# server的线程选择，默认的是单线程，主要是测试用，上面也对线程的使用做了总结
    server = TServer.TSimpleServer(processor, transport, tfactory, pfactory)

    # You could do one of these for a multithreaded server
    # server = TServer.TThreadedServer(
    #     processor, transport, tfactory, pfactory)
    # server = TServer.TThreadPoolServer(
    #     processor, transport, tfactory, pfactory)

    print('Starting the server...')
    server.serve()
    print('done.')
```

下面是官网给的client端示例：

```
import sys
import glob
# 确定系统路径
sys.path.append('gen-py')
sys.path.insert(0, glob.glob('../../lib/py/build/lib*')[0])

from tutorial import Calculator  # 在对应文件夹中引入相应模块，这个与服务端的模块一致
from tutorial.ttypes import InvalidOperation, Operation, Work   # 定义了struct也需要引入

# 下面的协议与服务端一致
from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol


def main():
    # Make socket
    transport = TSocket.TSocket('localhost', 9090)

    # Buffering is critical. Raw sockets are very slow
    transport = TTransport.TBufferedTransport(transport)

    # Wrap in a protocol
    protocol = TBinaryProtocol.TBinaryProtocol(transport)

    # Create a client to use the protocol encoder
    client = Calculator.Client(protocol)  # 这里是用自己的模块

    # Connect! 从这里开始到下面的close就是自己定义的函数的位置了
    transport.open()

    client.ping()
    print('ping()')

    sum_ = client.add(1, 1)
    print('1+1=%d' % sum_)

    work = Work()

    work.op = Operation.DIVIDE
    work.num1 = 1
    work.num2 = 0

    try:
        quotient = client.calculate(1, work)
        print('Whoa? You know how to divide by zero?')
        print('FYI the answer is %d' % quotient)
    except InvalidOperation as e:
        print('InvalidOperation: %r' % e)

    work.op = Operation.SUBTRACT
    work.num1 = 15
    work.num2 = 10

    diff = client.calculate(1, work)
    print('15-10=%d' % diff)

    log = client.getStruct(1)
    print('Check log: %s' % log.value)

    # Close!
    transport.close()
```

### 项目中的设计思路

这里的设计思路，与另一篇blog里面写的改进版一致，同样是随着等待时间的变化扩大了分数匹配的范围，并且通过了django_channels调用了接口，返回了匹配结果。

因为python中有线程安全的队列作为消息队列使用，所以实现起来更为简单，详细见github代码中。







