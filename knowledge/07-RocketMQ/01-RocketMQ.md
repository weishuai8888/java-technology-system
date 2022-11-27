# RocketMQ 
## RocketMQ 的使用场景
- 应用解耦
- 流量削峰
- 流量限制
- 数据分发

## RocketMQ的角色介绍
- Producer：消息的发送者
- Consumer：消息的接收者
- Broker：暂存和传输消息
- NameServer：管理Broker
- Topic：区分消息的种类
    - 一个发送者可以发送消息给一个或多个Topic；一个接收者可以订阅一个或多个Topic消息；
- Message Queue：相当于Topic的分区，用于并行发送或接收消息
- NameServer是一个几乎无状态的节点，可集群部署，节点之间无任何信息同步；
- Broker部署相对复杂，Broker分为master和Slave，一个master对应多个slave，但一个slave只能对应一个master。master和slave的对应关系通过指定**相同的BrokerName，不同的BrokerId**来定义。BrokerId为0表示master，非0表示slave。master也可以部署多个。每个broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer；
  注意：当前RockerMQ 版本在部署架构上支持一Master多Slave，但只有**BrokerId = 1的从服务器才会参与消息的读负载**；
- Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期与NameServer获取Topic路由信息，并向提供Topic服务的的master建立长连接且定时向master发送心跳。Producer完全无状态，可集群部署；
- Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的master、slave建立长连接，且定时向mater、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息。消费者在向master拉取消息时，master 服务器会根据拉取偏移量和最大偏移量的距离判断是否读老消息，产生读IO，以及服务器是否可读等因素建议下一次是从master还是Slave拉取消息；

## RocketMQ 的执行流程
1. 启动NameServer，NameServer起来后监听端口等待Broker、Producer、Consumer连上来，相当于一个路由控制中心；
2. Broker启动，跟所有NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息（IP+端口等）以及存储所有Topic信息。注册成功后，NameServer集群就有Topic和Broker的映射关系；
3. 收发消息前先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic；
4. Producer发送消息，启动时先跟NameServer其中一台建立长连接，并从NameServer中获取当前发送的Topic存放在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的broker建立长连接，从而向Broker发送消息；
5. Consumer与Producer类似，跟其中一台NameServer建立长连接，获取当前订阅的Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息；

## RocketMQ的特性
1. 订阅与发布
    - 消息的发布是指某个生产者向某个topic发送消息；
    - 消息的订阅是指某个消费者关注了某个topic中带有某些**tag**的消息
2. 消息有序
    - 消息有序是指一类消息消费时，能按照发送的顺序来消费。RocketMQ可以严格的保证消息有序；
3. 消息过滤
    - RockerMQ的消费者可以**根据Tag进行消息过滤**，也支持自定义属性过滤。消息过滤目前是在broker端实现的；
    - 优点是减少了对Consumer无用消息的网络传输；
    - 缺点是增加了broker的负担，而且实现相对复杂；
4. 消息可靠性
    - RocketMQ支持消息的高可靠。影响消息可靠性的几种情况：
        1. Broker非正常关闭；
        2. Broker异常Crash；
        3. OS Crash;
        4. 机器掉电，但是能立即恢复供电的情况；
        5. 机器无法开机（硬件故障）；
        6. 磁盘设备损坏
    - 1、2、3、4种情况都属于硬件资源可立即恢复情况，Rocker MQ在这四种情况下能保证消息不丢，或者丢少量的消息（依赖刷盘方式是同步还是异步）；
    - 5、6种情况属于单点故障，且无法恢复，一旦发生在此单点上的消息全部丢失；Rocker MQ在这两种情况下，采用异步复制，可保证99%的消息不丢，但仍然会有极少量的消息可能丢失；通过**同步双写技术可完全避免单点**，但势必会影响性能。适合对消息可靠性要求极高的场合，例如money相关应用。注：RocketMQ从3.0版本开始支持同步双写。
