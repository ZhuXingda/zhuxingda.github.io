---
title: "【Kubernetes】记录一些在使用 Kubernetes 时遇到的问题"
date: "2025-02-25T16:57:16+08:00"
categories:
  - "开源框架"
tags:
  - "Kubernetes"
---
<!--more-->
## 网络
##### Service 的 type
1. ClusterIP
将 Service 暴露在集群内部的子网，可以通过 Ingress 和 网关暴露到外部。
2. NodePort
将 Service 以静态端口暴露在集群每个节点的 IP 上，访问任意节点外网 IP + static port 就能访问到 Service，在集群内部同 ClusterIP 一样为服务绑定了子网 IP 和端口。
```shell
NAME                        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE   SELECTOR
flink-job-rest   NodePort   10.200.88.165   <none>        8081:8153/TCP   53d   app=flink-job,component=jobmanager,type=flink-native-kubernetes
```
3. LoadBalancer
通过一个第三方 Load Balancer 暴露 Service 到集群外部
4. ExternalName
## 存储
### 迁移节点的文件系统目录到别的磁盘
- **nodefs**：包含非内存的 emptyDir volumes，log storage，ephemeral storage 和更多，默认的根目录为 `/var/lib/kubelet`。如果路径所在磁盘空间不足节点会进入 **NodeHasDiskPressure** 的状态，然后被驱逐。
- **imagefs**：CR 用来保存 container images 和 container writable layers，containerd 的默认路径是 `/var/lib/containerd`。如果路径所在磁盘空间不足节点会报 **FreeDiskSpaceFailed** 的错误，错误信息是 `Failed to garbage collect required amount of images. Attempted to free xxx bytes, but only found 0 bytes eligible to free.`   
#### 迁移 nodefs
节点停止调度
```shell
$ kubectl cordon worker-node-04
```
复制 kubelet 目录到新磁盘位置，并更新证书软链接指向新的文件。
```shell
$ cp -rf /var/lib/kubelet /home/disk3/var/kubelet
$ ll /home/disk3/var/kubelet/pki/kubelet-client-current.pem
lrwxrwxrwx 1 root root 59 Feb 26 16:58 /home/disk3/var/kubelet/pki/kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-09-03-11-40-15.pem
$ rm /home/disk3/var/kubelet/pki/kubelet-client-current.pem
$ ln -s /home/disk3/var/kubelet/pki/kubelet-client-2024-09-03-11-40-15.pem /home/disk3/var/kubelet/pki/kubelet-client-current.pem
$ ll /home/disk3/var/kubelet/pki/kubelet-client-current.pem
lrwxrwxrwx 1 root root 59 Feb 26 16:58 /home/disk3/var/kubelet/pki/kubelet-client-current.pem -> /home/disk3/var/kubelet/pki/kubelet-client-2024-09-03-11-40-15.pem
```
根据 [这个问答](https://stackoverflow.com/questions/70931881/what-does-kubelet-use-to-determine-the-ephemeral-storage-capacity-of-the-node) 在配置文件中加上新的 root dir 路径
```shell
cat << EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--root-dir=/home/disk3/var/kubelet 
EOF
```
然后重启 kubelet
```shell
$ source /etc/sysconfig/kubelet
$ systemctl restart kubelet
```
检查 kubelet 状态的时候发现有报错
> kubelet[66904]: E0226 17:44:46.723343   66904 file_linux.go:61] "Unable to read config path" err="path does not exist, ignoring" path="/etc/kubernetes/manifests"
根据 [这个 ISSUE](https://github.com/kubernetes/kubeadm/issues/1345) 创建对应目录解决
```shell
$ mkdir -p /etc/kubernetes/manifests
```
最后恢复节点调度，查看节点信息可以看到 ephemeral-storage 明显增大
```shell
$ kubectl uncordon worker-node-04
$ kubectl describe node worker-node-04
...
Capacity:
  ...
  ephemeral-storage:  7752283564Ki
  ...
Allocatable:
  ...
  ephemeral-storage:  7144504520754
...
```
#### 迁移 imagefs
迁移 imagefs 需要重启 containerd，必须把节点上运行的 pod 都先驱逐掉，可能会造成服务的中断，`对线上环境存在风险！` 

节点停止调度并驱逐
```shell
$ kubectl cordon worker-node-04
$ kubectl drain worker-node-04 --ignore-daemonsets --delete-emptydir-data
```
创建新的 imagefs 目录并修改 containerd 配置文件
```shell
$ mkdir -p /home/disk3/var/lib/
$ cp -R /var/lib/containerd /home/disk3/var/lib/
$ sed -i 's/\/var\/lib\/containerd/\/home\/disk3\/var\/lib\/containerd/g' /etc/containerd/config.toml
$ systemctl stop kubelet
$ systemctl restart containerd
$ systemctl start kubelet
```
检查 containerd 和 kubelet 状态
```shell
$ systemctl status containerd
$ systemctl status kubelet
```
用 `kubectl describe node` 命令没办法查看 imagefs 的磁盘使用情况，需要用下面的命令
```shell
$ kubectl get --raw "/api/v1/nodes/worker-node-04/proxy/stats/summary"
```
输出信息中有这样一段分别是 nodefs 和 imagefs 的磁盘使用情况，可见 imagefs 调整成功
```json
{
"fs": {
   "time": "2025-02-22T06:56:09Z",
   "availableBytes": 7929085259776,
   "capacityBytes": 7938338369536,
   "usedBytes": 9236332544,
   "inodesFree": 244175428,
   "inodes": 244191232,
   "inodesUsed": 15804
  },
  "runtime": {
   "imageFs": {
    "time": "2025-02-22T06:56:09Z",
    "availableBytes": 7929085259776,
    "capacityBytes": 7938338369536,
    "usedBytes": 5989388288,
    "inodesFree": 244175428,
    "inodes": 244191232,
    "inodesUsed": 12920
   }
  }
}
```
恢复节点调度   
```shell
$ kubectl uncordon worker-node-04
```
#### 踩坑
注意上面两个目录特别是 nodefs 在迁移时需要注意新的目录所属的权限，如果不是属于 root:root 有可能在任务运行时出现容器内目录没有权限的问题，比如我在更改目录时把 nodefs 配置到 work 用户的目录下，导致 Flink 任务运行时没有容器内 /tmp 目录的写权限，产生如下报错
>Shutting KubernetesSessionClusterEntrypoint down with application status FAILED. Diagnostics java.lang.RuntimeException: unable to generate a JAAS configuration file
        at org.apache.flink.runtime.security.modules.JaasModule.generateDefaultConfigFile(JaasModule.java:183)
        at org.apache.flink.runtime.security.modules.JaasModule.install(JaasModule.java:92)
        at org.apache.flink.runtime.security.SecurityUtils.installModules(SecurityUtils.java:76)
        at org.apache.flink.runtime.security.SecurityUtils.install(SecurityUtils.java:57)
        at org.apache.flink.runtime.entrypoint.ClusterEntrypoint.installSecurityContext(ClusterEntrypoint.java:279)
        at org.apache.flink.runtime.entrypoint.ClusterEntrypoint.startCluster(ClusterEntrypoint.java:231)
        at org.apache.flink.runtime.entrypoint.ClusterEntrypoint.runClusterEntrypoint(ClusterEntrypoint.java:739)
        at org.apache.flink.kubernetes.entrypoint.KubernetesSessionClusterEntrypoint.main(KubernetesSessionClusterEntrypoint.java:61)
Caused by: java.nio.file.AccessDeniedException: /tmp/jaas-4451190208096254189.conf
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:84)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
        at sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:214)
        at java.nio.file.Files.newByteChannel(Files.java:361)
        at java.nio.file.Files.createFile(Files.java:632)
        at java.nio.file.TempFileHelper.create(TempFileHelper.java:138)
        at java.nio.file.TempFileHelper.createTempFile(TempFileHelper.java:161)
        at java.nio.file.Files.createTempFile(Files.java:852)
        at org.apache.flink.runtime.security.modules.JaasModule.generateDefaultConfigFile(JaasModule.java:173)
        ... 7 more