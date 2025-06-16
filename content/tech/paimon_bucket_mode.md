---
title: "【Paimon】Paimon 的 Bucket 模式"
description: ""
date: "2025-05-27T13:19:43+08:00"
categories:
  - "分布式系统"
tags:
  - "Paimon"
draft: true
---
Paimon 的主键表和 Append Only 表在 Bucket 配置上有不同模式，以及针对各模式有相应的配置和优化
<!--more-->
## 1. Bucket Mode 类型
#### 1.1 主键表
1. `HASH_DYNAMIC`   
不考虑 Cross Partition Update 的 Dynamic Bucket Mode，即组成 Primary Key 的列包含了所有划分 partition 的 fields，这样同一个主键的数据就只会被写到同一个 Partition 中。   
Dynamic Bucket Mode 会将先到达的数据写入旧的 bucket，后到达的数据写入新的 bucket，通过维护一个 Hash Index 文件来映射数据的 key 和 bucket

2. `CROSS_PARTITION`   
考虑存在 Cross Partition Update 的 Dynamic Bucket Mode，需要维护所有 key 和 partition、bucket 的映射，写入任务启动时读取所有表中所有 key 来初始化这个索引

3. `HASH_FIXED`    
固定 bucket 数量，同时也就固定了 Streaming Write 和 Streaming Read 的并发数，适用于单个 partition 内数据量变化不大的情况。

4. `Postpone Bucket`   
Postpone 模式下数据先被写到 partition 的 bucket-postpone 目录中处于不可读的状态，通过 dedicated compcat job 将数据文件分到 bucket 中，不同 partition 可以划分成不同数量的bucket   

#### 1.2 Append Only 表
1. `BUCKET_UNAWARE`   
partition 下不再划分 bucket，

2. `HASH_FIXED`   
同主键表的 HASH_FIXED 模式，同一 bucket 中的数据在被读取时严格遵循写入时的顺序
## 2. 各 Bucket Mode 的代码实现
Flink 的数据流在 org.apache.paimon.flink.sink.FlinkSinkBuilder#build 方法中完成写入 Paimon 表的一系列操作
```java
public DataStreamSink<?> build() {
        // 省略...
        BucketMode bucketMode = table.bucketMode();
        // 在这里根据 Bucket Mode 由不同 DataStreamSink 实现数据流写入 Paimon 表
        switch (bucketMode) {
            case HASH_FIXED:
                if (table.coreOptions().bucket() == BucketMode.POSTPONE_BUCKET) {
                    // Postpone Bucket
                    return buildPostponeBucketSink(input);
                } else {
                    return buildForFixedBucket(input);
                }
            case HASH_DYNAMIC:
                return buildDynamicBucketSink(input, false);
            case CROSS_PARTITION:
                return buildDynamicBucketSink(input, true);
            case BUCKET_UNAWARE:
                return buildUnawareBucketSink(input);
            default:
                throw new UnsupportedOperationException("Unsupported bucket mode: " + bucketMode);
        }
    }
```
#### 2.1 HASH_DYNAMIC
HASH_DYNAMIC 对应的 FlinkSink 是 RowDynamicBucketSink，继承自 DynamicBucketSink
```java
// DynamicBucketSink 的 build 方法实现了将数据流里的 record assign 到对应的 bucket
public DataStreamSink<?> build(DataStream<T> input, @Nullable Integer parallelism) {
        String initialCommitUser = createCommitUser(table.coreOptions().toConfiguration());
        // 先按 key 和 partition 的 hash shuffle 到对应的 assigner，然后 assign bucket，
        // 再按 partition 和 bucket shuffle 到 writer
        // Topology:
        // input -- shuffle by key hash --> bucket-assigner -- shuffle by partition & bucket -->
        // writer --> committer

        // 1. shuffle by key hash
        Integer assignerParallelism = table.coreOptions().dynamicBucketAssignerParallelism();
        if (assignerParallelism == null) {
            assignerParallelism = parallelism;
        }

        Integer numAssigners = table.coreOptions().dynamicBucketInitialBuckets();
        // 将 Paimon 的 ChannelComputer 包装成 Flink 的 StreamPartitioner，由 Flink 的 PartitionTransformation
        // 完成分流的转换，ChannelComputer 由 RowDynamicBucketSink 创建 RowAssignerChannelComputer
        DataStream<T> partitionByKeyHash =
                partition(input, assignerChannelComputer(numAssigners), assignerParallelism);

        // 2. bucket-assigner
        // 由 HashBucketAssignerOperator 算子实现 record 和 bucket 的绑定
        HashBucketAssignerOperator<T> assignerOperator =
                createHashBucketAssignerOperator(
                        initialCommitUser, table, numAssigners, extractorFunction(), false);
        TupleTypeInfo<Tuple2<T, Integer>> rowWithBucketType =
                new TupleTypeInfo<>(partitionByKeyHash.getType(), BasicTypeInfo.INT_TYPE_INFO);
        SingleOutputStreamOperator<Tuple2<T, Integer>> bucketAssigned =
                partitionByKeyHash.transform(
                        DYNAMIC_BUCKET_ASSIGNER_NAME, rowWithBucketType, assignerOperator);
        forwardParallelism(bucketAssigned, partitionByKeyHash);

        String uidSuffix = table.options().get(SINK_OPERATOR_UID_SUFFIX.key());
        if (!StringUtils.isNullOrWhitespaceOnly(uidSuffix)) {
            bucketAssigned =
                    bucketAssigned.uid(
                            generateCustomUid(
                                    DYNAMIC_BUCKET_ASSIGNER_NAME, table.name(), uidSuffix));
        }

        // 3. shuffle by partition & bucket
        DataStream<Tuple2<T, Integer>> partitionByBucket =
                partition(bucketAssigned, channelComputer2(), parallelism);

        // 4. writer and committer
        return sinkFrom(partitionByBucket, initialCommitUser);
    }
```
HashBucketAssignerOperator 负责将 record assign 到 对应的 bucket，在该 Operator 中又是由 BucketAssigner 实现这一功能，Dynamic Bucket Sink 对应的实现类是 HashBucketAssigner
```java
    /** Assign a bucket for key hash of a record. */
    @Override
    // 两个参数分别是 record 所属的 partition 和 record key 的 hash
    public int assign(BinaryRow partition, int hash) {
        int partitionHash = partition.hashCode();
        // 再算一遍 record 对应 assigner 的 ID
        int recordAssignId = computeAssignId(partitionHash, hash);
        checkArgument(
                recordAssignId == assignId,
                "This is a bug, record assign id %s should equal to assign id %s.",
                recordAssignId,
                assignId);
        // PartitionIndex 维护了一个 partition 下的 bucket 信息 
        PartitionIndex index = this.partitionIndex.get(partition);
        if (index == null) {
            partition = partition.copy();
            index = loadIndex(partition, partitionHash);
            this.partitionIndex.put(partition, index);
        }
        // PartitionIndex 决定 record key hash 对应的 bucket
        int assigned = index.assign(hash, this::isMyBucket, maxBucketsNum, maxBucketId);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Assign {} to the partition {} key hash {}", assigned, partition, hash);
        }
        if (assigned > maxBucketId) {
            maxBucketId = assigned;
        }
        return assigned;
    }
```
#### 2.2 HASH_FIXED