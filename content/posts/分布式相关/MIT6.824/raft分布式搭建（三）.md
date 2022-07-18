---
title: Raft分布式搭建（三）
date: 2022-07-14T19:36:33+08:00
lastmod: 2022-07-14T19:36:33+08:00
cover: http://oss.surfaroundtheworld.top/blog-pictures/7_1/%E5%A4%9C.jpg
# images:
#   - /img/cover.jpg
categories:
  - 分布式
tags:
  - raft
# nolastmod: true
draft: false
---

MIT8.624中的lab4与challenges的实践，实现shard功能。

<!--more-->

## lab介绍

lab4在lab2的基础上更进一步，需要利用shard（分片）实现一个multi-raft的系统。每个副本组拥有一些shard，并且每个副本组包含很多server，构成一个raft系统对外提供服务，所有的副本组对外是并行工作的，因此这能大大提高系统的并发量。

lab4分为两个部分，第一个部分是实现一个shardmaster，提供一个高可用的集群配置管理服务，负责分片的迁移与配置的查询；第二个部分是实现高可用的分片容错KV存储服务，同时在challenge中要求副本组能够实现对shard的垃圾回收与配置更改期间对客户端请求的响应。

在总结实现之前，需要先整理一下论文中的重要知识点。

## 项目地址

lab的代码地址：https://github.com/ARIA-PKU/MIT6.824

## 论文中Figure 8讨论的是什么问题？会带来什么问题？

这个问题在lab2中遇到过，但是没有详细整理，这里重新梳理一下这个问题。在实际生产过程中存在，以及在lab4的实现过程中又遇到了这个问题的衍生问题。

Figure 8 用来说明为什么 Leader **不能提交之前任期的日志，只能通过提交自己任期的日志，从而间接提交之前任期的日志。**

以下根据参考文章1中的内容整理：

- 图（a）中，S1是leader，将日志复制到S2中
- 图（b）中，S1宕机，S5获得S3、S4、S5的选票称为leader，将日志index=2，term=3写入
- 图（c）中，S5写完后宕机，S1重新当选leader，此时currentTerm=4，**没有新的请求进来**，S1如果将index=2，term=2复制到S3后，S1会提交该日志（term=2并不是当前任期，这里讨论的是错误情况）。此时请求进来，S1刚刚写完index=3和term=4的日志，S1故障。
- 图（d）中，S5通过S2、S3、S4的投票当选leader，然后将index=2，term=3的日志复制到其他节点并提交，**此时index=2的日志提交了两次。**这是存在问题的，因为已经提交的日志不能覆盖。
- 图（e）中，讨论的则是正确的情况，S1在宕机前将term=4的日志复制到大多数服务器，正常运行。

因此，leader不能提交之前任期的日志，只有等新的请求进来，超过半数节点复制了先前日志之后，先前term的日志才能跟着新日志一起提交。

![](https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_8.png)

在加上这个约束之后不会重复提交了，但是如果一直没有新的请求进来，index=2的日志就一直无法提交，如果此时有读请求来请求index=2的内容，就会一直被阻塞，**这就会阻塞读请求。**

因此raft论文中又引入了no-op日志来解决这个问题，这个在etcd中有实现。

no-op日志只有index和term，command的信息为空，也需要持久化到磁盘与其他log一样。

具体实现是在leader当选后追加一条no-op日志，并将其复制到其他节点，no-op提交后，leader中哪些未提交的日志全都被间接提交，这样就能快速响应客户端的查询。

## 如何实现高可用的集群配置管理？

首先，需要理解整个配置的实现流程，先看lab4中给出的如下配置。shardmaster的目的就是维护这样的一个配置，并且需要记录下所有版本的配置信息，即把所有配置存放到一个数组里。

其中，Num代表的是配置的编号，每次查询都是请求获得最新的（即编号最大的）配置信息；Shards是一个数组，将shard编号与group id对应，通过给定的shard编号可以知道该shard属于哪个group；Groups是一个map，将group id与server对应，可以根据gid获得该group下的所有server节点。

```
// A configuration -- an assignment of shards to groups.
// Please don't change this.
type Config struct {
	Num    int              // config number
	Shards [NShards]int     // shard -> gid
	Groups map[int][]string // gid -> servers[]
}
```

