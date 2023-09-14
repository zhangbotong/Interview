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

6. like %x 使用索引的特殊情况？ -- 只有 id, name 且name有索引，where name=%x时，走的是二级索引的全扫描，虽说是全扫面，但也是走了二级索引的。因为主键索引除了有数据还有事物、mvcc这些额外东西，所以用二级索引扫更优。

# 事务

## 事务特性（ACID）

|               | 概念             | 实现方式                                         |
| ------------- | ---------------- | ------------------------------------------------ |
| Atomicity     | 全完成或回滚     | undo log（用于回滚，mvcc也用到了 undo）          |
| Consistency   | 略               |                                                  |
| **Isolation** | 多个事物并发读写 | 快照读（**mvcc**） + 当前读（**next key lock**） |
| Durability    | 持久化磁盘       | redo log                                         |

**BASE**：Basically、Available Soft Sate、Eventual consistency

## 并发事务引发问题

### 脏读（dirty read）

读到另一个未提交事物修改的数据。

### 不可重复读（non-repeatable read）

前后两次读取统一数据不一致，如读到另一提交事物修改的数据。

### **幻读（phantom read）**

多次查询某个符合查询条件的「记录数量」，前后两次查询的记录数量不一样，像发生了幻觉。

## 解决问题（隔离级别）

<img src="https://cdn.xiaolincoding.com//mysql/other/4e98ea2e60923b969790898565b4d643.png" style="zoom:45%;" />

### 读未提交（read uncommitted）

### 读提交（read committed）-- 解决读未提交（脏读）

### **可重复读（repeatable read）** -- 解决不可重复读（幻读也是小概率）

#### MVCC（快照读）

RR 在开启事物时生成快照，RC 每次读之前生成快照



<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png" style="zoom:40%;" />

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/ReadView.drawio.png" alt="img" style="zoom:60%;" />

不可见（未提交事物）：

* try_id >= max_trx_id
* 在 m_ids 里的事物

可见（已提交事物）：

* trx_id < min_trx_id
* min_trx_id < trx_id < max_trx_id且 trx_id 不在 m_ids 里。这部分是比 min 后启动的事物，但在此次事物开始前先提交了。

例如，下图所示，A的修改对于B不可见。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/%E4%BA%8B%E5%8A%A1ab%E7%9A%84%E8%A7%86%E5%9B%BE2.png" alt="img" style="zoom:50%;" />

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

## 行锁

如果查询没走索引，那么会在所有记录上加 next-key lock，相当于锁表了。因此，线上在执行 update、delete、select ... for update 等具有加锁性质的语句，一定要检查语句是否走了索引。

### 基础

* Next-Key Lock（记录锁+间隙锁）
* 记录锁
* 间隙锁
* 插入意向锁

**间隙锁之间互相不冲突**，A，B可以加同样范围的间隙锁；

**加锁对象是索引，加锁基本单位是 next-key lock(]**

**加锁的目的是解决幻读，因此锁范围是否退化，看是否出现幻读就行**

### 举例

|            | 唯一索引                                                     | 非唯一索引                                                   |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 等值存在   | 记录锁                                                       | 锁**左右间隙加记录**（就一个目的就是不幻读）                 |
| 等值不存在 | 间隙锁                                                       | 间隙锁                                                       |
| >          | 大于就加 next-key lock                                       | **同左**                                                     |
| >=         | 大于就加 next-key lock；等于退化为记录锁                     | 大于就加 next-key lock；等于也加 next-key lock               |
| <          | 小于就加 next-key lock；第一个不满足退化为间隙锁             | **同左**                                                     |
| <=         | 小于就加 next-key lock；等于加 next-key lock；大于时加间隙锁。等于或大于停止 | 小于就加 next-key lock；等于加 next-key lock；大于停止并加间隙锁（加锁一致，但等于时不停） |

**二级索引同时在本索引和主键索引加锁，主键索引加记录锁。**

#### 1、唯一索引+等值查询（值存在加记录锁，不存在加间隙锁）

1. 查询值存在（锁该条记录）

   由于唯一索引，不存在幻读，next-key lock 退化为记录锁。

2. 查询值不存在（锁值在的间隙）

> ```sql
> mysql> select * from user where id = 2 for update;
> ```

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E9%97%B4%E9%9A%99%E9%94%81.drawio.png" alt="img" style="zoom:40%;" />

锁 (1,5)

#### 2、唯一索引+范围查询

##### 大于（大于就加 next-key lock）

> ```sql
> mysql> select * from user where id > 15 for update;
> ```

* 15 = 15，不满足，不加锁；
* 20 > 15，满足，加 next-key lock
* 直至最后都加 next-key lock

