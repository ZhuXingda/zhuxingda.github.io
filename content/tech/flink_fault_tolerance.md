---
title: 【Flink】Fault Tolerance 原理和实现
date: 2023-10-12
tags:
    - Flink 
categories:
    - "分布式系统"
---
Flink 是如何在程序异常后恢复运行的，本文简述一下其中的原理。
<!--more-->
## 原理
#### Flink 容错机制的实现原理
1. Flink 程序的状态（**state**）保存在 state backend 中；
2. Flink 定时执行检查点（**checkpoint**）时，会对所有算子的状态做快照（**snapshot**），并把这些快照持久化保存到一个稳定的地方（比如分布式文件系统）；
3. 当任务出现故障，Flink 会重启任务并从最新的快照处重新执行任务，从而实现容错。
#### 状态快照的执行过程
1. Taskmananger 从 Jobmanager 接收到执行 **checkpoint** 的指令后，会让所有 source 算子记录当前的 **offsets**，并在数据流中插入一个编号的 **checkpoint barrier**;
2. **checkpoint barrier** 会沿着数据流传递到所有的算子，各算子在接收到之后执行 **state snapshot**，执行完之后继续将 barrier 向下游传递。
3. 有两个输入流的算子（如CoprocessFunction）会执行屏障对齐 (**barrier aligment**)，使快照包含两个输入流在 **checkpoint barrier** 之前的所有 events 产生的 state；
#### 状态快照的恢复过程
1. Flink 重新启动部署整个分布式任务，从最新的 **checkpoint** 恢复每个算子的状态；
2. 数据源从最新的 **checkpoint** 恢复 **offset**，继续消费数据流；
3. 如果状态快照是增量式的，算子会先恢复到最新的全量快照，再逐个按增量快照更新。
#### 非对齐的 checkpoint
![](https://nightlies.apache.org/flink/flink-docs-release-2.0/fig/stream_unaligning.svg)
算子在执行 checkpoint 时也可以不必 **barrier aligment**，其原理是： 
1. 算子在某个 input buffer 接收到 checkpoint 的第一个 barrier 时，就将其添加到 output buffer 的最后面（立即输出到下游）；
2. 算子将这次 checkpoint 对应的所有未输出到下游的数据（包含上游算子的 output buffer 在 barrier 之前的数据，这个算子 input buffer 在 barrier 之后的数据，这个算子 output buffer 里的所有数据）做 snapshot 保存到算子的 state
3. 恢复时先将状态里所有 inflght 数据恢复到算子，再开始处理上游传来的数据
#### 不同程度的容错机制
任务发生故障时，根据不同的容错机制可能会出现以下结果：
  - 最多一次 **at-most-once** ：Flink 不对故障做恢复，数据可能会丢失；
  - 至少一次 **at-least-once** ：Flink 从故障恢复，但可能产生重复结果；   
**at-least-once** 需要数据源 **source** 支持回朔到快照发生的进度，比如 Kafka 回到某个 offset，同时整个数据链路保证在成功执行一次 checkpoint 快照时，所有数据源已发出的数据都被成功 sink 到下游。   
  - 精确一次 **exactly-once** ：Flink 从故障恢复，没有结果丢失和重复。   
**exactly-once** 在 **at-least-once** 的基础上还需要数据输出 **Sink** 支持事务写入或者幂等写入。

## 实现
#### JobMaster 发起 checkpoint
###### JobMaster 创建 ExecutionGraph
1. JobMaster#JobMaster
2. JobMaster#createScheduler
3. DefaultSlotPoolServiceSchedulerFactory#createScheduler
4. DefaultSchedulerFactory#createInstance
5. DefaultScheduler#DefaultScheduler
6. SchedulerBase#SchedulerBase
7. SchedulerBase#createAndRestoreExecutionGraph
8. DefaultExecutionGraphFactory#createAndRestoreExecutionGraph
9. DefaultExecutionGraphBuilder#buildGraph

###### ExecutionGraph 通过 CheckpointCoordinator 开启 checkpoint
1. DefaultExecutionGraph#enableCheckpointing
2. CheckpointCoordinator#createActivatorDeactivator   
    在 CheckpointCoordinator 外包装一层 JobStatusListener，Job 状态变成 RUNNING 时开始 checkpoint schedule，从 RUNNING 变成其他状态时停止 checkpoint。
3. CheckpointCoordinatorDeActivator#CheckpointCoordinatorDeActivator
4. CheckpointCoordinator#startCheckpointScheduler
5. CheckpointCoordinator#scheduleTriggerWithDelay   
用一个 ScheduledExecutor 定期触发 ScheduledTrigger 执行
6. CheckpointCoordinator.ScheduledTrigger#run
7. CheckpointCoordinator#triggerCheckpoint

###### CheckpointCoordinator 触发 CheckpointTriggerRequest 执行
1. CheckpointCoordinator.CheckpointTriggerRequest 创建一个 CheckpointTriggerRequest   
2. CheckpointRequestDecider#chooseRequestToExecute 对 CheckpointTriggerRequest 做一个限流   
3. CheckpointCoordinator#startTriggeringCheckpoint 触发 CheckpointTriggerRequest 执行   
  1. **Source Operator 的 OperatorCoordinator 执行 checkpoint**      
    1. OperatorCoordinatorCheckpoints#triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion      
    2. OperatorCoordinatorCheckpoints#triggerAndAcknowledgeAllCoordinatorCheckpoints 触发所有 OperatorCoordinator 执行 checkpoint，并校验执行结果   
    3. OperatorCoordinatorCheckpoints#triggerAllCoordinatorCheckpoints   
    4. OperatorCoordinatorCheckpoints#triggerCoordinatorCheckpoint   
    5. OperatorCoordinatorHolder#checkpointCoordinator   
    6. OperatorCoordinatorHolder#checkpointCoordinatorInternal   
        - OperatorCoordinatorHolder#closeGateways 关闭所有和 SubTask 通讯的 SubtaskGateway   
        - OperatorCoordinatorHolder#completeCheckpointOnceEventsAreDone 如果有已发送给 SubTask 但还未执行完成的 Events 等待其执行完成      
  2. **JobMaster snapshot state**   
  3. **触发 StreamTask 执行 checkpoint**   
    1. CheckpointCoordinator#triggerCheckpointRequest   
    2. CheckpointCoordinator#triggerTasks   
        这里只对 CheckpointPlan 里的 tasksToTrigger 发送 checkpoint 请求，即 Source Tasks   
    3. Execution#triggerCheckpoint   
    4. Execution#triggerCheckpointHelper   
    5. TaskManagerGateway#triggerCheckpoint   
    6. RpcTaskManagerGateway#triggerCheckpoint  RPC 请求触发 TaskManager 执行 checkpoint   
    6. TaskExecutorGateway#triggerCheckpoint   
    7. TaskExecutor#triggerCheckpoint   

#### SourceStreamTask 触发 checkpoint
1. Task#triggerCheckpointBarrier
2. SourceOperatorStreamTask#triggerCheckpointAsync
3. SourceOperatorStreamTask#triggerCheckpointNowAsync   
    如果 Source 的 Source Reader 不是 ExternallyInducedSourceReader 则直接执行异步 checkpoint 
4. StreamTask#triggerUnfinishedChannelsCheckpoint   
    创建 CheckpointBarrier 并下发到每个 InputGate 的每个 InputChannel
5. CheckpointBarrierHandler#processBarrier  
    CheckpointBarrierHandler 接口有两个实现：
    - CheckpointBarrierTracker 对应 checkpoint mode 为 `at-least-once`，不会阻塞 InputChannel 的输入，直到所有 InputChannel 都收到 barrier 调用 CheckpointBarrierHandler#notifyCheckpoint 触发 checkpoint 执行   
    - SingleCheckpointBarrierHandler 对应 checkpoint mode 为 `exactly-once`，接收和记录 barriers 并交由 **BarrierHandlerState** 决定何时触发 checkpoint 以及对 Inputchannel 的操作    
6. SingleCheckpointBarrierHandler#processBarrier    
    SingleCheckpointBarrierHandler 支持 `aligned` 和 `unaligned` 两种 checkpoint 模式
7. SingleCheckpointBarrierHandler#markCheckpointAlignedAndTransformState    
    将 barrier 交由 BarrierHandlerState 处理并记录 InputChannel 的 barrier 对齐情况   
8. BarrierHandlerState#barrierReceived  
    BarrierHandlerState 接收 barrier 并根据 barrier 转换自身的类型，由不同类型代表算子处理 checkpoint 时的多个状态：       
    - **WaitingForFirstBarrier** `aligned` 模式下等待第一个 barrier  
    - **CollectingBarriers** `aligned` 模式下等待所有 barrier  
    - **AlternatingWaitingForFirstBarrier** `aligned` 模式下等待第一个 barrier，有超时限制
    - **AlternatingCollectingBarriers** `aligned` 模式下等待所有 barrier，有超时限制
    - **AlternatingWaitingForFirstBarrierUnaligned** `unaligned` 模式下等待第一个 barrier，有超时限制
    - **AlternatingCollectingBarriersUnaligned** `unaligned` 模式下等待所有 barrier，有超时限制
9. AbstractAlignedBarrierHandlerState#barrierReceived  以 `aligned` 模式为例   
    SourceTask 并不暂停 InputChannel 的输入，所有 barrier 都收到后触发全局 checkpoint   
10. AbstractAlignedBarrierHandlerState#triggerGlobalCheckpoint  
    执行全局 checkpoint ，完成后恢复所有 InputChannel 的输入，并进入 WaitingForFirstBarrier 的状态    
11. SingleCheckpointBarrierHandler#triggerCheckpoint
12. CheckpointBarrierHandler#notifyCheckpoint   
    回调 StreamTask 执行 checkpoint

#### StreamTask 执行 checkpoint
1. StreamTask#triggerCheckpointOnBarrier
2. StreamTask#performCheckpoint
3. SubtaskCheckpointCoordinatorImpl#checkpointState
    对所有 SubTask 执行 checkpoint
    - OperatorChain#prepareSnapshotPreBarrier 所有 SubTask 执行 StreamOperator#prepareSnapshotPreBarrier
    - OperatorChain#broadcastEvent 向所有 SubTask 的下游广播 CheckpointBarrier
    - ChannelStateWriter#finishOutput 如果是 unaligned 的 checkpoint 停止持久化 channel state
    - SubtaskCheckpointCoordinatorImpl#takeSnapshotSync 所有 SubTask 执行 OperatorChain#snapshotState，这一步会传入 CheckpointStreamFactory 用于输出 State 持久化后的数据流，根据配置数据流会被写入不同的 State Backend
    - SubtaskCheckpointCoordinatorImpl#finishAndReportAsync 
    SubTask 执行 checkpoint 结束后通知 JobMaster

#### StreamTask 接收上游的 barrier
1. OperatorChain#broadcastEvent    
    遍历所有 RecordWrters 广播 CheckpointBarrier
2. RecordWriterOutput#broadcastEvent
3. RecordWriter#broadcastEvent
4. ResultPartitionWriter#broadcastEvent    
    将 CheckpointBarrier 写到 Buffer    
... 省略从上游算子的 Output Buffer 到下游算子的 Input Buffer     
5. AbstractStreamTaskNetworkInput#emitNext
6. CheckpointedInputGate#pollNext
7. CheckpointedInputGate#handleEvent
8. CheckpointBarrierHandler#processBarrier   
    到这一步就和前面 SourceStreamTask 触发 checkpoint 一样了

#### JobMaster 确认 checkpoint 执行结果
1. AsyncCheckpointRunnable#run
2. AsyncCheckpointRunnable#reportCompletedSnapshotStates
3. TaskStateManagerImpl#reportTaskStateSnapshots
4. RpcCheckpointResponder#acknowledgeCheckpoint RPC 请求 JobMaster checkpoint 执行结束
5. JobMaster#acknowledgeCheckpoint
6. SchedulerBase#acknowledgeCheckpoint
7. ExecutionGraphHandler#acknowledgeCheckpoint
8. CheckpointCoordinator#receiveAcknowledgeMessage
9. PendingCheckpoint#acknowledgeTask    
    PendingCheckpoint 记录完成 checkpoint 的 Task
10. CheckpointCoordinator#completePendingCheckpoint
11. CheckpointCoordinator#finalizeCheckpoint

#### 从 checkpoint 恢复