再分析一下lab中需要完成的四个操作：

- Join：增加一些新的groups（即上面的Groups数据结构），并且要求shard在各个group之间尽可能均匀。增加group就是增加raft集群，再将所有的shard均匀分配。
- Leave：删除一些groups，要求shard尽可能均匀。与Join相反的操作，将删除的groups的shard再分给其他存活的groups。
- Move：给定一个shard number和GID，将该shard移动到指定group中。lab中表示这个操作是为了测试软件。
- Query：询问指定配置Num下的配置，默认询问最新配置。

简单来说，集群配置的管理就是实现对config中shards数组与Groups的map进行高并发的KV存储操作，可以发现处理stateMachine处理过程比简单KV增加了一些操作外，其余的内容与lab3基本一致。同时考虑到配置的大小，甚至无需使用日志压缩的快照系统。

这里不再赘述lab3中的KV实现，关于如何使得shard在group中尽可能均匀，可以采用每次选取最多和最少分片的两个group，将最多的分一个给最少的，循环直至相差小于等于一。需要注意的细节是，在go中map是无序的，为了保证各个节点apply到状态机的结果一致，现将map按key排序再寻找最大最小值即可。

```
// Join(servers) -- add a set of groups (gid -> server-list mapping).
func (cf *MemoryConfigStateMachine) Join(groups map[int][]string) Err {
	lastConfig := cf.Configs[len(cf.Configs) - 1]
	newConfig := Config{
		Num:	len(cf.Configs),
		Shards:	lastConfig.Shards,
		Groups:	deepCopy(lastConfig.Groups),
	}
	for gid, servers := range groups {
		if _, ok := newConfig.Groups[gid]; !ok {
			newServers := make([]string, len(servers))
			copy(newServers, servers)
			newConfig.Groups[gid] = newServers
		}
	}
	g2s := Group2Shards(newConfig)
	for {
		source, target := GetGidWithMaxShards(g2s), GetGidWithMinShards(g2s)
		if source != 0 && len(g2s[source]) - len(g2s[target]) <= 1 {
			break
		}
		g2s[target] = append(g2s[target], g2s[source][0])
		g2s[source] = g2s[source][1:]
	}
	var newShards [NShards]int
	for gid, shards := range g2s {
		for _, shard := range shards {
			newShards[shard] = gid
		}
	}
	newConfig.Shards = newShards
	cf.Configs = append(cf.Configs, newConfig)
	return OK
}

func GetGidWithMinShards(g2s map[int][]int) int {
	var keys []int
	for k := range g2s {
		keys = append(keys, k)
	}
	sort.Ints(keys)
	index, min := -1, NShards + 1
	for _, gid := range keys {
		if gid != 0 && len(g2s[gid]) < min {
			index, min = gid, len(g2s[gid])
		}
	}
	return index
}

func GetGidWithMaxShards(g2s map[int][]int) int {
	// left shards by operation Leave will be in position 0
	if shards, ok := g2s[0]; ok && len(shards) > 0 {
		return 0
	}

	var keys []int
	for k := range g2s {
		keys = append(keys, k)
	}
	sort.Ints(keys)
	index, max := -1, -1
	for _, gid := range keys {
		if len(g2s[gid]) > max {
			index, max = gid, len(g2s[gid])
		}
	}
	return index
}
```

Leave的操作类似，Move是简单地修改，Query则是直接返回最新配置：

```
func NewMemoryConfigStateMachine() *MemoryConfigStateMachine {
	cf := &MemoryConfigStateMachine{make([]Config, 1)}
	cf.Configs[0] = DefaultConfig()
	return cf
}

// Query(num) -> fetch Config # num, or latest config if num==-1.
func (cf *MemoryConfigStateMachine) Query(num int) (Config, Err) {
	if num < 0 || num >= len(cf.Configs) {
		return cf.Configs[len(cf.Configs) - 1], OK
	}
	return cf.Configs[num], OK
}
```

## 实现高可用的分片容错KV存储服务

在节点宕机，节点重启，网络分区等状况可能发生的条件下，实现存在分片的系统上进行KV存储服务并不算困难，但是需要在shard迁移过程中动态提供服务线性一致性服务，并且还要及时清理不属于本分片的数据需要注意很多细节。

### 客户端及RPC的设计

