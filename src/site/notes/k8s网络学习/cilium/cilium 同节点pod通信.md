---
{"dg-publish":true,"permalink":"/k8s网络学习/cilium/cilium 同节点pod通信/","dgPassFrontmatter":true}
---

#### 一、通过抓包分析查看

1. 在bpf3节点上的两个pod间（cni cni-sd8x8）进行ping抓包分析

==root@bpf1:~/.kube# kubectl exec -it cni – ping -c 1 10.0.2.239==

```
root@bpf1:~/ydzs# kubectl get pods -owide
NAME        READY   STATUS    RESTARTS   AGE    IP           NODE   NOMINATED NODE   READINESS GATES
cni         1/1     Running   0          36s    10.0.2.19    bpf3   <none>           <none>
cni-76fqx   1/1     Running   0          111s   10.0.0.127   bpf1   <none>           <none>
cni-sd8x8   1/1     Running   0          111s   10.0.2.239   bpf3   <none>           <none>
cni-zdfr6   1/1     Running   0          111s   10.0.1.109   bpf2   <none>           <none>
```

2. 查看pod cni的信息

```shell
#pod eth0网卡信息
root@bpf1:~/.kube# kubectl exec -it cni -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether ea:71:a3:82:c7:cc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.2.19/32 scope global eth0
    
#pod eth0网卡对应虚拟网桥对信息
root@bpf3:~# kubectl exec -it cni -- ethtool -S eth0
NIC statistics:
     peer_ifindex: 16  ##虚拟网桥对在宿主机上对应的index是16
     rx_queue_0_xdp_packets: 0
     rx_queue_0_xdp_bytes: 0
     rx_queue_0_drops: 0
     rx_queue_0_xdp_redirect: 0
     rx_queue_0_xdp_drops: 0
     rx_queue_0_xdp_tx: 0
     rx_queue_0_xdp_tx_errors: 0
     tx_queue_0_xdp_xmit: 0
     tx_queue_0_xdp_xmit_errors: 0

#BPF3宿主机上对应index 16的虚拟网桥对为lxc702aaab824ce
16: lxc702aaab824ce@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000  
    link/ether 0e:5e:41:ce:1b:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::c5e:41ff:fece:1bb8/64 scope link 
       valid_lft forever preferred_lft forever
```

3. 在cni的eth0、eth0对应的虚拟网桥对lxc702aaab824ce和cni-sd8x8的虚拟网桥对进行tcpdump抓包分析

```shell
#cni的eth0
15:02:42.712529 ea:71:a3:82:c7:cc > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.0.2.140 tell 10.0.2.19, length 28
15:02:42.712560 0e:5e:41:ce:1b:b8 > ea:71:a3:82:c7:cc, ethertype ARP (0x0806), length 42: Reply 10.0.2.140 is-at 0e:5e:41:ce:1b:b8, length 28
15:02:42.712563 ea:71:a3:82:c7:cc > 0e:5e:41:ce:1b:b8, ethertype IPv4 (0x0800), length 98: 10.0.2.19 > 10.0.2.239: ICMP echo request, id 7680, seq 0, length 64
15:02:42.712632 0e:5e:41:ce:1b:b8 > ea:71:a3:82:c7:cc, ethertype IPv4 (0x0800), length 98: 10.0.2.239 > 10.0.2.19: ICMP echo reply, id 7680, seq 0, length 64{ #C}

131 packets captured
131 packets received by filter
0 packets dropped by kernel
    
   
#cni的veth对lxc702aaab824ce
08:02:35.099040 0e:5e:41:ce:1b:b8 > 33:33:00:00:00:fb, ethertype IPv6 (0x86dd), length 202: fe80::c5e:41ff:fece:1bb8.5353 > ff02::fb.5353: 0*- [0q] 2/0/0 (Cache flush) PTR bpf3-267.local., (Cache flush) AAAA fe80::c5e:41ff:fece:1bb8 (140)
08:02:42.712533 ea:71:a3:82:c7:cc > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.0.2.140 tell 10.0.2.19, length 28
08:02:42.712559 0e:5e:41:ce:1b:b8 > ea:71:a3:82:c7:cc, ethertype ARP (0x0806), length 42: Reply 10.0.2.140 is-at 0e:5e:41:ce:1b:b8, length 28
08:02:42.712567 ea:71:a3:82:c7:cc > 0e:5e:41:ce:1b:b8, ethertype IPv4 (0x0800), length 98: 10.0.2.19 > 10.0.2.239: ICMP echo request, id 7680, seq 0, length 64{ #C}

130 packets captured
130 packets received by filter
0 packets dropped by kernel

#cni-sd8x8的veth对lxcf04c90721764
08:02:42.712604 da:9d:d9:e0:9b:86 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.0.2.140 tell 10.0.2.239, length 28
08:02:42.712612 32:61:c2:f7:7e:d6 > da:9d:d9:e0:9b:86, ethertype ARP (0x0806), length 42: Reply 10.0.2.140 is-at 32:61:c2:f7:7e:d6, length 28
08:02:42.712614 da:9d:d9:e0:9b:86 > 32:61:c2:f7:7e:d6, ethertype IPv4 (0x0800), length 98: 10.0.2.239 > 10.0.2.19: ICMP echo reply, id 7680, seq 0, length 64{ #C}

68 packets captured
68 packets received by filter
0 packets dropped by kernel
```

