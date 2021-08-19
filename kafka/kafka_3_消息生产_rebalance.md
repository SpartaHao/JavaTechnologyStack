## 10. 关于kafka中ISR、AR、HW、LEO、LSO、LW的含义详解
**分区中的所有副本统称为AR（Assigned Replicas）。所有与leader副本保持一定程度同步的副本（包括Leader）组成ISR（In-Sync Replicas），ISR集合是AR集合中的一个子集**。
消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。前面所说的“一定程度”是指可以忍受的滞后范围，
这个范围可以通过参数进行配置。与leader副本同步滞后过多的副本（不包括leader）副本，组成**OSR(Out-Sync Relipcas)**,由此可见：AR=ISR+OSR。在正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即AR=ISR,OSR集合为空。

Leader副本负责维护和跟踪ISR集合中所有的follower副本的滞后状态，当follower副本落后太多或者失效时，leader副本会吧它从ISR集合中剔除。在ISR集合中的副本才有资格被选举为leader，而在OSR集合中的副本则没有机会（这个原则可以通过修改对应的参数配置来改变）

ISR的伸缩：Kafka在启动的时候会开启两个与ISR相关的定时任务，名称分别为“isr-expiration"和”isr-change-propagation".。isr-expiration任务会周期性的检测每个分区是否需要缩减其ISR集合。
这个周期和“replica.lag.time.max.ms”参数有关。大小是这个参数一半。默认值为5000ms，当检测到ISR中有失效的副本的时候，就会缩减ISR集合。

有缩减就会有补充，那么kafka何时扩充ISR的？  
随着follower副本不断进行消息同步，follower副本LEO也会逐渐后移，并且最终赶上leader副本，此时follower副本就有资格进入ISR集合，**追赶上leader副本的判定准侧是此副本的LEO是否小于leader副本HW**，
这里并不是和leader副本LEO相比。ISR扩充之后同样会更新ZooKeeper中的/broker/topics//partition//state节点和isrChangeSet，之后的步骤就和ISR收缩的时的相同。

#### LW
LW是Low Watermark的缩写，俗称“低水位”，**代表AR集合中最小的logStartOffset值**，副本的拉取请求（FetchRequest，它有可能触发新建日志分段而旧的的被清理，进而导致logStartoffset的增加）和删除请求（DeleteRecordRequest）都可能促使LW的增长。

#### HW
HW是High Watermak的缩写， 俗称高水位，它表示了一个特定消息的偏移量（offset），消费之只能拉取到这个offset之前的消息。**ISR集合中最小的LEO即为分区的HW，对消费这而言只能消费HW之前的消息**。

