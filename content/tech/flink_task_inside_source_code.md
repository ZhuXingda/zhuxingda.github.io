---
title: "【Flink】Flink Task 和 OperatorChain 源码"
description: ""
date: "2025-03-23T12:26:28+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "Flink"
---
Flink Task 的部署过程   
<!--more-->
Flink 版本：1.18.0
### TaskManager 启动
1. **KubernetesTaskExecutorRunner** 或 **YarnTaskExecutorRunner** main 函数启动   
调用 TaskManagerRunner#runTaskManager 创建和启动 TaskExecutor
2. 创建 **TaskExecutor** 包含 
    - TaskManagerServices 
    - RpcService 
    - HighAvailabilityServices 
    - HeartbeatServices 
    - TaskManagerMetricGroup 
    - TaskExecutorBlobService 等   
TaskExecutor 运行时会用到的组件，其中 TaskManagerServices 还包含了 
    - ShuffleEnvironment 
    - KvStateService 
    - TaskExecutorLocalStateStoresManager 
    - TaskExecutorStateChangelogStoragesManager 
    - TaskExecutorChannelStateExecutorFactoryManager   
等组件
3. **TaskExecutor** 继承了 **RpcEndpoint**，初始化和启动时一并创建和启动了 RpcServer
    - **RpcEndpoint#RpcEndpoint** 创建 RpcEndpoint
    - **PekkoRpcService#startServer** 启动 RpcServer
    - **PekkoRpcService#registerRpcActor** 注册 RpcActor   
    - **PekkoRpcActor#handleControlMessage** RpcActor 会接收到启动的远程调用
    - **PekkoRpcActor.StoppedState#start** RpcActor 启动
    - **RpcEndpoint#internalCallOnStart** RpcEndpoint 启动
    - **TaskExecutor#onStart** TaskExecutor 启动
### JobMaster 创建和启动
1. **StreamContextEnvironment#execute** 开始执行 StreamGraph
2. **PipelineExecutor#execute** StreamGraph
3. **ClusterClient#submitJob** 不同类型 ClusterClient 提交 JobGraph，RestClusterClient 会将 JobGraph 文件及 user jars、user artifacts 等文件通过 HTTP 接口上传到 JobManager 的 Job Submit 接口
4. **JobSubmitHandler#handleRequest** JobManager 的 HTTP 接口接收请求
5. **Dispatcher#submitJob** 
6. **Dispatcher#persistAndRunJob** Dispatcher 将 JobGraph 持久化后开始执行
7. **JobManagerRunner#start** (JobMasterServiceLeadershipRunner#start) JobMasterServiceLeadershipRunner 继承了 JobManagerRunner 和 LeaderContender， 调用 Leader Election 并获得 Leader 资格后 grantLeadership 接口会被执行
8. **DefaultLeaderElection#startLeaderElection** 选举 Leader 这部分暂时跳过不深入
9. **JobMasterServiceLeadershipRunner#grantLeadership**
10. **JobMasterServiceLeadershipRunner#startJobMasterServiceProcessAsync**
11. **JobMasterServiceLeadershipRunner#verifyJobSchedulingStatusAndCreateJobMasterServiceProcess**
12. **JobMasterServiceLeadershipRunner#createNewJobMasterServiceProcess** 创建 JobMasterServiceProcess 并对运行结果做处理
13. **DefaultJobMasterServiceProcessFactory#create** 创建 JobMasterServiceProcess
14. **DefaultJobMasterServiceFactory#internalCreateJobMasterService** 创建 JobMaster 并启动 RpcServer
### JobMaster 调度任务执行
1. **JobMaster#onStart** JobMaster 也继承 RpcEndpoint，在 RpcServer 启动后 onStart 接口被调用
2. **JobMaster#startScheduling** 启动任务调度
3. **SchedulerBase#startScheduling** 开始调度
4. **OperatorCoordinatorHandler#startAllOperatorCoordinators**
5. **SchedulingStrategy#startScheduling** 不同的调度逻辑通过这个接口来实现部署 Vertex 到对应的 Slot
6. **DefaultScheduler#allocateSlotsAndDeploy** 将 Vertex 转换成 Execution 并部署 到 Slot
7. **DefaultExecutionDeployer#allocateSlotsAndDeploy**
    - validateExecutionStates 校验所有 Execution 都是 CREATED 状态
    - transitionToScheduled 将所有 Execution 转换成 SCHEDULED 状态
    - allocateSlotsFor 给每一个 Execution 分配对应的 Slot
        1. ExecutionSlotAllocator#allocateSlotsFor
        2. PhysicalSlotProvider#allocatePhysicalSlots
        3. PhysicalSlotProviderImpl#tryAllocateFromAvailable
    - createDeploymentHandles 为每一个 Execution 创建 ExecutionDeploymentHandle
    - waitForAllSlotsAndDeploy 开始部署各个 Execution
        1. DefaultExecutionDeployer#assignResource
        2. Execution#tryAssignResource
