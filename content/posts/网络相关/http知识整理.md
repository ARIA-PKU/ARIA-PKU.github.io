---
title: Http协议整理
date: 2022-05-16T22:00:56+08:00
lastmod: 2022-05-16T22:00:56+08:00
author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: https://oss.surfaroundtheworld.top/blog-pictures/cup_green.jpg
# images:
#   - /img/cover.jpg
categories:
  - 计算机网络
tags:
  - http
# nolastmod: true
draft: false
---

对http1.0，http1.1，http2.0，http3.0的整理总结

<!--more-->

# Http

http是一种无状态、无连接的应用层协议，主要靠session和cookie进行状态记录，在3.0之前都是基于TCP协议的

## 1、HTTP1.0

早期版本，默认是短连接，每次都需要重新建立TCP连接，无法复用之前建立的连接。

## 2、HTTP1.1

继承了HTTP1.0的简单，也解决了性能上的一些问题。

1）**长连接**

试想一下：请求一张图片，新开一个连接，请求一个CSS文件，新开一个连接，请求一个JS文件，新开一个连接。HTTP协议是基于TCP的，TCP每次都要经过**三次握手，四次挥手，慢启动**...这都需要去消耗我们非常多的资源的！

HTTP1.1增加Connection字段，通过设置**Keep-Alive**保持HTTP连接不断卡。避免每次客户端与服务器请求都要重复建立释放建立TCP连接。提高了网络的利用率。

如果客户端想关闭HTTP连接，可以在请求头中携带Connection:false来告知服务器关闭请求。

2）**管道化（pipelining）**

HTTP 1.1管线化(pipelining)理论，客户端可以同时发出多个HTTP请求，而不用一个个等待响应之后再请求

- 注意：这个pipelining仅仅是**限于理论场景下**，大部分桌面浏览器仍然会**选择默认关闭**HTTP pipelining！
- 所以现在使用HTTP1.1协议的应用，都是有**可能会开多个TCP连接**的！

## 3、HTTP2.0

对1.x协议语意的完全兼容，2.0协议是在1.x基础上的升级而不是重写，1.x协议的方法，状态及api在2.0协议里是一样的。

性能的大幅提升，2.0协议重点是对终端用户的感知延迟、网络及服务器资源的使用等性能的优化。

1） **二进制分帧（Binary Format）- http2.0的基石**

http2.0之所以能够突破http1.X标准的性能限制，改进传输性能，实现低延迟和高吞吐量，就是因为其新增了二进制分帧层。

在二进制分帧层上，http2.0会将所有传输信息分割为更小的消息和帧，并对它们采用二进制格式的编码将其封装，新增的二进制分帧层同时也能够保证http的各种动词，方法，首部都不受影响，兼容上一代http标准。其中，http1.X中的首部信息header封装到Headers帧中，而request body将被封装到Data帧中。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qMicvibdvl7p3saqATlDB6k03rzfHDibnsKhgWzwkhce01AmXwchJJhrYOBWs4NXspYY9L3wziaLxjoHYxNGoAL70g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

2） **多路复用 (Multiplexing) / 连接共享**

在http1.1中，浏览器客户端在同一时间，针对同一域名下的请求有一定数量的限制，超过限制数目的请求会被阻塞。这也是为何一些站点会有多个静态资源 CDN 域名的原因之一。

而http2.0中的多路复用优化了这一性能。多路复用允许同时通过单一的http/2 连接发起多重的请求-响应消息。有了新的分帧机制后，http/2 不再依赖多个TCP连接去实现多流并行了。每个数据流都拆分成很多互不依赖的帧，而这些帧可以交错（乱序发送），还可以分优先级，最后再在另一端把它们重新组合起来。

http 2.0 连接都是持久化的，而且客户端与服务器之间也只需要一个连接（每个域名一个连接）即可。http2连接可以承载数十或数百个流的复用，多路复用意味着来自很多流的数据包能够混合在一起通过同样连接传输。当到达终点时，再根据不同帧首部的流标识符重新连接将不同的数据流进行组装。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qMicvibdvl7p3saqATlDB6k03rzfHDibnsKUD0icQCcP5QOx2iaWUhDdZE974qRK6rmy98GPlk7DzdxDk1utGAEOvqQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



