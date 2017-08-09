---
title: Linux IO模型
---

## 参考文章
- [聊聊Linux 五种IO模型](http://www.jianshu.com/p/486b0965c296)
同步模型（同步阻塞IO、同步非阻塞IO、IO多路复用、信号驱动IO）、异步模型（异步非阻塞IO）
- [聊聊IO多路复用之select、poll、epoll详解](http://www.jianshu.com/p/dfd940e7fca2)
epoll有回调、fd数不受限制、fd只用拷贝一次
- [Linux文件描述符](http://blog.csdn.net/cywosp/article/details/38965239)
包含文件指针
- [epool边缘触发和水平触发](http://www.cnblogs.com/yuuyuu/p/5103744.html)
读写缓冲区太小，fd上有可读内容，水平触发每次都触发，边缘触发只触发一次。