**最终锁范围(15, + ∞)**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%E5%A4%A7%E4%BA%8E15.drawio.png" alt="img" style="zoom:40%;" />

##### 大于等于（等于加记录锁，大于加 next-key lock）

1. 存在等于记录

   ```sql 
   mysql> select * from user where id >= 15 for update;
   ```

   15 = 15，满足且等于，退化为记录锁；其后所有记录加 next-key lock。

​		**最终锁 [15, +∞)**

​		<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%E5%A4%A7%E4%BA%8E%E7%AD%89%E4%BA%8E15.drawio.png" alt="img" style="zoom:40%;" />

2. 不存在等于记录（大于加 next-key lock）	

   ```sql
   mysql> select * from user where id >= 18 for update;
   ```

   扫描到 20 > 18，在 20 及其后所有记录上加 next-key lock。

​		**最终锁(15, +∞)**

##### 小于（小于加next-key lock，第一个不满足条件记录加间隙锁）

1. 不存在等值记录

   ```sql
   mysql> select * from user where id < 6 for update;
   ```

   * 1 < 6，next-key lock，继续向后

   * 5 < 6，next-key lock，继续向后

   * 10 > 6，退化为间隙锁，停止

​		**锁范围(-∞, 10)**

​	<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%E5%B0%8F%E4%BA%8E%E7%AD%89%E4%BA%8E6.drawio.png" alt="img" style="zoom:40%;" />

2. 存在等值记录

   ```sql
   mysql> select * from user where id < 5 for update;
   ```

   * 1 < 5，next-key lock，继续向后

   * 5 = 5，退化为间隙锁，停止向后

 		**最终锁范围(-∞, 5)**

​	<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%E5%B0%8F%E4%BA%8E5.drawio.png" alt="img" style="zoom:40%;" />

##### 小于等于（小于加 next-key lock、等于加 next-key lock、大于退化为间隙锁，遇到等于或大于的记录停止向后）

1. 存在等值记录

   ```sql
   mysql> select * from user where id <= 5 for update;
   ```

   * 1 < 5，加锁，向后

   * 5 = 5，加锁并退化为记录锁，停止

​		共锁 **(-∞, 5]**

​	<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%E5%B0%8F%E4%BA%8E%E7%AD%89%E4%BA%8E5.drawio.png" alt="img" style="zoom:40%;" />

2. 不存在等值记录

   ```sql
   mysql> select * from user where id <= 8 for update;
   ```

   * 1 < 8，加锁，向后；

   * 5 < 8，加锁，向后；

   * 10 > 8，加锁退化为间隙锁，停止； 

​		共锁 **(-∞, 10)**。

#### 3、非唯一索引+等值查询（记录不存在时锁值所在间隙；记录存在时锁该记录的左右间隙，同时在主键索引加记录锁）

1. 记录不存在

   二级索引上符合条件的记录也会在其主键索引上加记录锁。

   ```sql
   mysql> select * from user where age = 25 for update;
   ```



​		<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E7%AD%89%E5%80%BC%E6%9F%A5%E8%AF%A2age=25.drawio.png" alt="img" style="zoom:40%;" />

​		在 index_age 上的 39 上加锁，由于 39 != 25，所以退化为间隙锁。

​		锁范围：(22(id=10), 39(id=20))

​		边界值：

| age  | id   | 是否在锁范围 |
| ---- | ---- | ------------ |
| 22   | 9    | 否           |
| 22   | 11   | 是           |
| 39   | 19   | 是           |
| 39   | 21   | 否           |

2. 记录存在

```sql
mysql> select * from user where age = 22 for update;
```

index_age 上对 22 加锁 (21, 22]，对 39 加间隙锁 (22, 39)

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E7%AD%89%E5%80%BC%E6%9F%A5%E8%AF%A2%E5%AD%98%E5%9C%A8.drawio.png" alt="img" style="zoom:50%;" />

**锁范围：(21, 39)，并对主键在 id=10（二级索引符合条件） 加记录锁**

| age  | id   | 是否落在锁范围 |
| ---- | ---- | -------------- |
| 21   | 4    | 否             |
| 21   | 6    | 是             |
| 39   | 19   | 是             |
| 39   | 21   | 否             |

#### 4、非唯一索引+范围查询（）

**设计算法逻辑目的是为了避免幻读，所以站在高点看，不用去看算法如何实现，只要在逻辑上能避免幻读就是他的实现逻辑。**

##### 大于（大于加 next-key lock）

```sql
mysql> select * from user where age > 22  for update;
```

锁范围：(22, +∞)

* 所有满足条件的节点上加 next-key lock
* 由于是">"，22 不满足条件且在左侧，所以即使存在 22 也不会在其上加锁，就更没有退化一说了

