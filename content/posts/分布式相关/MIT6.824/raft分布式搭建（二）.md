---
title: Raft分布式搭建（二）
date: 2022-07-14T14:35:08+08:00
lastmod: 2022-07-14T14:35:08+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/7_1/%E6%B5%81%E6%98%9F.jpg

categories:
  - 分布式
tags:
  - raft
# nolastmod: true
draft: false
---

基于MIT6.824搭建的分布式系统，记录一下lab3的实现。

<!--more-->

## 背景介绍

在（一）中的lab2中，实现了raft系统的选举，日志同步与持久化。lab3则是在lab2的基础上，实现基于raft的高可用的KV存储服务，支持相应的操作并实现日志压缩的功能。

## 线性化语义

在论文中，作者详细介绍了线性化语义的概念，这里主要参考网上的对于具体的通用方法的[总结](https://www.zhihu.com/question/278551592)。

基本解决思路：

- 为每个client设置一个唯一的ID，对每个client的proposal进行顺序递增的编号，根据client的id与proposal的编号可以记录应用状态机后的结果。
- 当一个proposal超时，重试不会增加proposal编号。
- 当一个proposal被成功提交并返回，client会递增proposal编号，并记录成功的编号。raft节点收到请求后会记录最大proposal编号，删掉该client小于该编号之前的内容。如果该proposal已经被应用过，则不再应用。即**确保所有的日志可以被commit多次，但是只能被apply一次。**
- 系统可以采用LRU策略维护一定数量的client，如果client已被淘汰，则直接返回fail。
- client的信息即配套的proposal的结果，需要在raft组内进行一致性维护。上述数据结构在快照时也需要保存回复。

## Client端实现

按照以上的原则，通过clientId与commandId确定一个应用的内容。

```
func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
	return &Clerk{
		servers:	servers,
		leaderId:	0,
		clientId:	nrand(),
		commandId:	0,
	}
}
```

在客户端一共有三种操作，get、put和append。在proposal成功后，commandId + 1，否则不改变：

```
func (ck *Clerk) Get(key string) string {
	
	value := ck.Command(&CommandRequest{
		Key: key, 
		Op:	 OpGet,
	})
	return value
}

func (ck *Clerk) Put(key string, value string) {
	ck.Command(&CommandRequest{
		Key: key, 
		Value: value,
		Op:	 OpPut,
	})
}

func (ck *Clerk) Append(key string, value string) {
	ck.Command(&CommandRequest{
		Key: key, 
		Value: value,
		Op:	 OpAppend,
	})
}

func (ck *Clerk) Command(request *CommandRequest) string {
	request.ClientId, request.CommandId = ck.clientId, ck.commandId
	for {
		reply := &CommandReply{}
		if !ck.servers[ck.leaderId].Call("KVServer.Command", request, &reply) || reply.Err == ErrWrongLeader || reply.Err == ErrTimeOut {
			ck.leaderId = (ck.leaderId + 1) % int64(len(ck.servers))
			continue
		}
		ck.commandId ++
		return reply.Value
	}
}
```

## Server端的实现

对每个server端的节点维护以下数据结构：

```
type KVServer struct {
	mu      sync.RWMutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg
	dead    int32 // set by Kill()

	maxraftstate 	int // snapshot if log grows this big
	lastApplied  	int // record the lastApplied to prevent stateMachine from rollback

	stateMachine	KVStateMachine
	lastOperation	map[int64]OperationHistory
	notifyChans		map[int]chan *CommandReply	// notify client goroutine by applier goroutine to response
}
```

以下就是详细的流程介绍。

### 接收到客户端的请求后如何处理？

处理流程如下：（需要尤其注意的是及时return，并且在return之前一定要Unlock锁）。

- 对于非get操作，需要按照线性化语义中的思路，先判断是否已经应用过，如果应用过则更新返回即可。
- 对于上面的duplicateRPC的判断，使用的是server节点的lastOperation的数据结构，它会存储除了get以外的最大commandId的操作，通过比较防止重复apply
- 通过raft层的start命令，将command放到applyCh中并返回命令的index编号与是否为leader。如果当前节点不是leader则返回wrongLeader给客户端。
- 同时，会有一个并发的协程coordinator处理applyCh中的命令（类似于raft底层的设计，具体会在下面介绍），并将reply的结果放入notifyChans通道中。
- 生成一个ch通道，通过select来监听结果，如果超时没有返回结果，则返回Timeout错误。
- 最后，异步删除该通道，异步为了提高并发，删除为了节省空间开销。

```
func (kv *KVServer) Command(request *CommandRequest, reply *CommandReply) {
	kv.mu.RLock()
	if request.Op != OpGet && kv.isDuplicateRPC(request.ClientId, request.CommandId) {
		lastReply := kv.lastOperation[request.ClientId].LastReply
		reply.Value, reply.Err = lastReply.Value, lastReply.Err
		kv.mu.RUnlock()
		return
	}
	kv.mu.RUnlock()

	index, _, isLeader := kv.rf.Start(Command{request})
	if !isLeader {
		reply.Err = ErrWrongLeader
		return
	}

	kv.mu.Lock()
	ch := kv.getNotifyChan(index)
	kv.mu.Unlock()

	select {
	case result := <-ch:
		reply.Value, reply.Err = result.Value, result.Err
	case <-time.After(ExecuteTimeout):
		reply.Err = ErrTimeOut
	}
	go func() {
		kv.mu.Lock()
		kv.removeOutOfDateNotifyChan(index)
		kv.mu.Unlock()
	}()
}

```

### 异步处理ApplyMsg

类似于raft底层的设计思路，为了提高并发量，实现高可用，开启一个协程，并行处理applyCh中发送过来的请求。执行的要点如下：

- 这里的killed()是底层判断该server是否正常运行的函数，如果server已经down掉则无需操作。
- applych中的ApplyMsg有两类，一类是正常的command，另一类是快照日志。
- 对于command命令而言：
  - 如果index小于applied的index则直接跳过
  - 如果非get操作，而且是duplicateRPC则返回上一次操作即可
  - 否则，将操作应用到状态机中，如果是非get操作，将reply与commandId存储到lastOperation中
  - 确认当前节点仍是leader，request的term也是当前term，确保操作没有过期。则将reply放入notifyChan对应的index的通道中，在上面command命令中取出调用。
  - 判断是否需要日志压缩，判断标准是当前日志大小是否大于server数据结构中的maxraftstate。如果需要，则进行日志压缩
- 对于快照命令：
  - 在raft层判断快照是否过期，没有过期则更新raft层快照以及日志，然后在kv层更新状态机和lastOperation，即将
  - 最后将kv层的lastApplied更新成快照的Index

```
func (kv *KVServer) coordinator() {
	for kv.killed() == false {
		select {
		case msg := <-kv.applyCh:
			if msg.CommandValid {
				kv.mu.Lock()
				if msg.CommandIndex <= kv.lastApplied {
					DPrintf("{Node: %v} does not apply duplicated message", kv.rf.Me())
					kv.mu.Unlock()
					continue
				}
				kv.lastApplied = msg.CommandIndex
				var reply *CommandReply
				cmd := msg.Command.(Command)
				if cmd.Op != OpGet  && kv.isDuplicateRPC(cmd.ClientId, cmd.CommandId) {
					reply = kv.lastOperation[cmd.ClientId].LastReply
				} else {
					reply = kv.applyLogToStateMachine(cmd)
					if cmd.Op != OpGet {
						kv.lastOperation[cmd.ClientId] = OperationHistory{cmd.CommandId, reply}
					}
				}

				if currentTerm, isLeader := kv.rf.GetState(); isLeader && msg.CommandTerm == currentTerm {
					ch := kv.getNotifyChan(msg.CommandIndex)
					ch <- reply
				}
				
				if kv.needSnapshot() {
					kv.takeSnapshot(msg.CommandIndex)
				}

				kv.mu.Unlock()
			} else if msg.SnapshotValid{
				// recover from snapshot
				kv.mu.Lock()
				if kv.rf.CondInstallSnapshot(msg.SnapshotTerm, msg.SnapshotIndex, msg.Snapshot) {
					kv.restoreSnapshot(msg.Snapshot)
					kv.lastApplied = msg.SnapshotIndex
				}
				kv.mu.Unlock()
			} else {
				panic("unexpected message in coordinator")
			}
		}
	}
}
```

### kv层的日志压缩

kv层的日志压缩是在raft的基础上进行，根据前面的分析，kv层的日志压缩，不仅需要保存状态机的快照还需要保存lastOperation的快照，记录每个client的最新一次put/append操作的内容。