上图展示了一个连接上的多个传输数据流：客户端向服务端传输数据帧stream5，同时服务端向客户端乱序发送stream1和stream3。这次连接上有三个响应请求乱序并行交换。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qMicvibdvl7p3saqATlDB6k03rzfHDibnsKQgfiaAic60DmWkDib8LUhiaLmfibc8ICW0xVwyXmoxWqrwGwfFDox8xbUGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



上图就是http1.X和http2.0在传输数据时的区别。

3） **头部压缩（Header Compression）**

http1.x的头带有大量信息，而且每次都要重复发送。http/2使用encoder来减少需要传输的header大小，通讯双方各自缓存一份头部字段表，既避免了重复header的传输，又减小了需要传输的大小。

对于相同的数据，不再通过每次请求和响应发送，通信期间几乎不会改变通用键-值对(用户代理、可接受的媒体类型，等等)只需发送一次。

事实上,如果请求中不包含首部(例如对同一资源的轮询请求)，那么，首部开销就是零字节，此时所有首部都自动使用之前请求发送的首部。

如果首部发生了变化，则只需将变化的部分加入到header帧中，改变的部分会加入到头部字段表中，首部表在 http 2.0 的连接存续期内始终存在，由客户端和服务器共同渐进地更新。

需要注意的是，http 2.0关注的是首部压缩，而我们常用的gzip等是报文内容（body）的压缩，二者不仅不冲突，且能够一起达到更好的压缩效果。

http/2使用的是专门为首部压缩而设计的HPACK算法。压缩的原理就是用header字段表里的索引代替实际的header。

4）**请求优先级（Request Priorities）**

把http消息分为很多独立帧之后，就可以通过优化这些帧的交错和传输顺序进一步优化性能。每个流都可以带有一个31比特的优先值：0 表示最高优先级；2的31次方-1 表示最低优先级。

服务器可以根据流的优先级，控制资源分配（CPU、内存、带宽），而在响应数据准备好之后，优先将最高优先级的帧发送给客户端。高优先级的流都应该优先发送，但又不会绝对的。绝对地准守，可能又会引入首队阻塞的问题：高优先级的请求慢导致阻塞其他资源交付。

分配处理资源和客户端与服务器间的带宽，不同优先级的混合也是必须的。客户端会指定哪个流是最重要的，有一些依赖参数，这样一个流可以依赖另外一个流。优先级别可以在运行时动态改变，当用户滚动页面时，可以告诉浏览器哪个图像是最重要的，你也可以在一组流中进行优先筛选，能够突然抓住重点流。

```
优先级最高：主要的html

优先级高：CSS文件

优先级中：js文件

优先级低：图片

```

5) **服务端推送（Server Push）**

服务器可以对一个客户端请求发送多个响应，服务器向客户端推送资源无需客户端明确地请求。并且，服务端推送能把客户端所需要的资源伴随着index.html一起发送到客户端，省去了客户端重复请求的步骤。

正因为没有发起请求，建立连接等操作，所以静态资源通过服务端推送的方式可以极大地提升速度。Server Push 让 http1.x 时代使用内嵌资源的优化手段变得没有意义；如果一个请求是由你的主页发起的，服务器很可能会响应主页内容、logo 以及样式表，因为它知道客户端会用到这些东西，这相当于在一个 HTML 文档内集合了所有的资源。

不过与之相比，服务器推送还有一个很大的优势：可以缓存！也让在遵循同源的情况下，不同页面之间可以共享缓存资源成为可能。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qMicvibdvl7p3saqATlDB6k03rzfHDibnsKtnRc9hVCeoibb3wBlPyuNKqOPqsWS2ibwxh0tu3SsNiayJO1RiayxJ5Efw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



注意两点：

1、推送遵循同源策略；

2、这种服务端的推送是基于客户端的请求响应来确定的。

当服务端需要主动推送某个资源时，便会发送一个 Frame Type 为 PUSH_PROMISE 的 Frame，里面带了 PUSH 需要新建的 Stream ID。意思是告诉客户端：接下来我要用这个 ID 向你发送东西，客户端准备好接着。客户端解析 Frame 时，发现它是一个 PUSH_PROMISE 类型，便会准备接收服务端要推送的流。

