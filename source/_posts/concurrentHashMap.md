---
title: ConcurrentHashMap相关知识汇总
tags:
    - ConcurrentHashMap
---

## 相关博客
- [ConcurrentHashMap源码剖析](http://www.importnew.com/21781.html)
- [聊聊并发（四）  深入分析ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap/)
- [探索 ConcurrentHashMap 高并发性的实现机制](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/index.html)
 - [ReentrantLock与synchronized](http://uule.iteye.com/blog/1488356)

 <!--more-->

## 总结
- Java存储模型的Happens-Before规则：
    - 监视器锁法则：对一个监视器锁的解锁 happens-before于每一个后续对同一监视器锁的加锁。
    - volatile变量法则：对volatile域的写入操作happens-before于每一个后续对同一个域的读写操作。
- 编译器的重排序。
- 锁分段技术：分离锁，即需要锁时，仅在当前Segment
段锁，而并不是锁整个map。
- get、put方法的精妙处。
- ReentrantLock与Synchronized区别：简单理解，ReentrantLock是Synchronized的升级版，例如ReentrantLock有锁超时的功能，更容易避免死锁。这种类似于我使用Redis写的tryLock功能。这种锁机制是单机版的，在多机环境下不适用。


