---
title: "【Flink】Flink Watermark 产生和传递原理"
date: "2025-03-25T19:27:37+08:00"
categories:
  - "分布式系统"
tags:
  - "Flink"
---
Flink Watermark 的产生原理和传递过程
<!--more-->
1. BoundedOutOfOrdernessWatermarks#onPeriodicEmit
2. WatermarkToDataOutput#emitWatermark
3. PushingAsyncDataInput.DataOutput#emitWatermark
4. Input#processWatermark
5. AbstractStreamOperator#processWatermark
6. Output#emitWatermark
7. RecordWriterOutput#emitWatermark
