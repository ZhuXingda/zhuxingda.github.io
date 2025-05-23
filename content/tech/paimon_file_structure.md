---
title: "【Paimon】文件结构"
description: ""
date: "2025-05-20T14:35:22+08:00"
thumbnail: ""
categories:
  - ""
tags:
  - ""
---
[官方文档](https://paimon.apache.org/docs/1.1/concepts/basic-concepts/) 介绍了 Paimon 的文件结构，通过文件结构能够更好理解如何实现数据更新等操作
<!--more-->
![全局文件结构](https://paimon.apache.org/docs/1.1/img/file-layout.png)
## 1. 文件结构
#### 1.1 Snapshot
每个 Snapshot 文件对应该 Paimon 表的一个版本，向 Paimon 表写数据的 Flink 任务每一次 Checkpoint 触发 Paimon 表的 commit，创建对应的 Snapshot 文件。   
如果开启了同步 Compact 每次 commit 会产生`增量写入`和 `Compact` 两个 Snapshot，如果是异步 Compact 则只产生一个 Snapshot   
Snapshot 文件中包含：   
- 这一版本数据表的 schema file，定义表的结构
- 这一版本数据表的 manifest list，定义表的数据
#### 1.2 Manifest List 和 Manifest File
Manifest List 文件包含一批 Manifest File 文件名，Manifest File 里表示新增和删除了哪些 Data File 和 Changelog
#### 1.3 Data File
Data File 是数据实际保存数据的文件，按 Partition 和 Bucket 划分到不同目录中
#### 1.4 Schema
Schema 文件定义了数据表的结构
#### 1.5 Index
Index 文件对应表的 Index，Dynamitc Bucket 的表有 Dynamic Bucket Index，详见[官方文档](https://paimon.apache.org/docs/1.1/concepts/spec/tableindex/)
## 2. 文件操作
根据 [官方文档](https://paimon.apache.org/docs/1.1/learn-paimon/understand-files/#understand-files) 执行 SQL 命令观察文件变化
#### 2.1 创建表格
```SQL
CREATE CATALOG paimon WITH (
 'type' = 'paimon',
 'warehouse' = 'file:///home/zhuxingda/tmp/paimon'
);
USE CATALOG paimon;
CREATE TABLE T (
  id BIGINT,
  a INT,
  b STRING,
  dt STRING COMMENT 'timestamp string in format yyyyMMdd',
  PRIMARY KEY(id, dt) NOT ENFORCED
) PARTITIONED BY (dt);
```
对应路径下创建了 Schema 文件 schema/schema-0
```json
{
  "version" : 3,
  "id" : 0,
  "fields" : [ {
    "id" : 0,
    "name" : "id",
    "type" : "BIGINT NOT NULL"
  }, {
    "id" : 1,
    "name" : "a",
    "type" : "INT"
  }, {
    "id" : 2,
    "name" : "b",
    "type" : "STRING"
  }, {
    "id" : 3,
    "name" : "dt",
    "type" : "STRING NOT NULL",
    "description" : "timestamp string in format yyyyMMdd"
  } ],
  "highestFieldId" : 3,
  "partitionKeys" : [ "dt" ],
  "primaryKeys" : [ "id", "dt" ],
  "options" : { },
  "comment" : "",
  "timeMillis" : 1747730863469
}
```
#### 2.2 写入数据
```SQL
INSERT INTO T VALUES (1, 10001, 'varchar00001', '20230511');
```
写入之后文件变化：   
```shell
$ ls
'dt=20230511'   index   manifest   schema   snapshot
```
###### snapshot   
snapshot 目录下有三个文件：
- EARLIEST 最旧的 Snapshot ID
- LATEST 最新的 Snapshot ID
- snapshot-1 

``snapshot-1`` 文件内容：
```json
{
  "version" : 3,
  "id" : 1,
  // 对应前面 schema/schema-0 文件 
  "schemaId" : 0,
  // 对应后面 manifest/manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-0 文件
  "baseManifestList" : "manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-0",
  // 对应后面 manifest/manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-1 文件
  "baseManifestListSize" : 884,
  "deltaManifestList" : "manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-1",
  "deltaManifestListSize" : 1004,
  // 没有开启 changelog 所以为空
  "changelogManifestList" : null,
  // 对应后面 index/index-manifest-b0b72e67-cd38-403b-8329-3d40376199a6-0 文件
  "indexManifest" : "index-manifest-b0b72e67-cd38-403b-8329-3d40376199a6-0",
  "commitUser" : "390abd21-7e45-4f16-a749-d0a02dfb49f4",
  "commitIdentifier" : 9223372036854775807,
  // commit 类型为 APPEND，区别于 COMPACT
  "commitKind" : "APPEND",
  "timeMillis" : 1747739233763,
  "logOffsets" : { },
  "totalRecordCount" : 1,
  "deltaRecordCount" : 1,
  "changelogRecordCount" : 0,
  // 建表语句里没有配置 watermak 所以为 Long.MIN_VALUE
  "watermark" : -9223372036854775808
}
```
###### manifest
manifest 目录下有四个文件：
- index-manifest-b0b72e67-cd38-403b-8329-3d40376199a6-0 记录 Index 文件的 manifest 文件 
- manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0 Manifest 文件
- manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-0 Base Manifest List 文件
- manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-1 Delta Manifest List 文件

``index-manifest-b0b72e67-cd38-403b-8329-3d40376199a6-0`` 文件内容
```json
{
    "_VERSION": 1,
    "_KIND": 0,
    "_PARTITION": "\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\b\u0000\u0000\u0000\u0010\u0000\u0000\u000020230511",
    "_BUCKET": 0,
    "_INDEX_TYPE": "HASH",
    // 对应的 index 文件 
    "_FILE_NAME": "index-a5b79ea6-71b5-48f2-bad5-b13cc28e3efe-0",
    "_FILE_SIZE": 4,
    "_ROW_COUNT": 1,
    "_DELETIONS_VECTORS_RANGES": null
}
```
``manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0`` 文件内容
```json
{
    "_VERSION": 2,
    "_KIND": 0,
    "_PARTITION": "\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\b\u0000\u0000\u0000\u0010\u0000\u0000\u000020230511",
    "_BUCKET": 0,
    // -1 为 Dynamic Bucket Mode
    "_TOTAL_BUCKETS": -1,
    // 对应的 Data Files
    "_FILE": {
        "_FILE_NAME": "data-07a031d4-5b2c-4710-8fad-ebea22b4efcf-0.parquet",
        "_FILE_SIZE": 1514,
        "_ROW_COUNT": 1,
        "_MIN_KEY": "\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000",
        "_MAX_KEY": "\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0001\u0000\u0000\u0000\u0000\u0000\u0000\u0000",
        // Data File 中 Key 的统计信息
        "_KEY_STATS": {
           // ...
        },
        "_VALUE_STATS": {
           // ...
        },
        "_MIN_SEQUENCE_NUMBER": 0,
        "_MAX_SEQUENCE_NUMBER": 0,
        "_SCHEMA_ID": 0,
        "_LEVEL": 0,
        "_EXTRA_FILES": [

        ],
        "_CREATION_TIME": {
            "long": 1747768033648
        },
        "_DELETE_ROW_COUNT": {
            "long": 0
        },
        "_EMBEDDED_FILE_INDEX": null,
        "_FILE_SOURCE": {
            "int": 0
        },
        "_VALUE_STATS_COLS": null,
        "_EXTERNAL_PATH": null
    }
}
```
``manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-0`` 文件为空，因为是初始 Base Manifest List 所以没有对应的 Manifest File   
``manifest-list-44f32000-5ecd-43b9-b3f1-21982b56f31c-1`` 文件内指定了 manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0 作为新增的 Manifest File   
###### Data File
dt=20230511/bucket-0/data-07a031d4-5b2c-4710-8fad-ebea22b4efcf-0.parquet
#### 2.3 追加写入数据
```SQL
INSERT INTO T VALUES 
(2, 10002, 'varchar00002', '20230502'),
(3, 10003, 'varchar00003', '20230503'),
(4, 10004, 'varchar00004', '20230504'),
(5, 10005, 'varchar00005', '20230505'),
(6, 10006, 'varchar00006', '20230506'),
(7, 10007, 'varchar00007', '20230507'),
(8, 10008, 'varchar00008', '20230508'),
(9, 10009, 'varchar00009', '20230509'),
(10, 10010, 'varchar00010', '20230510');
```
- snapshot 新增 snapshot-2，baseManifestList 变成 manifest-list-56aee536-859b-45ea-bef2-5d322ea0e0b2-0，deltaManifestList 变成 manifest-list-56aee536-859b-45ea-bef2-5d322ea0e0b2-1 
- manifest 新增：
    - manifest-list-56aee536-859b-45ea-bef2-5d322ea0e0b2-0 内容包含 manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0，即前一次写入时创建的 Manifest File
    - manifest-list-56aee536-859b-45ea-bef2-5d322ea0e0b2-1 内容包含 manifest-e789f1bb-60d5-4b8e-aa55-52789ecd06f6-0，即新一次写入创建的 Manifest File
    - manifest-e789f1bb-60d5-4b8e-aa55-52789ecd06f6-0 内容包含从 dt=20230502 到 dt=20230510 新增的所有 Data Files
    - index-manifest-d18b2c00-420f-457d-b8d3-183613f3c80d-0 内容包含 index 目录下新增的 Index Files
#### 2.4 删除数据
```SQL
SET 'execution.runtime-mode' = 'batch';
DELETE FROM T WHERE dt >= '20230503';
```
- snapshot 新增 snapshot-3，baseManifestList 为 manifest-list-c2a06170-533b-4df8-9510-e37daa4e8ff4-0，deltaManifestList 为 manifest-list-c2a06170-533b-4df8-9510-e37daa4e8ff4-1，commitKind 依然是 APPEND
- manifest 新增
    - manifest-list-c2a06170-533b-4df8-9510-e37daa4e8ff4-0 内容包含 manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0 和 manifest-e789f1bb-60d5-4b8e-aa55-52789ecd06f6-0，对应前面两次写入创建的 Manifest File
    - manifest-list-c2a06170-533b-4df8-9510-e37daa4e8ff4-1 内容为 manifest-7b96fb3e-74b3-46ee-ad21-ef76151d7e09-0，对应删除时创建的 Manifest File
    - manifest-7b96fb3e-74b3-46ee-ad21-ef76151d7e09-0 内容为从 dt=20230503 到 dt=dt=20230510 新增的 Data Files（删除也是新增数据文件）
- Data Files dt=20230503 到 dt=20230510 新增一个数据文件
#### 2.5 Compact
```SQL
CALL sys.compact('default.T');
```
- snapshot 新增 snapshot-4，baseManifestList 为 manifest-list-ebca7993-71a3-447f-930c-4f9b6412cb1a-0，deltaManifestList 为 manifest-list-ebca7993-71a3-447f-930c-4f9b6412cb1a-1，commitKind 为 COMPACT
- manifest 新增
    - manifest-list-ebca7993-71a3-447f-930c-4f9b6412cb1a-0 内容包含 manifest-7b05638a-9361-406c-afa5-bcf4ec005d8a-0、manifest-e789f1bb-60d5-4b8e-aa55-52789ecd06f6-0、manifest-7b96fb3e-74b3-46ee-ad21-ef76151d7e09-0，对应前三次操作新增的所有 DataFiles
    - manifest-list-ebca7993-71a3-447f-930c-4f9b6412cb1a-1 内容包含 manifest-7fd62edf-7d04-46ec-bde1-339899ab6139-0，_NUM_ADDED_FILES 为 1，_NUM_DELETED_FILES 为 19
    - manifest-7fd62edf-7d04-46ec-bde1-339899ab6139-0 中将 19 个 Data Files 标记为删除（"_KIND":1），并将 dt=20230502 中的唯一一个 Data File 的 LEVEL 从 0 升到 5
- Data File 中数据文件没有变化
#### 2.6 清理过期 Snapshot
清理过期 Snapshot 时才会真正删除被标记为删除的数据文件