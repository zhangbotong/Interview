[TOC]

# 索引

## 基础

### 1、联合索引

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E7%B4%A2%E5%BC%95/%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95%E6%A1%88%E4%BE%8B.drawio.png" alt="img" style="zoom:70%;" />

eg. 

1. select * from t_table where a > 1 and b = 2。**只有a用到了联合索引**，b没用到。在找到第一个 a>1 的记录的范围时，b字段是无序的，因此用不到b。
2. select * from t_table where a >= 1 and b = 2。**a,b 都用到了**。其中，在用索引找到第一个a = 1的数据后，针对 a=1 可以用到该索引直接找到 b=2，但是后面的从 a=2 开始的数据b就用不到索引了。
3. SELECT * FROM t_table WHERE a BETWEEN 2 AND 8 AND b = 2。**同2**。
4. SELECT * FROM t_user WHERE name like 'j%' and age = 22。**同2**。

对于有 where 和 order by 或 group by 的，建联合索引能避免数据库发生文件排序（file sort）。如下：

> ```sql
> select * from order where status = 1 order by create_time asc
> ```

建一个 status, create_time 的联合索引比只建 status 的索引好。

### 2、索引pushdown

> select * from table where a > 1 and b = 2

只有a用到了联合索引，在找到 a=2 时，如果没有pushdown，则只能回表去主键索引找到数据行再比较b。如果有pushdown直接在联合索引上判断b满足条件后才去回表。

Extra：Using index condition

### 3、B+ Tree

* 只在 leaf 存数据，中间只存索引，矮胖。

* 中间节点是冗余数据，在插入删除时对树结构影响小，效率高。

* 双向链表，适合**范围查询**。比如说我们想知道 12 月 1 日和 12 月 12 日之间的订单，这个时候可以先查找到 12 月 1 日所在的叶子节点，然后利用链表向右遍历，直到找到 12 月12 日的节点，这样就不需要从根节点查询了

## 索引常见问题

1. 为什么 MySQL InnoDB 选择 B+tree 作为索引的数据结构？

   1. B+Tree VS B Tree

      |        | 是否只在叶子存数据 | 叶子是否有双向链表 | 叶子是否有全量数据 |
      | ------ | ------------------ | ------------------ | ------------------ |
      | B+Tree | 是                 | 是                 | 是                 |
      | B Tree | 否                 | 否                 | 否                 |

      * 只在叶子存数据保证了中间节点空间小，一个page能放更多的节点，更**矮胖**。
      * 插入、删除效率高。中间有冗余节点数据，这样在插入、删除时树形结构变化较小，效率高。

      * 双向链表支持**范围查询**。

   2. B+Tree VS Hash（Hash不支持**范围查询**）

2. 什么情况下**索引失效**？函数、范围查询中的>, <, like 中的 %x 、没有遵循最左前缀原则、or 语句。
3. 有什么优化索引的方法？覆盖索引、索引pushdown、前缀索引
4. mysql 页 = 16KB，os 页 = 4KB
5. MySQL 单表不超2kw行？-- 2kw行内B+树是3层
6. like %x 使用索引的特殊情况？ -- 只有 id, name 且name有索引，where name=%x时，走的是二级索引的全扫描，虽说是全扫面，但也是走了二级索引的。因为主键索引除了有数据还有事物、mvcc这些额外东西，所以用二级索引扫更优。

# 事务

## 事务特性（ACID）

|               | 概念             | 实现方式                                         |
| ------------- | ---------------- | ------------------------------------------------ |
| Atomicity     | 全完成或回滚     | undo log（用于回滚，mvcc也用到了 undo）          |
| Consistency   | 略               |                                                  |
| **Isolation** | 多个事物并发读写 | 快照读（**mvcc**） + 当前读（**next key lock**） |
| Durability    | 持久化磁盘       | redo log                                         |

## 并发事务引发问题

### 脏读（dirty read）

读到另一个未提交事物修改的数据。

