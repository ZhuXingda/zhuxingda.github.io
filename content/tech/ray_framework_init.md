---
title: '【Ray】分布式计算框架 Ray 入门'
date: 2024-09-16
categories:
    - '分布式系统'
tags:
    - 'Ray'
---
**Ray** 是一个开源的分布式计算框架，当前主要被应用于机器学习领域。
<!--more--> 相比于常用的分布式计算框架 Spark 和 Flink，Ray 提供了更为灵活通用的 API，开发者无需遵循一套固定的编程规则（比如 Spark 和 Flink 里的各种算子）就可以将本地运行的 Python 应用部署到分布式环境执行。
## Ray Core
Ray Core 是 Ray 分布式计算的核心组件，逻辑上包含以下概念：
- **Tasks**：在分布式环境下异步执行的无状态函数
- **Actors**：由 class 定义的有状态的 worker，分布式执行时 worker 的状态可以被读取和修改
- **Objects**：tasks 和 actors 执行过程中创建和计算的数据对象，被保存在由 Ray Cluster 维护的分布式对象存储中
- **Placement Groups**：定义一组在集群保留的资源，执行 tasks 或 actors 时使用
## Ray Cluster
#### 集群架构   
![一个简单 Ray 集群的架构图](https://docs.ray.io/en/latest/_images/ray-cluster.svg)
{width="700px"}   
一个 Ray 集群由一个 head node 和多个 worker node 组成。
- **Worker Node**   
负责执行用户代码，支持集群的调度，组成集群的分布式对象存储。
- **Head Node**   
在 woker node 的职责上额外负责集群的管理（可配置仅运行管理进程），包括：
    - Autoscaling：管理资源的进程，当集群需要的资源不足时自动扩大 worker node 规模，worker node idle 时缩减。
    - Global Control Store（GCS）：存储 Ray 集群的 metadata。
    - Driver Process：交互式开发时负责在 Ray Cluster 上启动用户的代码。
#### KubeRay
KubeRay 是在 Kubernetes 上部署和管理 Ray 集群的 Kubernetes Operator，类似 Flink 以前版本提供的 Flink Kubernetes Operator。   
KubeRay 提供了三种部署模式：
- RayCluster：部署和管理一个 Ray 集群，包括销毁、扩缩容、容错，可以多次提交任务，类似 Flink 的 Session Mode。
- RayJob：启动一个 Ray 集群并提交一个任务在该集群运行，任务结束后支持自动销毁该集群，类似 Flink 的 Application Mode。
- RayService：部署 Ray 集群并运行 Ray Serve 应用，支持动态调整 Ray Serve 配置，无损升级 Ray 集群，并提供高可用支持。
#### 用 KubeRay 部署一个 Ray Cluster
因为我的 Kubernetes 在局域网环境没有外网访问，所以需要手动部署   
1. **安装 Helm**   
从 Helm 的 [Github 仓库](https://github.com/helm/helm/releases) 下载编译好的二进制，解压后放在 /usr/local/bin/ 目录下。
2. **安装 KubeRay**   
在有外网的机器上用 docker 从 docker hub 拉取 kuberay-operator 的镜像，并推送到私有仓库
```shell
$ docker image pull quay.io/kuberay/operator:v1.2.2
$ docker tag quay.io/kuberay/operator:v1.2.2 local.repo.com/kuberay/operator:v1.2.2
$ docker push local.repo.com/kuberay/operator:v1.2.2
```
从 [Github仓库](https://github.com/ray-project/kuberay-helm) 下载 kuberay-operator v1.2.2 的压缩包，解压后修改 kuberay-operator/values.yaml
```yaml
image:
  # 修改为私有仓库地址
  repository: local.repo.com/kuberay/operator
  tag: v1.2.2
  pullPolicy: IfNotPresent
  imagePullSecrets:
    - name: localrepo-secret-token
...
service:
  # 修改 port 避免和其他服务冲突
  type: ClusterIP
  port: 9080
```
helm 安装 kuberay-operator   
创建 namespace 和私有仓库的 docker-registry secret
```shell
$ kubectl create namesapce kuberay-operator
$ kubectl create secret docker-registry localrepo-secret-token --docker-server=local.repo.com --docker-username=$USERNAME --docker-password=$PASSWORD -n kuberay-operator
```
helm 打包并安装 kuberay-operator
```shell
$ helm package kuberay-operator
Successfully packaged chart and saved it to: /xxxx/helm-chart/kuberay-operator-1.2.2.tgz
$ helm install kuberay-operator ./kuberay-operator-1.2.2.tgz
```
启动后发现 pod 状态报错
```shell
$ kubectl get pods -o wide -n kuberay-operator
NAME                                READY   STATUS             RESTARTS      AGE    IP             NODE                          NOMINATED NODE   READINESS GATES
kuberay-operator-5b5d75db55-gz8cw   0/1     CrashLoopBackOff   4 (15s ago)   114s   10.100.6.110   host-07   <none>           <none>
$ kubectl logs -f kuberay-operator-5b5d75db55-gz8cw -n kuberay-operator
exec /manager: exec format error
```
检查 kuberay 的代码发现 manager 是 ray-operator 编译后的二进制，"exec format error" 这个报错常见于 golang 的二进制跨平台运行，原因是前面从 docker hub 拉取镜像的机器是 MAC OS，回退到第一步重新拉取镜像，用 crictl 删除掉 kubernetes 集群上的本地镜像，其他操作不变
```shell
$ docker image pull --platform linux/amd64 quay.io/kuberay/operator:v1.2.2
$ crictl rmi local.repo.com/kuberay/operator:v1.2.2
```
重新安装成功
```shell
$ kubectl get pods -n kuberay-operator
NAME                                READY   STATUS    RESTARTS   AGE
kuberay-operator-5b5d75db55-65kwf   1/1     Running   0          3m55s
```
同时发现安装时还创建了 3 个 CustomResourceDefinition
```shell
$ kubectl get crd -n kuberay-operator
NAME                 CREATED AT
rayclusters.ray.io   2025-02-25T11:25:27Z
rayjobs.ray.io       2025-02-25T11:25:27Z
rayservices.ray.io   2025-02-25T11:25:28Z
```
3. **部署 Ray Cluster**   
同样在有外网的机器上拉取 ray 的镜像并推到私有仓库
```shell
$ docker image pull --platform linux/amd64 rayproject/ray:2.9.0
$ docker tag rayproject/ray:2.9.0 local.repo.com/rayproject/ray:2.9.0
$ docker push local.repo.com/rayproject/ray:2.9.0
```
从 [Github仓库](https://github.com/ray-project/kuberay-helm) 下载 ray-cluster-1.2.2.tgz 并解压，修改 values.yaml
```yaml
image:
  repository: local.repo.com/rayproject/ray
  tag: 2.9.0
  pullPolicy: IfNotPresent

nameOverride: "kuberay"
fullnameOverride: ""

imagePullSecrets: 
  - name: localrepo-secret-token
```
helm 打包安装 ray-cluster
```shell
$ helm package ray-cluster
$ helm install ray-cluster ./ray-cluster-1.2.2.tgz -n kuberay-operator
```
运行后发现 head 节点拉不起来，排查发现如下异常
> The node was low on resource: ephemeral-storage. Threshold quantity: 2214605502, available: 1443468Ki.   

即临时存储空间不足，根据 [官方文档](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/#configurations-for-local-ephemeral-storage) 本地临时性存储默认为 /var/lib/kubelet，排查发现该目录确实已无太多剩余空间，迁移后重试部署成功   
```shell
$ kubectl get pods -n kuberay-operator
NAME                                           READY   STATUS    RESTARTS       AGE
kuberay-operator-5b5d75db55-65kwf              1/1     Running   82 (84m ago)   1d
ray-cluster-kuberay-head-vbdcr                 1/1     Running   0              1m
ray-cluster-kuberay-workergroup-worker-kjvfk   1/1     Running   0              1m
$ kubectl get rayclusters -n kuberay-operator
NAME                  DESIRED WORKERS   AVAILABLE WORKERS   CPUS   MEMORY   GPUS   STATUS   AGE
ray-cluster-kuberay   1                 1                   2      3G       0      ready    1m
```
4. **提交任务**   
按照官方文档提交一个简单的测试Job
```shell
$ export HEAD_POD=$(kubectl get pods --selector=ray.io/node-type=head -o custom-columns=POD:metadata.name --no-headers -n kuberay-operator)
$ kubectl exec -it $HEAD_POD -n kuberay-operator -- python -c "import ray; ray.init(); print(ray.cluster_resources())"
2024-09-09 20:27:21,035 INFO worker.py:1405 -- Using address 127.0.0.1:6379 set in the environment variable RAY_ADDRESS
2024-09-09 20:27:21,035 INFO worker.py:1540 -- Connecting to existing Ray cluster at address: 10.100.7.132:6379...
2024-09-09 20:27:21,042 INFO worker.py:1715 -- Connected to Ray cluster. View the dashboard at http://10.100.7.132:8265 
{'CPU': 2.0, 'node:10.100.7.133': 1.0, 'memory': 3000000000.0, 'object_store_memory': 669735320.0, 'node:__internal_head__': 1.0, 'node:10.100.7.132': 1.0}
```
5. **Dashboard**   
将 Dashboard 的端口暴露到集群外
```shell
$ kubectl port-forward --address 0.0.0.0 pods/ray-cluster-kuberay-head-vbdcr 8265:8265 -n kuberay-operator
```
访问 http://<节点IP>:8265/ 即可看到刚才提交的Job