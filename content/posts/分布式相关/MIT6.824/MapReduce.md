---
title: MapReduce
date: 2022-06-27T23:28:52+08:00
lastmod: 2022-06-27T23:28:52+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/6_15/maple.jpg
# images:
#   - /img/cover.jpg
categories:
  - 分布式
tags:
  - MIT6.824
  - MapReduce
draft: false
---

实现了一下MIT6.824课程中的lab1，对着MapReduce的论文又重新理解了一遍。好久没写go了，channel和协程用起来确实比其他语言更优雅一些。

<!--more-->

# 一、论文精读

## 1、论文背景

Google 的很多程序员在处理大量的原始数据，比如，文档抓取（类似网络爬虫的程 序）、Web 请求日志等等；也为了计算处理各种类型的衍生数据，比如倒排索引、Web 文档的图结构的各种表示形式、每台主机上网络爬虫抓取的页面数量的汇总、每天被请求的最多的查询的集合等等。

大多数这样的 数据处理运算在概念上很容易理解。然而由于输入的数据量巨大，因此要想在可接受的时间内完成运算，只 有将这些计算分布在成百上千的主机上。如何处理并行计算、如何分发数据、如何处理错误？所有这些问题 综合在一起，需要大量的代码处理，因此也使得原本简单的运算变得难以处理。

为了解决上述问题，而**不必关心并行计算、容错、数据分布、负载均衡等复杂的细节**，提出了MapReduce模型。

大多数的运算都包含这样的操作：在输入数据的“逻辑”记录上应用 Map 操作得出一个中间 key/value  pair 集合，然后在所有具有相同 key 值的 value 值上应用 Reduce 操作，从而达到合并中间的数据，得到一个想要的结果的目的。使用 MapReduce 模型，**再结合用户实现的 Map 和 Reduce 函数，我们就可以非常容易的 实现大规模并行化计算**。

## 2、实际应用示例

论文中还给出了很多实际的例子：

**分布式的 Grep**：Map 函数输出匹配某个模式的一行，Reduce 函数是一个恒等函数，即把中间数据复制到 输出。

**计算 URL 访问频率**：Map 函数处理日志中 web 页面请求的记录，然后输出(URL,1)。Reduce 函数把相同URL 的 value 值都累加起来，产生(URL,记录总数)结果。

**倒转网络链接图**：Map 函数在源页面（source）中搜索所有的链接目标（target）并输出为(target,source)。 Reduce 函数把给定链接目标（target）的链接组合成一个列表，输出(target,list(source))。

**倒排索引**：Map 函数分析每个文档输出一个(词,文档号)的列表，Reduce 函数的输入是一个给定词的所有 （词，文档号），排序所有的文档号，输出(词,list（文档号）)。所有的输出集合形成一个简单的倒排索引，它 以一种简单的算法跟踪词在文档中的位置。

## 3、具体实现

Google 内部广泛使用的运算环境的实现：用以太网交换机连接、由普通 PC 机组成的大型集群。