5. 每个消息至少投递一次
    - 至少一次（At Least Once）指每个消息必须投递一次。Consumer先Pull消息到本地，消费完成后，才向服务器返回ACK，如果没有消费一定不会ACK消息。所以RocketMQ可以很好的支持此特性。
6. 回溯消费 \
   回溯消费是指consumer已经消费成功的消息，由于业务需求需要重新消费，Broker向Consumer投递成功消息后，消息仍然要保留，重新消费一般是按照时间维度。RockerMQ支持按照时间回溯消费，时间维度精确到秒。
7. 事务消息
    - RocketMQ的事务消息（Transactional Message）是指应用本地事务和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败；
    - Rocket MQ的事务消息提供类似X/Open XA的分布式事务功能，通过事务消息能达到分布式事务的最终一致性；
8. 定时消息
    - 定时消息（延迟队列）是指消息发送到Broker后，不会被立即消费，等待特定时间投递给真正的Topic；
    - Broker有配置项messageDelayLevel，默认值有：1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 10m, 20m, 30m, 1h, 2h。18个Level；
    - messageDelayLevel是Broker的属性，不属于某个Topic。发消息时设置delayLevel等级即可：msg.setDelayLevel(level)。 \
      Level有以下三种：
        - level == 0：消息为非延迟消息；
        - 1 <= level <=maxLevel：消息延迟特定时间。例如level == 1, 消息延迟1秒；
        - level > maxLevel：则level == maxLevel。例如level == 30，消息延迟2h \
          定时消息回暂存在名为`SCHEDULE_TOPIC_XX`的topic中，并根据delayTimeLevel存入特定的queue，`queueId = delayTimeLeval - 1`，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker会调度的消费`SCHEDULE_TOPIC_XX`，将消息写入真实的Topic。 \
          需要注意的是，定时消息需要在第一次写入和调度写入真实topic时都会计数，因此发送数量和TPS都会变高。
9. 消息重试 \
   Consumer消费消息失败后，要提供一种重试机制，令消息再消费一次；
    - 由于消息本身原因，如反序列化失败，消息数据本身无法处理等。这种错误通常要跳过这条消息，再消费其他消息。而这条失败的消息即使立即重试消费也大概率会失败。所以最好提供一种定时重试机制，即过10秒后再重试；
    - 由于依赖的下游服务不可用，如DB连接不可用，外网网络不可达等。遇到这种错误，即使跳过当前失败的消息，消费其他消息同样报错。这种情况建议sleep 30秒，再消费下一条消息，可以减轻Broker重试消息的压力；
10. 消息重投
    生产者在发送消息时 \
    - 同步消息失败会重投
    - 异步消息有重试
    - oneway没有任何保证 \
      消息重投保证消息尽可能发送成功、不丢失，但可能会造成重复消费。**消息重复在RocketMQ中是无法避免的问题**。消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会是大概率事件。另外消息主动重发、consumer负载变化也会导致重复消息；
11. 流量控制 \
    生产者流控：因为broker处理能力达到瓶颈；消费者流控：因为消费能力达到瓶颈；**生产者流控不会尝试消息重投**。
    1. 生产者流控
        - commitLog文件被锁时间超过`osPageCacheBusyTimeOutMills`时，参数默认为1000毫秒，发生流控；
        - 如果开启`transientStorePoolEnable = true`，且broker为异步刷盘的主机，且`transientStorePool`中资源不足，拒绝当前send请求，发生流控；
        - broker每隔100ms检查send请求队列头部请求的等待时间，如果超过`waitTimeMillsInSendQueue`，默认200ms,拒绝当前send请求，发生流控；
        - broker通过send拒绝请求方式，实现流控。
    2. 消费者流控
        - 消费者本地缓存消息数超过`pullThresholdForQueue`时，默认1000；
        - 消费者本地缓存消息大小超过`pullThresholdSizeForQueue`时，默认为100M；
        - 消费者本地缓存消息跨度超过`consumeConcurrentlyMaxSpan`时，默认2000；
        - 消费者流控的结果是降低拉取频率；
