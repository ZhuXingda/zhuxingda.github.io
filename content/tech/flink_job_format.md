---
title: "【Flink】Flink 任务格式转换"
description: ""
date: "2025-04-24T17:22:01+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Flink"
draft: true
---
分析 Flink 任务解析提交运行过程中任务的格式变化
<!--more-->
#### 1. Flink SQL 任务
###### 1.1 CalciteParser#parseSqlList: SQL -> SqlNode
由 CalciteParser 将 SQL String 解析成 calcite 的抽象语法树（AST）
###### 1.2 SqlNodeToOperationConversion#convert: SqlNode -> Operation
转换前会经过 FlinkPlannerImpl#validate 校验 AST 的有效性，因为不是所有 SQL 类型 Flink 都支持   
校验通过后从 AST 转换成 Flink Table API 的 Operation，SQL 的类型有：
- ``DQL`` Data Query Language 数据查询语言，用于从数据库中检索数据，例如 SELECT
- ``DML`` Data Manipulation Language 数据操纵语言，用于操作数据库中的数据，例如 INSERT DELETE UPDATE 等
- ``DDL`` Data Definition Language 数据定义语言，用于定义数据库的结构和模式，例如 CREATE ALTER DROP 等
- ``DCL`` Data Control Language 数据控制语言，用于创建用户角色、设置或更改数据库用户或角色权限
不同的 SQL 类型对应 Flink Table API 不同的 Operation 接口实现   
对于从 DataSource 读取数据，然后将结果写入 DataSink 的 Flink SQL 任务，其 SQL 属于 DML 类型，对应的 Operation 是 ``ModifyOperation`` 和 ``SinkModifyOperation``
###### 1.3 Planner#translate: List<ModifyOperation> modifyOperations -> List<Transformation<?>>
上一步解析的 ModifyOperation 作为 Singleton List 交给 Flink Table API 的 Planner 转换成 Flink Core 的 Transformation List
```scala
abstract class PlannerBase(
    executor: Executor,
    tableConfig: TableConfig,
    val moduleManager: ModuleManager,
    val functionCatalog: FunctionCatalog,
    val catalogManager: CatalogManager,
    isStreamingMode: Boolean,
    classLoader: ClassLoader)
  extends Planner {
    ...
    override def translate(
        modifyOperations: util.List[ModifyOperation]): util.List[Transformation[_]] = {
        beforeTranslation()
        if (modifyOperations.isEmpty) {
          return List.empty[Transformation[_]]
        }
        // 将 ModifyOperation 转换回 calcite 的 AST
        val relNodes = modifyOperations.map(translateToRel)
        // 利用 Optimizer 对 RelNode DAG 做优化，优化后的 DAG 节点从 calcite 的 RelNode 变为 Flink 的 FlinkPhysicalRel，具体过程后文详解
        val optimizedRelNodes = optimize(relNodes)
        // 将 FlinkPhysicalRel 组成的 DAG 转换成 ExecNodeGraph
        val execGraph = translateToExecNodeGraph(optimizedRelNodes, isCompiled = false)
        // 将 ExecNodeGraph 转换成 Transformation 组成的 DAG
        val transformations = translateToPlan(execGraph)
        afterTranslation()
        transformations
    }
    ...
}
```
###### 1.4 Executor#createPipeline：List<Transformation<?>> -> StreamGraph
Transformations 转换成 StreamGraph，具体逻辑在 ``org.apache.flink.streaming.api.graph.StreamGraphGenerator#generate``
## 1. 原理
#### 1.1 任务格式演变
1. SQL 任务：   

| 语法解析 | 
SQL -> SqlNode -> Operation -> RelNode -> ExecNode -> ExecNodeGraph -> Transformation -> StreamGraph -> JobGraph -> ExecutionGraph   
2. Streaming 任务：   
Transformation -> StreamGraph -> JobGraph -> ExecutionGraph   
#### 1.2 任务格式结构
1. StreamGraph   
StreamGraph 是原始的逻辑计划，由 StreamNode 组成，每个 StreamNode 对应一个 StreamOperator   
2. JobGraph   
JobGraph 是经过优化后的逻辑计划，由 JobVertex 组成   
3. ExecutionGraph   
ExecutionGraph 是执行计划，由 ExecutionJobVertex 组成   