先从简单的部分出发，在client.go中发现，lab给了key与shard的直接映射函数，可以快速确定key对应的shard。

```
//
// which shard is a key in?
// please use this function,
// and please do not change it.
//
func key2shard(key string) int {
	shard := 0
	if len(key) > 0 {
		shard = int(key[0])
	}
	shard %= shardmaster.NShards
	return shard
}
```

线性一致性请求的部分与lab3一致，多了一个config信息：

```
type Clerk struct {
	sm       	*shardmaster.Clerk
	config   	shardmaster.Config
	makeEnd 	func(string) *labrpc.ClientEnd
	leaderIds	map[int]int   // store current raft's leaderId
	clientId 	int64
	commandId	int64
}
```

在RPC接口设计的时候，没有采用lab中给定的设计模式，为了使得操作更统一，将Get、Put和Append操作的接口统一设计：

```
type CommandRequest struct {
	Key			string
	Value		string
	Op			OperationOp
	ClientId	int64
	CommandId   int64
}

type CommandReply struct {
	Err 	Err
	Value 	string
}
```

command函数的设计与lab3中类似，寻找raft组的leader，发送请求，更新配置。

```
//
// fetch the current value for a key.
// returns "" if the key does not exist.
// keeps trying forever in the face of all other errors.
// You will have to modify this function.
//
func (ck *Clerk) Get(key string) string {
	return ck.Command(&CommandRequest{Key: key, Op: OpGet})
}

//
// shared by Put and Append.
// You will have to modify this function.
//
func (ck *Clerk) Put(key string, value string) {
	ck.Command(&CommandRequest{Key: key, Value: value, Op: OpPut})
}
func (ck *Clerk) Append(key string, value string) {
	ck.Command(&CommandRequest{Key: key, Value: value, Op: OpAppend})
}

func (ck *Clerk) Command(request *CommandRequest) string {
	request.ClientId, request.CommandId = ck.clientId, ck.commandId
	for {
		shard := key2shard(request.Key)
		gid := ck.config.Shards[shard]
		if servers, ok := ck.config.Groups[gid]; ok {
			if _, ok = ck.leaderIds[gid]; !ok {
				ck.leaderIds[gid] = 0
			}
			preLeaderId := ck.leaderIds[gid]
			newLeaderId := preLeaderId
			for {
				var reply CommandReply
				ok := ck.makeEnd(servers[newLeaderId]).Call("ShardKV.Command", request, &reply)
				if ok && (reply.Err == OK || reply.Err == ErrNoKey) {
					ck.commandId ++
					return reply.Value
				} else if ok && reply.Err == ErrWrongGroup {
					// the shard's group may have changed
					break
				} else {
					newLeaderId = (newLeaderId + 1) % len(servers)
					if newLeaderId == preLeaderId {
						break
					}
					continue
				}
			}	
		}
		time.Sleep(100 * time.Millisecond)
		ck.config = ck.sm.Query(-1)
	}
}
```

调用服务端的command请求大体与lab3类似，但是增加了shard状态判断的步骤，shard状态的判断主要是为了实现challege中的垃圾回收功能。

```
// deal with the command from client
func (kv *ShardKV) Command(request *CommandRequest, reply *CommandReply) {
	kv.mu.RLock()
	// if get the duplicate rpc, return result without the participation of raft layer
	if request.Op != OpGet && kv.isDuplicateRequest(request.ClientId, request.CommandId) {
		lastReply := kv.lastOperations[request.ClientId].LastReply
		reply.Value, reply.Err = lastReply.Value, lastReply.Err
		kv.mu.RUnlock()
		return
	}
	// 
	if !kv.canServe(key2shard(request.Key)) {
		reply.Err = ErrWrongGroup
		kv.mu.RUnlock()
		return
	}
	kv.mu.RUnlock()
	kv.Execute(NewOperationCommand(request), reply)
}
```

### shard的数据结构的设计

shard是存放最终KV值的位置，每个server会保存对应Group负责的shard的数据，最简单的设计就是将shard设计为一个存放KV值的map，但是这样无法实现垃圾回收与配置更改期间的动态请求返回了。为了提高可用性，需要将每个shard独立起来，因此为每个shard增加一个Status信息，开启多个协程异步处理shard的维护。

