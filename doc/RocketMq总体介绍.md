# RocketMq总体介绍

#### 1. RocketMQ 是什么，有什么特点

![](http://15878290.s21i.faiusr.com/2/ABUIABACGAAgyOqq4gUogs2-7wUw8Qc43wQ.jpg)

>1. 是一个***队列模型***的消息中间件，具有高性能、高可靠、高实时、分布式特点
>2. Producer、Consumer、队列都可以分布式
>3. Producer向一些队列轮流发送消息，队列集合称为Topic，Consumer 如果做**广播消费**，则一个Consumer实例消费这个Topic 对应的所有队列，如果做**集群消费**，则多个Consumer 实例平均消费这个topic 对应的队列集合
>4. 能够保证严格的消息顺序
>5. **亿级消息堆积能力**
>6. **实时的消息订阅机制**

#### 2. RocketMQ 物理部署结构

![](http://15878290.s21i.faiusr.com/2/ABUIABACGAAgrOuq4gUojI74xAUwwwg45gQ.jpg)

>1. Name Server 是一个几乎无状态节点，可集群部署，类似于ZK的存在，但是比ZK更加的轻量。
>2. Broker 部署相对复杂，Broker 分为Master和Slave，一个Master 可以对应多个Slave，但是一个Slave 只能对应一个Master，***注意：slave只提供读，不提供写***
>3. Producer与Name Server 集群中的其中一个节点（随机选择）建立长连接，定期从Name Server获取Topic路由信息，并向提供Topic服务的Master 建立长连接，且定时向Master发送心跳。Producer 完全无状态，可集群部署
>4. Consumer也与Name Server 集群中的其中一个节点（随机选择）建立长连接，定期从Name Server获取Topic路由信息，并向提供Topic服务的Master，Slave建立长连接，且定时向Master，Slave发放心跳，Consumer既可以从Master 订阅消息，也可以从Slave 订阅消息，订阅规则由Broker 配置决定。

#### 3. RocketMQ 逻辑部署结构

![](http://15878290.s21i.faiusr.com/2/ABUIABACGAAgwfOq4gUo05HAvQcwqQg4zAQ.jpg)

> 1. Producer Group
>
>    用来表示一个发送消息应用，一个Producer Group下包含多个Producer 实例，可以是多台机器，也可以是一台机器的多个迕程，或者一个进程的多个Producer 对象。一个Producer Group 可以发送多个Topic消息，Producer Group 作用如下：
>
>    - 标识一类  Producer
>
>    - 可以通过运维工具查询这个发送消息应用下有多少个Producer实例
>
>    - ***发送分布式事务消息，如果Producer中途意外宕机，Broker会主动调用回调Producer Group内的任意一台机器来确认事务状态***
>
> 2. Consumer Group
>
>    用来表示一个消费消息应用，支持广播消费以及集群消费。

