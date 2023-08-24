---
{"dg-publish":true,"permalink":"/k8s网络学习/网络基础/手动实现linux bridge/","dgPassFrontmatter":true}
---


##### 1、创建虚拟网桥并启动

```
ip link add br0 type bridge
ip link set br0 up
root@bpf3:~# brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.06689fa9d07c	no		
cni0		8000.02893b950fb5	no		veth0a0a1550
							vethcd12b400
docker0		8000.024299947d6e	no	
```

##### 2、添加网络名称空间ns1 ns2

```
root@bpf3:~# ip netns add ns1
root@bpf3:~# ip netns add ns2
```

##### 3、添加虚拟网卡，一端与vethpair相连，并将网卡指定命名空间

```
root@bpf3:~# ip link add veth0 type veth peer br-veth0
root@bpf3:~# ip link add veth1 type veth peer br-veth1
root@bpf3:~# ip link set veth0 netns ns1
root@bpf3:~# ip link set veth1 netns ns2
```

##### 4、查看名称空间下的IP信息

```
root@bpf3:~# ip netns exec ns1 ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: veth0@if9: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 72:1f:e4:71:f5:c0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

##### 5、将vethpair另一端与虚拟网桥br0相连

```
root@bpf3:~# ip link set br-veth0 master br0
root@bpf3:~# ip link set br-veth1 master br0
```

##### 6、查看虚拟设备绑定信息

```
root@bpf3:~# brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.06689fa9d07c	no		br-veth0
							br-veth1
cni0		8000.02893b950fb5	no		veth0a0a1550
							vethcd12b400
docker0		8000.024299947d6e	no	
```

##### 7、启动虚拟网卡veth0和veth1，并查看网络信息

```
root@bpf3:~# ip netns exec ns1 ip link set veth0 up
root@bpf3:~# ip netns exec ns2 ip link set veth1 up
root@bpf3:~# ip netns exec ns1 bash
root@bpf3:~# ifconfig
veth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 72:1f:e4:71:f5:c0  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

##### 8、进入名称空间，为虚拟网卡配置IP

```
root@bpf3:~# ip netns exec ns1 bash
root@bpf3:~# ifconfig veth0 192.168.1.61/24
root@bpf3:~# ip netns exec ns2 bash
root@bpf3:~# ifconfig veth1 192.168.1.62/24
root@bpf3:~# ifconfig
veth1: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.1.62  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 12:3a:90:ad:77:b2  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

##### 9、打开各名称空间的本地回环接口，测试本地协议栈是否连通

```
root@bpf3:~# ifconfig lo up
root@bpf3:~# ping 192.168.1.62
```

##### 10、若ping对端出现丢包，可判断为iptable已经drop，需要设置为接收

```
iptables -A FORWARD -i br0 -j ACCEPT
```