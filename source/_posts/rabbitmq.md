---
title: RabbitMQ 知识汇总
date: 2017-11-07 18:55
tags:
    - RabbitMQ
---

### 突破 rabbitmq-delayed-message-exchange 插件对延迟消息延迟时间的设置
该插件最多只能支持到 (2^32)-1 milliseconds的延迟，大约49天，可以满足绝大多数延迟业务的需求。如果需要延迟更长的时间，例如：A业务需要延迟61天，今天是 2017-11-1，需延迟到 2018-1-1，A先延迟30天，到 2017-12-1 接收到该延迟消息，发现未到 2018-1-1，确认本次延迟消息后，再发送延迟消息到 2018-1-1，以满足该延迟要求。

### channel.basicQos方法
现象：队列有任务，且有消费者，但是任务堆积，大量未消费。
channel.basicQos(2)表示MQ同一时间仅将两条消息给到消费者，即MQ对该消费者的 unack-ed message不会大于2。channel.basicQos(0)即对MQ发送消息不做限制。不建议统一设置 channel.basicQos(0)，例如A消费者处理任务耗时长，且任务过多，但对处理的及时性不做限制，B对处理的及时性有要求，逐渐的A有可能会占用大量的消费线程，导致B无线程可用，从而影响了B处理的不及时。
该限制应由各业务决定，各业务需要根据自身业务需求，MQ的消费者线程总数来做设置。

### channel.basicConsume(queueName, true, consumer) 和 channel.basicReject(deliveryTag, true)
现象：重启项目后，队列消费正常，一段时间后，队列没有了消费者。
channel.basicConsume(queueName, true, consumer) 设置了该 channel autoAck 为 true，表示自动 ACK，如果业务处理出现异常，你希望通过 channel.basicReject(deliveryTag, true) 将消息重新入队，该做法是不正确的，和你之前的设置自相矛盾，在这种情况下，就会发现该队列没有消费者了，但是 MQ 客户端不抛错，仍然处于消费阻塞状态，目前还不清楚为什么 MQ 是这样的逻辑。