12. 死信队列
    - 死信队列用于处理无法被正常消费的消息；\
      当一条消息初次被消费失败，消息队列会自动进行消息重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正常的消费该消息。此时消息队列不会立刻丢弃该消息，而是将其发送到该消费者对应的特殊队列中。RocketMQ将这种正常情况下无法被消费的消息称为死信消息（Dead-Letter Message）,将存储死信消息的特殊队列称为死信队列（Dead-Letter Queue）。
    - 在RocketMQ中，可以通过使用console控制台对死信队列中的消息进行重发来使得消费者实例再次进行消费。

## RocketMQ的消费模式 push or pull
RocketMQ消息订阅有两种模式，一种是Push模式（MQ PushConsumer），即MQServer主动向消费端推送；一种是Pull模式(MQPullConsumer)，即当消费端在需要时，主动到MQ Server拉取。但在具体实现时，**本质都是采取消费端主动拉取的方式**，即Consumer轮训Broker拉取消息。
- Push模式特点 \
优点是实时性高，缺点在于消费端的处理能力有限，当瞬间推送很多消息给消费端时，容易造成消费端消息积压，严重时会压垮客户端；其本质上是对Pull模式的一种封装，其实先是消息拉取线程在服务器拉取到一批消息，然后提交到消息消费线程池后，又继续向服务器再次尝试拉取消息。
- Pull模式特点 \
优点就是主动权掌握在消费端自己手中，根据自己的处理能力量力而行。缺点是如何控制Pull的频率，时间间隔太久担心影响时效性，间隔太短担心做太多“无用功”浪费资源。**比较折中的办法就是长轮训**。
- Push模式和Pull模式的区别
    - Push方式里，consumer把长轮训的动作封装了，并注册MessageListener监听器，取到信息后，唤醒MessageListener的consumerMessage()来消费，对用户而言感觉消息是被推过来的；
    - Pull方式里，取消息的过程需要用户自己主动调用。首先通过打算消费的Topic拿到message Queue的集合，遍历message Queue集合，然后针对每个Message Queue批量取消息。一次取完后，记录该队列下一次要取的开始offset，直到取完了，再换另一个message Queue。

  **Rocket MQ采用长轮训机制来模拟Push效果，算是兼顾了二者的优点**
### RocketMQ中的角色及相关术语
1. 消息模型 - Message Model \
RocketMQ主要由Producer、Broker、Consumer三部分组成。Producer负责产生消息，Consumer负责消费消息，Broker负责存储消息。Broker在实际部署的过程中对应一台服务器，每个Broker可以存储多个Topic消息，每个Topic消息也可以分片存储在不同的Broker。Message Queue负责存储消息的物理地址，每个Topic中的消息地址存储于多个Message Queue中。Consumer Group由多个Consumer组成。
2. Producer \
消息生产者，负责产生消息。一般由业务系统负责产生消息。
3. Consumer \
消息消费者，负责消费消息。一般由后台系统负责异步消费。
4. Push Consumer \
Consumer消费的一种类型，该模式下Broker收到数据后会主动推给消费端。应用通常向Consumer对像注册一个Listener接口，一旦收到消息，Consumer对象立刻回调Listener接口方法。该消费模式一般实时性较高。
5. Pull Consumer \
Consumer消费的一种类型，应用通常主动调用Consumer的拉消息方法从Broker服务器拉消息，主动权由应用控制。一旦获取了批量消息，应用就会启动消费过程。
6. Producer Group \
同一类Producer的集合，这类Producer发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则Broker会联系同一生产者组的其他生产者实例以提交或回溯消费。
7. Consumer Group \
同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。需要注意的是，**消费者组的消费者实例必须订阅完全相同的Topic。**Rocket MQ支持两种消息模式：集群消费（Clusting）和广播消费（Broadcasting）。
8. Broker \
消息中转角色，负责存储消息，转发消息，一般也成为Server。在JMS规范中称为Provider.
9. 广播消费 \
一条消息被多个Consumer消费，即使这些Consumer属于同一个ConsumerGroup，消息也会被Consumer Group中的每个Consumer都消费一次，广播消费中的Consumer Group概念可以认为在消息划分方面无意义。

