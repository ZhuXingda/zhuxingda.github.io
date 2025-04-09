---
title: "【Flink】Flink Watermark 产生和传递原理"
date: "2025-03-25T19:27:37+08:00"
categories:
  - "分布式系统"
tags:
  - "Flink"
---
Flink Watermark 的产生原理和传递过程
<!--more-->
## 1. 产生 Watermark
#### 1.1 定义 WatermarkStrategy 并在创建 SourceOperator 时传入  
**WatermarkStrategy** 配置传入 SourceOperatorFactory   
1. StreamExecutionEnvironment#fromSource
2. DataStreamSource#DataStreamSource
3. SourceTransformation#SourceTransformation
4. SourceTransformationTranslator#translateInternal
5. SourceOperatorFactory#SourceOperaotorFactory
6. SourceOperatorFactory#createStreamOperator
7. SourceOperator#SourceOperator

#### 1.2 初始化 SourceOperator 时创建 TimestampsAndWatermarks 对象
1. SourceOperator#open
2. TimestampsAndWatermarks#createProgressiveEventTimeLogic  
TimestampsAndWatermarks 接口定义了时间戳提取和 watermark 生成   
3. ProgressiveTimestampsAndWatermarks#ProgressiveTimestampsAndWatermarks
4. ProgressiveTimestampsAndWatermarks#startPeriodicWatermarkEmits   
5. ProgressiveTimestampsAndWatermarks#triggerPeriodicEmit 如果 `pipeline.auto-watermark-interval` 配置不为 0 开启周期性触发 watermark

#### 1.3 SourceOperator 在开始推送数据时创建 MainOutput 和 SplitLocalOutput
1. SourceOperator#emitNext
2. SourceOperator#emitNextNotReading
3. SourceOperator#initializeMainOutput
4. TimestampsAndWatermarks#createMainOutput
5. ProgressiveTimestampsAndWatermarks#createMainOutput 创建 MainOutput 和 SplitLocalOutput

#### 1.4 触发 MainOutput 周期性输出 Watermark
1. ProgressiveTimestampsAndWatermarks#triggerPeriodicEmit
2. SourceOutputWithWatermarks#emitPeriodicWatermark
3. WatermarkGenerator#onPeriodicEmit
4. BoundedOutOfOrdernessWatermarks#onPeriodicEmit 回到 WatermarkStrategy 里配置的 WatermarkGenerator，调用定义输出 watermark
5. WatermarkOutput#emitWatermark

#### 1.5 触发 SplitLocalOutputs 周期性输出 Watermark
1. ProgressiveTimestampsAndWatermarks#triggerPeriodicEmit
2. SplitLocalOutputs#emitPeriodicWatermark   
    2.1 SourceOutputWithWatermarks#emitPeriodicWatermark 后面的逻辑同 MainOutput   
    2.2 WatermarkOutputMultiplexer#onPeriodicEmit    
    2.3 WatermarkOutputMultiplexer#updateCombinedWatermark    
    更新 combinedWatermark 并推送到 underlyingOutput   
        2.3.1 CombinedWatermarkStatus#updateCombinedWatermark    
        取所有 partialWatermarks 里的最小值作为 combinedWatermark   
        2.3.2 WatermarkToDataOutput#emitWatermark   
        2.3.3 PushingAsyncDataInput.DataOutput#emitWatermark    
        将 watermark 推送到下游   

#### 1.6 DataSource 触发 MainOutput watermark 更新
1. SourceOutput#collect(T, long) 这里需要 DataSource 支持，在 [【Flink】Data Source API 结构及实现](../flink_source_api) 里有介绍
2. SourceReaderBase.SourceOutputWrapper#collect(T, long)
3. SourceOutputWithWatermarks#collect(T, long)    
    3.1 TimestampAssigner#extractTimestamp 根据定义从数据和传入的时间戳中提取 watermark   
    3.1 PushingAsyncDataInput.DataOutput#emitRecord 先将带 watermark 的数据推送到下游   
    3.2 WatermarkGenerator#onEvent    
        3.2.1 BoundedOutOfOrdernessWatermarks#onEvent 更新最大的 watermark （以 BoundedOutOfOrdernessWatermarks 为例）

#### 1.7 TimestampsAndWatermarksOperator 产生 Watermark


## 2. 传递 Watermark
1. BoundedOutOfOrdernessWatermarks#onPeriodicEmit
2. WatermarkToDataOutput#emitWatermark
3. PushingAsyncDataInput.DataOutput#emitWatermark
4. Input#processWatermark
5. AbstractStreamOperator#processWatermark

#### 2.1 TimeService 处理 Watermark
1. AbstractStreamOperator#processWatermark
2. InternalTimeServiceManager#advanceWatermark
3. InternalTimeServiceManagerImpl#advanceWatermark
4. InternalTimerServiceImpl#advanceWatermark
5. Triggerable#onEventTime

