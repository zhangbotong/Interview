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

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/%E4%BA%8B%E5%8A%A1ab%E7%9A%84%E8%A7%86%E5%9B%BE2.png" alt="img" style="zoom:80%;" />



# 锁

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

# 内存
# 常见面试问题
