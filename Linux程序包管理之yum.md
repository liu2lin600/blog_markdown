---
title: Linux程序包管理之yum
date: 2016-06-03 13:37:35
categories: Linux基础
tags: [yum,程序包管理]
---

简要介绍yum的常见用法
<!-- more -->
## 简介
>　　yum(Yellow dog Updater, Modified)是RedHat系统的前端的软件包管理工具，基于对RPM包的管理，能够根据配置文件从指定的服务器自动下载RPM包进行安装，并自动处理依赖性关系，从而在一定程度上解决了在手动安装RPM包时烦人的依赖问题   
　　yum依赖于基于C/S架构的文件服务器，Server端可以是来自互联网，也可以来自局域网，这些文件服务器用来存放yum在安装程序包和所依赖的各种程序包，根据客户端的配置文件提供的多个仓库地址找到该仓库服务器进行查询，仓库服务器找到后会通过下载协议把相关文件包下载到本地的缓存(含元数据和程序包)目录中，并进行安装操作，之后再删除相关缓存。  
　　不过在安装有依赖程序过程中如果发生断电等非正常情况时，yum将很难在下次安装时解决此问题，新一代的管理工具dnf在这方面已经有比较好的处理方法，相信dnf取代yum应该也是迟早的事，但目前yum还是比较普遍的解决方式，而且dnf的用法基本跟yum一样，学好yum为了将来更好的使用dnf......

## 配置文件
**/etc/yum.conf**：全局配置文件  

```bash
[main]
cachedir=/var/cache/yum/$basearch/$releasever   #yum下载RPM包时的缓存目录
keepcache=0                                     #缓存是否需要保存 1表示保存 0表示不保存
debuglevel=2                                    #调试级别 默认为2
logfile=/var/log/yum.log                        #定义yum默认日志文件名称
exactarch=1                                     #是否只升级与和安装软件包的cpu体系一致的包
obsoletes=1                                     #是否允许更新陈旧的RPM包 1允许，2不允许
gpgcheck=1                                      #是否执行GPG签名检查，1执行，0不执行
plugins=1                                       #是否允许使用插件 1允许 0不允许
installonly_limit=5                             #允许保留多少个内核包
......

```
**/etc/yum.repos.d/*.repo**：各yum仓库相关配置  
　　默认生成的有：CentOS-Base.repo(默认启用)、CentOS-Debuginfo.repo、CentOS-fasttrack.repo、CentOS-Media.repo、CentOS-Vault.repo等，可以手动创建以`.conf`结尾文件指定相应仓库，**设置<font color=red>enabled=1</font>或不写则开启此仓**

```bash
[base]                                  #repo-ID，base为基础仓，默认还有updates、extras等
name=                                   #yum源的说明信息
mirrorlist=                             #yum源的镜像地址
#baseurl={http://|ftp://|file://}       #yum源的真正地址，为repodata目录所在目录路径，可多个路径。与mirrorlist二选一
enabled=1                               #1为生效，0不生效，不写默认为1
gpgcheck=1                              #是否进行数字证书验证
gpgkey=                                 #数字证书公钥的保存位置，自带光盘为/etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
enablegroups={1|0}：                    #是否基于组来批量管理程序包
failovermethod={roundrobin|priority}：  #有多个url时，选择的次序，roundrobin随机选择，默认轮询
cost=                                   #仓库优先级 ,默认为1000
```
> **yum配置文件可用变量**  
`$releasever`： 当前OS发行版本主版本号 如6或7  
`$arch`： 平台 如ia32e架构  
`$basearch`： 基础平台 如x86_64  
`$uuid`：为本机生成唯一码，存放/var/lib/yum/uuid   
`$YUM0-$YUM9`：可使用的环境变量    

*更多配置信息查看`man yum.conf`*

## 主要用法  
`yum [options] [command] [package ...]`

**[options]**
> -y：所有交互式自动回答为yes  
> -q：安静模式操作  
> --nogpgcheck： 禁止gpg检查  
> --disablerepo=repoidglob： 临时禁用此处指定的仓库（默认enabled=1）  
> --enablerepo=repoidglob： 临时启用  
> --noplugins： 禁用所有插件  

**[command]**

> `repolist` [all|enabled|disabled]：显示配置仓库信息   
> `list` [all|glob_exp1]：可安装或更新及已经安装的rpm包  
> `list` {available|installed|updates} [glob_exp]：加通配符查询相关匹配包  
>> list available 可用包  
>> list updates 可用升级包  
>> list installed 已安装包  

> `install` package1...：安装新软件包，也可指定版本号安装，默认安装最新版  
>> `reinstall` package1...：重新安装  

> `update` package1...： 升级程序包  
>> `check-update` package...：检测可用升级  

> `remove|erase` package1...：卸载  
> `info` package：查看信息  
> `provides|whatprovides` feature1...： 查看特性或文件由哪个程序提供  
> `clean` [packages|metadata|expire-cache|rpmdb|plugins|all]：清理本地缓存  
> `search` string1...： 搜索包名及简介  
> `makecache`： 构建缓存  
> `downgrade` package：降级  
> `deplist` package1...： 包的依赖信息  
> `history`：查看yum事务历史(安装，卸载，升级)  
>> history info  
>> history list  
>> history summary  
>> history stats  

> `group*`： 包组管理  
>> grouplist：查看系统中已经安装的和可用的软件包组
>> groupinfo：查询包组内软件包列表
>> groupinstall：安装指定软件组中的所有软件包
>> groupupdate：更新指定软件组中的所有软件包
>> groupremove：卸载指定软件组中的所有软件包

常用例子

```bash
yum repolist                           #列出当前yum仓信息
yum list php*                          #列出要php相关程序包
yum -y install httpd                   #安装apache
yum -y update httpd                    #升级
yum info tree                          #查看程序包tree信息
yum clean all                          #清理缓存
yum makecache                          #生成yum缓存
yum history                            #显示yum历史
yum grouplist                          #查看仓库中的组包信息
yum groupinstall 'Development Tools'   #安装组包中所有程序
yum groupremove 'Development Tools'    #移除组包中所有程序
```

## 本地光盘做yum源
1. `mount -r -t iso9660 /dev/sr0 /mnt/cdrom`：挂载光盘
2. `vim /etc/yum.repo.d/CentOS-Local.repo`：新建仓配置文件，并写入以下信息
3. `yum repolist`：查看仓信息

```
[centos7]  
name=yum local cdrom
baseurl=file:///mnt/cdrom
gpgcheck=0
enabled=1
```

## 简单搭建本地yum仓
1. 安装createrepo工具
2. 准备好rpm包放置/mnt/yum目录下
3. createrepo /mnt/yum：创建仓，会在包目录下产生一个repodata目录
4. 新建一个.repo配置文件（同上）
5. yum repolist：查看仓信息

## 其它
centos默认在线源为官方，有时访问会出现问题，可简单下载阿里或163提供的镜像repo文件，更换系统配置文件

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak                  #备份原repo
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repos     #下载并重命名，对应版本5，6，7
yum clean all                                                                               #清除缓存
yum makecache                                                                               #生成缓存
```

以上内容整理自马哥笔记
