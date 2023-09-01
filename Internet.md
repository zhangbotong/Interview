[TOC]

# 应用层

## HTTP

<img src="https://camo.githubusercontent.com/8fc01fbb786ca5745f6eeeef9c3200223914ba96df7153aada1a166070ad530f/68747470733a2f2f63646e2e7869616f6c696e636f64696e672e636f6d2f67682f7869616f6c696e636f6465722f496d616765486f73742f2545382541452541312545372541452539372545362539432542412545372542442539312545372542422539432f485454502f32372d48545450332e706e67" alt="img" style="zoom:50%;" />

### HTTP 1.1

### HTTPS

### HTTP 2（长连接，stream 并发）

### HTTP 3（基于 QUIXC，解决：队头阻塞、握手时延、换网重新建立连接）

基于 QUIC（UDP）

HTTP2存在问题：

1. 顺序确认，1、2、3、4 中若 2 丢包， 2、3、4 都需要重传。（基于TCP的有序确认特性，所以想要解决此问题必须釜底抽薪换掉 TCP）

2. 握手时延：TCP、TLS

3. 换网需要重新建立连接，比如手机从无线切换为运营商，由于TCP基于四元组（IP、port），换网肯定 IP 变了，因此需要重新建立连接，会卡顿。

HTTP 3 ，通过基于 UDP 的 QUIC 解决了问题1和3及2的TCP时延，TLS 从 1.2 升级到 1.3 解决了 TLS 时延

## RPC（公司内部通信，即将淘汰）

先有 TCP(1970s)，再有 RPC(1980s)，再有 HTTP(1990s)。由于 TCP 是面向字节流的所以直接用不行看不到边界，所以在70s~90s出现了很多基于 TCP 的协议。

其中 RPC 主要用于客户端到公司自己的服务器间通信（C/S架构），怎么都好说，自己定义一套默认参数；HTTP 用于向其他公司服务器通信（B/S），因此需要更通用，在 header 中定义所有参数，不能有默认了不是自家的。

**性能：HTTP1.1 < RPC < HTTP2 < HTTP3**

## WebSocket（全双工）

WHY

基于 TCP 的全双工协议（HTTP 是半双工）。

TCP 是全双工（同一时刻有2个通道，客户端和服务端可以同时收发包），但 HTTP 设计初衷只是为了【请求-响应】，所以半双工就可以，所以没设计成全双工。但网页游戏有全双工的需求，服务端还要能主动push数据给客户端（野怪走过来），因此便有了 WebSocket。

# 传输层（Transportation Layer）

## TCP

### 重传

#### 超时重传

#### 快速重传（Fast Retrainsmit，3个相同的 ACK 触发快速重传）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/10.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="快速重传机制" style="zoom:50%;" />

包2丢了，包3、4、5到达服务端后回ACK2。Client 收到 3 个相同 ACK，触发快速重传包 2、3、4、5（若开启 SACK，则服务端会记录收到了3、4、5，只会重传包 2）。

#### SACK(Selective ACK)

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/11.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="选择性确认" style="zoom:50%;" />

### 拥塞控制（避免发送速率高于网络速率）

#### 慢启动（窗口翻倍）

#### 拥塞避免（窗口+1）

#### 拥塞发生(ssthresh=cwnd/2, cwnd 变为 1 或减半)

##### 超时重传（网络很拥挤，cwnd=1，ssthresh=cwnd/2）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/29.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="拥塞发送 —— 超时重传" style="zoom:50%;" />

##### 快速重传（快速恢复，网络没那么糟，偶尔丢了一个包，窗口变小别太激进，cwnd=ssthresh=cwnd/2）

TCP 认为这种情况不严重，因为大部分没丢。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/%E6%8B%A5%E5%A1%9E%E5%8F%91%E7%94%9F-%E5%BF%AB%E9%80%9F%E9%87%8D%E4%BC%A0.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="快速重传和快速恢复" style="zoom:50%;" />

- `cwnd = cwnd/2` ，也就是设置为原来的一半;
- `ssthresh = cwnd`;（下面进入快速恢复）
- 拥塞窗口 `cwnd = ssthresh + 3` （ 3 的意思是确认有 3 个数据包被收到了）；
- 重传丢失的数据包；
- 如果再收到重复的 ACK，那么 cwnd 增加 1；（包2 还未收到）
- 如果收到新 ACK 后（包2到了），把 cwnd 设置为第一步中的 ssthresh 的值，，该恢复过程结束，可以回到恢复之前的状态了，也即再次进入拥塞避免状态；

### 流量控制（滑动窗口）（避免发送速率高于接收速率）

#### 发送方窗口（min(接收方窗口，网络窗口)）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/19.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="SND.WND、SND.UN、SND.NXT" style="zoom:50%;" />

#### 接收方窗口

### 3次握手

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4/%E7%BD%91%E7%BB%9C/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B.drawio.png" alt="TCP 三次握手" style="zoom:40%;" />

#### Why（避免连接到历史连接、同步双方序列号 ）

1. client 发 SYN=90
2. client down机
3. client 重启后新发 SYN=91，这时 SYN=90的请求就是错误的历史连接。（如果是SYN丢失会重发一样的SYN，即90，只有down机才会发新的）

<img src="https://cdn.xiaolincoding.com//mysql/other/format,png-20230309230525514.png" alt="三次握手避免历史连接" style="zoom:50%;" />

#### How

