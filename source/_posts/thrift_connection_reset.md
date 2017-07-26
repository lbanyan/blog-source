---
title: Thrift服务端Connection Reset by Peer和Broken pipe
tags:
  - Thrift
  - Broken pipe
  - Connection Reset by Peer
---

## 现象
虎牙主播推荐功能中，用于获取推荐列表的客户端出现大量的请求服务端超时现象，但服务端日志无ERROR的异常。

## 原因
经排查，一是网络正常，二是客户端只是请求了服务端，并没有做其他耗时操作，三是客户端发送请求的数量大于服务端处理请求的数量（此处客户端有客户端发送请求的数量统计，服务端在每次处理完请求后会异步记录入数据库）。那么问题应该出在客户端和服务端的连接这里。
最终找到了原因：Thrift的“Connection reset by peer”和“Broken pipe”异常等级为WARN级，由于平台的QPS很高，在服务端我们的Log使用的是ERROR级，因而并未打印出这类WARN异常。
![](/img/thrift_connection_reset.png)
关于此种异常可以参看此文：
[从tcp原理角度理解Broken pipe和Connection Reset by Peer的区别](http://lovestblog.cn/blog/2014/05/20/tcp-broken-pipe/)
出现“Connection reset by peer”是由于客户端发送请求后，服务端处理超时，客户端主动断开连接，服务端处理完成后回应客户端时，便会抛此异常。
既然出现“Connection Reset by Peer”，为什么还会出现“Broken pipe”呢？那就说明服务端接收到此客户端的任务有堆积，在客户端已断开连接后，服务端将任务结果一个一个的返回给客户端时，就会出现“Broken pipe”。
![](/img/thrift_timeout.png)
故而根本问题应该出在服务端性能上。当QPS过大时，服务端无法及时处理完所有任务，导致部分任务堆积，处理超时的情况。在QPS较小的情况下，不会出现此问题。Thrift将此类异常设置未WARN级别异常，应该是觉得此类异常对使用者来说并非特别重要，使用者不应该去关心它。