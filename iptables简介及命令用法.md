---
title: iptables简介及命令用法
date: 2016-07-23 16:43:43
categories: Linux服务
tags: [iptables, 防火墙]
---

简要介绍Linux防火墙及iptables的常见用法

<!--more-->

## 简介
> 防火墙主要用于屏蔽来自于互联网，或来自于企业内部的用户的攻击操作及防范非授权的访问的。一般工作于网络或主机的边缘，对于进出本网络或主机的报文根据事先定义的检查规则做匹配检测，对于被规则匹配到的报文做出相应处理，时刻检查出入防火墙的所有数据包，决定拦截或是放行哪些数据包。

根据设备来划分通常可分为硬件防火墙和软件防火墙：

- **硬件防火墙**：专门的一台防火墙硬件设备，工作在整个网络入口处
- **软件防火墙**：工作在主机中TCP/IP协议站上面的软件（工作在内核中）

根据防火墙工作位置又可分为网络防火墙和主机防火墙：

- **网络防火墙：**工作与一个网络的边缘，能够实现对进出本网络的所有主机报文加以防护
- **主机防火墙：**主要是用来防范单台主机的进出报文

### iptables/netfilter
Linux上的防火墙套件为`iptables/netfilter`，iptables是用户空间上配置与修改过滤规则的命令，生成的规则直接送往linux内核空间netfilter中，netfilter是Linux核心中的一个通用架构，用于接收并生效规则，起到防火墙作用，二者关系如下：
![iptables](iptables.png)

在netfilter上定义了5个钩子函数(hook function)，分别作用于5个链上：

1. 路由前，目标地址转换       == > PREROUTING
2. 到达本机内部的报文必经之路 == > INPUT
3. 由本机转发的报文必经之路   == > FORWARD
4. 由本机发出的报文的必经之路 == > OUTPUT
5. 路由后，源地址转换         == > POSTROUTING

netfilter提供了一系列的表(tables),每个表由若干个链(chains)组成，而每条链可以由一条或若干条规则(rules)组成，关系如下：
![table-chain-rule](iptables1.png)

#### 4表
1. `raw`：用于配置数据包，raw 中的数据包不会被系统跟踪
4. `mangle`：用于对特定数据包的修改
3. `nat`：用于网络地址转换，如SNAT、DNAT、MASQUERADE、REDIRECT
2. `filter`：过滤，定义是否允许通过防火墙

#### 5链
1. `INPUT`链：当接收到防火墙本机地址的数据包（入站）时，应用此链中的规则
2. `OUTPUT`链：当防火墙本机向外发送数据包（出站）时，应用此链中的规则
3. `FORWARD`链：当接收到需要通过防火墙发送给其他地址的数据包（转发）时，应用此链中的规则
4. `PREROUTING`链：在对数据包作路由选择之前，应用此链中的规则
5. `POSTROUTING`链：在对数据包作路由选择之后，应用此链中的规则

#### 规则(处理机制)
1. `ACCEPT`：允许数据包通过
2. `DROP`：直接丢弃数据包，不给任何回应信息
3. `REJECT`：拒绝数据包通过，同时会给数据发送端一个响应的信息
4. `SNAT`：源地址转换，解决内网用户用同一个公网地址上网的问题，仅作用于nat表上POSTROUTING，INPUT上。在进入路由层面的route之前，重新改写源地址，目标地址不变，并在本机建立NAT表项，当数据返回时，根据NAT表将目的地址数据改写为数据发送出去时候的源地址，并发送给主机
5. `MASQUERADE`：是SNAT的一种特殊形式，适用于像adsl这种临时会变的ip上
6. `DNAT`:目标地址转换，让互联网上主机访问本地内网上的某服务器上的服务，仅作用于nat表上PREROUTING和OUTPUT。和SNAT相反，IP包经过route之后、出本地的网络栈之前，重新修改目标地址，源地址不变，在本机建立NAT表项，当数据返回时，根据NAT表将源地址修改为数据发送过来时的目标地址，并发给远程主机，可以隐藏后端服务器的真实地址
7. `REDIRECT`：是DNAT的一种特殊形式，将网络包转发到本地host上（不管IP头部指定的目标地址是啥），方便在本机做端口转发
8. `LOG`：仅记录日志信息，然后将数据包传递给下一条规则
9. `RETURN`：一般用于自定义链上，自定义链被内置链引用时，当没有规则被匹配时，返回内置链的下一条规则

