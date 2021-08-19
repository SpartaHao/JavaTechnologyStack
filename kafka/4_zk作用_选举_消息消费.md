## 13. Zookeeper在kafka中有哪些作用？
### Broker注册
Broker是分布式部署并且相互之间相互独立，但是需要有一个注册系统能够将整个集群中的Broker管理起来，此时就使用到了Zookeeper。在Zookeeper上会有一个专门用来进行Broker服务器列表记录的节点.  
每个Broker在启动时，都会到Zookeeper上进行注册，即**到/brokers/ids下创建属于自己的节点**，如/brokers/ids/[0…N]。

Kafka使用了全局唯一的数字来指代每个Broker服务器，不同的Broker必须使用不同的Broker ID进行注册，**创建完节点后，每个Broker就会将自己的IP地址和端口信息记录到该节点中去。其中，Broker创建的节点类型是临时节点，一旦Broker宕机，则对应的临时节点也会被自动删除**。

### Topic注册
在Kafka中，同一个Topic的消息会被分成多个分区并将其分布在多个Broker上，这些**分区信息及与Broker的对应关系也都是由Zookeeper在维护**，由专门的节点来记录，如：
Kafka中每个Topic都会以/brokers/topics/[topic]的形式被记录，如/brokers/topics/login和/brokers/topics/search等。

### 生产者负载均衡
由于同一个Topic消息会被分区并将其分布在多个Broker上，因此，生产者需要将消息合理地发送到这些分布式的Broker上，那么如何实现生产者的负载均衡，Kafka支持传统的四层负载均衡，也支持Zookeeper方式实现负载均衡。
使用Zookeeper进行负载均衡，由于每个Broker启动时，都会完成Broker注册过程，**生产者会通过该节点的变化来动态地感知到Broker服务器列表的变更，这样就可以实现动态的负载均衡机制**。
生产者发送消息是从/brokers/topics/[topic]/partitions/[partition]/state节点获取leader信息，发送到leader的

### 消费者负载均衡
与生产者类似，Kafka中的消费者同样需要进行负载均衡来实现多个消费者合理地从对应的Broker服务器上接收消息，每个消费者分组包含若干消费者，每条消息都只会发送给分组中的一个消费者，不同的消费者分组消费自己特定的Topic下面的消息，互不干扰。

### 消费分区与消费者的关系
对于每个消费者分组，Kafka都会为其分配一个全局唯一的Group ID，同一个消费者分组内部的所有消费者共享该ID。同时，Kafka为每个消费者分配一个Consumer ID，通常采用"Hostname:UUID"形式表示。
在Kafka中，**规定了每个消息分区有且只能同时有一个消费者进行消费，因此，需要在Zookeeper上记录消息分区与消费者之间的关系**，/consumers/[group_id]/owners/[topic]/[broker_id-partition_id]
其中，[broker_id-partition_id]就是一个消息分区的标识，节点内容就是该消费分区上消息消费者的Consumer ID。

### 消息消费进度Offset记录
在消费者对指定消息分区进行消息消费的过程中，需要定时地将分区消息的消费进度Offset记录到Zookeeper上，**以便在该消费者进行重启或者其他消费者重新接管该消息分区的消息消费后，
能够从之前的进度开始继续进行消息消费**。Offset在Zookeeper中由一个专门节点进行记录，其节点路径为:
/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id]，节点内容就是Offset的值。

### controller选举
Kafka Controller的选举是依赖Zookeeper来实现的，**在Kafka集群中哪个broker能够成功创建/controller这个临时（EPHEMERAL）节点他就可以成为Kafka Controller**。

### 消费者注册
消费者服务器在初始化启动时加入消费者分组的步骤如下：  
* 注册到消费者分组。每个消费者服务器启动时，都会到Zookeeper的指定节点下创建一个属于自己的消费者节点，例如/consumers/[group_id]/ids/[consumer_id]，完成节点创建后，**消费者就会将自己订阅的Topic信息写入该临时节点**。
* 对消费者分组中的消费者的变化注册监听。每个消费者都需要关注所属消费者分组中其他消费者服务器的变化情况，即对/consumers/[group_id]/ids节点**注册子节点变化的Watcher监听，一旦发现消费者新增或减少，就触发消费者的负载均衡**。
* 对Broker服务器变化注册监听。消费者需要对/broker/ids/[0-N]中的节点进行监听，如果发现Broker服务器列表发生变化，那么就根据具体情况来决定是否需要进行消费者负载均衡。
* 进行消费者负载均衡。为了让同一个Topic下不同分区的消息尽量均衡地被多个消费者消费而进行消费者与消息分区分配的过程，通常，对于一个消费者分组，如果组内的消费者服务器发生变更或Broker服务器发生变更，会发出消费者负载均衡。

以下是kafka在zookeep中的详细存储结构图：

