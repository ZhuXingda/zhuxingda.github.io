---
title: 【Flink】Flink Kubernetes Application Mode 启动分析
date: 2024-05-21
tags:
    - Flink
    - Kubernetes
categories:
    - "分布式系统"
---
梳理 Flink 在 Kubernetes Application Mode 下启动任务的过程，分析为什么 Flink SQL 任务不能以 Application Mode 运行
<!--more-->
## Flink K8s Application Mode 启动代码调用栈
#### Console Client 启动流程
1. org.apache.flink.client.cli.CliFrontend#runApplication   
2. org.apache.flink.client.deployment.application.cli.ApplicationClusterDeployer#run    
    2.1 org.apache.flink.client.deployment.DefaultClusterClientServiceLoader#getClusterClientFactory    
    2.2 org.apache.flink.client.deployment.AbstractContainerizedClusterClientFactory#getClusterSpecification    
3. org.apache.flink.client.deployment.ClusterDescriptor#deployApplicationCluster    
    3.1 org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployApplicationCluster 启动 Kubernetes deployment     
        3.1.1 org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployClusterInternal 
        构建 Jobmanager 的 Container 、Pod Template 和 Deployment，然后下发给 Kubernetes 集群执行部署 
            3.1.1.1 org.apache.flink.kubernetes.kubeclient.factory.KubernetesJobManagerFactory#buildKubernetesJobManagerSpecification   
                3.1.1.1.1 org.apache.flink.kubernetes.kubeclient.decorators.CmdJobManagerDecorator#decorateFlinkPod 
                设置 Pod 的启动命令kubernetes-jobmanager.sh kubernetes-application [...kubernetes.jobmanager.entrypoint.args]  

一些发现：
1. 在启动 JobManager 的过程中有在配置里设置 kubernetes.internal.jobmanager.entrypoint.class 配置项为 org.apache.flink.kubernetes.entrypoint.KubernetesApplicationClusterEntrypoint，但在装饰 JobManager 的 Pod 时并没有用到这项配置。
2. 在装饰 JobManager 和 TaskManager 时 FlinkConfMountDecorator 会将在提交任务的客户端查找 `$internal.deployment.config-dir` 或 `kubernetes.flink.conf.dir` 两个配置指定的目录，如果目录下有 **logback-console.xml** 或 **log4j-console.properties** 配置文件，则会在创建 Pod 的 Volumn 时挂载镜像内的同名文件到 /opt/flink/conf 目录下，作为程序运行时日志输出的配置。

#### Flink Pod 启动脚本
1. kubernetes-jobmanager.sh 脚本在 flink-dist/src/main/flink-bin/kubernetes-bin/ 目录下，按照前文 Pod Template 的构建，Flink Pod 启动时执行该脚本。
2. kubernetes-jobmanager.sh 进一步调用 ${FLINK_BIN_DIR}/flink-console.sh kubernetes-application [...args]，在该脚本中，CLASS_TO_RUN 变量被设置为 org.apache.flink.kubernetes.entrypoint.KubernetesApplicationClusterEntrypoint，然后以该类为入口正式启动 JobManager。

#### 容器内 Pod 启动流程
1. KubernetesApplicationClusterEntrypoint#main    
    1.1 KubernetesApplicationClusterEntrypoint#getPackagedProgram 获取用户定义任务程序包    
    1.2 KubernetesApplicationClusterEntrypoint#getPackagedProgramRetriever
2. ClusterEntrypoint#runClusterEntrypoint   
3. ClusterEntrypoint#startCluster   启动集群
4. ClusterEntrypoint#runCluster 
    4.1 ClusterEntrypoint#initializeServices    
    创建 workingDirectory rpcService ioExecutor blobServer metricRegistry executionGraphInfoStore 等集群运行需要的的服务和资源 
    4.2 ClusterEntrypoint#createDispatcherResourceManagerComponentFactory   
    创建 DispatcherResourceManagerComponentFactory 实例，该 abstract 方法根据部署方式不同有多种实现
    4.3 ApplicationClusterEntryPoint#createDispatcherResourceManagerComponentFactory    
    这里 DefaultDispatcherResourceManagerComponentFactory 包含了一个 DefaultDispatcherRunnerFactory，又包含了一个 ApplicationDispatcherLeaderProcessFactoryFactory  
    ```java
    @Override
    protected DispatcherResourceManagerComponentFactory
            createDispatcherResourceManagerComponentFactory(final Configuration configuration) {
        return new DefaultDispatcherResourceManagerComponentFactory(
                new DefaultDispatcherRunnerFactory(
                        ApplicationDispatcherLeaderProcessFactoryFactory.create(
                                configuration, SessionDispatcherFactory.INSTANCE, program)),
                resourceManagerFactory,
                JobRestEndpointFactory.INSTANCE);
    }
    ```
