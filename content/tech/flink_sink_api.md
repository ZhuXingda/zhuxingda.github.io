---
title: 【Flink】Flink Data Sink API
date: 2024-11-28
tags:
    - Flink 
    - Paimon
categories:
    - "分布式系统"
---
新的 Flink Sink API 取代了旧的 SinkFunction API，以提供更灵活的功能
<!--more-->
Flink 版本：2.0
## Sink 和 SinkWriter
Sink 和 SinkWriter 接口是初级接口，保证 at-least-once
#### SinkWriter
- **write** 上游数据传入，执行具体写到外部的逻辑
- **flush** 在执行 `checkpoint` 或输出结束时会被调用，调用后将所有已传入但未写出的数据写出，保证 `at-least-once`
- **writeWatermark** 从上游传递的 watermark 会在此写入，可以用来实现一些特殊的逻辑
## SupportsWriterState 和 StatefulSinkWriter
在 Sink 接口的基础上继承 `SupportsWriterState` 即表示该 Sink 的 SinkWriter 是有状态的，需要实现 `StatefulSinkWriter` 接口
#### SupportsWriterState
- **restoreWriter** 从 State 恢复 StatefulSinkWriter 实例
- **getWriterStateSerializer** 定义 StatefulSinkWriter 的 State 序列化器
#### StatefulSinkWriter
StatefulSinkWriter 可以不用在 SinkWriter.flush 时保证将所有数据写出，而是将其存入状态中   
- **snapshotState** 执行 checkpoint 时被调用，将当前状态存入 State ()
## SupportsCommitter、CommittingSinkWriter 和 Committer
在 Sink 接口的基础上继承 `SupportsCommitter` 表示该 Sink 支持二段提交，保证 `exactly-once`。   
- CommittingSinkWriter 负责输出数据，逻辑上视为 preCommits；
- Committer 在 CommittingSinkWriter 输出后确保这段输出完全成功，执行 actually commits
#### SupportsCommitter
- **createCommitter** 创建 Committer
- **getCommittableSerializer** 创建 Committable 的序列化器
#### CommittingSinkWriter
- **prepareCommit** 在 SinkWriter.flush 后，StatefulSinkWriter.snapshotState 前被调用，产生 Committables 并输出到下游
#### Committer
- **commit** 执行真正的 commit 操作，根据 CommitRequest 里的 Committable 执行特定的 commit 操作，使 CommittingSinkWriter 的写入完全成功
## StreamOperator
#### SinkWriterOperator
SinkWriterOperator 是 SinkWriter 对应的 StreamOperator，其执行流程大致如下
1. **初始化**   
    1. SinkWriterOperator 初始化 SinkWriterStateHandler 和 CommittableSerializer
    2. setup
    3. initializeState   从 State 中恢复未提交的 Committable (如果存在，兼容旧接口) 和 endOfInput 状态
2. **processElement** 处理写入的数据
3. **processWatermark** 处理 watermark   
4. **prepareSnapshotPreBarrier** 执行 Checkpoint 前
    1. 调用 SinkWriter.flush
    2. emitCommittables 将 Committables 封装成 CommittableMessage 并输出到下游
        1. 调用 CommittingSinkWriter#prepareCommit
        2. 如果有从 State 恢复的 Committable 先提交
        3. 提交 CommittingSinkWriter 产生的 Committables
        4. CommittableMessage 包含一批 Committable 的汇总信息 CommittableSummary 和每个 Committable 的详细信息 CommittableWithLineage
5. **snapshotState** 执行 Checkpoint
    1. StatefulSinkWriterStateHandler#snapshotState 更新 StatefulSinkWriter 的 State
6. **endInput** 输入结束
    1. 更新 endOfInput 和 endOfInput 状态
    2. 调用 SinkWriter.flush
    3. 将 SinkWriter 剩下的 Committables 都推送到下游（注意不是保存到 State）
#### CommitterOperator
CommitterOperator 是 Committer 对应的 StreamOperator，也是 SinkWriterOperator 的下游算子，其执行流程大致如下
1. **初始化**
    1. CommitterOperator
    2. setup
    3. initializeState 
        1. 调用 SupportsCommitter#createCommitter 创建 Committer
        2. 从 State 中恢复未提交的 Committables 并保存至 CommittableCollector
        3. 如果 2 中有未提交 Committables 存在，则调用 CommitterOperator#commitAndEmitCheckpoints 将其和 `CompletedCheckpointId` 一起提交
2. **processElement**   
CommittableCollector 处理 CommittableMessage
3. **snapshotState** 执行 Checkpoint   
将 CommittableCollector 中未提交的 Committables 保存到 State
4. **notifyCheckpointComplete** Checkpoint 执行完成   
commitAndEmitCheckpoints
    1. 从 CommittableCollector 中取出所有这次 Checkpoint 及之前的未提交的 Committables
    2. CheckpointCommittableManager#commit 提交每一个取出的 Committable，组成 CommitRequest 交由 Committer 执行，如果 Sink 实现了 SupportsPostCommitTopology 接口则将 Committable 和 CheckpointId 继续推到下游算子
    3. 从 CommittableCollector 中移除已提交的 Committable
5. **endInput** 输入结束
    1. 更新 endInput
    2. 将 CommittableCollector 中未提交的 Committable 全部提交
## 自定义 Sink 拓扑
1. **SupportsPreWriteTopology 接口**   
    支持在 SinkWriter 之前执行自定义逻辑的算子
2. **SupportsPreCommitTopology 接口**   
    支持在 SinkWriter 之后 Committer 之前执行自定义逻辑的算子
3. **SupportsPostCommitTopology 接口**   
    支持在 Committer 之后执行自定义逻辑的算子
## Paimon 是如何实现 Flink Sink 的
