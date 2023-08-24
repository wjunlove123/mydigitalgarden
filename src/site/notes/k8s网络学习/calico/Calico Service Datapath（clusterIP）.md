---
{"dg-publish":true,"permalink":"/k8s网络学习/calico/Calico Service Datapath（clusterIP）/","dgPassFrontmatter":true}
---


该集群一共有3台节点，1master，2node，同时节点之间不在同一个网段

#####  1. 创建集群：1-setup-k8s.sh
```shell
#!/bin/bash
date
set -v
  
# 1.prep noCNI env
cat <<EOF | kind create cluster --name=calico-fullmesh --image=kindest/node:v1.23.4 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
        disableDefaultCNI: true
nodes:
        - role: control-plane
        - role: worker
        - role: worker
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."111.230.110.182:5000"]
    endpoint = ["http://111.230.110.182:5000"]
EOF
  
# 2.remove taints
controller_node_ip=`kubectl get node -o wide --no-headers | grep -E "control-plane|bpf1" | awk -F " " '{print $6}'`
kubectl taint nodes $(kubectl get nodes -o name | grep control-plane) node-role.kubernetes.io/master:NoSchedule-
kubectl get nodes -o wide

# 3. install CNI[Calico v3.23.2]
kubectl apply -f calico.yaml
```

##### 2. 创建应用验证 （clusterIP）
应用资源包括客户端POD，后端POD以及后端POD对应的Service
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: backend
  name: backend
spec:
  containers:
  - name: backend
    image: 111.230.110.182:5000/nettool:v1.0
  restartPolicy: Always
  nodeName: calico-fullmesh-worker
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: client
  name: client
spec:
  containers:
  - name: client
    image: 111.230.110.182:5000/nettool:v1.0
  restartPolicy: Always
  nodeName: calico-fullmesh-control-plane 
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: backend
  name: backend-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: backend
```
![1691051292657.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691051292657.png)

1. 从客户端POD 的eth0抓包，分析源IP为POD IP，目的IP为service IP
![1691051937431.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691051937431.png)

2. 从客户端POD所在节点的eth0抓包，目的IP已经变为**10.244.84.196**，这是由于源POD 发出的数据包经过veth pair对送到root namespace后，**在kube-proxy所加载的iptables规则下，将目的IP更换为后端真实的POD IP**
![1691052936948.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691052936948.png)

数据包流向路径：
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230803154831.png)
在典型的Kubernetes部署中，kube-proxy在每个节点上运行，并负责拦截对集群IP地址的连接，并在支持每个服务的一组Pod之间进行负载均衡。作为此过程的一部分，使用DNAT将目标IP地址从Cluster IP映射到所选的后备Pod。然后，在连接上返回时，响应数据包会通过NAT反向回到发起连接的Pod。
重要的是，网络策略是基于 Pod 而不是服务 Cluster IP 来执行的。（即在 DNAT 改变连接目标 IP 为所选服务支持的 Pod 后，出口网络策略将对客户端 Pod 执行。由于只有连接的目标 IP 发生了改变，因此后台 Pod 的入口网络策略将把原始客户端 Pod 视为连接的源头）

##### 3. 创建应用验证 （NodePort）

![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230803174227.png)
最基本的从集群外部访问服务的方式是使用 NodePort 类型的服务。Node Port 是在集群中每个节点上保留的端口，通过该端口可以访问服务。在典型的 Kubernetes 部署中，kube-proxy 负责拦截对 Node Port 的连接，并将其负载均衡到支持每个服务的 Pod 上。
作为这个过程的一部分，NAT 用于将目标 IP 地址和端口从节点 IP 和 Node Port 映射到选择的后备 Pod 和服务端口。此外，源 IP 地址也会从客户端 IP 映射到节点 IP，以便响应数据包通过原始节点回流，其中 NAT 可以被撤销。（执行 NAT 的节点具有反转 NAT 所需的连接跟踪状态。）

![1691063300742.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691063300742.png)
![1691063438918.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691063438918.png)

此时，后端pod在worker节点上，通过访问control-plane节点IP+nodeport让请求流转至work节点下的backend pod

在control-plane节点的eth0抓包，
![1691064100817.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1691064100817.png)
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230803200224.png)

数据流量路径：
1. 外部client IP为172.18.0.1，先和control-plane节点（172.18.0.4）进行tcp三次握手，见如下图红色部分
2. control-plane->pod也需要经过三次握手，见如下图蓝色部分
3. 完成握手后，get请求包中srcIP=172.18.0.1，DstIP=172.18.0.4，经过control-plane节点nat后，变为srcIP=172.18.0.4，DstIP=10.44.84.196，回包也是经过reverse nat后返回给client
![2023-08-03_1.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/2023-08-03_1.png)
