---
{"dg-publish":true,"permalink":"/k8s网络学习/网络基础/CNI/","dgPassFrontmatter":true}
---

### 简介
- CNCF项目，提供实现无关的规范来创建配置容器网络（不仅仅用于k8s，包括Mesos，CloudFoundry，Podman，CRI-O）
- 定义执行流程和配置格式

### 主要功能
- 创建网络接口，连接Pod之间的网络。通过overlay、route & underlay etc（不通过NAT）
- 进行IP地址管理，分配Pod IP地址
- 提供网络安全策略

### 工作机制
- 通过JSON配置文件定义网络配置；/etc/cni/net.d/xxnet.conf
- 通过调用可执行程序（CNI插件）来对容器网络执行配置；/opt/cni/bin/xxnet
- 通过链式调用的方式来支持多插件的组合使用
- 调用过程：kubelet->CRI->CNI->读取配置->执行二进制插件

Pod网络创建过程
![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230730113525.png)
