---
title: '【Kubernetes】高可用 Kubernetes 集群搭建（with kubeadm）'
date: 2024-09-05
categories:
    - '分布式系统'
tags:
    - 'Kubernetes'
---
利用 kubeadm 搭建高可用 Kubernetes 集群的流程记录
<!--more-->

## 0. 集群概况
| 节点 | 角色 |
| ---- | ---- |
| sever-01 | Master01 |
| sever-02 | Master02 |
| sever-03 | Master04 |
| sever-04 | Worker01 |
| sever-05 | Worker02 |
| ... | ... |
| sever-18 | Worker15 |
| sever-19 | Worker16 |
| sever-20 | Worker17 |

操作系统：CentOS 7.6
## 1. 基础网络配置（所有节点）
安装依赖 
```shell
$ yum install socat conntrack
```
bridge-nf 使得 netfilter 可以对 Linux 网桥上的 IPv4/ARP/IPv6 包过滤。比如设置net.bridge.bridge-nf-call-iptables＝1 后，二层的网桥在转发包时也会被 iptables 的 FORWARD 规则所过滤。
```shell
$ cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
$ modprobe overlay && modprobe br_netfilter
```
IPv4 流量转发 & 桥接流量走 iptables
```shell
$ cat > /etc/sysctl.d/99-kubernetes-cri.conf << EOF
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
$ sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
```
## 2. 配置 Container Runtime（所有节点）
1. 参考 教程 里的 Option1，下载 cni-plugins-linux-amd64-v1.5.1.tgz  containerd-1.6.35-linux-amd64.tar.gz  runc.amd64 三个安装包   
2. containerd-1.6.35-linux-amd64.tar.gz 解压到 /usr/local
```shell
tar Cxzvf /usr/local containerd-1.6.35-linux-amd64.tar.gz
```
3. 启动 containerd：在 /usr/lib/systemd/system 创建 containerd.service，内容参考 [官方文件](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service)，然后用 systemctl 启动：
```
$ cp containerd.service /usr/lib/systemd/system/
$ systemctl daemon-reload
$ systemctl enable --now containerd
$ systemctl status containerd.service
```
4. 安装 runc
```shell
$ install -m 755 runc.amd64 /usr/local/sbin/runc
```
5. 安装 cni-plugins
```shell
$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz
```
6. 配置 containerd
```shell
$ mkdir -p  /etc/containerd/
$ containerd config default > /etc/containerd/config.toml
```
结合 runc 使用 systemd cgroup 驱动，顺便可以修改一个可用的镜像仓库地址，在 /etc/containerd/config.toml 中设置：
``` toml
    [plugins."io.containerd.grpc.v1.cri"]
        sandbox_image = "registry.k8s.io/kubernetes/pause:3.9"
      ...
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      ...
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
修改后重启
```shell
$ systemctl restart containerd.service
```
## 3. 安装 kubeadm、kubelet 和 kubectl（所有节点）
1. 安装 cri-tools   
下载 crictl-v1.28.11-linux-amd64.tar.gz，校验后解压到 /usr/local/bin
```shell
$ tar Cxzvf /usr/local/bin/ crictl-v1.31.1-linux-amd64.tar.gz
```
2. 安装 kubectl
```shell
# 下载 kubectl 二进制
$ wget https://dl.k8s.io/release/v1.28.11/bin/linux/amd64/kubectl
$ install -m +x kubectl /usr/local/bin/
# 复制配置文件
$ cp /etc/kubernetes/admin.conf /home/work/.kube/config
$ chown work:work /home/work/.kube/config
$ chmod 744 /home/work/.kube/config
```
3. 安装  kubelet kubeadm
```shell
$ wget https://dl.k8s.io/release/v1.28.11/bin/linux/amd64/kubeadm
$ wget https://dl.k8s.io/release/v1.28.11/bin/linux/amd64/kubelet
$ install -m +x kubeadm /usr/local/bin/
$ install -m +x kubelet /usr/local/bin/
$ curl -sSL https://raw.githubusercontent.com/kubernetes/release/v0.16.2/cmd/krel/templates/latest/kubelet/kubelet.service | sed "s:/usr/bin:/usr/local/bin:g" > kubelet.service
$ curl -sSL https://raw.githubusercontent.com/kubernetes/release/v0.16.2/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf | sed "s:/usr/bin:/usr/local/bin:g" > 10-kubeadm.conf
$ chmod 644 kubelet.service 10-kubeadm.conf
$ cp kubelet.service /usr/lib/systemd/system/
$ mkdir /usr/lib/systemd/system/kubelet.service.d/
$ cp 10-kubeadm.conf /usr/lib/systemd/system/kubelet.service.d/
$ systemctl daemon-reload
$ systemctl enable --now kubelet.service
# kubelet 现在每隔几秒就会重启，因为它陷入了一个等待 kubeadm 指令的死循环。
```
## 4. 启动集群
1. 申请 VIP
创建一个 VIP，代理端口为 6443，代理机器为 Master 节点的三台机器：
* sever-01 
* sever-02 
* sever-03
2. 启动 Master01 节点（sever-01 ）   
kubeadm init 配置文件
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: ${my_k8s_cluster_token}
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
localAPIEndpoint:
  # master01 节点的 IP
  advertiseAddress: 10.93.83.222
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: sever-01
  taints: null
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
# VIP 和 代理端口
controlPlaneEndpoint: "10.11.110.15:6443"
apiServer:
  timeoutForControlPlane: 4m0s
  extraArgs:
    service-node-port-range: 8100-8300
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    imageTag: "3.5.9-0"
    dataDir: /var/lib/etcd
imageRepository: /google_containers
kubernetesVersion: 1.28.11
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.200.0.0/16
  podSubnet: 10.100.0.0/16
scheduler: {}
```
拉取镜像 (所有节点)
```shell
$ kubeadm config images --config ./master01_init.yaml list
registry.k8s.io/kubernetes/kube-apiserver:v1.28.11
registry.k8s.io/kubernetes/kube-controller-manager:v1.28.11
registry.k8s.io/kubernetes/kube-scheduler:v1.28.11
registry.k8s.io/kubernetes/kube-proxy:v1.28.11
registry.k8s.io/kubernetes/pause:3.9
registry.k8s.io/kubernetes/etcd:3.5.9-0
registry.k8s.io/coredns:v1.10.1
$ kubeadm config images --config ./master01_init.yaml pull
```
启动控制面
```shell
# 应该执行的命令
$ kubeadm init --config master01_init.yaml --upload-certs
# 我执行的命令，少了一个 --upload-certs 会导致没有 certificate-key 生成，control-plane 加入集群必须要有这个 key
$ kubeadm init --config master01_init.yaml
```
```shell
W0902 16:13:49.154849   30355 initconfiguration.go:307] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta3", Kind:"ClusterConfiguration"}: strict decoding error: unknown field "dns.type"
[init] Using Kubernetes version: v1.28.11
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local sever-01] and IPs [10.200.0.1 10.93.83.222 10.11.110.15]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost sever-01] and IPs [10.93.83.222 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost sever-01] and IPs [10.93.83.222 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 18.636847 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-check] Initial timeout of 40s passed.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node sever-01 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node sever-01 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: ${my_k8s_cluster_token}
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.11.110.15:6443 --token ${my_k8s_cluster_token} \
        --discovery-token-ca-cert-hash sha256:${my_discovery_token_ca_cert_hash} \
        --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.11.110.15:6443 --token ${my_k8s_cluster_token} \
        --discovery-token-ca-cert-hash sha256:${my_discovery_token_ca_cert_hash}
```
如果上一条启动命令忘记加 '--upload-certs' 参数导致 certificate-key 没有生成，可以用下面这个命令重新生成
```shell
$ kubeadm init phase upload-certs --upload-certs
```
3. 启动 Master02 和 Master03（sever-02 和 sever-03 ）
用 2 的 master01_init.yaml 文件执行同样的拉镜像操作，然后按配置加入集群
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.11.110.15:6443
    token: ${my_k8s_cluster_token}
    caCertHashes:
      - sha256:${my_discovery_token_ca_cert_hash}
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: ${my_k8s_cluster_token}
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: sever-02 (或者 sever-03)
  taints: null
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.93.83.222
    bindPort: 6443
  certificateKey: 7cf18abaf200ac50cf3d31d51fbe1ca1851c7918bfbd136fc090884ec13aaf01
