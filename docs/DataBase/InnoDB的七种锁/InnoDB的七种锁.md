

> 原创： 58沈剑 架构师之路

## 前言

前面谈到了[**InnoDB为何并发如此高**](http://blog.ztgreat.cn/article/74)，今天我们要来 了解一下 InnoDB中的锁

这部分可以参考官方的文档，官方的文档写得很详细的 [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

## InnoDB 中的锁

官方文档上面列举了8中锁

- AUTO-INC Locks （自增锁）
- Shared and Exclusive Locks（共享/排他锁）
- Intention Locks（意向锁）
- Insert Intention Locks（插入意向锁）
- Record Locks（记录锁）
- Gap Locks（间隙锁）
- Next-Key Locks（临键锁）
- Predicate Locks for Spatial Indexes （空间索引谓词锁）

接下来 我们结合官方的文档和一些实践来理解这些锁(前七种)。

### 自增锁

自增锁是一种特殊的**表级别锁**（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。`innodb_autoinc_lock_mode` 这个参数可以 控制 用于 `auto-increment` 锁定的算法.

官网是这么说的：

> An `AUTO-INC` lock is a special table-level lock taken by transactions inserting into tables with `AUTO_INCREMENT` columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.
>
> The [`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) configuration option controls the algorithm used for auto-increment locking. It allows you to choose how to trade off between predictable sequences of auto-increment values and maximum concurrency for insert operations.

#### insert 语句分类

insert 语句分三种类型：simple insert, bulk insert, mixed insert

- simple insert
   insert 时可以预先知道插入的行记录数量。例如insert into t values(a), (b), 这个语句插入时，mysql就可以预先知道插入的行记录数量为2.
   insert ... on duplicate...语句不属于simple insert

- bulk insert
   insert 时可以预先知道插入的行记录数量。
   例如load data语句、INSERT ... SELECT语句, REPLACE ... SELECT语句都属于bulk insert。

- mixed insert
   simple insert中有一部分指定了AUTO_INCREMENT的值，类型下来的例子:

  > INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');

#### innodb_autoinc_lock_mode

innodb_autoinc_lock_mode有三个值：

- 0, traditional
- 1, consecutive
- 2, interleaved

##### traditional

这种模式下，所有的insert语句在开始时都会获得一个表锁autoinc_lock.该锁会一直持有到insert语句执行结束才会被释放。对于一条insert插入多个行记录的语句，他保证了同一条语句插入的行记录的自增ID是连续的。
 这个锁并不是事务级别的锁。
 在这种模式下，主从复制时，基于语句复制模式下，主和从的同一张表的同一个行记录的自增ID是一样的。但是同样基于语句复制模式下，interleaved模式，也就是innodb_autoinc_lock_mode=2时则不能保证主从同一张表的同一个行记录的自增ID一样。
 这种模式下，表的并发性最低。

##### consecutive

这种模式下，insert语句在开始时会获得一个表锁autoinc_lock, simple insert在获取到需要增加的ID的量后，autoinc_lock就会被释放,不必等到语句执行结束。但对于bulk insert，自增锁会被一直持有直到语句执行结束才会被释放。
 这种模式仍然保证了同一条语句插入的行记录的自增ID是连续的。
 这种模式下的主从复制表现跟traditional模式下一样，但是性能会有所提高。

##### interleaved

这种模式下，simple insert语句能保证ID是连续的，但是bulk insert的ID则可能有空洞。
 主从复制的同一张表下的同一行id有可能不一样。

### 共享/排它锁

共享/排它锁（Shared and Exclusive Locks）：

(1)事务拿到某一行记录的共享S锁，才可以读取这一行；

(2)事务拿到某一行记录的排它X锁，才可以修改或者删除这一行；

其**兼容互斥表**如下：

​          S          X

S      兼容      互斥

X      互斥      互斥

即：

(1)多个事务可以拿到一把S锁，读读可以并行；

(2)而只有一个事务可以拿到X锁，写写/读写必须互斥；

共享/排它锁的潜在问题是，不能充分的并行，解决思路是**数据多版本**。

### 意向锁

InnoDB支持多粒度锁(multiple granularity locking)，它允许行级锁与表级锁共存，实际应用中，InnoDB使用的是意向锁。

**意向锁（Intention Locks）**是指，未来的某个时刻，事务可能要加共享/排它锁了，先提前声明一个意向。

意向锁有这样一些特点：

(1)首先，意向锁，是一个表级别的锁(table-level locking)；

(2)意向锁分为：

- **意向共享锁**(intention shared lock, IS)，它预示着，事务有意向对表中的某些行加共享S锁
- **意向排它锁**(intention exclusive lock, IX)，它预示着，事务有意向对表中的某些行加排它X锁

 

#### 举个例子：

select ... lock in share mode，要设置**IS锁**；

select ... for update，要设置**IX锁**；

(3)意向锁协议(intention locking protocol)并不复杂：

- 事务要获得某些行的S锁，必须先获得表的IS锁
- 事务要获得某些行的X锁，必须先获得表的IX锁

(4)由于意向锁仅仅表明意向，它其实是比较弱的锁，意向锁之间并不相互互斥，而是可以并行，其**兼容互斥表**如下：

​          IS          IX

IS      兼容      兼容

IX      兼容      兼容

(5)额，既然意向锁之间都相互兼容，那其意义在哪里呢？它会与共享锁/排它锁互斥，其**兼容互斥表**如下：

​          S          X

IS      兼容      互斥

IX      互斥      互斥

> 画外音：排它锁是很强的锁，不与其他类型的锁兼容。这也很好理解，修改和删除某一行的时候，必须获得强锁，禁止这一行上的其他并发，以保障数据的一致性。

 

### 插入意向锁

对已有数据行的**修改与删除**，必须加强互斥锁X锁，那对于**数据的插入**，是否还需要加这么强的锁，来实施互斥呢？插入意向锁，孕育而生。

 

**插入意向锁（Insert Intention Locks）**，是间隙锁(Gap Locks)的一种（所以，也是实施在索引上的），它是专门针对insert操作的。

> 画外音：有点尴尬，间隙锁下一篇文章才会介绍，暂且理解为，它是一种实施在索引上，锁定索引某个区间范围的锁。

它的玩法是：

多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。

> 画外音：官网的说法是
>
> Insert Intention Lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.

#### 举例

在MySQL，InnoDB，RR下：

```
t(id unique PK, name);
```

数据表中有数据：

```
10, shenjian

20, zhangsan

30, lisi
```

事务A先执行，在10与20两条记录中插入了一行，还未提交：

```
insert into t values(11, xxx); 
```

事务B后执行，也在10与20两条记录中插入了一行：

```
insert into t values(12, ooo); 
```

(1)会使用什么锁？

(2)事务B会不会被阻塞呢？

#### 回答

虽然事务隔离级别是RR，虽然是同一个索引，虽然是同一个区间，但插入的记录并不冲突，故这里：

- 使用的是插入意向锁
- 并不会阻塞事务B

#### 思路总结

(1)InnoDB使用**共享锁**，可以提高读读并发；

(2)为了保证数据强一致，InnoDB使用强**互斥锁**，保证同一行记录修改与删除的串行性；

(3)InnoDB使用**插入意向锁**，可以提高插入并发；

MySQL的InnoDB的细粒度行锁，是它最吸引人的特性之一,但是，如果查询没有命中索引，也将退化为表锁。

InnoDB的细粒度锁，是实现在索引记录上的。 

### InnoDB的索引

InnoDB的索引有两类索引，**聚集索引**(Clustered Index)与**普通索引**(Secondary Index)。

InnoDB的每一个表都会有聚集索引：

(1)如果表定义了PK，则**PK**就是聚集索引；

(2)如果表没有定义PK，则**第一个非空unique列**是聚集索引；

(3)否则，InnoDB会**创建一个隐藏的row-id**作为聚集索引；

为了方便说明，后文都将以PK说明。

索引的结构**是B+树**，这里不展开B+树的细节，说几个结论：

(1)在索引结构中，非叶子节点存储key，叶子节点存储value；

(2)**聚集索引**，叶子节点存储行记录(row)；

> 画外音：所以，InnoDB索引和记录是存储在一起的，而MyISAM的索引和记录是分开存储的。

(3)**普通索引**，叶子节点存储了PK的值；

> 画外音：
>
> 所以，InnoDB的普通索引，实际上会扫描两遍：
>
> 第一遍，由普通索引找到PK；
>
> 第二遍，由PK找到行记录；
>
> 索引结构，InnoDB/MyISAM的索引结构，如果大家感兴趣，未来撰文详述。

 

举个例子，假设有InnoDB表：

```
t(id PK, name KEY, sex, flag);
```

表中有四条记录：

```
1, shenjian, m, A

3, zhangsan, m, A

5, lisi, m, A

9, wangwu, f, B
```



![20190224093919](http://img.blog.ztgreat.cn/document/database/20190427185955.jpg)



以看到：

(1)第一幅图，id PK的聚集索引，叶子存储了所有的行记录；

(2)第二幅图，name上的普通索引，叶子存储了PK的值；

 

对于：

```
select * from t where name=’shenjian’;
```



(1)会先在name普通索引上查询到PK=1；

(2)再在聚集索引衫查询到(1,shenjian, m, A)的行记录；

 

下文简单介绍InnoDB七种锁中的剩下三种：

- 记录锁(Record Locks)
- 间隙锁(Gap Locks)
- 临键锁(Next-Key Locks)

为了方便讲述，如无特殊说明，后文中，默认的事务隔离级别为**可重复读**(Repeated Read, RR)。

 

### 记录锁

**记录锁(Record Locks)**，它封锁索引记录，例如：

```
select * from t where id=1 for update; 
```

它会在id=1的索引记录上加锁，以阻止其他事务入，更新，删除id=1的这一行。

需要说明的是：

```
select * from t where id=1;
```

则是**快照读**(SnapShot Read)，它并不加锁，具体在[**InnoDB为何并发如此高**](http://blog.ztgreat.cn/article/74)中做了详细阐述。

### 间隙锁

**间隙锁(Gap Locks)**，它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

#### 举例

依然是上面的例子，InnoDB，RR：

```
t(id PK, name KEY, sex, flag); 
```

表中有四条记录：

```
1, shenjian, m, A

3, zhangsan, m, A

5, lisi, m, A

9, wangwu, f, B
```

这个SQL语句

```
select * from t 

where id between 8 and 15 

for update;
```

会封锁区间，以阻止其他事务id=10的记录插入。

> 画外音：
>
> 为什么要阻止id=10的记录插入？
>
> 如果能够插入成功，头一个事务执行相同的SQL语句，会发现结果集多出了一条记录，即幻影数据。

 

间隙锁的**主要目的**，就是为了防止其他事务在间隔中插入数据，以导致“不可重复读”。

如果把事务的隔离级别降级为**读提交**(Read Committed, RC)，间隙锁则会自动失效。

### 临键锁

**临键锁(Next-Key Locks)**，是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。

更具体的，临键锁会封锁索引记录本身，以及索引记录之前的区间。

如果一个会话占有了索引记录R的共享/排他锁，其他会话不能立刻在R之前的区间插入新的索引记录。

> 画外音：原文是说
>
> If one session has a shared or exclusive lock on record R in an index, another session cannot insert a new index record in the gap immediately before R in the index order.

#### 举例

依然是上面的例子，InnoDB，RR：

t(id PK, name KEY, sex, flag);

表中有四条记录：

1, shenjian, m, A

3, zhangsan, m, A

5, lisi, m, A

9, wangwu, f, B

 

PK上潜在的临键锁为：

(-infinity, 1]

(1, 3]

(3, 5]

(5, 9]

(9, +infinity]

临键锁的主要目的，也是为了避免**幻读**(Phantom Read)。如果把事务的隔离级别降级为RC，临键锁则也会失效。

今天的内容，主要对InnoDB的索引，以及三种锁的概念做了介绍。场景与例子，也都是最简单的场景与最简单的例子。

InnoDB的锁，与索引类型，事务的隔离级别相关，更多更复杂更有趣的案例，后续和大家介绍。

### 总结

(1)InnoDB使用**共享锁**，可以提高读读并发；

(2)为了保证数据强一致，InnoDB使用强**互斥锁**，保证同一行记录修改与删除的串行性；

(3)InnoDB使用**插入意向锁**，可以提高插入并发；

(4)InnoDB的索引与行记录存储在一起，这一点和MyISAM不一样；

(5)InnoDB的聚集索引存储行记录，普通索引存储PK，所以普通索引要查询两次；

(6)**记录锁**锁定索引记录；

(7)**间隙锁**锁定间隔，防止间隔中被其他事务插入；

(8)**临键锁**锁定索引记录+间隔，防止幻读；



> 架构师之路-分享可落地的架构文章
>
> 保留原创作者二维码

![20190224093919](http://img.blog.ztgreat.cn/document/database/20190224093919.png)

