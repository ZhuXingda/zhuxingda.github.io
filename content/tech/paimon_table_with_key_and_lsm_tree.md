---
title: "【Paimon】Paimon 主键表和 LSM Tree"
date: "2025-02-19T14:37:57+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Paimon"
---
Paimon 以数据表是否有主键，将表分为 `Append Only Table` 和 `Primary Key Table`，对数据的操作前者只支持 Insert，后者还支持 Update 和 Delete，本文简单分析 Paimon 主键表的原理和格式
<!--more-->
## LSM Tree
[论文原文](https://www.cs.umb.edu/~poneil/lsmtree.pdf)    
Log Structured Merge Tree 是一种为解决大规模数据写入瓶颈的数据存储结构，其核心是利用磁盘的顺序写在性能上优于随机写这一特点，将批量的随机写转换为一次顺序写，适当牺牲读性能的情况下提升写数据的性能，LSM Tree 被广泛应用于 LevelDB、HBase、Iceberg、RocksDB、Paimon 等分布式数据库   
#### 基本结构
1. LSM Tree 的基本结构由 L0、L1 ... Ln 多棵子树组成，其中 L0 在内存，L1 至 Ln 在磁盘
2. L0 采用一种排序算法来保持数据按 Key 有序排列，例如红黑树、AVL 树、跳表、TreeMap 等
3. 每一颗子树中的数据量在超过一定阈值后会执行合并，结果被写入下一颗子树
4. L1 至 Ln 中的数据只是按 Key 顺序保存于磁盘，并不维护排序的数据结构
5. 只有 L0 中的数据支持原地更新，其余子树中的数据更新采用追加写实现
#### 基本操作
###### 写入操作
数据直接被写入内存中的 L0 ，按 Key 插入数据结构中的对应位置，如果该 Key 对应的数据在 L0 中已存在则直接被新数据覆盖，写入操作的时间复杂度为 `O(logn)`
###### 删除操作
删除数据时存在三种情况：
1. **被删除的数据在 L0**   
则将 L0 中的该数据置为 `DELETED` 状态，在下一次 L0 合并时被写入 L1。这里不是直接将该数据从 L0 中移除的原因是这条数据可能已经存在于后面的子树中，如果直接移除并不会对后面子树中的这条数据产生影响，同时按照后文查询时的逻辑还会造成被删除的数据被查询到
2. **被删除的数据在 L1 - Ln**   
不对 L1 - Ln 中的数据做修改，而是直接在 L0 中插入一条 `DELETED` 状态的该数据，后续该数据传到 L1 - Ln 后会在合并时覆盖掉原有的数据   
3. **被删除的数据不存在**   
同样在 L0 中插入一条 `DELETED` 状态的该数据   
综上三种情况执行删除都是在 L0 中操作，其时间复杂度为 `O(logn)`
###### 更新操作
更新操作同删除操作一样也是三种情况：
1. **被更新的数据在 L0**   
直接覆盖更新 L0 中的旧数据
2. **被更新的数据在 L1 - Ln**   
不对 L1 - Ln 中的数据做修改，而是直接将新的数据插入 L0，后续该数据传到 L1 - Ln 后会在合并时覆盖掉原有的数据   
3. **被更新的数据不存在**   
同样直接在 L0 中插入新数据   
综上三种情况执行更新都是在 L0 中操作，其时间复杂度为 `O(logn)`
###### 合并操作
各子树中的数据量在达到阈值后，需要执行`合并（Compact）`操作：
1. **L0 合并**
L0 执行合并时直接将有序数据写入 L1 的一个新的 Block 
2. **L1 - Ln 合并（以L1为例）**
    - L1 执行合并时，将新的 Block 和旧的 Block 进行归并，并将结果写入 L2 的新 Block 并以此类推，因为新旧两个 Block 都是有序的，所以每个子树合并的时间复杂度为 `O(n)`
    - 两个 Block 合并时相同 Key 对应的数据由新 Block 中的替代旧 Block 中的
    - Ln 合并时如果相同 Key 的数据同时存在于新旧 Block 且新 Block 中该数据被标记为 `DELETED` 状态则将数据从合并结果中完全移除（这一项是我猜的）
###### 查询操作
查询时按照 L0 - Ln 的顺序从各子树中逐级查询，一旦查到则直接返回。这样保证了查到的数据一定是最新版本的数据
1. **被查询的数据在 L0**
直接利用 L0 的数据结构查询数据，时间复杂度 `O(logn)` 
2. **被查询的数据在 L1 - Ln**
目标数据在 L0 没有查到逐级查询 L1 - Ln，如果没有任何索引用二分查找每一级时间复杂度 `O(logn)`，也可以在 L1 - Ln 建立稀疏矩阵来减少 IO 次数加速查询
#### 特点总结
从基本操作可以看到 LSM Tree 的增删改操作都在内存执行，然后利用磁盘顺序读写合并来逐级消除数据冲突，适用于大量数据的导入。在查询上 LSM Tree 需要逐级查找，与 B+ 树 O(logn) 的查询速度相比没有优势。

## Paimon 主键表中的 LSM Tree 结构
#### 原理
1. 用 Flink 写 Paimon 表时首先将数据按指定的字段分到不同 bucket，在各 bucket 中维护一个 LSM Tree。
2. 一个 bucket 的数据首先被分发到一个 Flink SubTask，被缓存在内存中并按主键排序和 Merge，结果写入文件作为 LSM Tree 的 L0。
3. 
#### 源码分析
###### 数据写入过程
1. org.apache.paimon.flink.sink.FlinkSink#doWrite   
传入数据流并将数据流写入 Paimon ，写入的方式是用 Flink 的 DataStream transform 接口，传入的 OneInputStreamOperatorFactory 提供具体的写入逻辑
2. org.apache.paimon.flink.sink.FlinkSink#createWriteOperatorFactory   
接口定义了不同类型的 OneInputStreamOperatorFactory，对应 Paimon 支持的多种数据格式，这里以 Fixed Bucket 模式为例，实际的 OneInputStreamOperator 是 RowDataStoreWriteOperator
3. org.apache.paimon.flink.sink.RowDataStoreWriteOperator#processElement   
RowDataStoreWriteOperator 将传入的 record 交由 StoreSinkWrite 执行实际的写入   
StoreSinkWrite 在 RowDataStoreWriteOperator 的父类 TableWriteOperator 中完成初始化   
初始化时如果 `sink.use-managed-memory-allocator` 配置参数为 true ，会将 TableWriteOperator 父类 PrepareCommitOperator 的 FlinkMemorySegmentPool 作为 MemorySegmentPool 传给 StoreSinkWrite，用于给 LSM Tree 分配内存，内存来自于 Flink 的 `Managed Memory`
4. org.apache.paimon.flink.sink.StoreSinkWriteImpl#write   
StoreSinkWrite 的实现类 StoreSinkWriteImpl 使用 TableWriteImpl 执行实际的写入，如果前面有传入 FlinkMemorySegmentPool 则进一步传给 TableWriteImpl，否则创建 HeapMemorySegmentPool 传给 TableWriteImpl，HeapMemorySegmentPool 使用的是 `JVM Heap`
5. org.apache.paimon.table.sink.TableWriteImpl#writeAndReturn
从 TableWriteImpl 开始进入 Paimon 单独的写入流程逻辑不再和 Flink 相关，TableWriteImpl 使用 FileStoreWrite 执行实际的写入
6. org.apache.paimon.operation.AbstractFileStoreWrite#write
AbstractFileStoreWrite 是 FileStoreWrite 的唯一实现类，内部使用 WriterContainer 的 RecordWriter 执行 write 和 compact，RecordWriter 有 AppendOnlyWriter、MergeTreeWriter、PostponeBucketWriter 三个实现，前两个分别对应 Append Only Table 和 Primary Key Table，后一个是新的格式还没来得及看
7. org.apache.paimon.mergetree.MergeTreeWriter#write   
数据被写到 WriteBuffer，其唯一实现类 SortBufferWriteBuffer 内部使用 SortBuffer 来实际缓存数据，前面传入的 MemorySegmentPool 作为 Buffer 内存的来源， MergeTreeWriter#flushWriteBuffer 时会将 Buffer 中的数据按 keyComparator 排序，相同 key 的数据按 mergeFunction 聚合，结果被 RollingFileWriter 写入文件   
8. org.apache.paimon.io.RollingFileWriter#write   
###### 数据 Compact 过程
MergeTreeWriter 内通过一个 CompactManager 来完成 LSM Tree 每一级的 compact
1. org.apache.paimon.mergetree.MergeTreeWriter#flushWriteBuffer   
MergeTreeWriter 将内存中的数据落盘，新增的文件元信息添加给 CompactManager，并在最后触发 CompactManager 执行 compact 
2. org.apache.paimon.mergetree.compact.MergeTreeCompactManager#triggerCompaction   
MergeTreeCompactManager 内维护一个 Levels 对象对应 LSM Tree 的每一级，Levels 中每一级由一个 SortedRun 表示，其内部是一个 DataFileMeta 列表表示这一级对应哪些文件。 MergeTreeCompactManager 由 CompactStrategy 挑选出 CompactUnit 执行 compact    
3. org.apache.paimon.mergetree.compact.UniversalCompaction#pick   
CompactStrategy 挑选需要 compact 的文件，
###### Commit 过程
1. org.apache.paimon.flink.sink.RowDataStoreWriteOperator#prepareCommit   
RowDataStoreWriteOperator 继承自 PrepareCommitOperator，每次执行 checkpoint 前 PrepareCommitOperator#prepareSnapshotPreBarrier 会调用 RowDataStoreWriteOperator#prepareCommit 完成 commit 并将 Committable 传递到下游   
2. org.apache.paimon.flink.sink.StoreSinkWriteImpl#prepareCommit
3. org.apache.paimon.table.sink.TableWriteImpl#prepareCommit   
进入 Paimon 单独的 commit 信息生成逻辑
4. org.apache.paimon.operation.AbstractFileStoreWrite#prepareCommit   
遍历每个 partition + bucket 对应的 MergeTreeWriter 执行 prepareCommit
5. org.apache.paimon.mergetree.MergeTreeWriter#prepareCommit
commit 调用 MergeTreeWriter#flushWriteBuffer 完成数据落盘和 compact，然后根据配置决定是否等待 compact 执行结束，最后将数据文件的变动和 compact 的记录写入 CommitIncrement
6. org.apache.paimon.flink.sink.RowDataStoreWriteOperator#prepareCommit
CommitIncrement 被转换为 CommitMessageImpl 被传递到下游

