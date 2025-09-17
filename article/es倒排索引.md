https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247538041&idx=2&sn=8d38d4117584f696b09d070c8e39b73a&chksm=f98d11d3cefa98c5d90f8d6b0e47d31c9cfb8319a5d8e58e8dcf5c3b262584cb5224efdb7a02&scene=178&cur_album_id=3752960238937030659#rd


# 面试官：ES 的倒排索引说一下？- 网页总结
## 一、ES 核心概念与底层基础
### 1. ElasticSearch（ES）定位
- 开源搜索引擎，介于应用与数据之间，支持通过关键词快速检索数据，类似百度搜索的效果，对外提供 HTTP 接口，适配任意语言客户端。
- 底层基于单机文本检索库 **Lucene** 构建，通过架构优化解决 Lucene 高性能、高扩展性、高可用不足的问题。

### 2. Lucene 核心组成
- **Segment**：具备完整搜索功能的最小单元，由倒排索引、Term Index、Stored Fields、Doc Values 四种结构复合而成。
  - 特性：一旦生成不可修改，新增文档需生成新 Segment；可通过“段合并”将多个小 Segment 合并为大 Segment，避免文件句柄耗尽。
  - 搜索逻辑：并发读取多个 Segment，整合结果后返回。


## 二、关键数据结构解析
|数据结构|核心作用|原理细节|
|----|----|----|
|倒排索引|实现关键词快速定位文档|1. 先对文本进行 **分词**，得到“词项（Term）”；<br>2. 构建 **Term Dictionary**（按字典序排序的词项集合，便于二分查找，时间复杂度优化至 O(lgN)）；<br>3. 搭配 **Posting List**（词项对应的文档 ID、词频、词项偏移量等信息），共同构成倒排索引。|
|Term Index|加速倒排索引查询|1. 提取 Term Dictionary 中词项的公共前缀，构建“精简目录树”；<br>2. 目录树节点存储词项在磁盘的偏移量，体积小适合放内存；<br>3. 查询时先查 Term Index 定位词项大致位置，再到磁盘的 Term Dictionary 精准匹配，减少磁盘 IO。|
|Stored Fields|存储完整文档内容|行式存储结构，通过倒排索引获取文档 ID 后，从这里读取原始文档内容，返回给用户。|
|Doc Values|支持文档排序与聚合|列式存储结构，将分散在各文档的同一字段（如时间、价格）集中存放；<br>排序时无需读取完整文档，直接读取该结构，用空间换时间提升效率。|


## 三、ES 分布式架构优化（解决 Lucene 痛点）
### 1. 高性能优化
- **数据分类（Index Name）**：将数据按类别（如体育新闻、八卦新闻）划分到不同 Index Name，每个 Index Name 对应独立的 Lucene 实例，降低单个 Lucene 压力。
- **分片（Shard）**：将单个 Index Name 的数据拆分为多个 Shard，每个 Shard 本质是独立 Lucene 库，读写操作分摊到多个 Shard，减少资源争抢。

### 2. 高扩展性优化
- **节点（Node）**：将多个 Shard 分散部署在不同机器（Node）上，当单机 CPU/内存过高时，可通过增加 Node 扩展资源，缓解性能压力。

### 3. 高可用优化
- **主从分片（Primary/Replica Shard）**：1. 每个 Shard 分为 Primary Shard（主分片，负责写入）和 Replica Shard（副本分片，同步主分片数据）；<br>2. Replica Shard 可分担读请求，且在 Primary Shard 故障时自动升级为主分片，保证服务不中断。

### 4. 节点角色分化
|角色|职责|优势|
|----|----|----|
|主节点（Master Node）|管理集群（如节点加入/退出、分片分配）|避免所有节点承担管理职责，简化集群维护；小规模集群中可与其他角色复用节点。|
|数据节点（Data Node）|存储与管理数据（Shard、Segment 等）|专注数据操作，提升存储与检索效率。|
|协调节点（Coordinate Node）|接收客户端请求，转发与聚合结果|无需存储数据，专注请求处理，提升响应速度。|

### 5. 去中心化
- 无需引入 Zookeeper 等中心节点，通过类似 Raft 一致性算法的协调模块，实现 Node 间数据同步，保证集群状态一致，支持选主与节点故障检测。


## 四、ES 核心流程
### 1. 写入流程
1. 客户端发起写入请求，先发送至 **协调节点**；
2. 协调节点通过 Hash 路由，定位数据应写入的 **Primary Shard**（所在数据节点）；
3. Primary Shard 将数据写入底层 Lucene 的 Segment，生成倒排索引、Stored Fields 等结构；
4. Primary Shard 同步数据至 **Replica Shard**；
5. Replica Shard 写入成功后，Primary Shard 向协调节点返回 ACK；
6. 协调节点响应客户端“写入完成”。

### 2. 搜索流程（分两阶段）
#### （1）查询阶段（Query Phase）
1. 客户端发送搜索请求至协调节点；
2. 协调节点根据 Index Name 信息，将请求转发至对应 Shard（分散在各数据节点）；
3. 每个 Shard 内部并发搜索多个 Segment，通过倒排索引获取文档 ID，结合 Doc Values 得到排序信息，聚合结果后返回协调节点；
4. 协调节点整合所有 Shard 结果，排序后舍弃无用数据，保留核心结果。

#### （2）获取阶段（Fetch Phase）
1. 协调节点携带保留的文档 ID，再次请求对应 Shard；
2. Shard 从 Stored Fields 中读取完整文档内容，返回给协调节点；
3. 协调节点将最终结果返回客户端，完成搜索。


## 五、架构类比与总结
### 1. ES 与 Kafka 架构对比
|ES 组件|Kafka 对应组件|作用相似性|
|----|----|----|
|Index Name|Topic|用于数据分类，区分不同类型的业务数据|
|Shard（分片）|Partition（分区）|拆分数据，实现并行读写，提升性能|
|Node（节点）|Broker（代理）|承载数据存储与处理，实现分布式部署|

### 2. 核心总结
- Lucene 是 ES 底层基础，Segment 是 Lucene 最小搜索单元，倒排索引等结构是检索核心；
- ES 通过“Index Name 分类+Shard 分片+Node 分布式部署+主从副本”，实现高性能、高扩展、高可用；
- 节点角色分化与去中心化设计，进一步优化资源利用率与集群稳定性；
- 学习 ES 架构可类比 Kafka、RocketMQ，优秀架构逻辑具有共通性。
