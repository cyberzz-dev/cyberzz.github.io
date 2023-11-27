---
title: Kubernetes
description: install k8s 
date: 2023-09-09
slug: test-chinese
categories:
    - Kubernetes
    - Install
---

Kubernetes

以下是Kubernetes的主要知识点，可以作为学习和使用Kubernetes的参考：

1. 容器技术基础：Docker，Pod
2. Kubernetes的基本概念：Node，Pod，Deployment，Service，ConfigMap，Secret，Ingress，Namespace，Label
3. Kubernetes的核心组件：API Server，etcd，kube-scheduler，kube-controller-manager，kubelet，kube-proxy
4. Kubernetes的对象模型：Kubernetes API 对象，Kubernetes API 的版本
5. Kubernetes的部署方式：单节点，多节点，云平台
6. Kubernetes的资源管理：CPU，内存，存储，网络
7. Kubernetes的服务发现和负载均衡：Service，Endpoint，Ingress，LoadBalancer
8. Kubernetes的存储管理：PersistentVolume，PersistentVolumeClaim，StorageClass
9. Kubernetes的网络管理：CNI，网络策略
10. Kubernetes的自动化：自动伸缩，自动扩容，自动升级
11. Kubernetes的扩展：CRD，Operator
12. Kubernetes的安全：TLS，RBAC，Pod Security Policy，Secrets Encryption
13. Kubernetes的调试和故障排除：kubectl 命令，事件，日志，debug container
14. Kubernetes的监控：Heapster，Prometheus，Grafana
15. Kubernetes的CI/CD：Jenkins，GitLab，Spinnaker

以上是Kubernetes的主要知识点，Kubernetes是一个庞大而复杂的系统，需要不断的学习和实践才能掌握。建议按照学习步骤逐一深入学习。



# 1. 准备

https://www.cnblogs.com/bamboo-green/p/17128474.html

## service monitor 无法发现

https://www.kococ.cn/20211124/cid=721.html

https://www.lvbibir.cn/posts/tech/kubernetes-prometheus-3-servicemonitor-podmonitor/



```yaml

```



## 1.1 系统配置

- ### 修改hosts

在安装之前，需要先做好如下准备。3台CentOS 7.9主机如下：

```bash
[root@master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.20 master
192.168.10.21 node01
192.168.10.22 node02
192.168.10.23 node03
```

在各个主机上完成下面的系统配置。

- ### 关闭防火墙