```
type Shard struct {
	KV		map[string]string
	Status	ShardStatus
}
```

可以根据需求，将shard的状态分为以下四种：

- Serving，默认可用状态
- Pulling，不可用。会从上一个config该分片所在的raft组中拉取数据
- BePulling，不可用。当前config下，该分片不属于当前raft，会在拉取完后进行垃圾回收
- GCing，可用。注意这里的GCing并不是代表当前raft组下的分片正在进行GC，而是指当前分片在上个config下的所在raft组中的分片信息需要被GC。GC成功后，当前分片会变为serving，因此GCing状态也是可用的。

### 服务端的整体设计

服务端的每一个节点是属于各自Group的一个节点，除了与基本raft相同的数据结构外，还需要保存GID来记录自己所属的Group，stateMachines记录shardID与shard的对应（这里的shard是自己设计的数据结构，后面会介绍）以及lastOperations作为去重表。

```
type ShardKV struct {
	mu           sync.RWMutex
	dead         int32
	rf           *raft.Raft
	applyCh      chan raft.ApplyMsg
	makeEnd      func(string) *labrpc.ClientEnd
	gid          int
	masters      []*labrpc.ClientEnd
	sm			 *shardmaster.Clerk

	maxRaftState 	int // snapshot if log grows this big
	lastApplied 	int	// record the lastApplied to prevent StateMachine from rollback

	lastConfig		shardmaster.Config
	currentConfig	shardmaster.Config

	stateMachines	map[int]*Shard		// shardId -> Shard
	lastOperations	map[int64]OperationContext	
	notifyChans		map[int]chan *CommandReply  // notify applier's goroutine to reply
}
```

在启动server的时候，除了初始化相关变量外，还开启了applier与四个serve协程，既为了提高整体并发度，也为了实现lab中的challenge要求。后面会逐一介绍各个协程的作用。

```
func StartServer(servers []*labrpc.ClientEnd, me int, persister *raft.Persister, maxraftstate int, gid int, masters []*labrpc.ClientEnd, make_end func(string) *labrpc.ClientEnd) *ShardKV {
	// call labgob.Register on structures you want
	// Go's RPC library to marshall/unmarshall.
	labgob.Register(Command{})
	labgob.Register(ShardOperationRequest{})
	labgob.Register(ShardOperationReply{})
	labgob.Register(shardmaster.Config{})
	labgob.Register(CommandRequest{})

	applyCh := make(chan raft.ApplyMsg)

	kv := &ShardKV {
		maxRaftState: 	maxraftstate,
		makeEnd:		make_end,
		gid:			gid,
		sm:				shardmaster.MakeClerk(masters),
		applyCh:		applyCh,
		dead:			0,
		rf:				raft.Make(servers, me, persister, applyCh),

		lastApplied:	0,
		currentConfig:	shardmaster.DefaultConfig(),
		lastConfig:		shardmaster.DefaultConfig(),
		notifyChans:	make(map[int]chan *CommandReply),
		stateMachines:	make(map[int]*Shard),
		lastOperations:	make(map[int64]OperationContext),
	}

	kv.restoreSnapshot(persister.ReadSnapshot())

	// start a goroutine to apply committed to stateMachine
	go kv.applier()
	
	// start a goroutine to update config
	go kv.Serve(kv.configureAction, ConfigureTimeout)

	// start a goroutine to check migration
	go kv.Serve(kv.migrationAction, MigrationTimeout)

	// start a goroutine to check gc
	go kv.Serve(kv.gcAction, GCTimeout)

	// start a goroutine to solve empty entry problems
	go kv.Serve(kv.checkEntryInCurrentTermAction, EmptyEntryDetectorTimeout)

	return kv
}
```

### 定时拉取配置信息

需要强调的是，拉取配置，迁移，垃圾回收和发送no-op日志都是leader才能进行的，否则会引起brain split问题。

**需要注意的是操作最终是被放入applyCh中，等待applier协程统一处理，这样可以确保高并发且安全地处理。以下定时执行的所有操作都是，这个的实现类似在raft底层的实现。**

Serve是一个定期执行函数：

```
func (kv *ShardKV) Serve(action func(), timeout time.Duration) {
	for kv.killed() == false {
		if _, isLeader := kv.rf.GetState(); isLeader {
			action()
		}
		time.Sleep(timeout)
	}
}
```