1. client 发 **SYN（请求建立client->server 的写通道）**
2. server 发 **ACK（允许建立 client->server的写通道）** + **SYN（请求建立 server->client 的写通道）**
3. client 发 **ACK（允许建立到 client 的写通道）**

至此，两个通道才都建立完成。理论上在 2 之后，client 就可以给 server 发数据且server 也能正常接收；2 之后 server 理论上也能给 client 发数据，但其一 server 没收到 ACK 前不会这么做，因为他知道发了白发人家不收。

### 4次挥手

#### How

<img src="https://cdn.xiaolincoding.com//mysql/other/format,png-20230309230614791.png" alt="客户端主动关闭连接 —— TCP 四次挥手" style="zoom:50%;" />

1. client 发 FIN（我client不写了）
2. server 回 ACK（我server知道你不写了，那我就不读了，client -> server 的通道关闭完成）
3. server 发 FIN（我server也不写了）
4. client 回 ACK（我client知道你也不写了，那我也不读了，server -> client 的通道关闭完成）

在2、3之间，server 可能还在发数据，所以，2、3不能合并发送。

### 优化TCP

#### 优化 3 次握手

* 握1 调整 SYN 重传次数
* 握2 调整 SYN+ACK 重传次数
* 调大半连接队列、全连接（accept）队列
* 设置绕过 3 次握手，用 cokkie。

#### 优化 4 次挥手

* 挥1、3 调整 FIN 重传次数
* 调 TIME_WAIT 
  * 调低 TIME_WAIT 状态的个数，后面断开直接断
  * 同一四元组断开后又新建连接，对于处在 TIME_WAIT 状态的不等直接建新连接

#### 优化传输过程

* 增大缓冲区大小

### TCP 基础（字节流、可靠、面向连接）

**TCP 发包过程若收到非法数据包（比 ACK 还小），会发 RST 断开连接。**

<img src="https://cdn.xiaolincoding.com//mysql/other/format,png-20230309230534096.png" alt="TCP 头格式" style="zoom:50%;" />

* seq：有序
* 确认号：不丢，可靠
* 控制位
  * ACK
  * RST -- 异常断开(Reset)
  * SYN -- 握手，用来同步双方的初始序列号(Synchronized number)
  * FIN -- 挥手

`RTT`（Round-Trip Time 往返时延）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/6.jpg?" alt="RTT" style="zoom:50%;" />

* 字节流（沾包）
* 面向连接（同步初始序列号）
* 可靠（确认号）
* 全双工

### 常见问题

1. TCP和UDP能使用同一个端口吗？-- 能

2. 为什么每次建立 TCP 连接时，初始化序列号要不一样？ -- 为了避免相同四元组的连接，上一 **rst 断开**（正常断开由于有 2MSL 所以不会有旧包还在网络中）下一连上，旧包发到新连接上。（避免新瓶装旧酒：其一正常断开的由 2MSL 保证；其二异常断开的由随机序列号保证）

3. 2次握手为什么不行？-- 只同步了 client 的序列号，server的还未同步。

4. 已建立连接，再收到 SYN 会怎样？server会正常发 ACK，client 收到 ACK 发现跟自己的序列号不一致，则会发 RST 终止现在的连接。<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/est_syn.png" alt="img" style="zoom:25%;" />

5. SYN 什么情况会被丢弃？-- 半连接队列满

6. TCP 挥手为什么等 2MSL(Max Segment Lifetime)？-- 

   1. 保证 server 正确关闭（FIN 的 ACK 收到）
   2. 为了保证网络中不再有此连接的数据包，防止我的包被后面相同四元组建立的TCP连接接收。

   （第四次挥手是开始计时，挥手4 丢失，当过了 1MSL 时，挥手3 重发 FIN，再经过 1MSL 到client，此时距离发4时是 2MSL。在挥手3 前甚至同时发送的数据同理也需要 2MSL 重发才能到 client。）

7. 已建立连接时，client进程崩溃会怎样？-- client 的 os 内核会发起挥手。

8. 已建立连接时，client关机（os崩）会怎样？-- 

   1. server 如有包发给 client （探测包或数据包）
      1. 若发数据时 client 没起来，探测包石沉大海，几次没有响应后server会关闭连接。
      2. 若发数据时 client 起来了，但由于没有连接，所以client会发 RST异常关闭。

9. 挥手时收到 FIN 但其序列号不对怎么处理？-- 处理顺序以序号为准，若序号大，则说明网络中还有数据包没过来，等过来了再继续挥手

10. 拔网线 ，TCP连接还在吗？有探测的，在探测内插上网线，则恢复正常，没插上则会断开；没探测的有发包（相当于有探测）则会断开 ，没发包（无探测）会一直在。

11. reuse 同一四元组连接为什么默认关闭？-- reuse 相当于缩短了 2MSL ，不安全，网络中可能会有残余包。

## UDP

### Why（TCP 的痛点：建立连接耗时、切换网络需重建连接、队头阻塞）

### UDP常见问题

1. **如何基于 UDP 可靠传输？** -- Packet Header、QUIC Frame Header。每个 Packet Header 都不重复，避免了等待确认。Stream ID + Offset 实现有序。<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/network/quic/Packet%E4%B8%A2%E5%A4%B1.jpeg" alt="img" style="zoom:35%;" />

# 网络层

## 基础

## 相关协议

### DNS

### ARP（Adress resolve protocol）

### ICMP(Internet Control Message Protocol，错误信息)

### NAT(Net Adress Translation)



