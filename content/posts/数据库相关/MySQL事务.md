---
title: MySQL事务与日志
date: 2022-06-20T17:18:48+08:00
lastmod: 2022-06-20T17:18:48+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/6_20/mysql_3.jpg

categories:
  - 数据库
tags:
  - mysql
  - 隔离级别
# nolastmod: true
draft: false
---

事务、隔离级别、日志和锁的相关的内容

<!--more-->

# 一、事务与隔离级别

## 什么是事务?

事务是逻辑上的一组操作，要么都执行，要么都不执行。

## 事务的特性(ACID)

1.  **原子性：** 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2.  **一致性：** 执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；
3.  **隔离性：** 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
4.  **持久性：** 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

## **数据事务的实现原理呢？**

我们这里以 MySQL 的 InnoDB 引擎为例来简单说一下。

MySQL InnoDB 引擎使用 **redo log(重做日志)** 保证事务的**持久性**，使用 **undo log(回滚日志)** 来保证事务的**原子性**。

MySQL InnoDB 引擎通过 **锁机制**、**MVCC** 等手段来保证事务的隔离性（ 默认支持的隔离级别是 **`REPEATABLE-READ`** ）。

保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。

## 并发事务带来哪些问题?

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务（多个用户对同一数据进行操作）。并发虽然是必须的，但可能会导致以下的问题。

- **脏读（Dirty read）:** 当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改（Lost to modify）:** 指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。 例如：事务 1 读取某表中的数据 A=20，事务 2 也读取 A=20，事务 1 修改 A=A-1，事务 2 也修改 A=A-1，最终结果 A=19，事务 1 的修改被丢失。
- **不可重复读（Unrepeatableread）:** 指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。
- **幻读（Phantom read）:** 幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

**不可重复读和幻读区别：**

不可重复读的重点是修改比如多次读取一条记录发现其中某些列的值被修改，幻读的重点在于新增或者删除比如多次读取一条记录发现记录增多或减少了

## 事务隔离级别有哪些?

SQL 标准定义了四个隔离级别：

- **READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。
- **READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。
- **REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**。
- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。

---

|     隔离级别     | 脏读 | 不可重复读 | 幻读 |
| :--------------: | :--: | :--------: | :--: |
| READ-UNCOMMITTED |  √   |     √      |  √   |
|  READ-COMMITTED  |  ×   |     √      |  √   |
| REPEATABLE-READ  |  ×   |     ×      |  √   |
|   SERIALIZABLE   |  ×   |     ×      |  ×   |

### MySQL 的默认隔离级别是什么?

MySQL InnoDB 存储引擎的默认支持的隔离级别是 **REPEATABLE-READ（可重读）**。我们可以通过`SELECT @@tx_isolation;`命令来查看，MySQL 8.0 该命令改为`SELECT @@transaction_isolation;`