#### 2.2 传递给 Output
1. AbstractStreamOperator#processWatermark
2. Output#emitWatermark
3. RecordWriterOutput#emitWatermark 将 watermark 推送到下游算子
4. RecordWriter#broadcastEmit 将 watermark 封装成 Record 推送到 targetPartition
5. ChannelSelectorRecordWriter#broadcastEmit 将 Record 序列化为 ByteBuffer 并推到所有 subPartitions
6. ResultPartitionWriter#emitRecord
7. BufferWritingResultPartition#emitRecord 将数据写入 Buffer
8. BufferWritingResultPartition#appendUnicastDataForRecordContinuation
9. BufferWritingResultPartition#addToSubpartition
10. ResultSubpartition#add
11. PipelinedSubpartition#add 数据写入 ResultSubpartition 的 Buffer

#### 2.3 Input 读出 Watermark （LocalInputChannel 为例）
1. StatusWatermarkValve#inputWatermark
2. AbstractStreamTaskNetworkInput#processElement
3. AbstractStreamTaskNetworkInput#processBuffer
4. CheckpointedInputGate#pollNext
5. InputGate#pollNext
6. SingleInputGate#pollNext
7. SingleInputGate#getNextBufferOrEvent
8. SingleInputGate#waitAndGetNextData
9. SingleInputGate#readBufferFromInputChannel
10. InputChannel#getNextBuffer
11. LocalInputChannel#getNextBuffer 从 subpartitionView 读取 Buffer
12. LocalInputChannel#requestSubpartition 从 partitionManager 获取 ResultSubpartitionView

#### 2.4 Input 输出 Watermark 到下游
1. StreamTask#processInput
2. StreamInputProcessor#processInput
3. StreamOneInputProcessor#processInput
4. PushingAsyncDataInput#emitNext
5. AbstractStreamTaskNetworkInput#emitNext
6. AbstractStreamTaskNetworkInput#processElement
7. StatusWatermarkValve#inputWatermark    
更新该 inputChannel 的 watermark 和 watermark aligned 状态，并更新所有 channel 的 watermark 组成的 priorityQueue 顺序   
8. StatusWatermarkValve#findAndOutputNewMinWatermarkAcrossAlignedChannels    
将当前所有 channel 里最小的 watermark 输出到下游   
9. DataOutput#emitWatermark

## 3. 总结
1. 数据源的 **SourceReader** 调用 **SourceOutput** 输出 record 或同时输出 rocord 和时间戳
2. 如果是流式任务 **SourceOperator** 会用 **ProgressiveTimestampsAndWatermarks** 作为 eventTimeLogic，并周期性触发 **WatermarkGenerator** 的 `onPeriodicEmit` 方法；
2. **SourceOutputWithWatermarks** 实现了 **SourceOutput** 接口，接收 record 后使用 **TimestampAssigner** 从 record 和传入时间戳中提取新的时间戳，并将提取的时间戳通过 **WatermarkGenerator** 的 `onEvent` 方法传递给 **WatermarkGenerator**；
3. **WatermarkGenerator** 决定 watermark 的累积逻辑以及何时推送到下游
4. watermark 被转换成 Record 并推送到下游

## 4. 关于空闲数据源和 Watermark 对齐
#### 4.1 空闲数据源
如果 Source 长时间没有数据输出，会导致 watermark 无法更新，下游依赖 watermark 更新的算子无法继续处理数据    
1. 为解决这一问题，**WatermarkStrategy** 提供了 `withIdleness` 选项设置空闲数据源的 idle timeout duration，当超过这个时间没有数据输出时会将 `WatermarkStatus#IDLE_STATUS` 通过 `Output#emitWatermark` 传递到下游   
2. 如果数据源的一个或多个 partitions/splits/shards 没有数据输出，但剩余的 partitions/splits/shards 还有数据输出，**CombinedWatermarkStatus** 会忽略掉 idle 的 `partialWatermark`，如果所有都 idle 则会输出 `WatermarkStatus#IDLE_STATUS` 到下游
3. 当下游的算子从一个 InputChannel 接收到 `WatermarkStatus#IDEL_STATUS` 时会将该 channel 设置成 `Unaligned` 状态，并忽略该 channel 之前的 watermark
4. 当下游算子的所有 InputChannel 都接收到 `WatermarkStatus#IDLE_STATUS` 时，算子也进入 idle 状态并向下游发送 `WatermarkStatus#IDLE_STATUS`
5. 如果 Source 结束输出数据，应该往下游推 `Long.MAX_VALUE` 作为 watermark 而不是 `WatermarkStatus#IDLE_STATUS`
6. 所有发送过 `WatermarkStatus#IDLE_STATUS` 的算子在重新发送 `WatermarkStatus#ACTIVE_STATUS` 后就能恢复到正常工作状态

#### 4.2 Watermark 对齐
如果数据源的一个或多个 partitions/splits/shards 输出数据的速度比其他的快很多/慢很多，会导致 watermark 更新速度不一致下游依赖 watermark 更新的算子无法正确处理数据，比如 TumblingEventTimeWindows 缓存很多数据而不触发聚合   