```
```shell
# 在 sever-02 执行
$ kubeadm join --config master02_join.yaml
# 在 sever-03 执行
$ kubeadm join --config master03_join.yaml
```
4. 配置网络插件
```shell
# 下载 flannel 部署配置文件
$ wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# 修改镜像地址（如果有需要）、 pod 子网
```
```yaml
# ...
---
apiVersion: v1
data:
  # ...
  # 这里 Network 和前面 pod subnet 保持一致
  net-conf.json: |
    {
      "Network": "10.100.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
# ...
```
在集群中启动 flannel
```shell
$ kubectl apply -f kube-flannel.yml
```
5. 加入 Worker 节点   
节点数量较多，用脚本批量执行
```shell
#!/bin/bash
yum install -y socat conntrack
cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
modprobe overlay && modprobe br_netfilter
cat > /etc/sysctl.d/99-kubernetes-cri.conf << EOF
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
tar Cxzvf /usr/local containerd-1.6.35-linux-amd64.tar.gz
cp containerd.service /usr/lib/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd.service
install -m 755 runc.amd64 /usr/local/sbin/runc
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz
mkdir -p  /etc/containerd/
containerd config default > /etc/containerd/config.toml
# sed -i 's/sandbox_image = .*/sandbox_image = "registry.k8s.io\/kubernetes\/pause:3.9"/g' /etc/containerd/config.toml
sed -i 's/SystemdCgroup = true/SystemdCgroup = false/g' /etc/containerd/config.toml
systemctl restart containerd.service
tar Cxzvf /usr/local/bin/ crictl-v1.31.1-linux-amd64.tar.gz
install -m +x kubeadm /usr/local/bin/
install -m +x kubelet /usr/local/bin/
chmod 644 kubelet.service 10-kubeadm.conf
cp kubelet.service /usr/lib/systemd/system/
mkdir /usr/lib/systemd/system/kubelet.service.d/
cp 10-kubeadm.conf /usr/lib/systemd/system/kubelet.service.d/
systemctl daemon-reload
systemctl enable --now kubelet.service
kubeadm config images --config ./master01_init.yaml list
kubeadm config images --config ./master01_init.yaml pull
hostname=$(hostname)
sed -i "s/name: .*/name: ${hostname}/g" worker_join.yaml
kubeadm join --config worker_join.yaml
```
```shell
$ ls ./k8s
10-kubeadm.conf                       containerd.service                 kubelet             runc.amd64
cni-plugins-linux-amd64-v1.5.1.tgz    crictl-v1.31.1-linux-amd64.tar.gz  kubelet.service     worker_join.sh
containerd-1.6.35-linux-amd64.tar.gz  kubeadm     
$ tar -czf k8s_worker.tgz ./k8s
# 重复分发到各个 Worker 节点上并部署
$ scp ./k8s_worker.tgz root@Workerxx-host:/root/
$ ssh root@Workerxx-host "cd /root/k8s; sh worker_join.sh"
```
6. 部署 metric-server   
下载 [metric-server 部署文件](https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.2/components.yaml) 并修改配置中的镜像仓库地址（如有需要）
```yaml
# ...
apiVersion: apps/v1
kind: Deployment
# ...
spec:
# ...
  template:
  # ...
    spec:
      containers:
      - args:
        # ...
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-insecure-tls
        image: registry.k8s.io/metrics-server/metrics-server:v0.6.3
```
部署 metric-server
```shell
$ kubectl apply -f components.yaml
$ kubectl top node sever-10
NAME                          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
sever-10   917m         0%     31836Mi         6%
```
7. 部署 Kubernetes-Dashboard   
下载 [kubernetes-dashboard 部署文件](https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml) 并修改    
recommended.yaml
```yaml
# ...
# 修改 kubernetes-dashboard service 绑定的端口
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 8443
      nodePort: 8100
  selector:
    k8s-app: kubernetes-dashboard
...
      # 修改镜像地址
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.6.1
...
      # 修改镜像地址
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.8
```
```shell
# 部署 kubernetes-dashboard
$ kubectl apply -f recommended.yaml
# 创建管理员 service account
$ kubectl apply -f create_dashboard_admin.yaml
$ kubectl get secret dashboard-admin-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```
create_dashboard_admin.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
  
---

apiVersion: v1
kind: Secret
metadata:
  name: dashboard-admin-secret
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: dashboard-admin
type: kubernetes.io/service-account-token
```
控制台地址：https://sever-01-host:8100

## 参考资料
https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#kube-vip