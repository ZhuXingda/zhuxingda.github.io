---
title: "【Paimon】Paimon 主键表数据读取逻辑分析"
date: "2025-06-02T21:42:11+08:00"
categories:
  - "分布式系统"
tags:
  - "Paimon"
draft: true
---
Paimon 主键表的 ``Merge On Read`` 和 ``Copy On Write`` 模式分别需要在读和写数据时对同一 partition/bucket 下属于不同 Sorted Runs 的数据文件做合并，合并的过程在
<!--more-->
## Sort Engine
## Merge Engine