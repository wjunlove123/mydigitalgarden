---
{"dg-publish":true,"permalink":"/k8s网络学习/网络基础/手动实现简单CNI/","dgPassFrontmatter":true}
---

##### 1、分别在bpf1和bpf2节点创建两个容器

``` bash
bpf1:
root@bpf1:/tmp# docker run --name d1 -td burlyluo/nettoolbox  //创建d1容器
1440ca761e6c4ee66a1bc61e2eaf22bdd2b1147ae2735854e13283838fe37cec
bash-5.1# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
       
bpf2： //由于docker网卡虚拟IP都是172.17网段，所以在BPF2节点需要先创建172.18网段的网桥br18
root@bpf2:~# docker network create -d bridge --subnet 172.18.0.0/16 br18
03fb14c78c9015357e826e7b97d29623bfa3c10da7ff2569045d3a947c79c12a
root@bpf2:~# docker network list
NETWORK ID     NAME      DRIVER    SCOPE
03fb14c78c90   br18      bridge    local
9ec842480991   bridge    bridge    local
740a371ff932   host      host      local
3667c9da6f49   none      null      local
root@bpf2:~# docker run --name d2 --network br18 -td burlyluo/nettoolbox  //在新建的容器绑定在br18上
1928d376d8befbdb6212a6fd8374bc8ab55954b3d37bd846d1059bfa3882eb67
bash-5.1# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
12: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

##### 2、进入容器，互ping对方，由于没有路由信息，无法ping通

```bash
bash-5.1# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes{ #C}

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

##### 3、给bpf2节点添加一条到bpf1的路由信息

```bash
root@bpf2:~# route add -net 172.17.0.0/16 gw 172.27.137.128
root@bpf2:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.27.137.2    0.0.0.0         UG    100    0        0 ens33
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 ens33
172.17.0.0      172.27.137.128  255.255.0.0     UG    0      0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-03fb14c78c90
172.27.137.0    0.0.0.0         255.255.255.0   U     100    0        0 ens33
```

##### 4、给bpf1节点添加一条到bpf2的路由信息

```bash
root@bpf1:~# route add -net 172.18.0.0/16 gw 172.27.137.129
root@bpf1:/tmp# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.27.137.2    0.0.0.0         UG    100    0        0 ens33
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      172.27.137.129  255.255.0.0     UG    0      0        0 ens33
172.27.137.0    0.0.0.0         255.255.255.0   U     100    0        0 ens33
```

##### 5、再次进入容器验证是否能ping通对方