---
title: 故障定位分析及解决
date: 2019-11-19 21:24
---

### 案例一

线上用户公钥服务，收到如下告警：
``` java
告警分派:
严重 健康拨测
2019-11-13 15:31:46: 任务名(secretkey-jingwei.huya.com_16033),域名(secretkey-jingwei.huya.com),路径(/), 健康拨测： ip：10.185.32.105, port:8080, 多次拨测失败(3次)

【此告警需要处理，请点击下方我的告警或登录告警平台进行操作】
```
收到告警的第一反应应该是当前是否有人在发版，近期是否是有代码变更。此处均没有。
到监控系统查询WEB质量情况，关注的点由前到后分为（故障发生前后时段）：请求总数、请求\响应时间、分机请求数、分机请求响应时间、URI请求分布、URI返回码分布、接入层WEB专区\Nginx统计数据等。
从这里看出本告警是由于该服务/downloadUserKey接口负载过高导致，如下图所示：
![](/img/trouble/downloadUserKey.png)
但该接口之前以做过优化，增加了JVM和DB间的Redis缓存，以及JVM缓存。初步怀疑存在缓存穿透问题。

<!--more-->

使用Arthas trace命令对该接口进行分析，分析结果如下：
在缓存未命中下的执行情况：
![](/img/trouble/trace_no_cache.png)
假设40ms处理一个，则1s可以处理25个，4台机，单机4核，即使跑满，也只能达到400qps，这还未算服务器自身损耗。
缓存命中时的执行情况：
![](/img/trouble/trace_cache.png)
跟相关研发同学沟通后，发现其缓存设计问题，其设置了缓存失效时间，但在缓存失效后，未主动拉新，导致上述问题发生。

### 案例二

混合云交付平台告警：
``` java
告警分派:
警告 web质量运营告警
告警级别:告警 
域名:autoidc-jingwei.huya.com 
Note:autoidc-jingwei.huya.com4XX请求数每分钟大于60 
最近3个值为:70.00 , 341.00 , 146.00 
Timestamp:2019-11-15 09:32:00 
https://skynet-jingwei.huya.com/web.html#/domain/autoidc-jingwei.huya.com/summary?type=webproxy

【此告警需要处理，请点击下方我的告警或登录告警平台进行操作】
```
根据之前的分析路数，查询到下列异常统计数据：
![](/img/trouble/autoidc_5XX.png)
![](/img/trouble/autoidc_4XX.png)
确认是WEB专区问题，跟相关运维同学沟通，确认在该时段，深圳机房WEB专区磁盘高载，导致其服务质量下降，影响到后端服务。

TODO 故障分析流程
代码变动->发布变动->web质量->硬件监控->JVM相关


