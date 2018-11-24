---
title: Linux程序包管理之编译安装
date: 2016-06-03 10:54:49
categories: Linux基础
tags: [源码,编译]
---

程序包安装最后一课
<!--more-->

## 简介
相比rpm包安装，源码安装的好处如下： 

1. 可以自行调整编译参数，最大化地定制安装结果   
2. 可以选择最新的软件包，而Linux发行版自带的软件包一般都是最稳定的老版本  
3. 相对而言，源码安装的性能是最优异的  
4. 卸载也比较方便和简单，直接删除安装的指定的目录即可，同时它比较安全 

## 安装
Linux上真正认识的可执行文件也是二进制文件，源码包的安装其实就是通过自己的配置生成二进制可执行文件的过程，一般的过程为：**源程序-->预处理-->编译-->汇编-->链接-->二进制格式**。而在linux的大部分程序是由C语言来编写的，gcc就是用来编译C的编译器，所以在安装前需要先安装gcc。以下就是C代码编译安装三步骤（先下载源码并解压cd进去）  

**建议：安装前查看INSTALL，README**

```bash
./configure     # 生成makefile文件
make            # 将源代码文件变为二进制的可执行程序
make install    # 将编译好的程序文件复制到系统中
```
- `./configure`：这步在源码安装过程中是比较重要的一步，通过选项传递参数，指定启用特性、安装路径等，执行时会参考用户的指定以及Makefile.in文件生成makefile，同时检测安装环境（开发过程中，由autoconf来生成configure脚本，automake来生成Makefile.in），一般有以下选项
> `--help`：查看帮助信息   
> `--prefix=/usr/local/xxx`：指明默认安装位置，建议安装在/usr/local下
> `--sysconfdir=/PATH/TO/CONFFILE_PATH`：配置文件安装位置  
> `--with-PACKAGE[=ARG]`：依赖包选项
> `--without-PACKAGE`：  
> `--disable-FEATURE`：开启或关闭某特性
> `--enable-FEATURE[=ARG]`  

## 安装后配置
1. 将程序二进制文件路径加入**_PATH环境变量_**  
> 新建/etc/profile.d/NAME.sh，添加`export PATH=$PATH:/usr/local/xxx/bin`
2. 导出**_库文件_**：默认系统搜索库文件的路径/lib[64], /usr/lib[64]，要增添额外搜寻路径
> 新建/etc/ld.so.conf.d/xxx.conf，写入`/usr/local/xxx/lib`
> 使用`ldconfig [-v]`命令，通知系统重新搜寻库文件(-v: 显示过程)
3. 导出**_头文件_**：默认路径为/usr/include，增添头文件搜寻路径
> 方式1：ln -s /usr/local/xxx/include/* /usr/include/
> 方式2：ln -s /usr/local/xxx/include  /usr/include/xxx
4. 导出**_man文件_**路径：使man命令可以搜索到程序的帮助信息  
> 在/etc/man.config中添加一条`MANPATH /usr/local/xx/man`（centos7: /etc/man_db.conf）


以上就是简要的源码安装过程，更多内容以后再添加