8. **Execution#deploy** 运行 Execution 的 deploy 流程
9. **RpcTaskManagerGateway#submitTask** 执行 RPC 远程调用，将 TaskDeploymentDescriptor 传递给对应的 TaskExecutor 执行
### Task 执行
1. **TaskExecutor#submitTask** 创建 Task 并在专门的线程执行
2. **Task#doRun** Task 执行
    - **Task#loadAndInstantiateInvokable** 初始化 Task 实际执行的 TaskInvokable 对象，取决于 TaskInformation 里的 invokableClassName，即 JobVertex 的 name 对应的 Class
3. **Task#restoreAndInvoke**
4. **TaskInvokable#invoke** 执行 JobVertex 对应的实际任务，TaskInvokable 接口有多个实现，包含 BatchTask、StreamTask、DataSinkTask、DataSourceTask 等
5. **StreamTask#invoke** 这里以 StreamTask 为例往下分析
6. **MailboxProcessor#runMailboxLoop** 运行 Mailbox 循环，实际是循环执行 StreamTask#processInput
7. **StreamTask#processInput** 调用 StreamInputProcessor 的  processInput ， 并对返回的 status 做处理
8. **StreamInputProcessor#processinput** 不同的 StreamTask 实现类使用了对应的 StreamInputProcessor
9. **StreamOneInputProcessor#processInput** 这里以 StreamOneInputProcessor 为例，其作用是将 StreamTaskInput 传入的数据直接推到 DataOutput
### StreamTask 初始化
1. **StreamTask#invoke** StreamTask 在运行 Mailbox 循环前会先执行 restoreInternal 实现初始化
2. **StreamTask#restoreInternal** 
    1. **OperatorChain#OperatorChain** 创建 OperatorChain
    2. **StreamTask#init** 由各接口实现具体的初始化逻辑，这里以 OneInputStreamTask 为例
    3. **OneInputStreamTask#init** 主要是初始化 StreamInputProcessor，OneInputStreamTask 对应 StreamOneInputProcessor
        - **OneInputStreamTask#createDataOutput** 创建 DataOutput，在 OneInputStreamTask 的实现是 OneInputStreamTask.StreamTaskNetworkOutput，其内部用 Counter 计数每个传入的 StreamRecord，然后转交给 recordProcessor 处理，这里 recordProcessor 实际主要是把 record 推给了创建 StreamTaskNetworkOutput 时传入的 Input，即 StreamTask 的 mainOperator，亦即
        - **OneInputStreamTask#createTaskInput** 创建 StreamTaskNetworkInput，每一次 StreamTask.processInput 运行时都会调用 StreamTaskNetworkInput 从 inputGate 读取数据 buffer，并交由 RecordDeserializer 解析，解析后的 StreamRecord 推送到 DataOutput#emitRecord
