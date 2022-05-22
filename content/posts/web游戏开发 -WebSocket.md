---
title: Web游戏开发（六）
date: 2022-05-16T20:53:26+08:00
lastmod: 2022-05-16T20:53:26+08:00
author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - web游戏开发
tags:
  - websocket
# nolastmod: true
draft: false
---

web游戏开发日志，对于多人联机对战中websocket的使用

<!--more-->

# WebSocket的总结整理

## 1、WebSocket的使用场景

WebSocket的一大特性就是高实时性，项目中准备利用ws实现玩家对战和游戏界面聊天的功能，其应用的场景有：

1）IM，即时通信的软件；

2）弹幕，例如A站、B站以及各大直播平台的弹幕功能；

3）多玩家游戏，项目即将实现的功能；

4）协同编辑，像是石墨，腾讯文档等；

5）视频会议；

6）股票、体育实况更新；

## 2、与http的区别

在以上的特定场景中是基本不会使用http的，无论是http1.0的短链接还是http1.1的长连接，都需要请求方发送request才能收到更新的内容，这个过程最快也需要$2*RSS$的时间，http2.0的推送功能也不能直接让应用程序感知，需要通过SSE（Server Sent Event）来实现，因此还是会采用ws来实现。关于http的各个版本会在单独写一遍博客进行总结。

## 3、WS的原理

Websocket是一个应用层协议，它必须依赖 HTTP 协议进行一次握手 ，握手成功后，数据就直接从 TCP 通道传输，与 HTTP 无关了。

Websocket的数据传输是frame形式传输的，比如会将一条消息分为几个frame，按照先后顺序传输出去。这样做会有几个好处：

1) 大数据的传输可以分片传输，不用考虑到数据大小导致的长度标志位不足够的情况。

2) 和http的chunk一样，可以边生成数据边传递消息，即提高传输效率。

```ruby
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+



    FIN      1bit 表示信息的最后一帧，flag，也就是标记符
    RSV 1-3  1bit each 以后备用的 默认都为 0
    Opcode   4bit 帧类型，稍后细说
    Mask     1bit 掩码，是否加密数据，默认必须置为1 
    Payload  7bit 数据的长度
    Masking-key      1 or 4 bit 掩码
    Payload data     (x + y) bytes 数据
    Extension data   x bytes  扩展数据
    Application data y bytes  程序数据
```

## 4、WS和Socket的区别与联系

 Socket 其实并不是一个协议。它工作在 OSI 模型会话层（第5层），是为了方便大家直接使用更底层协议（一般是 TCP 或 UDP ）而存在的一个抽象层。Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口(API)。

![](..\..\static\pics\socket.jpg)

Socket通常也称作”套接字”，用于描述IP地址和端口，是一个通信链的句柄。网络上的两个程序通过一个双向的通讯连接实现数据的交换，这个双向链路的一端称为一个Socket，一个Socket由一个IP地址和一个端口号唯一确定。应用程序通常通过”套接字”向网络发出请求或者应答网络请求。

Socket在通讯过程中，服务端监听某个端口是否有连接请求，客户端向服务端发送连接请求，服务端收到连接请求向客户端发出接收消息，这样一个连接就建立起来了。客户端和服务端也都可以相互发送消息与对方进行通讯，直到双方连接断开。

所以基于WebSocket和基于Socket都可以开发出IM社交聊天类的app。

## 5、WS在JS中的方法

  客户端支持WebSocket的浏览器中，在创建socket后，可以通过onopen、onmessage、onclose和onerror四个事件对socket进行响应。WebSocket的所有操作都是采用**事件**的方式触发的，这样不会阻塞UI，是的UI有更快的响应时间，有更好的用户体验。

　　浏览器通过Javascript向服务器发出建立WebSocket连接的请求，连接建立后，客户端和服务器就可以通过TCP连接直接交换数据。当你获取WebSocket连接后，可以通多send()方法向服务器发送数据，可以通过onmessage事件接收服务器返回的数据。

        //申请一个WebSocket对象，参数是服务端地址，同http协议使用http://开头一样，WebSocket协议的url使用ws://开头，另外安全的WebSocket协议使用wss://开头
        var ws = new WebSocket("ws://localhost:8080");
        
        //当WebSocket创建成功时，触发onopen事件
        ws.onopen = function () {
            console.log("open");
            ws.send("hello"); //将消息发送到服务端
        }
        //当客户端收到服务端发来的消息时，触发onmessage事件，参数e.data包含server传递过来的数据
        ws.onmessage = function (e) {
            console.log(e.data);
        }
        //当客户端收到服务端发送的关闭连接请求时，触发onclose事件
        ws.onclose = function(e){
            console.log("close");
        }
        //如果出现连接、处理、接收、发送数据失败的时候触发onerror事件
        ws.onerror = function(e){
            console.log(error);
        }