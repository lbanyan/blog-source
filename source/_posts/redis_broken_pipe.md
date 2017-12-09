---
title: Redis客户端Broken pipe
date: 2017-05-31 19:59
tags:
    - Redis
    - Broken pipe
---

### 异常场景
在使用Java Jedis连接池，Jedis客户端仅作消息传递作用时，消息很少的情况下，偶见一下异常：
``` java
JedisConnectionException: java.net.SocketException: Broken pipe
```

### 原因及解决办法
我所遇到的此种现象是由于Redis服务端设置了timeout参量，即当客户端在一段timeout长的时间内没有发出任何指令，那么关闭该连接。由于采用Jedis连接池，客户端和服务端建立的是长连接，在没有大量消息传递的情况下，连接池中部分Jedis客户端会在timeout时间段内对服务端未发出任何指令，如果在服务端和客户端关闭连接的情况下，恰巧客户端向服务端发出了指令，则会发生上述异常现象。
解决的办法是将Redis服务端的timeout参量设置为0，以关闭此功能。
