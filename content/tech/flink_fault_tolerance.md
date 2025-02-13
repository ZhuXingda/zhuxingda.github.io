---
title: 【Flink 基础】Fault Tolerance 实现原理
date: 2023-10-12
tags:
    - Flink 
categories:
    - "大数据"
---
Flink 是如何在程序异常结束后恢复的，本文简述一下其中的原理。
<!--more-->
### Flink 容错机制的实现原理
1. Flink 程序的状态（**state**）保存在 state backend 中；
2. Flink 定时执行检查点（**checkpoint**）时，会对所有算子的状态做快照（**snapshot**），并把这些快照持久化保存到一个稳定的地方（比如分布式文件系统）；
3. 当任务出现故障，Flink 会重启任务并从最新的快照处重新执行任务，从而实现容错。
### 状态快照的执行过程
1. Taskmananger 从 Jobmanager 接收到执行 **checkpoint** 的指令后，会让所有 source 算子记录当前的 **offsets**，并在数据流中插入一个编号的 **checkpoint barrier**;
2. **checkpoint barrier** 会沿着数据流传递到所有的算子，各算子在接收到之后执行 **state snapshot**，执行完之后继续将 barrier 向下游传递。
3. 有两个输入流的算子（如CoprocessFunction）会执行屏障对齐 (**barrier aligment**)，使快照包含两个输入流在 **checkpoint barrier** 之前的所有 events 产生的 state；
### 状态快照的恢复过程
1. Flink 重新启动部署整个分布式任务，从最新的 **checkpoint** 恢复每个算子的状态；
2. 数据源从最新的 **checkpoint** 恢复 **offset**，继续消费数据流；
3. 如果状态快照是增量式的，算子会先恢复到最新的全量快照，再逐个按增量快照更新。
### 精确一次（exactly once）
任务发生故障时，根据不同的容错机制可能会出现一下结果：
  - 最多一次（**at most once**）：Flink 不对故障做恢复，数据可能会丢失；
  - 至少一次（**at least once**）：Flink 从故障恢复，但可能产生重复结果；
  - 精确一次（**exactly once**）：Flink 从故障恢复，没有结果丢失和重复。   

实现 **at least once** 需要数据源 **source** 支持回朔到快照发生的进度，比如 Kafka 回到某个 offset。   
实现 **exactly once** 在 **at least once** 的基础上还需要数据输出 **sink** 支持事务写入或者幂等写入。