在CORAB Notification规范中，消费方式都属于广播消费；

在JMS规范中，相当于JMS Topic模型（public/subscribe）

10. 集群消费 \
一个Consumer Group中的Consumer实例平均分摊消费消息。例如某个Topic有9个消息，其中一个Consumer Group有3个实例，那每个实例只能消费其中3个消息。
11. 顺序消息 \
消费消息的顺序要和发送消息的顺序一致，在RocketMQ中主要指的是局部顺序。即一类消息为满足顺序性，必须Producer单线程顺序发送，且发送到同一个队列，这样Consumer就可以按照Consumer发送的顺序去消费消息。
12. 普通顺序消息 \
正常情况下，可以保证完全的顺序消费。但是一旦发生通信异常，Broker重启，由于队列总数发生变化，哈希取模后定位的队列会变化，产生短暂的消息顺序不一致。如果业务能容忍在集群异常情况下（如某个Broker宕机或重启），消息短暂的乱序，使用普通顺序方式比较合适。
13. 严格顺序消息 \
无论正常或异常下都能保证顺序消费。但是牺牲了分布式Failover特性，即只要Broker集群中只要一台不可用，整个集群机器都不可用，服务可用性大大降低。如果服务器部署为同步双写模式，此缺陷可以通过备机自动切换为主机来避免，不过仍然会存在几分钟的服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前还未实现）

目前已知的应用只有数据库binlog同步依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序。**推荐使用普通顺序消息。**
14. Message Queue \
在RecketMQ 中，所有消息队列都是持久化的、长度无限的数据结构。所谓长度无限是指队列中的某个存储单元都定长，访问其中的存储单元使用`offset`来访问，offset为Java的`long类型、64位`，理论上在100年内不会溢出，所以认为是长度无限。另外队列中只保存最近几天的数据，之前的数据都会按照过期时间来删除。也可以认为Message Queue时一个长度无限的数组，offset就是下标。
15. 标签 - Tag \
为消息设置的标志，用于同一主题下区分不同类型的消息。**来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。**消费者可以根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性。

## Rocket MQ的高级特性
### 消息发送
- 提高消息发送速度的方式
    - 采用OneWay方式发送 \
      OneWay方式只发送请求不等待应答。即将数据写入客户端的socket缓冲区就返回，不等待对方返回结果。用这种方式发送消息的耗时可缩短到微秒级；
    - 增加Producer的并发量 \
      使用多个Producer同时发送。Rocket MQ引入了一个并发窗口，在窗口内消息可以并发的写入DirectMem中，然后异步的将连续一段无空洞的数据刷入文件系统中。`顺序写`CommitLog可让RocketMQ 无论在HDD还是在SSD磁盘情况下都能保持较高的写入性能；
### 消息消费
- 提高Consumer处理能力的方式
    - 当Consumer处理速度跟不上消息的产生速度时，会造成越来越多的消息积压，这个时候首先查看消息逻辑本身有没有优化空间；
    - 提高Consumer并行度 \
      同一个Consumer Group（Clustering方式）下，可通过增加Consumer实例的数量来提高并行度。增加机器或者在已有机器中启动多个Consumer进程都可以增加Consumer实例数； \
      **注：**总的Consumer数量不要超过Topic下的`Read Queue`数量，超过的Consumer实例接收不到消息；通过提高单个Consumer实例中的并行处理线程数，可以在同一个Consumer内增加并行度来提高吞吐量。（设置方法是修改consumeThreadMin和consumeThreadMax）
    - 以批量的方式进行消费 \
      多条消息同时处理的消费方式。实现方法是设置Consumer的consumeMessageBatchMaxSize这个参数，默认是1，如果设置为N，在消息多的时候每次收到的是个长度为N的**消息链表**。
    - 检测延时情况，跳过非重要消息 \
      消息堆积严重时，可选择丢弃不重要的消息。
### 消息存储
1. 消息存储介质
    - 关系行数据库：ActiveMQ采用此方式
    - 文件系统：RocketMQ、Kafka、Rabbit MQ采用此方式
