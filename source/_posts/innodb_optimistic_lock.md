---
title: MySQL InnoDB 不使用乐观锁的危害
date: 2017-12-01 19:38
tags:
    - MySQL
    - InnoDB
    - 乐观锁
---

MySQL InnoDB 默认为 Repeatable read 隔离级别，业务开发中如果不使用乐观锁，会造成业务处理混乱的问题，具体可以看下述例子：

``` sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `status` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

该表有两条记录:

| id        | status   |
| --------  | -----:   |
| 1         | active   |
| 2         |   down   |

先关闭自动提交：

```sql
show variables like 'autocommit';
set autocommit = 0;
```
执行如下两个事务：

![](/img/innodb_optimistic_lock.png)

事务 A 在 SELECT 操作时，只会获取共享锁，而非排它锁，因而事务 B 可以先于事务 A 成功提交，导致 id = 1 的记录状态发生了改变，但事务 A 仍然可以成功执行，并修改了其他表数据。如果这并非您业务想看到的结果的话，应该使用 Hibernate @Version 乐观锁，这样在事务 B 执行完成后，事务 A 中对 test 表最后的修改必然失败，整个事务 A 便处理失败，不会出现您预想不到的混乱现象。

看看有意思的，事务 A 在执行第 4 行 SELECT 语句时，看到的 id = 1 的 status 仍然是 active，这是受事务的隔离级别 Repeatable read 决定的。第 5 行如果增加 status = 'active' 的 WHERE 条件，则此 UPDATE 操作便不会修改 status 为 finish。这一点很有用，我常利用此来实现相关的锁任务操作。

MySQL 底层使用了乐观锁，为什么业务层这里还要使用乐观锁？
乐观锁只是处理并发问题的一种方法，MySQL 底层使用乐观锁是解决它底层的并发问题，业务层使用乐观锁，是解决业务层并发问题，并不冲突。
