---
{"dg-publish":true,"permalink":"/k8s网络学习/calico/calico-bgp-rr/","dgPassFrontmatter":true}
---

架构图如下所示: 搭建的真实网络环境将模拟该topo，包括ip地址 ASN编号

该集群一共有4台节点，1master，3node，同时节点之间不在同一个网段
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230801101456.png)

#####  1. 创建集群：1-setup-k8s.sh
```bash
#!/bin/bash
date
set -v

# 1.prep noCNI env
cat <<EOF | kind create cluster --name=clab-calico-bgp-rr --image=kindest/node:v1.23.4 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "10.244.0.0/16"

nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: 10.1.5.10
        node-labels: "rack=rack0"
  
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: 10.1.5.11
        node-labels: "rack=rack0"
  
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: 10.1.8.10
        node-labels: "rack=rack1"

- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: 10.1.8.11
        node-labels: "rack=rack1"

  

containerdConfigPatches:
- |-
 [plugins."io.containerd.grpc.v1.cri".registry.mirrors."111.230.110.182:5000"]
    endpoint = ["http://111.230.110.182:5000"]
EOF


# 2.remove taints
controller_node_ip=`kubectl get node -o wide --no-headers | grep -E "control-plane|bpf1" | awk -F " " '{print $6}'`

kubectl taint nodes $(kubectl get nodes -o name | grep control-plane) node-role.kubernetes.io/master:NoSchedule-

kubectl get nodes -o wide
```
注意点:
-     注意集群版本相比之前的测试有所变动，本次演示使用的1.27的k8s，注意更换节点上的kubectl版本 
-     集群安装保留kube-proxy  
-     创建node的时候需要设置`nodeRegistration.kubeletExtraArgs.node-labels=rack=rackx`    
-     设置`nodeRegistration.kubeletExtraArgs.node-ip=10.1.5.10` 创建4个节点，构建每2个节点处于同一个二层。测试不同二层机器上面的pod如何互访

##### 2. 创建clab容器环境 
2-setup-clab.sh，vyos镜像可用: `docker pull burlyluo/vyos:1.4.0`
```bash
#!/bin/bash
set -v

brctl addbr br-leaf0
ifconfig br-leaf0 up
brctl addbr br-leaf1
ifconfig br-leaf1 up

cat <<EOF>clab.yaml | containerlab deploy -t clab.yaml -
name: calico-bgp-rr
topology:
  nodes:
    spine0:
      kind: linux
      image: 111.230.110.182:5000/vyos:1.2.8
      cmd: /sbin/init
      binds:
        - /lib/modules:/lib/modules
        - ./startup-conf/spine0-boot.cfg:/opt/vyatta/etc/config/config.boot
  
    spine1:
      kind: linux
      image: 111.230.110.182:5000/vyos:1.2.8
      cmd: /sbin/init
      binds:
        - /lib/modules:/lib/modules
        - ./startup-conf/spine1-boot.cfg:/opt/vyatta/etc/config/config.boot

    leaf0:
      kind: linux
      image: 111.230.110.182:5000/vyos:1.2.8
      cmd: /sbin/init
      binds:
        - /lib/modules:/lib/modules
        - ./startup-conf/leaf0-boot.cfg:/opt/vyatta/etc/config/config.boot

    leaf1:
      kind: linux
      image: 111.230.110.182:5000/vyos:1.2.8
      cmd: /sbin/init
      binds:
        - /lib/modules:/lib/modules
        - ./startup-conf/leaf1-boot.cfg:/opt/vyatta/etc/config/config.boot

    br-leaf0:
      kind: bridge
    br-leaf1:
      kind: bridge
  
    server1:
      kind: linux
      image: 111.230.110.182:5000/nettool:v1.0
      network-mode: container:control-plane
      exec:
      - ip addr add 10.1.5.10/24 dev net0
      - ip route replace default via 10.1.5.1
  
    server2:
      kind: linux
      image: 111.230.110.182:5000/nettool:v1.0
      network-mode: container:worker
      exec:
      - ip addr add 10.1.5.11/24 dev net0
      - ip route replace default via 10.1.5.1

    server3:
      kind: linux
      image: 111.230.110.182:5000/nettool:v1.0
      network-mode: container:worker2
      exec:
      - ip addr add 10.1.8.10/24 dev net0
      - ip route replace default via 10.1.8.1

    server4:
      kind: linux
      image: 111.230.110.182:5000/nettool:v1.0
      network-mode: container:worker3
      exec:
      - ip addr add 10.1.8.11/24 dev net0
      - ip route replace default via 10.1.8.1
   
  links:
    - endpoints: ["br-leaf0:br-leaf0-net0", "server1:net0"]
    - endpoints: ["br-leaf0:br-leaf0-net1", "server2:net0"]

    - endpoints: ["br-leaf1:br-leaf1-net0", "server3:net0"]
    - endpoints: ["br-leaf1:br-leaf1-net1", "server4:net0"]  

    - endpoints: ["leaf0:eth1", "spine0:eth1"]
    - endpoints: ["leaf0:eth2", "spine1:eth1"]
    - endpoints: ["leaf0:eth3", "br-leaf0:br-leaf0-net2"]

    - endpoints: ["leaf1:eth1", "spine0:eth2"]
    - endpoints: ["leaf1:eth2", "spine1:eth2"]
    - endpoints: ["leaf1:eth3", "br-leaf1:br-leaf1-net2"]  
EOF
```
部署完成后: `clab inspect -t clab.yaml` 可以看到8个容器
![1690808990357.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1690808990357.png)
同时我们的集群节点ip也被分配了
![1690809226726.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1690809226726.png)