#### 数据流向
- 与本机内部进程通信：
    **进入**：--> PREROUTING --> INPUT
    **出去**：--> OUTPUT --> POSTROUTING

- 由本机转发：
    **请求**：-->PREROUTING-->FORWARD-->POSTROUTING
    **响应**：-->PREROUTING-->FORWARD-->POSTROUTING

数据流向及相应的表链关系图如下：
![iptables/netfilter](iptables2.jpg)

## iptables命令用法
### 语法
`iptables [-t table] COMMAND chain [num] [-m match [match-options]] [-j target [target-options]]`

### 常用选项
> **链管理**
> `-F`：flush, 清空规则链，无法还原
> `-N`：new, 新建一条自定义链，被内建链上规则调用才能生效
> `-X`：delete, 删除引用计数为0的自定义空链
> `-P`：policy，设置默认策略，对filter表来讲，默认规则为ACCEPT或DROP
> `-E`：重命名引用计数为0的自定义链
> `-Z`：zero，计数器归零
> **规则管理**
> `-A`：Append，在尾后追加
> `-I`：Insert，在指定位插入规则，省略位置则为链首
> `-D`：Delete，删除指定规则
> `-R`：Replace，将指定规则替换为新规则
> **显示**
> `-L`：list，列出表中的链上的规则；
>> `-n`：numeric，以数值格式显示；
>> `-v`：verbose，显示详细格式信息，更详细`-vv`, `-vvv`
>> `-x`：exactly，计数器的精确结果；
>> `--line-numbers`：显示链中的规则编号

```bash
iptables -vnL               # 默认显示filter表规则，可指定表显示
iptables -F -t filter       # 清空filter表中规则，不指定则清空所有表中的规则
iptables -P INPUT DROP      # 设置INPUT链默认处理机制为DROP
iptables -D INPUT 2         # 删除INPUT链上第2条规则
iptables -N test_chain      # 添加自定义链test_chain，其只能被内置链接所引用
iptables -X test_chain      # 删除引用计数为0的自定义空链
iptables -I INPUT 2 xxxx    # 添加规则到INPUT上第2条
iptables -A OUTPUT xxxx     # 在OUTPUT链尾添加
```

### 匹配条件
#### 通用匹配
> `[!] -s ADDR[/mask]`：指定报文源IP地址匹配的范围，可以是IP和网络地址，`!`表示取反
> `[!] -d ADDR[/mask]`：指定报文目标IP地址匹配的范围
> `[!] -p PROTOCOL`：指定匹配报文的协议类型，一般有三种tcp, udp和icmp
> `[!] -i INTERFACE`：数据报文流入的接口；只适用于PREROUTING, INPUT, FORWARD
> `[!] -o INTERFACE`：数据报文流出的接口；只适用于OUTPUT, FORWARD, POSTROUITING

#### 扩展匹配
##### -p tcp
> `--sport PORT[:PORT]`: tcp 指定源端口，可以是连续多个端口
> `--dport PORT[:PORT]`: tcp 指定目标端口
> `--tcp-flags`: tcp标示位，包括SYN，ACK，FIN，RST，URG，PSH，分2段，表示在左段的标示位中，只有右段中的为1

```bash
# 不指定表默认为filter过滤

# 允许172.16网段的主机可通过22号端口到达本机172.16.60.1
iptables -A INPUT -s 172.16.0.0/16 -d 172.16.60.1 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -s 172.16.60.1 -d 172.16.0.0/16 -p tcp --dport 22 -j ACCEPT

# 拒绝tcp标示位不正常数据
iptables -A INPUT  -p tcp --tcp-flags all all -j DROP       # tcp标示位全为1
iptables -A OUTPUT  -p tcp --tcp-flages all none -j DROP    # tcp标示位全为0
```

