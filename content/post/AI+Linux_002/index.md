---
title: AI辅助学习_Linux002_Linux网络接口与网络空间（二）
description: AI+Linux Learning
slug: AI_Linux002
date: 2025-11-23T15:00:00+08:00
#image: go.jpg
categories:
    - AI+
    - Linux
tags:
    - AI+Learning
#weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# AI 辅助学习_Linux_网络接口与网络空间（二）

上文简要介绍了linux 网络接口相关内容，本文将继续介绍linux 网络空间相关内容。

## Linux 网络命名空间基础
在Linux中，网络命名空间（Network Namespace）是一种内核特性，允许您创建隔离的网络环境，每个环境都有自己的网络设备、IP地址、路由表、端口等。这对于容器技术（如Docker）和网络虚拟化非常有用

## veth pair
veth（虚拟以太网设备）对是 Linux 内核提供的一种虚拟网络设备，核心特性是成对出现、数据双向传递，可理解为 “虚拟网线”

一对设备的两端形成逻辑上的 “链路”，从一端发送的网络数据会直接传递到另一端，反之亦然。

核心作用：连接隔离的网络环境（如不同的网络命名空间），解决 “隔离环境间数据互通” 的问题。

## Linux 网络命名空间实验
### 网络命令空间基本操作
1. 创建网络命名空间
```bash
ip netns add <namespace_name>
```
2. 删除网络命名空间
```bash
ip netns del <namespace_name>
```
3. 列出所有网络命名空间
```bash
ip netns list
```
4. 在网络命名空间中执行命令
```bash
ip netns exec <namespace_name> <command>
```
5. 创建veth pair
```bash
ip link add <veth_name> type veth peer name <veth_name_peer>
```
6. 将veth pair 放入网络命名空间
```bash
ip link set <veth_name> netns <namespace_name1>
ip link set <veth_name_peer> netns <namespace_name2>
```
7. 为veth pair 两端配置IP地址
```bash
ip addr add <ip_address>/<netmask> dev <veth_name>
ip addr add <ip_address>/<netmask> dev <veth_name_peer>
```
8. 启动veth pair 两端
```bash
ip link set <veth_name> up
ip link set <veth_name_peer> up
```

### 在wsl2中实验

#### 使用veth pair 连接网络命名空间
```bash
# 增加网络命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns add ns1
[sudo] password for canteen_ns:
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns add ns2

# 列出所有网络命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns list
ns2
ns1

# 增加veth pair
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link add veth1 type veth peer name veth2

# 查看veth pair，此时属于默认命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:15:5d:ed:52:ec brd ff:ff:ff:ff:ff:ff
3: veth2@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ee:1e:d5:2a:53:ef brd ff:ff:ff:ff:ff:ff
4: veth1@veth2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2e:19:83:50:37:a3 brd ff:ff:ff:ff:ff:ff

# 将veth pair 放入网络命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set veth1 netns ns1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set veth2 netns ns2
canteen_ns@DESKTOP-CANTEEN-NS:~$

# 配置veth pair 两端IP地址
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip addr add 192.168.100.1/24 dev veth1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip addr add 192.168.100.2/24 dev veth2

# 启动veth pair 两端
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip link set veth1 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip link set lo up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip link set veth2 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip link set lo up
canteen_ns@DESKTOP-CANTEEN-NS:~$

# 测试网络连通性
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=0.041 ms
64 bytes from 192.168.100.2: icmp_seq=3 ttl=64 time=0.032 ms
64 bytes from 192.168.100.2: icmp_seq=4 ttl=64 time=0.036 ms
64 bytes from 192.168.100.2: icmp_seq=5 ttl=64 time=0.040 ms
64 bytes from 192.168.100.2: icmp_seq=6 ttl=64 time=0.043 ms
q64 bytes from 192.168.100.2: icmp_seq=7 ttl=64 time=0.034 ms
64 bytes from 192.168.100.2: icmp_seq=8 ttl=64 time=0.029 ms
^C
--- 192.168.100.2 ping statistics ---
8 packets transmitted, 8 received, 0% packet loss, time 7163ms
rtt min/avg/max/mdev = 0.029/0.036/0.043/0.004 ms

```

