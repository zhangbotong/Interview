Growth 服务 JVM 调优

目录

## 问题

com.sankuai.qcs.service.growth 服务每次启动服务时（弹性机器启动出现过几次），在初始时间都会 young gc 变多，停顿时间长，导致调下游接口超时：

雷达链接：https://radar.mws.sankuai.com/event/detail/2669806/content?from=all

## 排查思路

- 从日志角度，看系统在出问题时刻日志，分析问题可能出现原因；
- 从监控及启动脚本，看当前 jvm 配置信息，分析可能原因；
- 从代码角度，看系统刚启动时，加载了哪些东西到内存，是不是这时占用内存过高；（可能是 lion？）

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-1-gc-count-time.png)

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-2-jvm%E5%9B%9B%E5%86%85%E5%AD%98%E4%BD%BF%E7%94%A8%E6%83%85%E5%86%B5.png)

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-3-%E7%BA%BF%E7%A8%8B%E9%98%BB%E5%A1%9E%E6%83%85%E5%86%B5.png)

## JVM内存排查

ps -ef | grep java 找到 java 进程的 pid；

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-4-%E6%89%BE%E8%BF%9B%E7%A8%8Bpid.png)

jmap -heap 95936 查看堆空间分配及使用情况；

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-5-%E6%9F%A5%E7%9C%8B%E8%BF%9B%E7%A8%8B%E5%A0%86%E7%A9%BA%E9%97%B4.png)

|       | Total  | Used  | Free   |
| ----- | ------ | ----- | ------ |
| Eden  | 162MB  | 88MB  | 74MB   |
| Old   | 1883MB | 785MB | 1097MB |
| Total | 2GB    | 877MB | 1170MB |

结论：Eden 区共 162MB 较小，而 Old 区常驻内存就有 785MB，会导致新启动服务时通过频繁 minor gc 才会往 Old 区移动内容，造成系统停顿时间长。

## 原因推测

本来就需要那么些内存，由于 young 内存小，需不断 gc 后才到 old 区趋于稳定。

## 解决方法

- 调整 young 区及 Survivor 大小、调整 gc 停顿时间参数、调整整个 JVM 分配内存大小；
- 更换 CMS、ParNewGC  回收器。经验证有一定提升，但不如调整 G1 参数提升大，且 g1 是 CMS、ParNewGC 的升级版，oracle 从 java 8 开始默认使用 g1 垃圾回收器，因此暂不更换垃圾回收器。

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-6-idea%E8%AE%BE%E7%BD%AE%E8%B0%83%E6%95%B4jvm%E5%9B%9B%E5%8C%BA%E5%86%85%E5%AD%98%E5%8F%8A%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%99%A8.png)

### Growth st 调参记录

| NewRatio(Old:Young) | SurvivorRatio | Mem  | MaxGCPauseMillis | GC Time (mm/1min) | GC Count |
| ------------------- | ------------- | ---- | ---------------- | ----------------- | -------- |
| 1                   | 8             | 2G   | 50               | 886               | 4        |
| 2                   | 8             | 2G   | 50               | 776/825           | 7        |
| 4                   | 8             | 2G   | 50               | 873               | 11       |
| 2                   | 8             | 2.5G | 50               | 1020              | 6        |
| 2                   | 4             | 2G   | 50               | 1300              | 7        |
| 2                   | 16            | 2G   | 50               | 763               | 6        |
| 2                   | Undefine      | 2G   | 50               | 1110              | 7        |
| 2                   | 12            | 2G   | 50               | 623               | 6        |

最终选择上表最后一行参数。

### Prod 发布结果

优化前：

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-%E7%BB%93%E6%9E%9C-%E4%BC%98%E5%8C%96%E5%89%8D-gc-count.png)

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-%E7%BB%93%E6%9E%9C-%E4%BC%98%E5%8C%96%E5%89%8D-gc-time.png)

