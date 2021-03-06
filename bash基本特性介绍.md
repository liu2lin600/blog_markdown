---
title: bash基本特性介绍
date: 2016-06-10 12:53:29
categories: Linux基础
tags: bash
---

bash特性介绍

<!--more-->

## 简介
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;管理整个计算机硬件的其实是操作系统的核心（kernel），这个核心是需要被保护的！ 一般使用者就只能通过内核提供的接口shell(壳) 来跟核心沟通，以让核心达到我们所想要达到的工作。在Linux各种发行版上shell包括sh、csh、bash、ksh、zsh等，可以通过查看`/etc/shells`来查看，其中bash是默认shell，bash (Bourne-again Shell) 是一个来自 GNU Project的命令行解释器/编程语言，它的名字是向它的前身————很早以前的Bourne shell致敬。既然是默认shell，那么掌握其特性对于Linux系统的学习就很有必要了

## Bash特性

### 1. 补全
通过`Tab`键补全命令或路径，唯一时直接补全，否则敲2下列出所有符合的

1. 命令：外部命令通过读取PATH环境变量
2. 路径：根据给定的起始路径

### 2. 命令历史
`history`命令可以查看命令历史，保存在`~/.bash_history`文件中，打开shell时内核读取此文件到内存中，在退出shell前命令缓存在内存中，退出时将历史再写入该文件
> history选项：
> 数字：显示最近几条  
> `-c`：清空内存中的历史  
> `-d #`：删除指定编号的历史  
> `-w`：从内存中写入到`~/.bash_history`文件中  
> `-r`：从历史文件中读取到内存 

相关环境变量

1. HISTSIZE：历史文件保存最大条数  
2. HISTFILE：历史文件路径  
3. HISTFILESIZE：历史条数  
4. HISTTIMEFORMAT：显示时间，如 HISTTIMEFORMAT="%F %T"
5. HISTIGNORE：忽略指定字符命令历史，如 HISTIGNORE="str1:str2:… "  
6. HISTCONTROL：记录方式控制，值有如下：
> ignorespace：忽略空格开头，此时敲空格再输入命令将不会被记录  
> ignoredups：连续且重复的不记录   
> ignoreboth：包括以上两者  
> *相关配置文件为`/etc/profile`和`~/.bash_profile`*

常用技巧

- `!STR`：调用最近以STR开头的命令  
- `!5`：调用第5个历史，为负数表示倒数    
- `!5:2`：调用第5条历史的第2个参数  
- `!-2:2`：调用倒数第二个命令第2个参数  
- `ESC`+`.`：调用最后一条命令，注意按`ESC`后松开再按`.`  
- `!$`：最后一条命令的最后一个参数 (!5:$ 第5条命令最后参数，下同)  
- `!^`：最后一条命令的第一个参数
- `!:2`：上一命令第二个参数    
- `!*`：上一命令全部参数  
- `!!`：调用上一条命令  

### 3. 快捷键
`ctrl+a`：光标移到行首  
`ctrl+e`：光标移到行尾  
`ctrl+u`：删除光标到行首  
`ctrl+k`：删除光标到行尾  
`ctrl+w`：向前删除一个词  
`ctrl+l`：清屏  
`ctrl+c`：取消命令的执行  
`ctrl+r`：搜索命令历史

### 4. 命令行展开
`~`：自动展开为用户家目录，`~USER`表示USER用户家目录  
`{}`：展开内字段，用逗号隔开  

```bash
mkdir -p /tmp/x{y1/{a,b},y2}      # 生成/tmp/xy1/a, /tmp/xy1/b, /tmp/xy2
mkdir -p /tmp/m/{a,b}_{x,y}       # /tmp/m/a_x, /tmp/m/a_y, /tmp/m/b_x, /tmp/m/b_y
cp /tmp/test{,.bak}               # 在当前目录下备份文件
```

### 5. 命令执行结果状态  
命令执行结果状态返回值：成功为0，失败为1-255  
查看：`echo $?`

### 6. 文件名通配(glob)
通配符有如下：
> `*`: 代表任意长度的任意字符  
> `?`： 代表任意单个字符  
> `[]`：代表匹配指定范围内的任意单个字符  
>> `[a-z]`：小写字母，同[[:lower:]]  
>> `[A-Z]`：大写字母，同[[:upper:]]  
>> `[0-9]`：数字，同[[:digit:]]  

> `[^]`: 匹配指定范围之外的任意单个字符   
> `[[:space:]]`：空白字符  
> `[[:punct:]]`：标点符号  
> `[[:lower:]]`：小写字母，如同[a-z]  
> `[[:upper:]]`: 大写字母，如同[A-Z]  
> `[[:alpha:]]`: 大小写字母，如同[a-zA-Z]  
> `[[:digit:]]`: 数字，如同[0-9]  
> `[[:alnum:]]`: 数字和大小写字母，如同[0-9a-zA-Z]  

*可通过`man 7 glob`查看更多帮助*

### 7. IO重定向
- 输出重定向：将原本显示到屏幕的内容重定向到文件或其它命令中  
- 输入重定向：将文件的内容作为标准输入
- 报错重定向：将命令执行的错误信息重定向到文件或其它地方

`>`：覆盖输出重定向，`set -C`：关闭覆盖功能，`set +C`：开启  
`>>`：追加输出重定向  
`>1`：强行覆盖输出  
`2>`,`2>>`：错误输出重定向    
`&>`, `&>>`：正常或错误合并输出重定向  
`<`：输入重定向  
`<<`：从标准输入中读入，知道遇到delimiter分界符  

