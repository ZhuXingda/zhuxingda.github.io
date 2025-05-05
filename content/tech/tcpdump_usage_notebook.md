---
title: "【Linux】Linux 服务器抓包记录"
description: ""
date: "2025-04-29T16:40:40+08:00"
thumbnail: ""
categories:
  - "操作系统"
tags:
  - "Network"
---
经常遇到需要在服务器抓包分析问题的情况，这里记录一下常用的抓包工具。
<!--more-->
## tcpdump
###### 抓包查看 Flink JobManager 往监控指标服务器上传的指标数据   
因为 Flink JobManager 运行在 Kubernetes 容器中，所以往外发送请求时是从 Pod 内的虚拟网卡到 Bridge cni0，然后在宿主机上通过 iptables 转发到宿主机的网卡 bond0
```shell
$ tcpdump -i bond0 dst host 10.224.0.1 and -i flannel.1 src host 10.100.4.57 -A -vv -w dump_http_request.pcap
```
