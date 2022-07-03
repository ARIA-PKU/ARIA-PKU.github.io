---
title: Raft
date: 2022-06-28T23:43:36+08:00
lastmod: 2022-06-28T23:43:36+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/7_1/%E7%A9%BA.jpg

categories:
  - 分布式
tags:
  - MIT6.824
  - raft
# nolastmod: true
draft: false
---

Raft论文阅读与实践

Raft课程笔记整理。

<!--more-->

## 一、容错系统的模式

1、MR采用复制计算，依赖于单个master节点组织；

2、GFS采用复制数据，依赖于master节点选择的primary节点管理；

3、VMware FT 复制服务，依赖于test-and-set的方法选取primary节点。

它们全都依赖于单个实体来做关键决定，这样的好处在于可以**避免split brain**。（split brain的中文翻译是脑裂，听起来有点奇怪，后面还是都直接用英文吧）

## 二、分区与大多数投票

在分布式网络中，由于网络故障或者服务器崩溃，会产生分区的情况，分区可以理解为部分服务器可以互联，但是整体无法连通。

可以采用的方式是投票机制。

1、**服务器的个数需要为奇数**，这样可以确保大多数只有一个，而不会产生相同个数的分区。

2、**大多数指的是所有服务器中的大多数，而不是存活的大多数。**如果系统中有2f + 1台服务器，则最多可以容忍f台服务器无法连接。

3、大多数的另一个功能在于两次选举服务器的相交。因为需要超过一半的服务器，所以两次选举中必然有重叠的服务器，这样**可以通过重叠的服务器传递先前决策的信息**。

## 三、RAFT的概述

raft是一种共识算法，是实际解决分布式中的一致性问题的一种算法。

命令的执行流程：

1、client会向leader的k/v层发送命令；

2、leader会将命令添加到日志中，并向follwers发送AppendEntries RPC；

3、followers收到后会将命令追加到日志中；

4、leader会等待大多数的回复（这里大多数会计算自己），如果大多数都返回加入日志，则entry会被commit；

5、leader执行命令，回复client，并在下一个AppendEntries RPC中将这个entry发送，followers发现该条entry已经commit则会执行该条目。

## 四、日志

### 4.1、日志的作用

1、日志可以记录命令的顺序。可以确保副本按照相同的执行顺序执行，也可以帮助人leader确保其他follwer的日志一致。

2、日志可以存储零时的命令，等到收到committed时候再执行。

3、日志也可以使得leader重新发送命令给followers

4、日志持久化命令，可以在重启后重新执行。

### 4.2、日志是强一致的吗

不是，一些副本的日志滞后。但是它们会最终一致，提交机制确保服务器只会执行一致稳定的条目。

## 五、leader选举

### 5.1、为什么需要leader

为了确保所有副本能够以相同的顺序执行相同的命令。（在Paxos等设计中是没有leader的）

### 5.2、raft为leaders编号

新的leader -> 新的term(任期)

一个任期内最多只有一个leader，也可能没有leader；

这个编号帮助服务器确定最新的leader而不是被取代的leader。

### 5.3、什么时候raft开始leader选举

当它在超过“election timeout”的时间后，没有收到当前leader的信息时候，该节点会增加自己的任期编号，然后收集投票。

这会导致不必要的选举，但是这是一种慢但是安全的方式。旧leader此时仍认为自己是leader。

### 5.4、如何确保任期至多一个leader

leader当选必须要获得大多数服务器的"yes"，每一个服务器在一个term内只能投票一次：

1）如果是candidate则只会投给自己；

2）如果不是candidate则会投给收到请求的第一个服务器（服务器还要满足论文中Figure 2中的规则）。

### 5.5、服务器如何得知新选出的leader

新的leader收到majority投票后当选，其他的服务器会收到带有更大term number的AppendEntries，即来自新leader的信息。

心跳会suppress所有新的选举。

### 5.6、选举失败后会发生什么

大多数服务器不可达或者多个candidate瓜分了vote会产生选举失败的情况。

在经过新的timeout仍然没有心跳接受，会提交新的选举并产生更高的term number选举。

### 5.7、raft如何避免split votes

采用随机延迟的方式，这个延迟的时间需要足够长，以确保能完成当前选举，否则选举过程中会不断有新的candidate分裂选票。

随机延迟是网络协议中的常见模式。

### 5.8、选举的timeout的选择

1、至少有是心跳间隔的两倍以上（防止心跳丢失情况），这样可以避免不必要的选举。

2、随机的部分要足够长，使得在下一个candidate开始前成功，避免split votes的情况。

3、也要足够短，使其对failure做出快速反应，避免长时间停顿。

### 5.9、旧leader没有意识到新leader选举成功会怎么样

也许旧leader没有看到选举信息，或者在少数网络分区。

新的leader产生会使得大多数服务器都增加了 currentTerm，所以旧leader（带旧任期编号）无法获得 AppendEntries 的多数，所以旧leader不会提交或执行任何新的日志条目，因此没有发生split brain，但少数人可能会接受旧服务器的 AppendEntries，所以日志可能会在旧期限结束时diverge。



## 参考资料：

1、[6.824 2020 Lecture 6: Raft (1)](http://nil.csail.mit.edu/6.824/2020/notes/l-raft.txt)

2、[6.824 2020 Lecture 7: Raft (2)](http://nil.csail.mit.edu/6.824/2020/notes/l-raft2.txt)