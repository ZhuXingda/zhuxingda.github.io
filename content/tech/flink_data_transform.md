---
title: "【Flink】Flink 数据传输原理"
date: "2025-06-06T11:29:11+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Flink"
draft: true
---
StreamRecord 在 SubTask 之间传递的过程
<!--more-->
## 数据传输的拓扑形成
#### JobGraph
代表逻辑执行计划的 JobGraph 由 ``JobVertex`` 组成，JobVertex 中的 ``JobEdge`` 代表数据输入的 inputs，``IntermediateDataSet`` 代表数据输出的 results   
一对 IntermediateDataSet 和 JobEdge 代表了 JobGraph 中 JobVertexs 之间的数据流向
```java
public class JobVertex implements java.io.Serializable {
    // 将上游的 JobVertex 与本 JobVertex 连接到一起
    public JobEdge connectNewDataSetAsInput(
            JobVertex input,
            DistributionPattern distPattern,
            ResultPartitionType partitionType,
            IntermediateDataSetID intermediateDataSetId,
            boolean isBroadcast,
            boolean isForward,
            int typeNumber,
            boolean interInputsKeysCorrelated,
            boolean intraInputKeyCorrelated) {
        // 上游的 JobVertex 添加该连接的 IntermediateDataSet
        IntermediateDataSet dataSet =
                input.getOrCreateResultDataSet(intermediateDataSetId, partitionType);
        // 创建本 JobVertex 的 JobEdge并添加到 inputs
        JobEdge edge =
                new JobEdge(
                        dataSet,
                        this,
                        distPattern,
                        isBroadcast,
                        isForward,
                        typeNumber,
                        interInputsKeysCorrelated,
                        intraInputKeyCorrelated);
        this.inputs.add(edge);
        // IntermediateDataSet 和 JobEdge 建立连接
        dataSet.addConsumer(edge);
        return edge;
    }
}
```
#### ExecutionGraph
代表执行计划的 ExecutionGraph 由 ``ExecutionJobVertex`` 组成，ExecutionJobVertex 中每一个并行度对应一个 ``ExecutionVertex``，代表并行执行的 JobVertex   
ExecutionJobVertex 中的输入和输出都由 ``IntermediateResult`` 表示：
- 输出的 IntermediateResult[] producedDataSets 对应 JobVertex 的 results
- 输入的 List<IntermediateResult> inputs 对应 JobVertex 的 inputs

每个 IntermediateResult 内部有：
- 一个 ExecutionJobVertex 作为 producer
- 一个 ``IntermediateResultPartition`` 数组，数量等于 producer 的 ExecutionVertex 数量    

每个 ExecutionVertex 内部有 Map<IntermediateResultPartitionID, IntermediateResultPartition> resultPartitions 表示该 ExecutionVertex 的输出，resultPartitions 中每个 ``IntermediateResultPartition`` 对应 ExecutionJobVertex 输出的每个 IntermediateResult   

由 DefaultExecutionGraph 的 ``EdgeManager`` 管理 IntermediateResultPartition 和 ExecutionJobVertex 之间的消费关系，即 ``ConsumedPartitionGroup``
#### Task
ExecutionGraph 部署执行时 ExecutionVertex 经过到 ``TaskDeploymentDescriptor`` 再到 ``Task`` 的转换   
- ExecutionVertex 所属 ExecutionJobVertex 的 inputs 对应的 ``IntermediateResult`` -> ``InputGateDeploymentDescriptor`` -> ``InputGate``
- ExecutionVertex resultPartitions 对应的 ``IntermediateResultPartition`` -> ``ResultPartitionDeploymentDescriptor`` -> ``ResultPartition``

InputGate 和 ResultPartition 分别表示 Task 的输入和输出
IntermediateResultPartition#getNumberOfSubpartitions 方法决定了 ResultPartition 中 ``ResultSubpartition`` 的数量，即下游有多少 Tasks 在消费 
## 数据传输栈
## 背压控制
## Shuffle 实现
## 参考
https://blog.jrwang.me/2019/flink-source-code-data-exchange/