当所有shard都处于serving状态时，说明需要拉取新的配置，拉取配置的等待时间需要大于迁移和GC的时间。因此，将迁移和GC的等待时间设置为50ms，将拉取配置的等待时间设为100ms。

```
// fetch latest configuration
func (kv *ShardKV) configureAction() {
	canGetNextConfig := true
	kv.mu.RLock()

	for _, shard := range kv.stateMachines {
		if shard.Status != Serving {
			canGetNextConfig = false
			break
		}
	}

	currentConfigNum := kv.currentConfig.Num
	kv.mu.RUnlock()
	if canGetNextConfig {
		nextConfig := kv.sm.Query(currentConfigNum + 1)
		if nextConfig.Num == currentConfigNum + 1 {
			kv.Execute(NewConfigurationCommand(&nextConfig), &CommandReply{})
		}
	}
}
```

### 定时迁移数据

定时检查当前raft组下的shard处于pulling的shard，从其他的raft组异步拉取数据更新到本地。

```
// migrate related shards by pull
func (kv *ShardKV) migrationAction() {
	kv.mu.RLock()
	gid2shardIds := kv.getShardIdsByStatus(Pulling)
	var wg sync.WaitGroup
	for gid, shardIds := range gid2shardIds {
		wg.Add(1)
		go func(servers []string, configNum int, shardIds []int) {
			defer wg.Done()
			pullTaskRequest := ShardOperationRequest{configNum, shardIds}
			for _, server := range servers {
				var pullTaskReply ShardOperationReply
				serverEnd := kv.makeEnd(server) 
				if serverEnd.Call("ShardKV.GetShardsData", &pullTaskRequest, &pullTaskReply) && pullTaskReply.Err == OK {
					kv.Execute(NewInsertShardsCommand(&pullTaskReply), &CommandReply{})
				}
			}
		}(kv.lastConfig.Groups[gid], kv.currentConfig.Num, shardIds)
	}
	kv.mu.RUnlock()
	wg.Wait()
}
```

在被拉取方也需要leader进行处理。需要注意的是，如果被拉取raft集群的leader的config小于请求的config，则说明被请求方当前的config没有更新到最新，所以不能被正常拉取，返回ErrNotReady。否则，返回状态机内容并更新判重表。

```
func (kv *ShardKV) GetShardsData(request *ShardOperationRequest, reply *ShardOperationReply) {
	// only pull shards from leader
	if _, isLeader := kv.rf.GetState(); !isLeader {
		reply.Err = ErrWrongLeader
		return
	}
	kv.mu.RLock()
	defer kv.mu.RUnlock()

	// leader's config num is smaller than request's
	if kv.currentConfig.Num < request.ConfigNum {
		reply.Err = ErrNotReady
		return
	}

	reply.Shards = make(map[int]map[string]string)
	for _, shardId := range request.ShardIds {
		reply.Shards[shardId] = kv.stateMachines[shardId].deepCopy()
	}

	reply.LastOperations = make(map[int64]OperationContext)
	for clientID, operation := range kv.lastOperations {
		reply.LastOperations[clientID] = operation.deepCopy()
	}

	reply.ConfigNum, reply.Err = request.ConfigNum, OK
}
```

### 定时进行垃圾回收

与迁移类似，当前raft组发起请求，远程raft组清除垃圾后返回，然后本地将处理命令放入applyCh中等待统一处理。

