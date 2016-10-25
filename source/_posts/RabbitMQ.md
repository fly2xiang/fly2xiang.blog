title: "RabbitMQ 配置与使用记录"
date: 2016-10-24 13:22:52
tags:
- MQ
- RabbitMQ
- AMQP
- Message Queue
- 中间件
categories: 
- 中间件

---

## 安装

RabbitMQ 的安装可以参考 [安装页面](https://www.rabbitmq.com/download.html)，根据不同的操作系统选择对应的安装方式。

安装之后系统中会出现 `rabbitmqctl`、`rabbitmq-plugins`、`rabbitmq-server` 几个可执行文件。

<!-- more -->

## 概念

RabbitMQ 实现了 AMQP 0-9-1[Advanced Message Queuing Protocol]，
其中重要的概念是 交换机[Exchange] 和 队列[Queue]，Exchange 和 Queue 通过 绑定[Bind] 操作进行绑定。
生产者[Producer] 不需要关注 消息[Message] 实际被 路由[Routing] 到哪个队列，而 消费者[Consumer] 也不需要关注 Message 是从哪个 Exchange 接收的。

Exchange 和 Queue 的 Bind 方式有 direct、
