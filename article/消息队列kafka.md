https://www.cnblogs.com/flaming/p/11304278.html

# 一文看懂-Kafka消息队列
## 一、Kafka简介
### 1.1 什么是kafka
kafka是一个**分布式、高吞吐量、高扩展性**的消息队列系统。kafka最初由Linkedin公司开发，2010年贡献给Apache基金会，成为开源项目。主要应用于**日志收集系统和消息系统**，与RabbitMQ、ActiveMQ等消息队列中间件功能类似（也可称KafkaMQ），核心优势在于**稳定性更高、效率更强**。

### 1.2 Kafka中的相关概念
Kafka核心概念包含：2个Producer（生产者）、1个Topic（主题）、3个Partition（分区）、3个Replica（副本）、3个Broker（Kafka实例/节点）、1个Consumer Group（消费者组，含3个Consumer），具体说明如下：

#### 1.2.1 Producer（生产者）
生产者即**发送消息的角色**，每发送一条消息必须指定一个Topic（主题），持续向Kafka服务器推送消息。

#### 1.2.2 Topic（主题）
每一条发送到Kafka的消息都对应一个主题，可理解为“消息类别”，类似传统数据库中的“表名”（例如：主题为“order”的消息，均为订单相关内容）。

#### 1.2.3 Partition（分区）
- Topic的消息数据会存储在分区中，与ElasticSearch的“分片”概念一致，核心目的是**拆分数据、实现负载均衡**（分区分布在不同Broker节点上，分散大量消息的存储压力）。
- 每个Topic至少需指定1个分区，可手动配置多个分区；**单个分区内的数据是有序的，但不同分区间的数据不保证有序**（因多分区消费速度不同）。
- 若需保证消息顺序消费，需将Topic设置为**1个分区**（仅消费单个分区，自然保证顺序）。

#### 1.2.4 Replica（副本）
- 副本是分区数据的**备份**，用于防止数据丢失或服务器宕机，保障数据完整性（例如：3个分区需备份所有分区，1份副本含3个分区，2份副本含6个分区）。
- 副本分区会分布在不同服务器上，同样实现负载均衡；逻辑与传统数据库“主从复制”一致：
  - 1个分区作为**主分区（leader）** ，控制消息的读写操作；
  - 其他副本作为**从分区（follower）** ，同步主分区数据；
  - 若服务器宕机，可通过副本恢复数据，或临时用副本提供服务。
- 注：与ElasticSearch的副本机制完全一致。

#### 1.2.5 Broker（实例或节点）
Broker即**Kafka的运行实例**，启动1个Kafka程序就是1个Broker，多个Broker构成Kafka集群（分布式架构的核心体现）， Broker数量越多，集群的吞吐率和效率越高。

#### 1.2.6 Consumer Group（消费者组）和 Consumer（消费者）
- Consumer（消费者）：读取Kafka消息的角色，可消费任意Topic的数据。
- Consumer Group（消费者组）：多个Consumer组成的分组，**每个消费者必须归属一个组**（无指定组名则分配默认组名）。

### 1.3 Kafka的架构与设计
Kafka集群核心组成：1个及以上Producer、1个及以上Broker、1个及以上Consumer Group、1个Zookeeper集群（用于**管理集群配置、负载均衡、故障转移与恢复**）。
- 数据流向：Producer通过**Push（推送）** 方式向Broker发布消息；Consumer通过**Pull（拉取）** 方式从Broker获取消息。

#### 1.3.1 Topic和Partition
- 设计核心：为实现“高吞吐率、高速度”，Topic拆分多个分区，每个Partition在物理磁盘对应1个文件夹，存储该Partition的所有消息和索引文件。
- 消息存储：向Kafka发送消息时，消息会**均衡分散到不同分区**，且每条消息均**顺序写入磁盘**（顺序写磁盘效率高于随机写内存，是Kafka速度快的核心原因之一）。
- 消息保留与删除：
  - 传统MQ消费后消息会删除，Kafka则**保留所有消息**，仅通过“消费者消费偏移量（offset）”标记消费进度，支持消费者自定义offset实现重复消费。
  - 消息删除策略（通过配置选择）：①基于时间；②基于Partition文件大小（避免消息过多占用磁盘空间）。

#### 1.3.2 Producer
- 生产者发送消息时，通过**Partition策略**决定消息存储的分区，默认策略为Kafka内置的“均衡分布”，实现负载均衡。
- 若消息对顺序无要求，可设置多分区提升吞吐率；发送消息需指定**key值**，Producer通过key值与Partition数量（哈希算法）确定目标分区。

