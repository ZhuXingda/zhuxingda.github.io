---
title: Flink Stream Operator 状态控制
date: 2024-11-28
tags:
    - Flink 
categories:
    - "分布式系统"
draft: true
---
前文分析了 Flink Checkpoint 的产生和传递过程，对于单个 StreamOperator 的 State 涉及 `缓存` 和 `持久化`，分别对应 `State Backend` 和 `Checkpoint Storage`   
<!--more-->
[![Os4Qvi.png](https://ooo.0x0.ooo/2025/06/11/Os4Qvi.png)](https://img.tg/image/Os4Qvi)

如上图所示 TaskManager 内可能有一至多个 TaskSlot，每个 TaskSlot 内可能有一至多个 Task（Slot Sharing），每个 StreamTask 独立维护 `State Backend` 和 `Checkpoint Storage` 的实例，然后被 OperatorChain 内的 StreamOperator 共用，StreamTask 对应 ExecutionGraph 里的 ExecutionVertex，即算子的一个并发度
## 1. State Backend
StreamOperator 的 State 分为 `OperatorState` 和 `KeyedState` 两部分，其中 `KeyedState` 只有算子处于 KeyedStream 才存在   
算子正常运行时 State 被缓存在 State Backend，目前有三种实现：
- `HashMapStateBackend`   
    基于 Heap Memory，提供 HeapKeyedStateBackend 和 DefaultOperatorStateBackend
- `EmbeddedRocksDBStateBackend`    
    基于 Managed Memory，提供 RocksDBKeyedStateBackend 和 DefaultOperatorStateBackend
- `ForStStateBackend`（Experimental）
###### 1.1 OperatorStateBackend
开发者定义的 UDF 可以通过实现 CheckpointedFunction 接口获取 OperatorState   
OperatorStateBackend 接口只有 `DefaultOperatorStateBackend` 一个实现类，实现了 Snapshotable 接口用于在 checkpoint 时将 State 持久化到 Checkpoint Storage ，DefaultOperatorStateBackend 提供：
    - `BroadcastState`：用于保存 BreadcastStream 的 State，因为是针对 BroadcastStream 所以每个 Operator 实例接收到的数据都是一样的，所以每个 state element 都可以在 BroadcastState 访问到，其实现类是 `HeapBroadcastState` 
    - `ListState`：OperatorState 的 ListState ，其实现类是 `PartitionableListState`，ListState 支持在 Operator 并行度发生变化时在 Operators 之间重新分配，具体分配逻辑见后文
###### 1.2 KeyedStateBackend
开发者定义的 RichFuction 可以通过 RuntimeContext 获取 KeyedState
```text
    KeyedStateBackend <---|                                                                       
                          |                                                                       |--- HeapKeyedStateBackend
                          |--- CheckpointableKeyedStateBackend <--- AbstractKeyedStateBackend <---| 
                          |                                                                       |--- RocksDBKeyedStateBackend
    Snapshotable <--------|                                                                                                                
```
KeyedStateBackend 的继承关系如上所示，实现类同样继承 Snapshotable 来持久化 State，AbstractStreamOperator 内部在创建 KeyedState 时调用流程大致如下：      
> UDF -> StreamingRuntimeContext -> DefaultKeyedStateStore -> AbstractKeyedStateBackend -> HeapKeyedStateBackend / RocksDBKeyedStateBackend

AbstractKeyedStateBackend 创建时需要 `InternalKeyContext` 作为参数用于表示该 StateBackend 所在 StreamTask 的 keyGroups 划分情况，其中包含：
- `keyGroupRange` StreamTask 对应的 key group 范围，在 InternalKeyContext 创建时传入
- `numberOfKeyGroups` key groups 的总数量，等于算子的 max parallelism，在 InternalKeyContext 创建时传入
- `currentKey` StreamTask 正在处理的 Data Record 的 key，由 StreamOperator 运行时调用 StreamOperatorStateHandler#setCurrentKey 设置
- `currentKeyGroupIndex` currentKey 所属的 key group
## 2. Checkpoint Storage
Checkpoint Storage 负责在 checkpoint 时将 State Backend 中的数据持久化保存，目前支持两种类型：
- JobManagerCheckpointStorage 将 state 数据保存在 JobManager 的内存中，只适用于 state 特别小或者测试的环境
- FileSystemCheckpointStorage 将 state 数据保存到外部的文件系统，适用于 state 较大或生产环境

StreamTask 初始化时利用 Checkpoint Storage 初始化 SubtaskCheckpointCoordinator，由其
## 3. StreamOperator 状态持久化
org.apache.flink.streaming.runtime.tasks.SubtaskCheckpointCoordinatorImpl#checkpointState
## 4. StreamOperator 状态恢复
Flink 任务在从 Checkpoint 或 Savepoint 恢复时需要将持久化的 State 反序列化后重新应用到新的（也可能和之前完全一样） ExecutionGraph，这一逻辑在 `StateAssignmentOperation` 中实现
#### Operator Rescale 后 State 恢复
###### OperatorState
OperatorState 也被称为 none-key state，与 Stream Record 之间没有映射关系，因此在 Operator Rescaling 时 OperatorState 的 ListState 可以按 Round Robin 纷发到 Rescale 之后的 Operators

OperatorState 的两种 ListState：
    - `Event-split ListState`：ListState 在 Operator Rescaling 时，之前每个 Operator 的 ListState 作为整体按 Round Robin 划分成 Operator 新并行度的 SubLists
    - `Union ListState`：ListState 在 Operator Rescaling 时，之前每个 Operator 的 ListState 作为整体分发给 Rescaling 之后的每个 Operator   

具体逻辑在 RoundRobinOperatorStateRepartitioner 的 repartition repartitionSplitState repartitionUnionState 方法 

官方文档里的例子是 Kafka Connector：KafkaSourceReader 在执行 snapshotState 时将当前消费的 KafkaPartitionSplit（内含 Partition 和 offset） List 返回给 SourceOperator，然后被更新到 SourceOperator 的 readerState，readerState 是一个来自 OperatorStateBackend 的 `Event-split ListState`。这样在 KafkaSourceReader Rescale 然后从 State 恢复时，之前来自所有 KafkaSourceReaders 的 KafkaPartitionSplit List 可以被均分到所有新的 KafkaSourceReaders 中
###### KeyedState













#### 1.1 StreamOperator 的 State 类型

## 2. StreamOperator 状态持久化和从状态恢复
###### CheckpointListener 接口
- **notifyCheckpointComplete**
某个 checkpoint 成功执行完成时回调该接口
- **notifyCheckpointAborted**
某个 checkpoint 执行失败时回调该接口
###### StreamOperator 接口    
- **prepareSnapshotPreBarrier**   
该接口在算子应该执行 snapshot 的时候调用，此时算子还没有将 checkpoint barier 发送给下游算子。在该接口中算子不做真正的状态持久化，而是在 checkpoint barrier 发送到下游前执行一些发送数据到下游的操作。对于一些无需维护状态的算子，可以利用这个接口在 checkpoint 时将已经接收的在这个 checkpoint barrier 之前的数据处理后发送到下游，这样就在实时上完成了这次 checkpoint
- **snapshotState**
该接口在算子执行 checkpoint 的 snapshot 时调用，算子在该接口中执行真正的状态持久化操作
- **initializeState**
该接口在算子从 checkpoint 恢复时调用，提供 StreamOperatorStateContext 对象恢复算子的状态
###### AbstractStreamOperator
继承 StreamOperator 接口，内部使用 StreamOperatorStateHandler 对象来实际实现 StreamOperator checkpoint 相关的接口   
同时还继承了 CheckpointedStreamOperator 接口，将两个接口做了转换：
- **snapshotState(long,long,CheckpointOptions,CheckpointStreamFactory)** 被转换到 **snapshotState(StateSnapshotContext)**
- **initializeState(StreamTaskStateInitializer)** 被转换到 **initializeState(StateInitializationContext)**
###### CheckpointedFunction
该接口由需要手动操作状态持久化和恢复的 UDF 实现，同样定义了
- **snapshotState(FunctionSnapshotContext)**
- **initializeState(FunctionInitializationContext)**
###### AbstractUdfStreamOperator
UDF 转换成 ExecuteGraph 后对应的 Operator 都继承自 **AbstractUdfStreamOperator**，同时继承自 **AbstractStreamOperator**    
snapshotState 和 initializeState 接口由 `StreamingFunctionUtils` 实现，执行时会传入用户写的 Function 对象
###### StreamingFunctionUtils
- **snapshotFunctionState** 
如果 UDF 是 `WrappingFunction` 则递归地对其内部 Function 做状态持久化，如果 UDF 实现了 `CheckpointedFunction` 接口调用其 snapshotState 方法
- **restoreFunctionState**
如果 UDF 是 `WrappingFunction` 则递归地对其内部 Function 做状态恢复，如果 UDF 实现了 `CheckpointedFunction` 接口调用其 initializeState 方法

## 2. StateBackend 的创建和使用
AbstractStreamOperator#initializeState 从 state 初始化时调用 StreamTaskStateInitializer#streamOperatorStateContext 创建 `StreamOperatorStateContext`，并用 StreamOperatorStateContext 初始化 `StreamOperatorStateHandler`
###### StreamOperatorStateContextImpl
- **isRestored** 是否为从 checkpoint / savepoint 恢复
- **getRestoredCheckpointId** 如果是从 checkpoint / savepoint 恢复 获取其 ID
- **operatorStateBackend** 获取 OperatorStateBackend 对象
- **keyedStateBackend** 获取 KeyedStateBackend 对象
- **rawOperatorStateInputs** 
- **rawKeyedStateInputs** 
- **internalTimerServiceManager** 
#### 配置 StreamOperator 的 stateBackend
1. AbstractStreamOperator 将自己作为参数传入 StreamOperatorStateHandler#initializeOperatorState，在该方法中创建 `StateInitializationContext` 并将 context 作为参数调用 `CheckpointedStreamOperator#initializeState` 初始化自己的状态
2. 这一步由各 Operator 自己实现，可以通过 `ManagedInitializationContext#getOperatorStateStore` 和 `ManagedInitializationContext#getKeyedStateStore` 分别获得 OperatorStateStore 和 KeyedStateStore
#### 配置 UDF 的 stateBackend
1. AbstractStreamOperator 从 StreamOperatorStateHandler#getKeyedStateStore 获取 KeyedStateStore 传递给 StreamingRuntimeContext
2. AbstractUdfStreamOperator#setup 时将 StreamingRuntimeContext 设置为 `RichFunction` 的 runtimeContext    
所以 UDF 实现的 RichFunction 通过 RuntimeContext 只能获取到 KeyedStateStore ，获取不了 OperatorStateStore
#### StateBackend 和 StateStore 之间的关系
###### KeyedStateBackend
1. KeyedStateBackend
2. `CheckpointableKeyedStateBackend`     
继承 `KeyedStateBackend` 和 `Snapshotable<SnapshotResult<KeyedStateHandle>>` 接口，以支持在 checkpoint 时将 Keyed State 持久化，另有 savepoint 接口在 savepoint 时将 Keyed State 持久化     
3. `AbstractKeyedStateBackend`
    - HeapKeyedStateBackend
    - RocksDBKeyedStateBackend    
    HeapKeyedStateBackend 和 RocksDBKeyedStateBackend 使用 CheckpointStreamFactory 执行 snapshot    
###### KeyedStateStore
1. KeyedStateStore    
注册和获取 Keyed State 的接口    
2. DefaultKeyedStateStore     
KeyedStateStore 的基础实现，内部包含一个 KeyedStateBackend 来实际执行 Keyed State 的注册和获取    
    - AbstractPerWindowStateStore
    - MergingWindowStateStore
    - PerWindowKeyedStateStore
    - PerWindowStateStore
###### OperatorStateBackend
1. OperatorStateStore       
注册和获取 Operator State 的接口    
2. OperatorStateBackend     
继承 OperatorStateStore 和 Snapshotable，以支持在 checkpoint 时将 Operator State 持久化    
3. DefaultOperatorStateBackend    
实现 OperatorStateBackend 接口，用 SnapshotStrategyRunner 执行 snapshot    
#### 执行 SnapshotStrategy
将 StateBackend 维护的 State 持久化到系统外部的存储系统
1. SnapshotStrategyRunner.snapshot    
DefaultKeyedStateStore HeapKeyedStateBackend RocksDBKeyedStateBackend 都调用这个方法    
2. SnapshotStrategy#asyncSnapshot   
接口有三个实现：
    - DefaultOperatorStateBackendSnapshotStrategy 对应 DefaultOperatorStateBackend
    - HeapSnapshotStrategy 对应 HeapKeyedStateBackend
    - SavepointSnapshotStrategy 对应 keyStateBackend 执行 savepoint 
    - RocksNativeFullSnapshotStrategy 对应 RocksDBKeyedStateBackend 的全量 snapshot
    - RocksIncrementalSnapshotStrategy 对应 RocksDBKeyedStateBackend 的增量 snapshot
