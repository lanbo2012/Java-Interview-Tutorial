Kafka，具有高水平扩展和高吞吐量的分布式消息系统。

Kafka对消息保存时根据Topic进行归类，发送消息者成为Producer，消息接受者成为Consumer，此外kafka集群有多个kafka实例组成，每个实例(server)称为broker。

无论是Kafka集群,还是producer和consumer都依赖于zookeeper来保证系统可用性，为集群保存一些meta信息。

- 主流MQ对比
![](https://img-blog.csdnimg.cn/20191113003523808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9qYXZhZWRnZS5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)


**Apache Kafka是消息引擎系统，也是一个分布式流处理平台（Distributed Streaming Platform）**

> Kafka是LinkedIn公司内部2011年孵化的项目。LinkedIn最开始有强烈的数据强实时处理方面的需求，用作LinkedIn的活动流(Activity Stream)和运营数据处理管道(Pipeline) 的基础。其内部的诸多子系统要执行多种类型的数据处理与分析，主要包括业务系统和应用程序性能监控，以及用户行为数据处理等。

# 1 Kafka主要特性
## 1.1 流特性
Kafka是一个流处理平台，流平台需如下特性：
- 可`发布和订阅`流数据，类似MQ
- 以容错方式`存储`流数据
- 可在流数据产生时就进行处理(Kafka Stream)

## 1.2 适用场景
- 基于Kafka，构造实时流数据管道，让系统或应用之间可靠地获取数据
- 构建实时流式应用程序，处理流数据或基于数据做出反应

# 2 遇到的问题
## 数据正确性不足
数据的收集主要采用轮询（Polling），确定轮询间隔时间就成了高度经验化的难题。虽然可采用一些启发式算法（Heuristic）来评估，但一旦指定不当，还是会造成较大数据偏差。
## 系统高度定制化，维护成本高
各子系统都需对接数据收集模块，引入了大量定制开销。

LinkedIn尝试过使用ActiveMQ解决这些问题，但并不理想，显然需要有一个“大一统”系统来取代现有的工作方式，它就是Kafka。

Kafka自诞生就是以消息引擎系统的面目出现在大众视野，翻看0.10.0.0之前的官网说明：
![](https://img-blog.csdnimg.cn/20190825021942674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNTg5NTEw,size_1,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20190825022242473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNTg5NTEw,size_1,color_FFFFFF,t_70)Kafka社区将其清晰地定位为一个分布式、分区化且带备份功能的日志提交（Commit Log）服务。

> Kafka作者之一Jay Kreps曾经谈及过命名的原因。
因为Kafka系统的写性能很强，所以找了个作家的名字来命名似乎是一个好主意。大学期间我上了很多文学课，非常喜欢Franz Kafka这个作家，另外为开源软件起这个名字听上去很酷。

Kafka旨在提供如下
# 3 特性
- 提供一套API实现生产者和消费者
- 降低网络传输和磁盘存储开销
- 实现高伸缩性架构

# 4 流处理
随Kafka不断完善，Jay等大神们意识到将其开源是个非常棒的主意，因此在2011年Kafka正式进入到Apache基金会孵化并于次年10月顺利毕业成为Apache顶级项目。

在大数据领域，Kafka在承接上下游、串联数据流管道方面发挥了重要的作用：
`所有的数据几乎都要从一个系统流入Kafka然后再流向下游的另一个系统中`。

这引发了Kafka社区的思考：与其我把数据从一个系统传递到下一个系统中做处理，何不自己实现一套流处理框架？
Kafka社区于0.10.0.0版本正式推出了流处理组件**Kafka Streams**，也正是从这个版本开始，Kafka正式“变身”为分布式的流处理平台，而不仅仅是消息引擎系统。
今天Apache Kafka是和Storm/Spark/Flink同等级的实时流处理平台。

## 优势
### 更易实现端到端的正确性（Correctness）
Google大神Tyler曾经说过，流处理要最终替代它的“兄弟”批处理需要具备两点核心优势：
- 实现正确性
- 提供能够推导时间的工具

**实现正确性是流处理能够匹敌批处理的基石**
正确性一直是批处理的强项，而`实现正确性的基石则是要求框架能提供精确一次处理语义，即处理一条消息有且只有一次机会能够影响系统状态`
目前主流的大数据流处理框架都宣称实现了精确一次处理语义，但这是有限定条件的，即它们`只能实现框架内的精确一次处理语义，无法实现端到端`
因为当这些框架与外部消息引擎系统结合时，无法影响到外部系统的处理语义，所以Spark/Flink从Kafka读取消息之后进行有状态的数据计算，最后再写回Kafka，只能保证在Spark/Flink内部，这条消息对于状态的影响只有一次
但是计算结果有可能多次写入到Kafka，因为它们不能控制Kafka的语义处理
相反地，Kafka则不是这样，因为所有的数据流转和计算都在Kafka内部完成，故Kafka可以实现端到端的精确一次处理语义

> 举个例子，使用Kafka计算某网页的PV——我们将每次网页访问都作为一个消息发送的Kafka
> PV的计算就是我们统计Kafka总共接收了多少条这样的消息即可
> 精确一次处理语义表示每次网页访问都会产生且只会产生一条消息，否则有可能产生多条消息或压根不产生消息。

## 流式计算的定位
官网明确**Kafka Streams**是一个用于搭建实时流处理的客户端库而非是一个完整的功能系统。

不能期望Kafka提供类似集群调度、弹性部署等开箱即用的运维特性，需要自己选择适合的工具或系统来帮助Kafka流处理应用实现这些功能。这的确是一个“双刃剑”设计，也是Kafka社区“剑走偏锋”不正面PK其他流计算框架的特意考量。
大公司的流处理平台一定是大规模部署，因此具备集群调度功能以及灵活的部署方案是不可或缺的要素，但毕竟这世界上还存在着很多中小企业，它们的流处理数据量并不巨大，逻辑也并不复杂，部署几台或十几台机器足以应付。在这样的需求之下，搭建重量级的完整性平台实在是“杀鸡焉用牛刀”，而这正是Kafka流处理组件的用武之地。
因此未来在流处理框架中，Kafka应该有一席之地。

> Kafka能够被用作分布式存储系统
Kafka作者之一Jay Kreps曾经专门写过一篇文章阐述为什么能把Kafka用作分布式存储。不过还真没见过在生产环境中把Kafka当作持久化存储来用的。

> 参考
> - Apache Kafka实战