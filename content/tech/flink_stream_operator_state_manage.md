---
title: Flink Stream Operator 状态控制
date: 2024-11-28
tags:
    - Flink 
categories:
    - "分布式系统"
draft: true
---
前文分析了 Flink Checkpoint 的产生和传递过程，本文接着分析算子在 checkpoint 过程中的状态控制。
<!--more-->
## 状态持久化和从状态恢复的过程
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
UDF 转换成 ExecuteGraph 后对应的 Operator 都继承自 **AbstractUdfStreamOperator**，而该又继承自 **AbstractStreamOperator**    
snapshotState 和 initializeState 接口由 `StreamingFunctionUtils` 实现，执行时会传入用户写的 Function 对象
###### StreamingFunctionUtils
- **snapshotFunctionState** 
如果 UDF 是 `WrappingFunction` 则递归地对其内部 Function 做状态持久化，如果 UDF 实现了 `CheckpointedFunction` 接口调用其 snapshotState 方法
- **restoreFunctionState**
如果 UDF 是 `WrappingFunction` 则递归地对其内部 Function 做状态恢复，如果 UDF 实现了 `CheckpointedFunction` 接口调用其 initializeState 方法
## stateBackend 的创建和使用
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
2. CheckpointableKeyedStateBackend     
继承 KeyedStateBackend 和 Snapshotable，以支持在 checkpoint 时将 Keyed State 持久化，同时增加了 savepoint 接口用于在 savepoint 时将 Keyed State 持久化     
3. AbstractKeyedStateBackend
    - HeapKeyedStateBackend
    - RocksDBKeyedStateBackend    
    HeapKeyedStateBackend 和 RocksDBKeyedStateBackend 使用 RocksDBKeyedStateBackend 执行 snapshot    
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
