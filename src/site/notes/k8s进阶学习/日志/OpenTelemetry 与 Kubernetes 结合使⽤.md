---
{"dg-publish":true,"permalink":"/k8s进阶学习/日志/OpenTelemetry 与 Kubernetes 结合使⽤/","dgPassFrontmatter":true}
---




Kubernetes 以多种不同的⽅式暴露了许多重要的遥测数据。它具有⽤于许多不同对象的⽇志、事件和指标，以及 其⼯作负载⽣成的数据。 

为了收集这些数据，我们将使⽤ OpenTelemetry Collector。该收集器可以⾼效地收集所有这些数据。 

为了收集所有的数据，我们将需要安装两个收集器，⼀个作为 DaemonSet，⼀个作为 Deployment。收集器的 DaemonSet 将⽤于收集**服务、⽇志和节点**、Pod 和容器的指标，⽽ Deployment 将⽤于收集**集群的指标和事件**。

#### 指标采集

通过helm chart安装后，查看其配置文件

> `helm  upgrade --install opentelemetry-collector -f otel-collector-ds-values.yaml -n kube-otel  .`

> `kubectl get cm opentelemetry-collector-agent -n kube-otel -oyaml`

得到如下配置信息（已去除其它配置）

```yaml
exporters:
  logging:
    loglevel: debug
  prometheus:
    endpoint: 0.0.0.0:9090
    metric_expiration: 180m
    resource_to_telemetry_conversion:
      enabled: true
extensions:
  health_check: {}
  memory_ballast:
    size_in_percentage: 40
processors:
  batch: {}
  k8sattributes:
    extract:
      metadata:
      - k8s.namespace.name
      - k8s.deployment.name
      - k8s.statefulset.name
      - k8s.daemonset.name
      - k8s.cronjob.name
      - k8s.job.name
      - k8s.node.name
      - k8s.pod.name
      - k8s.pod.uid
      - k8s.pod.start_time
    filter:
      node_from_env_var: K8S_NODE_NAME
    passthrough: false
    pod_association:
    - sources:
      - from: resource_attribute
        name: k8s.pod.ip
    - sources:
      - from: resource_attribute
        name: k8s.pod.uid
    - sources:
      - from: connection
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
  metricstransform:
    transforms:
      action: update
      include: .+
      match_type: regexp
      operations:
      - action: add_label
        new_label: k8s.cluster.id
        new_value: abcd1234
      - action: add_label
        new_label: k8s.cluster.name
        new_value: youdian-k8s
receivers:
  hostmetrics:
    collection_interval: 10s
    root_path: /hostfs
    scrapers:
      cpu: null
      disk: null
      filesystem:
        exclude_fs_types:
          fs_types:
          - autofs
          - binfmt_misc
          - bpf
          - cgroup2
          - configfs
          - debugfs
          - devpts
          - devtmpfs
          - fusectl
          - hugetlbfs
          - iso9660
          - mqueue
          - nsfs
          - overlay
          - proc
          - procfs
          - pstore
          - rpc_pipefs
          - securityfs
          - selinuxfs
          - squashfs
          - sysfs
          - tracefs
          match_type: strict
        exclude_mount_points:
          match_type: regexp
          mount_points:
          - /dev/*
          - /proc/*
          - /sys/*
          - /run/k3s/containerd/*
          - /var/lib/docker/*
          - /var/lib/kubelet/*
          - /snap/*
      load: null
      memory: null
      network: null
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
  kubeletstats:
    auth_type: serviceAccount
    collection_interval: 20s
    endpoint: ${K8S_NODE_NAME}:10250
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
  prometheus:
    config:
      scrape_configs:
      - job_name: opentelemetry-collector
        scrape_interval: 10s
        static_configs:
        - targets:
          - ${env:MY_POD_IP}:8888
  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
service:
  extensions:
  - health_check
  - memory_ballast
  pipelines:
    logs:
      exporters:
      - logging
      processors:
      - k8sattributes
      - memory_limiter
      - batch
      receivers:
      - otlp
    metrics:
      exporters:
      - prometheus
      processors:
      - memory_limiter
      - metricstransform
      - k8sattributes
      - batch
      receivers:
      - otlp
      - hostmetrics
      - kubeletstats
      - prometheus
    traces:
      exporters:
      - logging
      processors:
      - k8sattributes
      - memory_limiter
      - batch
      receivers:
      - otlp
      - jaeger
      - zipkin
  telemetry:
    metrics:
      address: ${env:MY_POD_IP}:8888
```

从上⾯配置⽂件可以看出定义了 4 个接收器(receiver)：

+ hostmetrics 接收器 
+ kubeletstats 接收器 
- otlp 接收器 
- prometheus 接收器