## 2. 代码分析
#### 2.1 任务解析提交
###### SQL 任务解析
1. ParserImpl#parse
2. CalciteParser#parseSqlList： ``SQL`` -> ``SqlNode``   
将 SQL 解析成 calcite 的 AST 抽象语法树   
3. SqlNodeToOperationConversion#convert：``SqlNode`` -> ``Operation``
    3.1 FlinkPlannerImpl#validate
    校验 SqlNode 的有效性
    3.2 SqlNodeConverters#convertSqlNode
    将 SqlNode 组成的抽象语法树转换成 Flink 的 Operation，如果 SQL 是 DML(Data Manipulation Language) 则对应 ModifyOperation   
4. Planner#translate： ``ModifyOperation`` -> ``List<Transformation<?>>``   
    4.1. PlannerBase#translateToRel：ModifyOperation -> RelNode   
    将 ModifyOperation 再次转换回 calcite 的关系表达式   
    4.2. Optimizer#optimize：RelNode -> RelNode   
    利用优化器对查询表达式做优化   
    4.3. ExecNodeGraphGenerator#generate：FlinkPhysicalRel -> ExecNode -> ExecNodeGraph   
    将 RelNode 组成的查询表达式转换成 Flink 的 ExecNode 组成的 ExecNodeGraph   
    4.4. PlannerBase#translateToPlan：ExecNodeGraph -> List<Transformation<?>>    
    ```java
    override def translate(
        modifyOperations: util.List[ModifyOperation]): util.List[Transformation[_]] = {
        beforeTranslation()
        if (modifyOperations.isEmpty) {
          return List.empty[Transformation[_]]
        }

        val relNodes = modifyOperations.map(translateToRel)
        val optimizedRelNodes = optimize(relNodes)
        val execGraph = translateToExecNodeGraph(optimizedRelNodes, isCompiled = false)
        val transformations = translateToPlan(execGraph)
        afterTranslation()
        transformations
    }
    ```
5. Executor#createPipeline：``List<Transformation<?>>`` -> ``StreamGraph``   
    5.1 StreamGraphGenerator#generate   
###### Streaming 任务解析
1. DataStream#transform 编写 Flink Streaming 任务定义各个算子时，即是定义了各个 Transformation
2. StreamExecutionEnvironment#addOperator Transformation 被添加到 StreamExecutionEnvironment
3. StreamExecutionEnvironment#execute 将汇总的 Transformation 转换为 StreamGraph，然后提交到集群
4. PipelineExecutor#execute 提交任务
###### 任务提交
1. DefaultExecutor#executeAsync
2. StreamExecutionEnvironment#executeAsync
3. PipelineExecutor#execute
4. AbstractSessionClusterExecutor#execute 以提交任务到 Session Cluster 为例   
    4.1 FlinkPipelineTranslator#translateToJobGraph：StreamGraph -> JobGraph   
5. ClusterClient#submitJob
6. RestClusterClient#submitJob 将 JobGraph 序列化到文件，再将文件作为请求体通过 HTTP 发送给 JobManager

#### 2.2 任务格式转换
###### 优化 calcite 关系表达式

