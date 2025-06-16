---
title: "【Paimon】Paimon 主键表和 LSM Tree"
date: "2025-02-19T14:37:57+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Paimon"
---
Paimon 以数据表是否有主键，将表分为 `Append Only Table` 和 `Primary Key Table`，对数据的操作前者只支持 Insert，后者还支持 Update 和 Delete，本文主要分析 Paimon 主键表的结构和实现原理
<!--more-->
## LSM Tree 原理
[论文原文](https://www.cs.umb.edu/~poneil/lsmtree.pdf)    
Log Structured Merge Tree 是一种为解决大规模数据写入瓶颈的数据存储结构，其核心是利用磁盘的顺序写在性能上优于随机写这一特点，将批量的随机写转换为一次顺序写，适当牺牲读性能的情况下提升写数据的性能，LSM Tree 被广泛应用于 LevelDB、HBase、Iceberg、RocksDB、Paimon 等分布式数据库   
#### 基本结构
1. LSM Tree 的基本结构由 L0、L1 ... Ln 多个 Level 组成，其中 L0 在内存，L1 至 Ln 在磁盘
2. L0 采用一种排序算法来保持数据按 Key 有序排列，例如红黑树、AVL 树、跳表、TreeMap 等
3. 每一颗子树中的数据量在超过一定阈值后会执行合并，结果被写入下一颗子树
4. L1 至 Ln 中的数据只是按 Key 顺序保存于磁盘，并不维护排序的数据结构，同一 Level 中的多个数据文件中的 key 范围不相交，也称一层 Level 为一个 Sorted Run
5. 只有 L0 中的数据支持原地更新，其余子树中的数据更新采用追加写
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
L0 执行合并时直接将有序数据写入 L1 的一个新的 SST文件（磁盘上实际保存数据的文件） 
2. **L1 - Ln 合并（以L1为例）**
    - L1 - Ln 执行合并时，新旧两个 SST 文件的 key 范围相交，因为新旧两个 SST 都是有序的所以每个子树合并的时间复杂度为 `O(n)`
    - 两个 SST 合并时相同 Key 对应的数据由新 SST 中的替代旧 SST 中的
###### 查询操作
查询时按照 L0 - Ln 的顺序从各子树中逐级查询，一旦查到则直接返回。这样保证了查到的数据一定是最新版本的数据
1. **被查询的数据在 L0**
直接利用 L0 的数据结构查询数据，时间复杂度 `O(logn)` 
2. **被查询的数据在 L1 - Ln**
目标数据在 L0 没有查到逐级查询 L1 - Ln，如果没有任何索引用二分查找每一级时间复杂度 `O(logn)`，也可以在 L1 - Ln 建立稀疏矩阵来减少 IO 次数加速查询
#### 特点
从基本操作可以看到 LSM Tree 的增删改操作都在内存执行，然后利用磁盘顺序读写合并来逐级消除数据冲突，适用于大量数据的导入。在查询上 LSM Tree 需要逐级查找，与 B+ 树 O(logn) 的查询速度相比没有优势。
#### Paimon 的 LSM Tree 结构
[RocksDB Universal Compaction 文档](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)   
上述 LSM Tree 的结构被称为 Leveled Compaction，除此之外 RocksDB 还提出了另一种称为 Universal Compaction 的结构：
- 该结构中数据按 Sorted Run 划分，每个 Sorted Run 中的 SST 文件对应一个时间范围内产生的数据，各 Sorted Runs 的时间范围不相交
- 执行 Compact 时时间范围相邻的 Sort Runs 中的 SST 文件合并到一起，新产生的文件尽可能放到旧的 Sorted Run 中（但要满足各 Sorted Runs 的时间范围不相交的条件）

Paimon 的 LSM Tree 也采用 Universal Compaction 结构，Level0 的每个数据文件单独是一个 Sorted Run，其余每个 Level 是一个 Sorted Run
## Paimon 代码分析
#### 1. 主键表数据写入和触发 Compaction
Flink 数据流从 ``org.apache.paimon.flink.sink.FlinkSink#doWrite`` 方法传入，经过：
- org.apache.paimon.flink.sink.RowDataStoreWriteOperator#processElement
- org.apache.paimon.flink.sink.StoreSinkWriteImpl#write
- org.apache.paimon.table.sink.TableWriteImpl#writeAndReturn
进入 Paimon 的数据表写入逻辑，根据 主键表/非主键表 + Bucket Mode 对应不同的 MemoryFileStoreWrite 的实现类：
- KeyValueFileStoreWrite
- AppendOnlyUnawareBucketFileStoreWrite
- AppendOnlyFixedBucketFileStoreWrite
- PostponeBucketFileStoreWrite
``KeyValueFileStoreWrite`` 对应主键表，与其对应的 RecordWriter 接口实现是 ``MergeTreeWriter``，CompactManager 接口实现是 ``MergeTreeCompactManager``

``MergeTreeWriter`` 将传入的 record 缓存在 WriteBuffer，缓存满后调用 flushWriteBuffer 方法将数据落盘，然后将新增的这一批数据文件的元信息添加到 ``MergeTreeCompactManager`` 的 Level0，并触发 ``MergeTreeCompactManager`` Universal Compaction（非 Full Compaction）
#### 2. 主键表执行 Compaction
``MergeTreeCompactManager#triggerCompaction`` 方法进入 Compaction 的流程，一共有 Full Compaction 和 Universal Compaction 两种模式
- Full Compaction 将所有 Sorted Runs 合并到最高 Level 对应的 Sorted Run
- Universal Compaction 包含 1. Compact Level 0 的全部 Sorted Runs 2. Compact Level 0 ～ Level n-1 的 Sorted Runs 3. Full Compaction 三种情况

Universal Compaction 从 ``CompactStrategy#pick`` 接口获取 Compact 的数据文件，有 UniversalCompaction 和 ForceUpLevel0Compaction 两个实现，其中 ForceUpLevel0Compaction 会先执行 UniversalCompaction，如果没有 pick 到 Sorted Runs 则强制 Level 0 的 Sorted Runs Compact
###### UniversalCompaction 的 pick 策略
1. 是否触发 Optimized Compaction   
如果开启了 ``compaction.optimization-interval`` 会周期性触发 optimization full compation 来保障 [optimized table](https://paimon.apache.org/docs/1.1/concepts/system-tables/#read-optimized-table) 的时效性，Full Compation 会将所有 Sorted Run 都 compat 到最高 Level 对应的 Sorted Run
2. 是否需要减小空间放大   
如果以下条件满足则触发一次 Full Compaction
    > ``当前全部数据文件的 size`` / ``Highest Level 数据文件的 size`` * 100 > ``compaction.max-size-amplification-percent``   
3. 是否有 Sorted Run 满足 Size Ratio
从 L0 的第一个 Sorted Runs 开始遍历，累加每个 Sorted Runs 数据文件的 total size，当 total size 满足条件时合并对应的 Sorted Runs
    ```java
        /**
         * @param maxLevel Higest Level
         * @param runs 所有 Sorted Runs
         * @param candidateCount 这里是 1
         * @param forcePick 这里是 false
         */
        public CompactUnit pickForSizeRatio(
                int maxLevel, List<LevelSortedRun> runs, int candidateCount, boolean forcePick) {
            // candidateCount 为 1，candidateSize 即第一个 Sorted Run 的 total size
            long candidateSize = candidateSize(runs, candidateCount);
            // 按 Level0 ~ Higest Level 遍历所有 Sorted Run
            for (int i = candidateCount; i < runs.size(); i++) {
                LevelSortedRun next = runs.get(i);
                // sizeRatio 由 'compaction.size-ratio' 配置项决定，假设这里是默认值 1
                // 如果 下一个 Sorted Run 的 total size 比当前累加的 total size 大超过 1% 则停止累加
                if (candidateSize * (100.0 + sizeRatio) / 100.0 < next.run().totalSize()) {
                    break;
                }

                candidateSize += next.run().totalSize();
                // candidateCount 记录需要被 compact 的 Sorted Runs
                candidateCount++;
            }

            if (forcePick || candidateCount > 1) {
                return createUnit(runs, maxLevel, candidateCount);
            }

            return null;
        }

        @VisibleForTesting
        CompactUnit createUnit(List<LevelSortedRun> runs, int maxLevel, int runCount) {
            int outputLevel;
            if (runCount == runs.size()) {
                // Full Compaction
                outputLevel = maxLevel;
            } else {
                // Next Sorted Run 对应 Level - 1
                outputLevel = Math.max(0, runs.get(runCount).level() - 1);
            }

            if (outputLevel == 0) {
                // 如果需要压缩的 Sorted Runs 都在 Level 0，则把所有 Level 0 的 Sorted Runs 都压缩到下一个 Level
                for (int i = runCount; i < runs.size(); i++) {
                    LevelSortedRun next = runs.get(i);
                    runCount++;
                    if (next.level() != 0) {
                        outputLevel = next.level();
                        break;
                    }
                }
            }
            // 如果是 Full Compaction 更新前面 Optimized Compaction 执行的时间，避免重复 Full Compaction
            if (runCount == runs.size()) {
                updateLastOptimizedCompaction();
                outputLevel = maxLevel;
            }
            // 最终有三种 Compact 情况：
            // 1. 将 Level 0 的所有 Sorted Runs Compact 到下一个 Level 的 Sorted Run
            // 2. 将 Level 0 ～ Level n-1 的所有 Sorted Runs Compact 到 Level n 的 Sorted Run
            // 3. Full Compaction
            return CompactUnit.fromLevelRuns(outputLevel, runs.subList(0, runCount));
        }
    ```
4. 是否需要控制 Sorted Runs 数量   
如果当前 Sorted Runs 的数量比配置项``num-sorted-run.compaction-trigger``大 n ，则将 n 作为 candidateCount 参数，调用 ``UniversalCompaction#pickForSizeRatio`` 方法得到需要被 Compact 的 Sorted Runs   
和条件 3 相比条件 4 像是对条件 3 的兜底，比如 Level 0 的前两个 Sorted Runs 恰好导致条件 3 的 candidateCount 小于 2，Level 0 的 Compact 就会不触发   
5. 是否需要强制 Compact Level 0 的 Sorted Runs   
当主键表开启 LookUp ChangeLog Producer 或者 First Row MergeEngine 或者 Deletion Vectors 或者 'forece-lookup' 配置为 True，且 'lookup-compact' 配置为 'gentle' 时，如果前面 4 个条件都没有触发，则会为了 Lookup Compact 周期性地触发 Level 0 的 Sorted Runs Compact。
###### Compact 具体的执行过程
前面的过程是挑选出需要执行 Compact 的 CompactUnit，然后将其提交给 ``MergeTreeCompactTask`` 异步执行具体的合并   
``CompactRewriter`` 负责重写 Compact 后的数据文件，接口有多个实现，``MergeTreeCompactRewriter`` 实际执行数据文件合并的 Writer，FullChangelogMergeTreeCompactRewriter 和 LookupMergeTreeCompactRewriter 在 MergeTreeCompactRewriter 的基础上增加写 changlog 的逻辑   
MergeTreeCompactRewriter 执行数据文件合并时先用 ``MergeTreeReaders`` 从 Sorted Runs 读取数据文件，然后将读入的数据合并后写到 Output Level 对应 Sorted Run 的新数据文件，文件类型为 ``FileSource.COMPACT``
#### 3. 读取主键表
合并过程在读取时完成，首先将不同 Sorted Runs 的数据做排序，然后对相同 key 的数据做 merge，这部分涉及的逻辑和模式比较多，单独放到 [【Paimon】Paimon 主键表数据读取逻辑分析](tech/paimon_primary_key_table_read) 一文描述
#### 问题
###### 1. 面试遇到的问题：写入数据时一个 SubTask 可能会被分到不同 partition 和 bucket 的数据，如何将 Buffer 划分给这些数据？
在 `AbstractFileStoreWrite` 里会按 partition 和 bucket 将数据分给不同的 `WriterContainer` ，由 container 里的 `RecordWriter` 负责写到 Buffer 和落盘
```java
public abstract class AbstractFileStoreWrite<T> implements FileStoreWrite<T> {
    ...
    @Override
    public void write(BinaryRow partition, int bucket, T data) throws Exception {
        WriterContainer<T> container = getWriterWrapper(partition, bucket);
        container.writer.write(data);
        if (container.indexMaintainer != null) {
            container.indexMaintainer.notifyNewRecord(data);
        }
    }
    ...
}
```
AbstractFileStoreWrite 的实现类 MemoryFileStoreWrite 在创建 `RecordWriter` 后会调用 `MemoryPoolFactory` 给 `RecordWriter` 分配一个 `MemorySegmentPool` 作为 Buffer，这个 Pool 由 MemoryFileStoreWrite 的 MemorySegmentPool 封装而来，RecordWriter 在使用 Buffer 时由被封装的这个 Pool 实际负责提供 `MemorySegment`

**结论**：同一 SubTask 下分属不同 partition 和 bucket 的 RecordWriter 共用同一个 MemoryPool 作为 Buffer，但在使用时各自维护自己的 MemorySegment