##### -p udp
> `--sport PORT[:PORT]` ：udp 源端口，可连续
> `--dport PORT[:PORT]`：udp 目标端口

```bash
# 放行本机dns，本机172.16.60.1
iptables -A INPUT -s 172.16.0.0/16 -d 172.16.60.1 -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -s 172.16.60.1 -d 172.16.0.0/16 -p udp --sport 53 -j AACEPT

# 本机访问其它主机DNS
iptables -A OUTPUT -s 172.16.60.1 -p udp --sport 53 -j ACCEPT  
iptables -A INPUT -d 172.16.60.1 -p udp --dport 53 -j ACCEPT
```

##### -p icmp
> `--icmp-type 8`：echo-request,请求
> `--icmp-type 0`：echo-reply,响应

```bash
# 放行ping其它主机
iptables -A INPUT -d 172.16.60.1 -p icmp --icmp-type 0 -j ACCEPT
iptables -A OUTPUT -s 172.16.60.1 -p icmp --icmp-type 8 -j ACCEPT
```

##### -m multiport
多端口匹配，可用于匹配非连续(,)或连续端口(:)，需指明相应tcp或udp协议
> `[!] --sports PORT[,PORT...]`
> `[!] --dports PORT[,PORT...]`
> `[!] --ports PORT[,PORT...]`

```bash
# 开放本机21，22，23，80端口
iptables -I INPUT -d 172.16.60.1 -p tcp -m multiport --dports 21:23,80 -j ACCEPT
iptables -I OUTPUT -s 172.16.60.1 -p tcp -m multiport --sports 21:23,80 -j ACCEPT
```

##### -m iprange
匹配指定范围内的地址，匹配一段连续的地址而非整个网络时有用
> `[!] --src-range FROM_IP[-TO_IP]`
> `[!] --dst-range FROM_IP[-TO_IP]`

```bash
iptables -A INPUT -d 172.16.60.1 -p tcp --dport 23 -m iprange --src-range 172.16.60.1-172.16.60.100 -j ACCEPT
iptables -A OUTPUT -s 172.16.60.1 -p tcp --sport 23 -m iprange --dst-range 172.16.60.1-172.16.60.100 -j ACCEPT
```

##### -m string
字符串匹配，能够检测报文应用层中的字符串
> `[!] --string PATTERN`: 指定字符串
> `[!] --hex-string "HEX_STRING"`: HEX_STRING为编码成16进制格式的字串
>> `--algo {kmp|bm}`: 指定算法
    iptables -I OUTPUT -m string --algo kmp --string "sex" -j DROP

##### -m time
基于时间做访问控制
> `--datestart YYYY[-MM][-DD[Thh[:mm[:ss]]]]`：开始日期
> `--datestop YYYY[-MM][-DD[Thh[:mm[:ss]]]]`：结束日期
> `--timestart hh:mm[:ss]`：开始时间
> `--timestop hh:mm[:ss]`：
> `[!]--monthdays day[,day...]`：指定日期
> `[!]--weekdays day[,day]|1-7`：指定星期，可用Mon,Tue...或 Mo,Tu...或 1-7

```bash
iptables -A INPUT -d 172.16.60.1 -p tcp --dport 90 -m time --timestart 08:20 --timestop 18:40 --weekdays Mon,Tue,Thu,Fri -j ACCEPT
iptables -A OUTPUT -s 172.16.60.1 -p tcp --dport 901 -j ACCEPT
```

##### -m connlimit
连接数限制，对每IP所能够发起并发连接数做限制
> `--connlimit-upto N`：小于等于N
> `--connlimit-above N`：大于等于

```bash
iptables -A INPUT -d 172.16.60.1 -p tcp --dport 80 -m connlimit --connlimit-above 5 -j DROP
```

