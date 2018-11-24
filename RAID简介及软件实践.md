---
title: RAID简介及软件实践
date: 2016-05-30 10:41:27
categories: Linux基础
tags: 
---

独立磁盘冗余阵列简要介绍及软件方式实现实例
<!--more-->

## RAID简介
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RAID全称独立磁盘冗余阵列（Redundant Array of Independent Disks），旧称廉价磁盘冗余阵列（Redundant Array of Inexpensive Disks），简称磁盘阵列。其基本思想就是把多个相对便宜的硬盘组合起来，成为一个硬盘阵列组，使性能达到甚至超过一个价格昂贵、容量巨大的硬盘。根据选择的版本不同，RAID比单颗硬盘有以下一个或多个方面的好处：增强数据集成度，增强容错功能，增加处理量或容量。另外，磁盘阵列对于电脑来说，看起来就像一个单独的硬盘或逻辑存储单元。分为RAID-0，RAID-1，RAID-1E，RAID-5，RAID-6，RAID-7，RAID-10，RAID-50，RAID-60等    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;简单来说，RAID把多个硬盘组合成为一个逻辑扇区，因此，操作系统只会把它当作一个硬盘。RAID常被用在服务器电脑上，并且常使用完全相同的硬盘作为组合。由于硬盘价格的不断下降与RAID功能更加有效地与主板集成，它也成为玩家的一个选择，特别是需要大容量存储空间的工作，如：视频与音频制作。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;最初的RAID分成了不同的档次，每种档次都有其理论上的优缺点，不同的档次在两个目标间获取平衡，分别是增加数据可靠性以及增加存储器（群）读写性能   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前RAID的实现方式有硬件RAID和软件RAID，硬件RAID是通过RAID卡连接多个硬盘，或者主板中集成了RAID控制器来实现RAID的相关功能。软件RAID则通过软件层面模拟实现RAID的相关功能

下面来介绍常用RAID不同的级别的区别：

## RAID 0
> 条带化（Stripe）存储，它将两个以上的磁盘并联起来，成为一个大容量的磁盘。在存放数据时，分段后分散存储在这些磁盘中，因为读写时都可以并行处理，所以在所有的级别中，RAID 0的速度是最快的。但是RAID 0既没有冗余功能，也不具备容错能力，如果一个磁盘（物理）损坏，所有数据都会丢失，危险程度与JBOD相当,因此RAID 0常用于图象，视频等领域，不能应用于数据安全性要求高的场合

![raid0](raid0.png) 


## RAID 1
> 镜像（Mirror）存储，它是通过磁盘数据镜像实现数据冗余，在成对的独立磁盘上产生互为备份的数据。当原始数据繁忙时，可直接从镜像拷贝中读取数据，因此RAID 1可以提高读取性能。RAID 1是磁盘阵列中单位成本最高的，但提供了很高的数据安全性和可用性。当一个磁盘失效时，系统可以自动切换到镜像磁盘上读写，而不需要重组失效的数据。因此RAID 1常用于对容错要求极严的应用场合，如财政、金融等领域

![raid1](raid1.png)

## RAID 4
> 奇偶校验（XOR）条带存储，共享校验盘，数据条带存储单位为块。由3块或3块以上设备组成，并行处理提高磁盘性能，一个硬盘存储冗余校验码，通过异或运算还原数据。三个或三个以上的硬盘组成阵列可以实现当某一个硬盘损坏可以使其数据通过异或运算还原，通过读写并行处理提高性能，但是因为冗余校验码都是存放在单一硬盘上，所以此硬盘性能可能会很差，并且易损坏，在生产中不常用

![raid4](raid4.png)


## RAID 5
> 相比于RAID-4而言，冗余校验码分别存放在每个硬盘中。RAID-5就是RAID-4的升级版，弥补了RAID-4的缺陷，将RAID-4中被诟病的缺点：“冗余校验码存放在一个硬盘上”得以解决，RAID-5采用将冗余校验码分别存放在每一个磁盘上来达到负载均衡的效果，而且极大的提高了整体性能，但是只能提供一块硬盘的冗余，当有两块盘坏掉的时候，整个RAID的数据失效

