---
title: Linux特殊权限及ACL简介
date: 2016-06-25 16:30:32
categories: Linux基础
tags: ['特殊权限','acl']
---

对于linux中文件或目录的权限，除了普通的读(r)、写(w)、执行(x)权限外，还存在些特殊权限

<!-- more -->

> 任何一个可执行程序文件能不能启动为进程，取决发起者对程序文件是否拥有执行权限，一旦启动为进程之后，其进程的属主为发起者，进程的属组为发起者所属的组，进程访问文件时的权限，取决于进程的发起者，不过特殊权限的存在让这一情况发生了点改变，这些权限主要有SUID,SGID,STICKY，chattr，ACL，SELinux设置的权限

先看张图，图中的s,t就是SUID,SGID,STICKY3种特殊权限的表示
!['特殊权限'](suid.png)
下面对这3种特殊权限分别作下介绍

## SUID

- **功能：**作用于可执行文件，当其运行为进程时，该程序属主不再是发起人，而是程序文件自身属主  
- **展示：**展示于文件属主执行权限位，如属主有执行权限则显示为`s`，否则为`S`  
- **修改：**`chmod u{+|-}s file1`，`+`为添加，`-`为删除  
- **实验：**管理员复制`cat`命令到其它目录并添加SUID，切换普通用能否查看/etc/shadow文件？  
![suid实验](suid1.png)

*SUID非常有风险，慎用。不过`passwd`命令正是使用了suid权限才使得普通用户可以使用该命令来修改自己的密码并写入/etc/shadow文件中，同时普通用户使用`passwd`时，不可带任何参数，确保了只能修改自己的密码*

## SGID
- **功能：**对目录赋权，则在目录下创建的文件和目录都继承该目录属组权限，而非创建者本身属主; 对可执行文件赋权，同SUID，执行时将拥有文件属组的权限  
- **展示：**展示于文件属组执行权限位，如属组有执行权限则显示为`s`，否则为`S`  
- **修改：**`chmod g{+|-}s file1`


## STICKY
- **功能：**对目录赋权，在该目录创建的文件或目录只有属主才有权限删除
- **展示：**展示于目录其它用户执行权限位，如原来有执行权限则显示为`t`，否则为`T`
- **修改：**`chmod {+|-}t DIR`
![sticky实验](sticky1.png)

*系统默认情况下/tmp和/usr/tmp文件夹带有sticky权限*

### 权限数字表示
- SUID = 4 (100)
- SGID = 2 (010)
- Sticky = 1 (001)

例如：文件权限位为`rwsr-Sr-T`，用数字表示为`7744`

## chattr设置权限
使用`chattr`设置的特殊权限不能使用`ls -h`看，使用`lsattr`查看，设置有两种形式：

- `i`：作用于的文件不可删除，修改，改名及设置链接，对root用户同样起作用  
- `a`：可向文件中追加内容

```bash
chattr +|-i file1   # +为添加，-为删除权限
chattr +|-a file1   
lsattr file1        # 查看
```

## ACL
access control lists：访问控制列表，被ACL控制权限的文件会在权限字符串后面加上`+`，如`rwxrwxrwx+`，其优先级高于普通设置的权限。当文件权限符后出现`+`时，其属组的权限位表示受*ALC_MASK*控制，使用`ls -l`查看的属组所显示的权限会不正确，相关命令有`getfacl`,`setfacl`

*命令用法见*：[setfacl命令用法](http://liu2lin600.github.io/2016/05/21/%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4--%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86/)

暂时写到这......
