---
title: 【Flink 基础】Data Source API 结构及实现
date: 2024-07-13
tags:
    - Flink 
categories:
    - "大数据"
---
Flink 1.11 引入新的 **Data Source API** 以取代 **SourceFunction** 接口，本文将简述其结构，以便针对某个数据源开发自定义的 Source 连接器。
<!--more-->
### Data Source API 结构
![Data Source 模型](https://nightlies.apache.org/flink/flink-docs-master/fig/source_components.svg)
- **Split**： 描述数据源中部分数据的元数据，不是数据本身，**SourceReader** 拿着这个元数据去数据源读取对应的数据。
- **SplitEnumerator**： 位于 **JobManager**，负责把数据源的数据从逻辑上拆分成多个 **Splits**，并把每个 **Split** 分发给 **SourceReader**。
- **SourceReader**： 位于 **TaskManager**，负责向 **SplitEnumerator** 请求新的 **Split**，然后根据 **Split** 读取数据，并把数据传递给下游的算子。

### 接口定义
在 flink-core 包下 org.apache.flink.api.connector.source 目录里定义了这些接口：
1. **Source**   
数据源的最外层接口
    - getBoundedness：是否为有界数据源
    - createEnumerator：创建新的 **SplitEnumerator**
    - restoreEnumerator：从 checkpoint 恢复 **SplitEnumerator**
    - getSplitSerializer： 获取 **Split** 的序列化工具，Split 在从 **SplitEunmerator** 传输到 **SourceReader** 时需要序列化，**SourceReader** 执行 checkpoint 快照时也需要对 **Split** 序列化
    - getEnumeratorCheckpointSerializer: 获取 **SplitEnumerator** 的 checkpoint 序列化工具，**SplitEnumerator** 执行 checkpoint 快照时需要将其状态序列化，在 restoreEnumerator 时恢复
2. **SplitEnumerator**   
Split 分配器
    - start：启动分配器
    - handleSplitRequest：处理 **SourceReader** 发来的请求，为其分配新的 **Split**，这里是一个经典的生产者消费者模型
    - addSplitsBack：**SourceReader** 执行失败时会把分配的 **Split** 还给 **SplitEnumerator**
    - addReader：通知新添加了一个 **SourceReader** 及其对应的 subTask ID
    - snapshotState：执行 checkpoint 快照，返回分配器的状态序列化结果
    - handleSourceEvent：处理从某个 subTask 的 **SourceReader** 发来的事件
    - close：关闭分配器
3. **SourceReader**   
读取和传输数据
    - start：启动 **SourceReader**
    - pollNext：输出数据记录到下游，该方法不能阻塞，推荐一次只输出一条记录。返回包含三种 **InputStatus** 状态：
        - MORE_AVAILABLE：还有数据可以立即输出到下游
        - NOTHING_AVAILABLE：暂时没有数据，但后续还有数据可以输出
        - END_OF_INPUT：已经没有数据可以输出
    - snapshotState：执行 checkpoint 快照，将正在读取和还未读取的 **Split** 序列化，作为 **SourceReader** 的状态
    - isAvailable：返回一个 CompletableFuture 来标识当前 **SourceReader** 是否有数据可以输出。如果该 Future 走到 complete 状态，则 Flink 的 runtime 会持续调用 pollNext 方法来输出数据。如果 pollNext 返回除 MOER_AVAILABLE 外的状态，则 runtime 会再次调用 isAvailable 方法并不断重复。
    - addSplits：将 **SplitEnumerator** 分配的新 **Split** 发送到该 **SourceReader**
    - notiFyNoMoreSplits：当 **SplitEnumerator** 没有更多 Split 可以分配给该 **SourceReader** 时（**SplitEnumeratorContext** 的 SplitEnumeratorContext 方法调用时），会调用该方法
    - handleSourceEvents：处理从 **SplitEnumerator** 发来的事件
    - pauseOrResumeSplits：暂停或恢复从指定的 **Splits** 读取数据，**SourceOperator** 在对齐 **Splits** 的 watermark 时会调用该方法
4. **SourceSplit**   
Split 接口   
    Flink 并未对 Split 的格式做限制，完全取决于接口的实现
5. **SourceReaderContext**   
负责将一些 runtime 信息暴露给 **SourceReader**
    - metricGroup：当前 **Source** 的 metric group，用于上报 metric
    - getConfiguration：获取 Flink 任务的配置，实现自定义 Source 时自定义的一些配置从这里传到 **SourceReader**
    - sendSplitRequest：**SourceReader** 向 **SplitEnumerator** 请求新的 Split
    - getIndexOfSubtask：当前 **SourceReader** 对应的 subTask ID
    - currentParallelism：当前 **Source** 的并行度

### 高级接口定义
flink-core 包下的接口定义对 Data Source 里各成员的功能做了基本描述，其中 **SourceReader** 是完全异步的（ pollNext 不能阻塞）。但很多外部数据源在读数据时是同步的操作（比如 Kafka Client 的 poll），所以需要把这些同步读数据和 **SourceReader** 的异步 pollNext 分到不同的线程执行，中间用一个队列来传递数据。Flink 提供了一个高级接口 **SplitReader** 来实现这个功能。
1. **SourceReaderBase**   
实现了 **SourceReader** 接口，其内部用 **SplitFetcherManager** 管理负责读取数据的 **SplitFetcher**，读取的数据被写到阻塞队列经 **RecordEmitter** 消费后推送到下游
2. **SplitFetcher**   
继承 **Runnable** 接口，在 run 方法中不断地顺序执行 taskQueue 里的任务，直到被 shutdown 
主要的 fields 有：   
| field | 类型 | 功能 |
| ---- | ---- | ---- |
| taskQueue | Deque<SplitFetcherTask> | SplitFetcher 的任务队列，包含待完成的任务（SplitFetcherTask） |
| assignedSplits | Map<String, SplitT> | 分配到该 SplitFetcher 的 Splits，key 为 Split 的 ID，Split 读取完成后从 assignedSplits 里移除 |
| splitReader | SplitReader | 实际执行从 splits 读取数据，并把数据推到 elementsQueue |
| elementsQueue | FutureCompletingBlockingQueue<RecordsWithSplitIds<E>> | 存放从 splits 读出的数据记录，RecordEmitter 从这个队列里消费并推送到下游 |
3. **SplitFetcherTask**   
**SplitFetcher**里面循环执行的任务，包括：
    - **FetchTask**： 用 **SplitReader** 读取数据并推送到 elementsQueue；
    - **AddSplitsTask**： 往 assignedSplits 里添加 splits；
    - **RemoveSplitsTask**： 从 assignedSplits 和 splitReader 里移除读完的 splits，并调用 splitFinishedCallback；
    - **PauseOrResumeSplitsTask** 调用 splitReader 停止或恢复读取指定的 splits；
4. **SplitFetcherManager**   
![](https://nightlies.apache.org/flink/flink-docs-master/fig/source_reader.svg)
抽象类，负责维护一个 **SplitFetcher** 池，并把 splits 分配给 SplitFetchers ，每个 SplitFetcher 用一个 **SplitReader** 从 split 读取数据并转发到 elementQueue ；如果一个 **SplitFetcher** 长时间没有 splits 读取或者 taskQueue 为空，则将其关掉
5. **SplitReader**   
定义实际从 splits 读取数据的接口
    - fetch： 从 splits 读取数据返回 **RecordsWithSplitIds** 。该方法可以是**阻塞**的，但当 wakeUp 被调用时应跳出阻塞，可以抛出 interupted 异常或者仅只是返回结果，无论如何响应下一次调用时都应该从上次 fetch 响应的位置继续读取。实现该方法时可以一次性返回所有数据，也可以返回一批数据。
    - handleSplitsChanges： 响应 SplitsAddition 和 SplitsRemoval 的变更
    - wakeUp： fetch 方法阻塞时调用 wakeUp 跳出阻塞
    - pauseOrResumeSplits： 暂停或恢复读取指定的 splits
6. **RecordsWithSplitIds**   
用来包装数据记录的接口，splitsIterator 包含了每个 split 的 ID 和其对应的数据
7. **RecordEmitter**   
向下游推送数据的接口，推送的同时需要更新 splitState，这样当 Source 从 state 恢复时能够从最后一次成功推送的数据的下一个位置开始读取。
8. **SingleThreadMultiplexSourceReaderBase**   
SourceReaderBase 的抽象实现，提供了 elementsQueue （FutureCompletingBlockingQueue）和 SplitFetcherManager 的单线程模型实现 **SingleThreadFetcherManager**，其内部只保持最多一个 **SplitFetcher** 拉取数据

### Event Time 和 Watermarks
**WatermarkStrategy** 在构建 Source 时定义，包含 **TimeStampAssigner** 和 **WatermarkGenerator**，两个构造器在 Source 输出数据后调用   
##### Event Timestamps
Event Timestamp 可以由 **TimeStampAssigner** 在数据输出后绑定，也可以由 **SourceReader** 在往 **ReaderOutput** 输出数据时通过调用 collect(record, timestamp) 给数据记录附上时间戳
##### Watermarks Generation
- Watermark Generation 只在 streaming 模式下生效，支持对每个 split 生成独立的 watermark，以更好的观察 Event Time 倾斜以及防止暂停的 partitions 拖累整个任务的 Event Tiem 进度
- 继承高级接口的 **SplitReader** 可以自动实现分 split 生成 watermark
- 继承初级接口的 **SourceReader** 需要借用 **ReaderOutput** 的 createOutputForSplit(splitId) releaseOutputForSplit(splitId) 创建和释放 split 对应的输出

### 参考
https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/sources/#the-split-reader-api