2. 消息存储介质性能对比 \
   文件系统 > 关系型数据库
3. 消息的存储和发送
    1. 消息存储：RocketMQ采用顺序写，保证了消息的存储速度；
    2. 存储结构 \
       RocketMQ的消息存储是有Consume Queue和 commitLog配合完成的。Topic下的每个Message Queue都对应着一个Consume Queue；
        - Commit Log是消息的物理存储文件；消息主要是顺序写入日志文件，当文件满了就写入下一个日志文件；
        - Consume Queue消息的逻辑队列，存储指向物理存储的地址，用来提高消息的消费性能。他保存了消息Tag的hashCode、消息大小Size和指定Topic下的队列消息在CommitLog中的起始物理偏移量offset。Consumer根据topic的hashCode 在Consume Queue中找到带消费的消息。
        - IndexFile：提供了一种通过key或时间区间来查询消息的方法。IndexFile的底层存储设计为在文件系统中实现HashMap结构，故RocketMQ的索引文件其底层实现是Hash索引。
### 过滤消息 \
RocketMQ分布式消息队列的消息过滤方式有别于其他MQ中间件，是在Consumer端订阅消息时再做消息过滤的。所支持的过滤方式：
- Tag过滤方式 \
  在订阅消息时指定Tag，如果一个消息有多个Tag可用`||`隔开；
- SQL92过滤方式 \
  仅对push的消费者起作用；RocketMQ使用Bloom Filter避免了每次都去执行sql;
- Filter Server过滤方式 \
  用户自定义Java函数，根据Java函数的逻辑对消息进行过滤。这种过滤方式会占用很多Broker机器的CPU资源，要根据实际情况谨慎使用；上传的Java代码也要经过检查，不能有申请大内存、创建线程等这样的操作，否则容易造成Brocker服务器宕机。
### 零拷贝 - Zero Copy
- 从内存角度看，数据在内存中没有发生过拷贝，只是在内存和IO设备间传输；
- 虽然叫零拷贝，实际上sendFile是有2次数据拷贝。第1次是从磁盘拷贝到内核缓冲区，第2次是从内核缓冲区拷贝到网卡（协议引擎）。
### Broker的复制 \
如果broker有master和slave，消息需要从master复制到slave上，有同步和异步两种复制方式。
- 主从同步复制 \
  等master数据复制到slave上以后才反馈给客户端写成功状态；在同步复制下，如果master出故障，Slave上有全部的备份数据，容易恢复。但是同步复制会增大数据写入延迟，降低系统吞吐量；
- 主从异步复制 \
  只要master写成功即可反馈给客户端写成功状态；在异步复制下，系统拥有较低的延迟和较高的吞吐量，但是如果master出故障有些数据因为没有写入到Slave，就有可能丢失；
- Dledger复制 \
  Dledger在写入消息的时候，要求消息至少复制到半数以上的节点之后，才给客户端返回成功，并且它是支持通过选举来动态切换主节点的。
    - 选举过程中不能提供服务；
    - 至少需要3个节点才能保证数据一致性，资源利用率较低；
    - 由于至少要复制到半数以上才能返回写入成功，所以性能不如主从异步复制方式；
### RocketMQ的高可用 
RocketMQ分布式集群是通过master和slave的配合达到高可用的。
- Master和Slave的区别：
    - 在Broker的配置文件中，参数brokerId的值为0表明这个Broker时master，大于0表明这个Broker是slave。brokerRole参数也说明这个Broker是master和slave；
    - master角色的Broker支持读和写，Slave角色的Broker只支持读
- 消息消费高可用 \
  在consumer配置文件中，并不需要设置是从master还是从slave读，当master不可用或者繁忙的时候，Consumer会被**自动切换**到slave读。
- 消息发送高可用 \
  在创建Topic的时候，把Topic的多个Message Queue创建在多个Broker组上（相同Broker名称，不同brokerId的机器组成一个组）。当一个Broker组的master不可用后，其他组的master仍然可用，Producer依然可以发送消息。这样既可以在性能方面具有扩展性，也可以降低主节点故障对整体上带来的影响。 \
  **RockerMQ目前不支持自动把slave 转变为master。需要手动转变。**