5. DefaultDispatcherResourceManagerComponentFactory#create  
启动 webMonitor resourceManagerService dispatcherRunner
6. DefaultDispatcherRunnerFactory#createDispatcherRunner 
    6.1 ApplicationDispatcherLeaderProcessFactoryFactory#createFactory    
        创建 DispatcherLeaderProcess 的工厂类，这里创建的是 SessionDispatcherLeaderProcessFactory
        6.1.1 ApplicationDispatcherGatewayServiceFactory#ApplicationDispatcherGatewayServiceFactory 
    6.1 DefaultDispatcherRunner#create  
    创建 DispatcherRunner 实例
    6.2 DefaultDispatcherRunner#start   
    启动 DispatcherRunner 实例，开始 Leader 选举，在 Leader 选举结束后 Leader 的 DefaultDispatcherRunner#grantLeadership 方法会被调用
    6.3 DefaultDispatcherRunner#grantLeadership
    6.4 DefaultDispatcherRunner#startNewDispatcherLeaderProcess 
    创建并启动 DispatcherLeaderProcess
    6.5 DefaultDispatcherRunner#createNewDispatcherLeaderProcess
    6.6 DispatcherLeaderProcessFactory#create
    6.7 SessionDispatcherLeaderProcessFactory#create    
    这里创建的是 SessionDispatcherLeaderProcess 实例
    6.8 JobPersistenceComponentFactory#createJobGraphStore
    创建 JobGraphStore，用于从异常中恢复 Job 的运行
    6.9 HaServicesJobPersistenceComponentFactory#createJobGraphStore
    6.10 HighAvailabilityServices#getJobGraphStore  
    如果集群配置了高可用 Service，比如 KubernetesLeaderElectionHaServices 和 ZooKeeperLeaderElectionHaServices，则从高可用服务中获取 JobGraphStore，否则返回一个无用的 StandaloneJobGraphStore
7. DispatcherLeaderProcess#start
8. SessionDispatcherLeaderProcess#onStart
    8.1 SessionDispatcherLeaderProcess#startServices
        8.1.1 JobGraphStore#start   
        启动 JobGraphStore
    8.2 SessionDispatcherLeaderProcess#createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults
    ```java
    private CompletableFuture<Void>
            createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults() {
        // 获取执行失败的 Jobs 的 JobResults，如果是初次执行这里应该是没有的
        final CompletableFuture<Collection<JobResult>> dirtyJobsFuture =
                CompletableFuture.supplyAsync(this::getDirtyJobResultsIfRunning, ioExecutor);

        return dirtyJobsFuture
                .thenApplyAsync(
                        dirtyJobs ->
                                // 恢复失败 Jobs 的 JobGraphs
                                this.recoverJobsIfRunning(
                                        dirtyJobs.stream()
                                                .map(JobResult::getJobId)
                                                .collect(Collectors.toSet())),
                        ioExecutor)
                // 用失败 Jobs 的 JobResults 和 JobGraphs 创建 Dispatcher，初次执行的话两个都为空
                .thenAcceptBoth(dirtyJobsFuture, this::createDispatcherIfRunning)
                .handle(this::onErrorIfRunning);
    }
    ```
    8.3 SessionDispatcherLeaderProcess#createDispatcher
    8.4 DispatcherGatewayServiceFactory#create
    8.5 ApplicationDispatcherGatewayServiceFactory#create    
    创建 Dispatcher 实例，其中包含了一个 ApplicationDispatcherBootstrap 用于执行初次执行的 Job，或者恢复失败的 Job，    
    Dispatcher 被包装为 DispatcherGatewayService 返回
    8.6 RpcEndpoint#start   
    启动 Dispatcher

#### 执行用户定义的 main() 方法 
1. ApplicationDispatcherBootstrap#ApplicationDispatcherBootstrap    
2. ApplicationDispatcherBootstrap#runApplicationEntryPoint  
3. ClientUtils#executeProgram  
    切换线程的 ClassLoader 为 UserCode ClassLoader，执行用户定义的 main() 方法  
    ```java
    public static void executeProgram(
            PipelineExecutorServiceLoader executorServiceLoader,
            Configuration configuration,
            PackagedProgram program,
            boolean enforceSingleJobExecution,
            boolean suppressSysout)
            throws ProgramInvocationException {
        checkNotNull(executorServiceLoader);
        final ClassLoader userCodeClassLoader = program.getUserCodeClassLoader();
        final ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
        try {
            Thread.currentThread().setContextClassLoader(userCodeClassLoader);

            LOG.info(
                    "Starting program (detached: {})",
                    !configuration.getBoolean(DeploymentOptions.ATTACHED));
            // 设置 ContextEnvironment 和 StreamContextEnvironment，在执行用户 main() 方法构建完 Pipeline 时，
            // 这里传入的 EmbeddedExecutorServiceLoader 会被用来创建 PipelineExecutor，Pipeline 会被转换成 JobGraph
            // 然后提交到前面启动的 DispatcherGateway
            ContextEnvironment.setAsContext(
                    executorServiceLoader,
                    configuration,
                    userCodeClassLoader,
                    enforceSingleJobExecution,
                    suppressSysout);

            StreamContextEnvironment.setAsContext(
                    executorServiceLoader,
                    configuration,
                    userCodeClassLoader,
                    enforceSingleJobExecution,
                    suppressSysout);

            try {
                program.invokeInteractiveModeForExecution();
            } finally {
                ContextEnvironment.unsetAsContext();
                StreamContextEnvironment.unsetAsContext();
            }
        } finally {
            Thread.currentThread().setContextClassLoader(contextClassLoader);
        }
    }
    ```
4. PackagedProgram#invokeInteractiveModeForExecution   
    异步反射调用 main() 方法    

#### 总结
综上可以发现 Application Mode 启动过程中会在 JobManager 上执行用户开发的 Jar 包，解析成 JobGraph 后提交到自己的 DispatcherGateway 执行，而 Flink SQL 在 SQL Client 端就已经解析成 JobGraph 而不是 Jar 包，所以以 Application Mode 运行没有必要，如果想要像 Application Mode 一样，一个 Flink Cluster 上只运行一个 SQL 任务，任务结束时就销毁 Cluster，可以将这部分功能放在 SQL Client 来实现，任务结束后给 SQL Client 一个回调，收到回调后销毁 Cluster。