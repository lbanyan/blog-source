---
title: 基于Redis实现的分布式锁
---

本文节选自我的《自主实现SDN虚拟网络与企业私有云》一书。

平台支持多机部署，不可避免的需要考虑到分布式问题。在多机部署模式下对竞争资源的访问控制，可以借助第三方工具如Redis来实现。

Redis简单、灵活、高效，常常被用于实现分布式锁服务，目前平台的许多功能也使用了这种方式。

## Redis相关命令

在讲解之前，先对将要使用的Redis命令进行简单的介绍。

SET命令用于设置给定 key 的值，如表4-9所示。从Redis 2.6.12版本开始，SET命令在设置操作成功完成时才返回OK。
``` redis
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```
表4-9  Redis SET命令参数描述

|  命令参数 |  描述 |
| ------------ | ------------ |
|  EX second | 设置键的过期时间，单位为秒  |
| PX millisecond  | 设置键的过期时间，单位为毫秒  |
|  NX |  只在键不存在时，才对键进行设置操作 |
| XX  |  只在键已经存在时，才对键进行设置操作 |

SETNX命令用于在key不存在时设置key的值。
``` redis
SETNX key value
```
DEL命令用于删除指定的key。
``` redis
DEL key
```
EXPIRE命令为指定key设置生命周期。
``` redis
EXPIRE key seconds
```

## 分布式锁实现

起初，业务层使用SETNX和EXPIRE命令来实现分布式锁。在Redis中不存在指定key时，SETNX可以赋值成功，并返回TRUE；在Redis中存在指定key时，SETNX赋值失败，并返回FALSE；同时SETNX是原子性操作，利用这些特性可以实现分布式锁。

为什么还要使用EXPIRE命令设置过期时间呢？如果加锁后，程序异常终止，key将有可能永远无法释放，导致部分功能永远无法执行。加锁程序清单如下：
``` java
public boolean tryLock(String key, int seconds) {
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        boolean locked = jedis.setnx(key, "lock") > 0;
        if (locked) {
            jedis.expire(key, seconds);
        }
        return locked;
    } finally {
        if (jedis != null) {
            jedis.close();
        }
    }
}
```

但是这种实现仍然存在问题，因为tryLock对Redis的操作并非原子性，而是分为两步：SETNX和EXPIRE，在某些情况下，如断电时，将可能出现SETNX命令返回OK后，无法执行EXPIRE命令的问题，此时key锁将永远无法释放。但是在生产环境中出现此问题的概率极低，因而可以根据业务实际需要考虑是否允许出现此情况。

在使用Redis 2.6.12以上版本的情况下，可以使用SET命令来解决上述问题。即将SETNX和EXPIRE命令合并为一个命令：
``` java
jedis.set(key, System.currentTimeMillis() + "", "NX", "EX", seconds);
```
这样就保证了tryLock操作的原子性。其中"lock"为key的值，一般情况下是用不到的，可以写任意字符串。

解锁操作有两种：一种是等待锁key的过期；另一种是显式地删除此key。可以根据具体业务来决定采用哪种方式。

使用分布式锁的标准格式如下：
``` java
boolean r = jedisPoolManager.tryLock(key, seconds); // 加锁
if (!r) {
    return; // 说明已有线程在执行
}
Try {
    // 业务处理
} catch (Exception e) {
    // 异常处理
} finally {
    jedisPoolManager.del(key); // 释放锁
}
```

## 业务场景的使用

通常，在定时任务中，使用分布式锁的情况较多。

场景一：每日定时进行容量统计并记录入库的场景。此场景任务量极小，系统开销极小，而且每日只需要执行一次即可。在不使用分布式锁的情况下，多机会同时进行容量统计，并记入数据库，既浪费了系统资源，又可能在数据库中记入了重复数据，从而产生脏数据。在此场景下，可以使用固定的key作为锁，从而保证只允许有一个线程执行容量统计任务。这也是分布式锁的最简单应用场景。

场景二：多服务器获取到同一批任务的场景。MySQL的每日自动备份便属于此场景。多机获取到同样的一批日备任务，为避免重复备份，就需要使用到分布式锁。此时可以使用日备任务的业务ID作为key来加锁，从而保证每个日备任务只被执行一次。

## 优缺点

此方法的优点是使用灵活、简单易懂、易于维护；缺点是所使用的Redis是单点的，单点故障将导致系统不可用。由于平台使用Redis并未做高负荷的工作，Redis出现故障的可能性极低。同时，业务层使用线程池，即使出现故障，Redis恢复后系统便可正常恢复。对于高负荷使用Redis的场景，不建议使用上述方式。