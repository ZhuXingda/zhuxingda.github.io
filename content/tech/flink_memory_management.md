---
title: 【Flink】Taskmanager 内存管理
date: 2023-01-03
tags:
    - Flink 
categories:
    - "开源框架"
# thumbnail: https://nightlies.apache.org/flink/flink-docs-release-1.20/fig/detailed-mem-model.svg
---
Flink Taskmanager 的内存划分和调整策略。
<!--more-->
### Flink Taskmanager 内存模型
<img src="https://nightlies.apache.org/flink/flink-docs-release-1.20/fig/detailed-mem-model.svg" alt="Flink Taskmanager 内存模型" width="50%">

整个内存模型物理上分为 **JVM Heap** 和 **Off-Heap** 两部分，逻辑上可以分为 **Total Process Memory** 和 **Total Flink Memory** 两部分。     
**Total Process Memory** 为整个 Flink JVM 进程使用的全部内存

> **Total Process Memory** = Total Flink Memory + JVM Metaspace + JVM Overhead  

配置项为 [taskmanager.memory.flink.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-flink-size) 和 [	taskmanager.memory.process.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-process-size)，两个选项配置任意一个即可。对于 Kubernetes session mode 部署，在启动 session 时使用 **taskmanager.memory.process.size** 配置。
##### JVM Heap (堆内存)
1. **Framework Heap Memory**    
Flink 框架运行所需要使用的内存，配置项为 [taskmanager.memory.framework.heap.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-framework-heap-size)，默认值为 128 MB，一般不建议修改。
2. **Task Heap Memory**  
Flink 任务算子和用户代码运行使用的内存，配置项为[taskmanager.memory.task.heap.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-task-heap-size)，默认值为 **Total Flink Memory** 减去 **Framework Heap Memory**, **Framework Off-Heap Memory**, **Task Off-Heap Memory**, **Managed Memory** 和 **Network Memory**
##### Off-Heap（非堆内存）
1. **Managed memory**   
由 Flink 内存管理器管理的非堆内存，主要用于：
    - 内置算法：排序、哈希表、缓存中间结果
    - RocksDB state backend。
    - Python UDFs  

    配置项为 [taskmanager.memory.managed.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-managed-size) 和 [taskmanager.memory.managed.fraction](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-managed-fraction)。默认为 Total Flink Memory 的 40%。

    对于不使用三种功能的场景，可以把 taskmanager.memory.managed.fraction 设置为 0，来增大用户代码能使用的内存。
2. **Framework Off-Heap Memory**  
Flink 框架运行所需要使用的非堆内存，配置项为 [taskmanager.memory.framework.off-heap.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-framework-off-heap-size)，默认值为 128 MB，不建议修改。
3. **Task Off-Heap Memory**  
Flink 任务算子和用户代码运行使用的非堆内存，配置项为 [taskmanager.memory.task.off-heap.size](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-task-off-heap-size)，默认值为 0，如果用户代码里有用到非堆内存的情况，需要手动设置。
4. **Network Memory**  
TaskManager 之间传输数据用到的直接内存，比如网络缓存，配置项为 [taskmanager.memory.network.min](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-network-min) [taskmanager.memory.network.max](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-network-max) [taskmanager.memory.network.fraction](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-network-fraction)，网络缓存调优[相关指南](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/memory/network_mem_tuning/)。默认值为 Total Flink Memory 的 10%，最大不默认没有上限，最小默认 64MB。
5. **JVM Metaspace** 和 **JVM Overhead**  
JVM Metaspace 和 JVM Overhead 的内存，配置项为 [taskmanager.memory.jvm-metaspace.size
](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-jvm-metaspace-size) [taskmanager.memory.jvm-overhead.fraction]() 等。其中 Overhead 默认为 **Total Process Memory** 的 10%，最大默认不超过 1024MB，最小默认大于 192MB；Metaspace 默认为 256MB。

### On Kubernetes Cluster 具体配置
如果按如下配置
```yaml
# taskmanager 向 k8s 申请的总内存大小
taskmanager.memory.process.size: 8192m
# taskmanager.memory.managed.fraction 设为 0，增大用户代码能使用的内存
taskmanager.memory.managed.fraction: 0
```
最后分配结果就是：
- **Framework Heap Memory** 128MB
- **Managed Memory** 0MB
- **Framework Off-Heap Memory** 128MB
- **Task Off-Heap Memory** 0MB
- **Network Memory** (8192MB - 256MB - 819MB) * 0.1 = 712 MB
- **JVM Metaspace** 256MB
- **JVM Overhead** 8192MB * 0.1 = 819MB
- **Task Heap Memory** 8192MB - 128MB - 128MB - 0MB - 819MB - 256MB - 712MB = 6149MB
### 参考
https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/memory/mem_setup_tm/    