---
title: "【Flink】Flink Network IO 相关代码分析"
description: ""
date: "2025-03-12T14:19:45+08:00"
thumbnail: ""
categories:
  - ""
tags:
  - "Flink"
draft: true
---
数据在组成 Flink 任务的各 Task 之间传输，由 Flink Network IO 实现，包含常见的 Flink 如何实现背压的问题。
# Shuffle
Shuffle 决定了数据在 Task 之间如何划分 partition，以及各 partition 分别被哪个 subTask 消费   
ShuffleEnvironment
- createResultPartitionWriters
- createInputGates
ResultPartitionDeploymentDescriptor
InputGateDeploymentDescriptor

ResultPartitionWriter
InputGates————AbstractReader

org.apache.flink.streaming.runtime.tasks.OperatorChain#createChainedSources
org.apache.flink.streaming.runtime.tasks.OneInputStreamTask#init

在 org.apache.flink.streaming.runtime.tasks.StreamTask
InputGates 转换成 StreamInputProcessor
# Mailbox

# Network
## Buffer
## Partition
BufferWritingResultPartition
## Writer


org.apache.flink.runtime.io.network.api.writer.RecordWriterDelegate#getRecordWriter


org.apache.flink.streaming.runtime.tasks.OperatorChain#OperatorChain/createChainOutputs/createOutputCollector/mainOperatorOutput
RecordWriterOutput

org.apache.flink.streaming.runtime.io.RecordWriterOutput#collect/pushToRecordWriter
org.apache.flink.runtime.io.network.api.writer.RecordWriter#emit(T)
org.apache.flink.runtime.io.network.api.writer.ResultPartitionWriter#emitRecord
org.apache.flink.runtime.io.network.partition.BufferWritingResultPartition#emitRecord/appendUnicastDataForNewRecord/requestNewUnicastBufferBuilder/requestNewBufferBuilderFromPool
org.apache.flink.runtime.io.network.buffer.BufferProvider#requestBufferBuilderBlocking(int)
org.apache.flink.runtime.io.network.buffer.LocalBufferPool#requestBufferBuilderBlocking()/requestMemorySegmentBlocking(int)/requestMemorySegment(int)
```java
    private MemorySegment requestMemorySegmentBlocking(int targetChannel)
            throws InterruptedException {
        MemorySegment segment;
        while ((segment = requestMemorySegment(targetChannel)) == null) {
            try {
                // wait until available
                getAvailableFuture().get();
            } catch (ExecutionException e) {
                LOG.error("The available future is completed exceptionally.", e);
                ExceptionUtils.rethrow(e);
            }
        }
        return segment;
    }
```
## Reader

# 参考
https://juejin.cn/post/6844903997837410318?from=search-suggest
https://juejin.cn/post/6844903955831455752