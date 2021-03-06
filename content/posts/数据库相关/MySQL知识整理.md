---
title: MySQL知识整理
date: 2022-06-20T17:21:29+08:00
lastmod: 2022-06-20T17:21:29+08:00

cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/6_15/sand_clock.jpg
# images:
#   - /img/cover.jpg
categories:
  - 数据库
tags:
  - mysql
# nolastmod: true
draft: false
---

整理一些学习使用过程中遇到的细碎的知识点

<!--more-->

##  一、MySQL 的 delete、truncate、drop 有什么区别?

**从执行速度上来说**

drop > truncate >> DELETE

**从原理上来说**

### **1、DELETE**

```text
DELETE from TABLE_NAME where xxx
```

1、DELETE属于数据库DML操作语言，只删除数据不删除表的结构，会走事务，执行时会触发trigger；

2、在 InnoDB 中，DELETE其实并不会真的把数据删除，mysql 实际上只是给删除的数据打了个标记为已删除，因此 delete 删除表中的数据时，表文件在磁盘上所占空间不会变小，存储空间不会被释放，只是把删除的数据行设置为不可见。 虽然未释放磁盘空间，但是下次插入数据的时候，仍然可以重用这部分空间（重用 → 覆盖）。

3、 DELETE执行时，会先将所删除数据缓存到rollback segement中，事务commit之后生效;

4、 delete from table_name删除表的全部数据,对于MyISAM 会立刻释放磁盘空间，InnoDB 不会释放磁盘空间;

5、对于delete from table_name where xxx 带条件的删除, 不管是InnoDB还是MyISAM都不会释放磁盘空间;

6、 delete操作以后使用 optimize table table_name 会立刻释放磁盘空间。不管是InnoDB还是MyISAM 。所以要想达到释放磁盘空间的目的，delete以后执行optimize table 操作。

7、delete 操作是一行一行执行删除的，并且同时将该行的的删除操作日志记录在redo和undo表空间中以便进行回滚（rollback）和重做操作，生成的大量日志也会占用磁盘空间。

### **2、truncate**

```text
Truncate table TABLE_NAME
```

1、truncate：属于数据库DDL定义语言，不走事务，原数据不放到 rollback segment 中，操作不触发 trigger。

执行后立即生效，无法找回 执行后立即生效，无法找回 执行后立即生效，无法找回

2、 truncate table table_name 立刻释放磁盘空间 ，不管是 InnoDB和MyISAM 。truncate table其实有点类似于drop table 然后creat,只不过这个create table 的过程做了优化，比如表结构文件之前已经有了等等。所以速度上应该是接近drop table的速度;

3、truncate能够快速清空一个表。并且重置auto_increment的值。

但对于不同的类型存储引擎需要注意的地方是：

- 对于MyISAM，truncate会重置auto_increment（自增序列）的值为1。而delete后表仍然保持auto_increment。
- 对于InnoDB，truncate会重置auto_increment的值为1。delete后表仍然保持auto_increment。但是在做delete整个表之后重启MySQL的话，则重启后的auto_increment会被置为1。

也就是说，InnoDB的表本身是无法持久保存auto_increment。delete表之后auto_increment仍然保存在内存，但是重启后就丢失了，只能从1开始。实质上重启后的auto_increment会从 SELECT 1+MAX(ai_col) FROM t 开始。

4、 小心使用 truncate，尤其没有备份的时候

### **3、drop**

```text
Drop table Tablename
```

1、drop：属于数据库DDL定义语言，同Truncate；

执行后立即生效，无法找回 执行后立即生效，无法找回 执行后立即生效，无法找回

2、 drop table table_name 立刻释放磁盘空间 ，不管是 InnoDB 和 MyISAM; drop 语句将删除表的结构被依赖的约束(constrain)、触发器(trigger)、索引(index); 依赖于该表的存储过程/函数将保留,但是变为 invalid 状态。

3、 小心使用 drop ，要删表跑路的兄弟，请在订票成功后在执行操作！订票电话：400-806-9553

可以这么理解，一本书，delete是把目录撕了，truncate是把书的内容撕下来烧了，drop是把书烧了

## 二、sql的执行顺序

