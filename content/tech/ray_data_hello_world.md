---
title: "【Ray】Ray Data 模块入门"
description: ""
date: "2025-05-15T15:39:42+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Ray"
  - "Ray Data"
---
Ray 框架提供 Ray Data 模块用于分布式数据处理、模型批量离线推理、模型训练数据摄入等场景。
<!--more-->
## 1. 基本概念
#### 1.1 定义 
- **DataSet** 表示一组从外部系统读取或缓存在内存中的分布式数据集，对数据的导入和处理都基于 DataSet 定义
- **Block** 一个 DataSet 由多个 Blocks 组成，每个 Block 表示一组**连续**的数据行，每个 Block 在分布式系统中被独立处理，有点类似 Flink 里的 Split
- **Plan** 通过 Ray Dataset API 编写的任务被构造成 `Logical Plan`，经过 **LogicalOptimizer** 优化后变成 `Optimized Logical Plan`，开始执行后经过 **Planner** 转换成 `Physical Plan`，再经过 **PhysicalOptimizer** 优化成 `Optimized Physical Plan`，优化包括将多个 Operators 合并到一起
- **Operator** 是组成 Plan 的基本元素，`Logical Plan` 对应 `Logical Operator`，`Physical Plan` 对应 `Physical Operator`   
#### 1.2 Physical Operator 的运行过程
1. 获取一批 Blocks 的引用
2. 对这些 Blocks 执行 Operation（要么用 Ray 的 Tasks/Actors 对数据按 UDF 做转换，要么只操作 Blocks 的引用）
3. 输出新的 Blocks 的引用
#### 1.3 流式计算
Ray Data 支持以 streaming 的形式处理整个 DataSet（这里暂时不确定是以 Block 为基本处理单位的 mini batch 式还是以 Row 为单位的真 streaming）   
![官网示例拓扑图](https://docs.ray.io/en/latest/_images/streaming-topology.svg)
如官方文档里的拓扑图所示，Operators 通过 InputQueue 和 OutputQueue 连接成 pipeline，这里有一些与 Flink 对比后未解决的问题：
1. Operator 的上下游逻辑上是否支持多路输入输出
2. 如何实现容错和恢复
3. Operator 在并发执行的时候是如何划分 partition 以及如何 shuffle
## 2. 代码测试
#### 2.1 集群搭建
见 [前文](tech/ray_framework_init.md)
#### 2.2 测试环境和 Hello World
```shell
$ conda create --name ray_hello_world python=3.13.2
$ conda activate ray_hello_world
$ pip install -U "ray[data]"
```
```python
import ray
from typing import Dict
import numpy as np

# 从本地文件读取数据
ds = ray.data.read_csv("local:///Users/zhuxingda/Projects/ray_hello_world/test_data.csv")

# 打印第一行
ds.show(limit=1)

# 定义一个 transformation 在 dataset 里新增一列
def transform_batch(batch: Dict[str, np.ndarray]) -> Dict[str, np.ndarray]:
    weight = batch["weight (kg)"]
    height = batch["height (m)"]
    batch["BMI"] = weight/(height*height)
    return batch

# 将 transformation 应用到 dataset
transformed_ds = ds.map_batches(transform_batch)

# 展示更新后的 Schema 和数据 
# .materialize() 将会执行所有 lazy transformations 并将
# dataset materialize 到 ray 的 object store memory
print(transformed_ds.materialize())
transformed_ds.show(limit=1)
```
执行之后控制台输出
```shell
$ python3 hello_world.py 
2025-05-16 18:41:54,451 INFO worker.py:1888 -- Started a local Ray instance.
2025-05-16 18:41:55,153 INFO dataset.py:3027 -- Tip: Use `take_batch()` instead of `take() / show()` to return records in pandas or numpy batch format.
2025-05-16 18:41:55,158 INFO logging.py:290 -- Registered dataset logger for dataset dataset_1_0
2025-05-16 18:41:55,167 INFO streaming_executor.py:117 -- Starting execution of Dataset dataset_1_0. Full logs are in /tmp/ray/session_2025-05-16_18-41-53_987376_54433/logs/ray-data
2025-05-16 18:41:55,167 INFO streaming_executor.py:118 -- Execution plan of Dataset dataset_1_0: InputDataBuffer[Input] -> TaskPoolMapOperator[ReadCSV] -> LimitOperator[limit=1]
Running 0: 0.00 row [00:00, ? row/s]  2025-05-16 18:41:55,405   INFO streaming_executor.py:220 -- ✔️  Dataset dataset_1_0 execution finished in 0.24 seconds
✔️  Dataset dataset_1_0 execution finished in 0.24 seconds: : 1.00 row [00:00, 4.35 row/s] 
- ReadCSV->SplitBlocks(20): Tasks: 1; Queued blocks: 0; Resources: 1.0 CPU, 24.0B object store: : 1.00 row [00:00, 4.40 row/s]
- limit=1: Tasks: 0; Queued blocks: 1; Resources: 0.0 CPU, 36.0B object store: : 1.00 row [00:00, 4.40 row/s]                 
{'name': 'xiaomign', 'age': 10, 'weight (kg)': 70, 'height (m)': 1.73}
2025-05-16 18:41:55,409 INFO logging.py:290 -- Registered dataset logger for dataset dataset_3_0             
2025-05-16 18:41:55,411 INFO streaming_executor.py:117 -- Starting execution of Dataset dataset_3_0. Full logs are in /tmp/ray/session_2025-05-16_18-41-53_987376_54433/logs/ray-data
2025-05-16 18:41:55,411 INFO streaming_executor.py:118 -- Execution plan of Dataset dataset_3_0: InputDataBuffer[Input] -> TaskPoolMapOperator[ReadCSV] -> TaskPoolMapOperator[MapBatches(transform_batch)]
Running 0: 0.00 row [00:00, ? row/s]                      2025-05-16 18:41:56,032       INFO streaming_executor.py:220 -- ✔️  Dataset dataset_3_0 execution finished in 0.62 seconds->SplitBlocks(20) 1: 0.00 row [00:00, ? row/s]
✔️  Dataset dataset_3_0 execution finished in 0.62 seconds: 100%|████████████████████████████████████████████████████████████████████| 1.00/1.00 [00:00<00:00, 1.61 row/s] 
- ReadCSV->SplitBlocks(20): Tasks: 0; Queued blocks: 0; Resources: 0.0 CPU, 0.0B object store: 100%|█████████████████████████████████| 1.00/1.00 [00:00<00:00, 1.61 row/s]
- MapBatches(transform_batch): Tasks: 0; Queued blocks: 0; Resources: 0.0 CPU, 0.0B object store: 100%|██████████████████████████████| 1.00/1.00 [00:00<00:00, 1.61 row/s]
MaterializedDataset(                                      
   num_blocks=20,                                                                                                                                                         
   num_rows=1,
   schema={
      name: string,
      age: int64,
      weight (kg): int64,
      height (m): double,
      BMI: double
   }
)
2025-05-16 18:41:56,036 INFO logging.py:290 -- Registered dataset logger for dataset dataset_5_0
2025-05-16 18:41:56,039 INFO streaming_executor.py:117 -- Starting execution of Dataset dataset_5_0. Full logs are in /tmp/ray/session_2025-05-16_18-41-53_987376_54433/logs/ray-data
2025-05-16 18:41:56,039 INFO streaming_executor.py:118 -- Execution plan of Dataset dataset_5_0: InputDataBuffer[Input] -> TaskPoolMapOperator[ReadCSV] -> TaskPoolMapOperator[MapBatches(transform_batch)] -> LimitOperator[limit=1]
Running 0: 0.00 row [00:00, ? row/s]  2025-05-16 18:41:56,066   INFO streaming_executor.py:220 -- ✔️  Dataset dataset_5_0 execution finished in 0.03 seconds
✔️  Dataset dataset_5_0 execution finished in 0.03 seconds: : 1.00 row [00:00, 36.7 row/s] 
- ReadCSV->SplitBlocks(20): Tasks: 1; Queued blocks: 0; Resources: 1.0 CPU, 14.4B object store: : 1.00 row [00:00, 38.5 row/s]
- MapBatches(transform_batch): Tasks: 2; Queued blocks: 0; Resources: 2.0 CPU, 29.3B object store: : 1.00 row [00:00, 38.4 row/s]
- limit=1: Tasks: 0; Queued blocks: 2; Resources: 0.0 CPU, 44.0B object store: : 1.00 row [00:00, 38.4 row/s]
{'name': 'xiaomign', 'age': 10, 'weight (kg)': 70, 'height (m)': 1.73, 'BMI': 23.38868655818771}
```
看输出可以发现实际执行了三次 dataset 的流式计算，第三次的 Execution Plan 为：   
1. InputDataBuffer[Input]   
2. TaskPoolMapOperator[ReadCSV]
3. TaskPoolMapOperator[MapBatches(transform_batch)]
4. LimitOperator[limit=1]
## 3. 代码结构
```text
ray/
    data/
        context.py
        dataset.py
        _internal/
            planner/
                planner.py 定义 Planner 负责将 LogicalPlan 转换成 PhysicalPlan
            logical/
                interfaces/
                    plan.py LogicalPlan 和 PhysicalPlan 的公共接口
                    operator.py 定义 Operator 公共接口
                    logical_plan.py 定义 LogicalPlan
                    logical_operator.py 组成 LogicalPlan 的 LogicalOperator 接口
                    physical_plan.py 定义 PhysicalPlan
                    optimizer.py 定义优化 Plan 的 Rule 和 Optimizer 接口
                operators/ 包含一批 LogicalOperator 的实现
                rules/ 包含一批 Rule 的实现
                optimizers.py 定义实现 Optimizer 接口的 LogicalOptimizer 和 PhysicalOptimizer
            execution/
                interfaces/
                    physical_operator.py 定义 PhysicalOperator 接口
                    executor.py 定义 Executor 接口，用于执行 PhysicalOperator
                    task_context.py
                operators/ 包含一批 PhysicalOperator 的实现
                    actor_pool_map_operator.py 包含 _ActorPool 实现了 AutoscalingActorPool 接口，用于 ActorPoolMapOperator 的执行
                autoscaler/
                    autoscaler.py 定义 Autoscaler 接口
                    default_autoscaler.py Autoscaler 接口的默认实现 DefaultAutoscaler
                    autoscaling_actor_pool.py 定义 AutoscalingActorPool 接口，表示支持自动扩缩容的 Actor Pool，由 Autoscaler 决定 AutoscalingActorPool 的扩缩绒
                backpressure_policy/
                    backpressure_policy.py 定义 BackpressurePolicy 接口，表示当数据处理流出现背压后如何控制数据读入的速度
                streaming_executor.py 定义 StreamingExecutor 实现 Executor
                streaming_executor_state.py 定义 OpState 用于追踪 PhysicalOperator 的运行状态
            plan.py 定义 ExecutionPlan
```