### 刷盘机制
Rocket MQ的所有消息都是持久化的，先写入PageCache，然后刷盘，可以保证内存和磁盘都有一份数据，访问时直接从内存读取。消息通过Producer写入RocketMQ的时候，有两种写磁盘方式，分布式同步刷盘和异步刷盘。同步刷盘和异步刷盘的区别在于写完PageCache后，异步刷盘直接返回，同步刷盘需要等待刷盘完成才返回。
- 同步刷盘
    - 写入pageCache后，线程等待，通知刷盘线程刷盘；
    - 刷盘完成后，唤醒前端等待线程，可能是一匹线程；
    - 前端等待线程向用户返回成功
- 异步刷盘 \
  异步刷盘内存溢出问题不会出现。
### 负载均衡
RocketMQ的负载均衡均在Client端完成，主要分为Producer端发送消息时的负载均衡和Consumer端订阅消息时的负载均衡。
### 消息重试
1. 顺序消息重试 \
   当消费者消费消息失败后，消息队列RocketMQ会自动不断进行消息重试（每次间隔时间为1S），这时应用会出现消息消费被阻塞的情况。因此，在使用顺序消息时务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象发生。
2. 无需消息重试 \
   对于无需消息，当消费者消费消息失败时，您可以通过设置返回状态达到消息重试的结果。**无序消息的重试只对集群消费方式有效**；广播方式不提供失败重试特性，即消费失败后，失败消息不再重试，继续消费新消息。
    - 重试次数 \
      如果重试16次后仍然失败，消息将不再投递。一条消息在第一次消费失败的前提下，将会在接下来的4小时46分钟内进行16次消息重试，超过这个时间范围的消息将不再重试投递。\
      **注意：** 一条消息无论重试多少次，这些重试消息的 Message ID 不会改变。
    - 配置方式
        - 消息失败后，重试配置放方式
            - 返回 ConsumeConcurrentlyStatus.RECONSUME_LATER; （推荐）
            - 返回 Null
            - 抛出异常
        - 消费失败后，不重试配置方式 \
          集群消费方式下，消息失败后期望消息不重试，需要捕获消费逻辑中可能抛出的异常，最终返回ConsumeConcurrentlyStatus.CONSUME_SUCCESS，此后这条消息将不会再重试。
        - 自定义消息最大重试次数 \
          消息队列 RocketMQ 允许 Consumer 启动的时候设置最大重试次数，重试时间间隔将按照如下策略：
            - 最大重试次数小于等于 16 次，则重试时间间隔同下表描述。
            - 最大重试次数大于 16 次，超过 16 次的重试时间间隔均为每次 2 小时。
          **注意：** \
            1. **消息最大重试次数的设置对相同 Group ID 下的所有 Consumer** **实例有效**。
            2. 如果只对相同 Group ID 下两个 Consumer 实例中的其中一个设置了MaxReconsumeTimes，那么该配置对两个 Consumer 实例均生效。
            3. 配置采用覆盖的方式生效，即**最后启动的** **Consumer** **实例会覆盖之前的启动实例的配置**
        - 获取消息重试次数
        - 
      | 重试次数 | 重试间隔时间 | 重试次数 | 重试间隔时间 |
      | -------- | ------------ | -------- | ------------ |
      | 1        | 10秒         | 9        | 7分          |
      | 2        | 30秒         | 10       | 8分          |
      | 3        | 1分          | 11       | 9分          |
      | 4        | 2分          | 12       | 10分         |
      | 5        | 3分          | 13       | 20分         |
      | 6        | 4分          | 14       | 30分         |
      | 7        | 5分          | 15       | 1小时        |
      | 8        | 6分          | 16       | 2小时        |

