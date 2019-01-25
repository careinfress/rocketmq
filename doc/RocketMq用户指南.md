# RocketMq用户指南

- **客户端如何寻址**

  RocketMQ 有多种配置方式可以令客户端找到 Name Server, 然后通过 Name Server 再找到 Broker 

  1. 代码中指定 Name Server 地址

     ~~~java
     producer.setNamesrvAddr("192.168.0.1:9876;192.168.0.2:9876");
     consumer.setNamesrvAddr("192.168.0.1:9876;192.168.0.2:9876");
     ~~~

- **自定义客户端行为**

  1. 客户端的公共配置 

     | 参数名                        | 默认值  | 说明                                                         |
     | :---------------------------- | :-----: | ------------------------------------------------------------ |
     | namesrvAddr                   |         | Name Server 地址列表，多个 NameServer 地址用分号隔开         |
     | clientIP                      | 本机 IP | 客户端本机 IP 地址，某些机器会发生无法识别客户端  IP 地址情况，需要应用在代码中强制指定 |
     | instanceName                  | DEFAULT | 客户端实例名称，客户端创建的多个 Producer  Consumer 实际是共用一个内部实例（这个实例包含  网络连接、线程资源等） |
     | clientCallbackExecutorThreads |    4    | 通信层异步回调线程数                                         |
     | pollNameServerInteval         |  30000  | 轮询 Name Server 间隔时间，单位毫秒                          |
     | heartbeatBrokerInterval       |  30000  | 向 Broker 发送心跳间隔时间，单位毫秒                         |
     | persistConsumerOffsetInterval |  5000   | 持久化 Consumer 消费进度间隔时间，单位毫秒                   |

  2. Producer 配置 

     | 参数名                           | 默认值           | 说明                                                         |
     | -------------------------------- | ---------------- | ------------------------------------------------------------ |
     | producerGroup                    | DEFAULT_PRODUCER | Producer 组名，多个 Producer 如果属于一个应用，发送同样的消息，则应该将它们归为同一组 |
     | createTopicKey                   | TBW102           | 在发送消息时，自动创建服务器不存在的 topic，需要指定 Key     |
     | defaultTopicQueueNums            | 4                | 在发送消息时，自动创建服务器不存在的 topic，默认创建的队列数 |
     | sendMsgTimeout                   | 10000            | 发送消息超时时间，单位毫秒                                   |
     | compressMsgBodyOverHowmuch       | 4096             | 消息 Body 超过多大开始压缩（Consumer 收到消息会自动解压缩），单位字节 |
     | retryAnotherBrokerWhenNotStoreOK | FALSE            | 如果发送消息返回 sendResult，但是 sendStatus!=SEND_OK，是否重试发送 |
     | maxMessageSize                   | 131072           | 客户端限制的消息大小，超过报错，同时 服务端也会限制          |

  3.  PushConsumer 配置 

     | 参数名                       | 默认值                        | 说明                                                         |
     | ---------------------------- | ----------------------------- | ------------------------------------------------------------ |
     | consumerGroup                | DEFAULT_CONSUMER              | Consumer 组名，多个 Consumer 如果属于一个应用，订阅同样的消 息，且消费逻辑一致，则应该将它 们归为同一组 |
     | messageModel                 | CLUSTERING                    | 消息模型，支持以下两种  1、集群消费 2、广播消费              |
     | allocateMessageQueueStrategy | AllocateMessageQueueAveragely | Rebalance 算法实现策略                                       |
     | subscription                 | {}                            | 订阅关系                                                     |

- **Message 数据结构**

  1. 针对 Producer 

     | 字段名         | 默认值 | 说明                                                         |
     | -------------- | ------ | ------------------------------------------------------------ |
     | Topic          | null   | 必填，线下环境不需要申请，线上环境需要申请后才能使用         |
     | Body           | null   | 必填，二进制形式，序列化由应用决定，Producer 与 Consumer 要协商好序列化形式 |
     | Tags           | null   | 选填，类似于 Gmail 为每封邮件设置的标签，方便服务器过滤使用。目前只支持每个消息设置一个 tag，所以也可以类比为 Notify 的 MessageType概念 |
     | Keys           | null   | 选填，代表这条消息的业务关键词，服务器会根据 keys 创建哈希索引，设置后， 可以在 Console 系统根据 Topic、Keys 来查询消息，由于是哈希索引，请尽可能 保证 key 唯一，例如订单号，商品 Id等 |
     | DelayTimeLevel | 0      | 选填，消息延时级别，0 表示不延时，大于 0 会延时特定的时间才会被消费 |
     | WaitStoreMsgOK | TRUE   | 选填，表示消息是否在服务器落盘后才返回应答                   |

  2. 针对 Consumer

     在Producer端，使用 `com.alibaba.rocketmq.common.message.Message` 这个数据结构，由于Broker会为Message 增加数据结构，所以消息到达 Consumer 后，会在 Message 基础之上增加多个字段，Consumer 看到的是  `com.alibaba.rocketmq.common.message.MessageExt`  这个数据结构，MessageExt 继承于 Message 

