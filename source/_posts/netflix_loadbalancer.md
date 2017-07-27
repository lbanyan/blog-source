---
title: Netflix负载均衡器Ribbon的使用
tags:
  - Ribbon
  - 负载均衡
---

Ribbon原理及代码解析可参考： [Spring Cloud源码分析（二）Ribbon](http://blog.didispace.com/springcloud-sourcecode-ribbon/) ，本文仅简单介绍我们在生产环境中如何使用DynamicServerListLoadBalancer。

## 业务背景
如下图所示：
![](/img/matching_ranking.png)
Matching服务请求各宿主机上的Ranking服务来完成任务，Ranking服务提供的是HTTP的接口，Matching和Ranking在同一个机房，Matching可以通过http://ip:port/ranking 的方式访问到各Ranking服务。具体Matching和Ranking是做什么的，可以不需要知道。
为了实现Matching到Ranking的合理调度，我们使用了Ribbon。

<!--more-->

## 具体实现
### 负载均衡器的主体RankingLB类
``` java
public class RankingLB {

    DynamicServerListLoadBalancer<Server> lb;

    public RankingLB() {

        DefaultClientConfigImpl config = new DefaultClientConfigImpl();
        config.setClientName("ranking");
        config.setProperty(IClientConfigKey.Keys.ServerListRefreshInterval, 2 * 60 * 1000); // 2分钟更新一次Ranking服务列表
        config.setProperty(IClientConfigKey.Keys.NFLoadBalancerPingInterval, 15); // 注意单位s，用于ping检测各Ranking服务是否可用
        config.setProperty(IClientConfigKey.Keys.NFLoadBalancerPingClassName, RankingPing.class.getName()); // 实现我们自己的Ping检测服务
        config.setProperty(IClientConfigKey.Keys.NIWSServerListClassName, RankingServerList.class.getName()); // 实现我们自己的Ranking服务列表
        config.setProperty(IClientConfigKey.Keys.NFLoadBalancerRuleClassName,RankingRobinRule.class.getName()); // 实现我们自己的负载均衡算法
        config.setProperty(IClientConfigKey.Keys.ServerListUpdaterClassName, PollingServerListUpdater.class.getName());
        lb = new DynamicServerListLoadBalancer<Server>(config);
    }

    public Server chooseServer() {

        // 如果有特殊情况，可在此控制，如测试环境返回某个固定的Ranking服务，而不走LB挑选。
        return lb.chooseServer();
    }

    public DynamicServerListLoadBalancer<Server> getLb() {

        return lb;
    }
}
```
注意：IClientConfigKey.Keys.ServerListRefreshInterval更新时间，单位ms，经测试发现，设置过小，例如2ms的情况下，LB动态更新服务列表数次后就不在更新了，具体原因未深究。

### Ping检测服务RankingPing
``` java
public class RankingPing implements IPing {
	
	@Override
	public boolean isAlive(Server server) {
        // 预发布环境执行以下四步
        // 1. 采用重试机制请求此server服务，主要是防止网络波动导致误判。
        // 2. server服务异常，则告警（发短信），且将其记入服务异常集合中，这里我们使用了Redis作为此集合。
        // 3. server服务正常，如果服务异常集合中包含此server，则从集合剔除。
        // 4. 如果服务异常集合中包含此server，说明此server异常，返回false，否则，返回true
        // 正式环境仅执行上述的第4步
	}
}
```
预发布环境和正式环境在同一机房中，预发布环境不对外提供服务且只有一个Matching服务。此处如此设计，是一种优化手段，正式环境的Matching服务有多台，没必要每台都进行对Ranking server服务的直接检测，仅根据服务异常集合就可以判断此server是否正常了即可，否则，会对Ranking各服务造成压力。同时由于只有预发布环境的一个Matching服务做此检测，在发送告警时，也很好的避免了大量重复告警。
由于Ranking服务数数百个，在Ranking发布新版本时，我们仍然会收到大量的server不可用告警，优化的办法是，在第2步告警之前，使用全局的AtomicLongMap&lt;String&gt;.incrementAndGet(server.host + ":" + server.port)，在此返回值小于5的情况下告警，否则不告警；在第3步server服务正常时，使用AtomicLongMap&lt;String&gt;.remove(server.host + ":" + server.port)清理掉此server即可。之所以使用AtomicLongMap，是由于作为全局变量，对其的操作应该是线程安全的。

### 动态服务列表RankingServerList
``` java
public class RankingServerList implements ServerList<Server> {

	@Override
	public List<Server> getInitialListOfServers() {
		return getAllServers();
	}

	@Override
	public List<Server> getUpdatedListOfServers() {
		return getAllServers();
	}

	private List<Server> getAllServers() {

        // 1. 预发布环境，从Redis中获取所有Ranking服务
        // 2. 正式环境，从Redis中获取非服务异常集合中的Ranking服务
        // 3. 正式环境，重排此List<Server>列表，保证各服务器上的Ranking服务交替出现
	}
}
```
Redis中有两个列表，一个是Ranking所有服务的列表，一个是Ranking服务异常的列表，实际存放在Redis的Set中，第2步利用Redis Set的SDIFF便可获得。预发布环境塞入的是所有的Ranking服务，是为了让预发布环境来监测所有的Ranking服务，具体可看上面的RankingPing。第3步是一种优化手段，配合之后讲到的RankingRobinRule，避免出现Matching先请求Ranking服务的某一台服务器，将其上所有Ranking服务遍历完后，再去请求Ranking服务的其他服务器，导致单台服务器负载过高的情况。

### 轮询算法RankingRobinRule
``` java
public class RankingRobinRule extends RoundRobinRule {
    
	public Server choose(ILoadBalancer lb, Object key) {

		// 1. 从RoundRobinRule中拷贝其choose算法
        // 2. 修改其中的incrementAndGetModulo方式为全局轮询的方式
	}
}
```
之前使用的默认轮询方式RoundRobinRule，并不理想，因为有多个Matching服务，一方面很有可能请求到同一个Ranking服务，另一方面虽然在大量的请求Ranking情况下，对每一台Ranking的服务器的请求量大体大体相同，但是还是有可能出现在一段时间内，所有的请求都打在同一个Ranking的服务器上的情况。
使用Redis的String INCR便可实现全局轮询的功能，代码如下：
``` java
private int getIndex(int size) {
	long l = jedis.incrBy(RK_SERVERS_SEQ_KEY, 1L);
	int index = (int) (l % size);
	return index;
}
```
其中size未Ranking服务数，从Ranking服务列表中直接get到此index位置的server即可。l为long类型，根据估算，最近数年内是不会出现越界问题的。