如果各个主机启用了防火墙策略，需要开放Kubernetes各个组件所需要的端口，可以查看[Ports and Protocols](https://kubernetes.io/docs/reference/ports-and-protocols/)中的内容, 开放相关端口或者关闭主机的防火墙。

```bash
systemctl disable firewalld --now
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

- ### 关闭 selinux

```shell
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'  /etc/selinux/config
```

- ### 必须安装iptables

- ### 创建/etc/modules-load.d/containerd.conf配置文件:

```bash
cat << EOF > /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

使配置生效

```bash
modprobe overlay
modprobe br_netfilter
```

创建/etc/sysctl.d/99-kubernetes-cri.conf配置文件：

```bash
cat << EOF > /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
user.max_user_namespaces=28633
EOF
```

执行以下命令使配置生效:

```bash
sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
```

## 1.2 加载内核modules ipvs

可以参考https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md

由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

内核模块位置：

```
/lib/modules/5.4.209-1.el7.elrepo.x86_64
```

在各个服务器节点上执行以下脚本:

```shell
yum install ipvsadm ipset sysstat conntrack libseccomp -y
# 配置ipvs模块，在内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl enable --now systemd-modules-load.service
```

使用`lsmod | grep -e ip_vs -e nf_conntrack`命令查看是否已经正确加载所需的内核模块。

接下来还需要确保各个节点上已经安装了ipset软件包，为了便于查看ipvs的代理规则，最好安装一下管理工具ipvsadm。

```shell
yum install -y ipset ipvsadm
```

如果不满足以上前提条件，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式。



## 1.3 docker

### 安装Docker

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl enable docker && systemctl start docker
```

配置镜像下载加速器：

```javascript
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://292nm6nu.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl restart docker
docker info
```

### 安装cri-dockerd

Kubernetes v1.24移除docker-shim的支持，而Docker Engine默认又不支持CRI标准，因此二者默认无法再直接集成。为此，Mirantis和Docker联合创建了cri-dockerd项目，用于为Docker Engine提供一个能够支持到CRI规范的桥梁，从而能够让Docker作为Kubernetes容器引擎。

如图所示：

[https://img2023.cnblogs.com/blog/1772495/202302/1772495-20230216220036444-1454369319.png]: 

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1-3.el7.x86_64.rpm
rpm -ivh cri-dockerd-0.3.1-3.el7.x86_64.rpm
```

指定依赖镜像地址：

```javascript
vim /usr/lib/systemd/system/cri-docker.service
把原来的注释一下添加如下
#ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://

ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7

systemctl daemon-reload 
systemctl enable cri-docker && systemctl start cri-docker
```

### 添加阿里云YUM软件源

```makefile
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```bash
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet
```



## 1.4  Containerd、runc

**1、配置先决条件**

如果是由docker切containerd这步可省略。

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置必需的 sysctl 参数，这些参数在重新启动后仍然存在。
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**2、安装containerd**

```lua
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install -y containerd.io
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

**3、修改配置文件**

- pause镜像设置过阿里云镜像仓库地址
- 拉取Docker Hub镜像配置加速地址设置为阿里云镜像仓库地址

```csharp
vi /etc/containerd/config.toml
...
   [plugins."io.containerd.grpc.v1.cri"]
      sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"  
       ...
   [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
          [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
            endpoint = ["https://b9pmyelo.mirror.aliyuncs.com"]
          
systemctl restart containerd
```

**4、配置kubelet使用containerd**

```bash
vi /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8"

systemctl restart kubelet
```

**5、验证**

```csharp
kubectl get node -o wide

k8s-node1  xxx  containerd://1.5.6
```

![get node](https://img2023.cnblogs.com/blog/1772495/202302/1772495-20230216221027779-324412180.png)

**6、管理容器工具**

containerd提供了ctr命令行工具管理容器，但功能比较简单，所以一般会用crictl工具检查和调试容器。

项目地址：https://github.com/kubernetes-sigs/cri-tools/

设置crictl连接containerd：

```bash
vi /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```







在各个服务器节点上安装容器运行时Containerd。

可以选择yum安装：

```shell
yum install -y containerd.io runc
```

下载Containerd的二进制包:

```shell
wget https://github.com/containerd/containerd/releases/download/v1.6.6/cri-containerd-cni-1.6.6-linux-amd64.tar.gz
```

`cri-containerd-cni-1.6.4-linux-amd64.tar.gz`压缩包中已经按照官方二进制部署推荐的目录结构布局好。 里面包含了systemd配置文件，containerd以及cni的部署文件。 将解压缩到系统的根目录`/`中:

```shell
tar -zxvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz -C /

etc/
etc/systemd/
etc/systemd/system/
etc/systemd/system/containerd.service
etc/crictl.yaml
etc/cni/
etc/cni/net.d/
etc/cni/net.d/10-containerd-net.conflist
usr/
usr/local/
usr/local/sbin/
usr/local/sbin/runc
usr/local/bin/
usr/local/bin/critest
usr/local/bin/containerd-shim
usr/local/bin/containerd-shim-runc-v1
usr/local/bin/ctd-decoder
usr/local/bin/containerd
usr/local/bin/containerd-shim-runc-v2
usr/local/bin/containerd-stress
usr/local/bin/ctr
usr/local/bin/crictl
......
opt/cni/
opt/cni/bin/
opt/cni/bin/bridge
......
```

注意经测试cri-containerd-cni-1.6.4-linux-amd64.tar.gz包中包含的runc在CentOS 7下的动态链接有问题，这里从runc的github上单独下载runc，并替换上面安装的containerd中的runc:

```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.2/runc.amd64
```

接下来生成containerd的配置文件:

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

根据文档[Container runtimes ](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)中的内容，对于使用systemd作为init system的Linux的发行版，使用systemd作为容器的cgroup driver可以确保服务器节点在资源紧张的情况更加稳定，因此这里配置各个节点上containerd的cgroup driver为systemd。

修改前面生成的配置文件`/etc/containerd/config.toml`：

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

再修改`/etc/containerd/config.toml`中的

```toml
[plugins."io.containerd.grpc.v1.cri"]
  ...
  # sandbox_image = "k8s.gcr.io/pause:3.6"
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
```

配置containerd开机启动，并启动containerd

```bash
systemctl disable containerd --now
```

使用crictl测试一下，确保可以打印出版本信息并且没有错误信息输出:

```shell
crictl version
Version:  0.1.0
RuntimeName:  containerd
RuntimeVersion:  v1.6.4
RuntimeApiVersion:  v1alpha2
```

# 2. 使用kubeadm部署Kubernetes

## 2.1 安装kubeadm和kubelet

下面在各节点安装kubeadm和kubelet：

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```bash
yum makecache fast
yum install kubelet kubeadm kubectl
```

运行`kubelet --help`可以看到原来kubelet的绝大多数命令行flag参数都被`DEPRECATED`了，官方推荐我们使用`--config`指定配置文件，并在配置文件中指定原来这些flag所配置的内容。具体内容可以查看这里[Set Kubelet parameters via a config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)。**最初Kubernetes这么做是为了支持动态Kubelet配置（Dynamic Kubelet Configuration），但动态Kubelet配置特性从k8s 1.22中已弃用，并在1.24中被移除。如果需要调整集群汇总所有节点kubelet的配置，还是推荐使用ansible等工具将配置分发到各个节点**。

kubelet的配置文件必须是json或yaml格式，具体可查看[这里](https://github.com/kubernetes/kubelet/blob/release-1.24/config/v1beta1/types.go)。

Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。 关闭系统的Swap方法如下:

```bash
swapoff -a
```

修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用`free -m`确认swap已经关闭。

swappiness参数调整，修改/etc/sysctl.d/99-kubernetes-cri.conf添加下面一行：

```bash
echo "vm.swappiness=0" >> /etc/sysctl.d/99-kubernetes-cri.conf
sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
```

## 2.2 kubeadm init初始化

在各节点开机启动kubelet服务：

```bash
systemctl enable kubelet.service --now
systemctl enable containerd.service --now
```

使用

```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration
```

可以打印集群初始化默认的使用的配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.24.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

从默认的配置中可以看到，可以使用`imageRepository`定制在集群初始化时拉取k8s所需镜像的地址。基于默认配置定制出本次使用kubeadm初始化集群所需的配置文件kubeadm.yaml：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.96.151
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  taints:
  - effect: PreferNoSchedule
    key: node-role.kubernetes.io/master
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.24.0
imageRepository: registry.aliyuncs.com/google_containers
networking:
  podSubnet: 10.244.0.0/16
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
failSwapOn: false
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

这里定制了`imageRepository`为阿里云的registry，避免因gcr被墙，无法直接拉取镜像。`criSocket`设置了容器运行时为containerd。 同时设置kubelet的`cgroupDriver`为systemd，设置kube-proxy代理模式为ipvs。

在开始初始化集群之前可以使用`kubeadm config images pull --config kubeadm.yaml`预先在各个服务器节点上拉取所k8s需要的容器镜像。

```bash
kubeadm config images pull --config kubeadm.yaml
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.24.0
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.7
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.3-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.8.6
```

接下来使用kubeadm初始化集群，选择node1作为Master Node，在node1上执行下面的命令：

```bash
  kubeadm init \
  --apiserver-advertise-address=192.168.10.20 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.26.3 \
  --service-cidr=10.96.0.0/12 \
# --cri-socket=unix:///var/run/cri-dockerd.sock \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all
```

或者

```bash
kubeadm init --config kubeadm.yaml
W0526 10:22:26.657615   24076 common.go:83] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta2". Please use 'kubeadm config migrate --old-config old.yaml --new-config new.yaml', which will write the new, similar spec using a newer API version.
W0526 10:22:26.660300   24076 initconfiguration.go:120] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
[init] Using Kubernetes version: v1.24.0
[preflight] Running pre-flight checks
	[WARNING Swap]: swap is enabled; production deployments should disable swap unless testing the NodeSwap feature gate of the kubelet
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local node1] and IPs [10.96.0.1 192.168.96.151]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost node1] and IPs [192.168.96.151 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost node1] and IPs [192.168.96.151 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 17.506804 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node node1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node node1 as control-plane by adding the taints [node-role.kubernetes.io/master:PreferNoSchedule]
[bootstrap-token] Using token: uufqmm.bvtfj4drwfvvbcev
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.20:6443 --token uufqmm.bvtfj4drwfvvbcev \
	--discovery-token-ca-cert-hash sha256:5814415567d93f6d2d41fe4719be8221f45c29c482b5059aec2e27a832ac36e6
```

上面记录了完成的初始化输出的内容，根据输出的内容基本上可以看出手动初始化安装一个Kubernetes集群所需要的关键步骤。 其中有以下关键内容：

- `[certs]`生成相关的各种证书

- `[kubeconfig]`生成相关的kubeconfig文件

- `[kubelet-start]` 生成kubelet的配置文件"/var/lib/kubelet/config.yaml"

- `[control-plane]`使用`/etc/kubernetes/manifests`目录中的yaml文件创建apiserver、controller-manager、scheduler的静态pod

- `[bootstraptoken]`生成token记录下来，后边使用`kubeadm join`往集群中添加节点时会用到

- 下面的命令是配置常规用户如何使用kubectl访问集群：master节点

  ```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  export KUBECONFIG=/etc/kubernetes/admin.conf
  ```

- 最后给出了将节点加入集群的命令

  ```bash
  kubeadm join 192.168.10.20:6443 --token uufqmm.bvtfj4drwfvvbcev \
  	--discovery-token-ca-cert-hash sha256:5814415567d93f6d2d41fe4719be8221f45c29c482b5059aec2e27a832ac36e6
  ```

查看一下集群状态，确认个组件都处于healthy状态，结果出现了错误:

```bash
kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
```

## 2.2 kubeadm reset

集群初始化如果遇到问题，可以使用

```bash
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

命令进行清理。

## 2.3 安装helm 3

Helm是Kubernetes的包管理器，后续流程也将使用Helm安装Kubernetes的常用组件。 这里先在master节点node1上安装helm。

```bash
wget https://repo.huaweicloud.com/helm/v3.11.2/helm-v3.11.2-linux-amd64.tar.gz
tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
mv linux-amd64/helm  /usr/local/bin/
```

## 2.4 部署Network组件Calico

https://www.golinuxcloud.com/calico-kubernetes/

https://blog.csdn.net/qq_34556414/article/details/114937418

https://www.cnblogs.com/Cylon/p/14399682.html

https://blog.csdn.net/maomao5945/article/details/100773007

#### Install Calico

1. Download Calico CNI plugin

   We will download the Calico networking manifest and use it to install the plugin for the Kubernetes API datastore.

   ```
   [root@controller ~]# wget https://docs.projectcalico.org/v3.25/manifests/calico.yaml
   ```

4. Modify pod CIDR (Optional)

   Next you must assign a pod CIDR subnet. CIDR stands for Classless Inter-Domain Routing, also known as *supernetting*. By default Calico assumes that you wish to assign `192.168.0.0/16` subnet for the pod network but if you wish to choose any other subnet then you can add the same in `calico.yaml` file.

   I am already using `192.168.0.0/24` for my Kubernetes Cluster and I don't want to use the same range for my Pods. So I will assign a random subnet `10.142.0.0/24` as my CIDR for pods.

   We will open the `calico.yaml` using vim editor and modify `CALICO_IPV4POOL_CIDR` variable in the manifest and set it to `10.142.0.0/24` as shown below:

   ```
               - name: CALICO_IPV4POOL_CIDR
                 value: "10.142.0.0/24"
   ```

5. Install Calico Plugin

   Next we can go ahead and install the Calico network using `kubectl` command with calico manifest file:

   ```
   [root@controller ~]# kubectl apply -f calico.yaml
   ```

6. Confirm that all of the pods are running with the following command.

   ```
   watch kubectl get pods -n kube-system
   ```

   Wait until each pod has the `STATUS` of `Running`.

   > **Note**: The Tigera operator installs resources in the `calico-system` namespace. Other install methods may use the `kube-system` namespace instead.



https://www.tigera.io/blog/advertising-kubernetes-service-ips-with-calico-and-bgp/



host network

```shell
calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 192.168.10.6
  asNumber: 64512
EOF
```

pod network  service network

```
calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.96.0.0/12
  - cidr: 10.16.0.0/12
EOF
```



选择calico作为k8s的Pod网络组件，下面使用helm在k8s集群中安装calico。

下载`tigera-operator`的`helm chart`:

```bash
wget https://github.com/projectcalico/calico/releases/download/v3.23.1/tigera-operator-v3.23.1.tgz
```

查看这个chart的中可定制的配置:

```yaml
helm show values tigera-operator-v3.23.1.tgz

imagePullSecrets: {}

installation:
  enabled: true
  kubernetesProvider: ""

apiServer:
  enabled: true

certs:
  node:
    key:
    cert:
    commonName:
  typha:
    key:
    cert:
    commonName:
    caBundle:

resources: {}

# Configuration for the tigera operator
tigeraOperator:
  image: tigera/operator
  version: v1.27.1
  registry: quay.io
calicoctl:
  image: docker.io/calico/ctl
  tag: v3.23.1
```

定制的`values.yaml`如下:

```bash
# 可针对上面的配置进行定制,例如calico的镜像改成从私有库拉取。
# 这里只是个人本地环境测试k8s新版本，因此保留value.yaml为空即可
```

使用helm安装calico：

```bash
helm install calico tigera-operator-v3.23.1.tgz -n kube-system  --create-namespace -f values.yaml
```

等待并确认所有pod处于Running状态:

```bash
kubectl get pod -n kube-system | grep tigera-operator
tigera-operator-5fb55776df-wxbph   1/1     Running   0             5m10s

kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-68884f975d-5d7p9   1/1     Running   0          5m24s
calico-node-twbdh                          1/1     Running   0          5m24s
calico-typha-7b4bdd99c5-ssdn2              1/1     Running   0          5m24s
```

查看一下calico向k8s中添加的api资源:

```bash
kubectl api-resources | grep calico
bgpconfigurations                                                                 crd.projectcalico.org/v1               false        BGPConfiguration
bgppeers                                                                          crd.projectcalico.org/v1               false        BGPPeer
blockaffinities                                                                   crd.projectcalico.org/v1               false        BlockAffinity
caliconodestatuses                                                                crd.projectcalico.org/v1               false        CalicoNodeStatus
clusterinformations                                                               crd.projectcalico.org/v1               false        ClusterInformation
felixconfigurations                                                               crd.projectcalico.org/v1               false        FelixConfiguration
globalnetworkpolicies                                                             crd.projectcalico.org/v1               false        GlobalNetworkPolicy
globalnetworksets                                                                 crd.projectcalico.org/v1               false        GlobalNetworkSet
hostendpoints                                                                     crd.projectcalico.org/v1               false        HostEndpoint
ipamblocks                                                                        crd.projectcalico.org/v1               false        IPAMBlock
ipamconfigs                                                                       crd.projectcalico.org/v1               false        IPAMConfig
ipamhandles                                                                       crd.projectcalico.org/v1               false        IPAMHandle
ippools                                                                           crd.projectcalico.org/v1               false        IPPool
ipreservations                                                                    crd.projectcalico.org/v1               false        IPReservation
kubecontrollersconfigurations                                                     crd.projectcalico.org/v1               false        KubeControllersConfiguration
networkpolicies                                                                   crd.projectcalico.org/v1               true         NetworkPolicy
networksets                                                                       crd.projectcalico.org/v1               true         NetworkSet
bgpconfigurations                 bgpconfig,bgpconfigs                            projectcalico.org/v3                   false        BGPConfiguration
bgppeers                                                                          projectcalico.org/v3                   false        BGPPeer
caliconodestatuses                caliconodestatus                                projectcalico.org/v3                   false        CalicoNodeStatus
clusterinformations               clusterinfo                                     projectcalico.org/v3                   false        ClusterInformation
felixconfigurations               felixconfig,felixconfigs                        projectcalico.org/v3                   false        FelixConfiguration
globalnetworkpolicies             gnp,cgnp,calicoglobalnetworkpolicies            projectcalico.org/v3                   false        GlobalNetworkPolicy
globalnetworksets                                                                 projectcalico.org/v3                   false        GlobalNetworkSet
hostendpoints                     hep,heps                                        projectcalico.org/v3                   false        HostEndpoint
ippools                                                                           projectcalico.org/v3                   false        IPPool
ipreservations                                                                    projectcalico.org/v3                   false        IPReservation
kubecontrollersconfigurations                                                     projectcalico.org/v3                   false        KubeControllersConfiguration
networkpolicies                   cnp,caliconetworkpolicy,caliconetworkpolicies   projectcalico.org/v3                   true         NetworkPolicy
networksets                       netsets                                         projectcalico.org/v3                   true         NetworkSet
profiles                                                                          projectcalico.org/v3                   false        Profile
```

这些api资源是属于calico的，因此不建议使用kubectl来管理，推荐按照calicoctl来管理这些api资源。 将calicoctl安装为kubectl的插件:

```bash
cd /usr/local/bin
curl -o kubectl-calico -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.21.5/calicoctl-linux-amd64" 
chmod +x kubectl-calico
```

验证插件正常工作:

```bash
kubectl calico -h
```

## 2.4 部署Network组件kube-ovn

安装install.sh之前确认

```bash
--pod-network-cidr=10.16.0.0/16 
--service-cidr=10.96.0.0/12
```

要和install.sh里面的POD_CIDR、SVC_CIDR一致

```bash
POD_CIDR="10.16.0.0/16"                # Do NOT overlap with NODE/SVC/JOIN CIDR
POD_GATEWAY="10.16.0.1"
SVC_CIDR="10.96.0.0/12"                # Do NOT overlap with NODE/POD/JOIN CIDR
JOIN_CIDR="100.64.0.0/16"              # Do NOT overlap with NODE/POD/SVC CIDR
PINGER_EXTERNAL_ADDRESS="114.114.114.114"  # Pinger check external ip probe
PINGER_EXTERNAL_DOMAIN="alauda.cn"         # Pinger check external domain probe
SVC_YAML_IPFAMILYPOLICY=""
```

执行安装脚本

```bash
wget https://raw.githubusercontent.com/kubeovn/kube-ovn/master/dist/images/install.sh -o install.sh
sh install.sh
```



## 2.5 验证k8s DNS是否可用

```bash
kubectl run curl --image=radial/busyboxplus:curl -it
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$
```

进入后执行`nslookup kubernetes.default`确认解析正常:

```fallback
nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

## 2.6 添加Node节点

下面将node2, node3添加到Kubernetes集群中，分别在node01, node02, node03上执行:

```bash
kubeadm join 192.168.10.20:6443 --token uufqmm.bvtfj4drwfvvbcev \
	--discovery-token-ca-cert-hash sha256:5814415567d93f6d2d41fe4719be8221f45c29c482b5059aec2e27a832ac36e6
```

node01, node02, node03加入集群很是顺利，在master节点上执行命令查看集群中的节点：

```bash
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES           AGE    VERSION
master   Ready    control-plane   7h3m   v1.24.3
node01   Ready    <none>          7h2m   v1.24.3
node02   Ready    <none>          7h2m   v1.24.3
node03   Ready    <none>          7h2m   v1.24.3
```

# Multus Networking



# redis operrator

```
kubectl  create ns operator-redis-ibm
kubectl  apply -f ../redis-rbac.yaml

helm install op charts/operator-for-redis --wait --set image.repository=ibmcom/operator-for-redis --set image.tag=latest  -n operator-redis-ibm
helm install --wait mycluster charts/node-for-redis --set image.repository=ibmcom/node-for-redis --set image.tag=latest -n operator-redis-ibm
```



# 3. ipvs支持

1.启用netfilter模块

```
modprobe br_netfilter
```

2.修改内核参数

```
cat >> /etc/sysctl.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
 
sysctl -p
```

3.安装ipvs支持

```
yum -y install ipvsadm ipset
```

4.启用ipvs模块

```
cat > /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
 
chmod 755 /etc/sysconfig/modules/ipvs.modules
source /etc/sysconfig/modules/ipvs.modules
```

5.修改configmap并重启kube-proxy

```
kubectl edit cm kube-proxy -n kube-system  # mode: "ipvs"
kubectl get pod -n kube-system | grep kube-proxy | awk '{print $1}' | xargs kubectl -n kube-system delete pod
```

6.验证

```
ipvsadm -Ln
```



# 4. Kubernetes常用组件部署

## remoteWrite使能

```yaml
remoteWrite:
- url: http://192.168.10.200:8428/api/v1/write
```

## 4.1 安装Prometheus Grafana

```bash
git clone https://github.com/prometheus-operator/kube-prometheus
kubectl apply --server-side -f kube-prometheus/manifests/setup
kubectl apply --server-side -f kube-prometheus/manifests
```

注意版本对应：

| kube-prometheus stack                                        | Kubernetes 1.21 | Kubernetes 1.22 | Kubernetes 1.23 | Kubernetes 1.24 | Kubernetes 1.25 |
| ------------------------------------------------------------ | --------------- | --------------- | --------------- | --------------- | --------------- |
| [`release-0.11`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.11) | ✗               | ✗               | ✔               | ✔               | ✗               |
| [`release-0.12`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.12) | ✗               | ✗               | ✗               | ✔               | ✔               |
| [`main`](https://github.com/prometheus-operator/kube-prometheus/tree/main) | ✗               | ✗               | ✗               | ✗               | ✔               |



Network Policies会导致节点访问限制：

```python
[root@master manifests]# ls -alrth *networkPolicy*
-rw-r--r-- 1 root root  694 7月  22 23:04 prometheusOperator-networkPolicy.yaml
-rw-r--r-- 1 root root  564 7月  22 23:04 prometheusAdapter-networkPolicy.yaml
-rw-r--r-- 1 root root  671 7月  22 23:04 nodeExporter-networkPolicy.yaml
-rw-r--r-- 1 root root  723 7月  22 23:04 kubeStateMetrics-networkPolicy.yaml
-rw-r--r-- 1 root root  722 7月  22 23:04 blackboxExporter-networkPolicy.yaml
-rw-r--r-- 1 root root  977 7月  22 23:04 alertmanager-networkPolicy.yaml
-rw-r--r-- 1 root root  651 7月  24 10:16 grafana-networkPolicy.yaml
-rw-r--r-- 1 root root 1.1K 7月  24 14:40 prometheus-networkPolicy.yaml
```

## 4.2 安装ingress-nginx

https://github.com/kubernetes/ingress-nginx

默认是，daemonset安装前需要给node打标签edgenode，安装在edgenode上：

```bash
kubectl label nodes node02 edgenode=true
kubectl label nodes node03 edgenode=true
kubectl get node --show-labels
```

执行脚本：

```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.1/deploy/static/provider/cloud/deploy.yaml -O ingress/ingress-nginx-deploy-daemonset.yaml
kubectl apply -f ingress/ingress-nginx-deploy-daemonset.yaml
```

此次采用daemonset安装，需要修改下面两处

```yaml
kind: DaemonSet
```

```yaml
hostNetwork: true
nodeSelector:
  edgenode: 'true'
```

完整配置：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: anjia0532/google-containers.ingress-nginx.controller:v1.3.0
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      hostNetwork: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        edgenode: 'true'
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: ingress-nginx-admission
```

注意版本对应关系：

| Ingress-NGINX version | k8s supported version  | Alpine Version | Nginx Version | Helm Chart Version |
| --------------------- | ---------------------- | -------------- | ------------- | ------------------ |
| **v1.7.1**            | 1.27,1.26, 1.25, 1.24  | 3.17.2         | 1.21.6        | 4.6.*              |
| **v1.7.0**            | 1.26, 1.25, 1.24       | 3.17.2         | 1.21.6        | 4.6.*              |
| **v1.6.4**            | 1.26, 1.25, 1.24, 1.23 | 3.17.0         | 1.21.6        | 4.5.*              |



## 4.3 生成域名证书

```shell
./gen.cert.sh test.dev minio.test.dev
```

## 4.4 导入证书

```shell
kubectl create secret  tls test.dev  --key test.dev.key --cert test.dev.crt -n ns-test
kubectl create secret  tls test.dev  --key test.dev.key --cert test.dev.crt -n monitoring
```

## 4.5 ingress转发prometheus grafana

```shell
kubectl apply -f ingress/ingress-grafana-prometheus-service.yaml
```

## 4.6 ingress转发nginx测试服务
```shell
kubectl create namespace ns-test
kubectl apply -f nginx/nginx-deployment.yaml
kubectl apply -f nginx/nginx-service.yaml
kubectl apply -f ingress/ingress-nginx-service.yaml
```



# 5. 安装bashbord

https://github.com/kubernetes/dashboard/tree/v2.6.0/aio/deploy

```shell
wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl apply -f recommended.yaml

wget https:*//raw.githubusercontent.com/cby-chen/Kubernetes/main/yaml/dashboard-user.yaml*
kubectl apply -f dashboard-user.yaml
```

## 修改为nodePort

```shell
root@k8s-master01:~# kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
service/kubernetes-dashboard edited

root@k8s-master01:~# kubectl get svc kubernetes-dashboard -n kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.96.221.8   <none>        443:32721/TCP   74s
root@k8s-master01:~#
```

## 导入证书

```shell
kubectl create secret  tls test.dev  --key test.dev.key --cert test.dev.crt -n kubernetes-dashboard
```

## 创建token

```shell
kubectl -n kubernetes-dashboard create token admin-user
```

## dashboard-rbac.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:

 - kind: ServiceAccount
   name: admin-user
   namespace: kubernetes-dashboard
```



## ingress-bashbord.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
    - hosts:
      - dashboard.test.dev
      secretName: test.dev
  rules:

  - host: dashboard.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
```



# 6 静态PV

1. 准备一台机器，搭建NFS服务

```
yum install nfs-utils
```

```
echo "/data/k8s/ 192.168.10.0/24(sync,rw,no_root_squash)" > /etc/exports
cat /etc/exports
 
systemctl start nfs; systemctl start rpcbind 
systemctl enable nfs
```

1. 在node节点上测试

```
yum install nfs-utils
showmount -e 192.168.10.128
```

1. 创建pv（master上）


vim pv.yaml //内容如下

```
kubectl create -f pv.yaml
```

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /data/k8s/
    server: 192.168.10.12
```

```
kubectl get pv
[root@master pv]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv001   500Gi      RWX            Retain           Available                                   6m48s
```

创建pvc, pvc.yaml

```
kubectl create -f pvc.yaml
```

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
```

```
[root@master pv]# kubectl get pvc
NAME                      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-redis-cluster-0   Pending                                                     63m
myclaim                   Bound     pv001    500Gi      RWX                           94s
```

# 7 动态pv创建

https://blog.csdn.net/a13568hki/article/details/124351017

https://artifacthub.io/packages/helm/nfs-subdir-external-provisioner/nfs-subdir-external-provisioner

https://fabianlee.org/2022/01/12/kubernetes-nfs-mount-using-dynamic-volume-and-storage-class/

官网：

https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner



## nfs-client-provisioner.yaml

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-provisioner-01
  namespace: kube-system #与RBAC文件中的namespace保持一致
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-provisioner-01
  template:
    metadata:
      labels:
        app: nfs-provisioner-01
    spec:
      serviceAccountName: nfs-client-provisioner # 指定serviceAccount!
      containers:
        - name: nfs-client-provisioner
          image: easzlab/nfs-subdir-external-provisioner:v4.0.2 #镜像地址
          imagePullPolicy: IfNotPresent
          volumeMounts:  # 挂载数据卷到容器指定目录
            - name: nfs-client-root
              mountPath: /persistentvolumes   #不需要修改
          env:
            - name: PROVISIONER_NAME
              value: nfs-provisioner-01  # 此处供应者名字供storageclass调用
            - name: NFS_SERVER
              value: 192.168.10.128   # 填入NFS的地址
            - name: NFS_PATH
              value: /data/k8s   # 填入NFS挂载的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.10.128   # 填入NFS的地址
            path: /data/k8s   # 填入NFS挂载的目录
---
apiVersion: storage.k8s.io/v1
kind: StorageClass  # 创建StorageClass
metadata:
  name: nfs-boge
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  
provisioner: nfs-provisioner-01  #这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
# Supported policies: Delete、 Retain ,default is Delete
reclaimPolicy: Retain   #清除策略
```



```yaml
# nfs-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default        #根据实际环境设定namespace,下面类同
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
    # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io


# nfs-StorageClass.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs-storage #这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
parameters:
#  archiveOnDelete: "false"
  archiveOnDelete: "true"
reclaimPolicy: Retain


# nfs-provisioner.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default  #与RBAC文件中的namespace保持一致
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: docker.io/easzlab/nfs-subdir-external-provisioner:v4.0.2
          #image: quay.io/external_storage/nfs-client-provisioner:latest
          #这里特别注意，在k8s-1.20以后版本中使用上面提供的包，并不好用，这里我折腾了好久，才解决，后来在官方的github上，别人提的问题中建议使用下面这个包才解决的，我这里是下载后，传到我自已的仓库里
          #easzlab/nfs-subdir-external-provisioner:v4.0.1 
          #image: registry-op.test.cn/nfs-subdir-external-provisioner:v4.0.1
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-storage  #provisioner名称,请确保该名称与 nfs-StorageClass.yaml文件中的provisioner名称保持一致
            - name: NFS_SERVER
              value: 192.168.10.220   #NFS Server IP地址
            - name: NFS_PATH
              value: "/home/nfs"    #NFS挂载卷
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.10.220  #NFS Server IP地址
            path: "/home/nfs"     #NFS 挂载卷
```





# 8 Minio安装

## 安装minio  operator

```shell
wget https://github.com/minio/operator/releases/download/v4.4.28/kubectl-minio_4.4.28_linux_amd64 -O kubectl-minio
chmod +x kubectl-minio
mv kubectl-minio /usr/local/bin/
kubectl minio init
```

## 控制界面

```
kubectl minio proxy -n minio-operator
```

## 创建minio集群

```shell
kubectl minio tenant create tenant1 \
--servers 1 \
--volumes 4 \
--capacity 1Gi \
--namespace tenant1
--storage-class local-path
```

## 生成3级域名证书

```
gen.cert.sh minio.test.dev
```

## 导入证书

```
kubectl create secret  tls test.dev  --key test.dev.key --cert test.dev.crt -n tenant1
```

## ingress-minio.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-minio
  namespace: tenant1
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
    - hosts:
      - minio.test.dev
      - apiminio.test.dev
      secretName: test.dev
    - hosts:
      - web.minio.test.dev
      - api.minio.test.dev
      secretName: minio.test.dev
  rules:
  - host: minio.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant1-console
            port:
              number: 9443
  - host: web.minio.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant1-console
            port:
              number: 9443
  - host: apiminio.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio
            port:
              number: 443
  - host: api.minio.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio
            port:
              number: 443
```

## 配置域名解析

```
192.168.10.22 test.dev
192.168.10.23 web.minio.test.dev
192.168.10.22 minio.test.dev
192.168.10.23 api.minio.test.dev
192.168.10.22 apiminio.test.dev
192.168.10.22 www.test.dev
192.168.10.22 grafana.test.dev
192.168.10.23 prometheus.test.dev
192.168.10.23 dashboard.test.dev
```



## 413 Request Entity Too Large

### 方案1 修改自己的ingress

nginx.ingress.kubernetes.io/proxy-body-size: 10m

有些后端自带TLS证书这里忽略了后端的证书

```yaml
metadata:
  name: ingress-minio
  namespace: tenant1
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
```

### 方案2 全局 ingress configmap

proxy-body-size: 50000M

```yaml
apiVersion: v1
data:
  allow-snippet-annotations: "true"
  proxy-body-size: 50000M
kind: ConfigMap
```



# 6 kube_proxy service和iptables的关系

service 的代理是 kube-proxy

kube-proxy 运行在所有节点上，它监听 apiserver 中 service 和 endpoint 的变化情况，创建路由规则以提供服务 IP 和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上，而kube-proxy底层又是通过iptables和ipvs实现的

### iptables 模式

iptables 是一个 Linux 内核功能，是一个高效的防火墙，并提供了大量的数据包处理和过滤方面的能力。它可以在核心数据包处理管线上用 Hook 挂接一系列的规则。iptables 模式中 kube-proxy 在 `NAT pre-routing` Hook 中实现它的 NAT 和负载均衡功能。这种方法简单有效，依赖于成熟的内核功能，并且能够和其它跟 iptables 协作的应用（例如 Calico）融洽相处。

然而 kube-proxy 的用法是一种 O(n) 算法，其中的 n 随集群规模同步增长，这里的集群规模，更明确的说就是服务和后端 Pod 的数量。

### ipvs 模式

IPVS 是一个用于负载均衡的 Linux 内核功能。IPVS 模式下，kube-proxy 使用 IPVS 负载均衡代替了 iptable。这种模式同样有效，IPVS 的设计就是用来为大量服务进行负载均衡的，它有一套优化过的 API，使用优化的查找算法，而不是简单的从列表中查找规则。

这样一来，kube-proxy 在 IPVS 模式下，其连接过程的复杂度为 O(1)。换句话说，多数情况下，他的连接处理效率是和集群规模无关的。

另外作为一个独立的[负载均衡器](https://cloud.tencent.com/product/clb?from=10680)，IPVS 包含了多种不同的负载均衡算法，例如轮询、最短期望延迟、最少连接以及各种哈希方法等。而 iptables 就只有一种随机平等的选择算法。

IPVS 的一个潜在缺点就是，IPVS 处理数据包的路径和通常情况下 iptables 过滤器的路径是不同的。如果计划在有其他程序使用 iptables 的环境中使用 IPVS，需要进行一些研究，看看他们是否能够协调工作。（Calico 已经和 IPVS kube-proxy 兼容）



### kube-proxy ipvs和iptables的异同

iptables与IPVS都是基于Netfilter实现的，但因为定位不同，二者有着本质的差别：iptables是为防火墙而设计的；IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张。

与iptables相比，IPVS拥有以下明显优势：

- 为大型集群提供了更好的可扩展性和性能；
- 支持比iptables更复杂的复制均衡算法（最小负载、最少连接、加权等）；
- 支持服务器健康检查和连接重试等功能；
- 可以动态修改ipset的集合，即使iptables的规则正在使用这个集合。



# Service

NodePort



LoadBalancer







# 7 Calico BGP VRF

https://www.cnblogs.com/linwenye/p/13269871.html

https://larioy.gst.monster/2021/09/05/k8s-ji-chong-cni-fang-an-jie-xi/calico/ensp-mo-ni-calico-kua-wang-duan-bgp-wang-luo/

https://www.cnblogs.com/Cylon/p/14399682.html



**Full-mesh**

全互联模式，启用了 BGP 之后，Calico 的默认行为是在每个节点彼此对等的情况下创建完整的内部 BGP（iBGP）连接，这使 Calico 可以在任何 L2 网络（无论是公有云还是私有云）上运行，或者说（如果配了 IPIP）可以在任何不禁止 IPIP 流量的网络上作为 overlay 运行。对于 vxlan overlay，Calico 不使用 BGP。

Full-mesh 模式对于 100 个以内的工作节点或更少节点的中小规模部署非常有用，但是在较大的规模上，Full-mesh 模式效率会降低，较大规模情况下，Calico 官方建议使用 Route reflectors

**Route reflectors**

如果想构建内部 BGP（iBGP）大规模集群，可以使用 BGP 路由反射器来减少每个节点上使用 BGP 对等体的数量。在此模型中，某些节点充当路由反射器，并配置为在它们之间建立完整的网格。然后，将其他节点配置为与这些路由反射器的子集（通常为冗余，通常为 2 个）进行对等，从而与全网格相比减少了 BGP 对等连接的总数。

**Top of Rack (ToR)**

在本地部署中，可以将 Calico 配置为直接与物理网络基础结构对等。通常，这需要涉及到禁用 Calico 的默认 Full-mesh 行为，将所有 Calico 节点与 L3 ToR 路由器对等。



## spine  Leaf  RR



最近我司业务扩展在机房新开了一个区域，折腾了一段时间的 Calico BGP，为了能将整个过程梳理得更简单明了，我还是决定将这个过程记录下来。不管是对当下的总结还是未来重新审视方案都是值得的。大家都知道，云原生下的网络架构在 Kubernetes 里可以算是百花齐放，各有所长，这无形中也导致网络始终是横在广大 K8S 爱好者面前迈向高阶管理的几座大山之一。通常大家在公有云上使用厂家提供的 CNI 组件可能还感受不到其复杂，但一旦要在 IDC 自建集群时，就会面临 Kubernetes 网络架构选型的问题。 Calico 作为目前 Kubernetes 上用途最广的 Kubernetes CNI 之一，自然也有很多追随者。而本篇便是在自建机房内 BGP 组网下的一次总结。

## 关于 CNI 选型

云原生 CNI 组件这么多，诸如老牌的 Flannel、Calico、WeaveNet、Kube-Router 以及近两年兴起的 Antrea、 Kube-OVN 和 Cilium。它们都在各自的场景下各有所长，在选择前我们可以先对其做一个简单的功能性的比较:

|              | Flannel   | Calico          | Cilium     | WeaveNet   | Antrea    | Kube-OVN        |
| ------------ | --------- | --------------- | ---------- | ---------- | --------- | --------------- |
| 部署模式     | DaemonSet | DaemonSet       | DaemonSet  | DaemonSet  | DaemonSet | DaemonSet       |
| 包封装与路由 | VxLAN     | IPinIP,BGP,eBPF | VxLAN,eBPF | VxLAN      | Vxlan     | Vlan/Geneve/BGP |
| 网络策略     | No        | Yes             | Yes        | Yes        | Yes       | Yes             |
| 存储引擎     | Etcd      | Etcd            | Etcd       | No         | Etcd      | Etcd            |
| 传输加密     | Yes       | Yes             | Yes        | Yes        | Yes       | No              |
| 运营模式     | 社区      | Tigera          | 社区       | WeaveWorks | VMware    | 灵雀云          |

## 为什么选择 Calico

在对我司机房新区域的网络选型上，我们最终选择了 Calico 作为云端网络解决方案，在这里我们简单阐述下为什么选择 Calico 的几点原因：

- 支持 BGP 广播，Calico 通过 BGP 协议广播路由信息，且架构非常简单。在 kubernetes 可以很容易的实现 mesh to mesh 或者 RR 模式，在后期如果要实现容器跨集群网络通信时，实现也很容易。
- Calico配置简单，且配置都是通过 Kubernetes 中的 CRD 来进行集中式的管理，通过操作 CR 资源，我们可以直接对集群内的容器进行组网和配置的实时变更
- 丰富的功能及其兼容性，考虑到集群内需要与三方应用兼容，例如配置多租户网络、固定容器 IP 、网络策略等功能又或者与 Istio、MetalLB、Cilium 等组件的兼容，Calico 的的表现都非常不错
- 高性能， Calico 的数据面采用 HostGW 的方式，由于是一个纯三方的数据通信，所以在实际使用下性能和主机资源占用方面不会太差，至少也能排在第一梯队

结合我司机房新区域采购的是 H3C S9 系列的交换机，支持直接在接入层的交换机侧开启路由反射器。所以最终我们选择Calico 并以 BGP RR 的模式作为 Kubernetes 的 CNI 组件便水到渠成。

## 关于 MetalLB

在讲 MetalLB 之前，先回顾下应用部署在 Kubernetes 中，它的下游服务是如何访问的吧。通常有如下几种情况

- 集群内请求:

直接通过 Kubernetes 的 Service 访问应用。

- 集群外请求：

1. 通过 NodePort 在主机上以 nat 方式将流量转发给容器，优点配置简单且能提供简单的负载均衡功能， 缺点也很明显下游应用只能通过主机地址+端口来做寻址
2. 通过 Ingress-nginx 做应用层的 7 层转发，优点是路由规则灵活，且流量只经过一层代理便直达容器，效率较高。缺点是 ingress-nginx 本身的服务还是需要通过 NodePort 或者 HostNetwork 来支持

可以看到在没有外部负载均衡器的引入之前，应用部署在 kubernetes 集群内，它对南北向流量的地址寻址仍然不太友好。也许有的同学就说了，我在公有云上使用 Kubernetes 时，将 Service 类型设置成 LoadBalancer，集群就能自动为我的应用创建一条带负载均衡器地址的 IP供外部服务调用，那我们自己部署的 Kubernetes 集群有没有类似的东西来实现呢？

当然有！MetalLB 就是在裸金属服务器下为 Kubernetes 集群诞生的一个负载均衡器项目。

```
事实上当然不止 MetalLB，开源界里面还有其他诸如PureLB、OpenELB等负载均衡产品。不过本文旨在采用 BGP 协议来实现负载均衡，所以重点会偏向 MetelLB
```

简单来说，MetalLB包含了两个组件，Controler用于操作 Service 资源的变更以及IPAM。Speaker用于外广播地址以及 BGP 的连接。它支持两种流模式模式即：layer2 和 BGP。

- Layer2 模式

又叫ARP/NDP模式，在此模式下，Kubenretes集群中运行 Speaker 的一台机器通过 leader 选举，获取 Service 的 LoadBalancer IP 的所有权，并使用 ARP 协议将其 IP 和 MAC 广播出去，以使这些 IP 能够在本地网络上可访问。由此可见，使用 Layer2 的模式对现有网络并没有太多的要求，甚至不需要路由器的支持。不过缺点也显而易见，LoadBalancer IP 所在的 Node 节点承载了所有的流量，会产生一定的网络瓶颈。

```
此外，我们可以简单的将 Layer2 模式理解为与 Keepalived 原理相似，区别仅为 Layer2 的lead 选举并不是使用 VRRP 组播来通信的
```

- BGP 模式

MabelLB 在 BGP 模式下，集群中的所有运行 Speaker 的主机都将与上层交换机建立一条BGP 连接，并广播其 LoadBalancer 的IP 地址。优点是真正的实现了网络负载均衡，缺点就是配置相对而言要复杂许多，且需上层路由器支持BGP。



本文主要讲述在传统的自建数据中心，利用 Calico 和 MetalLB 来组件内部的 BGP 网络。并以此来为 Kubernetes 提供 `Pod - Pod` 、`Node - Pod` 和 `Node - Loadbalancer`网络互访的能力。



# 8 Metallb(Layer2 BGP) VRF

https://blog.csdn.net/wanger5354/article/details/124964489

https://metallb.universe.tf/configuration/

## INSTALLATION

Before starting with installation, make sure you meet all the [requirements](https://metallb.universe.tf/#requirements). In particular, you should pay attention to [network addon compatibility](https://metallb.universe.tf/installation/network-addons/).

If you’re trying to run MetalLB on a cloud platform, you should also look at the [cloud compatibility](https://metallb.universe.tf/installation/clouds/) page and make sure your cloud platform can work with MetalLB (most cannot).

There are three supported ways to install MetalLB: using plain Kubernetes manifests, using Kustomize, or using Helm.

### Preparation

If you’re using kube-proxy in IPVS mode, since Kubernetes v1.14.2 you have to enable strict ARP mode.

*Note, you don’t need this if you’re using kube-router as service-proxy because it is enabling strict ARP by default.*

You can achieve this by editing kube-proxy config in current cluster:

```bash
kubectl edit configmap -n kube-system kube-proxy
```

and set:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

You can also add this configuration snippet to your kubeadm-config, just append it with `---` after the main configuration.

If you are trying to automate this change, these shell snippets may help you:

```bash
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### Installation By Manifest

To install MetalLB, apply the manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.4/config/manifests/metallb-native.yaml
```

If you want to deploy MetalLB using the [experimental FRR mode](https://metallb.universe.tf/configuration/#enabling-bfd-support-for-bgp-sessions), apply the manifests:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.4/config/manifests/metallb-frr.yaml
```

Please do note that these manifests deploy MetalLB from the main development branch. We highly encourage cloud operators to deploy a stable released version of MetalLB on production environments!

This will deploy MetalLB to your cluster, under the `metallb-system` namespace. The components in the manifest are:

- The `metallb-system/controller` deployment. This is the cluster-wide controller that handles IP address assignments.
- The `metallb-system/speaker` daemonset. This is the component that speaks the protocol(s) of your choice to make the services reachable.
- Service accounts for the controller and speaker, along with the RBAC permissions that the components need to function.

The installation manifest does not include a configuration file. MetalLB’s components will still start, but will remain idle until you [start deploying resources](https://metallb.universe.tf/configuration/).

### Installation With Kustomize

You can install MetalLB with [Kustomize](https://github.com/kubernetes-sigs/kustomize) by pointing at the remote kustomization file.

In the following example, we are deploying MetalLB with the native bgp implementation :

```yaml
# kustomization.yml
namespace: metallb-system

resources:
  - github.com/metallb/metallb/config/native?ref=v0.13.4
```

In order to deploy the [experimental FRR mode](https://metallb.universe.tf/configuration/#enabling-bfd-support-for-bgp-sessions):

```yaml
# kustomization.yml
namespace: metallb-system

resources:
  - github.com/metallb/metallb/config/frr?ref=v0.13.4
```

### Installation With Helm

You can install MetallLB with [Helm](https://helm.sh/) by using the Helm chart repository: `https://metallb.github.io/metallb`

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb
```

A values file may be specified on installation. This is recommended for providing configs in Helm values:

```bash
helm install metallb metallb/metallb -f values.yaml
```

The speaker pod requires elevated permission in order to perform its network functionalities.

If you are using MetalLB with a kubernetes version that enforces [pod security admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) (which is beta in k8s 1.23), the namespace MetalLB is deployed to must be labelled with:

```yaml
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

If you want to deploy MetalLB using the [experimental FRR mode](https://metallb.universe.tf/configuration/#enabling-bfd-support-for-bgp-sessions), the following value must be set:

```yaml
speaker:
  frr:
    enabled: true
```

### Using The MetalLB Operator

The MetalLB Operator is available on OperatorHub at [operatorhub.io/operator/metallb-operator](https://operatorhub.io/operator/metallb-operator). It eases the deployment and life-cycle of MetalLB in a cluster and allows configuring MetalLB via CRDs.

If you want to deploy MetalLB using the [experimental FRR mode](https://metallb.universe.tf/configuration/#enabling-bfd-support-for-bgp-sessions), you must edit the ClusterServiceVersion resource named `metallb-operator`:

```bash
kubectl edit csv metallb-operator
```

and change the `BGP_TYPE` environment variable of the `manager` container to `frr`:

```yaml
- name: METALLB_BGP_TYPE
  value: frr
```

### FRR Daemons Logging Level

The FRR daemons logging level are configured using the speaker `--log-level` argument following the below mapping:

| Speaker log level | FRR log level |
| :---------------- | :------------ |
| all, debug        | debugging     |
| info              | informational |
| warn              | warnings      |
| error             | error         |
| none              | emergencies   |

To override this behavior, you can set the `FRR_LOGGING_LEVEL` speaker’s environment to any [FRR supported value](https://docs.frrouting.org/en/latest/basic.html#clicmd-log-stdout-LEVEL).

### Upgrade

When upgrading MetalLB, always check the [release notes](https://metallb.universe.tf/release-notes/) to see the changes and required actions, if any. Pay special attention to the release notes when upgrading to newer major/minor releases.

Unless specified otherwise in the release notes, upgrade MetalLB either using [plain manifests](https://metallb.universe.tf/installation/#installation-by-manifest) or using [Kustomize](https://metallb.universe.tf/installation/#installation-with-kustomize) as described above.

Please take the known limitations for [layer2](https://metallb.universe.tf/concepts/layer2/#limitations) and [bgp](https://metallb.universe.tf/concepts/bgp/#limitations) into account when performing an upgrade.

### Setting The LoadBalancer Class

MetalLB supports [LoadBalancerClass](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-class), which allows multiple load balancer implementations to co-exist. In order to set the loadbalancer class MetalLB should be listening for, the `--lb-class=<CLASS_NAME>` parameter must be provided to both the speaker and the controller.

The helm charts support it via the `loadBalancerClass` parameter.



## CONFIGURATION

MetalLB remains idle until configured. This is accomplished by creating and deploying various resources into **the same namespace** (metallb-system) MetalLB is deployed into.

There are various examples of the configuration CRs in [`configsamples`](https://raw.githubusercontent.com/metallb/metallb/v0.13.4/configsamples).

Also, the API is [fully documented here](https://metallb.universe.tf/apis/_index.md).

```
If you installed MetalLB with Helm, you will need to change the namespace of the CRs to match the namespace in which MetalLB was deployed.
```



### Defining The IPs To Assign To The Load Balancer Services

In order to assign an IP to the services, MetalLB must be instructed to do so via the `IPAddressPool` CR.

All the IPs allocated via `IPAddressPool`s contribute to the pool of IPs that MetalLB uses to assign IPs to services.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.0/24
  - 192.168.9.1-192.168.9.5
  - fc00:f853:0ccd:e799::/124
```

Multiple instances of `IPAddressPool`s can co-exist and addresses can be defined by CIDR, by range, and both IPV4 and IPV6 addresses can be assigned.

### Announce The Service IPs

Once the IPs are assigned to a service, they must be announced.

The specific configuration depends on the protocol(s) you want to use to announce service IPs. Jump to:

- [Layer 2 configuration](https://metallb.universe.tf/configuration/#layer-2-configuration)
- [BGP configuration](https://metallb.universe.tf/configuration/#bgp-configuration)
- [Advanced BGP configuration](https://metallb.universe.tf/configuration/_advanced_bgp_configuration)
- [Advanced L2 configuration](https://metallb.universe.tf/configuration/_advanced_l2_configuration)
- [Advanced IPAddressPool configuration](https://metallb.universe.tf/configuration/_advanced_ipaddresspool_config/)

Note: it is possible to announce the same service both via L2 and via BGP (see the relative [FAQ](https://metallb.universe.tf/faq/_index.md)).

### Layer 2 Configuration

Layer 2 mode is the simplest to configure: in many cases, you don’t need any protocol-specific configuration, only IP addresses.

Layer 2 mode does not require the IPs to be bound to the network interfaces of your worker nodes. It works by responding to ARP requests on your local network directly, to give the machine’s MAC address to clients.

In order to advertise the IP coming from an `IPAddressPool`, an `L2Advertisement` instance must be associated to the `IPAddressPool`.

For example, the following configuration gives MetalLB control over IPs from `192.168.1.240` to `192.168.1.250`, and configures Layer 2 mode:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

Setting no `IPAddressPool` selector in an `L2Advertisement` instance is interpreted as that instance being associated to all the `IPAddressPool`s available.

So in case there are specialized `IPAddressPool`s, and only some of them must be advertised via L2, the list of `IPAddressPool`s we want to advertise the IPs from must be declared (alternative, a label selector can be used).

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

### BGP Configuration

MetalLB needs to be instructed on how to establish a session with one or more external BGP routers.

In order to do so, an instance of `BGPPeer` must be created for each router we want metallb to connect to.

For a basic configuration featuring one BGP router and one IP address range, you need 4 pieces of information:

- The router IP address that MetalLB should connect to,
- The router’s AS number,
- The AS number MetalLB should use,
- An IP address range expressed as a CIDR prefix.

As an example if you want to give MetalLB AS number 64500, and connect it to a router at 10.0.0.1 with AS number 64501, your configuration will look like:

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: sample
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.0.0.1
```

Given an `IPAddressPool` like:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```

MetalLB must be configured to advertise the IPs coming from it via BGP.

This is done via the `BGPAdvertisement` CR.

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example
  namespace: metallb-system
```

Setting no `IPAddressPool` selector in a `BGPAdvertisement` instance is interpreted as that instance being associated to all the `IPAddressPool`s available.

So in case there are specialized `AddressPool`s, and only some of them must be advertised via BGP, the list of `ipAddressPool`s we want to advertise the IPs from must be declared (alternative, a label selector can be used).

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

#### Enabling BFD support for BGP sessions

With the experimental FRR mode, BGP sessions can be backed up by BFD sessions in order to provide a quicker path failure detection than BGP alone provides.

In order to enable BFD, a BFD profile must be added and referenced by a given peer:

```yaml
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata:
  name: testbfdprofile
  namespace: metallb-system
spec:
  receiveInterval: 380
  transmitInterval: 270
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: peersample
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64512
  peerAddress: 172.30.0.3
  bfdProfile: testbfdprofile
```

### Configuration Validation

MetalLB ships validation webhooks that check the validity of the CRs applied.

However, due to the fact that the global MetalLB configuration is composed by different pieces, not all of the invalid configurations are blocked by those webhooks. Because of that, if a non valid MetalLB configuration is applied, MetalLB discards it and keeps using the last valid configuration.

In future releases MetalLB will expose misconfigurations as part of Kubernetes resources, but currently the only way to understand why the configuration was not loaded is by checking the controller’s logs.









# 6. cri-o容器查看

- 告警消除：

```
crictl config runtime-endpoint unix:///run/containerd/containerd.sock
crictl config image-endpoint unix:///run/containerd/containerd.sock
```

- 容器查看：

```ini
[root@master ~]# crictl ps -a
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
05bf0b07de6d2       85c4a186478c5       27 minutes ago      Running             cni-server                1                   4e02c14191216
dc44549c2fb1c       85c4a186478c5       27 minutes ago      Running             kube-ovn-monitor          1                   af7f061cc99da
baecb15218ce4       7ecf17eb27602       27 minutes ago      Running             kube-rbac-proxy           1                   5b4a069503baa
ba5e1dfb820d2       1dbe0e9319764       27 minutes ago      Running             node-exporter             1                   5b4a069503baa
076ee4abc2236       2ae1ba6417cbc       27 minutes ago      Running             kube-proxy                1                   fcdec13137e9c
e9694c055e958       85c4a186478c5       27 minutes ago      Running             ovn-central               1                   6e97a806daa56
87f8bf7b461ee       85c4a186478c5       27 minutes ago      Running             openvswitch               1                   090c5b465fdfd
3947ea767ae98       85c4a186478c5       27 minutes ago      Exited              install-cni               1                   4e02c14191216
dedd5885b4364       d521dd763e2e3       27 minutes ago      Running             kube-apiserver            3                   52fe731b64d15
93ecf96257381       aebe758cef4cd       27 minutes ago      Running             etcd                      3                   f8fb9d0fddf59
189c8028b1b5f       586c112956dfc       27 minutes ago      Running             kube-controller-manager   3                   0cead8b71f125
da982f9d8504e       3a5aa3a515f5d       27 minutes ago      Running             kube-scheduler            4                   697f03b8343df
7795018ed9846       7ecf17eb27602       7 hours ago         Exited              kube-rbac-proxy           0                   eb7bbd6b1a3ae
52c6309afc865       1dbe0e9319764       7 hours ago         Exited              node-exporter             0                   eb7bbd6b1a3ae
a159a4a2cd630       85c4a186478c5       8 hours ago         Exited              cni-server                0                   ec18d4951aa24
311fb8d9d0915       85c4a186478c5       8 hours ago         Exited              kube-ovn-monitor          0                   29e23e19475f7
4994420dd49cd       85c4a186478c5       8 hours ago         Exited              openvswitch               0                   6aaa39b688767
316775dc2b0d5       85c4a186478c5       8 hours ago         Exited              ovn-central               0                   c557b57114e97
4d0c8a21aa3e2       2ae1ba6417cbc       8 hours ago         Exited              kube-proxy                0                   c2c23e8e506c4
32a274d36691e       3a5aa3a515f5d       8 hours ago         Exited              kube-scheduler            3                   35d60f66a07aa
018b4015ebb34       aebe758cef4cd       8 hours ago         Exited              etcd                      2                   21485439d6399
c4d4d3a7e0f99       d521dd763e2e3       8 hours ago         Exited              kube-apiserver            2                   6eb81ac0c0503
a9eeb491d46f9       586c112956dfc       8 hours ago         Exited              kube-controller-manager   2                   5802cc6895738
```







# 7. **k8s删除名称空间报错**

**一直处于Terminating状态中**

- 问题

删除ns，一直处于Terminating状态中
强制删除也是出现报错

原因：因为ingress controller的镜像 pull 失败，一直在 retry ，所以我就把 ingress-controller delete 掉，但是一直卡住在删除 namespace 阶段 Ctrl + c

```ini
[root@master1 ingress]# kubectl create -f mandatory.yaml
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
Error from server (AlreadyExists): error when creating "mandatory.yaml": object is being deleted: namespaces "ingress-ngin                                                               x" already exists
Error from server (Forbidden): error when creating "mandatory.yaml": configmaps "nginx-configuration" is forbidden: unable                                                                to create new content in namespace ingress-nginx because it is being terminated
Error from server (Forbidden): error when creating "mandatory.yaml": configmaps "tcp-services" is forbidden: unable to cre                                                               ate new content in namespace ingress-nginx because it is being terminated
Error from server (Forbidden): error when creating "mandatory.yaml": configmaps "udp-services" is forbidden: unable to cre                                                               ate new content in namespace ingress-nginx because it is being terminated
Error from server (Forbidden): error when creating "mandatory.yaml": serviceaccounts "nginx-ingress-serviceaccount" is for                                                               bidden: unable to create new content in namespace ingress-nginx because it is being terminated
Error from server (Forbidden): error when creating "mandatory.yaml": roles.rbac.authorization.k8s.io "nginx-ingress-role"                                                                is forbidden: unable to create new content in namespace ingress-nginx because it is being terminated
Error from server (Forbidden): error when creating "mandatory.yaml": rolebindings.rbac.authorization.k8s.io "nginx-ingress                                                               -role-nisa-binding" is forbidden: unable to create new content in namespace ingress-nginx because it is being terminated

Error from server (Forbidden): error when creating "mandatory.yaml": daemonsets.apps "nginx-ingress-controller" is forbidd                                                               en: unable to create new content in namespace ingress-nginx because it is being terminated
```



- 解决方案：

```
kubectl get ns ingress-nginx -o json > tmp.json
```

导出运行的名称空间至json文件，删掉其中的spec字段`内容`，因为k8s集群是携带认证的

```shell
[root@master1 ingress]# kubectl proxy --port=8081

# 新开一个shell终端执行curl命令
[root@master1 ~]# curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8081/api/v1/namespaces/ingress-nginx/finalize
```

- ok

```
[root@master1 ingress]# kubectl get ns
NAME                   STATUS        AGE
default                Active        7d20h
kube-node-lease        Active        7d20h
kube-public            Active        7d20h
kube-system            Active        7d20h
kubernetes-dashboard   Terminating   7d14h
```



# 8. k8s常用命令

## **查看所有** **pod** **列表,  -n** **后跟** **namespace,** **查看指定的命名空间**

```javascript
kubectl get pod 
kubectl get pod -n kube-system    #查看指定命名空间的pod
kubectl get pod -o wide    #查看更详细的信息，比如pod所在节点
kubectl get pod --show-labels    #获取pod并查看pod的标签
```

## **查看** **RC** **和** **service** **列表，** **-o wide** **查看详细信息**

```javascript
kubectl get rc,svc
kubectl get pod,svc -o wide
kubectl get pod <pod name> -o yaml
```

## **显示** **Node** **的详细信息**

```javascript
kubectl describe node 192.168.0.212 #可以跟Node IP或者主机名
```

## **显示** **Pod** **的详细信息,** **特别是查看** **pod** **无法创建的时候的日志**

```javascript
kubectl describe pod <pod-name>
eg:
kubectl describe pod redis-master-tqds9
```

## **根据** **yaml** **创建资源, apply** **可以重复执行，create** **不行**

```javascript
kubectl create -f pod.yaml
kubectl apply -f pod.yaml
```

## **基于 pod.yaml** **定义的名称删除指定资源**

```javascript
kubectl delete -f pod.yaml 
```

## **删除所有包含某个** **label** **的pod** **和** **service**

```javascript
kubectl delete pod,svc -l name=<label-name>
```

## **删除默认命名空间下的所有** **Pod**

```javascript
kubectl delete pod --all
```

## **执行** **pod** **命令**

```javascript
kubectl exec <pod-name> -- date
kubectl exec <pod-name> -- bash
kubectl exec <pod-name> -- ping 10.24.51.9
```

## **通过bash获得** **pod** **中某个**[**容器**](https://cloud.tencent.com/product/tke?from=10680)**的TTY，相当于登录容器**

```javascript
kubectl exec -it <pod-name> -c <container-name> -- bash
eg:
kubectl exec -it redis-master-cln81 -- bash
```

## **查看容器的日志**

```javascript
kubectl logs <pod-name>
kubectl logs -f <pod-name> # 实时查看日志
kubectl log  <pod-name>  -c <container_name> # 若 pod 只有一个容器，可以不加 -c 

kubectl logs -l app=frontend # 返回所有标记为 app=frontend 的 pod 的合并日志。
```

## **查看节点** **labels**

```javascript
kubectl get node --show-labels
```

## **重启** **pod**

```javascript
kubectl get pod <POD名称> -n <NAMESPACE名称> -o yaml | kubectl replace --force -f -
```

## **创建命令**

```javascript
kubectl apply -f ./my-manifest.yaml           # 创建资源
kubectl apply -f ./my1.yaml -f ./my2.yaml     # 使用多个文件创建
kubectl apply -f ./dir                        # 基于目录下的所有清单文件创建资源
kubectl apply -f https://git.io/vPieo         # 从 URL 中创建资源
kubectl create deployment nginx --image=nginx # 启动单实例 nginx
kubectl explain pods,svc                      # 获取 pod 清单的文档说明

# 从标准输入创建多个 YAML 对象
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000"
EOF

# 创建有多个 key 的 Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo -n "s33msi4" | base64 -w0)
  username: $(echo -n "jane" | base64 -w0)
EOF
```

## **查看和查找资源**

```javascript
# get 命令的基本输出
kubectl get services                          # 列出当前命名空间下的所有 services
kubectl get pods --all-namespaces             # 列出所有命名空间下的全部的 Pods
kubectl get pods -o wide                      # 列出当前命名空间下的全部 Pods，并显示更详细的信息
kubectl get deployment my-dep                 # 列出某个特定的 Deployment
kubectl get pods                              # 列出当前命名空间下的全部 Pods
kubectl get pod my-pod -o yaml                # 获取一个 pod 的 YAML

# describe 命令的详细输出
kubectl describe nodes my-node
kubectl describe pods my-pod

# 列出当前名字空间下所有 Services，按名称排序
kubectl get services --sort-by=.metadata.name

# 列出 Pods，按重启次数排序
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 列举所有 PV 持久卷，按容量排序
kubectl get pv --sort-by=.spec.capacity.storage

# 获取包含 app=cassandra 标签的所有 Pods 的 version 标签
kubectl get pods --selector=app=cassandra -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 获取所有工作节点（使用选择器以排除标签名称为 'node-role.kubernetes.io/master' 的结果）
kubectl get node --selector='!node-role.kubernetes.io/master'

# 获取当前命名空间中正在运行的 Pods
kubectl get pods --field-selector=status.phase=Running

# 获取全部节点的 ExternalIP 地址
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个特定 RC 的 Pods 的名称
# 在转换对于 jsonpath 过于复杂的场合，"jq" 命令很有用；可以在 https://stedolan.github.io/jq/ 找到它。
sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 显示所有 Pods 的标签（或任何其他支持标签的 Kubernetes 对象）
kubectl get pods --show-labels

# 检查哪些节点处于就绪状态
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# 列出被一个 Pod 使用的全部 Secret
kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq
```



```javascript
# 列举所有 Pods 中初始化容器的容器 ID（containerID）
# Helpful when cleaning up stopped containers, while avoiding removal of initContainers.
kubectl get pods --all-namespaces -o jsonpath='{range .items[*].status.initContainerStatuses[*]}{.containerID}{"\n"}{end}' | cut -d/ -f3

# 列出事件（Events），按时间戳排序
kubectl get events --sort-by=.metadata.creationTimestamp

# 比较当前的集群状态和假定某清单被应用之后的集群状态
kubectl diff -f ./my-manifest.yaml
```

## **更新资源**

```javascript
kubectl set image deployment/frontend www=image:v2               # 滚动更新 "frontend" Deployment 的 "www" 容器镜像
kubectl rollout history deployment/frontend                      # 检查 Deployment 的历史记录，包括版本 
kubectl rollout undo deployment/frontend                         # 回滚到上次部署版本
kubectl rollout undo deployment/frontend --to-revision=2         # 回滚到特定部署版本
kubectl rollout status -w deployment/frontend                    # 监视 "frontend" Deployment 的滚动升级状态直到完成
kubectl rollout restart deployment/frontend                      # 轮替重启 "frontend" Deployment

cat pod.json | kubectl replace -f -                              # 通过传入到标准输入的 JSON 来替换 Pod

# 强制替换，删除后重建资源。会导致服务不可用。
kubectl replace --force -f ./pod.json

# 为多副本的 nginx 创建服务，使用 80 端口提供服务，连接到容器的 8000 端口。
kubectl expose rc nginx --port=80 --target-port=8000

# 将某单容器 Pod 的镜像版本（标签）更新到 v4
kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

kubectl label pods my-pod new-label=awesome                      # 添加标签
kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
kubectl autoscale deployment foo --min=2 --max=10                # 对 "foo" Deployment 自动伸缩容
```

## **部分更新资源**

```javascript
# 部分更新某节点
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}' 

# 更新容器的镜像；spec.containers[*].name 是必须的。因为它是一个合并性质的主键。
kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'

# 使用带位置数组的 JSON patch 更新容器的镜像
kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'

# 使用带位置数组的 JSON patch 禁用某 Deployment 的 livenessProbe
kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'

# 在带位置数组中添加元素 
kubectl patch sa default --type='json' -p='[{"op": "add", "path": "/secrets/1", "value": {"name": "whatever" } }]'
```

## **删除资源**

```javascript
kubectl delete -f ./pod.json                                              # 删除在 pod.json 中指定的类型和名称的 Pod
kubectl delete pod,service baz foo                                        # 删除名称为 "baz" 和 "foo" 的 Pod 和服务
kubectl delete pods,services -l name=myLabel                              # 删除包含 name=myLabel 标签的 pods 和服务
kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除包含 label name=myLabel 标签的 Pods 和服务
kubectl -n my-ns delete po,svc --all                                      # 删除在 my-ns 名字空间中全部的 Pods 和服务
# 删除所有与 pattern1 或 pattern2 awk 模式匹配的 Pods
kubectl get pods  -n mynamespace --no-headers=true | awk '/pattern1|pattern2/{print $1}' | xargs  kubectl delete -n mynamespace pod
```

## **Pod常用操作**

```javascript
kubectl logs my-pod                                 # 获取 pod 日志（标准输出）
kubectl logs -l name=myLabel                        # 获取含 name=myLabel 标签的 Pods 的日志（标准输出）
kubectl logs my-pod --previous                      # 获取上个容器实例的 pod 日志（标准输出）
kubectl logs my-pod -c my-container                 # 获取 Pod 容器的日志（标准输出, 多容器场景）
kubectl logs -l name=myLabel -c my-container        # 获取含 name=myLabel 标签的 Pod 容器日志（标准输出, 多容器场景）
kubectl logs my-pod -c my-container --previous      # 获取 Pod 中某容器的上个实例的日志（标准输出, 多容器场景）
kubectl logs -f my-pod                              # 流式输出 Pod 的日志（标准输出）
kubectl logs -f my-pod -c my-container              # 流式输出 Pod 容器的日志（标准输出, 多容器场景）
kubectl logs -f -l name=myLabel --all-containers    # 流式输出含 name=myLabel 标签的 Pod 的所有日志（标准输出）
kubectl run -i --tty busybox --image=busybox -- sh  # 以交互式 Shell 运行 Pod
kubectl run nginx --image=nginx -n mynamespace      # 在指定名字空间中运行 nginx Pod
kubectl run nginx --image=nginx                     # 运行 ngins Pod 并将其规约写入到名为 pod.yaml 的文件
  --dry-run=client -o yaml > pod.yaml

kubectl attach my-pod -i                            # 挂接到一个运行的容器中
kubectl port-forward my-pod 5000:6000               # 在本地计算机上侦听端口 5000 并转发到 my-pod 上的端口 6000
kubectl exec my-pod -- ls /                         # 在已有的 Pod 中运行命令（单容器场景）
kubectl exec my-pod -c my-container -- ls /         # 在已有的 Pod 中运行命令（多容器场景）
kubectl top pod POD_NAME --containers               # 显示给定 Pod 和其中容器的监控数据
```

## **节点操作**

```javascript
kubectl cordon my-node                                                # 标记 my-node 节点为不可调度
kubectl drain my-node                                                 # 对 my-node 节点进行清空操作，为节点维护做准备
kubectl uncordon my-node                                              # 标记 my-node 节点为可以调度
kubectl top node my-node                                              # 显示给定节点的度量值
kubectl cluster-info                                                  # 显示主控节点和服务的地址
kubectl cluster-info dump                                             # 将当前集群状态转储到标准输出
kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state

# 如果已存在具有指定键和效果的污点，则替换其值为指定值
kubectl taint nodes foo dedicated=special-user:NoSchedule
```

**格式化输出**

要以特定格式将详细信息输出到终端窗口，可以将 -o 或 --output 参数添加到支持的 kubectl 命令

| 输出格式                          | 描述                                                         |
| :-------------------------------- | :----------------------------------------------------------- |
| -o=custom-columns=<spec>          | 使用逗号分隔的自定义列来打印表格                             |
| -o=custom-columns-file=<filename> | 使用 <filename> 文件中的自定义列模板打印表格                 |
| -o=json                           | 输出 JSON 格式的 API 对象                                    |
| -o=jsonpath=<template>            | 打印 jsonpath 表达式中定义的字段                             |
| -o=jsonpath-file=<filename>       | 打印在 <filename> 文件中定义的 jsonpath 表达式所指定的字段。 |
| -o=name                           | 仅打印资源名称而不打印其他内容                               |
| -o=wide                           | 以纯文本格式输出额外信息，对于 Pod 来说，输出中包含了节点名称 |
| -o=yaml                           | 输出 YAML 格式的 API 对象                                    |

使用 -o=custom-columns 的示例：

```javascript
# 集群中运行着的所有镜像
kubectl get pods -A -o=custom-columns='DATA:spec.containers[*].image'

 # 除 "k8s.gcr.io/coredns:1.6.2" 之外的所有镜像
kubectl get pods -A -o=custom-columns='DATA:spec.containers[?(@.image!="k8s.gcr.io/coredns:1.6.2")].image'

# 输出 metadata 下面的所有字段，无论 Pod 名字为何
kubectl get pods -A -o=custom-columns='DATA:metadata.*'
```



# ndots search域

```
[root@centos-76f4b54bb5-tph8f /]# cat /etc/resolv.conf    
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```



ndots修改

```
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```



# 附件：

### ingress-nginx-service.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-service
  namespace: ns-test
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:

   - hosts:
     - www.test.dev
     - test.dev
       secretName: tls-secret
       rules:

  - host: www.test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: test.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```



### ingress-grafana-prometheus-service.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-monitoring-service
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:
    - hosts:
      - grafana.test.dev
      - alertmanager.test.dev
      - prometheus.test.dev
      secretName: tls-secret
  rules:
    - host: foobar.com
      http: &http_rules
        paths:
        - backend:
            serviceName: foobar
            servicePort: 80
    - host: api.foobar.com
      http: *http_rules
    - host: admin.foobar.com
      http: *http_rules
    - host: status.foobar.com
      http: *http_rules

    - host: prometheus.test.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-k8s
                port:
                   number: 9090
    - host: alertmanager.test.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: alertmanager-main
                port:
                   number: 9093
    - host: grafana.test.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                   number: 3000


```

### nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: ns-test
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

### nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: ns-test
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:

  - protocol: TCP
    port: 80
    targetPort: 80
```



### nginx-minio.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-minio
  namespace: tenant
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
spec:
  tls:
    - hosts:
      - minio.test.dev
      - apiminio.test.dev
      secretName: test.dev
    - hosts:
      - web.minio.test.dev
      - api.minio.test.dev
      secretName: minio.test.dev
  rules:
  - host: minio.test.dev
    http: &web_minio
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant1-console
            port:
              number: 9443
  - host: web.minio.test.dev
    http: *web_minio
  - host: apiminio.test.dev
    http: &api_minio
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio
            port:
              number: 443
  - host: api.minio.test.dev
    http: *api_minio

```



### redis - rbac.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: operator-redis-ibm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: operator-redis-ibm
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
```







### ingress-nginx-deploy-daemonset.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx

  name: ingress-nginx
---

apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx

  namespace: ingress-nginx
---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission

  namespace: ingress-nginx
---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx
  namespace: ingress-nginx
rules:

- apiGroups:
  - ""
    resources:
  - namespaces
    verbs:
  - get
- apiGroups:
  - ""
    resources:
  - configmaps
  - pods
  - secrets
  - endpoints
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
    resources:
  - services
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
    resources:
  - ingresses
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
    resources:
  - ingresses/status
    verbs:
  - update
- apiGroups:
  - networking.k8s.io
    resources:
  - ingressclasses
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
    resourceNames:
  - ingress-controller-leader
    resources:
  - configmaps
    verbs:
  - get
  - update
- apiGroups:
  - ""
    resources:
  - configmaps
    verbs:
  - create
- apiGroups:
  - coordination.k8s.io
    resourceNames:
  - ingress-controller-leader
    resources:
  - leases
    verbs:
  - get
  - update
- apiGroups:
  - coordination.k8s.io
    resources:
  - leases
    verbs:
  - create
- apiGroups:
  - ""
    resources:
  - events
    verbs:
  - create
  - patch

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission
  namespace: ingress-nginx
rules:

- apiGroups:
  - ""
    resources:
  - secrets
    verbs:
  - get
  - create

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx
rules:

- apiGroups:
  - ""
    resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  - namespaces
    verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
    resources:
  - leases
    verbs:
  - list
  - watch
- apiGroups:
  - ""
    resources:
  - nodes
    verbs:
  - get
- apiGroups:
  - ""
    resources:
  - services
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
    resources:
  - ingresses
    verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
    resources:
  - events
    verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
    resources:
  - ingresses/status
    verbs:
  - update
- apiGroups:
  - networking.k8s.io
    resources:
  - ingressclasses
    verbs:
  - get
  - list
  - watch

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission
rules:

- apiGroups:
  - admissionregistration.k8s.io
    resources:
  - validatingwebhookconfigurations
    verbs:
  - get
  - update

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:

- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:

- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:

- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:

- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx

---

apiVersion: v1
data:
  allow-snippet-annotations: "true"
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-controller

  namespace: ingress-nginx
---

apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  ports:

  - appProtocol: http
    name: http
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: https
      selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
      type: LoadBalancer

---

apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  ports:

  - appProtocol: https
    name: https-webhook
    port: 443
    targetPort: webhook
      selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
      type: ClusterIP

---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: anjia0532/google-containers.ingress-nginx.controller:v1.3.0
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      hostNetwork: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        edgenode: 'true'
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:

          secretName: ingress-nginx-admission
---

apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission-create
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.3.0
      name: ingress-nginx-admission-create
    spec:
      containers:
      - args:
        - create
        - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=ingress-nginx-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: anjia0532/google-containers.ingress-nginx.kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: create
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000

      serviceAccountName: ingress-nginx-admission
---

apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission-patch
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.3.0
      name: ingress-nginx-admission-patch
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=ingress-nginx-admission
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=ingress-nginx-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: anjia0532/google-containers.ingress-nginx.kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: patch
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000

      serviceAccountName: ingress-nginx-admission
---

apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: nginx
spec:

  controller: k8s.io/ingress-nginx
---

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
  name: ingress-nginx-admission
webhooks:

- admissionReviewVersions:
  - v1
    clientConfig:
    service:
      name: ingress-nginx-controller-admission
      namespace: ingress-nginx
      path: /networking/v1/ingresses
    failurePolicy: Fail
    matchPolicy: Equivalent
    name: validate.nginx.ingress.kubernetes.io
    rules:
  - apiGroups:
    - networking.k8s.io
      apiVersions:
    - v1
      operations:
    - CREATE
    - UPDATE
      resources:
    - ingresses
      sideEffects: None
```

