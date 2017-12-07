---
title: 多后端时，性能下降的问题分析
---

### 背景
服务架构图请参见 [Netflix负载均衡器Ribbon的使用](https://lbanyan.github.io/2017/07/26/netflix_loadbalancer/)。增加Ranking服务器后，发现其平均响应时间从40ms增加到了70ms左右。增加的服务器为同机型。怀疑是使用的异步HTTP请求和负载均衡所致。

<!--more-->

### 分析
之前也误以为是新建立HTTP连接耗时导致，但是长时间平台稳定后，耗时并没有明显减少。统计发现，发送异步HTTP请求以及负载均衡并未耗时，分析异步HTTP配置，也并未发现有配置方面的问题。
通过统计每个HV上Ranking服务的耗时，发现部分HV偶尔会出现80-130ms的耗时，而另一批HV都在50ms以内处理完了所有任务。下线掉这些异常的HV后，发现平台耗时恢复正常。
那这些异常的HV和正常的HV有什么不同呢，他们可都是同样的机型哦。对比发现，同样3.4GHz的CPU，异常的核主频只有800MHz左右，排查发现这是服务器设置了省电模式导致的。线上的服务器出现这样的设置，就是运维的锅了(lll￢ω￢)。
查看CPU运行模式：
``` bash
sudo cpupower frequency-info
```
修改CPU为高性能模式：
``` bash
sudo cpupower frequency-set -g performance
```
