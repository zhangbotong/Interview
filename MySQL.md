[TOC]

# 索引

B+ Tree

* 只在 leaf 存数据，中间只存索引，a page can store more data, dimish disk IO. **Small level**, less IO, 矮胖

* 双向链表，适合**范围查询**。比如说我们想知道 12 月 1 日和 12 月 12 日之间的订单，这个时候可以先查找到 12 月 1 日所在的叶子节点，然后利用链表向右遍历，直到找到 12 月12 日的节点，这样就不需要从根节点查询了

Feature

* 联合索引：最左前缀

Index Condition Pushdown(索引下推)：select * from table where a > 1 and b = 2, 联合索引（a，b），从索引找到 a > 1 后，直接过滤掉 b != 2的数据，不用去回表判断。（Extra：using index condition）

Extra：

Using index: 覆盖索引

using index condition : index condition pushdown

索引失效

# 事务

原子性（automatic）: undo log

持久性（durability）：redo log

隔离性（isolation）：MVCC

	* 快照读：MVCC 解决幻读
	* 当前读（select for update）: next-key-lock 加锁解决幻读

一致性（consistency）

RR 在开启事物时生成快照，RC 每次读之前生成快照

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/ReadView.drawio.png" alt="img" style="zoom:80%;" />

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/%E4%BA%8B%E5%8A%A1ab%E7%9A%84%E8%A7%86%E5%9B%BE2.png" alt="img" style="zoom:70%;" />



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
|      | 存储引擎层             | 存粗引擎层         | server 层          |
| 用途 | 事物回滚，MVCC，原子性 | Crash-safe，持久性 | 数据备份、主从复制 |
|      |                        |                    | Statement, row     |

Buffer pool: 

两阶段提交

举例

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4%E5%B4%A9%E6%BA%83%E7%82%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="时刻 A 与时刻 B" style="zoom:70%;" />

# 内存(Buffer pool)

数据结构：3 种页和链表：Free page, Clean page, Dirty page

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpoll_page.png" alt="img" style="zoom:60%;" />

2 个问题：

* 预读失效：预读页放到 old 区，访问时放到 young
* 全表扫描（buffer pool 污染）：提高放到 young 的门槛，old 区停留超过 1s 才放到 young

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png" alt="img" style="zoom:67%;" />

# 常见面试问题
