---
title: Redis 半开连接问题
date: 2018-02-07 19:36
tags:
    - Redis
    - SUBSCRIBE
    - 半开连接
---

### 背景介绍

生产环境使用 Redis 的 PUBSUB 模式进行参数配置，我的服务起一个单独的线程订阅 Redis 的配置 Key，配置更改通过 Redis Publish 到我的各服务上来完成配置变更。

但运行过程中，经常出现配置未更新的情况，似乎我们的服务并未订阅到 Redis Publish 的信息。根据日志记录，也未看到消息订阅的行为。

生产环境：

- Linux Version: 3.10.0-514.16.1.el7.x86_64
- Redis Version: 3.0.2
- Java Version: 1.8.0_131
- Java Jedis Version: 2.9.0

### 问题分析

#### 查看该线程是否挂掉了

执行：

``` bash
// 查看我的服务的pid
ps aux | grep java | grep mapi
// 查看 pid = 3005 进程的线程信息
/usr/local/jdk1.8/bin/jstack 3005
```

查看到该线程信息：

![](/img/redis_half_open/stack_info.png)

可以看到该线程仍然正常运行，排除线程挂掉导致配置更新失败的问题。

#### 查看我的服务和 Redis 的连接情况

在配置更新失败的情况下，查看我的服务和 Redis 的连接是否正常。
执行：

``` bash
// 进程下使用的各端口连接，其中 3005 为 pid
netstat -nap | grep 3005
// 监听与 Redis 6379 端口连接的本地 35264 端口流量
tcpdump -vv  -i eth0 src port 35264
```

查看到该端口流量情况：

![](/img/redis_half_open/port_connect_info.png)

在关闭 Redis 服务器 和 多次 Redis Publish 的情况下，我的服务本地 35264 端口没有任何流量，状态依然为 ESTABLISHED。图中 35628 端口，为非订阅模式的连接，其在 Redis 服务器关闭后，其对应连接可以正常关闭。

#### 偶然发现

配置更新失败，大约 2 小时后，又可以使用 Redis Publish 更新成功了。

这边是典型的半开连接问题，具体可参考我的上一篇文章 [检测半开连接](http://blog.lbanyan.com/half_open_connections/)。

#### Redis 客户端 Keepalive

Jedis 默认是开启 Keepalive 的，该 Keepalive 工作在七层，由 Jedis 控制和实现，但在订阅模式下为什么没有生效呢？