#### 1.3.3 Consumer Group 和 Consumer
Kafka通过“消费者组”实现传统MQ的两种消息传播方式：
1. **单播（队列模式）**：1个消息仅被1个消费者消费。
   - 规则：1个Consumer Group中，1个Topic的消息仅能被组内1个Consumer消费；需保证“Topic分区数 = 消费者组内消费者数”（1个消费者对应1个分区），避免多余消费者无法消费消息。
2. **多播（发布-订阅模式）**：1个消息可被多个消费者同时消费。
   - 规则：不同Consumer Group可同时消费1个Topic的消息；只需将每个消费者配置到不同组（例如：多个程序需处理同一消息时，分别归属不同组）。


## 二、Kafka的安装与使用
### 前提条件
需先安装**JDK 1.7及以上版本**，并配置环境变量。

### 2.1 下载Kafka
- 下载地址：[http://kafka.apache.org/downloads](http://kafka.apache.org/downloads)
- 选择“Binary Downloads”，建议选择较旧版本（避免新版本改动导致问题）；下载后直接解压（压缩包支持Linux，Windows可直接使用命令行操作）。

### 2.2 启动Kafka（以Windows为例，路径参考：F:\Dev\kafka_2.11-2.1.0）
#### 2.2.1 启动Zookeeper（Kafka依赖Zookeeper）
1. 进入Kafka根目录，地址栏输入“cmd”打开命令行；
2. 执行命令：  
   `.\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties`  
   （无报错即启动成功）。

#### 2.2.2 启动Kafka
1. 新打开1个命令行（Kafka根目录）；
2. 执行命令：  
   `.\bin\windows\kafka-server-start.bat .\config\server.properties`  
   （无报错即启动成功）。

#### 2.2.3 创建Topic（主题示例：test）
1. 进入路径“F:\Dev\kafka_2.11-2.1.0\bin\windows”，地址栏输入“cmd”打开命令行；
2. 执行命令：  
   `kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test`  
   （无报错即创建成功，replication-factor=1表示1个副本，partitions=1表示1个分区）。

#### 2.2.4 创建Producer（生产者，主题：test）
1. 同路径下新打开命令行；
2. 执行命令：  
   `kafka-console-producer.bat --broker-list localhost:9092 --topic test`  
   （出现光标表示等待输入消息）。

#### 2.2.5 创建Consumer（消费者，主题：test）
1. 同路径下新打开命令行；
2. 执行命令：  
   `kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning`  
   （出现光标表示等待接收消息，--from-beginning表示从 Topic 起始位置消费）。

#### 2.2.6 测试生产和消费
在Producer控制台输入消息，Consumer控制台会实时显示该消息，即测试成功。


## 三、Kafka客户端驱动的使用（以.NET的Confluent.Kafka为例）
### 3.1 创建应用
创建1个解决方案，添加2个控制台项目：1个作为“生产者项目”，1个作为“消费者项目”。

### 3.2 添加Producer和Consumer类
在对应项目中添加类，配置Kafka服务器地址（默认地址需与启动的Kafka一致）。

### 3.3 添加Program.cs中的启动代码
分别在生产者、消费者项目的Program.cs中，编写消息发送、接收的核心代码（实现控制台输入/输出消息逻辑）。

### 3.4 启动服务与实例
1. 按“2.2.1 + 2.2.2”步骤启动Zookeeper和Kafka；
2. 分别启动“生产者实例”和“消费者实例”；
3. 在生产者控制台输入消息，消费者控制台会同步显示消息，与手动命令行测试结果一致。

### 参考与代码托管
- 参考文章：[https://www.cnblogs.com/xxinwen/p/10683416.html](https://www.cnblogs.com/xxinwen/p/10683416.html)、[https://www.cnblogs.com/qingyunzong/p/9004593.html](https://www.cnblogs.com/qingyunzong/p/9004593.html)
- 代码托管（GitHub）：[https://github.com/EmmaCCC/KafkaStudy.git](https://github.com/EmmaCCC/KafkaStudy.git)


## 四、总结
通过以上步骤可掌握Kafka的核心原理与基础使用，详细配置可在实际应用中进一步研究。Kafka作为高并发分布式消息队列系统，可承载千万级消息并发，是技术栈中重要的技能补充（可用于简历技能描述）。
