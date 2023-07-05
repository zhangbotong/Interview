## 定义
* C-Consistency：分布式系统所有节点数据强一致
* A-Availability
* P-Partition Tolerance: 分布式系统遇到网络分区的情况
<div align=center><image width = '350' height ='300' src = https://github.com/zhangbotong/Interview/assets/7106986/9d3d1984-0e38-44ad-bafa-086ccfc0b064/></div>
  
## 场景
在分布式系统中，网络分区不可避免，因此分区容错性 P 必须满足
### 正常场景（CAP）
![image](https://github.com/zhangbotong/Interview/assets/7106986/f76ad7ee-1df1-4162-9729-7f5e917b7853)
### CP
牺牲可用性，Server2 让 User2 的请求阻塞，直到网络恢复，DB2 中数据更新为最新值后，再给 User2 响应。
![image](https://github.com/zhangbotong/Interview/assets/7106986/3d8f0ddc-00c4-465e-a679-fd3f9a81d7d0)
应用：银行、金融 - 严谨
### AP
保证可用性，牺牲一致性。Server2 将旧数据 a=1 返给用户，等到网络恢复，再进行数据同步。
![image](https://github.com/zhangbotong/Interview/assets/7106986/44f15fbb-ae1d-48f4-b87f-ad4db3e6e1ae)
应用：电商 - 重用户体验

