---
title: AI辅助学习_Linux001_Linux网络接口与网桥
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

# AI 辅助学习_Linux_网络接口与网桥

这几天，在公司帮助同事查问题时，感觉对linux网桥相关的内容不熟悉，对于同事口中的br0、eth0这些东西感觉非常之陌生，虽然站主仅开发上层协议，对底层的数据转发，网络等内容并不了解，但是站主认为作为嵌入式领域开发，对这些底层知识应当还是需要有所了解，遂决定学习记录之。    

正好手头上有一块闲置的linux板子，可以用来实操（折腾 :)

## Linux 网络基础
[linux 网络知识体系](../ai_linux000 "linux 网络知识体系")

### 网络接口 
linux 网络接口分为物理接口和虚拟接口（也叫逻辑接口）  

- 物理接口： 直接连接到物理网络的接口，如 eth0、wlan0 等。
- 虚拟接口： 不直接连接到物理网络的接口，用于配置和管理网络，如 lo（localhost 接口）、br0（网桥接口）等。

查看网络接口： 可以使用 `ifconfig` 或 `ip addr` 命令查看系统中的网络接口。  
```bash
root@orangepi3b:~# ifconfig
eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 00:00:a4:0d:3f:df  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 41  

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 48719  bytes 147537675 (147.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 48719  bytes 147537675 (147.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.61  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 2409:8a4d:c2d:7730:c97:4554:e47e:f97c  prefixlen 64  scopeid 0x0<global>
        inet6 2409:8a4d:c2d:7730:72e8:4471:b9f8:4c40  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::ddf4:6f7d:2425:1a85  prefixlen 64  scopeid 0x20<link>
        ether e0:51:d8:67:79:68  txqueuelen 1000  (Ethernet)
        RX packets 282327  bytes 368152895 (368.1 MB)
        RX errors 0  dropped 1081  overruns 0  frame 0
        TX packets 96111  bytes 30822950 (30.8 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# ip addr

root@orangepi3b:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 00:00:a4:0d:3f:df brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether e0:51:d8:67:79:68 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.61/24 brd 192.168.1.255 scope global dynamic noprefixroute wlan0
       valid_lft 82167sec preferred_lft 82167sec
    inet6 2409:8a4d:c2d:7730:c97:4554:e47e:f97c/64 scope global temporary dynamic 
       valid_lft 86294sec preferred_lft 81952sec
    inet6 2409:8a4d:c2d:7730:72e8:4471:b9f8:4c40/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86294sec preferred_lft 86294sec
    inet6 fe80::ddf4:6f7d:2425:1a85/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```

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

### Linux网桥
工作在数据链路层（OSI第二层）的虚拟网络设备，基于mac地址学习，实现多个网络接口（不区分接口类型）的透明桥接

#### 网桥主要类型
- 基础类型：标准网桥
- 虚拟化专用：Open vSwitch
- 容器网络：Docker默认网桥
- 特殊用途：MACVLAN网桥