4. 可以发现宿主机上对应cni和cni-sd8x8的vethpair对，只有单发（cni的veth对lxc702aaab824ce） 和 单收（cni-sd8x8的veth对lxcf04c90721764），这是因为==通过bpf_redirect_peer()函数可以绕过veth pair ，直接到达pod的网卡==

```shell
08:02:42.712567 ea:71:a3:82:c7:cc > 0e:5e:41:ce:1b:b8, ethertype IPv4 (0x0800), length 98: 10.0.2.19 > 10.0.2.239: ICMP echo request, id 7680, seq 0, length 64

08:02:42.712614 da:9d:d9:e0:9b:86 > 32:61:c2:f7:7e:d6, ethertype IPv4 (0x0800), length 98: 10.0.2.239 > 10.0.2.19: ICMP echo reply, id 7680, seq 0, length 64
```

[bpf_redirect_peer（)]：同节点的Pod通信时候，Pod-A从自己的veth pair 的lxc出来以后直接送到Pod-B的veth pair的eth0网卡，而绕过Pod-B的veth pair的lxc网卡。

[bpf_redirect_neigh ()]：不同节点Pod通信的时候，Pod-A从自己的veth pair的lxc出来以后被自己的ingress方向上的from-container处理（vxlan封装或是native routing）然后tail call到vxlan处理然后再走bpf_redirect_neigh()到我们的物理网卡，然后到对端Pod-B所在的Node节点，此时再进行反向处理即可。这里有一点注意：就是from-container —> vxlan 封装 —> bpf_redirect_neigh () 处理

![img](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/1642326279126-af5d173e-375f-4e0d-bd2c-fcabe70ada24.png)

#### 二、通过cilium monitor -vv查看

![image-20220503171219814](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/image-20220503171219814.png)

2.1 从bpf3的cni 发送icmp报文给cni-sd8x8

```shell
root@bpf3:~# kubectl exec -it cni -- ping -c 1 10.0.2.7
PING 10.0.2.7 (10.0.2.7): 56 data bytes
64 bytes from 10.0.2.7: seq=0 ttl=63 time=0.282 ms

--- 10.0.2.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.282/0.282/0.282 ms
```

2.2 通过cilium monitor -vv将打印的日志存到same_pod.yaml

```shell

root@bpf3:/home/cilium# cilium monitor -vv > same_pod.yaml
level=info msg="Initializing dissection cache..." subsys=monitor
^Croot@bpf3:/home/cilium# 

```

2.3 将same_pod.yaml拷贝到本地查看

```python
root@bpf3:~# kubectl -n kube-system cp cilium-79zrm:/home/cilium/same_pod.yaml ./same_pod.yaml
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
tar: Removing leading `/' from member names
```

2.4 查看bpf3节点上的identity list

```python
root@bpf3:/home/cilium# cilium identity list
ID      LABELS
1       reserved:host
2       reserved:world
3       reserved:unmanaged
4       reserved:health
5       reserved:init
6       reserved:remote-node
9052    k8s:app=cni   #cni的identity
        k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=default
        k8s:io.kubernetes.pod.namespace=default
16955   k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default #cni-sd8x8的identity
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=default
        k8s:io.kubernetes.pod.namespace=default
        k8s:run=cni
64702   k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=coredns
        k8s:io.kubernetes.pod.namespace=kube-system