##### 大于等于（大于、等于都加 next-key lock）

若值存在，则第一个需要加锁的就是该值。

```sql
mysql> select * from user where age >= 22  for update;
```

**锁 (21, +∞)**

* 所有大于条件的节点上加 next-key lock
* 等于条件的节点也加 next-key lock
  * 若退化为记录锁，由于不具有唯一性，只锁该记录，但仍可以插入相同值的记录
  * 若退化为间隙锁，别扯淡锁你干啥呢

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E8%A1%8C%E7%BA%A7%E9%94%81/%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2age%E5%A4%A7%E4%BA%8E%E7%AD%89%E4%BA%8E22.drawio.png" alt="img" style="zoom:40%;" />

##### 小于（小于加 next-key lock，第一个不满足条件的加间隙锁）

```sql
mysql> select * from user where age < 22  for update;
```

**锁 (-∞, 22)**

* 所有满足条件的节点上加 next-key lock
* 第一个不满足条件的节点
  * 大于（记录不存在），加间隙锁
  * 等于（记录存在），加间隙锁

##### 小于等于（小于、等于加 next-key lock，第一个大于停止加间隙锁）

```sql
mysql> select * from user where age <= 22  for update;
```

锁 （-∞, 39）

* 小于、等于节点加 next-key lock
* 第一个不满足条件的节点 age=39 加间隙锁

## 表锁

**行锁与表锁读读共享、读写互斥、写写互斥**

* 表锁
* MDL（元数据锁，锁表结构的）
  * CRUD，对表加的 MDL 读锁
  * 表结构变更，对表加 MDL 写锁
* 意向锁，行锁会加表级意向锁用于在申请表锁时不至于遍历所有索引的所有记录。意向锁（表锁）不会和行锁冲突，意向锁之间也不会冲突，只会和表锁冲突。如果没有意向锁，那么在加表锁时需要先判断有没有行锁，就得遍历所有索引；加了意向锁，直接查表级意向锁即可。（快速判断是否有行锁）

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

|      | Undo log（原子性） | redo log（持久性） | Binlog             |
| ---- | ------------------ | ------------------ | ------------------ |
| 用途 | 事物回滚，MVCC     | 掉电恢复           | 数据备份、主从复制 |
|      |                    |                    | Statement, row     |

## Undo log（历史版本）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%89%88%E6%9C%AC%E9%93%BE.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" style="zoom:50%;" />

### 用途

1. 事务回滚，保证原子性
2. 实现MVCC ，看到正确的版本

### 持久化

在 Buffer pool 里有 undo buffer，其实就是数据版本，也通过 redo log 保证持久化

## Redo log

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/wal.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" style="zoom:40%;" />

### 用途（Why）

事务提交后，发生崩溃，进行恢复

* MySQL down
* 服务器 down

### 基础

#### 流程

redo buffer -- page cache(os) -- 磁盘

#### 刷盘

##### 刷盘时机

1. MySQL 正常关闭
2. 时间：每1s
3. 空间：buffer 满
4. 参数控制每次事务提交刷盘时（参数=1）

##### 刷盘参数

0：在 redo 缓存（MySQL down、服务器 down 数据都没了）

1：在磁盘

2：在 page cache（服务器 down 数据没）

**后台线程每隔1s进行 write（写到 page cache） 和 fsync （os 刷盘）**

### redo 与 undo 的不同

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%BA%8B%E5%8A%A1%E6%81%A2%E5%A4%8D.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="事务恢复" style="zoom:70%;" />

事务提交前崩溃 -- undo 历史版本回滚

事务提交后崩溃 -- redo 重放

### 常见问题

1. undo 要记录到 redo 吗？ -- 需要
2. redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？-- **顺序写**替代**随机写**

## Binlog

### 基础

* 过程：binlog 缓存 -- page cache(os) -- 磁盘

  <img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png" alt="binlog cach" style="zoom:60%;" />

* binlog 在 server 层，所以有线程概念，1 个线程 1 个 binlog 缓存

* **事务 commit 时，一次性写至os缓存并刷盘**

### 写入时机

commit 前仅写 binlog 缓存，commit 时写os缓存并持久化到磁盘

#### 文件格式

|                       | 描述                    | 缺点                                          |
| --------------------- | ----------------------- | --------------------------------------------- |
| STATEMENT（默认格式） | 记录 SQL                | sql 用了 now() 会主从不一致                   |
| ROW                   | 记录每行最终结果        | 臃肿，如 1条sql update 多行数据，会有多行记录 |
| MIXED                 | 无 now 选1， 有 now 选2 |                                               |

#### 刷盘

##### 刷盘参数

0：每次 commit 只到 page cache

