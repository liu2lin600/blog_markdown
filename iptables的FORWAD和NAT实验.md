---
title: iptables的FORWAD和NAT实验
date: 2016-07-24 07:05:36
categories: Linux服务
tags: [iptables,FORWARD,NAT]
---

在虚拟机中模拟实现iptables中的FORWARD及NAT功能

<!-- more -->


直接上此次实验的环境示意图：
![f](forward_nat.png)

环境说明

1. **系统**：CentOS 7.2.x86_64主机3台，关闭firewalld服务，同时清空iptables规则
2. 主机B上配2块网卡，通过vmnet2与A连接，与C直接桥接
3. 主机B充当转发和地址转换，开启核心转发功能(ip_forward)
4. 将A的默认网关指向B

## FORWARD(转发)

### 实验前配置
1. B主机开启核心转发功能，同时确保模块`nf_conntrack`处于开启状
        echo 1 > /proc/sys/net/ipv4/ip_forward
        lsmod | grep nf_conntrack

2. A主机默认网关指向B
        route add default gw 172.16.60.12

3. C主机添加一条网络路由到B主机
        route add -net 172.16.0.0/16 gw 192.168.1.71 

3. 测试A、B、C之间是否都能ping通

### 实验要求
A主机开启web,ftp,ssh服务，确保外网C主机可以访问

```bash
# 设置FORWARD默认规则为DROP，此时AC间不能进行任何通信
iptables -P FORWARD DROP

# 开放状态为已建立的连接及有关联连接
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# 开启A主机访问外部网络，此时外网不能访问A主机
iptables -A FORWARD -s 172.16.60.11 -m state --state NEW -j ACCEPT

# 开放A主机21,22,23,80端口
iptables -A FORWARD -d 172.16.60.11 -p tcp -m multiport --dports 21,22,80 -m state --state NEW -j ACCEPT

# B主机加载ftp模块，确保可以追踪ftp状态
modprobe nf_conntrack_ftp

# 在C主机测试
ftp 172.16.60.11
curl 172.16.60.11
ssh root@172.16.60.11
```

## NAT(地址转换)

### 实验前配置
1. 删除上实验中为C主机添加的到B主机网络路由
        route del -net 172.16.0.0/16

2. 清空所有规则链，并确保默认策略为ACCEPT
        iptables -F

### SNAT
**源地址转换**用于让内网主机访问互联网，仅作用于nat表上POSTROUTING，INPUT，同时也达到隐藏内网ip的作用

**实验要求**：内网主机A访问外网C主机，通过B主机的源地址转换，实现B代理访问C，同时隐藏内网ip。前提确保B主机能够正常访问C

```bash
# 设置转换
iptables -t nat -A POSTROUTING -s 172.16.60.11 -j SNAT --to-source 192.168.1.71

# A主机访问C主机web服务
curl 192.168.1.72

# 在C主机上查看访问日志
tail /var/log/httpd/access_log

    # 访问主机显示B主机IP
    192.168.1.71 - - [24/Jul/2016:11:17:10 +0800] "GET /index.html HTTP/1.1" 200 10
```


### DNAT
**目标地址转换**用于让互联网上主机访问本地内网上的某服务器上的服务，仅作用于nat表上PREROUTING和OUTPUT，同时也有端口转换作用

**实验要求**：内网主机的httpd服务监听在8080端口，让C访问主机B的80时转换到A主机的8080上，B上没有httpd服务，只负责地址转换

```bash
# 设置转换
iptables -t nat -A PREROUTING -d 192.168.1.71 -p tcp --dport 80 -j DNAT --to-destination 172.16.60.11:8080

# C主机访问测试
curl 192.168.1.71

# A主机上查看http访问日志
tail /var/log/httpd/access_log

    # 正常记录下C主机IP
    192.168.1.72 - - [24/Jul/2016:10:44:14 +0800] "GET / HTTP/1.1" 200 9 "-" "curl/7.29.0"
```










