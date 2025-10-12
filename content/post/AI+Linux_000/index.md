---
title: AI辅助学习_Linux000_网络知识体系
description: AI+Linux Learning Networking Summary
slug: AI_Linux000
date: 2025-10-12T14:00:00+08:00
#image: go.jpg
categories:
    - AI+
    - Linux
tags:
    - AI+Learning
#weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# Linux网络知识体系
## 核心知识领域：

### 网络配置管理
```bash
# 持久化配置（Ubuntu）
sudo nano /etc/netplan/01-netcfg.yaml
```
### 接口管理

```bash
ip link set eth0 mtu 1500  # 修改MTU值
ethtool -s eth0 speed 1000 duplex full  # 设置网卡参数
```
### 路由与转发
```bash
sysctl -w net.ipv4.ip_forward=1  # 启用IP转发
ip route add 10.0.0.0/24 via 192.168.1.1 metric 100
```
### 防火墙与安全

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
nft add table inet filter
```
### 网络服务
```bash
systemctl status sshd  # 服务状态查看
journalctl -u nginx --since "1 hour ago"  # 日志追踪
```
## 扩展领域：

- 容器网络（Docker bridge/CNI）
- 虚拟化网络（Libvirt/KVM）
- 无线网络管理（iwconfig/wpa_supplicant）
- 网络调试（tcpdump/tshark）