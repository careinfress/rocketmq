# RocketMq存储相关（重要）

#### 1. rocketmq零拷贝

​	Consumer消费消息的过程，使用了零拷贝的技术，RocketMq使用了**mmap + write**方式实现零拷贝。因为大部分的队列消息都是属于小块消息。这种方式去实现的效果是最好的。

​	两种零拷贝实现方式的区别：

>1. 使用namp+write方式
>   - 优点： 即使频繁使用，使用小块文件传输，效率很高
>   - 缺点： 不能很好的利用DMA方式，会比sendfile多消耗CPU，内存安全性控制复杂，需要避免JVM Crash问题
>2. 使用sendfile方式
>   - 优点： 可以利用DMA方式，消耗CPU较少，大块文件传输效率高，无内存安全性问题
>   - 缺点： 小块文件效率低于nmap方式，只能是BIO方式传输，不能使用NIO

#### 2. 文件系统

- RocketMQ 选择Linux Ext4 文件系统，原因如下：

>1. Ext4 文件系统删除1G 大小的文件通常耗时小于50ms，而Ext3 文件系统耗时约1s 左右，且删除文件时，磁盘IO压力极大，会导致IO写入超时

- 文件系统局面需要做以下调优措施：

>1. 文件系统IO 调度算法需要调整为deadline，因为deadline 算法在随机读情冴下，可以合幵读请求为顺序跳跃方式，从而提高读IO 吞吐量。

#### 	3. 数据存储结构

![](http://15878290.s21i.faiusr.com/2/ABUIABACGAAgw6Wr4gUoz7jN4AQwkQo49AU.jpg)

~~~java
|-abort
|-checkpoint
|-config
|	|-consumerFilter.json //消费者过滤参数
|	|-consumerFilter.json.bak
|	|-consumerOffset.json //消费偏移量
|	|-consumerOffset.json.bak
|	|-delayOffset.json  //延迟消费偏移量
|	|-delayOffset.json.bak
|	|-subscriptionGroup.json //订阅组
|	|-subscriptionGroup.json.bak
|	|-topics.json 
|	|-topics.json.bak
|-commitlog
|	|-00000000000000000000 //存储大小
|	|-00000000001073741824 //1G 这个配置可以在broker.properties中配置
|-consumequeue
|	|-TopicA //创建的Topic
|	|	|- 0
|	|	|- 1
|	|	|- 2
|	|	|- 3
|	|-TopicB//创建的Topic
|	|	|- 0
|	|	|- 1
|	|	|- 2
|	|	|- 3
~~~
#### 	4. 数据可靠性

