---
title: Java项目越运行越慢的问题分析
---

## 现象
我们的一个Java项目，刚启动时，运行一切正常，隔一段时间后，接口执行时间增长一倍，重启之后，又一切恢复正常。

## 分析
程序方面看不出有什么问题，于是我从JVM层面看看发生了什么。
可以参考以下两篇文章，我就不重复造轮子了：
[JVM性能调优监控工具jps、jstack、jmap、jhat、jstat、hprof使用详解](https://my.oschina.net/feichexia/blog/196575)
[java命令--jmap命令使用](http://www.cnblogs.com/kongzhongqijing/articles/3621163.html)
在我执行jps命令时，报错： **process information unavailable**，这个是因为查看的进程非当前用户下的进程，可以使用以下命令来执行：
``` bash
sudo -u www-data jps 42264
```
其中42264为我的java进程号，www-data为我的java启动的用户。
其他的小工具也是需要用同样的方式执行的。
使用以下命令获取当前堆中各对象数量及大小信息：
``` bash
sudo -u www-data jmap -histo 42264 > map.txt
```
我在项目刚启动时和第二天各打印了一次堆信息，经过对比，发现有一个ConcurrentHashMap对象的数量翻倍了，但是项目中有很多ConcurrentHashMap，通过jmap，不清楚是哪个类中的哪个ConcurrentHashMap属性的问题。
于是，dump了堆信息：
``` bash
sudo -u www-data jmap -dump:file=dump.txt 42264
```
并下载到我本地，在JDK中找到jviusalvm.exe工具，并双击打开，注意这个工具不能直接通过命令行执行jviusalvm来调用哦。
将dump.txt导入jviusalvm中，找到了这个ConcurrentHashMap。
原来是项目中维持了一个用ConcurrentHashMap实现的缓存，正常情况下每隔20s清洗一次，会将失效的缓存清理掉，但是有一部分缓存因为之前在其他地方增加了一个条件，导致在删除此类缓存时，判断此类缓存未失效，于是出现了堆积，这就导致了接口执行时间越来越长的问题。

