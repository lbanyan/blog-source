---
title: RabbitMQ 延迟队列插件 x-delay Bug
tags:
    - RabbitMQ
    - x-delay
    - x-dead-letter-exchange
    - x-dead-letter-routing-key
---

### rabbitmq-delayed-message-exchange 插件延迟队列实现
``` java
// ... elided code ...
long delayTime = 5000;
byte[] messageBodyBytes = "delayed payload".getBytes("UTF-8");
Map<String, Object> headers = new HashMap<String, Object>();
headers.put("x-delay", delayTime);
AMQP.BasicProperties.Builder props = new AMQP.BasicProperties.Builder().headers(headers);
channel.basicPublish("my-exchange", "", props.build(), messageBodyBytes);
// ... more code ...
```

<!--more-->

### 问题及分析
延迟队列生产者使用 x-delay 设置延迟时长，单位 ms。经测试，此 delayTime 设置过长（未超过 long 限制）时，延迟队列失效，会立刻被消费掉。
此处应该是该插件有问题，多次尝试后，发现一个有趣的现象，设置 delayTime = 2147483648l * 2 时出现此问题，delayTime = 2147483648l * 2 - 1 时正常，说明该插件使用了 4 个字节用来表示延时时常，delayTime设置过大溢出后，变为负数，导致该消息立即被执行。因而该插件最多只能支持延迟 49 天左右。
费了这么半天的劲，竟然没有去读它的文档。[rabbitmq-delayed-message-exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)文档Performance Impact一节已明确说明了这个限制。
> For each message that crosses an "x-delayed-message" exchange, the plugin will try to determine if the message has to be expired by making sure the delay is within range, ie: Delay > 0, Delay =< ?ERL_MAX_T (In Erlang a timer can be set up to (2^32)-1 milliseconds in the future).

### 使用 x-dead-letter-exchange 和 x-dead-letter-routing-key 实现延迟队列 Bug
发送延迟消息方法：
``` java
public void pushDelay(String queueName, byte[] message, long delayMillisecond) throws Exception {
		AssertUtil.isTrue(delayMillisecond > 0, "delayMillisecond must greater than zero");
		Channel channel = getChannel();
		// 定义队列
		channel.queueDeclare(queueName, true, false, false, null);
		// 定义一个中间队列
		HashMap<String, Object> arguments = new HashMap<String, Object>();
		arguments.put("x-dead-letter-exchange", "");
		arguments.put("x-dead-letter-routing-key", queueName);
		String tmpQueueName = queueName + "@tmp";
		channel.queueDeclare(tmpQueueName, true, false, false, arguments);
		// 设置消息的属性 过期时间+持久化
		AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
		AMQP.BasicProperties properties = builder.expiration(delayMillisecond + "").deliveryMode(2).build();
		// 消息发往中间队列,消息过期后mq会转发给queue
		channel.basicPublish("", tmpQueueName, properties, message);
	}
```
接收延迟消息的方法：
``` java
public void popDelay(String queueName, boolean autoAck, int prefetchCount, Consumer consumer) throws Exception {
		Channel channel = getChannel();
		// 定义队列
		channel.queueDeclare(queueName, true, false, false, null);
		// 定义一个中间队列
		HashMap<String, Object> arguments = new HashMap<String, Object>();
		arguments.put("x-dead-letter-exchange", "");
		arguments.put("x-dead-letter-routing-key", queueName);
		String tmpQueueName = queueName + "@tmp";
		channel.queueDeclare(tmpQueueName, true, false, false, arguments);
		channel.basicQos(prefetchCount);
		channel.basicConsume(queueName, autoAck, consumer);
	}
```
网上流传着大量的这种实现方法，这种方法也是有限制的，却没见有人提起。在仅对队列设置过期时间或对每个消息设置的过期时长相同的场景下，该实现是没有问题的。
有问题的情况出在这里，<font color='red'>队列中各消息的过期时间不定时</font>，例如电商秒杀活动，设置了秒杀的开始时间，到开始时间时，自动开启该秒杀活动等，就无法正常使用死信队列了。假设你使用上述代码定义的 queueName="delayQueue"，使用过程中，你会发现 delayQueue@tmp 队列消息会堆积，应该过期的消息未能及时流转到 delayQueue 队列中，从业务的角度来看，就是消费者通过 popDelay 一直没有消费延迟消息。
问题出在 MQ 检查 delayQueue@tmp 队列中第一个消息时，发现其并未到期，便不会继续检查之后的消息，于是，即使第一个消息之后有过期的消息，也无法流转到 delayQueue 队列。这是由队列的特性决定的。你不能去消费队列中间的消息，队列必须先进先出。在这种情况下，如果你定时遍历一遍该队列，并将其重新入队，在此期间，由于MQ会检测各消息的过期时间，从而已过期的消息便会被正常消费掉。