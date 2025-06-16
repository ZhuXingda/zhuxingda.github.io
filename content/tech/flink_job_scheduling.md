---
title: "【Flink】Flink 任务调度部署"
date: "2025-05-29T15:59:01+08:00"
categories:
  - "分布式系统"
tags:
  - "Flink"
draft: true
---
Flink 任务在部署到集群运行时涉及对集群资源的管理以及任务的调度，本文分析其逻辑和实现。
<!--more-->
## 1. 原理
Flink 以 Task Slot 作为基本的资源调度单位，由 JobManager 上的 ``ResourceManager`` 统一管理，包括创建、销毁和调度   
每个 TaskManager 是一个独立的 JVM 进程，包含一个或多个 Task Slot，多个 Task Slot 之间会均分 TaskManager 的 ``managed memory`` （Task Heap Memory 应该是共享？）   
同一个 Task Slot 支持运行不同 Task 的多个 SubTask，被称为 ``Slot Sharing``（区别于 Operator Chain），这样有两个好处：
- 运行 Flink 任务所需的 Task Slot 数量就等于任务中算子的最大并行数
- 可以将资源消耗得多的 SubTask 和消耗得少的 SubTask 一起运行，以获得更好的资源利用率

Flink 定义了 ``SlotSharingGroup`` 和 ``ColocationGroup`` 两种约束条件用于定义任务调度
- SlotSharingGroup：定义了不同 Task 的 SubTasks 可以被调度到同一个 Task Slot，是一个软性约束不一定保证成功
- ColocationGroup：定义了不同 Task 的第 i 个 SubTask 必须被调度到同一个 Task Slot，是一个强制性约束

## 2. 实现
#### 2.1 SlotSharingGroup 和 CoLocationGroup 的注册和传递
Flink 提供了   
1. StreamExecutionEnvironment#registerSlotSharingGroup 接口用于注册 SlotSharingGroup
2. SingleOutputStreamOperator#slotSharingGroup / org.apache.flink.streaming.api.datastream.DataStreamSink#slotSharingGroup 接口用于设置 Operator 所属的 SlotSharingGroup   

StreamGraph 中的 StreamNode 在 StreamGraph#addOperator 方法中被创建和添加到 StreamGraph，添加时会设置 SlotSharingGroup 和 CoLocationGroup：
- SlotSharingGroup 来自 StreamGraphGenerator#translate 执行时传入 TransformationTranslator.Context 中的 SlotSharingGroup，具体用哪个 SlotSharingGroup 由 `StreamGraphGenerator#determineSlotSharingGroup` 决定
- CoLocationGroup 由 `Transformation 的 coLocationGroupKey` 决定，该参数仅支持在框架内设置，对外没有暴露

StreamingJobGraphGenerator#createJobGraph 根据 StreamGraph 创建 JobGraph，创建过程中会设置 SlotSharingGroup 和 CoLocationGroup ``StreamingJobGraphGenerator#setSlotSharingAndCoLocation``   
#### 2.2 Task Slot 管理
#### 2.3 Task Slot 调度
#### 2.4 Task Slot 分配
1. JobMaster#onStart 启动
2. JobMaster#startScheduling 开始调度
3. SchedulerBase#startScheduling 调度器执行调度逻辑
4. SchedulingStrategy#startScheduling SchedulingStrategy 有 PipelinedRegionSchedulingStrategy 和 VertexwiseSchedulingStrategy 两个实现，对应 streaming 任务和 batch 任务的调度策略，这里以 PipelinedRegionSchedulingStrategy 为例
5. PipelinedRegionSchedulingStrategy#startScheduling 开始调度 region，具体的调度逻辑在 Task Slot 调度
6. PipelinedRegionSchedulingStrategy#scheduleRegion 将 SchedulingPipelinedRegion 下的 ExecutionVertexID List 分配到 Slot
7. DefaultScheduler#allocateSlotsAndDeploy 转换 ExecutionVertexID -> JobVertexID -> ExecutionVertex -> Execution ，然后将一批 Execution List 分配到 Slot   
8. DefaultExecutionDeployer#allocateSlotsAndDeploy   
    - 绑定 Slot：   
    1. DefaultExecutionDeployer#allocateSlotsFor Execution -> ExecutionAttemptID ， 交由 ExecutionSlotAllocator 执行 Slot 分配   
    2. ExecutionSlotAllocator#allocateSlotsFor 该接口有 SimpleExecutionSlotAllocator 和 SlotSharingExecutionSlotAllocator 两个实现，后者支持将一个 ExecutionSlotSharingGroup 里的 Execution 分配到同一个 Slot 运行   
    3. SlotSharingExecutionSlotAllocator#allocateSlotsFor 使用 SlotSharingStrategy 将 Execution 的 ExecutionVertexID 分配到 ExecutionSlotSharingGroup，然后将 ExecutionSlotSharingGroup 绑定到 LogicalSlot   
    - 部署到 Slot：   
    1. DefaultExecutionDeployer#createDeploymentHandles 创建 ExecutionDeploymentHandle   
    2. DefaultExecutionDeployer#waitForAllSlotsAndDeploy -> DefaultExecutionDeployer#deployAll -> DefaultExecutionDeployer#deployOrHandleError -> DefaultExecutionOperations#deploy -> Execution#deploy