---
title: Hadoop 常用命令
date: 2018-06-22 10:43:19
categories: hadoop
tags: [hadoop, hdfs, yarn]
---

记录一些常用的 hadoop 相关的命令
<!-- more -->

### hadoop

> `distcp`  
> `version`：版本号 
> `fs`：由 `hdfs dfs` 代替  
> `job`：由 `mapred job` 代替  
> `fsck`：由 `hdfs fsck` 代替  
> `balancer`：由 `hdfs balancer` 代替  
> `datanode`：由 `hdfs datanode` 代替  
> `namenode`：由 `hdfs namenode` 代替  
> `dfsadmin`：由 `hdfs dfsadmin` 代替  
> `secondarynamenode`：由 `hdfs secondarynamenode` 代替  

```
hadoop version
```

### hdfs

> `dfs`  
> `fsck`  
> `dfsadmin`  
> `datanode`  
> `balancer`  
> 

- `dfs`

> 以下例子中 /hdfs、hdfs://xx 为hdfs文件系统路，/tmp为本地文件系统路径

```
hdfs dfs -ls -h /hdfs/a hdfs://10.0.0.1:8020/hdfs
hdfs dfs -get /hdfs/a.log /tmp
hdfs dfs -put ./local.conf hdfs://10.0.0.1:8020:/hdfs
hdfs dfs -cat /hdfs/a.log /hdfs/b.conf
hdfs dfs -cp -f -p /hdfs/dir1/aa /hdfs/dir2/
hdfs dfs -mv /hdfs/dir1 /hdfs/dir2 /hdfs/tmp
hdfs dfs -rm -r -f -skipTrash /hdfs/a
hdfs dfs -count -v /hdfs/*
hdfs dfs -du -s -h /hdfs/*
hdfs dfs -df -h /hdfs
hdfs dfs -mkdir -p /hdfs/a/b/c
hdfs dfs -expunge
hdfs dfs -chmod 640 /hdfs/a.log
hdfs dfs -chown liulin /hdfs/b.conf
......

```

- `fsck`

`-delete`：删除损坏的文件  
`-move`：移动损坏的文件到/lost+found目录  
`-files`：打印被诊断的文件  
`-files -blocks`：打印被诊断的文件的块信息  
`-files -blocks -locations`：打印每个块的位置信息  
`-files -blocks -racks`：打印数据块的网络拓扑结构  
`-includeSnapshots`：如果给定的路径包含快照的路径或者快照在该路径下，则包含快照的数据  
`-list-corruptfileblocks`：打印丢失的块列表以及块所属的文件  
`-openforwrite`：打印正在被写入的文件  

```
hdfs fsck /hdfs
hdfs fsck -delete /hdfs
hdfs fsck -move /hdfs
```

- `dfsadmin`


`-safemode enter|leave|get|wait`：安全模式维护命令
`-setBalancerBandwidth`：设置数据均衡的速度
`-report [-live] [-dead] [-decommissioning]`：报告文件系统的信息和统计，其他的选项可以用来过滤节点  
`-printTopology`：打印哪些有在Namenode报告的节点的网络拓扑结构

```
hdfs dfsadmin -safemode get
hdfs dfsadmin -setBalancerBandwidth 104550400 	# 设置为100M/s
```


- `datanode`

- `balancer`

`-threshold`：默认均衡的阀值  
`-policy`：  
`-include -f host1,host2`：指定datanode  
`-exclude -f host3,host4`：排除  

```
hdfs balancer -threshold 1 -policy datanode 
```

- `getconf`

```
hdfs getconf -namenodes
hdfs getconf -nnRpcAddresses
```

- `oev`
`hadoop edit文件离线查看器`


- `oiv`
`hadoop image文件查看器`


### yarn

> `application`  
> `applicationattempt`  
> `container`  
> `logs`  
> `node`  

- `application`

`-list`               ：从RM查看application列表  
`-kill <AppId>`       ：杀死application  
`-status <AppId>`     ：打印指定application状态  
`-appTypes <Types>`   ：与-list一起使用，基于传入的逗号分隔的application types列表过滤  
`-appStates <States>` ：与-list一起，可取ALL, NEW, NEW_SAVING, SUBMITTED, ACCEPTED, RUNNING, FINISHED, FAILED, KILLED，可多个用逗号隔开

```
yarn application -list
yarn application -list -appStates FAILED,NEW
yarn application -status application_1534006015382_0379
```

- `applicationattempt`

```
yarn applicationattempt -list application_1530168666576_3446336
yarn applicationattempt -status appattempt_1437364567082_0106_000001
```

- `container`

```
yarn container -list appattempt_1501040274955_1945_000001
yarn container -status container_1501040274955_1945_01_000002
```

- `logs`

```
yarn logs -applicationId application_1491808105321_22924 [-appOwner hadoop]
```

- `node`

```
yarn node -list
yarn node -list -all
yarn node -list -states NEW,RUNNING
yarn node -status data01.liulin.com:8041
```


### mapred

```
mapred job -list
mapred job -logs job_xxx
mapred job -kill job_xxx
mapred job -status job_xxx
```



## 相关文章

https://blog.csdn.net/qianshangding0708/article/details/47423585