###### Transformation -> StreamGraph
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
###### StreamGraph -> JobGraph
1. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createJobGraph   
根据 StreamGraph 创建 JobGraph   
2. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#setChaining   
从 StreamGraph 中的 SourceNode 开始递归地创建 JobGraph 中的全部 JobVertex   
3. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createChain   
这一步将可以组成一个 Chain 的 StreamNode 合并为 OperatorChainInfo 并创建 JobVertex，这一步调用 org.StreamingJobGraphGenerator#isChainable 判断上下游算子是否可以组成 Chain    
```java
    public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
        StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);
        // 上游只有一个输入
        return downStreamVertex.getInEdges().size() == 1 && isChainableInput(edge, streamGraph);
    }

    private static boolean isChainableInput(StreamEdge edge, StreamGraph streamGraph) {
        StreamNode upStreamVertex = streamGraph.getSourceVertex(edge);
        StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);
        // 以下条件任一不满足则不能合并到一个 Chain
        if (!(// 全局配置支持
                streamGraph.isChainingEnabled()
                // 上游与本节点在同一个 SlotSharingGroup
                && upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
                // 节点自身支持 Chain
                && areOperatorsChainable(upStreamVertex, downStreamVertex, streamGraph)
                && arePartitionerAndExchangeModeChainable(
                        edge.getPartitioner(), edge.getExchangeMode(), streamGraph.isDynamic()))) {

            return false;
        }

        // check that we do not have a union operation, because unions currently only work
        // through the network/byte-channel stack.
        // we check that by testing that each "type" (which means input position) is used only once
        for (StreamEdge inEdge : downStreamVertex.getInEdges()) {
            if (inEdge != edge && inEdge.getTypeNumber() == edge.getTypeNumber()) {
                return false;
            }
        }
        return true;
    }

    static boolean areOperatorsChainable(
            StreamNode upStreamVertex, StreamNode downStreamVertex, StreamGraph streamGraph) {
        StreamOperatorFactory<?> upStreamOperator = upStreamVertex.getOperatorFactory();
        StreamOperatorFactory<?> downStreamOperator = downStreamVertex.getOperatorFactory();
        // 上下游算子都不为空
        if (downStreamOperator == null || upStreamOperator == null) {
            return false;
        }

        // Yielding Operator 不能和 Legacy Source 合并到同一个 Chain
        if (downStreamOperator instanceof YieldingOperatorFactory
                && getHeadOperator(upStreamVertex, streamGraph).isLegacySource()) {
            return false;
        }
        
        boolean isChainable;

        switch (upStreamOperator.getChainingStrategy()) {
            // 如果上游算子不支持 Chain，则不能合并到同一个 Chain
            case NEVER:
                isChainable = false;
                break;
            case ALWAYS:
            case HEAD:
            case HEAD_WITH_SOURCES:
                isChainable = true;
                break;
            default:
                throw new RuntimeException(
                        "Unknown chaining strategy: " + upStreamOperator.getChainingStrategy());
        }

        switch (downStreamOperator.getChainingStrategy()) {
            // 如果下游算子不支持 Chain，或者下游算子策略为 Head 则不能合并到同一个 Chain
            case NEVER:
            case HEAD:
                isChainable = false;
                break;
            // 如果下游算子策略为 ALWAYS，则取决于上游算子的 ChainingStrategy
            case ALWAYS:
                break;
            // 如果下游算子策略为 HEAD_WITH_SOURCES，则取决于上游算子是否为 Source
            case HEAD_WITH_SOURCES:
                isChainable &= (upStreamOperator instanceof SourceOperatorFactory);
                break;
            default:
                throw new RuntimeException(
                        "Unknown chaining strategy: " + downStreamOperator.getChainingStrategy());
        }

        // 上下游算子并行度必须一致
        isChainable &= upStreamVertex.getParallelism() == downStreamVertex.getParallelism();
        // 如果全局配置不支持不同最大并行度的算子合并到同一个 Chain，则上下游算子最大并行度必须一致
        if (!streamGraph.isChainingOfOperatorsWithDifferentMaxParallelismEnabled()) {
            isChainable &=
                    upStreamVertex.getMaxParallelism() == downStreamVertex.getMaxParallelism();
        }

        return isChainable;
    }
```
4. org.apache.flink.streaming.api.graph.StreamingJobGraphGenerator#createJobVertex   
根据 OperatorChainInfo 创建 JobVertex，并添加到 JobGraph
###### JobGraph -> ExecutionGraph
```java
// 负责构建 DefaultExecutionGraph 的 EdgeManager
public class EdgeManagerBuildUtil {

    // 构建的入口，将下游消费数据的 ExecutionJobVertex 和上游产出数据的 IntermediateResult 连接到一起
    // 即是将 ExecutionVertexs 和 IntermediateResultPartitions 连接到一起
    static void connectVertexToResult(
            ExecutionJobVertex vertex, IntermediateResult intermediateResult) {
        final DistributionPattern distributionPattern =
                intermediateResult.getConsumingDistributionPattern();
        // JobVertexInputInfo 决定了 ExecutionVertexs 的输入拓扑
        final JobVertexInputInfo jobVertexInputInfo =
                vertex.getGraph()
                        .getJobVertexInputInfo(vertex.getJobVertexId(), intermediateResult.getId());

        switch (distributionPattern) {
            // 上游一个 sub task 和一个或多个下游 sub task 相连
            case POINTWISE:
                connectPointwise(vertex, intermediateResult, jobVertexInputInfo);
                break;
            // 上游一个 sub task 和所有下游 sub task 相连
            case ALL_TO_ALL:
                connectAllToAll(vertex, intermediateResult);
                break;
            default:
                throw new IllegalArgumentException("Unrecognized distribution pattern.");
        }
    }
    ...

    private static void connectAllToAll(ExecutionJobVertex jobVertex, IntermediateResult result) {
        connectInternal(
                Arrays.asList(jobVertex.getTaskVertices()), // 下游所有的 ExecutionVertexs
                Arrays.asList(result.getPartitions()), // 上游所有的 IntermediateResultPartitions
                result.getResultType(),
                jobVertex.getGraph().getEdgeManager());
    }

    private static void connectPointwise(
            ExecutionJobVertex jobVertex,
            IntermediateResult result,
            JobVertexInputInfo jobVertexInputInfo) {

        Map<IndexRange, List<Integer>> consumersByPartition = new LinkedHashMap<>();

        for (ExecutionVertexInputInfo executionVertexInputInfo :
                jobVertexInputInfo.getExecutionVertexInputInfos()) {
            int consumerIndex = executionVertexInputInfo.getSubtaskIndex();
            List<IndexRange> ranges =
                    mergeIndexRanges(
                            executionVertexInputInfo.getConsumedSubpartitionGroups().keySet());
            checkState(ranges.size() == 1);
            consumersByPartition.compute(
                    ranges.get(0),
                    (ignore, consumers) -> {
                        if (consumers == null) {
                            consumers = new ArrayList<>();
                        }
                        consumers.add(consumerIndex);
                        return consumers;
                    });
        }

        consumersByPartition.forEach(
                (range, subtasks) -> {
                    List<ExecutionVertex> taskVertices = new ArrayList<>();
                    List<IntermediateResultPartition> partitions = new ArrayList<>();
                    for (int index : subtasks) {
                        taskVertices.add(jobVertex.getTaskVertices()[index]);
                    }
                    for (int i = range.getStartIndex(); i <= range.getEndIndex(); ++i) {
                        partitions.add(result.getPartitions()[i]);
                    }
                    connectInternal(
                            taskVertices,
                            partitions,
                            result.getResultType(),
                            jobVertex.getGraph().getEdgeManager());
                });
    }

    /** Connect all execution vertices to all partitions. */
    private static void connectInternal(
            List<ExecutionVertex> taskVertices,
            List<IntermediateResultPartition> partitions,
            ResultPartitionType resultPartitionType,
            EdgeManager edgeManager) {
        checkState(!taskVertices.isEmpty());
        checkState(!partitions.isEmpty());

        ConsumedPartitionGroup consumedPartitionGroup =
                createAndRegisterConsumedPartitionGroupToEdgeManager(
                        taskVertices.size(), partitions, resultPartitionType, edgeManager);
        for (ExecutionVertex ev : taskVertices) {
            ev.addConsumedPartitionGroup(consumedPartitionGroup);
        }

        List<ExecutionVertexID> consumerVertices =
                taskVertices.stream().map(ExecutionVertex::getID).collect(Collectors.toList());
        ConsumerVertexGroup consumerVertexGroup =
                ConsumerVertexGroup.fromMultipleVertices(consumerVertices, resultPartitionType);
        for (IntermediateResultPartition partition : partitions) {
            partition.addConsumers(consumerVertexGroup);
        }

        consumedPartitionGroup.setConsumerVertexGroup(consumerVertexGroup);
        consumerVertexGroup.setConsumedPartitionGroup(consumedPartitionGroup);
    }

    private static ConsumedPartitionGroup createAndRegisterConsumedPartitionGroupToEdgeManager(
            int numConsumers,
            List<IntermediateResultPartition> partitions,
            ResultPartitionType resultPartitionType,
            EdgeManager edgeManager) {
        List<IntermediateResultPartitionID> partitionIds =
                partitions.stream()
                        .map(IntermediateResultPartition::getPartitionId)
                        .collect(Collectors.toList());
        ConsumedPartitionGroup consumedPartitionGroup =
                ConsumedPartitionGroup.fromMultiplePartitions(
                        numConsumers, partitionIds, resultPartitionType);
        finishAllDataProducedPartitions(partitions, consumedPartitionGroup);
        edgeManager.registerConsumedPartitionGroup(consumedPartitionGroup);
        return consumedPartitionGroup;
    }

    private static void finishAllDataProducedPartitions(
            List<IntermediateResultPartition> partitions,
            ConsumedPartitionGroup consumedPartitionGroup) {
        for (IntermediateResultPartition partition : partitions) {
            // this is for dynamic graph as consumedPartitionGroup has not been created when the
            // partition becomes finished.
            if (partition.hasDataAllProduced()) {
                consumedPartitionGroup.partitionFinished();
            }
        }
    }
}
```
1. org.apache.flink.runtime.executiongraph.DefaultExecutionGraphBuilder#buildGraph   
根据 JobGraph 创建 ExecutionGraph   