---
title: "面试准备 checklist"
description: ""
date: "2025-03-27T16:48:23+08:00"
thumbnail: ""
categories:
  - ""
tags:
  - ""
draft: true
---
## 1. 算法
## 2. 编程语言
#### 2.1 Java
- JVM 内存模型
- 内存回收
    - G1
- 多线程
    - Volatile
    - CompletableFuture
    - StampedLock
    - ForkJoinPool
    - BlockingQueue
- 问题排查
    - 线程堆栈
    - 对象内存
#### 2.2 Golang
- 调度器 d
- 内存回收 
    - 逃逸分析
## 4. 数据库
#### 4.1 RocksDB
#### 4.2 Parquet 协议 d
#### 4.3 ORC 协议 d
#### 4.4 Paimon
1. LSM Tree 文件结构
2. 文件索引
3. 异步 Compaction
#### 4.5 Iceberg
1. 和 Paimon 的异同
## 5. 分布式系统
#### 5.1 分布式协议
1. CAP一致性协议
[CAP一致性协议及应用分析](https://tech.youzan.com/cap-coherence-protocol-and-application-analysis/)
2. Paxos 协议
[Paxos协议详解](https://chinalhr.github.io/post/distributed-systems-consensus-algorithm-paxos/)
3. Raft 协议
[Raft协议详解](https://zhuanlan.zhihu.com/p/32052223)
[Raft论文](https://docs.qq.com/doc/DY0VxSkVGWHFYSlZJ?_t=1609557593539)
4. ZAP 协议
5. Gossip 协议
6. CRAQ 协议
#### 5.2 Flink
- Flink State Backend 和持久化 d
- Flink Watermark 生成原理及作用 d
- Flink OperatorChain 生成过程
- Flink Operator 调度 d
- Flink SQL API、Stream API 任务解析及执行过程 d
- Flink 背压处理机制
- Flink HighAvailableService 原理及实现
- Flink Souce Sink Connector 源码分析 d
#### 5.3 Kafka
1. 架构设计
#### 5.4 Ray
1. Ray core 架构及实现
2. KubeRay
#### 5.5 Kubernetes
#### 5.6 JuiceFS
## 6. 操作系统
#### IPC
#### 内存
- 虚拟内存
#### 文件系统
- VFS
- inode
https://www.ruanyifeng.com/blog/2011/12/inode.html
- 页表
- mmap
- sendFile
https://xie.infoq.cn/article/a34cf4d2c6556d6c81be17303
#### 网络 IO
- IO 多路复用
#### 进程和线程
- 进程的虚拟内存，物理内存，共享内存
## 7. 网络
#### TCP/IP
#### 
## 8. AI

## 20250605
1. Flink 背压源码分析
2. Flink Shuffle
3. Flink
4. K8S 模块组成
5. K8S Operator