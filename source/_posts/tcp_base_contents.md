---
title: TCP相关基础知识
tags:
    - TCP
---

> TCP三次握手和四次挥手详细过程，TIME-WAIT和CLOSE_WAIT的原因。引申到服务端大量连接未释放的原因。

<!--more-->

[TCP协议中的三次握手和四次挥手(图解)](http://blog.csdn.net/whuslei/article/details/6667471/)
补充一点：TCP建立连接时，为什么还需要第三次握手，这是因为在第一次握手时，服务端接收到的客户端SYN有可能在客户端看来已经是超时的了，此时第二次握手，客户端回应时就可以拒绝掉，从而中止连接。
[CLOSE_WAIT状态和TIME_WAIT状态 (TCP连接)](https://my.oschina.net/u/593721/blog/186983)
总结一下：出现大量CLOSE_WAIT的连接，应考虑连接是否建立的过多，是否应采用连接池优化，使用有限个长连接来处理，而非每次请求都建立一个短连接来处理，服务端的性能是否有问题，为什么长时间不进入LAST_ACK状态。
检测端口的连接情况的命令：
``` bash
netstat -na | grep 8081
```
使用Wireshark观察TCP连接：
![](/img/wireshark_tcp.png)
