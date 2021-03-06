## 常见问题

### Mysql索引主要使用的两种数据结构

- 哈希索引，对于哈希索引来说，底层的数据结构肯定是哈希表，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；其余大部分场景，建议选择BTree索引
- B+Tree索引，Mysql的BTree索引使用的是B树中的B+Tree但对于主要的两种存储引擎（MyISAM和InnoDB）的实现方式是不同的。

### 聚集索引和非聚集索引

**聚集索引**：索引中数据的物理存放地址和索引的顺序是一致的

**非聚集索引**：索引中数据的物理存放地址和索引的顺序是不一致的

### MyISAM和InnoDB实现BTree索引方式的区别

- InnoDB，其数据文件本身就是索引文件，树的节点data域保存了完整的数据记录。因为InnoDB的数据文件本身要按主键聚集，所以InnoDB要求表必须有主键（MyISAM可以没有）
- MyISAM，B+Tree叶节点的data域存放的是数据记录的地址，在索引检索的时候，首先按照B+Tree搜索算法搜索索引，如果指定的key存在，则取出其data域的值，然后以data域的值为地址读区相应的数据记录，因此myisam有两个文件（一个索引文件一个数据文件）。

### 为什么不用B树？

- B+**树的层级更少**：相较于B树B+每个**非叶子**节点存储的关键字数更多，树的层级更少所以查询数据更快；

- B+**树查询速度更稳定**：B+所有关键字数据地址都存在**叶子**节点上，所以每次查找的次数都相同所以查询速度要比B树更稳定;

- B+**树天然具备排序功能：**B+树所有的**叶子**节点数据构成了一个有序链表，在查询大小区间的数据时候更方便，数据紧密性很高，缓存的命中率也会比B树高。

### 为什么不用红黑树？

- 红黑树是一个二叉树，当数据量很大的时候树的深度会非常大，查找次数显著升高

- 数据库中使用了磁盘预读原理，将一个节点的大小设为等于一个页，这样每个节点只需要一次I/O就可以完全载入。而红黑树的每个节点都很小，造成浪费。

### 数据库引擎innodb与myisam的区别？

- 事务: InnoDB 是事务型的，可以使用 Commit 和 Rollback 语句。
- 并发: MyISAM 只支持表级锁，而 InnoDB 还支持行级锁。
- 外键: InnoDB 支持外键。
- 备份: InnoDB 支持在线热备份。
- 崩溃恢复: MyISAM 崩溃后发生损坏的概率比 InnoDB 高很多，而且恢复的速度也更慢。
- 其它特性: MyISAM 支持压缩表和空间数据索引。



## 数据库事务？

事务是一个不可分割的数据库操作序列，也是数据库并发控制的基本单位，其执行的结果必须使数据库从一种一致性状态变到另一种一致性状态。事务是逻辑上的一组操作，要么都执行，要么都不执行。

## 事务ACID4大特性

### 举例

我们以从A账户转账50元到B账户为例进行说明一下ACID，四大特性。

### 原子性

事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚。

**如果无法保证原子性会怎么样？**

OK，就会出现**数据不一致**的情形，A账户减去50元，而B账户增加50元操作失败。系统将无故丢失50元~

#### Mysql怎么保证原子性的？

OK，是利用Innodb的`undo log`。 `undo log`名为回滚日志，是实现原子性的关键，当事务回滚时能够撤销所有已经成功执行的sql语句，他需要记录你要回滚的相应日志信息。 例如

- (1)当你delete一条数据的时候，就需要记录这条数据的信息，回滚的时候，insert这条旧数据
- (2)当你update一条数据的时候，就需要记录之前的旧值，回滚的时候，根据旧值执行update操作
- (3)当年insert一条数据的时候，就需要这条记录的主键，回滚的时候，根据主键执行delete操作

`undo log`记录了这些回滚需要的信息，当事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。

### 一致性

数据库在事务执行前后都保持一致性状态。在一致性状态下，所有事务对同一个数据的读取结果都是相同的。

#### Mysql怎么保证一致性的？

OK，这个问题分为两个层面来说。 从数据库层面，数据库通过原子性、隔离性、持久性来保证一致性。也就是说ACID四大特性之中，C(一致性)是目的，A(原子性)、I(隔离性)、D(持久性)是手段，是为了保证一致性，数据库提供的手段。数据库必须要实现AID三大特性，才有可能实现一致性。

