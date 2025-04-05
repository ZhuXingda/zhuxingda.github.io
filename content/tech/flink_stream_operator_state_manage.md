---
title: Flink Stream Operator 状态控制
date: 2024-11-28
tags:
    - Flink 
categories:
    - "分布式系统"
draft: true
---

#### org.apache.flink.streaming.api.operators.StreamOperator    
    基本接口
###### prepareSnapshotPreBarrier
在算子将 checkpoint barrier 往下游推之前调用，对于一些没有状态或者状态不需要持久化的算子，可以利用该接口在 checkpoint 前执行一些特定的逻辑，比如 SinkWriter 的 flush 操作。
###### snapshotState
对算子的 state 做快照，
###### notifyCheckpointComplete
###### notifyCheckpointAborted
#### org.apache.flink.streaming.api.operators.AbstractStreamOperator
继承 StreamOperator 接口，内部使用 StreamOperatorStateHandler 对象来实际执行 snapshotState notifyCheckpointComplete notifyCheckpointAborted
#### org.apache.flink.streaming.api.operators.StreamOperatorStateHandler