---
title: JDK中常用数据结构实现原理
---

## SET
[jdk HashSet, LinkedHashSet工作原理分析](https://fangjian0423.github.io/2016/03/30/jdk_hashset_linkedhashset/)
HashSet是通过HashMap实现的，value放了同一个静态不可变对象。