```
func (kv *ShardKV) DeleteShardsData(request *ShardOperationRequest, reply *ShardOperationReply) {
	// only pull shards from leader
	if _, isLeader := kv.rf.GetState(); !isLeader {
		reply.Err = ErrWrongLeader
		return
	}

	kv.mu.RLock()
	// leader's config has updated
	if kv.currentConfig.Num < request.ConfigNum {
		reply.Err = OK
		kv.mu.RUnlock()
		return
	}
	kv.mu.RUnlock()
	
	var	commandReply CommandReply
	kv.Execute(NewDeleteShardsCommand(request), &commandReply)

	reply.Err = commandReply.Err
}

func (kv *ShardKV) gcAction() {
	kv.mu.RLock()
	gid2shardIds := kv.getShardIdsByStatus(GCing)
	var wg sync.WaitGroup
	for gid, shardIds := range gid2shardIds {
		wg.Add(1)
		go func(servers []string, configNum int, shardIds []int) {
			defer wg.Done()
			gcTaskRequest := ShardOperationRequest{configNum, shardIds}
			for _, server := range servers {
				var gcTaskReply ShardOperationReply
				serverEnd := kv.makeEnd(server)
				if serverEnd.Call("ShardKV.DeleteShardsData", &gcTaskRequest, &gcTaskReply) && gcTaskReply.Err == OK {
					// if leader succeed to gc, current server update config
					kv.Execute(NewDeleteShardsCommand(&gcTaskRequest), &CommandReply{})
				}
			}
		}(kv.lastConfig.Groups[gid], kv.currentConfig.Num, shardIds)
	}
	kv.mu.RUnlock()
	wg.Wait()
}
```

### 定时进行no-op日志发送

这个在本文开头中详细介绍了这个问题，不进行no-op的判断可能会产生活锁，但是不是每次都会出现这种bug，但仍然需要解决。比较理想的做法是在raft层添加no-op日志，但是在6.824中添加后lab2的测试则无法通过，因此在上层添加no-op日志。

解决方式也很简单，在raft层判断leader的最后一个日志是否是当前term，如果不是，则添加一个no-op日志，同步日志信息。

```
func (kv *ShardKV) checkEntryInCurrentTermAction() {
	if !kv.rf.HasLogInCurrentTerm() {
		kv.Execute(NewEmptyEntryCommand(), &CommandReply{})
	}
}
```

### 处理所有请求的协程

额外开启一个applier()协程，统一处理以上的所有请求，这在前面的实现中多次使用这种方式，只是这次请求的种类更多。整体分为command和快照两部分，暂时只讨论command部分。

```
// a dedicated applier goroutine to apply committed entries to stateMachine, take snapshot and apply snapshot from raft
func (kv *ShardKV) applier() {
	for kv.killed() == false {
		select {
		case msg := <-kv.applyCh:
			if msg.CommandValid {
				kv.mu.Lock()
				if msg.CommandIndex <= kv.lastApplied {
					kv.mu.Unlock()
					continue
				}
				kv.lastApplied = msg.CommandIndex

				var reply *CommandReply
				cmd := msg.Command.(Command)
				switch cmd.Op {
				case Operation:
					operation := cmd.Data.(CommandRequest)
					reply = kv.applyOperation(&msg, &operation)
				case Configuration:
					nextConfig := cmd.Data.(shardmaster.Config)
					reply = kv.applyConfiguration(&nextConfig)
				case InsertShards:
					shardsInfo := cmd.Data.(ShardOperationReply)
					reply = kv.applyInsertShards(&shardsInfo)
				case DeleteShards:
					shardsInfo := cmd.Data.(ShardOperationRequest)
					reply = kv.applyDeleteShards(&shardsInfo)
				case EmptyEntry:
					reply = kv.applyEmptyEntry()
				}
				
				// only notify related channel for currentTerm's log when the node is leader
				if currentTerm, isLeader := kv.rf.GetState(); isLeader && msg.CommandTerm == currentTerm {
					ch := kv.getNotifyChan(msg.CommandIndex)
					ch <- reply
				}

				needSnapshot := kv.needSnapshot()
				if needSnapshot {
					kv.takeSnapshot(msg.CommandIndex)
				}
				kv.mu.Unlock()
			} else if msg.SnapshotValid {
				kv.mu.Lock()
				if kv.rf.CondInstallSnapshot(msg.SnapshotTerm, msg.SnapshotIndex, msg.Snapshot) {
					kv.restoreSnapshot(msg.Snapshot)
					kv.lastApplied = msg.SnapshotIndex
				}
				kv.mu.Unlock()
			} else {
				fmt.Println("run into wrong state in applier")
			}
		}
	}
}
```

command中的applyOperation与lab3的基本一致：

