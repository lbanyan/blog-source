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

但运行过程中，经常出现配置未更新的情况，似乎我的服务并未订阅到 Redis Publish 的信息。根据日志记录，也未看到消息订阅的处理行为。

生产环境：

- Linux Version: 3.10.0-514.16.1.el7.x86_64
- Redis Version: 3.0.2
- Java Version: 1.8.0_131
- Java Jedis Version: 2.9.0

<!--more-->

### 问题分析

#### 查看该线程是否挂掉了

执行：

``` bash
// 查看我的服务的 pid
ps aux | grep java | grep mapi
// 查到我的服务 pid = 3005，查看该进程的线程信息
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

#### Jedis Keepalive

Jedis 默认是开启 Keepalive 的，该 Keepalive 并非 Jedis 应用层实现的，而是依赖于 TCP 层协议，Linux 默认的 TCP Keepalive 时间为 2 小时。

#### Jedis Socket soTimeout

Jedis 有对 Socket 读超时设置 soTimeout，在配置时，我使用 2000ms 设置了该值，理论上，订阅者读超时，自动断开连接，我会使用新的 Jedis 重新订阅，但是 Jedis 并没有这样执行。

查看 Jedis 源码：

![](/img/redis_half_open/jedis_subscribe.png)

发现订阅模式将 Socket.soTimeout 设置为无效了，也即永不超时。

在 Jedis GitHub Issues [Specify connection timeout for blocking calls #426](https://github.com/xetorthio/jedis/issues/426) 有相关的讨论。当然在 Issues 中还能找到更多关于此类问题的讨论。

#### 其他

这说明我之前写过的一篇文章 [偶尔出现Redis客户端获取不到Redis数据的情况](http://blog.lbanyan.com/redis_blpop_null/) 中的分析可能是错误的，当时应该是由于链路拉的过长，增大了出现半开连接的概率，出现了该问题，而非数据在连接中丢失，但已没有当时的生产环境，故无法考证了。

另外，当使用 Jedis.subscribe 异常时，应关闭该 Jedis 连接，而非复用，相关的原因可以参考 [Maybe serious bug in JedisCluster.subscribe #1376](https://github.com/xetorthio/jedis/issues/1376) 以及正确的处理方式参考 [when jedis use subscribe exception can not be reconnected #1026](https://github.com/xetorthio/jedis/issues/1026)。

### 解决方案

#### Jedis 层修改

您可以 fork 一份 Jedis 源码，根据相关 Issues 进行修改，以使 Jedis.subscribe 支持 Socket 读超时，来解决当前问题。

#### 用户层修改

您还可以根据从 JedisPool 中获取的 Jedis，定期执行任意非阻塞命令（推荐使用 ping，对业务没有任何影响），当出现异常时，将所有阻塞读（如：subscribe、blpop）的 Jedis 连接全部关闭，并从 JedisPool 获取新的连接，重新阻塞读（例如执行 subscribe 或 blpop），来解决半开连接问题。此处的非阻塞命令、阻塞命令叫法并不严格，您只需要简单理解即可。

```java
public void subscribe() {
    
    Thread thread = new Thread(() -> {
        while (true) {
            try {
                final Jedis jedis1 = redisClient.getJedisPool().getResource();
                final Jedis jedis2 = redisClient.getJedisPool().getResource();
                Thread subscribeThread = subscribe(jedis2);
                while (true) {
                    try {
                        jedis1.ping();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                        try {
                            redisClient.close(jedis1);
                            redisClient.close(jedis2);
                            subscribeThread.interrupt();
                        } catch (Exception ex) {
                            logger.error(e.getMessage(), ex);
                        }
                        break;
                    }
                    ThreadUtil.sleep(1000);
                }
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
            ThreadUtil.sleep(1000);
        }
    });
    thread.setDaemon(true);
    thread.setName("appcmdb-thread-appconfig-ping");
    thread.start();
}

/**
 * 启动一个守护线程，监听配置的变化，利用redis的pubsub模式实现java中的分布式内存
 *
 * @return
 */
public Thread subscribe(Jedis jedis) {

    Thread thread = new Thread(() -> jedis.subscribe(new JedisPubSub() {
        @Override
        public void onMessage(String channel, String message) {
            // message是{"key":"key",""value":"value"}的json数据
            // 后台更新一个值后就要publish下
            try {
                Conf conf = JsonUtil.readValue(message, Conf.class);
                String key = conf.getPkey();
                String value = conf.getPvalue();
                if (EmptyUtil.isEmpty(value)) {
                    APP_CONFIG_CACHE.remove(key);
                } else {
                    APP_CONFIG_CACHE.put(key, value);
                }
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    }, RedisKeys.CHANNEL_APP_CONFIG));
    thread.setDaemon(true);
    thread.setName("appcmdb-thread-appconfig-subscribe");
    thread.start();
    return thread;
}
```

上述的定期检测时间不可过长，否则可能仍会出现阻塞读的半开连接问题。

有关 JedisPool 保活策略可参考 [Jedis源码分析](https://www.jianshu.com/p/dcf1491afbe7)。

#### 深入解析
TCP连接惰性特性，会在中间设备恢复正常后，TCP连接重新恢复正常，但上述情况，中间设备正常后，客户端依然接收不到消息。
以blpop为例，在中间设备断开后，服务端将信息推给客户端时，发现异常，服务端会认为该客户端已断开连接，从而将该客户端的ip:port从连接记录中删掉，之后即使中间设备恢复正常了，服务端也不会推消息到该客户端了。