### 不可重复读（non-repeatable read）

前后两次读取统一数据不一致，如读到另一提交事物修改的数据。

### **幻读（phantom read）**

多次查询某个符合查询条件的「记录数量」，前后两次查询的记录数量不一样，像发生了幻觉。

## 解决问题（隔离级别）

<img src="https://cdn.xiaolincoding.com//mysql/other/4e98ea2e60923b969790898565b4d643.png" style="zoom:70%;" />

### 读未提交（read uncommitted）

### 读提交（read committed）-- 解决读未提交（脏读）

### **可重复读（repeatable read）** -- 解决不可重复读（幻读也是小概率）

#### undo log

<img src="https://cdn.xiaolincoding.com//mysql/other/f595d13450878acd04affa82731f76c5.png" style="zoom:60%;" />

 版本链，每条记录都有的版本记录链表。

#### MVCC（快照读）

RR 在开启事物时生成快照，RC 每次读之前生成快照



<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png" style="zoom:60%;" />

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/ReadView.drawio.png" alt="img" style="zoom:60%;" />

可见（已提交事物）：

* trx_id < min_trx_id
* min_trx_id < trx_id < max_trx_id且 trx_id 不在 m_ids 里。这部分是比 min 后启动的事物，但在此次事物开始前先提交了。

不可见（未提交事物）：

* try_id >= max_trx_id
* 在 m_ids 里的事物

例如，下图所示，A的修改对于B不可见。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/%E4%BA%8B%E5%8A%A1ab%E7%9A%84%E8%A7%86%E5%9B%BE2.png" alt="img" style="zoom:60%;" />

#### next-key lock（当前读）

如果有其他事务在 next-key lock 锁范围内插入了一条记录，那么这个插入语句就会被阻塞

当前读：delete, select for update , update

#### RR的幻读场景

1. A快照读 -- B更新数据 -- A当前读

   - T1 时刻：事务 A 先执行「快照读语句」：select * from t_test where id > 100 得到了 3 条记录。

   - T2 时刻：事务 B 往插入一个 id= 200 的记录并提交；

   - T3 时刻：事务 A 再执行「当前读语句」 select * from t_test where id > 100 for update 就会得到 4 条记录，此时也发生了幻读现象。

2. A快照读 -- B更新数据 -- A更新同一数据 -- A快照读。虽然A在第一次未查到数据情况下，后面对其进行了更新这有点奇怪，但确实能产生这种操作，此后该数据产生了一个A生成的版本，所以此后A再读都能读到。

### 串行化（serializable ）-- 解决幻读

# 锁

**意向锁的目的是为了快速判断表里是否有记录被加锁**。

gap log 只存在于 RR 下。

Next-key-lock: (]

* Delete、update 都会加 X 锁
* 加锁的**对象**是**索引**，加锁的**基本单位**是 **next-key lock**，加锁的最终**目的**是为了**避免幻读**，在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成退化成记录锁或间隙锁
* **update** 和 **delete** 语句如果查询条件**不加索引**，那么扫描的方式是**全表扫描**，于是就会对每一条记录的索引上都会加 next-key 锁，这样就相当于**锁**住的**全表**
* 两个事务即使生成的间隙锁的范围是一样的，也不会发生冲突，而是会同时产生两个相同的间隙锁，导致死锁
* 解决幻读：快照读 -- MVCC；当前读 -- next-key lock