4个处理器（processor）：
- batch 处理器
- memory_limiter 处理器
- k8sattributes 处理器
- metricstransform 处理器

2个导出器(exporter):
- logging 导出器
- prometheus 导出器

##### otlp接收器

otlp 接收器是在 OTLP 格式中收集跟踪、指标和⽇志的最佳解决⽅案。如果以其他格式发出应⽤程序遥测数据，那么收集器很有可能也有⼀个相应的接收器

- OTLP 是指 OpenTelemetry Protocol，即 OpenTelemetry 数据协议。OTLP 规范规定了客户端和服务采集端 之间的遥测数据的编码、传输和投送

- OTLP 在实现上分为 OTLP/gRPC 和 OTLP/HTTP
  - OTLP/HTTP 在数据传输的时候⽀持两种模式：⼆进制和 json
  - ⼆进制使⽤ proto3 编码标准，且必须在请求头中标注 Content-Type: application/x-protobuf
  - JSON 格式使⽤ proto3 标准定义的 JSON Mapping 来处理 Protobuf 和 JSON 之间的映射关系
  - JSON 格式使⽤ proto3 标准定义的 JSON Mapping 来处理 Protobuf 和 JSON 之间的映射关系
    - JSON 格式使⽤ proto3 标准定义的 JSON Mapping 来处理 Protobuf 和 JSON 之间的映射关系
    - 并发请求：客户端可以在服务端未回应前发送下⼀个请求，以此提⾼并发量。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
```

##### hostmetrics 接收器

hostmetrics 接收器⽤于收集主机级别的指标，例如 CPU 使⽤率、磁盘使⽤率、内存使⽤率和⽹络流量。配置如下所示:

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s
    root_path: /hostfs
    scrapers:
      cpu: null
      disk: null
      filesystem:
        exclude_fs_types:
          fs_types:
          - autofs
          - binfmt_misc
          - bpf
          - cgroup2
          - configfs
          - debugfs
          - devpts
          - devtmpfs
          - fusectl
          - hugetlbfs
          - iso9660
          - mqueue
          - nsfs
          - overlay
          - proc
          - procfs
          - pstore
          - rpc_pipefs
          - securityfs
          - selinuxfs
          - squashfs
          - sysfs
          - tracefs
          match_type: strict
        exclude_mount_points:
          match_type: regexp
          mount_points:
          - /dev/*
          - /proc/*
          - /sys/*
          - /run/k3s/containerd/*
          - /var/lib/docker/*
          - /var/lib/kubelet/*
          - /snap/*
      load: null
      memory: null
      network: null
```

collection_interval 指定了每 10 秒收集⼀次指标，并使⽤根路径 /hostfs 来访问主机⽂件系 统,hostmetrics 接收器包括多个抓取器，⽤于收集不同类型的指标:

- cpu 抓取器⽤于收集 CPU 使⽤率指标
- disk 抓取器⽤于收集磁盘使⽤率指标
- memory 抓取器⽤于收集内存使⽤率指标
- load 抓取器⽤于收集 CPU 负载指标

注意配置中的cpu:null并不是表示禁⽤cpu抓取，⽽是表示启⽤，如果不启⽤的话则不配置即可

##### kubeletstats 接收器

Kubelet Stats Receiver⽤于从 kubelet 上的 API 服务器中获取指标。通常⽤于收集与 Kubernetes⼯作负载相关 的指标，例如 CPU 使⽤率、内存使⽤率和⽹络流量。这些指标可⽤于监视 Kubernetes集群和⼯作负载的健康状况和性能。

Kubelet Stats Receiver 默认⽀持在端⼝ 10250 暴露的安全 Kubelet 端点和在端⼝ 10255 暴露的只读 Kubelet 端点

默认情况下，该收集器将收集来⾃**容器**、pod 和**节点**的指标。我们可以通过设置⼀个 metric_groups 来指定要收 集的数据来源，可以指定的值包括 container、pod 、node 和 volume

```yaml
receivers:
  kubeletstats:
      collection_interval: 10s
      auth_type: serviceAccount
      endpoint: ${K8S_NODE_NAME}:10250
      insecure_skip_verify: true
        metric_groups:
          - node
          - pod
```

K8S_NODE_NAME 环境变量在 Kubernetes 集群⾥⾯可以通过 Downward API 来注⼊

##### prometheus 接收器

Prometheus 接收器以 Prometheus 格式接收指标数据。该接收器旨在最⼤限度地成为 Prometheus 的替代品。它⽀持 scrape_config 中的全部 Prometheus 配置， 包括服务发现。就像在启动 Prometheus 之前在 YAML 配置⽂件中写⼊⼀样，例如：

