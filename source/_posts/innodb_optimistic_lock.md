---
title: MySQL InnoDB 不使用乐观锁的危害
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

执行如下两个事务：

![](/img/innodb_optimistic_lock.png)

事务 B 先于事务 A 执行，id = 1 的记录状态已经发生了改变，但事务 A 仍然成功执行，并修改了其他表数据。如果这并非您业务想看到的结果的话，应该使用乐观锁，这样在事务 B 执行完成后，事务 A 中对 test 表最后的修改必然失败，整个事务 A 便处理失败，不会出现您预想不到的混乱现象。