![zkInKafka](https://img-blog.csdnimg.cn/20210419151340613.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvY2FpODA1OA==,size_16,color_FFFFFF,t_70#pic_center)  

https://blog.csdn.net/bocai8058/article/details/82956613


## 14. Kafka是怎么实现选举的？哪些地方需要选举？
### 控制器的选举
在Kafka集群中会有一个或多个broker，其中有一个broker会被选举为控制器（Kafka Controller），它**负责管理整个集群中所有分区和副本的状态等工作**。比如当某个分区的leader副本出现故障时，
由控制器负责为该分区选举新的leader副本。再比如当检测到某个分区的ISR集合发生变化时，由控制器负责通知所有broker更新其元数据信息。

Kafka Controller的选举是依赖Zookeeper来实现的，在Kafka集群中哪个broker能够成功创建/controller这个临时（EPHEMERAL）节点他就可以成为Kafka Controller。

### 分区leader的选举
分区leader副本的选举由Kafka Controller 负责具体实施。当**创建分区**（创建主题或增加分区都有创建分区的动作）或**分区上线**（比如分区中原先的leader副本下线，此时分区需要选举一个新的leader上线来对外提供服务）的时候都需要执行leader的选举动作。

基本思路是按照**AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中**。一个分区的AR集合在分配的时候就被指定，并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，
而分区的ISR集合中副本的顺序可能会改变。**注意这里是根据AR的顺序而不是ISR的顺序进行选举的**。这个说起来比较抽象，有兴趣的读者可以手动关闭/开启某个集群中的broker来观察一下具体的变化。

还有一些情况也会发生分区leader的选举，比如当**分区进行重分配**（reassign）的时候也需要执行leader的选举动作。这个思路比较简单：从重分配的AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中。  
再比如当发生优先副本（preferred replica partition leader election）的选举时，直接将优先副本设置为leader即可，AR集合中的第一个副本即为优先副本。

还有一种情况就是当**某节点被优雅地关闭**（也就是执行ControlledShutdown）时，位于这个节点上的leader副本都会下线，所以与此对应的分区需要执行leader的选举。  
这里的具体思路为：从AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中，与此同时还要确保这个副本不处于正在被关闭的节点上。

### 消费者相关的选举
**组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader**，这个选举的算法也很简单，分两种情况分析。**如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader**。
如果某一时刻leader消费者由于某些原因退出了消费组，那么会重新选举一个新的leader，这个重新选举leader的过程又更“随意”了，相关代码如下：
````javascript
//scala code.
private val members = new mutable.HashMap[String, MemberMetadata]
var leaderId = members.keys.head
````
解释一下这2行代码：在GroupCoordinator中消费者的信息是以HashMap的形式存储的，其中key为消费者的member_id，而value是消费者相关的元数据信息。leaderId表示leader消费者的member_id，
它的取值为HashMap中的第一个键值对的key，这种选举的方式基本上和随机无异。总体上来说，**消费组的leader选举过程是很随意的**。

https://honeypps.com/mq/kafka-basic-knowledge-of-selection/  

## 15. 如果kafka消费者消费超时会发生什么？怎么避免kafka的消费超时？
### 消费超时的影响
1. 会被踢出消费者组，进而发生rebalance
2. 因为没有消费成功，kafka会重新发送消息，导致**消息重复消费**，如果是业务问题导致，该消息一直消费不了，会频繁rebalance，进而整个消息处理停滞

### 如何避免kafka的消费超时
1. **提高max.poll.interval.ms参数**，该参数是kafka消息消费的超时时间，默认值是5分钟，超过会被踢出消费者组
2. **调低max.poll.records配置**，减少消费者每次拉取的消息数量。
上面的两个参数都是创建kafka消费者时的配置参数

## 16. 如何保证消息只消费一次？如何保证消息不丢失？
### 防止消息重复消费
消费kafka消息做幂等处理（幂等函数，或幂等方法，是指可以使用相同参数重复执行，并能获得相同结果的函数）。在redis中保存一个map，key是topic_partition,
value是上一次消费的offset值，在拉取消息的时候，如果缓存中有对应topic_partition的key（没有代表是第一次消费），且判断当前消息的offset值小于缓存中存的offset值，说明是重复消费，
直接忽略该消息（直接删除）。拉取的一批消息消费完成后，更新map对象的offset值，将其塞回缓存，然后手动提交offset值。
注意**如果kafka地址变了，需要将redis中的map清空，防止消息无法消费**。

### 如何做到消息不丢失：
1. 生产端要等消息写到日志，收到ack后在发送下一批，否则进行重试；ack设为-1；
2. 消费端要设为**手动提交offset值**，如果consumer挂了可以重新消费，要做好幂等保障

https://www.cnblogs.com/helios-fz/p/12119727.html  
<http://www.kafka.cc/archives/260.html>   

## 17. 如何设计保证kakfa中有某个特征的消息的是严格按照顺序消费的？
**kafka只保证单partition有序**，如果topic有多个partition，消费数据时就不能保证数据的顺序。在需要严格保证消息的消费顺序的场景下，需要将partition数目设为1，但是此时就没有办法保证吞吐量了，基本不用这种方式。

Kafka 中发送1条消息的时候，可以指定(topic, partition, key) 3个参数。partiton 和 key 是可选的。如果你指定了 partition，那就是所有消息发往同1个 partition，就是有序的。
并且在消费端，Kafka 保证1个 partition 只能被1个 consumer 消费。或者你**指定 key（比如 order id），具有同1个 key 的所有消息，会发往同1个 partition，也是有序的**。

但是消费者里可能会有多个线程来并发来处理消息。因为如果消费者是单线程消费数据，那么这个吞吐量太低了。而多个线程并发的话，顺序可能就乱掉了。将具有相同key的数据都存储在同一个queue，
然后对于N个线程，每个线程分别消费一个queue即可，但是放到内存的话有可能消息会丢失，业务复杂性也比较高，如果不是压力很大的话，一次只拉取一条消息即可。

https://blog.csdn.net/qq_31329893/article/details/90451889  
https://www.zhihu.com/question/266390197/answer/527249762。

## 18. Kafka怎么判断一个broker是否还存活？
（1）节点必须可以维护和ZooKeeper的连接，Zookeeper通过心跳机制检查每个节点的连接  
（2）如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久  

https://zhuanlan.zhihu.com/p/260648154