```sql
mysql> SELECT @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

正：**MySQL InnoDB 的 REPEATABLE-READ（可重读）并不保证避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是 Next-Key Locks。**

因为隔离级别越低，事务请求的锁越少，所以大部分数据库系统的隔离级别都是 **READ-COMMITTED(读取提交内容)** ，但是你要知道的是 InnoDB 存储引擎默认使用 **REPEAaTABLE-READ（可重读）** 并不会有任何性能损失。

InnoDB 存储引擎在 **分布式事务** 的情况下一般会用到 **SERIALIZABLE(可串行化)** 隔离级别。

# 二、Mysql的三种日志与MVCC

##  undolog：如何让更新的数据可以回滚？

undolog会将修改的数据串成一条版本链，事务回滚就可以通过undolog来使得数据回滚回去。

此外undolog在MVCC中也扮演了重要角色。

##  redolog：系统宕机了，如何避免数据丢失？

如果对应的脏页还没有被刷到磁盘中，数据库就宕机了，数据如何持久化？

为了解决这个问题，我们需要把内存所做的修改写入到 redo log buffer中，这是内存里的一个缓冲区，用来存在redo日志。

rodo log记录了你对数据所做的修改，如“将id=1这条数据的name从a变为abc”，物理日志哈，后面会再提一下。**「redo log是顺序写所以比随机写效率高」**

**「InnoDB的redo log是固定大小的」**，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么总大小为4GB。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![](http://oss.surfaroundtheworld.top/blog-pictures/6_20/redolog.png)

write pos是当前要写的位置，checkpoint是要擦除的位置，擦除前要把对应的脏页刷回到磁盘中。write pos和checkpoint中间的位置是可以写的位置。

##  binlog：主从库之间如何同步数据？

当我们把mysql主库的数据同步到从库，或者其他数据源时，如es，bi库时，只需要订阅主库的binlog即可。

为什么要弄2种日志呢？其实这都是由历史原因决定的

MySQL刚开始用binlog实现归档的功能，但是binlog没有crash-safe的能力，所以后来InnoDB引擎加了redo log来实现crash-safe。假如MySQL中只有一个InnoDB引擎，说不定就能用redo log来实现归档了，此时就可以将redo log和 binlog合并到一块了

这两种日志的区别如下：

1. redo log是InnoDB存储引擎特有，binglog是MySQL的server层实现的，所有引擎都可以使用
2. redo log是物理日志，记录的是数据页上的修改。binlog是逻辑日志，记录的是语句的原始逻辑，如给id=2的这一行的c字段加1
3. redo log是固定空间，循环写。binlog是追加写，当binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志

## 两阶段提交

接着我们来看一下将id=2的行c字段加1的执行流程。

![](http://oss.surfaroundtheworld.top/blog-pictures/6_20/%E4%B8%A4%E9%98%B6%E6%AE%B5.png)

1. 引擎将新数据更新到内存中，将操作记录到redo log中，此时redo log处于prepare状态，然后告知执行器执行完成了，可以提交事务
2. 执行器生成操作的binlog，并把binlog写入磁盘
3. 引擎将写入的redo log改为提交状态，更新完成

**「为什么要把relog的写入拆成2个步骤？即prepare和commit，两阶段提交」**

因为不管你先写redolog还是binlog，奔溃发生后，最终其实都有可能会造成原库和用日志恢复出来的库不一致

**「而两阶段提交可以避免这个问题」**

> redolog和binlog具有关联行，在恢复数据时，redolog用于恢复主机故障时的未更新的物理数据，binlog用于备份操作。每个阶段的log操作都是记录在磁盘的，在恢复数据时，redolog 状态为commit则说明binlog也成功，直接恢复数据；如果redolog是prepare，则需要查询对应的binlog事务是否成功，binlog成功则执行，如果binlog不完整则回滚数据。

##  MVCC是如何实现的

**对于使用InnoDB存储引擎的表来说，聚集索引记录中都包含下面2个必要的隐藏列**

**trx_id**：一个事务每次对某条聚集索引记录进行改动时，都会把该事务的事务id赋值给trx_id隐藏列

**roll_pointer**：每次对某条聚集索引记录进行改动时，都会把旧的版本写入undo日志中。这个隐藏列就相当于一个指针，通过他找到该记录修改前的信息

如果一个记录的name从貂蝉被依次改为王昭君，西施，会有如下的记录，多个记录构成了一个版本链

![](http://oss.surfaroundtheworld.top/blog-pictures/6_20/%E7%89%88%E6%9C%AC%E9%93%BE.png)

#### 通过ReadView实现MVCC

**为了判断版本链中哪个版本对当前事务是可见的，MySQL设计出了ReadView的概念**。4个重要的内容如下

**m_ids**：在生成ReadView时，当前系统中活跃的事务id列表
**min_trx_id**：在生成ReadView时，当前系统中活跃的最小的事务id，也就是m_ids中的最小值
**max_trx_id**：在生成ReadView时，系统应该分配给下一个事务的事务id值
**creator_trx_id**：生成该ReadView的事务的事务id

当对表中的记录进行改动时，执行insert，delete，update这些语句时，才会为事务分配唯一的事务id，否则一个事务的事务id值默认为0。

max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比如现在有事务id为1，2，3这三个事务，之后事务id为3的事务提交了，当有一个新的事务生成ReadView时，m_ids的值就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4

**mvcc判断版本链中哪个版本对当前事务是可见的过程如下**

![](http://oss.surfaroundtheworld.top/blog-pictures/6_20/readview.png)

执行过程如下：

1. 如果被访问版本的trx_id=creator_id，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问
2. 如果被访问版本的trx_id<min_trx_id，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问
3. 被访问版本的trx_id>=max_trx_id，表明生成该版本的事务在当前事务生成ReadView后才开启，该版本不可以被当前事务访问
4. 被访问版本的trx_id是否在m_ids列表中
   4.1 是，创建ReadView时，该版本还是活跃的，该版本不可以被访问。顺着版本链找下一个版本的数据，继续执行上面的步骤判断可见性，如果最后一个版本还不可见，意味着记录对当前事务完全不可见
   4.2 否，创建ReadView时，生成该版本的事务已经被提交，该版本可以被访问

# 参考文章

1、[InnoDB并发如此高，原因竟然在这？](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651967047&idx=1&sn=b605fe50e6dd74ecad659c464a0e29ee&chksm=bd2d7b9b8a5af28d35c13e469e8e2c6a7082f00e608fd52d999fc0f7a6aec016faf2a8d748fa&mpshare=1&scene=24&srcid=0517nAsRI4WTkEdKChyhFUOM&sharer_sharetime=1621266991409&sharer_shareid=fbcd095bb8806eb23f12fdb6f5d1c5de#rd)

2、[MySQL三种日志有啥用？如何提高MySQL并发度？](https://mp.weixin.qq.com/s?__biz=MzU4MDM3MDgyMA==&mid=2247504191&idx=1&sn=ebf8dceb0118bf26b54c97fd52f90fdd&chksm=fd5579d4ca22f0c2c0e076b9907ad99fcba74aa47cf690d3551c6f643126b68593cf9fedc34e&mpshare=1&scene=24&srcid=0723hReOGTQyQN4APB4UeoUj&sharer_sharetime=1627050069380&sharer_shareid=fbcd095bb8806eb23f12fdb6f5d1c5de#rd)

3、[MVCC是如何实现的？](https://mp.weixin.qq.com/s?__biz=MzIxMzk3Mjg5MQ==&mid=2247487338&idx=2&sn=495be2257ea355c52cb0771368159b75&chksm=97afed9ea0d8648867b0080d821de6bf0530669718046f57d838836eb0a3614e6d6674c5615a&mpshare=1&scene=24&srcid=07233okwPFDk86gVHBzNsuSj&sharer_sharetime=1627050567801&sharer_shareid=fbcd095bb8806eb23f12fdb6f5d1c5de#rd)