---
title: MySQL相关基础知识
tags:
    - MySQL
---

> MySQL引擎有哪些，各引擎的不同之处，事务级别，ACID，死锁，索引，BTREE。

<!--more-->

MySQL引擎主要有**MyISAM与InnoDB**两种，当然还有其他的。目前我们业务使用的引擎都是InnoDB，此两类引擎的对比可以参看这两篇博客： [MySQL存储引擎InnoDB与Myisam的六大区别](https://my.oschina.net/junn/blog/183341)和 [MyISAM InnoDB 区别](http://www.cnblogs.com/youxin/p/3359132.html) 。总结一下就是**InnoDB支撑事务、外键和行级锁**，删改较多的业务首选InnoDB；**MyISAM插入查询速度快、支撑全文检索、体积小、维护方便**，很适合监控平台记录监控使用。



