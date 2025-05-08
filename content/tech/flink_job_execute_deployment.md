---
title: "【Flink】Flink 任务执行和部署原理"
description: ""
date: "2025-04-24T17:22:01+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Flink"
---
分析 Flink 任务完整的提交过程，主要关注在这个过程中任务的格式变化和调度执行。
<!--more-->
## 代码分析
#### 1. SQL 任务解析提交
###### 1.1 SQL 解析
1. ParserImpl#parse
2. alciteParser#parseSqlList： SQL -> SqlNode   
3. SqlNodeConverters#convertSqlNode：SqlNode -> Operation
4. Planner#translate： Operation -> Transformation   
    4.1. PlannerBase#translateToRel：ModifyOperation -> RelNode   
    4.2. Optimizer#optimize：RelNode -> RelNode   
    4.3. ExecNodeGraphGenerator#generate：FlinkPhysicalRel -> ExecNode -> ExecNodeGraph   
    4.4. PlannerBase#translateToPlan：ExecNodeGraph -> List<Transformation<?>>   
5. Executor#createPipeline：List<Transformation<?>> -> Pipeline   
    5.1 StreamGraphGenerator#generate：List<Transformation<?>> -> StreamGraph   
###### 1.2 任务提交
1. DefaultExecutor#executeAsync
2. StreamExecutionEnvironment#executeAsync
3. PipelineExecutor#execute
4. AbstractSessionClusterExecutor#execute 以提交任务到 Session Cluster 为例   
    4.1 FlinkPipelineTranslator#translateToJobGraph：Pipeline -> JobGraph   
5. ClusterClient#submitJob
6. RestClusterClient#submitJob 将 JobGraph 序列化到文件，再将文件作为请求体通过 HTTP 发送给 JobManager

#### 2. Streaming 任务解析提交
1. DataStream#transform 编写 Flink Streaming 任务定义各个算子时，是在定义一个个 Transformation
2. StreamExecutionEnvironment#addOperator Transformation 被添加到 StreamExecutionEnvironment
3. StreamExecutionEnvironment#execute 将汇总的 Transformation 转换为 StreamGraph，然后提交到集群
4. PipelineExecutor#execute 后面就和 SQL 提交的流程一致了

#### 3. 任务调度和执行
###### 3.1 JobMaster 初始化
1. Dispatcher#createJobMasterRunner JobManager 接收请求，创建 JobManagerRunner 并启动
2. JobMasterServiceLeadershipRunner#start
3. JobMasterServiceLeadershipRunner#createNewJobMasterServiceProcess 选主之后创建 JobMasterService
4. DefaultJobMasterServiceFactory#internalCreateJobMasterService 创建 JobMaster，JobGraph 作为参数
5. SchedulerNGFactory#createInstance 创建 SchedulerNG -> SchedulerBase -> DefaultScheduler   
6. SchedulerBase#createAndRestoreExecutionGraph：JobGraph -> ExecutionGraph
7. SchedulingStrategyFactory#createInstance 创建 SchedulingStrategy，参数有 ExecutionGraph 的 SchedulingTopology 和 DefaultScheduler 本身
###### 3.2 JobMaster 调度和执行
1. JobMaster#onStart 启动
2. JobMaster#startScheduling 开始调度
3. SchedulerBase#startScheduling 调度器执行调度逻辑
4. SchedulingStrategy#startScheduling SchedulingStrategy 有 PipelinedRegionSchedulingStrategy 和 VertexwiseSchedulingStrategy 两个实现，对应 streaming 任务和 batch 任务的调度策略，这里以 PipelinedRegionSchedulingStrategy 为例
5. PipelinedRegionSchedulingStrategy#startScheduling 开始调度 region，具体的调度逻辑在后文详细分析
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

