---
title: Jar包冲突检查及处理
date: 2018-06-12 20:15
tags:
    - 包冲突
---

### 背景
项目中引入fasterxml Jar包后，出现如下图所示的Jar包冲突：
![](/img/jar_packet_conflict/packet_conflict.png)
通过项目引入的包：
![](/img/jar_packet_conflict/external_libraries.png)
在所有引入的fasterxml包中，只有jackson-annotations为2.8.0，其他均为2.8.8。修改了引入包jackson-annotations为2.8.8后，问题仍然没有解决。

<!--more-->

### 查找冲突的包
一种方法是，根据类名SerializationConfig查找该类，发现有多个包中存在该类，从而找到可能冲突的包。
另一种比较好的做法是，项目增加JVM启动参数-verbose:class，打印类加载信息，从中找到com.fasterxml.jackson.databind.SerializationConfig的相关包：
```
[Loaded com.fasterxml.jackson.databind.SerializationConfig from file:/D:/Maven/.m2/com/youzan/open-sdk-java/2.0.2/open-sdk-java-2.0.2.jar]
```
最终发现是open-sdk-java包将fasterxml 2.5版本打在了自己包内部，如下图所示：
![](/img/jar_packet_conflict/youzhan_package.png)