需要注意的地方:

- `brctl addbr br-leaf0;ifconfig br-leaf0 up;brctl addbr br-leaf1;ifconfig br-leaf1 up
    创建这两个网桥主要是为了让kind上节点通过虚拟交换机连接到containerLab，为什么不直连接containerLab，如果`10.1.5.10/24`使用vethPair和containerLab进行连接，`10.1.5.11/24`就没有额外的端口进行连接
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731210121.png)
- `- ./startup-conf/spine0-boot.cfg:/opt/vyatta/etc/config/config.boot` vyos网关配置文件的编写
- `network-mode: container:control-plane` 关键配置，该配置就是为了复用节点网络，共享网络命名空间
- `- ip addr add 10.1.5.10/24 dev net0 && ip route replace default via 10.1.5.1` 该配置是为了设置节点上的业务网卡(参考以前集群节点ip一般都是172.18.0.2)，同时将默认路由的网关进行更改。

##### 3. 安装calico网络插件
- 将fullmesh模式disable ，**nodeToNodeMeshEnabled: false**（默认情况下将calico.yaml中CALICO_IPV4POOL_IPIP和CALICO_IPV4POOL_VXLAN均置为Never时，会走fullmesh），修改BGP配置
- 创建BGP对等体拓扑信息
```bash
#!/bin/bash

set -v

# 1. install CNI[Calico v3.23.2]
kubectl apply -f ./calico.yaml  
kubectl wait --timeout=600s --for=condition=Ready=true pods --all -A
  
# 1.2. disable bgp fullmesh
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: BGPConfiguration
  metadata:
    name: default
  spec:
    logSeverityScreen: Info
    nodeToNodeMeshEnabled: false
kind: BGPConfigurationList
metadata:
EOF
  
# 1.3. add() bgp configuration for the nodes
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"clab-calico-bgp-rr-control-plane","kubernetes.io/os":"linux","node-role.kubernetes.io/control-plane":"","node-role.kubernetes.io/master":"","node.kubernetes.io/exclude-from-external-load-balancers":"","rack":"rack0"}'
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: clab-calico-bgp-rr-control-plane
    kubernetes.io/os: linux
    node-role.kubernetes.io/control-plane: ""
    node-role.kubernetes.io/master: ""
    node.kubernetes.io/exclude-from-external-load-balancers: ""
    rack: rack0
  name: clab-calico-bgp-rr-control-plane
spec:
  addresses:
  - address: 10.1.5.10
    type: InternalIP
  bgp:
    asNumber: 65005
    ipv4Address: 10.1.5.10/24
  orchRefs:
  - nodeName: clab-calico-bgp-rr-control-plane
    orchestrator: k8s
status:
  podCIDRs:
  - 10.244.0.0/24
