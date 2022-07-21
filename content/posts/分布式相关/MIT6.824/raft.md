---
title: Raft
date: 2022-06-28T23:43:36+08:00
lastmod: 2022-06-28T23:43:36+08:00

cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/7_1/%E7%A9%BA.jpg

categories:
  - 分布式
tags:
  - MIT6.824
  - raft
# nolastmod: true
draft: false
---

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

2、日志可以存储临时的命令，等到收到committed时候再执行。

3、日志也可以使得leader重新发送命令给followers

4、日志持久化命令，可以在重启后重新执行。

### 4.2、日志是强一致的吗

不是，一些副本的日志滞后。但是它们会最终一致，提交机制确保服务器只会执行一致稳定的条目。

## 五、leader选举

### 5.1、为什么需要leader

为了确保所有副本能够以相同的顺序执行相同的命令。（在Paxos等设计中是没有leader的）

### 5.2、raft记录leaders编号

新的leader -> 新的term(任期)

一个任期内最多只有一个leader，也可能没有leader；

这个编号帮助服务器确定最新的leader而不是被取代的leader。

### 5.3、什么时候raft开始leader选举

当它在超过“election timeout”的时间后，没有收到当前leader的信息时候，该节点会增加自己的任期编号，然后收集投票。

这会导致不必要的选举，但是这是一种慢但是安全的方式。旧leader此时仍认为自己是leader。

### 5.4、如何确保一个term内至多一个leader

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

## 六、日志进阶

以下面的leader崩溃为例：

```
        10 11 12 13  <- log entry #
    S1:  3
    S2:  3  3  4
    S3:  3  3  5
```

raft是通过使得followers采用新的leader日志来确保一致的。

日志对S2同步的流程如下：

1）当前S3被选为了term 6的leader，此时它需要向follower发送同步日志；

2）S3发送AppendEntries给S2，包含信息有：entry 13的内容，preLogIndex = 12, prevLogTerm = 5;

3）S2收到后，发现preLogIndex和prevLogTerm无法匹配，返回false

4）S3 将nextIndex[S2]降低到12， 然后发送AppendEntries，包含：

entry 12和13的内容，prevLogIndex=11, prevLogTerm=3；

5）S2删除自己entry 12的内容，同步新的日志。

同理，可以同步S1的日志。

总结一下，所有存活的follwers会删除结尾和leader不同的日志，然后在相同的位置之后，添加leader中的日志内容。

### Question：为什么可以遗忘S2的index=12， term=4的entry？

回滚这个entry的前提是该entry未被提交，如果以已经被提交则无法回滚该entry。因此，在选举新的leader的时候，需要确保被选中的leader拥有所有被提交的entry日志。

### 谁是合适的leader？

**错误的猜想：选择日志最长的服务器**

这里一种直观但是错误的猜想，举例如下：

```
    S1: 5 6 7
    S2: 5 8
    S3: 5 8
```

以上情况发生的条件：

1）在term 6的时候，S1作为leader，崩溃重启；在term 7又当选leader，再次崩溃，并且在两次崩溃之前都仅把日志保存到了自己的服务器上，而没有发送出去。

2）S2和S3经过选举后，假定确认S2作为leader，S2将日志发送给S3，再次崩溃。（这里S2的下一次term是8，因为S2或S3在给S1投票的时候，收到了term 7，因此必然会加一）。

在以上的情况下，谁应该成为新的leader？

S1的日志最长，但是entry 8的内容可能已经提交了，因此新的leader只能从S2和S3中选择，而不能选择最长日志。

在论文5.4.1小节的结尾解释了“election restriction":

RequestVote处理程序，只会投票给”at least as up to date“的candidate：

1）candidate的最后一个log的term更高；

2）candidate的最后一个log的term相同，但日志更长。

其实，这也确保了整个选举只会选出带有最新commit的日志的节点作为下一任leader。

### 如何进行日志的快速回滚

论文中的Figure 2给出的设计是，每次RPC回退一个entry，这对于落后大量日志的节点来说会非常的慢，在论文的5.3小节的某位概述了一种方案。论文中没有给出细节，但是lecture中老师给出了一种具体实现的猜测：

```
      Case 1      Case 2       Case 3
  S1: 4 5 5       4 4 4        4
  S2: 4 6 6 6 or  4 6 6 6  or  4 6 6 6
```

S2是term 6的leader，此时S1恢复工作，S2发送AppendEntries(以下简称AE)，AE的prevLogTerm=6；

S1发送的拒绝信息里包括：

1）XTerm： 存在冲突entry的term（如果存在）

2）XIndex：该term下的第一条entry（如果存在）

3）Xlen： 日志的长度

 case 1（leader没有 XTerm）：

​    nextIndex = XIndex

 case 2（领导者有 XTerm）：

​    nextIndex = 领导者在 XTerm 中的最后一个entry

 case 3（follwer日志太短）：

​    nextIndex = XLen

### 日志持久化

**如果服务器崩溃并且重启，raft需要将哪些内容持久化？**

在论文的Figure 2中列举出需要持久化的状态（需要在每次change的时候或在RPC发送或回复的时候持久化）：