![raid5](raid5.png)

## RAID 6
> RAID 6在RAID 5的基础上，把校验存储增加到了2台，容错能力比RAID 5有所提升，更大的“写损失”，“写性能”非常差，生产中不常用

![raid6](raid6.png)


## RAID 10/01
> RAID-10/01是两种混合型RAID，RAID 10是先镜射再分区数据，再将所有硬盘分为两组，视为是RAID 0的最低组合，然后将这两组各自视为RAID 1运作。RAID 01与RAID 10相反，当RAID 10有一个硬盘受损，其余硬盘会继续运作。RAID 01只要有一个硬盘受损，同组RAID 0的所有硬盘都会停止运作，只剩下其他组的硬盘运作，可靠性较低。如果以六个硬盘建RAID 01，镜射再用三个建RAID 0，那么坏一个硬盘便会有三个硬盘脱机。RAID 10远较RAID 01常用

![raid10](raid10.png)

篇幅所限，只列举以上几个常用级别，以下是各级别特点的对比图
![磁盘阵列比较表](raid.png '磁盘阵列比较表')

> 注：RAID2、3、4较少实际应用，因为RAID5已经涵盖了所需的功能，因此RAID2、3、4大多只在研究领域有实现，而实际应用上则以RAID5为主。RAID4有应用在某些商用机器上，像是NetApp公司设计的NAS系统就是使用RAID4的设计概念


## RAID软件实践
Linux中可以通过md(multi devices)模块来实现软RAID，使用`mdadm`命令来管理创建软RAID设备,mdadm支持的RAID级别：LINEAR,RAID0, RAID1,RAID4,RAID5,RAID6,RAID10  
`mdadm [mode] <raiddevice> [options] <component-device>`
> -C: 创建
>> -n #：使用#个块设备来创建此RAID  
>> -l #：指明RAID的级别  
>> -a {yes|no}：是否自动创建目标RAID设备的设备文件  
>> -c CHUNK_SIZE：指明块大小  
>> -x #：指明空闲盘的个数    

- `mdadm -C /dev/md0 -n 3 -l 5 -a yes -c 64k -x 1 /dev/sdb{5,6,7,8}`

> -D：查看详情  `mdadm -D /dev/md0`   
> -f：标记磁盘来为损坏  `mdadm /dev/md0 -f /dev/sdb5`  
> -r：移除磁盘  `mdadm /dev/md0 -f /dev/sdb5`  
> -a：添加磁盘  `mdadm /dev/md0 -r /dev/sdb5`   
> -S：停止md设备  `mdadm -S /dev/md0`    

**实例：**  
> 在CentOS6.7上创建一个可用空间为10G的RAID5设备, 要求其chunk大小为64K, 文件系统为ext4, 开机可自动挂载至/mnt/raid目录，同时一个空闲盘做备份  

> **思路：**  
> 1. 新建分区并调整类型: `fdisk /dev/sdb`   
> 2. 内核重读分区: `partx -a /dev/sdb`  
> 3. 创建RAID: `mdadm -C /dev/md0 -n 3 -l 5 -a yes -c 64k -x 1 /dev/sdb{5,6,7,8}`  
> 4. 格式化: `mke2fs -t ext4 /dev/md0`  
> 5. 挂载: `mount /dev/md0 /mnt/raid`    
> 6. 开机自动挂载：`vim /fstab`  

1. 准备环境: 添加/dev/sdb磁盘，分出4个区调整为Linux raid auto(system id: fd)  
![raid类型调整](raid_fdisk.png)

2. 内核重读分区
![内核重读分区](raid_partx.png)

3. 创建RAID设备  
![raid创建](raid_mdadm.png)

4. 创建文件系统  
![创建文件系统](raid_mkfs.png)

5. 挂载使用并设置开机自动挂载
![raid挂载](raid_mount.png)

**模拟故障**
1. 复制一文件到RAID设备上，以备查看RAID是否正常
2. 模拟损坏、移除和添加磁盘等操作
3. 查看RAID是否正常运行

![raid挂载](raid_faulty.png)

以上内容整理自马哥笔记


