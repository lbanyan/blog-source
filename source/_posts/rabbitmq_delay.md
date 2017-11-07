---
title: RabbitMQ 延迟队列插件 x-delay Bug
tags:
    - RabbitMQ
    - x-delay
---

## 源码
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

## 问题及分析
延迟队列生产者使用 x-delay 设置延迟时常，单位 ms。经测试，此 delayTime 设置过长（未超过 long 限制）时，延迟队列失效，会立刻被消费掉。
此处应该是该插件有问题，多次尝试后，发现一个有趣的现象，设置 delayTime = 2147483648l * 2 时出现此问题，delayTime = 2147483648l * 2 - 1 时正常，说明该插件使用了 4 个字节用来表示延时时常，delayTime设置过大溢出后，变为负数，导致该消息立即被执行。
