# RocketMQ粗记

[中文文档](https://rocketmq.apache.org/zh/)

一些注意点：

1. 连接公网IP需要在`conf/broker.conf`中配置brokerIP1，配置为公网IP；

   ```properties
   brokerIP1 = ****
   brokerClusterName = DefaultCluster
   brokerName = broker-a
   brokerId = 0
   deleteWhen = 04
   fileReservedTime = 48
   brokerRole = ASYNC_MASTER
   flushDiskType = ASYNC_FLUSH
   ```

2. 单条消息不建议传输超大负载；

3. 索引键建议传业务ID，如用户ID，订单ID等，方便后续查找定位；



## RocketMQ领域模型

![RocketMQ领域模型](https://rocketmq.apache.org/zh/assets/images/mainarchi-9b036e7ff5133d050950f25838367a17.png)

### 消息生产

- <u>生产者Producer</u>

Apache RocketMQ 中用于产生消息的运行实体，一般集成于业务调用链路的上游。生产者是轻量级匿名无身份的。

### 消息存储

- <u>主题Topic</u>

  消息传输和存储的分组容器，主题内部由多个队列组成，消息的存储和水平扩展实际是通过主题内的队列实现的。

- <u>队列MessageQueue</u>

  消息传输和存储的实际单元容器，类比于其他消息队列中的分区。 Apache RocketMQ 通过流式特性的**无限队列结构**来存储消息，消息在队列内具备顺序性存储特征。

- <u>消息Message</u>

  最小传输单元。消息具备不可变性，在初始化发送和完成存储后即不可变。

### 消息消费

- <u>消费者分组ConsumerGroup</u>

  Apache RocketMQ 发布订阅模型中定义的独立的消费身份分组，用于统一管理底层运行的多个消费者（Consumer）。同一个消费组的多个消费者必须保持消费逻辑和配置一致，**共同分担该消费组订阅的消息**，实现消费能力的水平扩展。

- <u>消费者Consumer</u>

   消费消息的运行实体，一般集成在业务调用链路的下游。消费者必须被指定到某一个消费组中。

- <u>订阅关系Subscription</u>

  Apache RocketMQ 发布订阅模型中消息过滤、重试、消费进度的规则配置。订阅关系以消费组粒度进行管理，消费组通过定义订阅关系控制指定消费组下的消费者如何实现消息过滤、消费重试及消费进度恢复等。

  Apache RocketMQ 的订阅关系**除过滤表达式之外都是持久化的，即服务端重启或请求断开，订阅关系依然保留**。



### 消息传输模型介绍

- 点对点模型

![点对点模型](https://rocketmq.apache.org/zh/assets/images/p2pmode-fefdc2fbe4792e757e26befc0b3acbff.png)

消费者匿名，一条消息分给一个消费者

- 发布订阅模型

![发布订阅模型](https://rocketmq.apache.org/zh/assets/images/pubsub-042a4e5e5d76806943bd7dcfb730c5d5.png)

消费者有身份（订阅关系）,每个消费组可以拿到全量的消息。

Apache RocketMQ用的就是发布订阅模型。