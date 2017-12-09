---
title: Java：获取一个对象的大小
date: 2017-09-21 22:36
---

### 获取对象大小的用处
- 用于项目上线前规划机器配置
- 用于项目中缓存的规划
- 用于检查项目内存泄漏的原因

<!--more-->

获取一个对象的大小方法很多，这里介绍两个简单的方法。
### 使用jmap获取
可参考我的博客[Java项目越运行越慢的问题分析](https://lbanyan.github.io/2017/07/27/java_slower_and_slower/)，这个也是一个内存泄漏的例子。

``` bash
jmap -histo pid
```
可以得到每个类的对象数，以及大小，从而很容易得到每个对象的大小。

### 使用com.javamex.classmexer.MemoryUtil工具
使用方法很简单，直接看其官网[Classmexer agent](http://www.javamex.com/classmexer/)。
如果使用IDE直接运行，可能会出现<font color='red'>Agent not initted</font>异常，需要设置VM options -javaagent参数为自己classmexer.jar的路径，例如我的设置：
``` bash
-javaagent:D:/UserData/Desktop/classmexer-0_03/classmexer.jar
```
更多内容可以参看这里 [查看 Java 对象大小](http://github.thinkingbar.com/lookup-objsize/)

引申内容： [内存泄漏和内存溢出有啥区别？](https://www.zhihu.com/question/40560123)
内存泄漏是说程序中有应该回收的对象而未被回收，内存溢出是说当前分配的内存不够用了。内存泄漏严重可能会导致内存溢出。
