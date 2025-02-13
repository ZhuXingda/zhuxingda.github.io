---
title: 【Flink 基础】Flink Kubernetes Application Mode 启动分析
date: 2024-05-21
tags:
    - Flink
    - Kubernetes
categories:
    - "大数据"
---
分析 Flink 在 Kubernetes Application Mode 下启动任务的过程 
<!--more-->
## Flink K8s Application Mode 启动代码调用栈
#### Console Client 启动流程
| sec | 代码 | 作用 |
| --- | --- | --- |
| 1 | org.apache.flink.client.cli.CliFrontend#runApplication |  |
| 2 | org.apache.flink.client.deployment.application.cli.ApplicationClusterDeployer#run |  |
| 2.1 | org.apache.flink.client.deployment.DefaultClusterClientServiceLoader#getClusterClientFactory |  |
| 2.2 | org.apache.flink.client.deployment.AbstractContainerizedClusterClientFactory#getClusterSpecification |  |
| 3 | org.apache.flink.client.deployment.ClusterDescriptor#deployApplicationCluster  | |
| 3.1 | org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployApplicationCluster | 启动 Kubernetes deployment |
| 3.1.1 | org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployClusterInternal | 构建 Jobmanager 的 Container 、Pod Template 和 Deployment，然后下发给 Kubernetes 集群执行 |
| 3.1.1.1 | org.apache.flink.kubernetes.kubeclient.factory.KubernetesJobManagerFactory#buildKubernetesJobManagerSpecification |  |
| 3.1.1.1.1 | org.apache.flink.kubernetes.kubeclient.decorators.CmdJobManagerDecorator#decorateFlinkPod | 设置 Pod 的启动命令 kubernetes-jobmanager.sh kubernetes-application [...kubernetes.jobmanager.entrypoint.args]。在启动 JobManager 的过程中有在配置里设置 kubernetes.internal.jobmanager.entrypoint.class 参数为 org.apache.flink.kubernetes.entrypoint.KubernetesApplicationClusterEntrypoint，但在装饰 JobManager 的 Pod 时并没有用到这项配置。

#### Flink Docker Image 启动流程
1. kubernetes-jobmanager.sh 脚本在 flink-dist/src/main/flink-bin/kubernetes-bin/ 目录下，按照前文 Pod Template 的构建，Flink Docker Image 启动 Container 时执行该脚本。
2. kubernetes-jobmanager.sh 进一步调用 ${FLINK_BIN_DIR}/flink-console.sh kubernetes-application [...args]，在该脚本中，CLASS_TO_RUN 变量被设置为 org.apache.flink.kubernetes.entrypoint.KubernetesApplicationClusterEntrypoint，然后以该类为入口正式启动 JobManager。这里解释了为什么前面设置了 kubernetes.internal.jobmanager.entrypoint.class 参数却没有用到。

#### KubernetesApplicationClusterEntrypoint 启动流程
| sec | 代码 | 作用 |
| --- | --- | --- |
| 1 | org.apache.flink.kubernetes.entrypoint.KubernetesApplicationClusterEntrypoint | |
| 2 | org.apache.flink.runtime.entrypoint.ClusterEntrypoint#runClusterEntrypoint | |
| 3 | org.apache.flink.runtime.entrypoint.ClusterEntrypoint#startCluster | |
| 4 | org.apache.flink.runtime.entrypoint.ClusterEntrypoint#runCluster | |
| 5 | org.apache.flink.runtime.entrypoint.component.DefaultDispatcherResourceManagerComponentFactory#create | |
| 6 | org.apache.flink.runtime.dispatcher.runner.DefaultDispatcherRunnerFactory#createDispatcherRunner | |
| 7 | org.apache.flink.client.deployment.application.ApplicationDispatcherLeaderProcessFactoryFactory#createFactory | |
| 8 | org.apache.flink.client.deployment.application.ApplicationDispatcherGatewayServiceFactory#ApplicationDispatcherGatewayServiceFactory | |
| 9 | org.apache.flink.client.deployment.application.ApplicationDispatcherGatewayServiceFactory#create | |
| 10 | org.apache.flink.client.deployment.application.ApplicationDispatcherBootstrap#ApplicationDispatcherBootstrap | |
| 11 | org.apache.flink.client.deployment.application.ApplicationDispatcherBootstrap#runApplicationEntryPoint | |
| 12 | org.apache.flink.client.ClientUtils#executeProgram | |