> 说明：shell在产生一个新进程后，新进程的前三个文件描述符都默认指向三个相关文件。这三个文件描述符对应的数组下标分别为0，1，2。0对应的文件叫做标准输入（stdin），1对应的文件叫做标准输出（stdout），2对应的文件叫做标准报错(stderr)。但是实际上，默认跟人交互的输入是键盘、鼠标，输出是显示器屏幕，这些硬件设备对于程序来说都是不认识的，所以操作系统借用了原来“终端”的概念，将键盘鼠标显示器都表现成一个终端文件。于是stdin、stdout和stderr就最终都指向了这所谓的终端文件上。于是，从键盘输入的内容，进程可以从标准输入的0号文件描述符读取，正常的输出内容从1号描述符写出，报错信息被定义为从2号描述符写出。这就是标准输入、标准输出和标准报错对应的描述符编号是0、1、2的原因。这也是为什么对报错进行重定向要使用2>的原因  

```
COMMAND > filename          # 标准输出重定向到一个新文件中
COMMAND 2> filename         # 把标准错误重定向到一个文件中
COMMAND 1> file 2>&1        # 把标准输出和错误一起重定向到一个文件中，同&>
COMMAND < file1 > file2     # 命令以file1作为标准输入，以file2文件作为标准输出
COMMAND < filename          # 以filename文件作为标准输入

COMMAND <<EOF               # 从标准输入中读入，知道遇到delimiter分界符为止，一般用EOF    
xxxxxxxx
EOF

```

### 8. 管道
linux的另一重要哲学思想：**由功能单一的程序组合完成复杂操作**，这其中的重要实现方式就是`|`管道  
即将上一个命令执行的结果传递到下一个，可以多次传递以达到特别强大的功能

```bash
tail -9 /etc/passwd | head -1 | cut -d: -f1,7 | tee /tmp/users
netstat -ant | awk '{print $NF}' | grep -v '[a-z]' | sort | uniq -c
```

### 9. 命令hash
缓存外部命令到内存hash表中，可以加快下次执行速度，通过`hash`来查看

```bash
hash                # 查看hash表中缓存的命令
hash -d COMMAND     # 删除指定命令缓存 
hash -r             # 清空hash缓存表
```

### 10. 多命令执行
多个命令间使用特殊符号连接，形成一定的逻辑关系，包括如下：
`;`：命令间无逻辑关系，从左到右执行  
`||`：或，前一个命令执行错误才执行下一个
`&&`：与，前一个命令正确执行才执行下一个
`!`：非，取反

```bash
COMMAND1 ; COMMAND2 ; COMMAND3      # 3个命令都执行
COMMAND1 && COMMAND2 || COMMAND3    # 1正确则再执行2，否则执行3
! COMMAND1 || COMMAND2              # 1正确时执行2
```

### 11. 命令别名
合理定义命令别名会大大提高工作效率  
`alias`、`unalias`：临时定义删除，写入配置文件`~/.bashrc`或`/etc/bashrc`永久有效

```bash
alias                           # 查看用户别名
alias l='ls -Alh --color=auto'  # 定义别名
unalias l                       # 删除别名
```
*如果别名同原命令同名，如果要执行原命令，可使用如下*

- \COMMAND
- 'COMMAND'
- /PATH/COMMAND：外部命令路径

### 12. 命令引用 
使用反引号**\`COMMAND\`**或**$(COMMAND)**来取一个命令的执行结果。（单引号：不解析变量，原样输出，双引号：解析变量）

```bash
file `which cat`
touch $(date +%F).log
```

### 12. 配置文件 
#### 1、登录类型

- 交互式登录shell进程
  > 1、通过某终端输入账号密码后登录shell
  > 2、使用su命令：`su - USER`, 或 `su -l USER`

- 非交互式登录shell进程
  > 1、su USER 执行的登录切换
  > 2、图形界面下打开的终端
  > 3、运行脚本

#### 2、配置文件类型

- profile类：为交互式登录的shell进程提供配置

  > 功用：用于定义环境变量，运行命令或脚本
  > 全局：`/etc/profile`、`/etc/profile.d/*.sh`
  > 个人：`~/.bash_profile`

- bashrc类：为非交互式登录的shell进程提供配置

  > 功用：定义本地变量；定义命令别名
  > 全局：`/etc/bashrc`
  > 个人：`~/.bashrc`

#### 3、bash路径搜索

- 交互式登录shell进程：
  > `/etc/profile` -> `/etc/profile.d/*` -> `~/.bash_profile` -> `~/.bashrc` -> `/etc/bashrc`

- 非交互式登录shell进程：
  > `~/.bashrc` -> `/etc/bashrc` -> `/etc/profile.d/*`

#### 4、定义生效方法

- 命令行上定义的变量和别名，其作用域为当前shell进程的生命周期
- 配置文件定义的特性，只对之后新启动的shell进程有效
- 通过命令行重复定义；或让shell进程重读配置文件，使用 `.` 或 `source` 来生效配置


### 13. 其它

- 执行上条命令，取代指定字符

    ```bash
    scp xxx data02:/ect/
    ^data02^data03 			# 等效于scp xxx data03:/ect/
    ```

- 排除

    ```bash
    rm !(*.py) 		# 删除除py结尾的其它文件
    ```