优化后：

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-%E7%BB%93%E6%9E%9C-%E4%BC%98%E5%8C%96%E5%90%8E-gc-count.png)

![image.jpeg](https://github.com/zhangbotong/Interview/blob/main/assets/growth-jvm-optimize/growth-%E7%BB%93%E6%9E%9C-%E4%BC%98%E5%8C%96%E5%90%8E-gc-time.png)

|          | 优化前重启时峰值 GC Count（弹性机器不是这几台） | 优化前重启时峰值 GC Time（弹性机器不是这几台） | 优化后重启时峰值GC Count | 优化后重启时峰值GC Time | 优化前平稳GC Count | 优化前平稳 GC Time | 优化后平稳后 GC Count | 优化后平稳后 GC Time |
| -------- | ----------------------------------------------- | ---------------------------------------------- | ------------------------ | ----------------------- | ------------------ | ------------------ | --------------------- | -------------------- |
| gorwth07 | 37                                              | 1390                                           | 8                        | 842                     | 9                  | 223                | 2                     | 62                   |
| growth04 |                                                 | 864                                            | 6                        | 600                     | 6                  | 125                | 2                     | 38                   |
| growth03 |                                                 | 1680                                           | 6                        | 631                     | 1                  | 35                 | 1                     | 41                   |
| growth05 |                                                 | 1520                                           | 6                        | 347                     | 1                  | 25                 | 1                     | 51                   |
| avg      | 37                                              | 1363.5                                         | 6.5                      | 605                     | 4.25               | 102                | 1.5                   | 48                   |

### 参考广告服务启动 gc 情况

广告服务 GC Count = 3次/min， GC Time = 589ms/min。

可以推断出服务启动时 gc 是不可避免的，广告的 gc 时间也在 600ms 左右，基本与 growth 现状一致。

### 结果

服务启动时 gc count 均值从 37 次/min -> 6.5 次/min，减少 84%；
服务启动时 gc time 均值从 1363.5ms/min -> 605ms/min，减少 56%；
启动平稳后 gc count 均值从 4.25 次/min -> 1.5 次/min，减少 65%；
启动平稳后 gc time 均值从 102 ms/min -> 48ms/min，减少 53%。
待后续持续观察。

## Ref

- jvm 分析工具：https://jvm.sankuai.com/#/

- [210707 一次younggc告警排查经历](https://km.sankuai.com/collabpage/1341601443)

- gc.log

  60.33KB

- GC 日志分析工具：https://github.com/chewiebug/GCViewer

- https://radar.mws.sankuai.com/event/detail/2669806/content?from=all

- 日志：[http://logcenter.data.sankuai.com/search/com.sankuai.qcs.service.growth/query/%7B%22start_time%22:%222024%2F03%2F29%2011:31:00%22,%22end_time%22:%222024%2F03%2F29%2011:31:59%22,%22keyword%22:%22set-hldy-qcs-service-growth02%22,%22model%22:%22absolute%22,%22from%22:50,%22sortInfo%22:%5B%7B%22es_datetime%22:%22desc%22%7D%5D%7D](http://logcenter.data.sankuai.com/search/com.sankuai.qcs.service.growth/query/{"start_time":"2024%2F03%2F29 11:31:00","end_time":"2024%2F03%2F29 11:31:59","keyword":"set-hldy-qcs-service-growth02","model":"absolute","from":50,"sortInfo":[{"es_datetime":"desc"}]})

- Raptor 监控：https://raptor.mws.sankuai.com/application/hosts?reportType=hour&startDate=20240329110000&endDate=20240329115900&ip=10.122.140.123&group=&domain=com.sankuai.qcs.service.growth&date=&r=12641&urlThreshold=1000&serviceThreshold=100&callThreshold=100&sqlThreshold=100&cacheThreshold=25&mqThreshold=100&bladeSqlThreshold=500&fold=SQL&query=&type=SQL.qcsgrowth.qcs_growth