#### 4. 任务格式转换
###### 4.1 Transformation -> StreamGraph
1. org.apache.flink.streaming.api.graph.StreamGraphGenerator#generate   
生成 StreamGraph，遍历 Transformation List，将每个 Transformation 转换为 StreamNode 并添加到 StreamGraph
2. org.apache.flink.streaming.api.graph.StreamGraphGenerator#transform   
如果有调用 SingleOutputStreamOperator#slotSharingGroup / org.apache.flink.streaming.api.datastream.DataStreamSink#slotSharingGroup 接口为 Transformation 设置 SlotSharingGroup，则将该 SlotSharingGroup 的 name 和 ResourceProfile 添加到 slotSharingGroupResources 集合中
3. org.apache.flink.streaming.api.graph.StreamGraphGenerator#translate   
由 TransformationTranslator 将 Transformation 转换为 StreamNode，并添加到 StreamGraph
4. org.apache.flink.streaming.api.graph.StreamGraph#addOperator
添加 StreamNode 到 StreamGraph，添加时的参数有：
```java
private <IN, OUT> void addOperator(
            Integer vertexID,
            @Nullable String slotSharingGroup,
            @Nullable String coLocationGroup,
            StreamOperatorFactory<OUT> operatorFactory,
            TypeInformation<IN> inTypeInfo,
            TypeInformation<OUT> outTypeInfo,
            String operatorName,
            Class<? extends TaskInvokable> invokableClass) {
                ...
            }
```
###### 4.2 StreamGraph -> JobGraph
1. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createJobGraph   
根据 StreamGraph 创建 JobGraph   
2. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#setChaining   
从 StreamGraph 中的 SourceNode 开始递归地创建 JobGraph 中的全部 JobVertex   
3. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createChain   
这一步将可以组成一个 Chain 的 StreamNode 合并为 OperatorChainInfo 并创建 JobVertex   
4. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createJobVertex   
根据 OperatorChainInfo 创建 JobVertex，并添加到 JobGraph
###### 4.3 JobGraph -> ExecutionGraph
1. org.apache.flink.runtime.executiongraph.DefaultExecutionGraphBuilder#buildGraph   
根据 JobGraph 创建 ExecutionGraph   
![](https://nightlies.apache.org/flink/flink-docs-master/fig/job_and_execution_graph.svg)
## 总结
#### 任务格式
###### 任务格式演变
1. SQL 任务：   
SQL -> SqlNode -> Operation -> RelNode -> ExecNode -> ExecNodeGraph -> Transformation -> StreamGraph -> JobGraph -> ExecutionGraph
2. Streaming 任务：   
Transformation -> StreamGraph -> JobGraph -> ExecutionGraph
###### 任务格式结构
1. StreamGraph
StreamGraph 由 StreamNode 组成，每个 StreamNode 对应一个 StreamOperator 
2. JobGraph
JobGraph 由 JobVertex 组成
3. ExecutionGraph
ExecutionGraph 由 ExecutionJobVertex 组成

#### 任务调度执行
###### 注册 SlotSharingGroup
Flink 提供了   
1. StreamExecutionEnvironment#registerSlotSharingGroup 接口用于注册 SlotSharingGroup
2. SingleOutputStreamOperator#slotSharingGroup / org.apache.flink.streaming.api.datastream.DataStreamSink#slotSharingGroup 接口用于设置 Operator 所属的 SlotSharingGroup   
官方提供的示例：https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/finegrained_resource/#usage
###### 构建 StreamGraph 时设置 SlotSharingGroup 和 CoLocationGroup
StreamGraph 中的 StreamNode 在 StreamGraph#addOperator 方法中被创建和添加到 StreamGraph，添加时会设置 SlotSharingGroup 和 CoLocationGroup：
- SlotSharingGroup 来自 StreamGraphGenerator#translate 执行时传入 TransformationTranslator.Context 中的 SlotSharingGroup，具体用哪个 SlotSharingGroup 由 `StreamGraphGenerator#determineSlotSharingGroup` 决定
- CoLocationGroup 由 `Transformation 的 coLocationGroupKey` 决定，目前（Flink V2.0）该功能还没有支持
###### 构建 JobGraph 时设置 SlotSharingGroup
1. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createJobGraph   
根据 StreamGraph 创建 JobGraph，创建过程中会设置 SlotSharingGroup 和 CoLocationGroup
2. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#setSlotSharing   