##### -m limit
传输速率限制
> `[!] --limit n[/second|/minute|/hour|/day]`: 限制每秒,分,小时,天n个
> `[!] --limit-burst N`: 峰值限制
    
```bash
iptables -A INPUT -d 172.16.60.1 -p icmp --icmp-type 8 -m limit --limit 20/second --limit-burst 5 -j ACCEPT
```

##### -m state
状态匹配，状态包括`NEW`(新建连接), `ESTABLISHED`(已建立的连接), `RELATED`(有关联关系), `INVALID`(无法识别),`UNTRACKED`(未追踪)
> `[!] --state STATE`

```bash
iptables -I INPUT -d 172.16.60.1 -m multiport --dports 22,80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -I OUTPUT -s 172.16.60.1 -m state --state ESTABLISHED -j ACCEPT
```

<font color=red>注：</font>开启状态追踪机制必须加载`nf_conntrack`模块，特定服务还需加载相应模块，如ftp需加载`nf_conntrack_ftp`

**状态追踪编写法则：**  

1. 所有进入或出去状态为`ESTABLISHED`都应该放行
2. 严格检查进入状态为`NEW`的连接
3. 所有状态有`INVALIED`都应该拒绝

**相关文件**

1. /proc/sys/net/nf_conntrack_max：连接追踪功能所能容纳的连接的最大数目，建议调大
2. /proc/net/nf_conntrack：记录当前追踪的所有连接
3. /proc/sys/net/netfilter/*：不同协议或连接类型追踪时的属性

### 自定义链
iptables中的自定义链只能被内置引用使用，可用于自定义一系列针对性的规则方便被不同的内置链所引用，同时建议在最后添加一条RETURN规则，已便未匹配到时返回主链继续匹配。删除自定义链需清空引用，同时清空链中的规则

```bash
# 自定义一条放行ping请求的链
iptables -N icmp_test
iptables -A icmp_test -p icmp -j ACCEPT
iptables -A icmp_test -j RETURN

# 引用自定义链
iptables -A INPUT -j icmp_test
iptables -A OUTPUT -j icmp_test
```

### NAT
网络地址转换，包括SNAT，DNAT，MASQUERADE
`SNAT`：源地址转换，用于内网主机访问互联网，作用于nat表上POSTROUTING，INPUT
`DNAT`：目标地址转换，让互联网上主机访问本地内网上的某服务器上的服务，作用于nat表上PREROUTING和OUTPUT

> `-j SNAT --to-source SIP`：源地址转换
> `-j DNAT --to-destination DIP[:PORT]`：目标地址转换，支持端口映射
> `-j MASQUERADE`：动态获取，仅用于nat表的POSTROUTING

```bash
# 本地内网172.16网段主机访问互联网时，将源地址转换在本公司的公网8.9.9.9上
iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -j SNAT --to-source 8.9.9.9
iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -j MASQUERADE
                    
# 外部访问公司对外服务器8.8.8.9的22022端口时，将请求转发给本地服务器192.168.0.1的22端口上
iptables -t nat -A PREROUTING -d 8.8.8.9 -p -tcp --dport 22022 -j DNAT --to-destination 192.168.0.1:22
```

### 保存和重载规则
使用iptables命令生成的规则在重启后将失效，所以可将规则保存至文件，重启时从文件中读取

```bash
iptables-save > /PATH/TO/SOME_RULE_FILE       # 将编写的规则保存到指定文件中
iptables-restore < /PATH/FROM/SOME_RULE_FILE  # 从指定文件中恢复规则

# centos6上也可使用如下命令
service iptables save       # 自动保存规则至/etc/sysconfig/iptables文件中
server iptables restore     # 从/etc/sysconfig/iptables文件中重载规则
```

### 规则优化
1. 可安全放行所有入站及出站，且状态为ESTABLISHED的连接
1. 服务于同一类功能的规则，匹配条件严格的放前面，宽松放后面
1. 服务于不同类功能的规则，匹配报文可能性较大扩前面，较小放后面
1. 设置默认策略：   
    (a) 最后一条规则设定  
    (b) 默认策略设定  













