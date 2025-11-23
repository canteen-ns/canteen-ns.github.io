---
title: AI辅助学习_Linux001_Linux网络接口与网络空间（一）
description: AI+Linux Learning
slug: AI_Linux001
date: 2025-10-12T15:00:00+08:00
#image: go.jpg
categories:
    - AI+
    - Linux
tags:
    - AI+Learning
#weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# AI 辅助学习_Linux_网络接口与网络空间（一）

这几天，在公司帮助同事查问题时，感觉对linux网桥相关的内容不熟悉，对于同事口中的br0、eth0这些东西感觉非常之陌生，虽然站主仅开发上层协议，对底层的数据转发，网络等内容并不了解，但是站主认为作为嵌入式领域开发，对这些底层知识应当还是需要有所了解，遂决定学习记录之。    

## Linux 网络基础
[linux 网络知识体系](../ai_linux000 "linux 网络知识体系")

### 网络接口 
linux 网络接口分为物理接口和虚拟接口（也叫逻辑接口）  

- 物理接口： 直接连接到物理网络的接口，如 eth0、wlan0 等。
- 虚拟接口： 不直接连接到物理网络的接口，用于配置和管理网络，如 lo（localhost 接口）、br0（网桥接口）等。

查看网络接口： 可以使用 `ifconfig` 或 `ip addr` 命令查看系统中的网络接口。  

#### 常见的物理接口
- 以太网接口：eth0/enp0s3/ens33等
- 光纤接口：常见类型：SFP/SFP+
- 无线接口：wlan0/wlp3s0

#### 常见的虚拟接口
- 网桥接口：br0
- 隧道接口：tun0/tap0
- 回环接口：lo
- VLAN接口：eth0.100
- 绑定接口：bond0

#### Linux网桥
工作在数据链路层（OSI第二层）的虚拟网络设备，基于mac地址学习，实现多个网络接口（不区分接口类型）的透明桥接

#### 网桥主要类型
- 基础类型：标准网桥
- 虚拟化专用：Open vSwitch
- 容器网络：Docker默认网桥
- 特殊用途：MACVLAN网桥

### 网络空间
linux 网络空间（network namespace）是一种将网络资源（如网络接口、路由表、防火墙规则等）隔离开来的机制。每个网络空间都有自己的网络栈，包括自己的IP地址、路由表、防火墙规则等。网络空间可以用于创建隔离的网络环境，例如在容器化环境中，每个容器都有自己的网络空间，容器之间的网络流量是隔离的。

## Linux 网络接口实验
本实验在wsl(windows subsystem for linux)中进行
```bash
canteen_ns@DESKTOP-CANTEEN-NS:~$ uname -a
Linux DESKTOP-CANTEEN-NS 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
canteen_ns@DESKTOP-CANTEEN-NS:~$
```
### 1、检测网络配置
- 查看网络接口
```bash
#使用更现代的ip命令，或者传统的ifconfig，此处仅以ip命令为例
canteen_ns@DESKTOP-CANTEEN-NS:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.255.255.254/32 brd 10.255.255.254 scope global lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```
接口状态: LOOPBACK,UP,LOWER_UP - 回环接口，已启用，链路正常

MTU: 65536 - 最大传输单元  

qdisc: noqueue - 无队列调度（回环接口不需要）   

状态: UNKNOWN - 正常状态   

IP地址配置: 
127.0.0.1/8 - 标准的本地回环地址 
10.255.255.254/32 - 额外的回环地址，可能是：
- 某些服务绑定的特定IP
- Docker 或容器网络配置
- 自定义的网络服务   

::1/128 - IPv6 回环地址

```bash
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:e5:66:c3 brd ff:ff:ff:ff:ff:ff
    inet 172.23.156.99/20 brd 172.23.159.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:fee5:66c3/64 scope link
       valid_lft forever preferred_lft forever
```
接口状态: BROADCAST,MULTICAST,UP,LOWER_UP - 支持广播、组播，已启用，链路正常

MTU: 1500 - 标准以太网MTU

qdisc: mq - 多队列调度

状态: UP - 接口正常运行

MAC地址：
- 00:15:5d:e5:66:c3 - 虚拟化环境的典型MAC地址（00:15:5d是Microsoft Hyper-V的OUI）

IPv4配置：
- 172.23.156.99/20 - 这是WSL2的NAT网络地址
- 网络: 172.23.144.0/20
- 可用范围: 172.23.144.1 - 172.23.159.254
- 广播: 172.23.159.255
- 你的IP: 172.23.156.99

IPv6配置：
 - fe80::215:5dff:fee5:66c3/64 - 链路本地地址（只在本地链路有效）

- 查看路由表
```bash
canteen_ns@DESKTOP-CANTEEN-NS:~$ ip route
default via 172.23.144.1 dev eth0 proto kernel
172.23.144.0/20 dev eth0 proto kernel scope link src 172.23.156.99
  ```
默认路由`default via 172.23.144.1 dev eth0 proto kernel`： 
   - default: 表示默认路由（0.0.0.0/0），所有不匹配其他路由的数据包都走这条路
   - via 172.23.144.1: 网关地址，这是数据包要发送到的下一跳
   - dev eth0: 使用 eth0 网络接口发送
   - proto kernel: 路由由内核自动设置
  
直连路由`172.23.144.0/20 dev eth0 proto kernel scope link src 172.23.156.99`：
   - 172.23.144.0/20: 这是 eth0 接口的网络地址范围
   - dev eth0: 使用 eth0 接口发送
   - proto kernel: 路由由内核自动设置
   - scope link: 这是一个直连路由，目标网络直接在本地链路可达
   - src 172.23.156.99: 当从这个网络发送数据包时，使用 172.23.156.99 作为源IP地址