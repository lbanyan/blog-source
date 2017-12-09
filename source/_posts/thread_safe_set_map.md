---
title: 线程安全的Set和Map
date: 2017-08-15 19:36
tags:
    - 线程安全
    - Set
    - Map
---

### 线程安全的SET
JDK中并没有直接提供线程安全的Set，可以通过使用ConcurrentHashMap来构造，当然最简单的方式是使用Google Guava提供Sets.newConcurrentHashSet()方法来构造。

### 线程安全的Map&lt;String, Set&lt;String&gt;&gt;
使用ConcurrentHashMap&lt;String, Set&lt;String&gt;&gt;，在其value为非线程安全的Set情况下，Map整体非线程安全的。
例如：A和B线程同时修改此Map相同Key的value，A和B同时得到此value并进行修改，此修改必然会出现线程非安全的隐患。

还有一个问题，由于不存在线程安全的Set类，故而无法直接在Map中声明其value为线程安全的Set。如何来实现线程安全的修改此Map操作呢？可以参考以下代码：
``` java
Set<String> set = Sets.newConcurrentHashSet("**");
Set<String> val = map.putIfAbsent(key, set);
if (val != null) {
    val.addAll(set);
}
```
代码中map为Map&lt;String, Set&lt;String&gt;&gt;类型，假设A线程是往键key的值set中加入“**”，B线程是往键key的值set中加入“tt”，刚开始map中不存在键key，A线程先执行到map.putIfAbsent(key, set)，则A线程就可以直接put成功，并返回值为null的val，注意map.putIfAbsent是线程安全的，不存在B线程put覆盖掉A线程的put结果。接着B线程再执行到map.putIfAbsent(key, set)，由于map中已经存在了键key，故此操作返回为此key对应的value值，其已经是一个线程安全的Set了。由于val是线程安全的Set，考虑多个线程同时执行到val.addAll(set)，也不会出现线程非安全的问题。
