---
{"dg-publish":true,"permalink":"/k8s进阶学习/日志/搭建EFK日志系统/","dgPassFrontmatter":true}
---

**Elasticsearch** 是⼀个实时的、分布式的可扩展的搜索引擎，允许进⾏全⽂、结构化搜 索，它通常⽤于索引和搜索⼤量⽇志数据，也可⽤于搜索许多不同类型的⽂档

**Elasticsearch** 通常与 **Kibana** ⼀起部署，Kibana 是 Elasticsearch 的⼀个功能强⼤的**数据可视化 Dashboard**，Kibana 允许你通过 web 界⾯来浏览 Elasticsearch ⽇志数据

**Fluentd**是⼀个流⾏的**开源数据收集器**，我们将在 Kubernetes 集群节点上安装 Fluentd， 通过**获取容器⽇志⽂件**、**过滤**和**转换⽇志数据**，然后将数据传递到 Elasticsearch 集群，在 该集群中对其进⾏索引和存储，其他比较流行的日志收集器有vector、ilogtail等

当日志量较大时，可在Fluentd和elasticsearch之间加入一层消息中间件（如KAFKA、MQ等），产生的日志先输入到消息队列中，通过Logstash消费后，再写入到ES中

![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230807142247.png)
##### 安装 Elasticsearch 集群
由于 Elasticsearch 集群复杂度⽐较⾼，对于这种复杂度较⾼的有状态应⽤，⼀般都会 去开发对应的 Operator 来进⾏部署，这样可以简化部署过程，也可以⽅便后续的升级、扩 容等操作，⽐如前⾯我们介绍的 Prometheus Operator、VictoriaMetrics Operator 等等。同样的，对于 Elasticsearch 应⽤，官⽅也有基于 Kubernetes Operator 的应⽤：**Elastic Cloud on Kubernetes (ECK)**，⽤户可使⽤该产品在 Kubernetes 上配置、管理和运⾏ Elasticsearch 集群

ECK 使⽤ Kubernetes Operator 模式 构建⽽成，需要安装在 Kubernetes 集群内，ECK 专注于简化所有后期运⾏⼯作，例 如：
+     管理和监测多个集群 
+     轻松升级⾄新的版本 
+     扩⼤或缩⼩集群容量 
+     更改集群配置 
+     动态调整本地存储的规模 
+     备份等等



[实验文档](https://github.com/wjunlove123/k8slearing/tree/main/log/efk)
