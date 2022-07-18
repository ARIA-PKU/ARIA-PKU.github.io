---
title: Raft分布式搭建（一）
date: 2022-07-13T14:21:25+08:00
lastmod: 2022-07-13T14:21:25+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/7_1/%E5%BF%83%E7%A9%BA.jpg

categories:
  - 分布式
tags:
  - raft
# nolastmod: true
draft: false
---

基于MIT6.824搭建的分布式系统，记录一下lab2实现过程中遇到的细节，以及一些对遇到的问题的思考。

<!--more-->

## lab介绍

整个项目是MIT6.824中[lab2](http://nil.csail.mit.edu/6.824/2020/labs/lab-raft.html)，[lab3](http://nil.csail.mit.edu/6.824/2020/labs/lab-kvraft.html)与[lab4](http://nil.csail.mit.edu/6.824/2020/labs/lab-shard.html)包括challenge的通过go语言实现的完整版本。实现了raft选举，在raft之上的KV服务以及shard服务等，以及实现了快照，崩溃恢复等功能以达到高可用的目的。

课程中关于代码的编写也给了一些建议，关于整体结构设计的[structure](http://nil.csail.mit.edu/6.824/2020/labs/raft-structure.txt)，关于go中锁使用的[lock](http://nil.csail.mit.edu/6.824/2020/labs/raft-locking.txt)，以及其他建议的[guide](https://thesquareplanet.com/blog/students-guide-to-raft/)。以上都可以帮助在代码实现中少走弯路。但是整体系统在各个地方的细节都很多，不能忽略论文中的每一处细节。

## 项目地址

lab的代码地址：https://github.com/ARIA-PKU/MIT6.824

## 如何搭建一个raft系统

在lab2中，项目要求要实现raft的选举，leader与followers的信息传递与同步以及raft服务器崩溃重启后的服务恢复三大功能。在对raft论文通读之后，需要对教授的课程做好笔记，教授在课堂中讨论的问题基本都是设计系统时的关键要点。在实现过程中还需要结合论文，考虑各个实现细节。

### Leader Election是怎么实现的？



在讨论这个问题之前，需要先了解论文中对各个节点的分类。

![](http://oss.surfaroundtheworld.top/blog-pictures/raft/figure_4.jpeg)

从上图可以看出，raft将所有节点分类为了三种身份：

- Leader：在一个任期内，集群中至多一个Leader，负责发送心跳，响应客户端，创建日志与同步日志。
- Candidate：是leader选举过程中的过渡状态，由follower转化而来，发起投票竞选leader。
- Follower：接收leader的同步消息，同时为选举投票。



接下来，还了解Term（任期）的概念：

![](https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_5.jpeg)

每一个term之前都是一次选举，一个或多个candidate会竞选leader，如果竞选失败会重新竞选。

每个节点会存储currentTerm，这是一个单调递增的编号。如果一个服务器在通信时候，发现自己的编号小于其他人的的term，则会更新currentTerm到较大值。如果candidate或leader发现自己的term较小，则会退回follwer状态。如果接收到更小的过期term请求，则会拒绝或忽略该请求。

在figure 4中提到，follwer如果没有收到心跳一段时间后，则会发起选举。此时，该follwer会增加自己的term号，向所有节点发起RequestVoteRPC请求，可能有多个candidate都在发起选举，因此也会收到其他节点声明自己是leader的心跳，此时有两种反馈方式：

- 该请求的term大于等于自己的term，则说明对方已经成为leader，自己回退为follower
- 该请求的term小于自己的term，则返回拒绝请求，并返回currentTerm使得对方更新term并回退为follower

如果多个candidate同时选举，则会导致一直无法选举出leader的结果，因此，raft采用了随机选举超时的机制，每个candidate选举后会随机一个选举超时时间，使得能在较短时间内选举出leader。

再通过voteFor来确保每个节点在一个term内只给一个节点投票，避免产生split brain的情况。



在以上理论基础上，就可以实现raft的选举机制。为了达到lab2对时间的要求，需要充分利用go的协程优势，尽可能实现高并发。

raft的结构体如下（每个raft的节点记录自己的信息，对于选举而言下面的很多内容还用不到，在下面会继续整理）：

```
type Raft struct {
	mu        sync.RWMutex        // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	applyCh			chan ApplyMsg
	applyCond   	*sync.Cond    // used to wakeup applier goroutine after committing new entries
	state			NodeState
	replicatorCond	[]*sync.Cond  // used to signal replicator goroutine to batch replicating entries

	// definition in Figure 2 
	currentTerm int
	votedFor    int
	logs        []Entry

	commitIndex int
	lastApplied int
	nextIndex   []int
	matchIndex  []int

	electionTimer	*time.Timer
	heartbeatTimer 	*time.Timer
}
```

对每个节点，设置两个timer（timer是go中的计时用的数据结构，内部通过channel的方式实现）对heartbeat和election计时。对于leader节点，定时发送heartbeat向follower发送同步日志并维护leader地位；对于follower节点，electiontimer时间内没有收到leader的心跳则发起选举，收到则更新计时器时间。开一个watchTimer的协程，监控计时器信息。

```
// watch all timer, choose to do heartbeat or start a new election
func (rf *Raft) watchTimer() {
	for rf.killed() == false {
		select {
		case <-rf.heartbeatTimer.C:
			rf.mu.Lock()
			if rf.state == Leader {
				rf.BroadcastHeartbeat(true)
				rf.heartbeatTimer.Reset(HeartbeatTimeout())
			}
			rf.mu.Unlock()
		case <-rf.electionTimer.C:
			rf.mu.Lock()
			rf.ChangeState(Candidate)
			rf.currentTerm += 1
			rf.startElection()
			rf.electionTimer.Reset(ElectionTimeout())
			rf.mu.Unlock()
		}
	
	}
}
```

下面就是关键的如何发起一次选举，将term + 1后首先将票投给自己，然后向除了自己以外的所有节点发送requestVoteRPC，在接收返回信息后，需要先进行判断当前状态是否仍是candidate，当前的term是否没有改变，因为在发送请求的同时也在接收其他节点的请求，因此可能节点状态已经改变。（这里通过的go封装的rpc进行调用）

在guide中也提到，对于过期的reply直接丢弃，无需处理。

```
func (rf *Raft) startElection() {
	lastLog := rf.getLastLog()
	request := &RequestVoteArgs {
		Term:			rf.currentTerm,
		CandidateId:	rf.me,
		LastLogIndex:	lastLog.Index,
		LastLogTerm:	lastLog.Term,
	}
	DPrintf("{Node: %v} starts election with request: %v", rf.me, request)

	grantedCount := 1
	rf.votedFor = rf.me
	rf.persist()
	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
		go func(peer int) {
			reply := &RequestVoteReply{VoteGranted: false}
			if rf.sendRequestVote(peer, request, reply) {
				rf.mu.Lock()
				defer rf.mu.Unlock()
				DPrintf("{Node: %v} receive reply: %v from {Node: %v} in term %v", rf.me, reply, peer, request.Term)
				// avoid current node receive heartbeat or already become follower before send request
				if rf.currentTerm == request.Term && rf.state == Candidate {
					if reply.VoteGranted {
						grantedCount += 1
						if grantedCount > len(rf.peers) / 2 {
							DPrintf("{Node: %v} receive majority vote in term: %v", rf.me, request.Term)
							rf.ChangeState(Leader)
							rf.BroadcastHeartbeat(true)
						}
					} else if reply.Term > rf.currentTerm {
						DPrintf("{Node: %v} find larger term as leader {Node: %v} with term: %v in term: %v", rf.me, peer, reply.Term, request.Term)
						rf.ChangeState(Follower)
						rf.currentTerm, rf.votedFor = reply.Term, -1
						rf.persist()
					}
				}
			}
		}(peer)
	}
}
```

同时，需要处理来自其他节点的requestVote的请求。这里数据结构的设计严格遵循了论文中figure 2给出的设计方案。term小于当前term或者已经投票的返回currentTerm与false即可；如果大于当前term，则直接修改currentTerm并将自己更新为follower。论文中还提到了up-to-date的log问题，这在之前整理的课程笔记的谁是合适的leader中详细介绍了，这里不再赘述。最后记录voteFor，重置timer，并返回term与voteGranted。

在课程的guide中的livelocks（活锁）一节中强调，需要在投票之后才能重置timer，否则在网络不可靠的情况下容易频繁选举，造成活锁。

raft的选举流程就成功实现了。

```
func (rf *Raft) RequestVote(request *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	defer rf.persist()
	// if currentTerm is larger or has voted for another node, return false
	if request.Term < rf.currentTerm || (request.Term == rf.currentTerm &&  rf.votedFor != -1 && rf.votedFor != request.CandidateId) {
		reply.Term, reply.VoteGranted = rf.currentTerm, false
		return
	}

	if request.Term > rf.currentTerm {
		rf.ChangeState(Follower)
		rf.currentTerm, rf.votedFor = request.Term, -1
	}

	// candidate’s log is at least as up-to-date as receiver’s log, grant vote
	if !rf.isUpToDate(request.LastLogTerm, request.LastLogIndex) {
		reply.Term, reply.VoteGranted = rf.currentTerm, false
		return
	}

	rf.votedFor = request.CandidateId
	rf.electionTimer.Reset(ElectionTimeout())
	reply.Term, reply.VoteGranted = rf.currentTerm, true
}
```

### Leader与Follower如何实现日志同步？

#### 论文中讨论的一些概念需要深入理解

在raft中，每一个事件都被称为一个entry，每个entry包含<term, index, cmd>。其中，index记录其在log中的位置，cmd是可以应用到状态机中的操作。

log（日志）就是由entry构成的数组，只有leader才能改变其他节点的log。entry先由leader添加进本地log中，然后发起共识请求，在获得大多数同意后提交状态机执行。follower则根据leader获取新日志和commitIndex，根据commitIndex确定是否应用到自己的状态机。

在下图中，每个方格中的数字对应的是term number，下面对应其cmd。

![](https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_6.png)

日志的实现过程中需要弄懂论文的Figure 2中给出的总结。其给出了各个数据结构的设计以及设计实现的各种细节。

对于raft中给出的结构如下（关于持久化的问题，会在后面整理），其中

- currentTerm，记录最大term编号
- voteFor，记录投票给了谁，voteFor在一个term里只会投票给一个人，否则会产生split brain
- log[]，存放entry的日志
- commitIndex，记录所有日志中被committed的最大index号
- lastApplied，与commitIndex不同，其记录的是被应用到状态机上的最大编号
- nextIndex[]和matchIndex[]，这两个是leader节点需要记录的数组。nextIndex记录的是对于每一个server下一个需要发送给该server的日志index；matchIndex记录的是已经复制给该server的最大日志index。

![](http://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_2_state.png)

然后就是在日志实现中的AppendEntries RPC的设计，分别包括了请求rpc与回复rpc，还包含了一些implement将在具体实现中应用到。

对于请求rpc：

- term和leaderId，leader的term和id信息
- prevLogTerm和prevLogTerm，发送日志的前一个日志的index和term，用于实现日志的对齐
- entries[]，发送的日志信息，心跳则不包含日志，也可能包含多个日志
- leaderCommit，leader的commitIndex，用于更新接收方的commitIndex

对于接收rpc：

- term，返回currentTerm用于leader更新
- success，如果follower的日志与prevLogIndex和prevLogTerm对齐，则返回true

![](http://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_2_appendentries.png)

#### 日志同步的实践

在实现中，遵循guide中的提示，将心跳与AppendEntries统一处理。为了确保复制操作的安全与高效，采用了sync.cond的数据结构，下面的的else对应的是leader收到client请求，增加新的日志后，唤醒复制操作（这里是一个额外的协程对每个peer执行的replicator操作），这样可以减少不必要的循环，释放CPU给其他协程。同时，还需要心跳过程中同步日志数据。

```
func (rf *Raft) BroadcastHeartbeat(isHeartbeat bool) {
	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
		if isHeartbeat {
			// sync the log and maintain leadership
			go rf.replicateOneRound(peer)
		} else {
			// signal replicator goroutine to send entries
			rf.replicatorCond[peer].Signal()
		}
	}
}

func (rf *Raft) replicator(peer int) {
	rf.replicatorCond[peer].L.Lock();
	defer rf.replicatorCond[peer].L.Unlock()
	for rf.killed() == false {
		// if not need to replicate to this peer, 
		// release CPU and wait for other goroutine's signal
		// else call replicateOneRound to make the 
		// peer to catch up the leader
		for !rf.needReplicate(peer) {
			rf.replicatorCond[peer].Wait()
		}
		rf.replicateOneRound(peer);
	}
}
```

在日志复制的过程中，可能遇到日志变得非常庞大，远远大于其状态机存储的大小，此时服务器可以采用快照的方式，存储所有状态，删除先前的日志（当然，删除操作也因实现各异，本项目中采用直接删除快照前的日志的方式）。快照是日志压缩的非常常用的一种方式。

这种情况下，在日志复制过程中可能会遇到一个问题：需要发送日志的server的前一个index小于当前日志的第一个index（因为先前的日志被快照后删除），这样就需要对快照进行复制同步。否则正常进行日志同步即可。

```
func (rf *Raft) replicateOneRound(peer int) {
	rf.mu.RLock()
	if rf.state != Leader {
		rf.mu.RUnlock()
		return
	}
	peerCurrentLogIndex := rf.nextIndex[peer] - 1
	
	if peerCurrentLogIndex < rf.logs[0].Index {
		// need InstallSnapshot RPC to catch up leader
		request := rf.genInstallSnapshotRequest()
		rf.mu.RUnlock()

		reply := &InstallSnapshotReply{}
		if rf.sendInstallSnapshot(peer, request, reply) {
			rf.mu.Lock()
			rf.handleInstallSnapshotReply(peer, request, reply)
			rf.mu.Unlock()
		}
	} else {
		request := rf.genAppendEntriesRequest(peerCurrentLogIndex)
		rf.mu.RUnlock()
		reply := &AppendEntriesReply{}
		if rf.sendAppendEntries(peer, request, reply) {
			rf.mu.Lock()
			rf.handleAppendEntriesReply(peer, request, reply)
			rf.mu.Unlock()
		}
	}
}
```

先从没有快照的情况谈起，leader发送的appendEntries请求与figure 2中的设计一致（注意这里firstIndex的使用其实就是为了应对快照的情况，定位到其真正的位置）：

```
func (rf *Raft) genAppendEntriesRequest(prevLogIndex int) *AppendEntriesRequest {
	firstIndex := rf.logs[0].Index
	entries := make([]Entry, len(rf.logs[prevLogIndex - firstIndex + 1:]))
	copy(entries, rf.logs[prevLogIndex - firstIndex + 1:])
	return &AppendEntriesRequest {
		Term:			rf.currentTerm,
		LeaderId:		rf.me,
		PrevLogIndex:	prevLogIndex,
		PrevLogTerm:	rf.logs[prevLogIndex - firstIndex].Term,
		Entries:		entries,
		LeaderCommit: 	rf.commitIndex,
	}
}
```

follower在接收到leader以上的请求后，会按照以下逻辑执行：

1. leader发送的term小于currentTerm，则直接返回currentTerm和false，leader在收到后会更改自己的状态为follower。
2. leader发送的term大于当前term，则更新currentTerm和voteFor。
3. 收到大于等于当前term的leader的信息，则将自己更新为follower，同时重置timer
4. 如果leader的日志小于当前最早的日志，说明已经更新过，直接丢弃，返回的消息leader也无需处理。
5. 在实际生产中，可能会产生远远落后于leader的节点，则日志term相差很大，如果一个个往前查找会产生大量的rpc，因此在日志没有对齐的时候，可以采用快速回滚的方式，可以采用二分法等方式。这里采用的是教授课上介绍的方式，具体实现在先前的博客的如何进行日志的快速回滚中介绍过，然后返回即可。
6. 从前往后遍历leader发送过来的日志，找到第一个不匹配的点，然后拼接日志。这里注意一定是找到第一个不匹配的位置，可能会覆盖follwer原有的日志，这里为什么能够覆盖也在先前的raft博客中整理过。
7. 根据论文中的描述，更新本地的commitIndex，并唤醒coordinator协程对其日志应用到状态机上
8. 最后更新返回的term，并返回true

```
func (rf *Raft) AppendEntries(request *AppendEntriesRequest, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	defer rf.persist()
	if request.Term < rf.currentTerm {
		reply.Term, reply.Success = rf.currentTerm, false
		return
	}
	if request.Term > rf.currentTerm {
		rf.currentTerm, rf.votedFor = request.Term, -1
	}

	rf.ChangeState(Follower)

	// drop unexpected logs
	if request.PrevLogIndex < rf.logs[0].Index {
		reply.Term, reply.Success = 0, false
		return
	}

	// fast back-up
	if !rf.logMatch(request.PrevLogTerm, request.PrevLogIndex) {
		reply.Term, reply.Success = rf.currentTerm, false
		lastIndex := rf.getLastLog().Index
		if lastIndex < request.PrevLogIndex {
			reply.ConflictTerm, reply.ConflictIndex = -1, lastIndex + 1
		} else {
			firstIndex := rf.logs[0].Index
			reply.ConflictTerm = rf.logs[request.PrevLogIndex - firstIndex].Term
			index := request.PrevLogIndex - 1
			for index >= firstIndex && rf.logs[index - firstIndex].Term == reply.ConflictTerm {
				index --
			}
			reply.ConflictIndex = index
		}
		return
	}

	firstIndex := rf.logs[0].Index
	for index, entry := range request.Entries {
		if entry.Index - firstIndex >= len(rf.logs) || rf.logs[entry.Index - firstIndex].Term != entry.Term {
			rf.logs = cutEntriesSpace(append(rf.logs[:entry.Index - firstIndex], request.Entries[index:]...))
			break
		}
	}

	// change commitIndex and signal gorotinue to execute
	// commited logs
	newCommitIndex := Min(request.LeaderCommit, rf.getLastLog().Index)
	if newCommitIndex > rf.commitIndex {
		rf.commitIndex = newCommitIndex
		rf.applyCond.Signal()
	}

	reply.Term, reply.Success = rf.currentTerm, true
}
```

leader接收到follwer的日志同步返回后，会按照以下逻辑进行处理：

- 细节很重要，执行前必须对leader当前状态进行判断，如果不再是leader则无须执行。
- follower返回日志同步成功，更新matchIndex与nextIndex。同时，需要根据matchIndex的情况，更新leader的commitIndex，直接sort，找到半数以上已经收到日志的index。更新前再确认日志状态，这里是**确保只能提交当前term的log**，这一点在figure 8中说明，是为了防止已经提交的日志被覆盖。更新后signal协程将已经commit的状态应用到状态机中。
- follower返回日志同步失败，则会分为两种情况：
  - follower的term大于leader，则直接将leader转换为follower，更新状态
  - 否则则是日志不匹配，根据先前的快速回退策略，快速返回日志

```
func (rf *Raft) handleAppendEntriesReply(peer int, request *AppendEntriesRequest, reply *AppendEntriesReply) {
	if rf.state == Leader && rf.currentTerm == request.Term {
		if reply.Success {
			rf.matchIndex[peer] = request.PrevLogIndex + len(request.Entries)
			rf.nextIndex[peer] = rf.matchIndex[peer] + 1

			// update leader's commitIndex
			n := len(rf.matchIndex)
			confirmIndex := make([]int, n)
			copy(confirmIndex, rf.matchIndex)
			sort.Ints(confirmIndex)
			newCommitIndex := confirmIndex[(n - 1) / 2]
			if newCommitIndex > rf.commitIndex {
				if rf.logMatch(rf.currentTerm, newCommitIndex) {
					DPrintf("{Node %d} advance commitIndex from %d to %d with matchIndex %v in term %d", rf.me, rf.commitIndex, newCommitIndex, rf.matchIndex, rf.currentTerm)
					rf.commitIndex = newCommitIndex
					rf.applyCond.Signal()
				} else {
					DPrintf("{Node: %d cannot update commitIndex}", rf.me)
				}
			}
		} else {
			if reply.Term > rf.currentTerm {
				rf.ChangeState(Follower)
				rf.currentTerm, rf.votedFor = reply.Term, -1
				rf.persist()
			} else if reply.Term == rf.currentTerm {
				rf.nextIndex[peer] = reply.ConflictIndex
				if reply.ConflictTerm != -1 {
					firstIndex := rf.logs[0].Index
					for i := request.PrevLogIndex; i >= firstIndex; i -- {
						if rf.logs[i - firstIndex].Term == reply.ConflictTerm {
							rf.nextIndex[peer] = i + 1
							break
						} else if rf.logs[i - firstIndex].Term < reply.ConflictTerm {
							break
						}
					}
				}
			}
		}
	}
}
```

#### 异步协程放入ApplyMsg

对所有节点，开启一个额外的协程将ApplyMsg传入applyCh中，applyCh中的数据会在raft的上层进行调用，用于将日志的内容应用到状态机中。

采用这种异步协程的方式可以提高整体的并发与吞吐量，在PingCap的这篇[文章](https://pingcap.com/zh/blog/optimizing-raft-in-tikv)中介绍了。为了确保在发送到applyCh通道中的内容不会重复，因此单独开一个协程将已经commit但还没有apply的日志apply到状态机中，可以更好地实现并行。同时也利用了sync.Cond的数据结构，空闲时候放弃CPU，需要更新时再唤醒协程。

```
func (rf *Raft) coordinator() {
	for rf.killed() == false {
		rf.mu.Lock()
		for rf.lastApplied >= rf.commitIndex {
			rf.applyCond.Wait()
		}
		firstIdx, commitIdx, lastApplied := rf.logs[0].Index, rf.commitIndex, rf.lastApplied
		entries := make([]Entry, commitIdx - lastApplied)
		
		copy(entries, rf.logs[lastApplied - firstIdx + 1: commitIdx - firstIdx + 1])
		rf.mu.Unlock()

		for _, entry := range entries {
			rf.applyCh <- ApplyMsg {
				CommandValid: 	true,
				Command: 		entry.Command,
				CommandIndex:	entry.Index,
				CommandTerm:	entry.Term,
			}
		}

		rf.mu.Lock()
		rf.lastApplied = Max(rf.lastApplied, commitIdx)
		rf.mu.Unlock()
	}
}
```

#### raft层的快照系统需要些什么？

论文中对日志压缩也给了详尽说明，其中

- term和leaderId记录任期和leaderId
- lastIncludedIndex和lastIncludedTerm记录的是快照前最后一个日志的index
- offset记录snapshot文件所在chunk块的字节偏移
- data[]记录snapshot，开始的位置从offset开始
- done，如果是最后一个chunk返回true

![](http://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/raft/figure_13.png)

论文中针对的是快照过大需要分块发送的情况，但是在8.624中无须考虑这个，因此可以根据实际情况做适当的简化：

```
type InstallSnapshotRequest struct {
	Term				int
	LeaderId			int
	LastIncludedIndex	int
	LastIncludedTerm	int
	Data				[]byte
}
```

在raft层的快照主要是为了上层服务。leader的快照时候，要注意go中切片的引用问题，为了防止内存泄漏，需要将日志进行深拷贝。

```
// use snapshot to trim logs and restore its state
func (rf *Raft) Snapshot(index int, snapshot []byte) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	startIndex := rf.logs[0].Index
	if index <= startIndex {
		// has been snapshot before
		return
	}
	rf.logs = cutEntriesSpace(rf.logs[index - startIndex:])
	rf.logs[0].Command = nil
	rf.persister.SaveStateAndSnapshot(rf.encodeState(), snapshot)
}
```

follower在接收到安装快照之后，按照如下流程工作，这个与日志的类似，要更简单一些。注意对于过期的snapshot直接抛弃，不做其他处理。

```
func (rf *Raft) InstallSnapshot(request *InstallSnapshotRequest, reply *InstallSnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm

	if request.Term < rf.currentTerm {
		return
	}

	if request.Term > rf.currentTerm {
		rf.currentTerm, rf.votedFor = request.Term, -1
		rf.persist()
	}

	rf.ChangeState(Follower)
	rf.electionTimer.Reset(ElectionTimeout())

	// snapshot is outdated
	if request.LastIncludedIndex <= rf.commitIndex {
		return
	}

	go func() {
		rf.applyCh <- ApplyMsg{
			SnapshotValid: true,
			Snapshot:      request.Data,
			SnapshotTerm:  request.LastIncludedTerm,
			SnapshotIndex: request.LastIncludedIndex,
		}
	}()
}
```

### 为什么要持久化与如何持久化？

raft在崩溃之后，需要重新启动，对于一些重要的数据结构是需要持久化来保证raft正常运行。

在figure 2中给出了需要持久化的三个数据结构：log[]，currentTerm和votedFor。原因在前一篇博客的日志持久化部分也进行了详细整理。

在6.824中的持久化是在所有修改以上三个数据机构的时候进行持久化，在实际生产中应该会采用wal的方式。