---
title: Redis使用技巧
date: 2019-9-3 20:15
tags:
    - Redis
---

### 借助Redis实现熔断
以下为错误的示例：
``` java
/**
 * 断电时，偶现key的过期时间未设置，长期驻留
 * @param key
 * @param increment
 * @param seconds
 * @return
 */
@Deprecated
public Long incrby(final String key, final long increment, final int seconds) {
	long value = incr(key);
	if (seconds <= 0) {
		return value;
	}
	// 仅在第一次设置过期时间
	if (value == increment) {
		redisClient.getKeyOps().expire(key, seconds);
	}
	return value;
}
```
这里可以借助lua来实现incr和expire的原子性操作，代码示例如下：
``` java
private final static String INCRBY_EVAL =
        "local current;\n" +
        "current = redis.call(\"incrby\",KEYS[1],ARGV[1]);\n" +
        "if tostring(current) == ARGV[1] then\n" +
        "    redis.call(\"expire\",KEYS[1],ARGV[2])\n" +
        "end;" +
        "return current;";

public Long evalIncrby(final String key, final long increment, final int seconds) {

    return redisClient.execute(new RedisCallback<Long>() {
        @Override
        public Long doInRedis(Jedis jedis) {
            List<String> keys = Lists.newArrayList(key);
            List<String> args = Lists.newArrayList(increment + "", seconds + "");
            return (Long) jedis.eval(INCRBY_EVAL, keys, args);
        }
    });
}
```

### Redis热点key或流量打爆的问题解决
可以交由上层服务直接内存缓存，对外提供服务，因为上层服务一般都支持横向扩展。
Redis流量打爆，是在对安全及监控组提供IP查询接口时遇到的，有6000个节点每分钟请求一次IP查询全量接口，共10000个IP，每分钟流量大概为7.7Gb，此处还使用了增量同步的改进技术，毕竟即使上层服务使用内存缓存，在服务器有限的情况下，依然存在上层服务网卡打爆的情况。
