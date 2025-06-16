---
title: "【Piamon】Paimon 数据表 Merge Engine"
description: ""
date: "2025-05-26T23:00:24+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Paimon"
draft: true
---
在前文 Paimon 主键表数据写入过程有提到对 LSM Tree 的 Compact 操作，Compact 过程中
<!--more-->
在将数据写入 Paimon 主键表时，LSM Tree 的 sorted runs 会不断增加造成读数据时合并数据的压力增大，因此需要 Compact 将 sorted runs 的数量限制在合理范围内   

非主键表也支持 Compact，其功能主要是合并小文件
## 1. Compact
#### 1.1 Compaction 模式
Paimon 采用 universal compaction 模式：
- Full Compaction（）：
- Level0 Compaction 
## 2. Sort Engine
## 3. Merge Engine