加锁规则举例：[Mysql 是怎么加锁的？](https://xiaolincoding.com/mysql/lock/how_to_lock.html#%E6%80%BB%E7%BB%93)

# 日志

|      | Undo log               | redo log           | Binlog             |
| ---- | ---------------------- | ------------------ | ------------------ |
|      | 存储引擎层             | 存储引擎层         | server 层          |
| 用途 | 事物回滚，MVCC，原子性 | Crash-safe，持久性 | 数据备份、主从复制 |
|      |                        |                    | Statement, row     |

Buffer pool: 

两阶段提交

举例

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4%E5%B4%A9%E6%BA%83%E7%82%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="时刻 A 与时刻 B" style="zoom:65%;" />

# 内存(Buffer pool)

数据结构：3 种页和链表：Free page, Clean page, Dirty page

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpoll_page.png" alt="img" style="zoom:60%;" />

2 个问题：

* 预读失效：预读页放到 old 区，访问时放到 young
* 全表扫描（buffer pool 污染）：提高放到 young 的门槛，old 区停留超过 1s 才放到 young

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png" alt="img" style="zoom:67%;" />

# 基础

## 执行一条 SQL 的过程

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/sql%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/mysql%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png" alt="img" style="zoom:67%;"/>

- 连接器：建立连接，管理连接、校验用户身份；
- 解析 SQL，通过解析器对 SQL 查询语句进行词法分析、语法分析，然后构建语法树，方便后续模块读取表名、字段、语句类型；
- 执行 SQL：执行 SQL 共有三个阶段：
  - 预处理阶段：检查表或字段是否存在；将 `select *` 中的 `*` 符号扩展为表上的所有列。
  - 优化阶段：基于查询成本的考虑， 选择查询成本最小的执行计划；
  - 执行阶段：根据执行计划执行 SQL 查询语句，从存储引擎读取记录，返回给客户端；

## MySQL 行记录存储结构

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/null%E5%80%BC%E5%88%97%E8%A1%A85.png" alt="img" style="zoom:67%;"/>

### 1、变长字段长度

存放一条记录的 varchar(n) 占用的字节数

### 2、null 字段

eg. 第2行记录如图：<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/null%E5%80%BC%E5%88%97%E8%A1%A84.png" alt="img" style="zoom:40%;"/>

允许为空的每个列占用 1bit，字节对齐。

eg. 

* 1个字段允许为空，占1B
* 2个字段允许为空，占1B
* 9个字段允许为空，占2B

### 3、头信息

* delete_mask：标记是否被删除
* next_record：下条记录位置
* record_type：记录类型

### 4、row_id, try_id, roll_pointer

### 5、常见问题：

- varchar(n) 中 n 最大取值为多少？

  ​	一行数据最大为 65535B，其中包含【变长字段长度】、【null字段】、【所有varchar(n)总长度】eg. varcher(255)和varchar(n)，字段且允许为空中，列1变长字段长度=1，列2变长字段长度2，null=1，则 max(n) = 65535 - 1 - 2 - 1 - 255 = 65276。

- MySQL 的 NULL 值会占用空间吗？-- Of course.

- MySQL 怎么知道 varchar(n) 实际占用数据的大小？ -- 变长字段长度字段存储，每行记录的该字段存该行记录中变长字段实际占用大小（1B最多255）

# 常见面试问题

# 常见排查问题方法

|      | type                                      | key      | key_len | ref  | rows           | Extra                                           |
| ---- | ----------------------------------------- | -------- | ------- | ---- | -------------- | ----------------------------------------------- |
|      | All（全表扫描）                           | 索引名称 |         |      | 扫描的数据行数 | Using index condition -- 索引下推，高效         |
|      | index（全索引扫描）                       |          |         |      |                | using index -- 覆盖索引，高效                   |
|      | range（索引范围扫描）                     |          |         |      |                | Using filesort：文件排序，低效                  |
|      | ref（非唯一索引）                         |          |         |      |                | Using temporary：使用了临时表保存中间结果，低效 |
|      | eq_ref（唯一索引）                        |          |         |      |                |                                                 |
|      | const（结果只有一条的主键或唯一索引扫描） |          |         |      |                |                                                 |



# 疑问

1. 用户更新记录，写到 buffer pool, undo log memory 就算写完了，如果这时 os 没有进行 sync，掉电不是丢数据了吗？ -- commit 必须等待 redo log 写到磁盘才算成功，这时便具有了 crash safe 能力 