如下，它代表一个日志文件，这个日志文件中有9条消息，第一消息的offset（LogStartOffset）为0，最后的一条消息offset为8，offset为9的消息用虚线框表示，代表下的一个待写入的消息。
日志文件的HW为6.表示消费者只能拉取到offset0至5之间的消息，而offset为6的消息对消费者而言是不可见的。  
![HW](https://img-blog.csdnimg.cn/20190621140447198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzk3NTIyMA==,size_16,color_FFFFFF,t_70)

#### LEO
LEO是Log End Offset的缩写，它表示了**当前日志文件中下一条待写入消息的offset**，如上图offset为9的位置即为当前日志文件LEO,**LEO的大小相当于当前日志分区中最后一条消息的offset值加1**。分区ISR集合中的每个副本都会维护自身的LEO。

**kafka的复制机制不是完全的同步复制，也不是单纯的异步复制，Kafka使用ISR的这种方式有效的权衡了数据可靠性与性能之间的关系**。事实上，同步复制要求所有能工作的Follower副本都复制完，这条消息才会被确认为成功提交，这种复制方式影响了性能。
而在异步复制的情况下， follower副本异步地从leader副本中复制数据，数据只要被leader副本写入就被认为已经成功提交。在这种情况下，如果follower副本都没有复制完而落后于leader副本，
如果突然leader副本宕机，则会造成数据丢失。

#### LSO
**LSO特指LastStableOffset，它具体与kafka的事务有关**。 消费端参数——isolation.level,这个参数用来配置消费者事务的隔离级别。
> “read_uncommitted”和“read_committed”，表示消费者所消费到的位置，**如果设置为“read_committed"，那么消费这就会忽略事务未提交的消息，既只能消费到LSO(LastStableOffset)的位置，默认情况下，”read_uncommitted",既可以消费到HW（High Watermak）的位置**。

> 注：follower副本的事务隔离级别也为“read_uncommitted"，并且不可修改。

这个LSO还会影响到kafka消费之后的量，(也就是kafka,Log,很多时候也称之为kafka堆积量)的计算 。如下图：  
![lso1] (https://img-blog.csdnimg.cn/20190621140605172.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzk3NTIyMA==,size_16,color_FFFFFF,t_70)

在图中，对每一个分区而言，**它Lag等于HW-ConsumerOffset的值**，其中ComsmerOffset表示当前的消费的位移，当然这只是针对普通的情况。如果为消息引入了事务，那么Lag的计算方式就会有所不同。
如果当消费者客户端的isolation.level的参数配置为“read_uncommitted"（默认），那么Lag的计算方式不受影响，**如果这个参数配置为“read_committed",那么就要引入LSO来进行计算了**。  
![lso2](https://img-blog.csdnimg.cn/20190621140742729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzk3NTIyMA==,size_16,color_FFFFFF,t_70)

**对于未完成的事务而言，LSO的值等于事务中的第一条消息所在的位置**，（firstUnstableOffset）  
**对于已经完成的事务而言，它的值等同于HW相同**，所以我们可以得出一个结论：**LSO≤HW≤LEO**
![lso3](https://img-blog.csdnimg.cn/20190621140811843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzk3NTIyMA==,size_16,color_FFFFFF,t_70)

所以，对于分区中未完成的事务，并且消费者客户端的isolation.level参数配置为”read_committed"的情况，**它对应的Lag等于LSO-ComsumerOffset的值**。

来自 <https://blog.csdn.net/lukabruce/article/details/101012815> 

## 11. Kafka的生产者发送消息流程是怎么样？
#### 写入方式
producer **采用 push 模式将消息发布到 broker，每条消息都被 append 到 patition 中，属于顺序写磁盘**（顺序写磁盘效率比随机写内存要高，保障 kafka 吞吐率）。

#### 消息路由
producer 发送消息到 broker 时，会根据分区算法选择将其存储到哪一个 partition。其路由机制为：  
1、 指定了 patition，则直接使用；  
2、 未指定 patition 但指定 key，通过对 key 的 value 进行hash 选出一个 patition  
3、 patition 和 key 都未指定，使用轮询选出一个 patition。  

#### 写入流程
producer 写入消息序列图如下所示：
![kafkaProduce](https://images2018.cnblogs.com/blog/1228818/201805/1228818-20180507200019142-182025107.png)

流程说明：  
1. producer 先从 zookeeper 的 "/brokers/.../state" 节点找到该 partition 的 leader 
2. producer 将消息发送给该 leader 
3. leader 将消息写入本地 log 
4. followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK 
5. leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark，最后 commit 的 offset） 并向 producer 发送 ACK

## 12. Kafka的reblance的流程？在什么情况下会发生reblance？如何避免不必要的rebalance？
### 什么是 Rebalance？
**Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 consumer 如何达成一致，来分配订阅 Topic 的每个分区**。  
例如：某 Group 下有 20 个 consumer 实例，它订阅了一个具有 100 个 partition 的 Topic 。正常情况下，kafka 会为每个 Consumer 平均的分配 5 个分区。这个分配的过程就是 Rebalance。

Rebalance 发生时，Group 下所有 consumer 实例都会协调在一起共同参与，kafka 能够保证尽量达到最公平的分配。但是 Rebalance 过程对 consumer group 会造成比较严重的影响。
**在 Rebalance 的过程中 consumer group 下的所有消费者实例都会停止工作，等待 Rebalance 过程完成**。

**reblance过程需要Group Coordinator的参与**。Group Coordinator是一个服务，**每个Broker启动的时候都会启动一个该服务。其作用是存储Group的Meta信息，并负责存储其订阅的Topic的partition对应offset信息**。

partition的offset信息的存储方式在Kafka不同版本中是不一样的:  
在0.9版本以前是存储在ZK中的，存放路径是consumers/{group}/offsets/{topic}/{partition}，其中ZK不适合频繁的写操作。  
在以后的版本中将Partition的Offset信息记录到Kafka内置Topic中，**Topic为__consumer_offsets**

reblance主要分为两个操作，**加入组(join group)和组信息同步(sync group)**。
#### 加入组(join group)
这一步主要是该Group的所有成员向其Group Coordinator发送JoinGroup请求，请求加入消费者组。一旦所有成员都发送了JoinGroup请求，Coordinator就会从所有消费者组成员中选取一个作为leader，并把组成员信息和订阅信息也发给leader。
![joinGroup](https://upload-images.jianshu.io/upload_images/3951014-30b27a129a67480a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/870/format/webp)  

#### 组信息同步(sync group)
这一步主要是leader分配消费方案,完成分配后，会把分配方案封装syncGroup请求中发送给Coordinator，其中非leader也会发送syncGroup请求给Coordinator，只是请求信息为空，
Coordinator接收到syncGroup请求中的分配方案后，会把方案作为syncGroup的响应信息发送给各个成员。这样每个组成员都知道自己该消费那些分区了。
![syncGroup](https://upload-images.jianshu.io/upload_images/3951014-63f8fcfbf894e17e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/885/format/webp)

总流程：
![rebalance](https://img2018.cnblogs.com/blog/900860/201907/900860-20190718202355371-1416685852.png)  

### Rebalance 场景分析
触发 Rebalance 的时机？    
1. **组成员发生变更**。同一个 consumer group 内新增了消费者；消费者离开当前所属的 consumer group，比如主动停机或者宕机
2. **组订阅topic数发生变更**。比如使用基于正则表达式的订阅，当匹配正则表达式的新topic被创建时则会触发rebalance
3. **组订阅topic的分区数发生变更**。比如使用命令行脚本增加了订阅topic的分区数。

* 新成员加入组
![rebalance1](https://upload-images.jianshu.io/upload_images/3951014-521275e040718fe3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/925/format/webp)  

* 组成员“崩溃”  
组成员崩溃和组成员主动离开是两个不同的场景。因为在崩溃时成员并不会主动地告知coordinator此事，coordinator有可能需要一个完整的session.timeout周期(心跳周期)才能检测到这种崩溃，
这必然会造成consumer的滞后。可以说**离开组是主动地发起rebalance,而崩溃则是被动地发起rebalance**。  
![rebalance2](https://upload-images.jianshu.io/upload_images/3951014-a3965eb26a2f2359.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900/format/webp)  

* 组成员主动离开组
![rebalance3](https://upload-images.jianshu.io/upload_images/3951014-82ead821d41224e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/915/format/webp)  

### 如何避免不必要的rebalance？
除了消费者成员正常的添加和停止之外，还有些情况下Coordinator会错误的认为消费者组成员已停止而将其踢出组以致发生reblance。在描述会发生上述误reblance之前，先解释下consumer端的几个参数：
* session.timeout.ms	用来控制最大多长时间向coordinator发送自己活着的心跳不会被认为超时，默认值1o秒
* heartbeat.interval.ms	用来控制发送心跳请求频率，值越小，发送心跳频率会越高
* max.poll.interval.ms	限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。默认值是 5分钟

通过上述三个参数可知，引起误reblance的有以下两种情况：  
> 超过session.timeout.ms**没有及时发送心跳信息，导致组成员被踢出组**。  
> **消费时间过长，超过max.poll.interval.ms还没有消费完本次poll的所有消息，导致 Consumer 主动发起 离开组 的请求**。

通过上面的分析，我们可以看一下那些rebalance是可以避免的：  
**第一类非必要 Rebalance 是因为未能及时发送心跳，导致 Consumer 被 “踢出”Group 而引发的。这种情况下我们可以设置 session.timeout.ms 和 heartbeat.interval.ms 的值**，来尽量避免rebalance的出现。（以下的配置是在网上找到的最佳实践，暂时还没测试过）  
设置 session.timeout.ms = 6s。  
设置 heartbeat.interval.ms = 2s。  
要保证 Consumer 实例在被判定为 “dead” 之前，能够发送至少 3 轮的心跳请求，即 session.timeout.ms >= 3 * heartbeat.interval.ms。  
将 session.timeout.ms 设置成 6s 主要是为了让 Coordinator 能够更快地定位已经挂掉的 Consumer，早日把它们踢出 Group。

**第二类非必要 Rebalance 是 Consumer 消费时间过长导致的。此时，max.poll.interval.ms 参数值的设置显得尤为关键**。如果要避免非预期的 Rebalance，你最好将该参数值设置得大一点，
比你的下游最大处理时间稍长一点。总之，要为业务处理逻辑留下充足的时间。这样，Consumer 就不会因为处理这些消息的时间太长而引发 Rebalance 。

参考文件：  
https://www.cnblogs.com/yoke/p/11405397.html  
https://www.jianshu.com/p/c2f808af2447  
https://www.cnblogs.com/lccsblog/p/11209341.html  

