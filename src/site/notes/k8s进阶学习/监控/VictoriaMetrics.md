---
{"title":"VictoriaMetrics","dg-publish":true,"permalink":"/k8s进阶学习/监控/VictoriaMetrics/","dgPassFrontmatter":true}
---

### 背景
通过学习prometheus的使用，了解了基本的PromSQL语句以及结合Grafana来进行监控图表展示，并通过Alertmanager来进行报警，结合这些工具已经可以搭建比较完整的监控报警系统，使用kube prometheus还可以搭建一站式的Kubernetes集群监控体系，但是单台的prometheus存在单点故障风险，随着监控规模的扩大，prometheus产生的数据量也会非常大，性能和存储都会面临问题。

### 高可用性
Prometheus采用pull机制获取监控数据，即使使用Push Gateway 对于Prometheus 也是pull，为了确保Prometheus 服务的可用性，我们可以部署多个Prometheus 实例，然后采用相同的metrics数据
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230725111348.png)
当一个实例down掉后 从LB里面自动剔除掉，而且还有负载均衡的作用，可以降低一个Prometheus的压力，但缺点比较明显，不满足数据一致性以及持久化问题（多个实例抓取相同监控指标，也不能保证抓取过来的值是一致的），不过对于监控报警来说，不会要求数据强一致性。所以这种场景适合保存短周期监控数据，监控规模不大的场景。

![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230725113239.png)
在给Prometheus配置上远程存储过后，就不用担心数据丢失的问题，即使当一个Prometheus实例宕机或者数据丢失过后，也可以通过远程存储的数据进行恢复。

### VictoriaMetrics简介
VictoriaMetrics是一个可水平扩容的本地全量持久化存储方案，主要优势如下：
+ 对外支持Prometheus相关的API，可直接用于Grafana作为Prometheus数据源使用 
+ 指标数据摄取和查询具备高性能和良好的扩扩展性，性能比InfluxDB TimescaleDB ⾼出 20 倍
+ 在处理⾼基数时间序列时，内存⽅⾯也做了优化，⽐ InfluxDB 少 10x 倍，⽐ Prometheus、Thanos 或 Cortex 少 7 倍
-  ⾼性能的数据压缩⽅式，与 TimescaleDB 相⽐，可以将多达 70 倍的数据点存⼊有 限的存储空间，与 Prometheus、Thanos 或 Cortex 相⽐，所需的存储空间减少 7 倍
- 针对具有⾼延迟 IO 和低 IOPS 的存储进⾏了优化
- 提供全局的查询视图，多个 Prometheus 实例或任何其他数据源可能会将数据摄取到 VictoriaMetrics 
- 操作简单
	- VictoriaMetrics 由⼀个没有外部依赖的⼩型可执⾏⽂件组成
	- 所有的配置都是通过明确的命令⾏标志和合理的默认值完成的
	- 所有数据都存储在 --storageDataPath 命令⾏参数指向的⽬录中
	- 可以使⽤ vmbackup/vmrestore ⼯具轻松快速地从实时快照备份到 S3 或 GCS 对象存储中
- ⽀持从第三⽅时序数据库获取数据源
- 由于存储架构，它可以保护存储在⾮正常关机（即 OOM、硬件重置或 kill -9）时 免受数据损坏
- 同样⽀持指标的 relabel 操作

### 架构
VM 分为单节点和集群两个⽅案，根据业务需求选择即可。单节点版直接运⾏⼀个⼆进制⽂ 件既，官⽅建议采集数据点(data points)低于 100w/s，推荐 VM 单节点版，简单好维护， 但不⽀持告警。集群版⽀持数据⽔平拆分
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230725145938.png)
主要包含的组件：
+     vmstorage：存储原始数据并返回指定标签过滤器在给定时间范围内的查询数据， 当 -storageDataPath 指向的⽬录包含的可⽤空间少于 - storage.minFreeDiskSpaceBytes 时，vmstorage 节点会⾃动切换到只读模 式，vminsert 节点也会停⽌向此类节点发送数据并开始将数据重新路由到剩余的 vmstorage 节点，默认端口为 8482
+     vminsert：接受摄取的数据并根据指标名称及其所有标签的⼀致性哈希将其分散存储到 vmstorage 节点，默认端口 8480
+     vmselect：通过从所有配置的 vmstorage 节点获取所需数据来执⾏查询，默认端口 8481
+     vmagent：⽤来收集指标数据然后存储到 VM 以及 Prometheus 兼容的存储系 统中，默认端口 8429
+     vmalert：报警相关组件，如果不需要告警功能可以不使⽤该组件，默认端口为 8880

### VictoriaMetrics Operator
kubernetes 中的Operator可以大大简化应用的安装、配置和管理，同样对于VictoriaMetrics官方也开发了一个对应的operator来进行管理  VictoriaMetrics Operator，它的设计和实现灵感来⾃Prometheus Operator，它是管理应⽤程序监控配置 的绝佳⼯具。
VictoriaMetrics Operator 定义了如下CRD：
-       VMServiceScrape：定义从 Service ⽀持的 Pod 中抓取指标配置
-       VMPodScrape：定义从 Pod 中抓取指标配置
-       VMRule：定义报警和记录规则
-       VMProbe：使⽤ blackbox exporter 为⽬标定义探测配置
此外该Operator 默认还可以识别Prometheus Operator中的ServiceMonitor、PodMonitor、PrometheusRule和Probe对象，还允许使用CRD对象来管理Kubernetes集群内的VM应用。



[实验文档](https://github.com/wjunlove123/k8slearing/tree/main/monitor/victoriametrics)