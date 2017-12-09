---
title: Multimaps.synchronizedMultimap经常抛出ConcurrentModificationException异常
date: 2017-09-17 19:22
tags:
    - Multimaps
    - synchronizedMultimap
    - ConcurrentModificationException
---

### 背景
使用Multimaps.synchronizedMultimap可以得到线程安全的multimap，但是在高并发下频繁修改此multimap不适用，因为其会经常抛出ConcurrentModificationException的异常，导致业务经常出错，这是无法容忍的。

### 解决办法
目前，我使用**ConcurrentHashMap**来代替Multimaps.synchronizedMultimap，结构上的不同，通过业务来弥补。ConcurrentHashMap不会出现ConcurrentModificationException这种异常的原因是：
> 在这种迭代方式中，当iterator被创建后集合再发生改变就不再是抛出ConcurrentModificationException，取而代之的是在改变时new新的数据从而不影响原有的数据，iterator完成后再将头指针替换为新的数据，这样iterator线程可以使用原来老的数据，而写线程也可以并发的完成改变。

具体可参考此文[ConcurrentModificationException异常解决办法](http://blog.sina.com.cn/s/blog_56d8ea900101h87e.html)
