### 数据结构

group：8 服务器 * 4 块盘。

set：同一 group 中不同服务器的同一盘序号，例如：[server1 node1, server2 node1,..., server7 node1, server8 node1] 构成一个 set。

冗余：2

这个 group 共有 4 个 set，每个文件存储在一个 set 中，每次分成 6 数据块，再计算 2 校验块。

### 扩容（rebalance）

每次扩容单位是 group，且每台服务器的盘数固定也是 4。如增加 8 * 4 或 16 * 4；

新增盘会从其他 set （记为 set1）中移动一些文件至新 set（记为 set2）。

1. 查看每个 set 的剩余可用容量，找到最小的，准备进行数据迁移。
2. 在该 set1 上选取 1G 的文件，如果文件都大于 1G，换一个 set。将数据复制到新 set2
3. 更新数据库中该文件的数据块的路径至新 set
4. 删除 set1 中该文件。

至 ABS（set2 剩余可用容量 - set1 剩余可用容量）< 10%，set2 置状态为可用。

#### 扩容时异常重启



### 上传

1. 请求 coor，插入数据库，获取锁；
2. 为文件找一个 set；
3. 在 coor 进行数据分片，例如，每个 set 有 8 块盘，配置冗余是 2，则将数据切分成 6 块，分别放到 6 块盘中，并且在客户端进行 reed-solomon 算法计算 2 冗余块内容，并分别上传至剩余 2 块盘。
4. coor 对该文件解锁。

### 下载

1. 请求 coor，获取数据所在 set 信息，如果所有服务器、磁盘都可用，就取所有数据块即可；若存在损坏块，就从数据库、校验块共 6 块，在客户端进行高斯消元求解损坏块。

### 坏盘替换，数据处理

* 坏盘中的数据块，根据高斯消元从剩余数据块+校验块中恢复（需从先拿到所有块写入本地磁盘，计算完再删掉）
* 坏盘中的校验块，mysql 中记录了缺失的是第几块，在这做个行列式运算即可。
* 多块坏盘，丢失同一文件既有数据块又有校验块，分别同上。对于校验块，需要先还原数据块后再计算。

### todo

- [ ] 列出上传下载速率指标，几台机器，多大文件，多块速率读写 
  - [ ] 8 服务器 * 4 盘 = 8 * 4 * 4T = 128 T，实际容量 = 128T * 0.75 = 96T。
- [ ] minio高可用：erasure code，highway hash，reed-solomon code。n 份原始数据 + m 份备份数据，能通过 n+m 中的任意 n 份数据还原数据。todo 查论文。
  - [ ] **范德蒙矩阵计算校验和**<img src="/Users/kyrie/Desktop/Screen Shot 2023-08-17 at 02.08.17.png" alt="Screen Shot 2023-08-17 at 02.08.17" style="zoom:40%;" />
  - [ ] **高斯消去法还原（行列式计算）**<img src="/Users/kyrie/Desktop/Screen Shot 2023-08-17 at 02.05.55.png" alt="Screen Shot 2023-08-17 at 02.05.55" style="zoom:40%;" />
  - [ ] 使用伽罗瓦场进行运算


### Why

* 高可用性，down 机
  * minio高可用：erasure code，highway hash，reed-solomon code。n 份原始数据 + m 份备份数据，能通过 n+m 中的任意 n 份数据还原数据。todo 查论文。

* 单机性能瓶颈
* 扩容（动态添加 group）
* 容错性（同一组，各机器互为备份）
* 允许多台计算机共享和访问文件

举例：HDFS，GFS(Google File System), Ceph

### What

文件存储、文件同步、文件下载

coor 是对等的，2台。

同一 group 内容相同，不同 group 内容不同。

group 内剩余容量，以最小的机器为准，因此建议机器配置相同，以免浪费。

### 问题

* group 内增加机器，自动同步，同步完成后，系统自动将新增的服务器切换至线上提供服务。如何实现的
* 扩容，动态添加 group，如何实现的
* 一台机器down，如何自动切换，并提醒
* 如何处理并发访问
* 负载均衡，不同 group 剩余存储最大优先？
* 多个 coor 如何工作，挂了如何切换？
* coor、contentserver，可随时加入、退出，原理是什么？
* 上传失败，如何处理；重试 2 次后报错。
* 断点续传

### 遇到的印象深刻的问题并如何解决

从 issue 中找

### 考点

* 同一组内，上传完一台机器即为成功，后台线程同步至其他机器；写文件同时，写 binlog，只记录元数据，用于后台同步（？）。
* 如何处理并发访问？
  * 同一时间只有一个客户端能够修改文件：文件锁、乐观锁、分布式锁
  * coor 处理文件锁，一台 coor 会有单点故障问题，多台有同步问题？
* 如何处理网络中的丢包和超时？
  - 可以使用超时机制来检测丢包和超时情况，并根据具体情况进行重传或重新请求。此外，可以实现一些错误检测和纠正机制，如使用校验和或冗余数据等。
* 如何实现负载均衡和故障恢复？
  - 负载均衡可以通过在分布式文件系统中使用负载均衡算法，如轮询、随机选择或基于性能的选择来分发请求。故障恢复：**冗余存储**、故障转移策略。
* 数据传输协议？
  * HTTP？
* 如何确保分布式文件系统的安全性？
  * 可以采用多层次的安全措施，如身份验证、访问控制列表（ACL）、数据加密和安全传输协议（如TLS/SSL）。另外，定期进行安全审计和漏洞扫描也是保障系统安全的重要措施。

### Key

* 不同的块可以被存储在不同的存储节点上，以实现数据的并行读写和负载均衡

### HighwayHash

利用现代计算机的 SIMD（Single Instruction, Multiple Data）架构，利用计算机硬件的并行性和数据并行性，通过同时处理多个数据块来加速哈希计算。

md5(message digest algorithm 5, 128-bit)：弱hash

SHA-256(secure hash algorithm 256-bit)：强hash

源码：利用一些与或非异或操作和移位操作，将每4字节（传入的key: a0,a1,a2,a3）与固定的4字节进行逻辑操作得到hash值。

底层使用 SIMD 指令，一次操作多个数据，加快速度。

### SIMD

Single Instruction Multi Data 一条指令处理多条数据。有点像向量（矩阵）运算。

![img](http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/CellProgrammingTutorial.files/image008.jpg)

### 分布式知识点

#### 选主算法

如果每个节点都可以写数据，这样容易造成数据的不一致，所以需要选举一个leader，往leader节点中写数据，然后同步到follower节点中。这样就能更好的保证一致性。

##### Raft算法

##### Bully 算法

![img](https://static001.geekbang.org/resource/image/fc/b8/fc0f00a3b7c9290bc91cb4d8721dc6b8.png?wh=571*378)

##### ZAB(zookeeper atomic broadcast) 算法

#### 分布式事务

2PC（2 Phase Commit），eg. mysql 

#### 分布式锁

##### 基于数据库

##### 基于内存

### 可靠性（Reed-solomon codes）

SEC-DED(Single Error Correcting - Double Error Detect)







