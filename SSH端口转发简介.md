---
title: SSH端口转发简介
date: 2017-03-31 11:58:08
categories: Linux基础
tags: [ssh转发]
---

ssh除了用于安全的远程登录以外，还有个强大的功能，即**端口转发**，从而实现流往某端口的数据被加密后传向另一机器，这个过程形似构造了一条通道，因此也称之为SSH隧道(SSH Tunnel)
<!--more-->

## 端口转发类型

1. 动态端口转发 (socks代理)
1. 本地端口转发
1. 远程端口转发

**常用的命令如下：**

```
ssh -C -f -N -g -L listen_port:DST_Host:DST_port user@Tunnel_Host
ssh -C -f -N -g -R listen_port:DST_Host:DST_port user@Tunnel_Host
ssh -C -f -N -g -D listen_port user@Tunnel_Host
```

**相关参数说明：**

`-D`：动态转发
`-L`：本地转发
`-R`：远程转发
`-N`：不执行shell[与 -g 合用]
`-f`：将ssh转到后台运行，即认证之后，ssh自动以后台运行，不在输出信息
`-g`：允许打开的端口让远程主机访问
`-C`：压缩数据传输
`-T`：不分配tty，只做代理用

### 1. 动态转发

动态端口转发可用于socks代理，方法常常被用来突破防火墙的限制进行代理服务(翻墙)

**应用场景：**

1. 主机A(1.1.1.1)不能上google，主机B(2.2.2.2)可以上
1. 主机A申请一个socket监听端口(8080)
1. 主机A与主机B建立ssh隧道
1. 当主机A有请求8080, 请求数据会从ssh隧道发往主机B
1. 主机B收到数据, 根据数据的应用程序协议去发送数据到指定的地址
1. 返回数据会按原有路径返还给主机A
1. 在主机B上做相应的代理设置即可实现翻墙，此处不多做介绍

```
# 在A(1.1.1.1)上执行
ssh -D 127.0.0.1:8080 user@2.2.2.2
curl --socks5 127.0.0.1:8008 www.google.com 	# 验证代理是否成功
```

### 2. 本地转发

本地端口转发为通过SSH隧道，将一个远端机器能够访问到的地址和端口，映射为一个本地的端口，如下图

![本地转发](local port forwarding.png)

**应用场景：**

1. 如上图，主机A(client:1.1.1.1)，B(server:2.2.2.2)，C(host:10.0.0.1)
1. A可以连B，但不能连C，但B能连C
1. 绑定A的8080，与B构建一条SSH隧道
1. 当我们请求A的8080时，请求的数据通过SSH隧道到达B，B就会把数据发送到C的80上
1. 返回的数据按照原路返回
2. 可以将C上的http、ftp、ssh等服务转到A上访问

```
# 在A(1.1.1.1)上执行
ssh -L :8080:10.0.0.1:80 root@2.2.2.2
curl 127.0.0.1:8080
```

### 3. 远程转发

远程端口转发用于某些单向阻隔的内网环境，比如说NAT，网络防火墙。在NAT设备之后的内网主机可以直接访问公网主机，但外网主机却无法访问内网主机的服务。如果内网主机向外网主机建立一个远程转发端口，就可以让外网主机通过该端口访问该内网主机的服务

![远程转发](remote port forwarding.png)

**应用场景：**

1. 如上图，主机A(server:1.1.1.1)，B(client:2.2.2.2)，C(host:10.0.0.1)
2. A不能连B，B能连A，BC互联
3. 在B上创建ssh隧道，在A上绑定端口(8080)，将C中的80转出来
4. 当请求A的8080时，会将数据发送到C的80上，将数据返回给A

```
# 在B(2.2.2.2)上执行
ssh -R 8080:10.0.0.1:80 root@1.1.1.1

# 在A上验证
curl 127.0.0.1:8080
```

## 实际场景配置说明

### 1. 客户的7180只对 relay 机器的出口 IP 做白名单限制，可以采用socks代理方式进行访问

1. 在本机使用以下命令连接relay，将自己电脑的出口IP设置为relay的IP（123.59.61.252）
     
    ```
    ssh -fND 127.0.0.1:8800 liulinlin@relay.sensorsdata.cn
    ```

2. 认证是否代理成功，测试出口IP

    ```
    curl --socks5 127.0.0.1:8800 myip.ipip.net
    ```

3. Mac上设置：`系统偏好设置` -> `网络` -> `高级` -> `代理` -> `勾选socks代理` -> `并填写地址及端口(127.0.0.1:8800)` -> `OK`
4. 在电脑浏览器上访问 myip.ipip.net 确认IP是否为relay IP（123.59.61.252），如果有问题先检查下浏览器是否有用到其它代理