SELECT语句中子句的执行顺序与SELECT语句中子句的输入顺序是不一样的，所以并不是从SELECT子句开始执行的，而是按照下面的顺序执行： 

开始->FROM子句->WHERE子句->GROUP BY子句->HAVING子句->ORDER BY子句->SELECT子句->LIMIT子句->最终结果 

每个子句执行后都会产生一个中间结果，供接下来的子句使用，如果不存在某个子句，就跳过 

对比了一下，mysql和sql执行顺序基本是一样的, 标准顺序的 SQL 语句为: 

select 考生姓名, max(总成绩) as max总成绩 

from tb_Grade 

where 考生姓名 is not null 

group by 考生姓名 

having max(总成绩) > 600 

order by max总成绩 

 在上面的示例中 SQL 语句的执行顺序如下: 

　　 (1). 首先执行 FROM 子句, 从 tb_Grade 表组装数据源的数据 

　　 (2). 执行 WHERE 子句, 筛选 tb_Grade 表中所有数据不为 NULL 的数据 

　　 (3). 执行 GROUP BY 子句, 把 tb_Grade 表按 "学生姓名" 列进行分组(注：这一步开始才可以使用select中的别名，他返回的是一个游标，而不是一个表，所以在where中不可以使用select中的别名，而having却可以使用，感谢网友  zyt1369  提出这个问题)
　　 (4). 计算 max() 聚集函数, 按 "总成绩" 求出总成绩中最大的一些数值 

　　 (5). 执行 HAVING 子句, 筛选课程的总成绩大于 600 分的. 

　　 (7). 执行 ORDER BY 子句, 把最后的结果按 "Max 成绩" 进行排序. 

## 三、记录锁、间隙锁与next-key lock

在RR的隔离级别下，存在幻读的情况。其解决方式是MVCC + Next-Key Lock，对于MVCC的读分为两种分别是快照读和当前读。其中，当前读就是简单的select，查询的都是快照版本，因此不会存在幻读的问题；因此，讨论的幻读都是发生在当前读的场景下。

当前读指的是 lock in share mode、for update、insert以及delete这些需要加锁的操作，在当前读的场景下通过Next-Key Lock解决幻读问题。

对mysql来说，实现了两种行级锁，共享锁（读锁）与排他锁（写锁）。行锁的实现算法有三种，分别是Record Lock（记录锁）、Gap Lock（间隙锁）以及Next-Key Lock。

- Record Lock，实际上指的是对索引记录的锁定。

  比如执行语句`select * from user where age=10 for update`，将会锁住`user`表所有`age=10`的行记录，所有对`age=10`的记录的操作都会被阻塞。

- Gap Lock，也就是**间隙锁**，它用于锁定的索引之间的间隙，但是不会包含记录本身。

  比如语句`select * from user where age>1 and age<10 for update`，将会锁住`age`在(1,10)的范围区间，此时其他事务对该区间的操作都会被阻塞。

- **Next-Key Lock**，实际上就是相当于Record Lock+Gap Lock的组合。比如索引有10，20，30几个值，那么被锁住的区间可能会是(-∞,10]，(10,20]，(20,30]，(30,+∞)。

### 如何解决幻读问题？

1、没有索引

操作的行数据没有索引，则存储引擎会将所有记录加锁，并且所有间隙也会加锁，因此相当于把整个表锁住了。

2、普通索引

假设数据库中有10， 20， 30的数据，并且是普通索引，执行`select * from user where age=20 for update`，则会在20上加行锁，在（10， 20）和（20， 30）之间加间隙锁。

3、唯一或主键索引

如果是唯一索引或者主键索引的话，并且是等值查询，实际上会发生锁降级，降级为Record Lock，就不会有间隙锁了。

因为主键或者唯一索引能保证值是唯一的，所以也就不需要再增加间隙锁了。

# 参考文章

1、[面试官灵魂一问: MySQL 的 delete、truncate、drop 有什么区别?](https://zhuanlan.zhihu.com/p/270331768)

2、[拿捏！隔离级别、幻读、Gap Lock、Next-Key Lock](https://zhuanlan.zhihu.com/p/402591869)

