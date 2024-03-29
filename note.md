# 可以看作是官方文档的~~简单翻译~~渣翻
版本：2.10.0  
始：2022-06-06  
终：  
状态：DOING
# 1. 概念与架构
## 1.1. 预览（Overview）
Pulsar是一个多租户，高性能的服务器间消息传递解决方案，最早由雅虎开发，现在Pulsar由[Apache软件基金会](https://www.apache.org)管理。  

罗列一下Pulsar的特性：
- 本地的一个Pulsar实例中支持多集群部署，集群间可以做到跨地域无缝复制消息。
- 拥有极低地发布和端到端延迟。
- 简单的客户端API，支持Java，Go，Python和C++
- 支持topic多种订阅类型（独占、共享和灾备）
- 通过[Apache BookKeeper](https://bookkeeper.apache.org)提供的消息持久化机制保证消息的传递。
- 由轻量级的serverless computing框架Pulsar Functions，实现了流原生的数据处理
- 拥有基于Pulsar Function的serverless connector框架 Pulsar IO，其能够使数据更好的迁入移除Apache Pulsar。
- 当数据老化时，通过分层存储，将数据从热存储转移到冷存储（例如S3和GCS）。
## 1.2 消息传递（Messaging）
Pulsar是基于 [发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 模式(也可以缩写为pub-sub)。在这种模式下，producers发布消息到topics中；consumers订阅这些topic，处理传入的消息，并且当处理消息成功地结束时发送一个ack给broker。  

当一个订阅被创建时，Pulsar会保留所有的消息，即使consumer断开链接。只有当某一个消费者成功处理完毕这些消息，发送了ack后，这些被保留下来的消息才会被丢弃。  

如果一个消息消费失败，并且你希望这个消息能够被再次消费，你可以启用消息重新传递机制来要求broker重新发送这些消息。
### 1.2.1 消息（Message）
消息（Message）是Pulsar的基本“单位”。下表列出了消息包含的一些组成信息。  
|组成（Component）|描述（Description）|
|:---:|:---|
|Value/data payload|消息携带数据。所有Pulsar消息都包含原始字节，即使消息也可以符合数据模式。|
|Key|消息可以随意地被key所标记，这对一些事情十分有用，比如topic压缩。|
|Properties|用户自定义的键值对（可选）。|
|Producer name|标记着生产这个消息的生产者名称，如果未设置生产者名称，将会使用默认生产者名称。|
|Topic name|标记着这个消息会被发往哪个topic中。|
|Schema version|标记着生产消息时使用的schema版本号。|
|Sequence ID|在topic中，每个Pulsar消息都属于一个有序序列，消息的序列ID可以由producer初始化，来指明其在序列中的顺序，也可以自定义。</br>序列ID可以用来消除重复的消息。如果 `brokerDeuplicationEnabled`被设置为`true`的话，那么每个消息的序列ID在每个topic（非分区）或者一个分区中唯一。|
|Message ID|消息的消息ID会在被bookies持久化时分配。消息ID指明了消息在ledger中的特殊位置，以及其在Pulsar集群中是唯一的。|
|Publish time|一个消息被发布时的时间戳，时间戳会自动由producer赋值。|
|Event time|一个由应用程序赋值给消息的可选时间戳。比如，应用程序可以选择在这个消息被处理时，给这个属性赋上一个时间。如果没有设置event time，它的值为`0`|

消息的最大默认大小为 5MB 。你可以在配置中设置消息的最大大小。
- 在`broker.conf`文件中
```
# message的最大长度（byte）
maxMessageSize=5242880
```
- 在`bookkeeper.conf`文件中
```
# netty的最大大小（byte），接收到的任何大于此值的消息都将被拒绝。默认值为5MB。
nettyMaxFrameSizeBytes=5253120
```
对于更多的Pulsar消息的信息，可以查看Pulsar binary protocol，
### 1.2.2 生产者（Producers）
生产者是一个与topic建立连接，并且可以把消息发布到Pulsar broker上的进程。Pulsar broker将会处理这些消息。
#### 1.2.2.1 发送模式（Send Modes）
生产者可以选择同步发送（sync）或者异步发送（async）
|模式（Mode）|描述（Description）|
|:---------:|:-----------------|
|同步发送（Sync）|producer每次发送消息后都会等待broker返回ack。如果producer没有收到ack，会将此次发送视为失败。|
|异步发送（Async）|producer会将消息放入阻塞队列中并且马上返回。客户端在后台将消息发送给broker。如果队列满了（可以在配置中设置最大size），当调用API时producer会被阻塞或者立马失败，这取决于传递给producer的参数。|
#### 1.2.2.2 访问模式（Access Mode）
producer对于topic可以有不同的访问模式
|访问模式（Access mode）|描述（Description）|
|:--------------------:|:-----------------|
|`共享（shared）`|多个producers可以向同一个topic进行消息发布。</br></br>这是**默认**的设置。|
|`独占（Exclusive）`|一个topic只能有一个producer进行消息发布。</br></br>如果已经有一个producer连接了该topic，其他producers尝试往这个topic上发布消息时会立马提示错误。</br></br>当“旧”producer与broker发生网络分区时，“旧”producer会被剔除，“新”producer会被选为下一个独占对象。|
|`等待独占（WaitForExclusive）`|如果已经有一个producer连接了该topic，那么新producer的连接会被挂起（而不是超时），直到新producer获取到`独占（Exclusive）`访问权。</br></br>获得到独占访问权的producer被视为leader。因此，如果你想要让你的应用实现leader选举方案，你可以使用这种访问模式。|
> #### ！注意
> 一旦一个应用程序成功获取到了`独占（Exclusive）`或`等待独占（WaitForExclusive）`的访问模式，那么可以保证该topic**只会被这个应用实例写入**。任何其他producers尝试该topic生产消息时都会得到一个错误响应或者等待直到它们得到`等待独占（WaitForExclusive）`的访问模式。更多信息，请查看PIP 68: Exclusive Producer。
你可以通过Java客户端API来设置producer的访问模式，对于更多信息，可以查看ProducerBuilder.java文件中的`ProducerAccessMode`。
#### 1.2.2.3 压缩（Compression）
你可以在producer发布消息的过程中进行消息压缩。Pulsar目前支持以下几种压缩类型。
- [LZ4](https://github.com/lz4/lz4)
- [ZLIB](https://zlib.net/)
- [ZSTD](https://facebook.github.io/zstd/)
- [SNAPPY](https://google.github.io/snappy/)
#### 1.2.2.4 批量处理（Batching）
当启用批量处理时，producer会将消息积累起来，在一个request中将这些消息一起发送。批量处理的量大小取决于最大消息数和最大发布延迟。因此，积压数量是批量处理的总数，而不是消息的总数。  

在Pulsar中，批（batch）作为存储和追踪的基本单位，而不是单个消息作为存储和追踪的基本单位。Consumer将一个batch拆解为单个消息。但是，即使开启了批处理，延时消息（被配置了参数`deliverAt`或`deliverAfter`）始终会被当但一个独立的消息进行发送。  

通常情况下，当一个consumer确认了batch中的所有消息，这个batch才会被视为确认。这意味着如果**没有**将一个batch中的所有消息进行确认（如意料之外的失败、否定确认或者是确认超时），那么该batch中的所有消息将会被重新发送，即使有部分消息已经被确认过了。

为了避免重新将batch中已经被确认的消息发送给consumer，Pulsar从2.6.0版本开始引入了批量索引确认（batch index acknowledgement）。当批量索引确认启用时，consumer会过滤掉那些已经确认过的batch index，并将这些batch index发送给broker。broker维护且追踪每个batch index的ack状态以防止向consumer发送那些已被确认过的消息。只有当batch中的所有消息被确认时，batch才会被删除。  

默认情况下，批量索引确认是禁用的（`acknowledgmentAtBatchIndexLevelEnable=false`）。你可以在broker端设置参数`acknowledgmentAtBatchIndexLevelEnable`为`true`来启用它。启用批量索引确认会带来更多的内存开销。
#### 1.2.2.5 分块（Chunking）
消息分块能够使Puslar在producer端将消息进行分块，在consumer端将聚合分块消息，这样能够很好的处理大型负载消息。  

当消息分块启用时，当消息的大小超过了允许的最大载荷（即在broker处的参数配置`maxMessageSize`），消息的工作流会如下所示：
1. producer端将原始消息拆分为分块消息，并且将他们与分块元数据（metadata）单独分开，按顺序发布到broker上。
2. broker会将分块消息同其他普通消息一样，放在一个managed-ledger上，并且会使用`chunkedMessageRate`参数来记录这个topic中分块消息的速度。
3. consumer端缓存分块消息，并且当收到了一个消息的所有分块时，会将它们聚合起来，放入receiver queue中。
4. 客户端消费从receiver queue中聚合的数据。
##### 局限性
- 分块只对**持久化**的topic有用。
- 分块只对**独占**和**灾备**的订阅类型有用。
- 分块无法与**批处理（batching）** 同时启用。
#### 1.2.2.6 消费者有序处理连续的分块消息
下图显示了拥有一个producer的topic，producer向topic发送一批大的分块消息和普通的非分块消息。Producer发布消息M1，M1有三个分块M1-C1，M1-C2和M1-C3，Broker会在managed-ledger中存储这三个分块消息，并且把他们以同样的顺序传输到consumer上（consumer为独占或灾备类型）。Consumer在内存中缓存收到的分块消息，当收到所有分块消息时，会将它们聚合成一整个消息M1，然后将原始消息M1发送给客户端。
<div align="center">
  <img src="/imgs/producer/chunking-01.png"></img>
</div>

#### 1.2.2.7 消费者有序处理不连续的分块消息
当多个消费者往同一个topic发布多个分块消息时，Broker会将所有来自不同的producer的分块消息保存到同一个managed-ledger中。分块们在managed-ledger可以不连续。如下图所示，Producer1发布了消息M1并且分为了三个块M1-C1，M1-C2和M1-C3。Producer2发布了消息M2并且分为了三个块M2-C1，M2-C2以及M2-C3。所有分块消息在该消息内是连续的，但在managed-ledger内可能不是连续的。
<div align="center">
  <img src="/imgs/producer/chunking-02.png"></img>
</div>

> #### ！注意
> 在这种情况下，不连续的分块消息可能会给consumer端带来一定的内存压力，因为consumer需要为每个大消息保留一定的缓冲区来将这些分块消息合并为原来的大消息。你可以通过配置参数`maxPendingChunkedMessage`来限制consumer端并发维持的最大消息分块数。当达到阈值时，consumer将会丢弃pending的消息通过静默ack或者要求broker过一段时间重新传递，以此来优化内存利用率。

#### 1.2.2.8 启用消息分块
**必要条件**：通过设置`enableBatching`参数为`false`来禁用批处理（batching）。  

消息分块的特性默认是关闭的。如果要启用它，可以在创建producer时设置`chunkingEnabled`参数为`true`。

> #### ！注意
> 如果consumer未能在指定时间能收到一个消息的所有分块，那么这些已收到的分块会过期。默认时间时1分钟。有关`expireTimeOfIncompleteChunkedMessage`参数的更多信息，可以查看org.apache.pulsar.client.api.

### 1.2.3 消费者（Consumers）
消费者是一个通过订阅topic与其建立连接，然后接收消息的进程。  

一个consumer向broker发送流许可申请（flow permit request）来获取消息。在消费者端有一个队列，用来接收从broker端推送过来的消息。你可以设置参数`receiverQueueSize`来设置这个队列的最大值（默认大小是`1000`）。每当`consumer.receive()`被调用时，就会从缓冲区获取一条消息。
#### 1.2.3.1 接收模式（Receive modes）
消息可以从brokers处异步（async）或同步（sync）接收。
|模式（Mode）|描述（Description）|
|:---------:|:-----------------|
|同步接收|同步接收在消息之前会一直被阻塞。|
|异步接收|异步接收会立马返回一个future value，例如Java中的`CompletableFuture`，当收到消息时，这个future会被立刻完成。|
#### 1.2.3.2 监听（Listeners）
客户端为consumer提供了监听器的实现类。例如，Java客户端提供了MessageListener接口。当收到一个新消息时，该接口下的`received`方法会被调用。
#### 1.2.3.3 确认（Acknowledgement）
当consumer成功消费一个消息时，它会向broker发送一个确认。这条消息是被永久保存的，只有当所有订阅者都确认了这条消息，这条消息才会被删除。如果你希望这些消息被consumer确认后依旧保留下来，你需要配置消息保留策略。  

对于批量消息，你可以启用批量索引确认来防止已经被ack的消息重复发送给消费者。详细内容请查看批量处理（batching）。

可以通过下面两种方式之一来进行消息确认：
- 独立消息确认。使用独立消息确认，消费者会为每一条消息发送一个ack给broker。
- 累计消息确认。使用累计消息确认，消费者**只会**确认收到的最后一个消息。所有之前（包含此条）的消息都不会再被发送给这个consumer（类似tcp的ack？）

如果你想使用独立消息确认，可以使用下面的API。
```
consumer.acknowledge(msg);
```
如果你想用累计消息确认，可以使用下面的API。
```
consumer.acknowledgeCumulative(msg);
```
> #### ！注意
> 累计消息确认无法用于共享订阅类型，因为共享订阅类型涉及到多个consumers，这些consumers可以访问到同一个订阅，如果使用累计消息确认，可能会将别的consumer消费的消息给确认掉。在共享订阅类型中，消息总是独立确认。
#### 1.2.3.4 否定确认（Negative acknowledgement）
否定确认机制允许你向broker发送一个提醒，这个提醒意味着consumer没有处理消息。当一个consumer消费消息失败，并且需要重新消费它时，consumer会向broker发送一个否定确认（nack），这会让broker重新发送这条消息给consumer。  

消息可以否定地独立确认，也可以否定地累计确认，这取决于消费者使用的订阅模式。  

在独占和灾备订阅类型下，consumers只会对他们收到的最后一条消息进行否定确认。  

在共享或者Key共享的订阅类型下，consumers可以对消息进行单条的否定确认。  

请注意，对于已有排序的订阅类型（如独占、灾备和Key共享模式）进行否定确认，可能会导致失败的消息没办法像原始顺序一样发送给consumers。  

如果你打算对消息使用否定确认，请确保你的否定确认在确认超时之前发送出去。  

使用下面的API来进行否定确认。  
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                            .topic()
                            .subscriptionName("sub-negative-ack")
                            .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                            .negativeAckRedeliveryDelay(2, TimeUnit.SECONDS)// the default value is 1 min
                            .subscribe();
                       
Message<byte[]> message = consumer.receive();

// call the API to send negative acknowledgement
consumer.negativeAcknowledge(message);

message = consumer.receive();
consumer.acknowledge(message);
```
为了让那些重新传递的消息有不同的延迟，你可以通过**重传补偿机制（Negative Redelivery Backoff）** 来设置最大重试传递消息次数。使用下面的API来启用`Negative Redelivery Backoff`。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                            .topic(topic)
                            .subscriptionName("sub-negative-ack")
                            .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                            .negativeAckRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
                                                          .minDelayMs(1000)
                                                          .maxDelayMs(60 * 1000)
                                                          .build())
                            .subscribe();
```
消息的重传行为会像下表一样。
|重传次数|重传延迟|
|:------|:------|
|1|10 + 1 seconds|
|2|10 + 2 seconds|
|3|10 + 4 seconds|
|4|10 + 8 seconds|
|5|10 + 16 seconds|
|6|10 + 32 seconds|
|7|10 + 60 seconds|
|8|10 + 60 seconds|

> #### ！注意
> 如果批处理（batching）开启，一批内的消息全部都会被重新传递给consumer。
#### 1.2.3.5 确认超时（Acknowledgement timeout）
确认超时机制允许你设置一个范围时间，consumer客户端会去追踪未确认的消息。当这些消息超过了确认时间（`ackTimeout`），客户端会发送`redeliver unacknowledged messages`请求给broker，然后broker会重新发送这些未被确认的消息给consumer。  

你可以配置确认超时机制来重传那些超过了`ackTimeout`却未ack的消息，或者运行一个timer task去检查在每个`ackTimeoutTickTime`间ack超时的消息。  

你也可以使用重传补偿机制（redelivery backoff mechanism），通过不同的延迟来多次重传消息。  

如果你想要用重传补偿，你可以使用如下API。
```
consumer.ackTimeout(10, TimeUnit.SECOND)
        .ackTimeoutRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
        .minDelayMs(1000)
        .maxDelayMs(60000)
        .multiplier(2).build())
```
消息的重传行为会像下表一样。
|重传次数|重传延迟|
|:------|:------|
|1|10 + 1 seconds|
|2|10 + 2 seconds|
|3|10 + 4 seconds|
|4|10 + 8 seconds|
|5|10 + 16 seconds|
|6|10 + 32 seconds|
|7|10 + 60 seconds|
|8|10 + 60 seconds|

> #### ！注意
> - 如果批处理（batching）开启，一批内的消息全部都会被重新传递给consumer。
> - 与确认超时相比，使用否定确认会更好。首先，确认超时很难设置一个超时值。其次，当broker重新发送这些确认超时的消息时，也许这些消息不需要再被重复消费了（超时，并没有消费失败）。

使用如下API来启用确认超时。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                .topic(topic)
                .ackTimeout(2, TimeUnit.SECONDS) // the default value is 0
                .ackTimeoutTickTime(1, TimeUnit.SECONDS)
                .subscriptionName("sub")
                .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                .subscribe();

Message<byte[]> message = consumer.receive();

// wait at least 2 seconds
message = consumer.receive();
consumer.acknowledge(message);
```
#### 1.2.3.6 消息重试topic（Retry letter topic）
重试topic允许你保存一些被consumer消费失败的数据，并且过一段时间consumer会再次尝试消费它们。在这种方式下，你可以自定义每个消息的重传间隔时间。Consumers在所在的原始topic也会自动订阅消息重试topic。一旦某个消息超过了重试最大次数，并且这个消息还是没法被消费时，它会被移动至死信topic（dead letter topic）。  

消息重试topic的概念如下图。
<div align="center">
  <img src="/imgs/consumer/retry-letter-topic.svg"></img>
</div>
使用消息重试topic与使用消息延迟传递不同，即使他们俩都旨于过一段时间消费消息。消息重试topic是处理消费失败的消息，以确保这些数据不会丢失，而延迟消息传递是单纯的为了在未来的某个特定时间点进行消息传递。  

默认情况下，消息自动重试是关闭的。你可以设置`enableRetry`为`true`来为consumer启用消息重试。  

如何使用消息重试API可以参考下面示例，当重试次数达到`maxRedeliverCount`时，未被消费的消息会被移动至死信topic。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .enableRetry(true)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                        .maxRedeliverCount(maxRedeliveryCount)
                        .build())
                .subscribe();
```
默认的消息重试topic会以下面格式命名。
```
<topicname>-<subscriptionname>-RETRY
```
使用Java客户端来命名你的消息重试topic
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Shared)
        .enableRetry(true)
        .deadLetterPolicy(DeadLetterPolicy.builder()
                .maxRedeliverCount(maxRedeliveryCount)
                .retryLetterTopic("my-retry-letter-topic-name")
                .build())
        .subscribe();
```
在消息重试topic中的消息会有一些特殊的属性，这些属性由客户端自动创建
|特殊属性|描述|
|:------|:---|
|`REAL_TOPIC`|对应的实际topic名称。|
|`ORIGIN_MESSAGE_ID`|原始的消息ID，这对消息追踪很重要。|
|`RECONSUMETIMES`|消息重试次数|
|`DELAY_TIME`|消息重试间隔时间|
##### 例子
```
REAL_TOPIC = persistent://public/default/my-topic
ORIGIN_MESSAGE_ID = 1:0:-1:0
RECONSUMETIMES = 6
DELAY_TIME = 3000
```
使用下面的API将消息放在重试队列中。
```
consumer.reconsumeLater(msg, 3, TimeUnit.SECONDS);
```
使用下面的API来为`reconsumerLater`函数增加一些自定义参数属性。在下一次消费时，自定义的属性可以通过message#getProperty获取。
```
Map<String, String> customProperties = new HashMap<String, String>();
customProperties.put("custom-key-1", "custom-value-1");
customProperties.put("custom-key-2", "custom-value-2");
consumer.reconsumeLater(msg, customProperties, 3, TimeUnit.SECONDS);
```
> #### ！注意
> - 目前，消息重试topic可以在共享订阅类型中启用。
> - 与否定确认（nack）相比，消息重试topic更适合需要大量重试的消息。因为在消息重试topic中的消息会被持久化到BookKeeper中，而那些由于被nack，需要重传的消息被缓存在客户端。
#### 1.2.3.7 死信topic（Dead letter topic）
死信topic允许你继续消费消息，即使一些消息没法被成功消费。这些被消费失败的消息会被保存到一个特殊的topic中，这个topic称为死信topic。你可以决定如何处理死信topic中的数据。  

在Java客户端中启用默认的死信topic。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                      .maxRedeliverCount(maxRedeliveryCount)
                      .build())
                .subscribe();
```
死信topic的默认以下格式
```
<topicname>-<subscriptionname>-DLQ
```
使用Java客户端来自定义你的死信topic名
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                      .maxRedeliverCount(maxRedeliveryCount)
                      .deadLetterTopic("my-dead-letter-topic-name")
                      .build())
                .subscribe();
```
默认情况下，在DLQ创建的时候不会有任何订阅存在，这可能会导致你丢失消息。为了给DLQ自动初始化创建一个订阅，你可以自定义`initialSubscriptionName`参数。如果设置了这个参数，但broker的`allowAutoSubscriptionCreation`是禁用的，那么DLQ producer将会创建失败。  

死信topic目前会由确认超时、否定确认或者消息重传topic触发。
> #### ！注意
> 目前来说，死信toic允许在share和key_share的订阅类型下使用。
### 1.2.4 主题（Topics）
像其他发布-订阅的系统一样，Pulsar中的topic是将producers的消息运输给consumers间的通道。Topic有良好的URL命名风格。
```
{persistent|non-persistent}://tenant/namespace/topic
```
|Topic名称组成|描述|
|:-----------|:--|
|`persistent`/`non-persistent`|用来标识topic的类型。Pulsar支持两种类型的topic：persistent和non-persistent。默认情况下是persistent，所以你如果你没有指定某种类型，那么这个topic为persistent类型。persistent的topic，所有消息都会被持久化到硬盘上（如果broker非单机，那么这些消息还会被持久化到多块硬盘上），然而non-persistent的topics不会被持久化到硬盘上。|
|`tenant`|实例中的topic租户。租户对Pulsar的多租户至关重要，并分布在集群中。|
|`namespace`|将相关联的topic作为一个组来管理，是管理Topic的基本单元。大多数topic的配置都是在namespace的级别执行。 每个租户里面可以有一个或者多个namespace。|
|`topic`|名称的最后一部分。topic名称在Pulsar实例中没有特殊意义。|
#### 不需要去显示创建新topics
在Pulsar中，你不需要去显示创建topics。当一个客户端尝试去往一个不存在的topic写/从一个不存在的topic中读的时候，Pulsar会在提供的topic名称中的命名空间下自动创建一个topic。如果客户端创建topic时未指定tenant或namespace，那么该topic会被自动创建到default tenant和namespace中。你也可以指定tenant和namespace创建topic，比如`persistent://my-tenant/my-namespace/my-topic`.`persistent://my-tenant/my-namespace/my-topic`意味着`my-topic`这个topic会被创建在命名空间为`my-namespace`以及租户名称为`my-tenant`下。
### 1.2.5 命名空间（Namespaces）
命名空间是一个租户（tenant）下的逻辑术语。一个tenant通过admin API创建命名空间（namespace）。举个例子，一个tenant有不同的应用，那么可以为每个应用创建一个namespace来将他们隔离开来。namespace允许应用创建和管理多个topics。topic `my-tenant/app1`是`my-tenant`租户下，应用程序为`app1`的命名空间。你可以在一个namespace下创建多个topic。
### 1.2.6 订阅（Subscriptions）
订阅是一种配置规则，用于决定如何将消息传递给消费者。在Pulsar中，有四种订阅规则可以使用：exclusive（独占），shared（共享），failover（灾备）和key_shared（key共享）。这些类型如下图所示。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/subscription-types.png" />
</div>

> #### Pub-Sub或Queuing
> 在Pulsar中，你可以灵活地使用不同的订阅类型
> - 如果你想要在消费者中实现传统的发布订阅类型，你可以为每个消费者的subscription设定一个独一无二的名称。这是独占订阅类型。
> - 如果你想要在消费者中实现消息队列，可以在多个消费者中共享相同的subscription（shared，failover，key_shared）
> - 如果你想要实现这两种效果，请将exclusive与其他订阅类型相结合。
 
#### 订阅类型（Subscription types）
当一个subscription没有consumer时，它的订阅类型是未定义的（undefined）。当一个consumer连入subscription时，它的类型才会被定义，可以通过更改消费者配置并重启所有消费来修改订阅类型。

##### 独占（Exclusive）
在**独占（Exclusive）** 模式下，只有单个consumer被允许链接到subscription中。如果多个consumer使用同一个subscription订阅了topic，那么会发生错误。注意如果topic被分区了，那么所有分区的消息只会被一个consumer消费。
> exclusive是默认的订阅类型
<div style="margin: 0 auto">
  <img src="/imgs/subscription/exclusive.png" />
</div>

##### 灾备（Failover）
在**灾备（Failover）** 模式下，多个消费者可以链接到同一个subscription。可以为非分区或每个分区选择一个主消费者来接受消息。当主消费者失去连接时，所有（未确认和后续）的消息都将会传递给下一个消费者。

对于存在分区的topic，broker将会对消费者通过优先级和名称字典顺序进行排序。然后broker将会尝试将不同分区内的消息平均分配给高优先级的consumer。

对于未分区的topic，broker将会按照订阅的顺序选择consumer。

例如：一个topic有15个分区以及3个consumer。每个consumer会消费5个分区，每个分区都会有1个活跃的consumer以及4个等待的consumer。

在下图中，**Consumer-B-0**是主consumer，当它失去连接时，**Consumer-B-1**会代替他去接收消息。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/failover.png" />
</div>

##### 共享（Shared）
在*shared*或*round robin*类型中，多个consumers可以连接到同一个subscription中。消息会通过轮询的方式传递给各个消费者，也有一些指定的消息会被发送给指定的消费者。当一个消费者失去连接，所有已经发送给它但未确认的消息将重新安排发送给其余消费者。

在下图中，**Consumer-C-1** 和 **Consumer-C-2** 可以订阅到同一个topic，当然 **Consumer-C-3** 和其他消费者也可以。

> ###### Shared类型的限制
> 当使用Shared类型时，请注意
> - 消息的顺序无法被保证
> - 无法使用累计消息确认
<div style="margin: 0 auto">
  <img src="/imgs/subscription/shared.png" />
</div>

##### Key共享（Key_Shared）
在Key_Shared类型下，多个消费者可以连接到同一个subscription中，消息会在consumers中分发传递，具有相同key的消息只会发给一个consumer。无论这个消息被重新发送过多少次，它都只会发送给相同的consumer。当一个一个消费者连接或失去连接时，这会导致服务消费者更改一部分消息key。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/key-shared.png" />
</div>

注意当consumers使用Key_Shared的订阅类型时，你需要对producers做出 **禁用batching**或**使用key-based batching**的操作。理由如下：
- 1.broker通过消息的key来进行消息分发，但默认的batching可能无法将具有相同key的消息打包到同一批中。
- 2.batch中的第一个消息的key作为整个batch中所有消息的key，这可能会导致一些上下文错误。
- 
key-based batching就是用于解决上述问题。这个batching会保证producers将具有相同key的消息保存到同一batch中。没有key的消息将会打包到同一个batch中，这个batch没有key。当broker分发这些batch信息时，他会使用`NON_KEY`来作为key。此外，每个consumer只会收到一个key或者收到一个相同key的batch信息。默认情况下，你可以限制producer往batch中打包信息的数量。
下面是在Key_Shared订阅类型下使用key-based batching的例子，使用pulsar client：
```
Producer<byte[]> producer = client.newProducer()
                                .topic("my-topic")
                                .batcherBuilder(BatcherBuilder.KEY_BASED)
                                .create();
```

> ###### Key_Shared类型的限制
> 当你使用Key_Shared类型时，请注意：
> - 你需要为消息指定一个key
> - 你无法使用累计消息确认

#### 订阅模式（Subscription modes）
##### 什么是订阅模式
订阅模式代表游标类型。
- 当一个订阅被创建时，一个游标会指向最后一个消息被消费的位置。
- 当该订阅的一个consumer重启时，它可以通过游标继续消费消息。

| 订阅模式   | 描述                                                                                         | 笔记                                                      |
|:-------|:-------------------------------------------------------------------------------------------|:--------------------------------------------------------|
| `持久化`  | 游标是持久的，它会保留以及持久当前指向位置。<br/>如果一个broker通过灾备重启了，它可以通过Bookeeper来进行游标恢复，所以消息可以从上次最后一次消费的地方继续消费。 | `持久化`是默认的订阅模式                                           |
| `非持久化` | 游标是非持久的。<br/>一旦broker停止，游标会丢失并且无法恢复，所以无法定位到上次消费的最后一条消息。                                    | Reader的订阅模式是`非持久化`的，它无法阻止topic中的数据被删除。Reader的订阅模式无法被改变。 |

一个订阅可以有一个或者多个consumers。当一个consumer订阅了一个topic，它必须指定subscription名称。一个持久化的subscription和一个非持久化的subscription可以拥有相同的名称。它们彼此独立。如果一个consumer指定了一个先前不存在的subscription，那么这个subscription会自动创建。

##### 何时使用
默认情况下，一个topic如果没有持久化的订阅，那么这个topic中的消息都会被标记为已删除。如果你想要防止这些消息被标记为已删除，你可以为这个topic创建一个持久化的订阅。这种情况下，只有被确认的消息才会被标记为已删除。更多信息请查看 message retention and expiry。

##### 如何使用
一个consumer被创建之后，consumer默认的订阅模式是`持久化`的。你可以通过配置来改变。
```
        Consumer<byte[]> consumer = pulsarClient.newConsumer()
                .topic("my-topic")
                .subscriptionName("my-sub")
                .subscriptionMode(SubscriptionMode.Durable)
                .subscriptionMode(SubscriptionMode.NonDurable)
                .subscribe();
```

### 多主题订阅（Multi-topic subscriptions）
当一个consumer订阅到一个Pulsar topic时，默认情况下，它是订阅到一个特殊的主题，比如`persistent://public/default/my-topic`。从Pulsar版本1.23.0开始，Pulsar消费者可以同时订阅多个topic。你可以通过两种方式来订阅。
- 基于正则，例如`persistent://public/default/finance-.*`
- 显示指定topic列表。

> 当使用正则订阅多个topic时，所有topic必须有相同的命名空间。

当订阅了多个topic时，Pulsar客户端会自动调用Pulsar API去查找这些正则/列表定义的topic，如果topic不存在，那么当这些topic创建后，consumer会自动订阅它们。
> ###### 多个topic无法保证消息顺序
> 当producer向单个topic发送消息时，所有消息都保证以相同的顺序从该topic读取。但是，这些保证不适用于多个topics。因此，当producer向多个topics发送消息时，从这些topics读取消息的顺序并不一定相同。

以下是多topic订阅的Java示例。
```
import java.util.regex.Pattern;

import org.apache.pulsar.client.api.Consumer;
import org.apache.pulsar.client.api.PulsarClient;

PulsarClient pulsarClient = // 初始化Pulsar client对象

// 订阅namespace下所有topic
Pattern allTopicsInNamespace = Pattern.compile("persistent://public/default/.*");
Consumer<byte[]> allTopicsConsumer = pulsarClient.newConsumer()
                            .topicsPattern(allTopicsInNamespace)
                            .subscriptionName("subscription-1")
                            .subscribe();
                            
// 订阅namespace下某个topic子集
Pattern someTopicsInNamespace = Pattern.compile("persistent://public/default/foo.*");
Consumer<byte[]> someTopicsConsumer = pulsarClient.newConsumer()
                            .topicsPattern(someTopicsInNamespace)
                            .subscriptionName("subscription-1")
                            .subscribe();
```
### 分区topic（Partitioned topics）
普通topic只有一个broker为其提供服务，这限制了topic的最大吞吐量。*分区topic*是一种特殊的topic类型，它有多个broker为其服务，因此允许更高的吞吐量。

一个分区topic是由N个内部topic实现的，N就是分区的数量。当发布一个消息到分区topic时，每个消息都会路由到其中一个broker上。分发的过程由Pulsar自动处理。

如下图所示。
<div style="margin: 0 auto">
  <img src="/imgs/partitioned-topics/partitioning.png" />
</div>

*Topic1*有5个分区（P0到P4）分别散落在三个broker上。由于它的分区数量大于broker数量，所以两个broker分别处理两个分区，第三个broker只处理一个分区。

这个topic中的消息会被分发给两个consumer。路由模式（routing mode）确定每个消息分发到哪个分区，而订阅类型决定这些消息发给哪些消费者。

大多数情况下，可以分别决定路由模式和订阅类型。一般来说路由/分区的选择与吞吐量有关，订阅策略与应用程序有关。

普通topic和分区topic在订阅类型上没有区别，因为分区只决定了消息在producer和consumer之间发生的事情。

分区topic需要通过admin API显示指定创建。同时可以指定分区数。

#### 路由模式（Routing modes）
当发布消息到分区topic时，你必须指定一种路由模式。路由模式决定了你的消息将会被发送到这个topic的哪个分区上。
有三种路由模式可以使用。

| 模式                  | 说明                                                                                                                                                            |
|:--------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RoundRobinPartition | 如果没有提供key，那么producer将会通过轮询的方式将消息发送给每个分区以达到最大的吞吐量。注意轮询不是以单个消息为单位进行轮询，而是与batching delay的边界相同以确保批处理有效。如果有为消息提供指定的key，producer会进行hash然后将消息发送给对应的partition。这是默认的模式 |
| SinglePartition     | 如果没有提供key，那么producer将会随机选择一个分区，并且将所有的消息发送到这个分区中。当消息被指定一个key时，producer会进行hash然后将消息发送给对应的partition                                                              |
| CustomPartition     | 使用自定义的消息路由来进行消息发送，可以通过使用Java客户端以及实现MessageRouter接口来创建自定义的路由模式。                                                                                                |

#### 顺序保证（Ordering guarantee）
消息的顺序与MessageRoutingMode 和 Message Key有关。通常来说，用户希望每个partition中相同key的消息能够保证有序。

如果给消息指定了一个key，那么当使用`SinglePartition`或`RoundRobinPartition`时消息会通过ProducerBuilder中的HashingScheme来决定发送到哪个分区。

| 顺序保证              | 描述                 | 路由模式与Key                                               |
|:------------------|:-------------------|:-------------------------------------------------------|
| Per-key-partition | 一个分区内所有相同key的消息都有序 | 使用`SinglePartition`或`RoundRobinPartition`模式，Key由每个消息提供 |
| Per-producer      | 来自同一个producer的消息有序 | 使用`SinglePartition`模式，消息不含有Key                         |

#### 哈希策略（Hashing scheme）
HashingScheme是一个枚举集合，表示在选择用于特定消息的分区时可用的标准哈希函数集。

有两种标准哈希函数可用：`JavaStringHash`和`MurruR3_32Hash`。producer的默认哈希函数是`JavaStringHash`。请注意，当producer来自不同的多语言客户端时，`JavaStringHash`会失效，在这种情况下，建议使用`Murrur3_32Hash`。

### 非持久化主题（Non-persistent topics）
默认情况下，Pulsar会将所有未确认的消息持久化存储在多个存储节点（BookKeeper bookies）上。持久化topic中的消息数据可以在broker重启后或订阅用户故障转移后恢复。

Pulsar同时也支持*非持久化topics*，这些topic中的消息不会被持久化到硬盘上，只会存储在内存中。当使用非持久时，终止broker或断开主题订户的连接意味着该（非持久）topic上的所有传输中消息都会丢失，这意味着客户端可能会看到消息丢失。

非持久化topic的格式：
```
non-persistent://tenant/namespace/topic
```

在非持久化topic中，broker收到消息后会立刻转发给订阅者而不将他们先进行持久化到BookKeeper中。如果一个订阅者失去连接，broker无法再次传递这些在途的消息，订阅者同时再也无法收到这些消息。在某些情况下，非持久化topic的消息传递速度会比持久化topic传递速度更快，但与此同时，Pulsar的核心优势也将丧失。

> 在非持久化topic下，消息数据只会存在于内存中，并不会对其进行特别的缓存。broker接收到消息的时候会立刻转发给所有连接着的consumer。如果broker宕机或无法从内存中检索消息，那么这个消息可能会丢失。只有当你确定业务场景需要使用非持久化topic时，才可以使用。

默认情况下，broker允许使用非持久化topic，你可以在broker的配置文件中去禁用她们。你可以通过`pulsar-admin topics`的命令去管理这些非持久化的topic。

目前，非持久化且未分区的topic不会持久化到ZooKeeper上，这意味着如果broker自身宕机了，他们不会重新分配给其他broker因为这些东西存在于它们broker自身的内存中。当前的解决方法是在broker配置中将`allowAutoTopicCreation`的值设置为true，并将`AllowAutoTopicCreationType`设置为`non-partitioned`（它们是默认值）。

#### 性能（Performance）
非持久topic消息传递通常比持久化topic消息传递更快，因为broker不会持久化消息，并在消息传递到连接的broker后立即将ACK发送回producer。因此，producer可以看到非持久topic发布延迟相对较低。

#### 客户端API（Client API）
Producers和consumers可以像连接持久化topic那样连接到非持久化topic上，主要区别就是在于非持久化topic必须以`non-persistent`开头。同样非持久化topic有三种订阅类型exclusive，shared和failover。

下面是Java consumer使用非持久化topic的例子。
```
PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar://localhost:6650")
        .build();
String npTopic = "non-persistent://public/default/my-topic";
String subscriptionName = "my-subscription-name";

Consumer<byte[]> consumer = client.newConsumer()
        .topic(npTopic)
        .subscriptionName(subscriptionName)
        .subscribe();
```
下面是Java producer使用非持久化topic的例子
```
Producer<byte[]> producer = client.newProducer()
        .topic(npTopic)
        .create();
```

### 系统Topic（System Topic）
系统Topic时Pulsar内置的topic。它可以是持久化或非持久化Topic。

系统Topic是用来实现某些功能以及消除对第三方组件的依赖，比如事务、心跳检测、topic-level策略以及资源组服务。系统topic使这些功能变得更加简单，独立和灵活。拿心跳检测举例，你可以允许生产者消费者在heartbeat namespace下进行生产和消费来进行health check来判断当前服务是否存货。

根据namespace的不同，存在不同的系统topic。下表概述了每个特定namespace的可用系统topic。

| Namespace       | TopicName                               | Domain         | Count                                                           | Usage                             |
|:----------------|:----------------------------------------|:---------------|:----------------------------------------------------------------|:----------------------------------|
| pulsar/system   | `transaction_coordinator_assign_${id}`  | Persistent     | Default 16                                                      | Transaction coordinator           |
| pulsar/system   | `__transaction_log_${tc_id}`            | Persistent     | Default 16                                                      | Transaction log                   |
| pulsar/system   | `resource-usage`                        | Non-persistent | Default 4                                                       | Resource group service            |
| host/port       | `heartbeat`                             | Persistent     | 1                                                               | Heartbeat detection               |
| User-defined-ns | `__change_events`                       | Persistent     | Default 4                                                       | Topic events                      |
| User-defined-ns | `__transaction_buffer_snapshot`         | Persistent     | One per namespace                                               | Transaction buffer snapshots      |
| User-defined-ns | `${topicName}__transaction_pending_ack` | Persistent     | One per every topic subscription acknowledged with transactions | Acknowledgments with transactions |

> #### 注意
> - 你无法创建任何系统Topics。当你想要通过Pulsar admin API获得topic列表时，你可以通过添加`--include-system-topic`来获取系统topics。
> - 从Pulsar2.11.0开始，系统topics默认启用。在先前的版本中，你需要在`conf/broker.conf`或`conf/standalone.conf`中启用。
> ```
> systemTopicEnabled = true
> topicLevelPoliciesEnabled = true
> ```

### 消息重传（Message redelivery）
Pulsar支持优雅的灾备处理保证数据不会丢失。软件总是会出现意料之外的异常，在某些时候消息可能无法发送成功。因此，具有处理故障的内置机制非常重要，特别是在异步消息传递中，比如
- Consumers与数据库或HTTP服务器断开连接。当这种情况发生的时候，消费者在往数据库写入数据时会造成短暂的脱机。消费者调用的外部HTTP服务器暂时不可用。
- 由于Consumer的crash导致Consumers与broke断开连接后，未确认的消息会发送给其他可用的consumers。

Apache Pulsar使用至少一次传递来避免这些和其他消息传递失败，确保Pulsar的多次处理消息。

要利用消息重新传递，您需要在broker可以在Apache Pulsar客户端中重新发送未确认消息之前启用此机制。您可以使用三种方法激活Apache Pulsar中的消息重新传递机制。
- Negative Acknowledgment
- Acknowledgment Timeout
- Retry letter topic

### 消息的保留和过期（Message retention and expiry）
默认情况下，Pulsar的broker：
- 立刻删除consumer的ack的消息。
- 持久化未ack的消息到message backlog。

Pulsar有两种特性，让你可以覆盖这两种默认的行为
- 消息保留：你可以在consumer ack消息之后继续保留消息。
- 消息过期：你可以为未ACK的消息设置一个TTL。

> 所有的消息保留和过期都是namespace级别的。

下图说明了两个概念。
<div style="margin: 0 auto">
  <img src="/imgs/retention-and-expiry/retention-expiry.png" />
</div>

### 重复消息删除（Message deduplication）

