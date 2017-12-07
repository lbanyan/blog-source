---
title: Python RabbitMQ 客户端和服务器端断开连接后，客户端不抛错的问题
tags:
    - RabbitMQ客户端
    - TCP Keepalive
    - TCP半打开问题
---

### 现象
线上服务，数千个agent订阅RabbitMQ，获取消息。agent使用Python开发。运行一段时间后，部分agent由于网络原因，实际已经断开和RabbitMQ的连接，但是未见报错，导致对这些agent发送的指令全部失败。其中发生的时间，发生的频率，出现此现象的agent都比较随机。

<!--more-->

### 分析
问题的关键并不在于客户端为什么会和服务端断开连接，因为网络并非完全可靠。问题的关键在于客户端和服务端既然断开连接了，客户端为什么没有意识到从而抛异常，这一点在我们的Java RabbitMQ客户端是没有遇到过的。
使用Wireshark本地测试Python的RabbitMQ客户端和服务端的连接情况，发现在建立连接后，客户端并未发送Keepalive包来保持连接的正常。于是怀疑Python的RabbitMQ客户端并未启用此功能，经查证确实是如此。客户端启动Keepalive后，以上现象得以解决。
再往深处研究，找到了一篇相关的博文：[使用 rabbitmq 中 heartbeat 功能可能会遇到的问题](https://my.oschina.net/moooofly/blog/209823)，发现我之前的理解有部分是错误的，但也歪打正着，因为查找问题的方向是正确的，此问题确实应该往TCP Keepalive方向查。
我所发现的未发送Keepalive包，确实是Python的RabbitMQ客户端并未启用此功能，但即使开启了此功能，也需要**2个小时**的时间才能触发，这一点我当时没有意识到。
以上问题实际就是<font color='red'>TCP的半打开问题</font>。详情及相关解决办法可以参考上述的博文，在此不做赘述。