### OperatorChain 初始化
```java
    public OperatorChain(
            StreamTask<OUT, OP> containingTask,
            RecordWriterDelegate<SerializationDelegate<StreamRecord<OUT>>> recordWriterDelegate) {
        // 与 OperatorCoordinator 交互 OperatorEvents
        this.operatorEventDispatcher =
                new OperatorEventDispatcherImpl(
                        containingTask.getEnvironment().getUserCodeClassLoader().asClassLoader(),
                        containingTask.getEnvironment().getOperatorCoordinatorEventGateway());

        final ClassLoader userCodeClassloader = containingTask.getUserCodeClassLoader();
        final StreamConfig configuration = containingTask.getConfiguration();
        // 用来创建 StreamOperator
        StreamOperatorFactory<OUT> operatorFactory =
                configuration.getStreamOperatorFactory(userCodeClassloader);

        // 创建 OperatorChain 里各 StreamOperator 所用到的 StreamConfig
        Map<Integer, StreamConfig> chainedConfigs =
                configuration.getTransitiveChainedTaskConfigsWithSelf(userCodeClassloader);

        // OperatorChain 对外输出的 StreamOperator 对应的信息
        List<NonChainedOutput> outputsInOrder =
                configuration.getVertexNonChainedOutputs(userCodeClassloader);
        Map<IntermediateDataSetID, RecordWriterOutput<?>> recordWriterOutputs =
                CollectionUtil.newHashMapWithExpectedSize(outputsInOrder.size());
        // OperatorChain 对外输出的 Outputs，为 RecordWriterOutput 类型
        this.streamOutputs = new RecordWriterOutput<?>[outputsInOrder.size()];
        this.finishedOnRestoreInput =
                this.isTaskDeployedAsFinished()
                        ? new FinishedOnRestoreInput(
                                streamOutputs, configuration.getInputs(userCodeClassloader).length)
                        : null;

        boolean success = false;
        try {
            createChainOutputs(
                    outputsInOrder,
                    recordWriterDelegate,
                    chainedConfigs,
                    containingTask,
                    recordWriterOutputs);

            // we create the chain of operators and grab the collector that leads into the chain
            List<StreamOperatorWrapper<?, ?>> allOpWrappers =
                    new ArrayList<>(chainedConfigs.size());
            this.mainOperatorOutput =
                    createOutputCollector(
                            containingTask,
                            configuration,
                            chainedConfigs,
                            userCodeClassloader,
                            recordWriterOutputs,
                            allOpWrappers,
                            containingTask.getMailboxExecutorFactory(),
                            operatorFactory != null);

            if (operatorFactory != null) {
                Tuple2<OP, Optional<ProcessingTimeService>> mainOperatorAndTimeService =
                        StreamOperatorFactoryUtil.createOperator(
                                operatorFactory,
                                containingTask,
                                configuration,
                                mainOperatorOutput,
                                operatorEventDispatcher);

                OP mainOperator = mainOperatorAndTimeService.f0;
                mainOperator
                        .getMetricGroup()
                        .gauge(
                                MetricNames.IO_CURRENT_OUTPUT_WATERMARK,
                                mainOperatorOutput.getWatermarkGauge());
                this.mainOperatorWrapper =
                        createOperatorWrapper(
                                mainOperator,
                                containingTask,
                                configuration,
                                mainOperatorAndTimeService.f1,
                                true);

                // add main operator to end of chain
                allOpWrappers.add(mainOperatorWrapper);

                this.tailOperatorWrapper = allOpWrappers.get(0);
            } else {
                checkState(allOpWrappers.size() == 0);
                this.mainOperatorWrapper = null;
                this.tailOperatorWrapper = null;
            }

            this.chainedSources =
                    createChainedSources(
                            containingTask,
                            configuration.getInputs(userCodeClassloader),
                            chainedConfigs,
                            userCodeClassloader,
                            allOpWrappers);

            this.numOperators = allOpWrappers.size();

            firstOperatorWrapper = linkOperatorWrappers(allOpWrappers);

            success = true;
        } finally {
            // 如果创建失败清理资源
            if (!success) {
                for (int i = 0; i < streamOutputs.length; i++) {
                    if (streamOutputs[i] != null) {
                        streamOutputs[i].close();
                    }
                    streamOutputs[i] = null;
                }
            }
        }
    }
```
```java
    private void createChainOutputs(
            List<NonChainedOutput> outputsInOrder,
            RecordWriterDelegate<SerializationDelegate<StreamRecord<OUT>>> recordWriterDelegate,
            Map<Integer, StreamConfig> chainedConfigs,
            StreamTask<OUT, OP> containingTask,
            Map<IntermediateDataSetID, RecordWriterOutput<?>> recordWriterOutputs) {
        for (int i = 0; i < outputsInOrder.size(); ++i) {
            NonChainedOutput output = outputsInOrder.get(i);

            RecordWriterOutput<?> recordWriterOutput =
                    createStreamOutput(
                            recordWriterDelegate.getRecordWriter(i),
                            output,
                            chainedConfigs.get(output.getSourceNodeId()),
                            containingTask.getEnvironment());

            this.streamOutputs[i] = recordWriterOutput;
            recordWriterOutputs.put(output.getDataSetId(), recordWriterOutput);
        }
    }

    private RecordWriterOutput<OUT> createStreamOutput(
            RecordWriter<SerializationDelegate<StreamRecord<OUT>>> recordWriter,
            NonChainedOutput streamOutput,
            StreamConfig upStreamConfig,
            Environment taskEnvironment) {
        OutputTag sideOutputTag =
                streamOutput.getOutputTag(); // OutputTag, return null if not sideOutput

        TypeSerializer outSerializer;

        if (streamOutput.getOutputTag() != null) {
            // side output
            outSerializer =
                    upStreamConfig.getTypeSerializerSideOut(
                            streamOutput.getOutputTag(),
                            taskEnvironment.getUserCodeClassLoader().asClassLoader());
        } else {
            // main output
            outSerializer =
                    upStreamConfig.getTypeSerializerOut(
                            taskEnvironment.getUserCodeClassLoader().asClassLoader());
        }

        return closer.register(
                new RecordWriterOutput<OUT>(
                        recordWriter,
                        outSerializer,
                        sideOutputTag,
                        streamOutput.supportsUnalignedCheckpoints()));
    }
```
```java
    private Map<StreamConfig.SourceInputConfig, ChainedSource> createChainedSources(
            StreamTask<OUT, OP> containingTask,
            StreamConfig.InputConfig[] configuredInputs,
            Map<Integer, StreamConfig> chainedConfigs,
            ClassLoader userCodeClassloader,
            List<StreamOperatorWrapper<?, ?>> allOpWrappers) {
        if (Arrays.stream(configuredInputs)
                .noneMatch(input -> input instanceof StreamConfig.SourceInputConfig)) {
            return Collections.emptyMap();
        }
        checkState(
                mainOperatorWrapper.getStreamOperator() instanceof MultipleInputStreamOperator,
                "Creating chained input is only supported with MultipleInputStreamOperator and MultipleInputStreamTask");
        Map<StreamConfig.SourceInputConfig, ChainedSource> chainedSourceInputs = new HashMap<>();
        MultipleInputStreamOperator<?> multipleInputOperator =
                (MultipleInputStreamOperator<?>) mainOperatorWrapper.getStreamOperator();
        List<Input> operatorInputs = multipleInputOperator.getInputs();

        int sourceInputGateIndex =
                Arrays.stream(containingTask.getEnvironment().getAllInputGates())
                                .mapToInt(IndexedInputGate::getInputGateIndex)
                                .max()
                                .orElse(-1)
                        + 1;

        for (int inputId = 0; inputId < configuredInputs.length; inputId++) {
            if (!(configuredInputs[inputId] instanceof StreamConfig.SourceInputConfig)) {
                continue;
            }
            StreamConfig.SourceInputConfig sourceInput =
                    (StreamConfig.SourceInputConfig) configuredInputs[inputId];
            int sourceEdgeId = sourceInput.getInputEdge().getSourceId();
            StreamConfig sourceInputConfig = chainedConfigs.get(sourceEdgeId);
            OutputTag outputTag = sourceInput.getInputEdge().getOutputTag();

            WatermarkGaugeExposingOutput chainedSourceOutput =
                    createChainedSourceOutput(
                            containingTask,
                            sourceInputConfig,
                            userCodeClassloader,
                            getFinishedOnRestoreInputOrDefault(operatorInputs.get(inputId)),
                            multipleInputOperator.getMetricGroup(),
                            outputTag);

            SourceOperator<?, ?> sourceOperator =
                    (SourceOperator<?, ?>)
                            createOperator(
                                    containingTask,
                                    sourceInputConfig,
                                    userCodeClassloader,
                                    (WatermarkGaugeExposingOutput<StreamRecord<OUT>>)
                                            chainedSourceOutput,
                                    allOpWrappers,
                                    true);
            chainedSourceInputs.put(
                    sourceInput,
                    new ChainedSource(
                            chainedSourceOutput,
                            this.isTaskDeployedAsFinished()
                                    ? new StreamTaskFinishedOnRestoreSourceInput<>(
                                            sourceOperator, sourceInputGateIndex++, inputId)
                                    : new StreamTaskSourceInput<>(
                                            sourceOperator, sourceInputGateIndex++, inputId)));
        }
        return chainedSourceInputs;
    }
```
```java
    private <IN, OUT> WatermarkGaugeExposingOutput<StreamRecord<IN>> createOperatorChain(
            StreamTask<OUT, ?> containingTask,
            StreamConfig prevOperatorConfig,
            StreamConfig operatorConfig,
            Map<Integer, StreamConfig> chainedConfigs,
            ClassLoader userCodeClassloader,
            Map<IntermediateDataSetID, RecordWriterOutput<?>> recordWriterOutputs,
            List<StreamOperatorWrapper<?, ?>> allOperatorWrappers,
            OutputTag<IN> outputTag,
            MailboxExecutorFactory mailboxExecutorFactory,
            boolean shouldAddMetricForPrevOperator) {
        // create the output that the operator writes to first. this may recursively create more
        // operators
        WatermarkGaugeExposingOutput<StreamRecord<OUT>> chainedOperatorOutput =
                createOutputCollector(
                        containingTask,
                        operatorConfig,
                        chainedConfigs,
                        userCodeClassloader,
                        recordWriterOutputs,
                        allOperatorWrappers,
                        mailboxExecutorFactory,
                        true);

        OneInputStreamOperator<IN, OUT> chainedOperator =
                createOperator(
                        containingTask,
                        operatorConfig,
                        userCodeClassloader,
                        chainedOperatorOutput,
                        allOperatorWrappers,
                        false);

        return wrapOperatorIntoOutput(
                chainedOperator,
                containingTask,
                prevOperatorConfig,
                operatorConfig,
                userCodeClassloader,
                outputTag,
                shouldAddMetricForPrevOperator);
    }
```
```java
    private <OUT, OP extends StreamOperator<OUT>> OP createOperator(
            StreamTask<OUT, ?> containingTask,
            StreamConfig operatorConfig,
            ClassLoader userCodeClassloader,
            WatermarkGaugeExposingOutput<StreamRecord<OUT>> output,
            List<StreamOperatorWrapper<?, ?>> allOperatorWrappers,
            boolean isHead) {

        // now create the operator and give it the output collector to write its output to
        Tuple2<OP, Optional<ProcessingTimeService>> chainedOperatorAndTimeService =
                StreamOperatorFactoryUtil.createOperator(
                        operatorConfig.getStreamOperatorFactory(userCodeClassloader),
                        containingTask,
                        operatorConfig,
                        output,
                        operatorEventDispatcher);

        OP chainedOperator = chainedOperatorAndTimeService.f0;
        allOperatorWrappers.add(
                createOperatorWrapper(
                        chainedOperator,
                        containingTask,
                        operatorConfig,
                        chainedOperatorAndTimeService.f1,
                        isHead));

        chainedOperator
                .getMetricGroup()
                .gauge(
                        MetricNames.IO_CURRENT_OUTPUT_WATERMARK,
                        output.getWatermarkGauge()::getValue);
        return chainedOperator;
    }
```
```java
    private <IN, OUT> WatermarkGaugeExposingOutput<StreamRecord<IN>> wrapOperatorIntoOutput(
            OneInputStreamOperator<IN, OUT> operator,
            StreamTask<OUT, ?> containingTask,
            StreamConfig prevOperatorConfig,
            StreamConfig operatorConfig,
            ClassLoader userCodeClassloader,
            OutputTag<IN> outputTag,
            boolean shouldAddMetricForPrevOperator) {

        Counter recordsOutCounter = null;

        if (shouldAddMetricForPrevOperator) {
            recordsOutCounter = getOperatorRecordsOutCounter(containingTask, prevOperatorConfig);
        }

        WatermarkGaugeExposingOutput<StreamRecord<IN>> currentOperatorOutput;
        if (containingTask.getExecutionConfig().isObjectReuseEnabled()) {
            currentOperatorOutput =
                    new ChainingOutput<>(
                            operator, recordsOutCounter, operator.getMetricGroup(), outputTag);
        } else {
            TypeSerializer<IN> inSerializer =
                    operatorConfig.getTypeSerializerIn1(userCodeClassloader);
            currentOperatorOutput =
                    new CopyingChainingOutput<>(
                            operator,
                            inSerializer,
                            recordsOutCounter,
                            operator.getMetricGroup(),
                            outputTag);
        }

        // wrap watermark gauges since registered metrics must be unique
        operator.getMetricGroup()
                .gauge(
                        MetricNames.IO_CURRENT_INPUT_WATERMARK,
                        currentOperatorOutput.getWatermarkGauge()::getValue);

        return closer.register(currentOperatorOutput);
    }
```
```java
    private StreamOperatorWrapper<?, ?> linkOperatorWrappers(
            List<StreamOperatorWrapper<?, ?>> allOperatorWrappers) {
        StreamOperatorWrapper<?, ?> previous = null;
        for (StreamOperatorWrapper<?, ?> current : allOperatorWrappers) {
            if (previous != null) {
                previous.setPrevious(current);
            }
            current.setNext(previous);
            previous = current;
        }
        return previous;
    }
```


## 参考资料
https://github.com/mickey0524/flink-streaming-source-analysis/blob/master/docs/flink-operator-chain.md
https://blog.jrwang.me/2019/flink-source-code-task-lifecycle/#oneinputstreamtask