> prometheus --config.file=prom.yaml

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: opentelemetry-collector
          scrape_interval: 10s
          static_configs:
            - targets:
              - ${env:MY_POD_IP}:8888
        - job_name: k8s
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels:
                [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              regex: "true"
              action: keep
          metric_relabel_configs:
            - source_labels: [__name__]
              regex: "(request_duration_seconds.*|response_duration_seconds.*)"
              action: keep
```

可以通过上⾯的配置来让收集器接收 Prometheus 的指标数据，使⽤⽅法和 Prometheus ⼀样，只需要 在 scrape_configs 中添加⼀个任务即可，这⾥添加的 opentelemetry-collector 任务，是去抓取 8888 端⼝的数据，⽽ 8888 端⼝就是 OpenTelemetry Collector 的端⼝，这个端⼝在 service.telemetry 中已经定义了，这样就可以通过该接收器来抓取 OpenTelemetry Collector 本身的指标数据

##### batch 处理器

批处理器接受追踪、指标或⽇志，并将它们分批处理。批处理有助于更好地压缩数据，并减少传输数据所需的外部 连接数量。该处理器⽀持基于⼤⼩和时间的批处理。批处理器应该在内存限制器（ memory_limiter ）以及任何其他采样处理 器之后的管道中定义。这是因为批处理应该在任何数据采样之后再发⽣

##### emory_limiter 处理器 

内存限制处理器⽤于防⽌收集器的内存不⾜情况。考虑到收集器处理的数据的数量和类型是环境特定的，并且收集 器的资源利⽤率也取决于配置的处理器，因此对内存使⽤情况进⾏检查⾮常重要。

 memory_limiter 处理器允许定期检查内存使⽤情况，如果超过定义的限制，将开始拒绝数据并强制 GC 减少内 存消耗。 memory_limiter 使⽤软内存限制和硬内存限制，硬限制始终⾼于或等于软限制。 

内存使⽤量随时间变化，**硬限制是进程堆分配的最⼤内存量**，超过此限制将触发内存限制操作。**软限制是内存使⽤量下降到硬限制以下的阈值**，恢复正常操作

##### k8sattributes 处理器 

Kubernetes 属性处理器允许使⽤ K8s 元数据⾃动设置追踪、指标和⽇志资源属性。当 k8sattributes处理器被应⽤于⼀个Kubernetes集群中的 Pod 时，它会从 Pod 的元数据中提取⼀些属性，例如 Pod 的名称、UID、启动时 间等其他元数据。这些属性将与遥测数据⼀起发送到后端，以便在分析和调试遥测数据时可以更好地了解它们来⾃哪个 Pod。

在 k8sattributes处理器中， pod_association 属性定义了如何将遥测数据与 Pod 相关联。例如，如果⼀个 Pod 发送了多个遥测数据，那么这些遥测数据将被关联到同⼀个 Pod 上，以便在后续的分析和调试中可以更好地了解 它们来⾃哪个Pod

```yaml
k8sattributes:
  extract:
    metadata: # 列出要从k8s中提取的元数据属性
      - k8s.namespace.name
      - k8s.deployment.name
      - k8s.statefulset.name
      - k8s.daemonset.name
      - k8s.cronjob.name
      - k8s.job.name
      - k8s.node.name
      - k8s.pod.name
      - k8s.pod.uid
      - k8s.pod.start_time
  filter: # 只有来⾃与该值匹配的节点的数据将被考虑。
    node_from_env_var: K8S_NODE_NAME
  passthrough: false # 表示处理器不会传递任何不符合过滤条件的数据。
  pod_association:
    - sources:
      - from: resource_attribute # from 表示规则类型
        name: k8s.pod.ip
    - sources:
      - from: resource_attribute # resource_attribute 表示从接收到的资源的属性列表中查找的属性名称
        name: k8s.pod.uid
    - sources:
      - from: connection
```

##### logging 导出器 

⽇志导出器，⽤于将数据导出到标准输出，主要⽤于调试阶段

##### prometheus 导出器 

该导出器可以指定⼀个端点，将从接收器接收到的指标数据通过这个端点进⾏导出，这样 Prometheus 只需要从这个端点拉取数据即可。⽽ prometheusremotewrite 导出器则是将指标数据直接远程写 ⼊到指定的地址，这个地址是⽀持 Prometheus 远程写⼊协议的地址

```yaml
prometheus:
  endpoint: 0.0.0.0:9090
  metric_expiration: 180m
  resource_to_telemetry_conversion:
    enabled: true
```

- endpoint ：指标将通过路径 /metrics 暴露的地址，也就是通过上⾯地址来访问指标数据，这 ⾥表示在 9090 端⼝来暴露指标数据
- metric_expiration（默认值= 5m）：定义了在没有更新的情况下暴露的指标的时间⻓度
- resource_to_telemetry_conversion （默认为 false）：如果启⽤为 true，则所有资源属性将默认转换为 指标标签