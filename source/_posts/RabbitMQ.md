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

Exchange 分为 直连[direct]交换机、扇形[fanout]交换机、主题[topic]交换机、头[headers]交换机。

* direct exchange 与 queue 绑定时需要指定一个 routing_key，消息被发送到 direct exchange 时也需要指定一个 routing_key，消息会被分发到具有相同 routing_key 绑定的 queue 中；

* fanout exchange 是一个广播型的交换机，发送到 fanout exchange 的消息被分发到与之绑定的所有 queue 中；

* topic exchange 是更复杂的 direct exchange，它的 routing_key 必须是以 `.` 分隔的几个字符串，一般是以多个维度分隔 (例如日志系统中以 “应用.级别” [shop.info]来制定)，topic exchange 与 queue 绑定时可以指定通配的 routing_key，`*` 表示一个单词，`#` 表示任意数量的单词；

* headers exchange 不通过 routing_key 来建立与 queue 的绑定，而是使用消息的属性来进行分发。

## 应用场景

### 工作队列

工作队列也就是经常说的“生产者/消费者”模型。特点是生产者生产的消息不会丢失，若此时没有消费者，则消息会被存储在 queue 中。

此时应当建立 持久化[durable] 的 queue，来保证消息不会丢失，当有消费者连接上时，消息按照 先进先出[FIFO] 的顺序被消费者消费掉。
同时可以开启多个消费者，加快处理速度。

### 发布/订阅模式

发布/订阅[Publish/Subscribe] 模式特点是生产者发布的消息只有在线的消费者能够收到，之后再连接上的消费者不能收到之前的消息。

此时每个消费者连接上之后应当 定义[declare] 一个属于自己的临时队列，并与 exchange 绑定，此队列在消费者断开连接时自动销毁。 

### 远程过程调用

利用RabbitMQ实现 远程过程调用[RPC] 也就是在 客户端[Client]向 服务端[Server]指定的 exchange 写入远程过程调用的参数时同时写入 Server 需要将返回值写入的 Queue。

简单的，可以为每一次 RPC 调用建立一个临时队列作为回调队列，然后将此队列的名称发送给 Server，Server在处理业务后将结果写入此回调队列。
但这样不是很高效，高效的方式是为每一个客户端建立一个回调队列，此时又带来的问题是无法确认 Server 写回的响应无法确认是哪个请求发出的。
此时可以在消息中指定 `correlation_id` 属性，用于标记单次请求，而在接收到响应时可以验证此 `correlation_id` 是否是上次请求发出的。

### 延迟队列

利用 RabbitMQ 的消息过期特性可以实现延迟队列。在一些需要延迟一段时间之后执行某些操作的场景下特别有用。

RabbitMQ 中若消息在队列中过期，则此时消息会变成 死信[dead-letter]，设置 queue 的 `x-dead-letter-exchange` 来将队列中的消息变为死信时发送到指定的 exchange。
设置 queue 的 `x-dead-letter-routing-key` 可以将消息的 routing_key 修改为指定的值。

可以建立两个 queue，一个设置 `x-dead-letter-exchange`，一个不设置，前一个队列中的消息在过期时自动被转发到后一个队列，消费者只消费后一个队列中的消息。

这里需要注意的是消息的过期时间应当设置在 queue 上[x-message-ttl]（或者同一个 queue 中的消息设置相同的过期时间），若在同一个 queue 上发布不同过期时间的消息会导致先入队的过期时间长的消息阻塞了后入队的过期时间短的消息。
也就是说，有多少个过期时间的情况就要设置多少个队列来存放这些消息。

## 权限系统

一个 RabbitMQ 实例已 vhost[Virtual Host] 来划分区域供不同的业务使用，每一个 RabbitMQ 的用户可以对多个 vhost 拥有权限，权限又分为
config、write、read，分别对应了配置 交换机和队列、写入交换机消息、从队列中读取消息。每一个配置可以指定一个正则表达式来指定可以操作的资源名称。

针对每个用户还可以设置 `tag` 其中有几个系统预先定义好的：

* administrator 可登陆管理控制台(启用management plugin的情况下)，可查看所有的信息，并且可以对用户，策略(policy)进行操作；

* monitoring 可登陆管理控制台(启用management plugin的情况下)，同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)；

* policymaker 可登陆管理控制台(启用management plugin的情况下), 同时可以对policy进行管理，但无法查看节点的相关信息；

* management 仅可登陆管理控制台(启用management plugin的情况下)，无法看到节点信息，也无法对策略进行管理。

## 命令行管理工具

`rabbitmqctl` 可以查看用户、权限、连接、交换机、队列等，可以对用户、vhost 进行新增、修改、删除等。

`rabbitmqadmin` 可以定义交换机、队列，写入消息，消费消息等。

`rabbitmq-plugins` 可以对 RabbitMQ 的插件进行管理。可以启用 `rabbitmq_management` 插件通过 web 界面进行管理(端口15672)，需要注意的是默认的 guest 用户只能在本机进行登录，如需要远程登录可以使用 `rabbitmqctl` 添加新的用户。

## 参考链接

[RabbitMQ - Getting started with RabbitMQ](http://www.rabbitmq.com/getstarted.html)

[RabbitMQ - Access Control (Authentication, Authorisation) in RabbitMQ](http://www.rabbitmq.com/access-control.html)

[RabbitMQ - Highly Available Queues](http://www.rabbitmq.com/ha.html)