6) **性能瓶颈**

启用http2.0后会给性能带来很大的提升，但同时也会带来新的性能瓶颈。因为现在所有的压力集中在底层一个TCP连接之上，TCP很可能就是下一个性能瓶颈，比如TCP分组的队首阻塞问题，单个TCP packet丢失导致整个连接阻塞，无法逃避，此时所有消息都会受到影响。未来，服务器端针对http 2.0下的TCP配置优化至关重要。

## 4、HTTP3.0

Google搞了一个基于UDP协议的QUIC协议，并且使用在了HTTP/3上， HTTP/3之前的名称为HTTP-over-QUIC。

早期Quic协议，存在IETF和Google两个版本，直到它被证实命名为HTTP3.0

> IETF的QUIC工作小组创造了QUIC传输协议。QUIC是一个使用UDP来替代TCP的协议。最初的时候，Google开始助力QUIC，其后QUIC更多地被叫做“HTTP/2-encrypted-over-UDP “。
> 社区中的人们已经使用非正式名称如iQUIC和gQUIC来指代这些不同版本的协议，以将QUIC协议与IETF和Google分开（因为它们在细节上差异很大）。通过“iQUIC”发送HTTP的协议被称为“HQ”（HTTP-over-QUIC）很长一段时间。
> 2018年11月7日，Litespeed的Dmitri宣布他们和Facebook已经成功地完成了两个HTTP/3实现之间的第一次互操作。Mike Bihop在该主题的HTTPBIS会话中的后续介绍可以在这里看到。会议结束时达成共识称新名称是HTTP/3！

**0-RTT — QUIC协议相比HTTP2.0的最大优势**

缓存当前会话的上下文，下次恢复会话的时候，只需要将之前的缓存传递给服务器，验证通过，就可以进行传输了。

**0-RTT**建连可以说是QUIC相比HTTP2最大的性能优势。

**什么是0-RTT建连**？

- 传输层0-RTT就能建立连接
- 加密层0-RTT就能建立加密连接

**多路复用**

QUIC基于UDP，一个连接上的多个stream之间没有依赖，即使丢包，只需要重发丢失的包即可，不需要重传整个连接。

**更好的移动端表现**

QUIC在移动端的表现比TCP好，因为TCP是基于IP识别连接，而QUIC是通过ID识别链接。 无论网络环境如何变化，只要ID不便，就能迅速重新连上。

**加密认证的根文 — 武装到牙齿**

TCP协议头没有经过任何加密和认证，在传输过程中很容易被中间网络设备篡改、注入和窃听。

QUIC的packet可以说武装到了牙齿，除了个别报文，比如PUBLIC_RESET和CHLO，所有报文头部都是经过认证的，报文Body都是经过加密的。

所以只要对 QUIC 做任何更改，接收端都能及时发现，有效地降低了安全风险。

**向前纠错机制**

QUIC协议有一个非常独特的特性，称为向前纠错（Foward Error Connec，FEC），每个数据包除了它本身的内容之外还包括了其他数据包的数据，因此少量的丢包可以通过其他包的冗余数据直接组装而无需重传。

向前纠错牺牲了每个数据包可以发送数据的上限，但是带来的提升大于丢包导致的数据重传，因为数据重传将会消耗更多的时间（包括确认数据包丢失，请求重传，等待新数据包等步骤的时间消耗）。

例如：

- 我总共发送三个包，协议会算出这个三个包的异或值并单独发出一个校验包，也就是总共发出了四个包。
- 当其中出现了非校验包丢失的情况，可以通过另外三个包计算出丢失的数据包的内容。
- 当然这种技术只能使用在丢失一个包的情况下，如果出现丢失多个包，就不能使用纠错机制了，只能使用重传的方式了。

## 5、参考文章

1、深入理解http2.0协议，看这篇就够了！https://mp.weixin.qq.com/s/a83_NE-ww36FZsy320MQFQ

2、https://www.zhihu.com/question/306768582/answer/595200654

3、快速掌握HTTP 1.0 1.1 2.0 3.0 的特点及其区别 https://zhuanlan.zhihu.com/p/266578819