但是，如果你在事务里故意写出违反约束的代码，一致性还是无法保证的。例如，你在转账的例子中，你的代码里故意不给B账户加钱，那一致性还是无法保证。因此，还必须从应用层角度考虑。

### 隔离性

一个事务所做的修改在最终提交以前，对其它事务是不可见的。

#### Mysql怎么保证隔离性的？

OK,利用的是锁和MVCC机制。还是拿转账例子来说明，有一个账户表如下 表名`t_balance`

![](https://pic4.zhimg.com/80/v2-86bedce66efcc4a3b90d09740163e3a7_hd.jpg)

其中id是主键，user_id为账户名，balance为余额。还是以转账两次为例，如下图所示

![](https://pic2.zhimg.com/80/v2-9e136615fa9a741763dfc7ba2f976a99_hd.jpg)

至于MVCC,即多版本并发控制(Multi Version Concurrency Control),一个行记录数据有多个版本对快照数据，这些快照数据在`undo log`中。 如果一个事务读取的行正在做DELELE或者UPDATE操作，读取操作不会等行上的锁释放，而是读取该行的快照版本。 由于MVCC机制在可重复读(Repeateable Read)和读已提交(Read Commited)的MVCC表现形式不同，就不赘述了。

但是有一点说明一下，在事务隔离级别为读已提交(Read Commited)时，一个事务能够读到另一个事务已经提交的数据，是不满足隔离性的。但是当事务隔离级别为可重复读(Repeateable Read)中，是满足隔离性的。

### 持久性

持久性是指事务一旦提交，它对数据库的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。

####  Mysql怎么保证持久性的？

是利用Innodb的`redo log`。 正如之前说的，Mysql是先把磁盘上的数据加载到内存中，在内存中对数据进行修改，再刷回磁盘上。如果此时突然宕机，内存中的数据就会丢失。 *怎么解决这个问题？* 简单啊，事务提交前直接把数据写入磁盘就行啊。 *这么做有什么问题？*

- 只修改一个页面里的一个字节，就要将整个页面刷入磁盘，太浪费资源了。毕竟一个页面16kb大小，你只改其中一点点东西，就要将16kb的内容刷入磁盘，听着也不合理。
- 毕竟一个事务里的SQL可能牵涉到多个数据页的修改，而这些数据页可能不是相邻的，也就是属于随机IO。显然操作随机IO，速度会比较慢。

于是，决定采用`redo log`解决上面的问题。当做数据修改的时候，不仅在内存中操作，还会在`redo log`中记录这次操作。当事务提交的时候，会将`redo log`日志进行刷盘(`redo log`一部分在内存中，一部分在磁盘上)。当数据库宕机重启的时候，会将`redo log`中的内容恢复到数据库中，再根据`undo log`和`binlog`内容决定回滚数据还是提交数据。



### 并发事务带来的问题

#### 脏读

![](https://www.pdai.tech/_images/pics/dd782132-d830-4c55-9884-cfac0a541b8e.png)

#### 丢弃修改

T1 和 T2 两个事务都对一个数据进行修改，T1 先修改，T2 随后修改，T2 的修改覆盖了 T1 的修改。例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最 终结果A=19，事务1的修改被丢失。

![](https://www.pdai.tech/_images/pics/88ff46b3-028a-4dbb-a572-1f062b8b96d3.png)

### 不可重复读

T2 读取一个数据，T1 对该数据做了修改。如果 T2 再次读取这个数据，此时读取的结果和第一次读取的结果不同。

![](https://www.pdai.tech/_images/pics/c8d18ca9-0b09-441a-9a0c-fb063630d708.png)

#### 幻读

T1 读取某个范围的数据，T2 在这个范围内插入新的数据，T1 再次读取这个范围的数据，此时读取的结果和和第一次读取的结果不同。

![](https://www.pdai.tech/_images/pics/72fe492e-f1cb-4cfc-92f8-412fb3ae6fec.png)

#### 不可重复度和幻读区别：

**不可重复读的重点是修改，幻读的重点在于新增或者删除。**

### 数据库的隔离级别

1. 读未提交，事务之间可以读取彼此未提交的数据 .**可能会导致脏读、幻读或不可重复读**.

    **读不会加任何锁。而写会加排他锁，并到事务结束之后释放。**（ 排他锁会阻止其它事务再对其**锁定的数据**加读或写的锁）

2. 读已经提交，事务之间可以读取彼此已提交的数据；**可以阻止脏读，但是幻读或不可重复读仍有可能发生**

3. 可重复读，就是对一个记录读取多次的记录是相同的，比如对于一个数A读取的话一直是A，前后两次读取的A是一致的；**可以阻止脏读和不可重复读，但幻读仍有可能发生。**

4. 可串行化读，在并发情况下，和串行化的读取的结果是一致的，没有什么不同，比如不会发生脏读和幻读；**该级别可以防止脏读、不可重复读以及幻读。** 全部的读写操作都会加锁，阻塞。

| 隔离级别         | 脏读 | 不可重复读 | 幻影读 |
| ---------------- | ---- | ---------- | ------ |
| READ-UNCOMMITTED | √    | √          | √      |
| READ-COMMITTED   | ×    | √          | √      |
| REPEATABLE-READ  | ×    | ×          | √      |
| SERIALIZABLE     | ×    | ×          | ×      |

**MySQL InnoDB 存储引擎的默认支持的隔离级别是 REPEATABLE-READ（可重读）。**

- 使用MVCC解决不可重复读
- 这里需要注意的是：与 SQL 标准不同的地方在于InnoDB 存储引擎在 REPEATABLE-READ（可重读）事务隔离级别 下使用的是**间隙锁**，因此可以避免幻读的产生。
    - Next-Key Lock=间隙锁+行锁。
    - 间隙锁锁住的是一个范围，但不包括记录本身，间隙锁只工作在REPEATABLE-READ隔离级别之下。
    - Next-Key Lock锁定一个范围，并且锁定记录本身。**当查询的索引含有唯一属性（唯一索引，主键索引）时，Innodb存储引擎会对next-key lock进行优化，将其降为record lock,即仅锁住索引本身，而不是范围！若是普通辅助索引，则会使用传统的next-key lock进行范围锁定！**

InnoDB 存储引擎在 分布式事务 的情况下一般会用到SERIALIZABLE(可串行化)隔离级别。

### MVCC（多版本并发控制）机制

**基本原理：**MVCC是通过保存数据在某个时间点的快照来实现该机制，这意味着一个事务无论运行多长时间，在同一个事务里能够看到数据都是一致的。具体就是在其在每行记录后面保存两个隐藏的列，分别保存这个行的**创建版本号和删除版本号**，可以理解为事务id.

增：更新创建版本号为当前事务id

删：更新删除版本号为事务id

改：将原数据删除版本号更新为事务id，并且设置新的创建版本号

查：查找需要满足两个条件。当前事务版本号大于等于创建版本号（确保数据已存在），删除版本号为空或者大于当前事务版本号（确保数据未删除）。

## 内连接和外连接

内连接只显示符合连接条件的记录，外连接除了显示符合条件的记录外还会显示表中的记录。

### 内连接

- 内连接语句：

    ```select filedlist from table1 inner join table2 on table1.column =table2.column```

### 外连接

- 左外连接（LEFT OUTTER JOIN）
- 右外连接（RIGHT OUTTER JOIN）
- 全外连接（FULL OUTTER JOIN）

外连接语句

```select filedlist from table1 outer join table2 on table1.column =table2.column```

### 为什么使用索引？

- 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。
- 可以大大加快数据的检索速度，这也是创建索引的最主要的原因。
- 帮助服务器避免排序和临时表
- 将随机IO变为顺序IO
- 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。

### 索引这么多优点，为什么不对表中的每一个列创建一个索引呢？

- 当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。
- 索引需要占物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立簇索引，那么需要的空间就会更大。
- 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加

### 使用索引的注意事项？

- 在经常需要搜索的列上，可以加快搜索的速度；
- 在经常需要排序的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间
- 在中到大型表索引都是非常有效的，但是特大型表的维护开销会很大，不适合建索引
- 避免where子句中对字段施加函数，这会造成无法命中索引
- 在使用InnoDB时使用与业务无关的自增主键作为主键，即使用逻辑主键，而不要使用业务主键。
- **将打算加索引的列设置为NOT NULL，否则将导致引擎放弃使用索引而进行全表扫描**
- 删除长期未使用的索引，不用的索引的存在会造成不必要的性能损耗
- 在使用limit offset查询缓存时，可以借助索引来提高性能。

### 主键索引和唯一索引的区别

- 主键索引是唯一索引的一种特殊类型
- 主键索引用于唯一标识表中的每一行，唯一索引则是不允许两行出现相同的索引值
- 主键索引不可为空，唯一索引可以
- 主键可以作为外键，唯一索引不行

### 索引失效的情况

- 违反最左前缀法则（即查询从索引的最左前列开始并且不跳过索引中的列）
- 在索引上做任何操作（比如计算，函数）
- 使用！=
- like后以通配符%开头
- 字符串不加单引号
- or连接

### count(1) count(*) count(列值)的区别

- count(1)包括了忽略所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL  

- count(*)包括了所有的列，相当于行数，在统计结果的时候，**不会忽略列值为NULL**  
- count(列名)只包括列名那一列，在统计结果的时候，**会忽略列值为NULL的计数**，即某个字段值为NULL时，不统计。

如果按照效率比较的话：

count(*)=count(1)>count(primary key)>count(column)

### drop,delete与truncate的区别

drop直接删掉表;truncate删除表中数据，再插入时自增长id又从1开始 ;delete删除表中数据，可以加where字句

### 什么是覆盖索引?

如果一个索引包含（或者说覆盖）所有需要查询的字段的值，我们就称 之为“覆盖索引”。我们知道在InnoDB存储引 擎中，如果不是主键索引，叶子节点存储的是主键+列值。最终还是要“回表”，也就是要通过主键再查找一次,这样就 会比较慢。覆盖索引就是把要查询出的列和索引是对应的，不做回表操作！

### MySQL数据库cpu飙升到500%的话他怎么处理？

当 cpu 飙升到 500%时，先用操作系统命令 top 命令观察是不是 mysqld 占用导致的，如果不是，找出占用高的进程，并进行相关处理。

如果是 mysqld 造成的， show processlist，看看里面跑的 session 情况，是不是有消耗资源的 sql 在运行。找出消耗高的 sql，看看执行计划是否准确， index 是否缺失，或者实在是数据量太大造成。

一般来说，肯定要 kill 掉这些线程(同时观察 cpu 使用率是否下降)，等进行相应的调整(比如说加索引、改 sql、改内存参数)之后，再重新跑这些 SQL。

也有可能是每个 sql 消耗资源并不多，但是突然之间，有大量的 session 连进来导致 cpu 飙升，这种情况就需要跟应用一起来分析为何连接数会激增，再做出相应的调整，比如说限制连接数等

### 大表怎么优化？某个表有近千万数据，CRUD比较慢，如何优化？分库分表了是怎么做的？分表分库了有什么问题？有用到中间件么？他们的原理知道么？

当MySQL单表记录数过大时，数据库的CRUD性能会明显下降，一些常见的优化措施如下：

1. **限定数据的范围：** 务必禁止不带任何限制数据范围条件的查询语句。比如：我们当用户在查询订单历史的时候，我们可以控制在一个月的范围内。；
2. **读/写分离：** 经典的数据库拆分方案，主库负责写，从库负责读；
3. **缓存：** 使用MySQL的缓存，另外对重量级、更新少的数据可以考虑使用应用级别的缓存；

还有就是通过分库分表的方式进行优化，主要有垂直分表和水平分表

1. **垂直分区：**

    **根据数据库里面数据表的相关性进行拆分。** 例如，用户表中既有用户的登录信息又有用户的基本信息，可以将用户表拆分成两个单独的表，甚至放到单独的库做分库。

    **简单来说垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。** 如下图所示，这样来说大家应该就更容易理解了。

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC82LzE2LzE2NDA4NDM1NGJhMmUwZmQ?x-oss-process=image/format,png)

    **垂直拆分的优点：** 可以使得行数据变小，在查询时减少读取的Block数，减少I/O次数。此外，垂直分区可以简化表的结构，易于维护。

    **垂直拆分的缺点：** 主键会出现冗余，需要管理冗余列，并会引起Join操作，可以通过在应用层进行Join来解决。此外，垂直分区会让事务变得更加复杂；

    #### 垂直分表

    把主键和一些列放在一个表，然后把主键和另外的列放在另一个表中

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy90dVNhS2M2U2ZQcjh0NFBaVVFJVUszVHl0aWF3T0VRa2dFZnBCTm4xZkNtdEVhMkRaNTlISFNSaWN2SEIzeU43Yk5LY1hkc3NWZGFNb25TOEFKanY5cFdBLzY0MA?x-oss-process=image/format,png)

    ##### 适用场景

    - 1、如果一个表中某些列常用，另外一些列不常用
    - 2、可以使数据行变小，一个数据页能存储更多数据，查询时减少I/O次数

    ##### 缺点

    - 有些分表的策略基于应用层的逻辑算法，一旦逻辑算法改变，整个分表逻辑都会改变，扩展性较差
    - 对于应用层来说，逻辑算法增加开发成本
    - 管理冗余列，查询所有数据需要join操作

2. **水平分区：**

    **保持数据表结构不变，通过某种策略存储数据分片。这样每一片数据分散到不同的表或者库中，达到了分布式的目的。 水平拆分可以支撑非常大的数据量。**

    水平拆分是指数据表行的拆分，表的行数超过200万行时，就会变慢，这时可以把一张的表的数据拆成多张表来存放。举个例子：我们可以将用户信息表拆分成多个用户信息表，这样就可以避免单一表数据量过大对性能造成影响。

    ![数据库水平拆分](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC82LzE2LzE2NDA4NGI3ZTllNDIzZTM?x-oss-process=image/format,png)

    水品拆分可以支持非常大的数据量。需要注意的一点是:分表仅仅是解决了单一表数据过大的问题，但由于表的数据还是在同一台机器上，其实对于提升MySQL并发能力没有什么意义，所以 **水平拆分最好分库** 。

    水平拆分能够 **支持非常大的数据量存储，应用端改造也少**，但 **分片事务难以解决** ，跨界点Join性能较差，逻辑复杂。

    《Java工程师修炼之道》的作者推荐 **尽量不要对数据进行分片，因为拆分会带来逻辑、部署、运维的各种复杂度** ，一般的数据表在优化得当的情况下支撑千万以下的数据量是没有太大问题的。如果实在要分片，尽量选择客户端分片架构，这样可以减少一次和中间件的网络I/O。

    #### 水平分表：

    表很大，分割后可以降低在查询时需要读的数据和索引的页数，同时也降低了索引的层数，提高查询次数

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy90dVNhS2M2U2ZQcjh0NFBaVVFJVUszVHl0aWF3T0VRa2dkQVpyU1Y3M2liMWZkRENYS2M3QUd6Wmhid3FjS0ZVWkpGWThwMFZkVXRPM3JNYzZ2eDFBdzVBLzY0MA?x-oss-process=image/format,png)

    ##### 适用场景

    - 1、表中的数据本身就有独立性，例如表中分表记录各个地区的数据或者不同时期的数据，特别是有些数据常用，有些不常用。
    - 2、需要把数据存放在多个介质上。

    ##### 水平切分的缺点

    - 1、给应用增加复杂度，通常查询时需要多个表名，查询所有数据都需UNION操作
    - 2、在许多数据库应用中，这种复杂度会超过它带来的优点，查询时会增加读一个索引层的磁盘次数

    **下面补充一下数据库分片的两种常见方案：**

    - **客户端代理：** **分片逻辑在应用端，封装在jar包中，通过修改或者封装JDBC层来实现。** 当当网的 **Sharding-JDBC** 、阿里的TDDL是两种比较常用的实现。
    - **中间件代理：** **在应用和数据中间加了一个代理层。分片逻辑统一维护在中间件服务中。** 我们现在谈的 **Mycat** 、360的Atlas、网易的DDB等等都是这种架构的实现。

**分库分表后面临的问题**

- **事务支持** 分库分表后，就成了分布式事务了。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价； 如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。

- **跨库join**

    只要是进行切分，跨节点Join的问题是不可避免的。但是良好的设计和切分却可以减少此类情况的发生。解决这一问题的普遍做法是分两次查询实现。在第一次查询的结果集中找出关联数据的id,根据这些id发起第二次请求得到关联数据。 分库分表方案产品

- **跨节点的count,order by,group by以及聚合函数问题** 这些是一类问题，因为它们都需要基于全部数据集合进行计算。多数的代理都不会自动处理合并工作。解决方案：与解决跨节点join问题的类似，分别在各个节点上得到结果后在应用程序端进行合并。和join不同的是每个结点的查询可以并行执行，因此很多时候它的速度要比单一大表快很多。但如果结果集很大，对应用程序内存的消耗是一个问题。

- **数据迁移，容量规划，扩容等问题** 来自淘宝综合业务平台团队，它利用对2的倍数取余具有向前兼容的特性（如对4取余得1的数对2取余也是1）来分配数据，避免了行级别的数据迁移，但是依然需要进行表级别的迁移，同时对扩容规模和分表数量都有限制。总得来说，这些方案都不是十分的理想，多多少少都存在一些缺点，这也从一个侧面反映出了Sharding扩容的难度。

- **ID问题**

    一旦数据库被切分到多个物理结点上，我们将不能再依赖数据库自身的主键生成机制。某个分区数据库自生成的ID无法保证在全局上是唯一的。因此生成id的方法常见的有：

    **UUID** 使用UUID作主键是最简单的方案，但是缺点也是非常明显的。由于UUID非常的长，除占用大量存储空间外，最主要的问题是在索引上，在建立索引和基于索引进行查询时都存在性能问题。 

    **Snowflake雪花算法**整个ID是64位的，41位的时间戳精确到毫秒，10位的工作机器ID，12位的序列号，这个值是在同一毫秒同一节点上从0开始不断累加，最多可以累加到4095。因此一毫秒时间内可以产生2^10*2^12个id。**唯一的缺点就是由于id和机器的时间有关，如果机器时间回拨则可能发生id重合。**![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGovUVwFkfA0yRdCYoer9mqxdkKsBd5aD96r6ygicrXlKjwmsIBCZpF4rrkUM7FR1U1zZdL4yjEF1Fw/640?wx_fmt=png)

- 跨分片的排序分页

    般来讲，分页时需要按照指定字段进行排序。当排序字段就是分片字段的时候，我们通过分片规则可以比较容易定位到指定的分片，而当排序字段非分片字段的时候，情况就会变得比较复杂了。为了最终结果的准确性，我们需要在不同的分片节点中将数据进行排序并返回，并将不同分片返回的结果集进行汇总和再次排序，最后再返回给用户。如下图所示：

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200310170753848.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RoaW5rV29u,size_16,color_FFFFFF,t_70)

### MySQL的复制原理以及流程

**MySQL主从复制工作原理**

- 在主库上把数据更高记录到二进制日志
- 从库将主库的日志复制到自己的中继日志
- 从库读取中继日志的事件，将其重放到从库数据中

**主从复制的作用**

1. 主数据库出现问题，可以切换到从数据库。
2. 可以进行数据库层面的读写分离。
3. 可以在从数据库上进行日常备份。

**MySQL主从复制解决的问题**

- 数据分布：随意开始或停止复制，并在不同地理位置分布数据备份
- 负载均衡：降低单个服务器的压力
- 高可用和故障切换：帮助应用程序避免单点失败
- 升级测试：可以用更高版本的MySQL作为从库

**基本原理流程，3个线程以及之间的关联**

**主**：binlog线程——记录下所有改变了数据库数据的语句，放进master上的binlog中；

**从**：io线程——在使用start slave 之后，负责从master上拉取 binlog 内容，放进自己的relay log中；

**从**：sql执行线程——执行relay log中的语句；

**复制过程**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC85LzIxLzE2NWZiNjgzMjIyMDViMmU?x-oss-process=image/format,png)

Binary log：主数据库的二进制日志

Relay log：从服务器的中继日志

第一步：master在每个事务更新数据完成之前，将该操作记录串行地写入到binlog文件中。

第二步：salve开启一个I/O Thread，该线程在master打开一个普通连接，主要工作是binlog dump process。如果读取的进度已经跟上了master，就进入睡眠状态并等待master产生新的事件。I/O线程最终的目的是将这些事件写入到中继日志中。

第三步：SQL Thread会读取中继日志，并顺序执行该日志中的SQL事件，从而与主数据库中的数据保持一致。

