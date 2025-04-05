---
title: "【Flink】Flink 跨 Task 数据传输代码分析"
description: ""
date: "2025-03-12T14:19:45+08:00"
thumbnail: ""
categories:
  - ""
tags:
  - "Flink"
---
主要分析数据如何在组成 Flink 任务的各 Task 之间输入输出，包含常见的 Flink 如何实现背压的问题。
<!--more-->
每一个算子都由 org.apache.flink.runtime.taskmanager.Task 类实现，构建 Task 时会使用 ShuffleEnvironment 分别创建 Task 的 inputGates 和 resultPartitions。二者分别对应算子的输入和输出。
# Task 执行
**节点启动 TaskExecutor**
1. org.apache.flink.kubernetes.taskmanager.KubernetesTaskExecutorRunner#main / org.apache.flink.yarn.YarnTaskExecutorRunner#main
2. org.apache.flink.runtime.taskexecutor.TaskManagerRunner#runTaskManager
3. org.apache.flink.runtime.taskexecutor.TaskManagerRunner#createTaskExecutorService
4. org.apache.flink.runtime.taskexecutor.TaskManagerRunner#startTaskManager
5. org.apache.flink.runtime.taskexecutor.TaskExecutor#TaskExecutor

**启动 Task**
1. org.apache.flink.runtime.taskexecutor.TaskExecutor#submitTask
2. org.apache.flink.runtime.taskmanager.Task#Task

**创建 NettyShuffleEnvironment**
1. org.apache.flink.runtime.taskexecutor.TaskManagerRunner#startTaskManager
2. org.apache.flink.runtime.taskexecutor.TaskManagerServices#fromConfiguration
3. org.apache.flink.runtime.taskexecutor.TaskManagerServices#createShuffleEnvironment
4. org.apache.flink.runtime.io.network.NettyShuffleServiceFactory#createNettyShuffleEnvironment   
创建 NettyShuffleEnvironment 的时候会一并创建 ResultPartitionManager，并作为参数传入 SingleInputGateFactory 和 ResultPartitionFactory
5. org.apache.flink.runtime.io.network.NettyShuffleEnvironment#NettyShuffleEnvironment

# 结构


# ResultPartitionWriter
**创建 ResultPartitionWriter 的调用栈**
1. org.apache.flink.runtime.taskmanager.Task#Task 
2. org.apache.flink.runtime.io.network.NettyShuffleEnvironment#createResultPartitionWriters
3. org.apache.flink.runtime.io.network.partition.ResultPartitionFactory#create   
在这一步会根据 ResultPartitionType 创建 PipelinedResultPartition / SortMergeResultPartition / TieredResultPartition / HsResultPartition   

**算子输出的调用栈**
1. org.apache.flink.streaming.runtime.io.RecordWriterOutput#collect
2. org.apache.flink.runtime.io.network.api.writer.ChannelSelectorRecordWriter#emit
3. org.apache.flink.runtime.io.network.api.writer.RecordWriter#emit
4. org.apache.flink.runtime.io.network.api.writer.ResultPartitionWriter#emitRecorod
5. org.apache.flink.runtime.io.network.partition.BufferWritingResultPartition#emitRecord

**ResultPartitionManager 关联 ResultPartition**
1. org.apache.flink.runtime.taskmanager.Task#setupPartitionsAndGates
2. org.apache.flink.runtime.io.network.partition.ResultPartition#setup
3. org.apache.flink.runtime.io.network.partition.ResultPartitionManager#registerResultPartition

**

# InputGate

**算子输入的调用栈**


# 网络栈
**创建 NettyServer 和 NettyClient**
1. org.apache.flink.runtime.io.network.NettyShuffleServiceFactory#createNettyShuffleEnvironment 
2. org.apache.flink.runtime.io.network.netty.NettyConnectionManager#NettyConnectionManager   
ConnectionManager 负责将 

3. org.apache.flink.runtime.io.network.netty.NettyServer#NettyServer / org.apache.flink.runtime.io.network.netty.NettyClient#NettyClient

**初始化 NettyServer**
1. org.apache.flink.runtime.taskexecutor.TaskManagerServices#fromConfiguration
2. org.apache.flink.runtime.io.network.NettyShuffleEnvironment#start
3. org.apache.flink.runtime.io.network.netty.NettyConnectionManager#start
4. org.apache.flink.runtime.io.network.netty.NettyServer#init   
NettyServer 在初始化时会创建 ServerChannelInitializer 并注册为 ServerBootstrap 的 ChannelHandler   
```java
    @VisibleForTesting
    static class ServerChannelInitializer extends ChannelInitializer<SocketChannel> {
        private final NettyProtocol protocol;
        private final SSLHandlerFactory sslHandlerFactory;

        public ServerChannelInitializer(
                NettyProtocol protocol, SSLHandlerFactory sslHandlerFactory) {
            this.protocol = protocol;
            this.sslHandlerFactory = sslHandlerFactory;
        }

        @Override
        public void initChannel(SocketChannel channel) throws Exception {
            if (sslHandlerFactory != null) {
                channel.pipeline()
                        .addLast("ssl", sslHandlerFactory.createNettySSLHandler(channel.alloc()));
            }
            // 对传入的 SocketChannel 添加 NettyProtocal 提供的 ChannelHandlers
            channel.pipeline().addLast(protocol.getServerChannelHandlers());
        }
    }
```



# 参考
https://blog.jrwang.me/2019/flink-source-code-data-exchange/
https://juejin.cn/post/6844903997837410318?from=search-suggest
https://juejin.cn/post/6844903955831455752