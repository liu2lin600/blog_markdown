---
title: kubectl 常用命令
date: 2019-12-20 13:07:29
tags: ['kubernetes', 'docker']
---

记录一些常用的 kubectl 相关的命令
<!-- more -->

## create


## expost


## run


## set


## explain

`kubuctl explain <type>.<fieldName>[.<fieldName>] [option]`



```
kubectl explain pods
kubectl explain pods.spec.containers.ports
```

## get


> `-l`: 


## describe


## exec 


## logs


## label
`kubectl label [--overwrite] (-f FILENAME | TYPE NAME) K1=V1 ... [--resource-version=version]
[options]`

标签的相关操作，包括增删改

```
kubectl label --overwrite pods foo status=unhealthy
kubectl label -f pod.json status=unhealthy
```


## version

## api-version

## config

## api-version