```
func (kv *ShardKV) applyOperation(msg *raft.ApplyMsg, operation *CommandRequest) *CommandReply {
	var reply *CommandReply
	shardId := key2shard(operation.Key)
	if kv.canServe(shardId) {
		if operation.Op != OpGet && kv.isDuplicateRequest(operation.ClientId, operation.CommandId) {
			return kv.lastOperations[operation.ClientId].LastReply
		} else {
			reply = kv.applyLogToStateMachines(operation, shardId)
			if operation.Op != OpGet {
				kv.lastOperations[operation.ClientId] = OperationContext{operation.CommandId,reply}
			}
			return reply
		}
	}
	return &CommandReply{ErrWrongGroup, ""}
}
```

applyConfiguration首先要确保config的Num匹配，然后再分情况讨论：

- 如果shard在当前配置下不属于该raft集群，在新配置中属于该集群，则将shard的状态更新为pulling。
- 如果shard在当前配置属于该raft集群，新配置中不属于，则shard的状态更新为BePulling。

```
func (kv *ShardKV) applyConfiguration(nextConfig *shardmaster.Config) *CommandReply {
	if nextConfig.Num == kv.currentConfig.Num + 1 {
		kv.updateShardStatus(nextConfig)
		kv.lastConfig = kv.currentConfig
		kv.currentConfig = *nextConfig
		return &CommandReply{OK, ""}
	}
	return &CommandReply{ErrOutDated, ""}
}

func (kv *ShardKV) updateShardStatus(nextConfig *shardmaster.Config) {
	for i := 0; i < shardmaster.NShards; i ++ {
		// pull data from other shard
		if kv.currentConfig.Shards[i] != kv.gid && nextConfig.Shards[i] == kv.gid {
			gid := kv.currentConfig.Shards[i]
			if gid != 0 {
				kv.stateMachines[i].Status = Pulling
			}
		}
		// be pulled data to other shard
		if kv.currentConfig.Shards[i] == kv.gid && nextConfig.Shards[i] != kv.gid {
			gid := nextConfig.Shards[i]
			if gid != 0 {
				kv.stateMachines[i].Status = BePulling
			}
		}
	}
}
```

applyInsertShards需要确保对应的config信息一致，否则直接返回过期error。

对于pulling完成的shard，将状态置为GCing，在GC协程中将其他raft组下数据进行垃圾回收。

```
func (kv *ShardKV) applyInsertShards(shardsInfo *ShardOperationReply) *CommandReply {
	if shardsInfo.ConfigNum == kv.currentConfig.Num {
		for shardId, shardData := range shardsInfo.Shards {
			shard := kv.stateMachines[shardId]
			if shard.Status == Pulling {
				for key, value := range shardData {
					shard.KV[key] = value
				}
				shard.Status = GCing
			} else {
				break
			}
		}
		for clientId, operationContext := range shardsInfo.LastOperations {
			if lastOperation, ok := kv.lastOperations[clientId]; !ok || lastOperation.MaxAppliedCommandId < operationContext.MaxAppliedCommandId {
				kv.lastOperations[clientId] = operationContext
			}
		}
		return &CommandReply{OK, ""}
	}
	return &CommandReply{ErrOutDated, ""}
```

applyDeleteShards类似，需要判断config，更新状态即可。

```
func (kv *ShardKV) applyDeleteShards(shardsInfo *ShardOperationRequest) *CommandReply {
	if shardsInfo.ConfigNum == kv.currentConfig.Num {
		for _, shardId := range shardsInfo.ShardIds {
			shard := kv.stateMachines[shardId]
			if shard.Status == GCing {
				shard.Status = Serving
			} else if shard.Status == BePulling {
				kv.stateMachines[shardId] = NewShard()
			} else {
				// fmt.Println("duplicated shards deletion")
				break
			}
		}
		return &CommandReply{OK, ""}
	}
	return &CommandReply{OK, ""}
}

```

### 分片KV的快照需要添加什么？

除了正常需要保存的stateMachine和去重表（lastOperations）外，还需要记录当前配置和先前的配置，在前面的迁移、垃圾回收中也可以看出，主要判断就是依赖config来执行。

## 参考文章

1、[Raft 的 Figure 8 讲了什么问题？为什么需要 no-op 日志？](https://mp.weixin.qq.com/s/jzx05Q781ytMXrZ2wrm2Vg)

2、[MIT 6.824 LAB 4B(分布式shard database)](https://www.jianshu.com/p/f5c8ab9cd577)

3、https://github.com/OneSizeFitsQuorum/MIT6.824-2021/blob/master/docs/lab4.md