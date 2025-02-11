---
title: Flink Stream Operator 状态控制
date: 2024-11-28
tags:
    - Flink 
categories:
    - "Software Development"
draft: true
---

#### org.apache.flink.streaming.api.operators.StreamOperator    
    基本接口
###### prepareSnapshotPreBarrier
在算子执行snapshot前调用，用于
###### snapshotState
###### notifyCheckpointComplete
###### notifyCheckpointAborted
#### org.apache.flink.streaming.api.operators.AbstractStreamOperator
继承 StreamOperator 接口，内部使用 StreamOperatorStateHandler 对象来实际执行 snapshotState notifyCheckpointComplete notifyCheckpointAborted
#### org.apache.flink.streaming.api.operators.StreamOperatorStateHandler