---
{"dg-publish":true,"permalink":"/k8s进阶学习/日志/OpenObserve/","dgPassFrontmatter":true}
---


openObserve 是⼀个 Rust 开发的开源的⾼性能云原⽣可观测平台（⽇志、指标、追踪），⽐起 Elasticsearch 它⼤约可以节省 140 倍的存储成本，OpenObserve 能够处理 PB 级的数据。

##### OpenObserve 对比 Elasticsearch

Elasticsearch 是⼀个通⽤搜索引擎，可以使⽤应⽤程序搜索或⽇志搜索。OpenObserve 是专门为⽇志搜索⽽构建的，它的特点如下：

+ 不依赖于数据索引，将未索引的数据以压缩格式存储在本地磁盘或以**parquet**列格式的对象存储中

+ 压缩率非常高，存储成本降低140倍
+ ⽆状态节点架构允许OpenObserve⽔平扩展，⽽⽆需担⼼数据复制或损坏
+ 相比于ES，运维成本和工作量低很多

##### 架构

OpenObserve 可以在单节点下运⾏，也可以在集群中以 HA 模式运⾏

###### 单节点模式

**Sled 和本地磁盘模式（默认模式）**

单节点每天可处理超2TB数据，M2的配置下，处理速度约为31MB/S，即每分钟处理1.8GB数据

![1692022766920(1)](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1692022766920(1).png)

**Sled 和对象存储模式**

该模式和 OpenObserve的默认模式基本上⼀致，只是数据存在了对象存储中，这样可以更好的⽀持⾼可⽤性，因为数据不会丢失

![image-20230814222518339](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/image-20230814222518339.png)

**Etcd 和对象存储模式**

该模式是使⽤ Etcd 来存储元数据，数据仍然存储在对象存储中

![1692023196012](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1692023196012.png)

###### HA模式

HA模式不⽀持本地磁盘存储，集群模式下OpenObserve会运⾏多个节点，每个节点都是⽆状态的，数据存储在对象存储中，元数据存储在Etcd中，这样可以更好的⽀持⾼可⽤性

![image-20230814222803810](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/image-20230814222803810.png)

在该模式下 OpenObserve 主要包括 Router、Querier、Ingester 和 Compactor 四个组 件，这些组件都可以⽔平扩展；Etcd ⽤于存储⽤户、函数、报警规则和集群节点信息等元数据；对象存储（例如 s3、minio、gcs 等等）存储 parquet⽂件和⽂件列表索引的所有数据

**Router**

> Router路由器将请求分发给 ingester 或 querier，它还通过浏览器提供 UI 界⾯。Router 实际上就是⼀个⾮常简单的代理，⽤于在数据摄⼊程序和查询程序之间发送适当的请求并进 ⾏响应。

Ingester

> ⽤于接收摄取请求并将数据转换为 parquet格式然后存储在对象存储中，它们在将数据传输到对象存储之前将数据临时存储在 WAL中

**Querier**

> ⽤于查询数据，查询器节点是完全⽆状态的

**Compactor**

> 将⼩⽂件合并成⼤⽂件，使搜索更加⾼效。 Compactor还处理数据保留策略、full stream 删除和⽂件列表索引更新



[实验文档](https://github.com/wjunlove123/k8slearing/tree/main/log/openobserve)

