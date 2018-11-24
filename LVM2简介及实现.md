---
title: LVM2简介及实现
date: 2016-05-31 17:23:35
categories: Linux基础
tags: LVM2
---

简要介绍LVM逻辑卷及简单运用

<!--more-->

## 简介
LVM是逻辑卷管理（Logical Volume Manager）的简称，它是建立在物理存储设备之上的一个抽象层，允许你生成逻辑存储卷,与直接使用物理存储在管理上相比,提供了更好灵活性。LVM将存储虚拟化,使用逻辑卷,你不会受限于物理磁盘的大小，另外，与硬件相关的存储设置被其隐藏，可以不用停止应用或卸载文件系统来调整卷大小或数据迁移，可以动态的来扩展或缩小逻辑卷大小，同时逻辑卷支持快照功能
![LVM示意图](lvm2.png)

**说明**
- **PV**：physical volume 物理卷在逻辑卷管理系统最底层，可为整个物理硬盘或实际物理硬盘上的分区
- **VG**：volume group 卷组建立在物理卷上，一卷组中至少要包括一物理卷，卷组建立后可动态的添加卷到卷组中，类似磁盘的扩展分区，本身不能创建文件系统，需再划分成LV来使用，一个逻辑卷管理系统工程中可有多个卷组
- **LV**：logical volume 逻辑卷建立在卷组基础上，卷组中未分配空间可用于建立新的逻辑卷，逻辑卷建立后可以动态扩展和缩小空间
- **PE**：physical extent 物理区域是物理卷中可用于分配的最小存储单元，物理区域大小在建立卷组时指定，一旦确定不能更改，同一卷组所有物理卷的物理区域大小需一致，新PV加入到VG后，PE大小自动更改为VG中定义的PE大小

**管理工具**
> PV管理工具
- pvs: 简要显示物理卷信息  
- pvdisplay: 显示物理卷详细信息    
- pvcreate: 创建物理卷  
- pvremove: 移除物理卷 

> VG管理工具
- vgs: 简要显示卷组信息
- vgdisplay: 显示卷组详细信息   
- vgcreate: 创建卷组
- vgextend: 扩展卷组
- vgreduce: 缩小卷组  
- vgremove：删除卷组

> LV管理工具
- lvs: 简要显示逻辑卷信息
- lvdisplay: 显示逻辑卷详细信息
- lvcreate: 创建逻辑卷
-L: 大小[mMgGtT]  
-n: 指定创建卷名   
-s: 指定创建为快照   
-p: 权限[r|rw],默认rw  
- lvextend: 扩展逻辑卷  
-L: 指定大小  
- resize2fs: 调整文件系统
- lvremove: 缩小逻辑卷

## 创建LVM
创建LVM前，使用`fdisk`将磁盘分区，并调整分区类型为8e(Linux LVM)
![lvm分区类型](lvm_8e.png)

1. 创建物理卷PV
`pvcreate /dev/sdc5 /dev/sdc6`  
![创建PV](pvcreate.png)

2. 创建卷组VG，并将PV加入卷组中
`vgcreate VG_name /dev/sdc5 /dev/sdc6`  
![创建VG](vgcreate.png)

3. 基于卷组创建逻辑卷LV
`lvcreate -n LV_name -L 2[mMgGtT] VG_name`  
![创建PV](lvcreate.png)

4. 为LV创建文件系统
`mkfs -t ext4 [-b 1024 -L MYLV] /dev/VG_name/LV_name` 
![创建文件系统](lv_mkfs.png)

5. 挂载使用
`mount /dev/VG_name/LV_name /mnt/LV`
![挂载逻辑卷](lv_mount.png)

- **补充说明**
![lvm设备映射关系](lvm_link.png)

以上为简单的创建逻辑卷过程

## 移除操作
PV/VG的移除需确保在LV未使用的空间上进行，删除整个lvm需先移除LV，再删VG，最后删PV
- ![PV/VG添加移除操作](vg_reduce.png)
- ![lvm删除](lvm_remove.png)

## 扩展逻辑卷
在VG大小范围内可创建多个逻辑卷，每个卷可以扩展缩小，缩小有风险，扩展可以在线执行，无需卸载逻辑卷，如果卷组空间不够，则需PV来扩充VG空间再进行扩展LV(扩充见上图)  
1. `lvextend -L [+]1G /dev/VG_name/LV_name` : 扩展逻辑卷(+表示增加的量，不加表示增加至)    
2. `resize2fs /dev/VG_name/LV_name`: 调整文件系统(物理边界)   
![逻辑卷扩展](lvextend.png)

## 缩小逻辑卷
缩小逻辑卷没有办法在线执行，如果在线执行缩小的操作，会导致数据丢失。如需缩小要先卸载再执行，但是此操作有风险，非必要情况尽量避免，同时确保空间足够容纳已有数据
1. `umount /dev/VG_name/LV_name`
2. `e2fsck -f /dev/VG/LV`
3. `resize2fs /dev/VG_name/LV_name [+]2[mMgGtT]`
4. `lvreduce [+]2[mMgGtT] /dev/VG_name/LV_name`
5. `mount /dev/VG_name/LV_name /mnt/mylv`
![逻辑卷缩小](lvreduce.png)

## 逻辑卷快照
快照卷(snapshot)为确保得到一个分区的完整某一时刻的备份创建，因为在备份过程中，一个分区的文件或者数据库可能发生改变，而导致备份出来的数据不完整，快照创建初期不占用实际空间，全部数据均指向源数据，同时监控数据变化，当数据发生变化时才将原数据保存到快照，所以快照的大小取决于原数据的变化量。
1. `lvcreate -L 2[mMgGtT] -p r -s -n snapshot_lv_name original_lv_name`
2. `mount /dev/VG_NAME/snapshot_lv_name /mnt/snap`
![lvm快照](lvm_snapshot.png)

LVM2就简单介绍到此