1）log[]，日志。

在leader的majority发送的服务器中，这些服务器需要记住commited的log entry，这样能确保在未来选举的leader能看到所有commited的内容。log是唯一记录这些状态的数据结构。而且重启的时候，需要通过log来重新执行所有命令。

2) currentTerm，当前term。

记录currentTerm，可以确保term是递增的，从而确保每个term中至多只有一个leader。只有log不能确保的，实际的例子上面合适leader中最长日志不能作为leader中给出了。

3）votedFor，投票给了谁。

考虑以下情况：

如果服务器崩溃前已经投票，在恢复后没有记录是否投票，则会重新投票，这样会发生多个candidate当选的情况。

因此，必须将投票给了谁持久化。

**其他内容为什么不需要持久化？**

例如commitIndex, lastApplied, next/matchIndex[]这些内容，有的可以直接在log中恢复，有的即使丢失也没有关系。

持久化是系统的性能瓶颈，硬盘的写入需要10ms，SSD的写入需要0.1ms；

所以，持久化的操作需要限制在100到10000次 ops/sec;（另一个瓶颈是RPC，在LAN中花费 << 1ms每次）

有许多解决持久化慢的技巧：

1）每次硬盘写入的时候批处理多条新的 log entries

2）持久化到RAM中，而不是disk中

**一个服务（例如k/v服务器）如何在崩溃重启后恢复它的状态？**

直接重新执行raft的整个持久化日志，因为lastApplied（已经执行日志的最大编号）是volatile（易失的），因此直接从0开始。这就是论文中Figure 2给出的方案。

但是对于长时间运行的系统，重新执行日志会太缓慢，因此可以采用快照加重新执行快照后的日志的方法 。

### 日志压缩与快照

可能遇到的问题：日志可能会变得非常庞大——远大于存储原本数据的状态。这样在重启replay日志或者发送日志给新的server会花费大量时间。

服务器可以采用快照的方式，存储所有状态，因为原本的状态所占的大小会远小于日志大小。

**哪些entries不能被server丢弃？**

1）未被执行的entries，没有在数据库中被记录下来

2）未提交的entries，可能是leader的majority一部分

**增加快照后崩溃重启如何执行？**

服务会从disk中读取快照；

raft会从disk中读取持久化的日志；

服务会告诉raft将lastApplied设置成快照中保存的最后的日志索引，避免从头执行日志。

**problem：如果follwer的日志在leader日志开始之前就结束怎么办？**

详细点说，就是follwer结束的位置在leader的快照中，如果快照中的日志已经被清除，则leader无法通过AE RPCs同步日志。

因此会采用InstallSnapshot RPC，即发送其快照及快照后的日志给follower。

**philosophical note：**

存储状态（即存储的内容）和操作历史是等价的，两者可以选择一个进行保存或者传递消息，这个思想在很多地方都能用到。

**practical notes：**

raft的快照在存储量较小的时候是合理的；

但是对于大的数据库，例如，复制千兆的数据就会非常缓慢；这种情况，可能直接将数据存放在磁盘中B树的数据库中，而不是显式的快照中会更好；

不过这对于处理已经滞后的副本很困难，因此leader应该保存一段时间的日志，或者记录下被更新过的部分。

## 七、线性化语义

线性化语义的提出是为了解决在用户请求超时的情况下，重试导致的多次应用的问题。

#### 重复RPC检测

如果put或get超时，即call函数返回false，对服务器而言有两种情况：

1）服务器崩溃，请求未被执行，则重新发送即可；

2）服务器已经执行，但是请求丢失，这种情况下重新发送会很危险。

**解决方法：**

让服务端检测client发送的重复请求

client为每个请求选择一个ID， 如果是重复发送的请求则使用同一个ID；

k/v服务维护一个由ID作为索引的表，对每个RPC请求执行后记录值；

如果接收到重复ID则返回表中的结果。

#### 如何使得duplicate表较小？

1）为每个客户端保存一个表格entry，而不是每个RPC

2）每个客户端一次只能有一个超时的RPC

3）每个客户段按顺序编号RPC，当收到客户端的#10RPC，删掉客户端更小的entry，即客户端不会请求更小的RPC了

**细节：**

每个client需要一个唯一的ID，在duplicate表中，通过client的ID索引;

RPC处理先检查表，只有收到编号大于表entry的时候，执行Start();

applyCh中出现操作后，更新client的表entry并唤醒对应的RPC处理程序（如果存在）。

#### 新的leader如何获取duplicate表？

所有的复制都应该在执行后更新到表中，因此在成为leader前表中内容已经存在。

#### 如果服务器崩溃，如何恢复duplicate表？

如果没有快照，则重播能填充表的日志；

如果有快照，快照中会包含表的复制。

## 参考资料

1、[6.824 2020 Lecture 6: Raft (1)](http://nil.csail.mit.edu/6.824/2020/notes/l-raft.txt)

2、[6.824 2020 Lecture 7: Raft (2)](http://nil.csail.mit.edu/6.824/2020/notes/l-raft2.txt)

3、https://www.zhihu.com/question/278551592