### 死信队列
Rocket MQ中消息重试超过设置值后，就会被放入死信队列（Dead-Letter Queue），这种消息被称为死信消息（Dead-Letter Message，DLM）。
- 死信消息-DLM特性
    - 不会再被消费者正常消费；
    - 有效期于正常消息相同，均为3天，3天后自动删除。因此，死信消息应在产生后的3天内及时处理；
    - 可在消息队列MQ控制台重新发送该消息，让消费者重新消费一次；
- 死信队列-DLQ特性
    - 一个死信队列对应一个Group ID，而不是对应单个消费者实例。；
    - 如果一个Group ID未产生死信消息，消息队列RocketMQ不会为其创建相应的死信队列；
    - 一个死信队列包含了对应Group ID产生的所有死信消息，不论该消息属于哪个topic；
### 延迟消息 - 定时消息
见Rocket MQ的特性 --> 定时消息
### 顺序消息
见RocketMQ中的角色及其术语 --> 顺序消息
### 事务消息
- 具体流程
    1. 发送方向MQ发送“待确认”消息；
    2. RocketMQ将收到的“待确认”消息持久化成功后，向发送方回复消息已经发送成功，此时第一阶段消息发送完成。
    3. 发送方开始执行本地事件逻辑。
    4. 发送方根据本地事件执行结果向RocketMQ发送二次确认（Commit或是Rollback）消息，RocketMQ收到Commit状态则将第一阶段消息标记为可投递，订阅方将能够收到该消息；收到Rollback状态则删除第一阶段的消息，订阅方接收不到该消息。
    5. 如果出现异常情况，步骤4提交的二次确认最终未到达RocketMQ，服务器在经过固定时间段后将对“待确认”消息发起回查请求。
    6. 发送方收到消息回查请求后（如果发送一阶段消息的Producer不能工作，回查请求将被发送到和Producer在同一个Group里的其他Producer），通过检查对应消息的本地事件执行结果返回Commit或Roolback状态。
    7. RocketMQ收到回查请求后，按照步骤4的逻辑处理。
### 消息的查询
”先尝后买“：尝 ==> 消息的查询；买 ==> 消息的消费
- 按照Message Id查询消息
- 按照Message Key查询消息
### 消息的优先级
RocketMQ是一个先入先出的队列，不支持消息级别或者Topic级别的优先级。
1. 多个不同的消息类型使用同一个topic时，由于某一个种消息流量非常大，导致其他类型的消息无法及时消费，造成不公平，所以把流量大的类型消息在一个单独的 Topic，其他类型消息在另外一个Topic。应用程序创建两个 Consumer，分别订阅不同的 Topic，这样就可以了。
2. 情况和第一种情况类似。
    ```markdown
    举个实际应用场景: 
    	一个订单处理系统，接收从 100家快递门店过来的请求，把这些请求通过 Producer 写入RocketMQ；订单处理程序通过Consumer 从队列里读取消 息并处理，每天最多处理 1 万单 。 如果这 100 个快递门店中某几个门店订单量 大增，比如门店一接了个大客户，一个上午就发出 2万单消息请求，这样其他 的 99 家门店可能被迫等待门店一的 2 万单处理完，也就是两天后订单才能被处 理，显然很不公平 。 
    
    	这时可以创建 一 个 Topic， 设置 Topic 的 MessageQueue 数 量 超过 100 个，Producer根据订单的门店号，把每个门店的订单写人 一 个 MessageQueue。 DefaultMQPushConsumer默认是采用循环的方式逐个读取一个 Topic 的所有 MessageQueue，这样如果某家门店订单量大增，这家门店对应的 MessageQueue 消息数增多，等待时间增长，但不会造成其他家门店等待时间增长DefaultMQPushConsumer 默认的 pullBatchSize 是 32，也就是每次从某个 MessageQueue 读取消息的时候，最多可以读 32 个 。 在上面的场景中，为了更 加公平，可以把 pullBatchSize 设置成1。
    ```
