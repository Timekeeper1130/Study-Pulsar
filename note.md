# 可以看作是官方文档的翻译
始：2022-06-06  
终：  
状态：DOING  
# 1. 概念与架构
## 1.1. 预览（Overview）
Pulsar是一个多租户，高性能的服务器间消息传递解决方案，最早由雅虎开发，现在Pulsar由[Apache软件基金会](https://www.apache.org)管理。  

罗列一下Pulsar的特性：
- 本地的一个Pulsar实例中支持多集群部署（什么鬼），集群间可以做到跨地域无缝复制消息。
- 拥有极低地发布和端到端延迟。
- 简单的客户端API，支持Java，Go，Python和C++
- 支持topic多种订阅模式（独占、共享和灾备）
- 通过[Apache BookKeeper](https://bookkeeper.apache.org)提供的消息持久化机制保证消息的传递。
- 由轻量级的serverless computing框架Pulsar Functions，实现了流原生的数据处理（啥玩意）
- 拥有基于Pulsar Function的serverless connector框架 Pulsar IO，其能够使数据更好的迁入移除Apache Pulsar。
- 当数据老化时，通过分层存储，将数据从热存储转移到冷存储（例如S3和GCS）。
## 1.2 消息传递（Messaging）
Pulsar是基于 [发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 模式(也可以缩写为pub-sub)。在这种模式下，producers发布消息到topics中；consumers订阅这些topic，处理传入的消息，并且当处理消息成功地结束时发送一个ack给broker。  

当一个订阅被创建时，Pulsar会保留所有的消息，即使consumer断开链接。只有当某一个消费者成功处理完毕这些消息，发送了ack后，这些被保留下来的消息才会被丢弃。  

如果一个消息消费失败，并且你希望这个消息能够被再次消费，你可以启用消息重新传递机制来要求broker重新发送这些消息。
### 消息（Message）
消息（Message）是Pulsar的基本“单位”。下表列出了消息的组件。  

### 生产者（Producers）
#### 发送模式（Send Modes）