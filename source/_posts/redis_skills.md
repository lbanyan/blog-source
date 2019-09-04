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