![](http://oss.surfaroundtheworld.top/blog-pictures/6_28/MapReduce.png)

执行流程：

1、用户程序首先调用的 MapReduce 库将输入文件分成 M 个数据片度，每个数据片段的大小一般从 16MB 到 64MB(可以通过可选的参数来控制每个数据片段的大小)。然后用户程序在机群中创建大量 的程序副本。

2、这些程序副本中的有一个特殊的程序–master。副本中其它的程序都是 worker 程序，由 master 分配 任务。有 M 个 Map 任务和 R 个 Reduce 任务将被分配，master 将一个 Map 任务或 Reduce 任务分配 给一个空闲的 worker。

3、被分配了 map 任务的 worker 程序读取相关的输入数据片段，从输入的数据片段中解析出 key/value  pair，然后把 key/value pair 传递给用户自定义的 Map 函数，由 Map 函数生成并输出的中间 key/value  pair，并缓存在内存中。

4、缓存中的 key/value pair 通过分区函数分成 R 个区域，之后**周期性的写入到本地磁盘**上。缓存的 key/value pair 在本地磁盘上的存储位置将被回传给 master，由 master 负责把这些**存储位置**再传送给 Reduce worker

5、当 Reduce worker 程序接收到 master 程序发来的数据存储位置信息后，使用 RPC 从 Map worker 所在 主机的磁盘上读取这些缓存数据。**当 Reduce worker 读取了所有的中间数据后，通过对 key 进行排序 后使得具有相同 key 值的数据聚合在一起。由于许多不同的 key 值会映射到相同的 Reduce 任务上， 因此必须进行排序。如果中间数据太大无法在内存中完成排序，那么就要在外部进行排序。**

6、 Reduce worker 程序遍历排序后的中间数据，对于每一个唯一的中间 key 值，Reduce worker 程序将这 个 key 值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追 加到所属分区的输出文件。

7、当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对 MapReduce 调用才返回。

在成功完成任务之后，MapReduce 的输出存放在 R 个输出文件中（对应每个 Reduce 任务产生一个输出 文件，文件名由用户指定）。一般情况下，用户不需要将这 R 个输出文件合并成一个文件–他们经常**把这些文件作为另外一个 MapReduce 的输入，或者在另外一个可以处理多个分割文件的分布式应用中使用。**

## 4、容错

MR的设计是在由成百上千的机器组成的集群来处理超大规模的数据，因此出现故障是常态，硬件故障或者是网络故障都会发生。因此，要有相应的机制处理错误。

### 4.1、worker故障

master 周期性的 ping 每个 worker。如果在一个约定的时间范围内没有收到 worker 返回的信息，master 将 把这个 worker 标记为失效。所有由这个失效的 worker 完成的 Map 任务被重设为初始的空闲状态，之后这些 任务就可以被安排给其他的 worker。同样的，worker 失效时正在运行的 Map 或 Reduce 任务也将被重新置为 空闲状态，等待重新调度。

**当 worker 故障时，由于已经完成的 Map 任务的输出存储在内存上，Map 任务的输出已不可访问了因此必须重新执行。而已经完成的 Reduce 任务的输出存储在全局文件系统上，因此不需要再次执行。**

### 4.2、master故障

一个简单的解决办法是让 master 周期性的将存储的内容的写入磁盘，即 检查点（checkpoint）。如果这个 master 任务失效了，可以从最后一个检查点（checkpoint）开始启动另一个 master 进程。然而，由于只有一个 master 进程，master 失效后再恢复是比较麻烦的，因此我们现在的实现是 如果 master 失效，就中止 MapReduce 运算。客户可以检查到这个状态，并且可以根据需要重新执行 MapReduce 操作。

## 5、存储位置

在我们的计算运行环境中，**网络带宽是一个相当匮乏的资源**。我们通过尽量把输入数据(由 GFS 管理)存 储在集群中机器的本地磁盘上来节省网络带宽。GFS 把每个文件按 64MB 一个 Block 分隔，每个 Block 保存 在多台机器上，环境中就存放了多份拷贝(一般是 3 个拷贝)。MapReduce 的 master 在调度 Map 任务时会考虑 输入文件的位置信息，**尽量将一个 Map 任务调度在包含相关输入数据拷贝的机器上执行；**如果上述努力失败 了，master 将尝试在保存有输入数据拷贝的机器附近的机器上执行 Map 任务(例如，分配到一个和包含输入数 据的机器在一个 switch 里的 worker 机器上执行)**。当在一个足够大的 cluster 集群上运行大型 MapReduce 操作的时候，大部分的输入数据都能从本地机器读取，因此消耗非常少的网络带宽。**

## 6、任务粒度

如前所述，我们把 Map 拆分成了 M 个片段、把 Reduce 拆分成 R 个片段执行。理想情况下，M 和 R 应当 比集群中 worker 的机器数量要多得多。在每台 worker 机器都执行大量的不同任务能够提高集群的动态的负载 均衡能力，并且能够加快故障恢复的速度：失效机器上执行的大量 Map 任务都可以分布到所有其他的 worker 机器上去执行。

但是实际上，在我们的具体实现中对 M 和 R 的取值都有一定的客观限制，因为 master 必须执行 O(M+R) 次调度，并且在内存中保存 O(M*R)个状态（对影响内存使用的因素还是比较小的：O(M*R)块状态，大概每 对 Map 任务/Reduce 任务 1 个字节就可以了）。

更进一步，R 值通常是由用户指定的，因为每个 Reduce 任务最终都会生成一个独立的输出文件。实际使 用时我们也倾向于选择合适的 M 值，以使得每一个独立任务都是处理大约 16M 到 64M 的输入数据（这样， 上面描写的输入数据本地存储优化策略才最有效），另外，我们把 R 值设置为我们想使用的 worker 机器数量 的小的倍数。我们通常会用这样的比例来执行 MapReduce：M=200000，R=5000，使用 2000 台 worker 机器。

## 7、备份任务

影响一个 MapReduce 的总执行时间最通常的因素是“落伍者”：在运算过程中，如果有一台机器花了很 长的时间才完成最后几个 Map 或 Reduce 任务，导致 MapReduce 操作总的执行时间超过预期。出现“落伍者” 的原因非常多。

我们有一个通用的机制来减少“落伍者”出现的情况。当一个 MapReduce 操作接近完成的时候，master 调度备用（backup）任务进程来执行剩下的、处于处理中状态（in-progress）的任务。无论是最初的执行进程、 还是备用（backup）任务进程完成了任务，我们都把这个任务标记成为已经完成。我们调优了这个机制，通 常只会占用比正常操作多几个百分点的计算资源。我们发现采用这样的机制对于减少超大 MapReduce 操作的 总处理时间效果显著。

## 8、技巧

1、通过hash方法按照key对数据进行分区

2、保证key有序

3、combine函数：Map 函数产生的中间 key 值的重复数据会占很大的比重，combine就是将重复数据合并，生成中间文件再发送给reduce执行。

4、多样的输入输出类型

5、在产生中间的辅助文件的时候，首先把输出结果写到 一个临时文件中，在输出全部数据之后，在使用系统级的原子操作 rename 重新命名这个临时文件。

6、记录并跳过损坏的记录

7、支持本地执行测试

8、支持状态信息的监控和展示

9、增加计数器

## 9、学习总结

首先，约束编程模式使得并行和分布式计算非常容易， 也易于构造容错的计算环境；其次，网络带宽是稀有资源。大量的系统优化是针对减少网络传输量为目的的： 本地优化策略使大量的数据从本地磁盘读取，中间文件写入本地磁盘、并且只写一份中间文件也节约了网络带宽；第三，多次执行相同的任务可以减少性能缓慢的机器带来的负面影响（注：即硬件配置的不平衡）， 同时解决了由于机器失效导致的数据丢失问题。

## 10、论文地址

https://github.com/ARIA-PKU/papers/tree/master/6.824/MapReduce

# 二、lab1的设计与实现

这个lab的实现过程让我收获颇多，很多论文中的细节都在实现过程中才理解，而且也久违的体验了一下go中channel和协程实现的优雅。下面尽可能详尽地记录一下这个lab的各个细节。

## 1、lab介绍

**lab要求：**http://nil.csail.mit.edu/6.824/2020/labs/lab-mr.html

单机实现一个分布式的MapReduce。由master和worker两种程序构成，worker通过rpc与master通信，向master请求任务，返回结果，并将输出写入文件；如果worker没有在合理时间内完成任务（lab规定是10s)，master将剥夺任务，分配给其他worker。

