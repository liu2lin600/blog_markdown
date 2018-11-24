---
title: Linux程序包管理之rpm
date: 2016-05-28 12:43:24
categories: Linux基础
tags: [rpm,程序包管理]
---

RPM包管理工具rpm
<!--more-->

## 简介
Linux下的程序安装包分为`二进制包(Binary)`和`源码包(Source)`，源代码包则需要编译安装，将在以后篇幅介绍。二进制包中的管理工具以Debian的deb包和RedHat的rpm包较为流行，这里主要介绍RPM的用法  
RPM是RPM Package Manager（RPM软件包管理器）的缩写，在多种Linux发行版中被采用，可以算是公认的行业标准了，默认以.rpm作为文件的后缀名 

### 家族
**Debian(.deb)**：Debian, Ubuntu, Xandros, Linspire  
**Red Hat(.rpm)**：RHEL, Fedora, CentOS, OpenSUSE, Mandriva, PCLinuxOS

### 工具
前端工具：yum(.rpm), apt-get(.deb), zypper(suse上rpm), dnf(Fedora 22+上rpm)  
后端工具：rpm, dpkg

### 命名
**源码包**：thrift-0.9.3.tar.gz 
```
thrift: 包名  
0.9.3:  版本信息，0为主版本号，9表示次版本信息，3为源码包发行号  
tar.gz：以gz格式压缩
```

**RPM包**：httpd-devel-2.4.6-40.el7.centos.x86_64.rpm
```
httpd: 主包名
devel: 分包名，如果是主包则无此项
2.4.6: 版本信息，2为主版本号，0表示次版本信息，35为源码包发行号，也叫修订号
40.el7: 发行信息，40表示rpm的发行号，el6表示REHL7
x86_64: 适用平台，常有i386-i686,x86_64,noarch,powerpc,ppc
rpm: 后缀名
```
### 获取软件包的途径：
1. 系统发版的光盘或官方的服务器  
<http://mirrors.aliyun.com>  
<http://mirrors.163.com>
2. 项目官方站点  
<https://thrift.apache.org/>  
<http://redis.io/>
3. 第三方组织（EPEL）  
<http://pkgs.org>  
<http://rpmfind.net>  
<http://rpm.pbone.net>
4. 自己制作rpm包

## 用法
说明：  
- PACKAGE_FILE：包文件名，如zsh-5.0.2-14.el7.x86_64.rpm  
- PACKAGE_NAME：包名，如zsh

主要用法如下，其中查询功能为学习重点  
- 安装：rpm -i [install-options] PACKAGE_FILE
- 升级：rpm -U|F [install-options] PACKAGE_FILE  
- 卸载：rpm -e [--allmatches] [--nodeps] [--noscripts] [--test] PACKAGE_NAME
- 查询：rpm -q [select-options] [query-options] PACKAGE_NAME
- 校验：rpm -V [select-options] [verify-options] PACKAGE_NAME

### 安装
`rpm -i [install-options] PACKAGE_FILE`  
> `-i`: 主选项安装  
> `-v`：显示安装详情  
> `-h`: 用#显示进度条  
>> `--test`: 测试安装，查看是否有依赖关系等  
>> `--replacepkgs`: 重新安装  
>> `--nodeps`: 忽略依赖安装，不推荐  
>> `--noscripts`：不执行程序包脚本片段，包括安装前%pre, 安装后%post, 卸载前%preun, 和卸载后%postun脚本  
>> `--nosignature`: 不检测数字签名  

- 注意：很多程序存在依赖关系，RPM方式不能很好解决，需手动安装所依赖的程序，有时会出现环形依赖，即a->b->c->a，解决相对比较麻烦
![rpm包安装](rpmInstall.png)

### 升级
`rpm -U [install-options] PACKAGE_FILE`   
`rpm -F [install-options] PACKAGE_FILE`
> `-U`: 主选项升级，存在老版本则升级，否则，则安装    
> `-F`: 主选项升级，存在老版本则升级，否则，则退出    
> `-v`：显示升级详情  
> `-h`: 用#显示进度条  
>> `--oldpackage`: 降级  
>> `--force`: 强制升级

![rpm包升级](rpmUpdate.png)
- 注意
1. 不要对内核做升级操作；Linux支持多内核版本并存，因此，将直接安装新版本内核
2. 如果某原程序包的配置文件安装后曾被修改过，升级时，新版本的程序提供的同一个配置文件不会覆盖原有版本的配置文件，而是把新版本的配置文件重命名(FILENAME.rpmnew)后提供

### 卸载
`rpm -e [--allmatches] [--nodeps] [--noscripts] [--test] PACKAGE_NAME`
> `-e`: 主选项卸载  
>> `--allmatches`: 卸载所有匹配的指定程序包   
>> `--nodeps`: 忽略依赖卸载   
>> `--test`: 测试卸载  
>> `--noscripts`: 

![rpm卸载](rpmErase.png)

### <font color='red'>查询</font>
`rpm -q [select-options] [query-options] PACKAGE_NAME`
`rpm -qp[dil] PACKAGE_FILE`
#### 已安装查询
> `-q`：查询指定的包是否已经安装
> `-qa`: 查询已经安装的所有包
> `-qf file`: 查询指定的文件是由哪个rpm包安装生成的  
> `-q --whatprovides CAP`: 指定功能由哪个程序提供  
> `-q --whatrequires CAP`: 指定功能由哪个包所依赖  
> `-q --provice`: 程序包提供哪个CAPABILITY(功能)  
> `-qi`: 查询指定包的说明信息  
> `-ql`: 查询指定包安装后生成的文件列表  
> `-qc`：查询指定包安装的配置文件  
> `-qd`: 查询指定包安装的帮助文件  
> `-q` --changelog: 程序包更新历史信息  
> `-qR`: 依赖关系  
> `-q --script`: 相应执行脚本  

#### 未安装 
> `-qpl`: 查询指定包安装后生成的文件列表  
> `-qpi`: 查询指定包说明信息  
> `-qpd`: 查询指定包安装后生成的文档  

![rpm查询](rpmQuery.png)

### 校验
`rpm -K PACKAGE_FILE`
`rpm -V PACKAGE_NAME`
> -K --nosignature: 忽略验证来源合法性，也即验正签名(dsa, gpg)  
> -K --nodigest：忽略验证软件包完整性(sha1, md5)  
> --import: 导入密钥  

- rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release  
导入密钥文件

![rpm校验](rpmVertify.png)
验证内容中的9位信息的具体内容如下：
```
S   Size 大小是否改变
M   Mode 类型或文件的权限是否改变
5   MD5 MD5效验和是否改变
D   Device 设备主从码是否改变
L   readLink 路径是否改变
U   User 属主是否改变
G   Group 属组是否改变
T   mTime 修改时间是否改变
P   caPabilities 功能是否变化
```

### 数据库管理
rpm管理器数据库路径`/var/lib/rpm/`，通过rpm 命令查询一个rpm 包是否安装了，也是通过查询rpm 数据库来完成的，当然没事别进行此操作
> --initdb：事先存在数据库，则不执行任何操作，否则，新建  
> --rebuilddb：直接重建，覆盖原有的数据库  
> –dbpath：指明要创建数据库的目录

以上内容整理自马哥笔记

