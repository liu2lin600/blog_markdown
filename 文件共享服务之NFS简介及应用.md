---
title: 文件共享服务之NFS简介及应用
date: 2016-07-21 20:04:53
categories: Linux服务
tags: [文件共享, nfs]
---

本文简要介绍nfs文件共享服务及基于nfs的web服务网页共享实验
<!--more-->

## 简介
> **NFS**(Network File System)即网络文件系统，通过网络让不同的机器、不同的操作系统能够彼此分享个别的数据，让应用程序在客户端通过网络访问位于服务器磁盘中的数据，是在类Unix系统间实现磁盘文件共享的一种方法。在实现文件传输需要依赖**RPC**，即远程过程调用。nfs服务在启动时，向rpc注册监听在某个端口上，rpc服务从还没被使用的端口中挑一个给此进程监听，换句话说，NFS是RPC服务器。当然，不但运行NFS的服务器需要启动RPC的服务，要挂载NFS文件系统的客户端，也需要同步启动RPC，这样服务器端与客户端才能由RPC的协议进程序端口的对应，Linux系统默认启动此服务

![nfs_rpc](nfs_rpc.png 'nfs工原理图')

以下实验环境为CentOS7.2

## nfs服务安装配置
```bash
yum install nfs-utils rpcbind   # rpcbind提供rpc服务，默认已安装，nfs-utils提供nfs服务所需程序包
systemctl start rpcbind nfs     # 开启服务
rpcinfo -p localhost            # 查看rpc服务监听端口及服务，可以指定地址查看该服务器上的共享
```

### 监听端口
1. nfs服务：2049/tcp, 2049/udp
2. rpc服务：111/tcp, 111/udp

### 相关进程
1. `rpc.mountd`：挂载守护进程，负责客户端来源认证进程，默认端口号随机，可在/etc/sysconfig/nfs中配置固定端口
2. `rpc.nfsd`：管理客户端是否能够使用服务器文件系统挂载信息等，其中还包含这个登入者的ID的判别
3. `rpc.idmap`：ID映射
4. `rpc.lockd`：文件锁，防止多人同时对文件操作造成问题，客户端和服务端同时开启
5. `rpc.statd`：检查文件的一致性，发生文件损毁时可尝试修复

### 权限问题
nfs的权限管理机制默认上以UID或GID映射，即要确保服务端和客户端上的存在相同用户且UID或GID也相机，否则映射时将会出现混乱，当然如果服务器与客户端之间存在中心身份认证服务器时，则不存在此问题，中心认证服务器包括NIS，Kerberos及LDAP等，此处暂不讨论。以下是几种映射情况：

1. Client和Server上同时存在lin00用户，且UID和GID都一致时，Client上以liu00用户对nfs共享文件的权限正常
2. Client上liu00用户ID为1001，Server上Tom用户ID为1001，此时Client上liu00用户将以Tom的权限访问nfs共享文件，从而造成一定的问题
3. Client上liu00用户ID为1001，Server上无UID为1001用户，此时liu00将被压缩成匿名用户访问，其UID为65534的nfsnobody
4. 默认情况下，root用户访问时也会被压缩成匿名用户访问，如需root权限要在定义nfs导出时添加`no_root_squash`选项

### 主配置
`/etc/exports`：在此文件中定义共享，导出时建议直接导出一个新分区

语法：`导出目录  客户端1(文件系统导出选项)  [客户端2(选项)...]`  
    
    /nfsshare  192.168.1.0/24(rw,no_root_squash)  172.16.60.0/24(rw)

- 客户端格式：
    指定IP：172.16.60.7
    指定域名：\*.liu2lin.com
    网络地址：172.16.0.0/16  192.168.1.0/255.255.255.0
    主机组：NIS域内主机组，@groups
    所有主机：使用\*通配所有

- 文件系统导出属性：
    `ro`：只读
    `rw`：读写
    `async`：异步,尽量使用异步
    `sync`：同步
    `root_squash`：压缩root用户，基于imapd，将root通过网络访问时映射为nfsnobody用户，默认启用
    `no_root_squash`：不压缩root用户
    `all_squash`：压缩所有用户
    `anonuid`，`anongid`：指定匿名用户映射为UID，GID

<font color=red>**注**：客户端上用户对共享文件的取决于导出属性及服务端上对应用户对导出目录下的文件的权限共同决定</font>

### 相关命令
1. `showmount`
    > `-a`: 在nfs服务器端显示所有的挂载会话
    > `-d SERVER_IP`: 服务器端执行，显示那个导出的文件系统被那些客户端挂载过
    > `-e SERVER_IP`: 客户端执行，探查某主机所导出的nfs文件系统

2. `exportfs`
    > `-ar`: 重新导出所有的文件系统
    > `-au`: 关闭导出的所有文件系统
    > `-u FS`: 关闭指定的导出的文件系统
    > `-v`: 显示详细过程

## 客户端访问
客户端启动rpcbind，以服务端(172.16.60.1)导出`/nfsshare`，客户端挂载到`/data/nfs`为例说明

1. 客户端挂载NFS文件系统：
        mount -t nfs 172.16.60.1:/nfsshare  /data/nfs

2. 客户端开机自动挂载nfs：`/etc/fstab`添加
        172.16.60.1:/nfsshare  /data/nfs  nfs  defaults,_netdev  0 0

## 实践