3. 强制优先级
    ```markdown
    	TypeA、 TypeB、 TypeC 三类消息 。 TypeA 处于第一优先级，要确保只要有TypeA消息，必须优先处理; TypeB处于第二优先 级; TypeC 处于第三优先级 。 
    	对这种要求，或者逻辑更复杂的要求，就要用 户自己编码实现优先级控制.
    		如果上述的 三 类消息在一个 Topic 里，可以使 用 PullConsumer，自主控制 MessageQueue 的遍历，以及消息的读取；
    		如果上述三类消息在三个 Topic下，需要启动三个Consumer， 实现逻辑控制三个 Consumer 的消费.
    ```
### 底层网络通信 - Netty
RocketMQ支持的通信方式主要有同步（sync）、异步（async）、单向（oneway）三种。“单向”通信模式相对简单，一般用在发送心跳包场景下，无需关注其response \
RocketMQ 消费端我们可以设置最大消费线程数，每次拉取消息条数；同时Push Consumer会判断获取但还未处理的消息个数、消息总大小、Offset的跨度，任何一个值超过设定的大小就隔一段时间再拉取消息，从而达到流量控制的目的。

## RocketMQ高级实战
### 生产者
1. tags的使用 \
   一个应用尽可能用一个Topic，而消息子类型则可以用tags来标识。tags可以由应用自行设置，只有生产者在发送消息设置了tags，消费方在订阅消息时才可以利用tags通过broker做消息过滤。`message.setTags("TagA")`
2. key的使用 \
   每个消息在业务层面的唯一标识码要设置到key字段，方便将来定位消息丢失问题。服务器会为每条消息创建哈希索引，应用可以通过Topic、key来查询这条内容，以及消息被谁消费。由于是哈希索引，务必保证key尽可能唯一，这样避免潜在的哈希冲突。
    ```java
    // 订单Id
    String orderId = "20034568923546";
    message.setKeys(orderId);
    ```
3. 日志的打印 \
   消息发送后务必要打印SendResult和key字段。seng消息方法只要不抛异常，就代表发送成功。发送成功会有多个状态，在SendResult定义。
    - SEND_OK \
      消息发送成功。虽然发送成功但不一定可靠，要确保不丢失消息，还应起动同步master服务器或同步刷盘。即`SYNC_MASTER, SYNC_FLUSH`;
    - FLUSH_DISK_TIMEOUT - 同步刷盘会出现此 \
      消息发送成功，但服务器刷盘超时。此时消息已经进入服务器队列，只有服务器宕机，消息才会丢失。
    - FLUSH_SLAVE_TIMEOUT - 同步master会出现 \
      消息发送成功，但服务器同步到slave超时。此时消息已经进入到服务器队列，只有服务器宕机，消息才会丢失。
    - SLAVE_NOT_AVAILABLE - 同步master，slave故障会出现 \
      消息发送成功，但是此时slave不可用。
4. **消息发送失败的处理方式** \
   Producer的send方法本身支持内部重试
    - 至多重试2次（同步发送为2次，异步发送为0次）；
    - 如果发送失败则轮转到下一个broker。这个方法的总耗时时间sendMsgTimeOut设置的值，默认是10S；
    - 如果本身向broker发送消息产生超时异常，就不会再重试；
5. 选择one way方式发送 \
   通常消息的发送是这样一个过程：
    - 客户端发送请求到服务器；
    - 服务器处理请求；
    - 服务器向客户端返回应答；\
      对可靠性要求不高的场合，比如日志收集类应用，可以采用oneway方式调用。**oneway方式只发送请求，不等待应答**，而发送请求在客户端实现层面仅仅是一个操作系统调用的开销，即将数据写入客户端的socket缓冲区，此过程耗时通常在微秒级。
### 消费者
1. **消费过程幂等** \
   借助关系行数据库去重。确定消息的唯一键，可以是msgId也可以是其他唯一字段。在消费之前判断唯一键是否存在，不存在则消费，存在则跳过。（考虑原子性问题）
2. 消费速度慢的处理方式
    - 提高消费并行度
    - 批量方式消费
    - 跳过非重要消息
3. 消费打印日志 \
   如果消息量较少，建议在消费入口方法打印消息、消费耗时等，方便后续排查问题。


