```

```python
##ping 请求对应的datapath  （identity=9052->identity=16955）
CPU 01: MARK 0x0 FROM 3455 DEBUG: Conntrack lookup 1/2: src=10.0.2.34:2560 dst=10.0.2.7:0  #pod 内部处理，此时的Conntrack lookup对应从cni出来的处理
CPU 01: MARK 0x0 FROM 3455 DEBUG: Conntrack lookup 2/2: nexthdr=1 flags=1
CPU 01: MARK 0x0 FROM 3455 DEBUG: CT verdict: New, revnat=0
CPU 01: MARK 0x0 FROM 3455 DEBUG: Successfully mapped addr=10.0.2.7 to identity=9052
CPU 01: MARK 0x0 FROM 3455 DEBUG: Conntrack create: proxy-port=0 revnat=0 src-identity=16955 lb=0.0.0.0
CPU 01: MARK 0x0 FROM 3455 DEBUG: Attempting local delivery for container id 613 from seclabel 16955
CPU 01: MARK 0x0 FROM 613 DEBUG: Conntrack lookup 1/2: src=10.0.2.34:2560 dst=10.0.2.7:0  #此时的Conntrack lookup对应进10.0.2.7的处理
CPU 01: MARK 0x0 FROM 613 DEBUG: Conntrack lookup 2/2: nexthdr=1 flags=0

##ping 回复对应的datapath  （identity=16955->identity=9052）
CPU 01: MARK 0x0 FROM 613 DEBUG: Conntrack lookup 1/2: src=10.0.2.7:0 dst=10.0.2.34:2560
CPU 01: MARK 0x0 FROM 613 DEBUG: Conntrack lookup 2/2: nexthdr=1 flags=1
CPU 01: MARK 0x0 FROM 613 DEBUG: CT entry found lifetime=16777757, revnat=0
CPU 01: MARK 0x0 FROM 613 DEBUG: CT verdict: Reply, revnat=0
CPU 01: MARK 0x0 FROM 613 DEBUG: Successfully mapped addr=10.0.2.34 to identity=16955
CPU 01: MARK 0x0 FROM 613 DEBUG: Attempting local delivery for container id 3455 from seclabel 9052
CPU 01: MARK 0x0 FROM 3455 DEBUG: Conntrack lookup 1/2: src=10.0.2.7:0 dst=10.0.2.34:2560
CPU 01: MARK 0x0 FROM 3455 DEBUG: Conntrack lookup 2/2: nexthdr=1 flags=0
```

#### 三、通过pwru 查看

3.1 发送icmp报文到

3.2 分别trace 同node上的ip，注意均需要使用源地址

```python
root@bpf1:/tmp# pwru --filter-dst-ip 10.0.1.75 --filter-proto icmp --output-tuple
2022/05/03 02:55:29 Attaching kprobes...
586 / 586 [--------------------------------------------------------------------------------------------] 100.00% 48 p/s
2022/05/03 02:55:41 Attached (ignored 13)
2022/05/03 02:55:41 Listening for events..
               SKB         PROCESS                     FUNC        TIMESTAMP
    0xffff8d9dc76f7200          [ping]           __ip_local_out    5113666766460 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]                ip_output    5113666784869 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]             nf_hook_slow    5113666787219 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping] __cgroup_bpf_run_filter_skb    5113666788979 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]     neigh_resolve_output    5113666798719 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]       __neigh_event_send    5113666800629 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]                skb_clone    5113666803339 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7a00          [ping]              consume_skb    5113666824609 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7a00          [ping]   skb_release_head_state    5113666826119 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]     neigh_resolve_output    5113666853519 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]               eth_header    5113666855089 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]                 skb_push    5113666856419 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]           dev_queue_xmit    5113666857619 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]      netdev_core_pick_tx    5113666858829 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]       netif_skb_features    5113666859999 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]  passthru_features_check    5113666861189 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]     skb_network_protocol    5113666862339 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]       validate_xmit_xfrm    5113666863539 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]      dev_hard_start_xmit    5113666864719 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]   skb_clone_tx_timestamp    5113666865929 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]        __dev_forward_skb    5113666867529 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]         skb_scrub_packet    5113666868679 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]           eth_type_trans    5113666869929 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]                 netif_rx    5113666871089 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]     tcf_classify_ingress    5113666892919 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]          skb_do_redirect    5113666930969 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]           dev_queue_xmit    5113666932989 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]             tcf_classify    5113666934669 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]      netdev_core_pick_tx    5113666937428 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]       netif_skb_features    5113666938778 10.0.0.252:0->10.0.1.75:0(icmp)
0xffff8d9dc76f7200          [ping]     skb_network_protocol    5113666940218 10.0.0.252:0->10.0.1.75:0(icmp)
```

![image.png](https://dennis-02.oss-cn-shenzhen.aliyuncs.com/img/20230725094806.png)
