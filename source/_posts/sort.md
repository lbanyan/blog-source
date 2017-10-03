---
title: 高效排序解决方案
tags:
    - Redis
    - 有序集合
    - ConcurrentSkipListMap
---

## 背景
推荐系统中，为用户推荐主播时，有一步需要通过主播当前直播观看人数（pcu）从高到底来筛选。简单的做法是，每次用户请求，使用TreeMap对当前的所有在线主播（通常一到两万个）进行一次基于pcu的排序即可，但是这样会造成一定的性能影响。

<!--more-->

## 解决办法

### 方案一：Redis.SortedSet
在线主播集是在另一个线程中不断更新的，可以对其根据pcu排序也在此更新中完成。具体的做法是借助Redis的有序集合SortedSet来完成。
当有在线主播更新时，使用：
``` bash
ZADD key score member
```
以pcu为score，该主播id为member来更新该key有序集合；
再通过：
``` bash
ZREVRANGE key start stop
```
start为0，stop为999，便可以取到pcu排名前1000的在线主播，然后进行相应的筛选动作。
还有一个问题是必须考虑的，我们既然维持了一个这样的缓存，就必须要考虑到缓存失效，在线主播是会下线的，下线后应当从该有序集合中移除，否则会出现异常。使用：
``` bash
ZREM key member
```
来移除失效的主播。
ZADD和ZREM的时间复杂度都是O(n)，ZREVRANGE在固定去前1000个时，也只用遍历前1000个主播即可得到。
该方案的特点是利用了Redis的有序集合来完成，简单高效。

### 方案二：ConcurrentSkipListMap
使用ConcurrentSkipListMap来取代Redis.SortedSet做缓存，主要优化点是避免使用Redis带来的网络开销。极严苛的性能要求下使用该优化方案。
之所以使用ConcurrentSkipListMap，是出于线程安全的考虑，如果没有这一方面顾虑，可以采用其他有序Map。
简单的测试样例如下：
``` java
ConcurrentSkipListMap<String, Long> map = new ConcurrentSkipListMap<>();
map.put("a", 1l);
map.put("c", 2l);
map.put("b", 3l);
for(String entry : map.descendingKeySet()) {
    System.out.println(entry);
}
System.out.println(map);
```
注意ConcurrentSkipListMap是根据key的Comparator来比较的，而非根据value来比较，具体可以参考 [并发容器Map之二：ConcurrentSkipListMap](http://www.cnblogs.com/duanxz/archive/2012/08/27/2658004.html)
根据上述理解，需构造implements Comparable的key SortableObject：
``` java
public static class SortableObject implements Comparable{
    
    /**
    * 主播ID
    */
    public String id;
    /**
     * 在线观看该主播直播的用户数（score是类似 redis sorted set 的 score 概念）
     */
    public Long score;

    public SortableObject(String id, Long score) {
        this.id = id;
        this.score = score;
    }

    @Override
    public int compareTo(Object obj) {
        if (obj == null || !(obj instanceof SortableObject)) {
            return 1;
        }
        SortableObject o2 = (SortableObject) obj;
        int d = o2.score.compareTo(score);
        if (d == 0) {
            d = id.compareTo(o2.id);
        }
        return d;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        }
        if (obj instanceof SortableObject) {
            SortableObject o2 = (SortableObject) obj;
            boolean eq = score.equals(o2.score);
            if (eq) {
                eq = id.equals(o2.id);
            }
            return eq;
        }
        return false;
    }

    @Override
    public String toString() {
        return "[" + id + ": " + score + "]";
    }
}
```
使用ConcurrentSkipListMap<SortableObject, Long> sortedPcuAnchors = new ConcurrentSkipListMap<>();便可完成该方案的功能需求。
该方案完全利用了JVM的缓存，节省了使用Redis的网络开销。

**小技巧**：使用新对象覆盖ConcurrentMap的老对象时，并不影响正在遍历该老对象的操作。

**小知识**：map和hash没有什么直接关系，map只是key value的数据对，至于十一hash形式存在还是以跳表形式存在，是有具体的如HashMap、SkipListMap实现的。