给定了八个txt文件作为已经split好的输入，统计其中所有单词的出现次数。其他的一些规则和提示在后面的实现中都会用到，这里暂时略过。

通过mrsequential.go引入，lab中的Map和Reduce都是给定的，就是简单的单词统计和整理，在sequential中的实现是序列化实现，可以借鉴其中的输入输出格式，文件读取方法。

## 2、设计

最初认为map和reduce两个阶段是可以并行实现的，但在思考后发现，map的内容会不断追加写入，同时进行reduce不仅会有冲突问题，更重要的是无法确定当前应该读取的位置。因此，实现中会先执行map任务，再执行reduce任务。

根据lab要求，需要实现两种任务程序，master和worker。通过论文和lab提示，设计整个流程：

1、master首先将当前阶段初始化为map，根据Map数量确定tasks数量，初始化每个task的初始状态，并监听worker的心跳请求；

2、worker向master周期性发送心跳，请求任务；

3、master接受到worker的心跳之后，会遍历所有tasks，将其中空闲的task或者是超时的task，根据当前阶段，分配任务给worker，并记录任务信息；（这里为了防止超时之后的task的任务执行两遍，可以在写入之前判断一下当前任务是否已经超时，如果超时则停止写入，防止重复）

4、worker根据接受到response确定是执行map还是reduce任务，并开始执行相应任务；