EOF
  
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"clab-calico-bgp-rr-worker","kubernetes.io/os":"linux","rack":"rack0"}'
  creationTimestamp: "2022-12-05T08:40:29Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: clab-calico-bgp-rr-worker
    kubernetes.io/os: linux
    rack: rack0
  name: clab-calico-bgp-rr-worker
spec:
  addresses:
  - address: 10.1.5.11
    type: InternalIP
  bgp:
    asNumber: 65005
    ipv4Address: 10.1.5.11/24  
  orchRefs:
  - nodeName: clab-calico-bgp-rr-worker
    orchestrator: k8s
status:
  podCIDRs:
  - 10.244.2.0/24
EOF
  
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"clab-calico-bgp-rr-worker2","kubernetes.io/os":"linux","rack":"rack1"}'
  creationTimestamp: "2022-12-05T08:40:29Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: clab-calico-bgp-rr-worker2
    kubernetes.io/os: linux
    rack: rack1
  name: clab-calico-bgp-rr-worker2
spec:
  addresses:
  - address: 10.1.8.10
    type: InternalIP
  bgp:
    asNumber: 65008
    ipv4Address: 10.1.8.10/24
  orchRefs:
  - nodeName: clab-calico-bgp-rr-worker2
    orchestrator: k8s
status:
  podCIDRs:
  - 10.244.3.0/24
EOF

cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"clab-calico-bgp-rr-worker3","kubernetes.io/os":"linux","rack":"rack1"}'
  creationTimestamp: "2022-12-05T08:40:29Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: clab-calico-bgp-rr-worker3
    kubernetes.io/os: linux
    rack: rack1
  name: clab-calico-bgp-rr-worker3
spec:
  addresses:
  - address: 10.1.8.11
    type: InternalIP
  bgp:
    asNumber: 65008
    ipv4Address: 10.1.8.11/24
  orchRefs:
  - nodeName: clab-calico-bgp-rr-worker3
    orchestrator: k8s
status:
  podCIDRs:
  - 10.244.1.0/24
EOF
  
# 1.4. peer to leaf0 switch
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rack0-to-leaf0
spec:
  peerIP: 10.1.5.1
  asNumber: 65005
  nodeSelector: rack == 'rack0'
EOF
  
# 1.5. peer to leaf1 switch
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rack1-to-leaf1
spec:
  peerIP: 10.1.8.1
  asNumber: 65008
  nodeSelector: rack == 'rack1'
EOF
```
以上所有步骤执行完，即完成了 calico BGP controlplane的搭建

登录leaf0交换机，查看BGP信息
![1690810764438.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1690810764438.png)

##### 4. Cilium BGP ControlPlane 特性验证

创建demo app
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: app
  name: app
spec:
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - image: 111.230.110.182:5000/nettool:v1.0
        name: nettoolbox
        env:
          - name: NETTOOL_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        securityContext:
          privileged: true
---
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  type: NodePort
  selector:
    app: app
  ports:
  - name: app
    port: 80
    targetPort: 80
    nodePort: 32000
```

在构建集群的时候，4个节点中，master 和worker1 处于同一二层网络平面，worker2和worker3 处于另外一个二层平面
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731215010.png)
1. control节点上pod 访问worker/worker2节点上的pod 进行测试，查看网络是否打通
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731220431.png)
可以发现跨节点进行访问时候，无轮同一网络平面还是不同网络平面都可以互通

另外通过在leaf0上分别对eth1（10.1.10.1） 和 eth2（10.1.12.1）抓包分析，发现eth1中只有request包，eth2上只有reply包，
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731221825.png)
将eth1 down掉后，eth2中既有request包，又有reply包，这是ECMP中的**load sharing（互为备份）**
![1690813615905.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1690813615905.png)


2. 访问service的IP时，发现无法访问
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731224232.png)
这是因为在节点上没有到10.96.0.0/16的路由
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731224440.png)

**通过advertisement将路由宣告**出去，再查看节点路由表
`calicoctl patch BGPConfig default --patch '{"spec": {"serviceClusterIPs": [{"cidr": "10.96.0.0/16"}]}}'`
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731224832.png)
发现多了一条到10.96.0.0的路由，再次访问service的ip，已经可以正常访问
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230731225117.png)
