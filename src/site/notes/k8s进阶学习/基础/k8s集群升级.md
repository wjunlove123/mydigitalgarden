---
{"title":"k8s集群升级","dg-publish":true,"permalink":"/k8s进阶学习/基础/k8s集群升级/","dgPassFrontmatter":true}
---

##### kubernetes 1.24.8版本升级到1.25.9
升级步骤：
升级主管理节点→升级其他管理节点→升级工作节点
​首先备份主管理节点的etcd，检查版本号，为了保证版本的兼容性，跨度最好不要超过两个版本

###### 1. 查看K8S集群版本

```bash
[root@k8smaster ~]# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
k8smaster  Ready    control-plane   5d18h   v1.24.8
k8snode    Ready    <none>          5d18h   v1.24.8
```

###### 2. 查找可更新版本号
```bash
[root@master ~]# yum list --showduplicates kubelet kubeadm kubectl --disableexcludes=kubernetes | grep 1.25
kubeadm.x86_64                       1.25.0-0                        kubernetes 
kubeadm.x86_64                       1.25.1-0                        kubernetes 
kubeadm.x86_64                       1.25.2-0                        kubernetes 
kubeadm.x86_64                       1.25.3-0                        kubernetes 
kubeadm.x86_64                       1.25.4-0                        kubernetes 
kubectl.x86_64                       1.25.0-0                        kubernetes 
kubectl.x86_64                       1.25.1-0                        kubernetes 
kubectl.x86_64                       1.25.2-0                        kubernetes 
kubectl.x86_64                       1.25.3-0                        kubernetes 
kubectl.x86_64                       1.25.4-0                        kubernetes 
kubelet.x86_64                       1.25.0-0                        kubernetes 
kubelet.x86_64                       1.25.1-0                        kubernetes 
kubelet.x86_64                       1.25.2-0                        kubernetes 
kubelet.x86_64                       1.25.3-0                        kubernetes 
kubelet.x86_64                       1.25.4-0                        kubernetes
```
###### 3. 备份集群
如果不需要备份就跳过
备份地址下载：​ ​https://github.com/solomonxu/k8s-backup-restore​
```bash
mkdir /data
tar -xzvf k8s-backup-restore-1.1.tar
mv k8s-backup-restore-1.1 /data/k8s-backup-restore
cd /data/k8s-backup-restore
chmod +x bin/*.sh
./bin/k8s_backup.sh
```
###### 4. 预先下载需要的镜像
所有节点都需要下载
```bash
# 查看需要下载的镜像
[root@master ~]# kubeadm config images list --kubernetes-version=v1.25.9
registry.k8s.io/kube-apiserver:v1.25.9
registry.k8s.io/kube-controller-manager:v1.25.9
registry.k8s.io/kube-scheduler:v1.25.9
registry.k8s.io/kube-proxy:v1.25.9
registry.k8s.io/pause:3.8
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3

# 提前下载镜像
kubeadm config images pull 

# yum升级kubelet、kubeadm、kubectl
yum install kubeadm-1.25.9-0 kubectl-1.25.9-0 kubelet-1.25.9-0 --disableexcludes=kubernetes -y
#如果版本较高，可从高版本降级到低版本
yum downgrade kubeadm-1.25.9-0 kubectl-1.25.9-0 kubelet-1.25.9-0 --disableexcludes=kubernetes -y

# 查看升级后版本
[root@master ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.9", GitCommit:"a1a87a0a2bcd605820920c6b0e618a8ab7d117d4", GitTreeState:"clean", BuildDate:"2023-04-12T12:15:07Z", GoVersion:"go1.19.8", Compiler:"gc", Platform:"linux/amd64"}
```

###### 5. master节点升级
```bash
# 验证下载操作正常，并且 kubeadm 版本正确及升级计划
kubeadm version
kubeadm upgrade plan

# 集群升级，一个一个的升级，避免出现问题
# master集群节点升级 --ignore-daemonsets设置不可调度
kubectl drain k8smaster --ignore-daemonsets  # 驱逐排空要升级节点上的pod
kubeadm upgrade apply v1.25.9

# 升级完成取消对master的不可调度保护
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon master

# 其它master节点重复执行一遍就可以
```
###### 6. node工作节点升级
```bash
yum install kubeadm-1.25.9-0 kubectl-1.25.9-0 kubelet-1.25.9-0 --disableexcludes=kubernetes -y
# 在master上执行命令驱逐node1节点上的pod，并设置node1节点不可调度
kubectl drain k8snode --ignore-daemonset

# 在k8snode上执行命令升级
kubeadm upgrade node

# 重启kubelet
systemctl daemon-reload
systemctl restart kubelet

# 取消对节点的保护
kubectl uncordon k8snode
```
###### 7. 查看验证
```bash
[root@k8smaster ~]# kubectl get nodes
NAME        STATUS   ROLES           AGE    VERSION
k8smaster   Ready    control-plane   129d   v1.25.9
k8snode     Ready    <none>          129d   v1.25.9
```