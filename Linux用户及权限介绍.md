---
title: Linux用户及权限介绍
date: 2016-06-12 11:56:44
categories: Linux基础
tags: ['用户','权限']
---

浅谈linux上用户、组及权限相关内容
<!--more-->

> Linux最优秀的地方之一就在于他的多用户多任务环境，在系统可建立多个用户，而多个用户可以在同时登录同执行各自不同的任务而互不影响，而为了让各个使用者具有较保密的文件数据，因此文件的权限管理就变的很重要了。也正是通过对于权限的划分与管理，实现了多用户多任务的运行机制

### 用户分类
1. **超级用户**：默认root用户，拥有对系统管理最高权限，权限设置对其不起作用（UID:0，GID:0）
2. **普通用户**：具有登录系统的权限，受相应权限约束（UID/GID:500-60000 [centos7:1000-60000]）
3. **系统用户**：不能登录系统，用于系统管理来满足相应的系统进程对文件属主的要求（UID/GID:1-499 [7:1-999]）

### 用户组
用户组是具有相同特征用户的逻辑集合，有时我们需要让多个用户具有相同的权限，比如查看、修改某一个文件的权限，就可以将用户加入相应组中，同时设置该组拥有某些权限，这样该组下的用户就同时具有相关权限，以简化用户管理
。根据功能组可有3种分类方式：

1. 普通用户组和管理员组
2. 基本组和附加组
3. 私有组和公共组

*新建用户时默认建立一个同名的组作为其私有组*

### 文件权限
Linux一般将文件可存取的身份分为三个类别：owner(所有者)、group(所属组)和others(其它)，且三种身份各有read(读)、write(写)和execute(执行) 等权限。当一个用户发起一个进程时，此进程将继承用户的属主、属组的权限，再以进程继承的权限来控制文件的访问权限  
通过`ls -l`命令可以用来查看文件权限  
![文件权限](file_mode.png)

**文件类型**：
> `-`：一般文件  
> `d`：directory 目录  
> `c`：character 字符设备  
> `b`：block 块设备  
> `l`：link 链接文件  
> `p`：pipe 管道  
> `s`：socket 套节字  

**用户权限**：
> 前3位：属主(u)权限  
> 中3位：属组(g)权限  
> 后3位：其它(o)权限  
 
- r：读，内核用4表示
- w：写，2
- x：执行，1

**目录和文件的权限区别**

|         |    读(read)    |   写(write)   | 执行(execute) |
| :-----: | :------------: | :-----------: | :-----------: |
| 对文件  | cat,more,head..|   vi,echo..   |   ./**执行    |
| 对目录  |      ls        |touch,cp,mv,rm |      cd       |

---

### 相关文件及配置
**1. /etc/passwd**：记录用户的详细信息

```bash
# 用户名:密码:UID:GID:注释信息:家目录:默认Shell
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
```
每个用冒号隔开的字段含义：

1. 用户名：用户的名称
2. 密码：X表示占位符，密码信息保存在/etc/shadow
3. UID：用户识别码
4. GID：用户所属组的GID(基本组)
5. 注释信息：备注用户的基本信息，如用户的住址、电话、姓名等
6. 家目录：系统登录用户后的工作目录
7. 默认shell：定义用户登录系统所使用的shell,指定的shell需要在/etc/shells中出现

**2. /etc/shadow**：用户的密码信息，只有root才有读写权限

```bash
# 用户名:密码:上一次修改密码的时间:密码最短使用期限:最长使用期限:警告时间:帐号禁用期限:帐户过期时间:保留字段
liu:$6$IEfGuajTtREgj2sW$iBPvAGGTKgkmaiUtOlqDjElzcPYGx1wvhbKgj:16930:0:99999:7:::
```

1. 用户名: 对应/etc/passwd文件
2. 密码: 加密的密码
3. 上一次修改密码的时间: 指用户上次修改密码的时间(时间戳)
4. 密码最短使用期限: 指用户修改密码后，需要到多少天后方可更改密码，0表示禁用
5. 密码最长使用期限: 指用户的密码到多少天后需要修改密码
6. 警告时间: 指用户密码到期前多少天提示用户修改密码，0和空字段表示禁用此功能
7. 帐号禁用期限：过期后不可用时间
8. 帐户过期时间: 批帐户在密码过期后多少天还未修改密码，将被停用
9. 保留字段：保留将来使用

**3. /etc/group**：用户组信息

```bash
# 组名:密码:GID:用户列表
root:x:0:
bin:x:1:bin,daemon
daemon:x:2:bin,daemon
```

1. 组名：组的名称，默认同名用户名
2. 密码：组的密码占位符
3. GID：组ID
4. 用户列表：隶属此组的用户（附加组）

**4. /etc/gshadow**：用户组密码信息

```bash
# 组名:密码:组管理者:用户列表
root:$6$PLRAiZsvLuGJqvFG3D8fbjYHDho2RQUe93glO::
```

1. 组名：组的名称，同步/etc/group文件中  
2. 密码：第一个$后面表示加密算法，第二个$后面表示加密的密码  
3. 组管理者：可以对此组成员有操作权限  
4. 用户列表： 用户的列表

**5. /etc/login.defs**:定义创建用户时的默认设置，如UID和GID的范围，用户的过期时间、是否需要创建用户主目录等

```bash
MAIL_DIR        /var/spool/mail     # 创建用户的同时在/var/spool/mail中创建同名邮件文件
PASS_MAX_DAYS   99999               # 指定密码保持有效的最长天数
PASS_MIN_DAYS   0                   # 表示自从上次密码修改以来多少天后用户才被允许修改口令
PASS_MIN_LEN    5                   # 指定密码的最小长度
PASS_WARN_AGE   7                   # 表示在口令到期前多少天系统开始通知用户口令即将到期
UID_MIN         500                 # 创建用户UID从500开始
UID_MAX         60000               # 最大UID为60000
GID_MIN         500                 # 最小GID为500
GID_MAX         60000               # 最大GID为60000
CREATE_HOME     yes                 # 是否创建用户主目录，yes为创建，no为不创建
```

**6. /etc/default/useradd**：添加用户默认选项

```bash
GROUP=100               # 
HOME=/home              # 新建用户的家目录存放路径
INACTIVE=-1             # 是否启用帐号过期，-1不启用
EXPIRE=                 # 帐号过期日期，默认不启用
SHELL=/bin/bash         # 默认使用的shell
SKEL=/etc/skel          # 新建用户家目录下的文件从此目录下复制而来的
CREATE_MAIL_SPOOL=yes   # 是否建立邮件文件
```

### 相关命令
`useradd`、`chown`、`chmod`、`usermod`、`userdel`、`groupadd`、`groupmod`、`groupdel`、`passwd`、`gpasswd`、`su`、`id`、`newgrp`、`chage`、`chfn`、`chsh`等，见[常用命令--用户管理](http://liu2lin600.github.io/2016/05/21/%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4--%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86/)

*最后：作为上帝般的root来说，任何权限都对其无效，谨慎使用*