1：每次 commit 到磁盘

N：每次 commit 到 page cache，N次commit后到磁盘

#### 主从复制

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="MySQL 主从复制过程" style="zoom:60%;" />

* 异步、半同步（半数从库成功即可）
* 有延迟
* 从库多有问题：主库的 log dump 线程是每个从库 1 个，占主库线程资源、网络资源。

## 两阶段提交

### 过程

**下图的写入都指 fsync**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4%E5%B4%A9%E6%BA%83%E7%82%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="时刻 A 与时刻 B" style="zoom:50%;" />

- **prepare 阶段**：将 XID（内部 XA 事务的 ID） 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘（innodb_flush_log_at_trx_commit = 1 的作用）；
- **commit 阶段**：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘（sync_binlog = 1 的作用），接着将 redo log 状态设置为 commit。（此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功）

### 异常分析

不管是时刻 A（redo log 已经写入磁盘， binlog 还没写入磁盘），还是时刻 B （redo log 和 binlog 都已经写入磁盘，还没写入 commit 标识）崩溃，**此时的 redo log 都处于 prepare 状态**。在 MySQL 重启后会按顺序扫描 redo log 文件，碰到处于 prepare 状态的 redo log，就拿着 redo log 中的 XID 去 binlog 查看是否存在此 XID：

- **如果 binlog 中没有当前内部 XA 事务的 XID，说明 redolog 完成刷盘，但是 binlog 还没有刷盘，则回滚事务**。（图A）
- **如果 binlog 中有当前内部 XA 事务的 XID，说明 redolog 和 binlog 都已经完成了刷盘，则提交事务**。（图B）

### binlog 组提交

#### 过程

- **flush 阶段**：多个事务的 binlog 逐个从缓存 --> page cache
- **sync 阶段**：所有 page cache 一起到磁盘
- **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A41.png" alt="img" style="zoom:40%;" />

Flush 阶段一：组提交持久化 redo log（2PC的 prepare阶段）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A42.png" alt="img" style="zoom:40%;" />

Flush 阶段二：binglog 写至 os 缓存

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/write_binlog.png" alt="img" style="zoom:40%;" />

Sync：binlog 组提交持久化

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A44.png" alt="img" style="zoom:40%;" />

Commit：

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A46.png" alt="img" style="zoom:40%;" />

### redo 组提交

在 binlog 组提价的 flush 阶段，进行 redo 的持久化

## 一个例子串联所有日志

1. 开启事务，写 undo（并写 redo 中的 undo 页）。undo 刷盘了吗？-- undo 其实就是 redo 中的 undo 页，他可能刷了（超过时间、空间）可能没刷，和其一致。
2. 更新记录，先更新 buffer pool（标记脏页），然后写 redo 至缓存。（可能在redo 缓存、os 缓存、磁盘）
3. 写 binlog 至 binlog 缓存
4. commit。
   1. prepare
      1. XID 写入 redo
      2. redo 刷盘
      3. 修改状态为 prepare
   2. commit
      1. XID 写入 binlog
      2. binlog 写到 os 缓存并刷盘
      3. 修改 redo 状态为 commit

## 常见问题

1. 事务没提交的时候，redo log 会被持久化到磁盘吗？-- 会，后台线程每秒持久化
2. MySQL 磁盘 I/O 很高，有什么优化的方法？
   1. 设置 binlog 刷盘参数为 N
   2. 设置 redo 刷盘参数为 2（只写至os缓存不刷盘，后台线程刷）
   3. 修改组提交参数：时间、空间。减少刷盘

# Buffer pool

**buffer pool 与 redo log 一起提高了 mysql 的响应速度。磁盘随机写 --> 缓存写+磁盘顺序写**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpool%E5%86%85%E5%AE%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" style="zoom:60%;" />

## 2 个问题及解决方案：

### 1、预读失效（预读页放LRU头部把真正热点数据都挤出去了） -- 增加 old 区，预读页放到 old 区，用时才放到 young

### 2、全表扫描（只会用到一次的记录被放到了） -- 提高放到 young 的门槛，old 区停留超过 1s 才放到 young

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png" alt="img" style="zoom:67%;" />

## 数据结构（3 个链表：Free List,  Dirty List，LRU List）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpoll_page.png" alt="img" style="zoom:60%;" />

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



# 问题列表

1. 用户更新记录，写到 buffer pool, undo log memory 就算写完了，如果这时 os 没有进行 sync，掉电不是丢数据了吗？ -- commit 必须等待 redo log 写到磁盘才算成功，这时便具有了 crash safe 能力 
1. 慢查询优化？-- 建立索引、索引覆盖、索引下推、优化 sql 语句逻辑、从业务上优化 sql
