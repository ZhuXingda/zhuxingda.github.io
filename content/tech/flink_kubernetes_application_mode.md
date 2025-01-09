---
title: Flink Kubernetes Application Mode 部署分析
date: 2024-11-28
tags:
    - Flink
    - Kubernetes
categories:
    - "Development"
draft: true
---
分析 Flink 在 Kubernetes Application Mode 下启动任务的过程 
<!--more-->
## Flink K8s Application Mode 启动代码调用栈
1. org.apache.flink.client.cli.CliFrontend#runApplication
2. org.apache.flink.client.deployment.application.cli.ApplicationClusterDeployer#run
    2.1.  org.apache.flink.client.deployment.DefaultClusterClientServiceLoader#getClusterClientFactory
    2.2. org.apache.flink.client.deployment.AbstractContainerizedClusterClientFactory#getClusterSpecification
3. org.apache.flink.client.deployment.ClusterDescriptor#deployApplicationCluster
    - org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployApplicationCluster （启动 Kubernetes deployment）
        3.1. org.apache.flink.kubernetes.KubernetesClusterDescriptor#deployClusterInternal
            3.2.1 org.apache.flink.kubernetes.kubeclient.factory.KubernetesJobManagerFactory#buildKubernetesJobManagerSpecification
            3.2.2 org.apache.flink.kubernetes.kubeclient.factory.KubernetesJobManagerFactory#createJobManagerDeployment