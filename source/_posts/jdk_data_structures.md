---
title: JDK中常用数据结构实现原理
---

## SET
[jdk HashSet, LinkedHashSet工作原理分析](https://fangjian0423.github.io/2016/03/30/jdk_hashset_linkedhashset/)
HashSet是通过HashMap实现的，value放了同一个静态不可变对象。

## MAP
[Java8 HashMap详解](http://www.jianshu.com/p/30bffabb2e5c)
HashMap使用链地址法。
[Java Core系列之TreeMap实现详解](https://yq.aliyun.com/articles/46901#)
TreeMap使用红黑树结构，有序，非线程安全。