#### 使用网桥连接网络命名空间

```bash
# 增加网络命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns add ns1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns add ns2
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns add ns3

# 增加网桥
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link add br0 type bridge

# 启动网桥
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br0 up

# 查看接口
canteen_ns@DESKTOP-CANTEEN-NS:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:15:5d:ed:52:ec brd ff:ff:ff:ff:ff:ff
5: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 62:96:6a:b3:fc:d3 brd ff:ff:ff:ff:ff:ff

# 增加veth pair
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link add veth1 type veth peer br-veth1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link add veth2 type veth peer br-veth2
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link add veth3 type veth peer br-veth3

# 将veth pair 放入网络命名空间
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set veth1 netns ns1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set veth2 netns ns2
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set veth3 netns ns3
canteen_ns@DESKTOP-CANTEEN-NS:~$

# 配置veth pair 两端IP地址
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns3 ip addr add 10.0.0.3/24 dev veth3

# 启动veth pair 两端
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip link set veth1 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip link set veth2 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns3 ip link set veth3 up

# 启动回环接口
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns3 ip link set lo up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip link set lo up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip link set lo up

# 将veth pair 另一端接入网桥
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth1 master br0
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth2 master br0
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth3 master br0

# 启动网桥veth pair端口
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth1 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth2 up
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip link set br-veth3 up
canteen_ns@DESKTOP-CANTEEN-NS:~$

canteen_ns@DESKTOP-CANTEEN-NS:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:15:5d:ed:52:ec brd ff:ff:ff:ff:ff:ff
5: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 2a:5d:7e:bd:e8:8c brd ff:ff:ff:ff:ff:ff
6: br-veth1@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 32:73:08:fa:4c:d9 brd ff:ff:ff:ff:ff:ff link-netns ns1
8: br-veth2@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 2a:5d:7e:bd:e8:8c brd ff:ff:ff:ff:ff:ff link-netns ns2
10: br-veth3@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether a6:59:ec:71:36:3a brd ff:ff:ff:ff:ff:ff link-netns ns3

# 配置网桥IP地址
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip addr add 10.0.0.100/24 dev br0

# 测试各个命名空间连通性
canteen_ns@DESKTOP-CANTEEN-NS:~$ ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.073 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.047 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.035 ms
^C
--- 10.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2043ms
rtt min/avg/max/mdev = 0.035/0.051/0.073/0.015 ms
canteen_ns@DESKTOP-CANTEEN-NS:~$ ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.065 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.093 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.041 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.035 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3061ms
rtt min/avg/max/mdev = 0.035/0.058/0.093/0.022 ms
canteen_ns@DESKTOP-CANTEEN-NS:~$ ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.067 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.076 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.086 ms
^C
--- 10.0.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2029ms
rtt min/avg/max/mdev = 0.067/0.076/0.086/0.007 ms
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.083 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.040 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.137 ms
^C
--- 10.0.0.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3071ms
rtt min/avg/max/mdev = 0.040/0.077/0.137/0.038 ms
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.044 ms
^C
--- 10.0.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.044/0.046/0.049/0.002 ms
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.121 ms
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1017ms
rtt min/avg/max/mdev = 0.043/0.082/0.121/0.039 ms

# 查看命名空间路由表
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns1 ip route
10.0.0.0/24 dev veth1 proto kernel scope link src 10.0.0.1
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns2 ip route
10.0.0.0/24 dev veth2 proto kernel scope link src 10.0.0.2
canteen_ns@DESKTOP-CANTEEN-NS:~$ sudo ip netns exec ns3 ip route
10.0.0.0/24 dev veth3 proto kernel scope link src 10.0.0.3
```