5、worker在执行完成后，会返回执行成功的任务记录，master记录并标记task为finished；

6、master当前阶段任务完成不能直接给worker返回退出，需要返回wait，因为可能只是map阶段完成，当reduce的任务完成后才能够给worker返回completed。

7、所有任务完成后，调用done方法，这里可以用通道控制，master与worker均会退出，获得若干最终结果文件。

其他注意点：

1、在map阶段，会将单词通过给定的hash函数，分配到对应的文件中，文件编号为mr-X-Y，X为map编号，Y为reduce编号，在reduce阶段worker会根据分配给自己的reduce编号，处理对应的文件。

2、数据写入中间文件，在论文中给出了写入临时文件并通过rename替代的方法，go中可以使用 `ioutil.TempFile`创建一个临时文件并使用`os.Rename` 自动重命名它。

3、worker中的reduce是直接通过读入内存，整理排序，如果内存不够用，需要考虑外部排序。

4、在可能并发的地方尽可能开协程并发，比如worker在map阶段，将map结果并发写入对应的各个文件等。

5、worker的两阶段工作：

map阶段，每次读取一个文件，并将内容追加写到对应的文件中；

reduce阶段，每次读取所有属于当前reduce任务的文件，并将文件内容计数写入文件。

## 3、实现

lab1：https://github.com/ARIA-PKU/MIT6.824

master的数据结构：

```
type Master struct {
	// Your definitions here.
	files []string  		// 记录输入文件
	nReduce int  			// reduce数量
	nMap int				// map数量
	phase OperationPhase	// master阶段，分为map、reduce和completed
	tasks []Task			// 记录当前所有任务

	heartbeatCh chan heartbeatMsg   // 接收心跳文件
	responseCh chan responseMsg		// 接收执行成功返回文件
	doneCh chan struct{}			// 最终执行完成文件
}
```

里面的task：

```
type Task struct {
	fileName string		 // map中需要确定读取文件
	id int			 	 // map或reduce对应的编号
	startTime time.Time  // 记录开始时间
	status TaskStatus	 // 任务状态，分为空闲，工作中和结束
}
```

worker中数据结构：

worker的结构依赖于master的返回内容

```
type HeartBeatResponse struct {
	FilePath string  	// 文件地址，map中需要
	WorkType WorkType	// 当前阶段，分为map、reduce、wait和completed
	NReduce int			// reduce总数
	Nmap int			// map总数
	Id int				// 当前的编号，结合以上阶段可以确定读取或写入文件的位置
}
```

最关键的就是数据结构的定义，具体实现都是围绕着这些数据进行操作。