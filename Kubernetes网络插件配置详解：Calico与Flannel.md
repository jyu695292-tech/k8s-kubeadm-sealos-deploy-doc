# Kubernetes 网络插件配置详解：Calico 与 Flannel

> 本文面向正在学习 Kubernetes 集群网络的读者，介绍 CNI 的基本工作方式，并以 **Calico** 和 **Flannel** 为例，完成网络插件安装、网段配置、连通性验证和常见故障排查。
>
> **文章定位：**学习笔记、实验环境指南和配置原理记录。
>
> **重要提醒：**本文命令会修改集群网络配置。请先在测试集群验证，生产环境应固定插件版本、评估兼容性、准备回滚方案，并参考对应项目的官方文档。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) [Kubernetes 集群网络文档](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

## 一、本文将解决什么问题

Kubernetes 能够调度 Pod，但并不会单独完成所有 Pod 网络配置。集群需要安装一个符合 CNI 规范的网络插件，由它为 Pod 分配 IP、创建网络接口、配置路由或隧道，并在节点之间转发 Pod 流量。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

完成本文后，你将能够：

- 理解 Kubernetes 中 Pod、Service、Node 三类网络地址的关系；
- 理解 CNI、CNI 配置文件、IPAM 和跨节点转发的基本概念；
- 在 kubeadm 集群中安装 Calico 或 Flannel；
- 验证 Pod-to-Pod、Pod-to-Service 和 DNS 连通性；
- 使用 Calico NetworkPolicy 编写一个最小网络隔离示例；
- 根据学习、测试和生产需求选择网络插件。

本文不会在同一个集群中同时安装 Calico 和 Flannel。**一个集群应明确选择一个主要 CNI 方案**，否则可能出现 CNI 配置冲突、多个 DaemonSet 同时接管节点网络等问题。

## 二、目录

- [一、本文将解决什么问题](#一本文将解决什么问题)
- [二、目录](#二目录)
- [三、先理解 Kubernetes 网络模型](#三先理解-kubernetes-网络模型)
- [四、CNI 到底负责什么](#四cni-到底负责什么)
- [五、Calico 与 Flannel 对比](#五calico-与-flannel-对比)
- [六、安装前的网段规划](#六安装前的网段规划)
- [七、安装前检查](#七安装前检查)
- [八、方案一：安装 Calico](#八方案一安装-calico)
- [九、方案二：安装 Flannel](#九方案二安装-flannel)
- [十、网络插件安装后的验证](#十网络插件安装后的验证)
- [十一、使用 Calico NetworkPolicy 实现访问控制](#十一使用-calico-networkpolicy-实现访问控制)
- [十二、常见故障排查](#十二常见故障排查)
- [十三、如何选择 Calico 或 Flannel](#十三如何选择-calico-或-flannel)
- [十四、升级、切换与清理注意事项](#十四升级切换与清理注意事项)
- [十五、学习路线建议](#十五学习路线建议)
- [参考资料](#参考资料)

## 三、先理解 Kubernetes 网络模型

Kubernetes 网络问题可以拆成四个层次：容器之间的通信、Pod 到 Pod 的通信、Pod 到 Service 的通信，以及集群外部到 Service 的通信。[Kubernetes 集群网络文档](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

| 通信类型 | 典型场景 | 主要相关组件 |
| --- | --- | --- |
| 容器到容器 | 同一个 Pod 内的两个容器通过 `localhost` 通信 | Pod network namespace |
| Pod 到 Pod | 一个 Pod 访问另一个 Pod 的 IP | CNI、路由、隧道或底层网络 |
| Pod 到 Service | Pod 访问 `ClusterIP` 或 Service DNS | kube-proxy、Service、CoreDNS、CNI |
| 外部到 Service | 用户访问 NodePort 或 LoadBalancer | Service、kube-proxy、CNI、负载均衡器 |

Kubernetes 网络规划至少需要考虑三类地址范围：

| 地址范围 | 用途 | 配置位置 |
| --- | --- | --- |
| Node 网段 | 节点操作系统和节点间通信 | 云平台、物理网络或虚拟网络 |
| Pod 网段 | 分配给 Pod 的地址 | kubeadm 的 `--pod-network-cidr`、CNI 配置 |
| Service 网段 | 分配给 Service 的虚拟地址 | kube-apiserver 的 `--service-cluster-ip-range` |

这三类网段必须避免重叠。Kubernetes 官方文档也明确指出，Pod、Service 和 Node 的地址范围需要从不冲突的地址空间中规划。[Kubernetes 集群网络文档](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

例如，本文实验环境可以这样规划：

```text
Node 网段：    192.168.8.0/24
Pod 网段：     10.244.0.0/16
Service 网段： 172.16.0.0/16
```

> **最容易犯的错误：**初始化集群时使用了 `10.244.0.0/16`，安装 CNI 时却使用了其他 Pod 网段。网段不一致通常会导致 CNI Pod 启动异常、节点长期 `NotReady` 或 Pod 之间无法通信。

## 四、CNI 到底负责什么

CNI 是 Container Network Interface 的缩写，是一组用于配置和清理容器网络的规范与插件。Kubernetes 通过容器运行时调用 CNI 插件，为 Pod sandbox 创建网络接口、分配 IP，并配置必要的路由和网络规则。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

在一个常见的 containerd + Kubernetes 环境中，网络相关文件通常位于：

```text
/etc/cni/net.d/   # CNI 配置文件
/opt/cni/bin/     # CNI 插件二进制文件
```

不同发行版和容器运行时可能使用不同路径，实际环境应以 containerd、CRI-O 或发行版配置为准。Kubernetes 还要求容器运行时提供 loopback CNI，用于 Pod sandbox 的 `lo` 接口。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

一个 CNI 方案通常包含以下几个部分：

| 部分 | 作用 |
| --- | --- |
| 主 CNI 插件 | 创建 Pod 的网络接口并接入集群网络 |
| IPAM | 为 Pod 分配和回收 IP 地址 |
| 跨节点后端 | 通过路由、VXLAN、IP-in-IP 或其他方式转发跨节点流量 |
| CNI 配置 | 告诉运行时该调用哪些插件以及插件参数 |
| 控制器或 DaemonSet | 在节点上安装配置，并持续维护网络状态 |
| NetworkPolicy 实现 | 按标签和规则允许或拒绝 Pod 流量，具体取决于 CNI 能力 |

## 五、Calico 与 Flannel 对比

### 5.1 Calico

Calico 是一个功能较丰富的 Kubernetes 网络方案。它可以使用路由或隧道方式连接节点，并提供 Kubernetes NetworkPolicy 以及 Calico 自定义策略等能力。Calico 官方文档还提供 IPPool、网络策略和流量观测等配置说明。[Calico 官方快速开始](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

Calico 更适合以下情况：

- 需要控制 Pod 入站和出站流量；
- 需要按 Namespace、Label、Service 或 CIDR 编写网络策略；
- 希望学习 Kubernetes 网络策略和网络安全；
- 集群规模、网络拓扑或安全需求较复杂。

### 5.2 Flannel

Flannel 的定位更偏向于简单、易理解的 Pod 网络连通性。它在每个节点运行 `flanneld`，从预先配置的网络地址空间中为节点分配子网租约，并通过 VXLAN 等后端转发跨节点流量。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

Flannel 更适合以下情况：

- 个人实验室或课程学习；
- 主要目标是快速让 Pod 跨节点通信；
- 集群规模不大，网络策略需求较少；
- 希望先理解 Pod 网段、节点子网和 VXLAN 等基础概念。

Flannel 项目本身主要关注网络连通性，原生不负责完整的 NetworkPolicy 执行。需要策略控制时，应使用 Flannel 官方支持的策略组件或选择具备策略能力的 CNI 方案。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

### 5.3 选型速览

| 对比项 | Calico | Flannel |
| --- | --- | --- |
| 学习难度 | 中等 | 较低 |
| 基础 Pod 网络 | 支持 | 支持 |
| NetworkPolicy | 支持，能力较丰富 | 通常需要额外策略组件或组合方案 |
| 配置复杂度 | 较高 | 较低 |
| 适合初学者 | 适合深入学习 | 适合快速入门 |
| 典型定位 | 网络与安全策略 | 简单的集群网络 fabric |
| 生产使用 | 需要按版本和拓扑验证 | 需要评估策略、性能和运维需求 |

## 六、安装前的网段规划

### 6.1 kubeadm 初始化时指定 Pod 网段

如果使用 kubeadm，初始化控制平面时需要指定 Pod 网段。例如：

```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=172.16.0.0/16 \
  --control-plane-endpoint=192.168.8.30:6443
```

上面的 `10.244.0.0/16` 只是示例。Flannel 官方 manifest 默认也常使用该网段；如果使用其他网段，就必须修改 Flannel 配置或 Helm 参数以保持一致。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

### 6.2 检查现有集群配置

如果集群已经初始化，可以先检查 kube-controller-manager 和节点 PodCIDR：

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDR:.spec.podCIDR
kubectl -n kube-system get pods -o wide
```

检查 Service 网段：

```bash
kubectl cluster-info dump | grep -m1 service-cluster-ip-range || true
```

不同安装方式的参数位置可能不同。不要只根据一条命令的输出判断网段，必要时同时检查 kube-apiserver、controller-manager 和 CNI manifest。

## 七、安装前检查

### 7.1 确认节点和 kubelet 正常

```bash
kubectl get nodes -o wide
systemctl status kubelet --no-pager
systemctl status containerd --no-pager
```

如果 kubelet 或 containerd 已经异常，先解决运行时问题，再安装 CNI。网络插件无法修复一个未正常运行的容器运行时。

### 7.2 确认内核模块和转发参数

Flannel 官方文档特别指出，`br_netfilter` 对其启动很重要；在较新的 kubeadm 版本中，相关检查可能不会替你完成，因此建议手工确认。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

```bash
lsmod | grep br_netfilter
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

必要时加载模块并刷新参数：

```bash
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
```

### 7.3 确认没有残留 CNI 配置

如果这是一个重装过网络插件的实验集群，可以检查：

```bash
ls -la /etc/cni/net.d/
ls -la /opt/cni/bin/
```

不要在生产环境直接删除 `/etc/cni/net.d/`。如果确定是全新实验集群并且需要清理旧配置，应先备份：

```bash
mkdir -p /root/cni-backup
cp -a /etc/cni/net.d/. /root/cni-backup/ 2>/dev/null || true
```

## 八、方案一：安装 Calico

Calico 官方当前推荐的快速开始流程包含安装 CRD、Tigera Operator 和 Calico 自定义资源。下面使用固定版本变量示例，发布文章或执行部署前应确认该版本与 Kubernetes 版本兼容。[Calico 官方快速开始](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

### 8.1 设置版本变量

```bash
export CALICO_VERSION=v3.32.1
```

> 版本号是示例。不要把 `latest` 当作生产环境的版本策略；生产环境应固定版本，并在升级前阅读兼容性说明。

### 8.2 安装 Calico CRD 与 Operator

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/v1_crd_projectcalico_org.yaml

kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml
```

检查 Operator：

```bash
kubectl get pods -n tigera-operator
kubectl get crd | grep -E 'projectcalico|tigera'
```

### 8.3 创建 Calico 自定义资源

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml
```

查看 Calico 组件：

```bash
kubectl get pods -n calico-system -o wide
kubectl get tigerastatus
```

如果集群的 Pod 网段不是官方 manifest 中的默认值，需要下载 `custom-resources.yaml`，检查其中的 Installation 和 IPPool 配置，再按照实际网段修改后应用：

```bash
curl -LO \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml

vi custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

### 8.4 使用 Calico IPPool

Calico 使用 IPPool 为工作负载分配 Pod IP。可以查看当前 IPPool：

```bash
kubectl get ippools.crd.projectcalico.org -o yaml
```

学习环境中常见的 IPPool 片段如下：

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  encapsulation: VXLAN
  natOutgoing: Enabled
  nodeSelector: all()
```

这里的 `cidr` 必须与集群规划一致。`encapsulation`、`natOutgoing` 和节点选择器应根据实际网络拓扑配置，不建议在不了解数据路径的情况下直接复制到生产环境。

## 九、方案二：安装 Flannel

Flannel 官方提供基于 manifest 和 Helm 的部署方式。下面先介绍 manifest 方式，因为它更适合初学者观察 Kubernetes 资源对象和 DaemonSet。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

### 9.1 使用官方 manifest 安装

对于默认使用 `10.244.0.0/16` 的实验集群，可以执行：

```bash
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

查看 Flannel 资源：

```bash
kubectl get ns kube-flannel
kubectl get pods -n kube-flannel -o wide
kubectl get ds -n kube-flannel
```

如果 kubeadm 初始化时使用了自定义 Pod 网段，需要先下载并检查 manifest：

```bash
curl -L \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml \
  -o kube-flannel.yml

grep -nE 'Network|podCidr|10\.244' kube-flannel.yml
```

根据 manifest 中的配置结构修改网络地址，再应用：

```bash
vi kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

### 9.2 使用 Helm 安装

Flannel 官方也提供 Helm Chart。先创建 Namespace 并设置特权 Pod Security 标签：

```bash
kubectl create namespace kube-flannel
kubectl label --overwrite namespace kube-flannel \
  pod-security.kubernetes.io/enforce=privileged
```

添加仓库并安装：

```bash
helm repo add flannel https://flannel-io.github.io/flannel/
helm repo update

helm install flannel flannel/flannel \
  --namespace kube-flannel \
  --set podCidr="10.244.0.0/16"
```

查看 Helm 状态：

```bash
helm list -n kube-flannel
helm status flannel -n kube-flannel
```

> manifest 和 Helm 二选一即可。不要在同一个集群中重复安装两套 Flannel 资源。

### 9.3 Flannel 后端与防火墙

Flannel 可以通过 VXLAN 等后端转发节点间流量。实际需要放行的端口取决于所选 backend、操作系统和网络环境。配置防火墙时应以当前 Flannel 版本的 backend 文档为准，而不是机械地复制某个教程中的端口。[Flannel 官方仓库与部署说明](https://github.com/flannel-io/flannel)

查看节点上的 Flannel 日志：

```bash
kubectl logs -n kube-flannel -l app=flannel --tail=100
```

如果标签选择器与当前版本不匹配，可以先查看 Pod 名称：

```bash
kubectl get pods -n kube-flannel --show-labels
kubectl logs -n kube-flannel <flannel-pod-name> --tail=100
```

## 十、网络插件安装后的验证

### 10.1 验证节点 Ready

```bash
kubectl get nodes -o wide
```

所有节点最终应为 `Ready`。如果 CNI Pod 已经运行但节点仍然 `NotReady`，继续检查节点 Conditions 和 kubelet 日志：

```bash
kubectl describe node <node-name>
journalctl -u kubelet -n 100 --no-pager
```

### 10.2 验证 Pod IP 与跨节点分布

```bash
kubectl get pods -A -o wide
```

重点观察：

- Pod 是否获得了非空 IP；
- Pod 是否分布在不同节点；
- CNI DaemonSet 是否在每个节点都有一个正常 Pod；
- CNI Pod 是否出现 `CrashLoopBackOff` 或反复重启。

### 10.3 验证 Pod-to-Pod

创建两个测试 Pod：

```bash
kubectl create namespace network-test

kubectl -n network-test run web \
  --image=nginx:stable \
  --restart=Never

kubectl -n network-test run client \
  --image=curlimages/curl:latest \
  --restart=Never \
  --command -- sleep 3600
```

等待 Pod 就绪：

```bash
kubectl wait -n network-test --for=condition=Ready pod/web --timeout=120s
kubectl wait -n network-test --for=condition=Ready pod/client --timeout=120s
```

获取 web Pod IP，并从 client 访问：

```bash
WEB_IP=$(kubectl -n network-test get pod web -o jsonpath='{.status.podIP}')
kubectl -n network-test exec client -- curl -sS --max-time 5 "http://${WEB_IP}"
```

如果能够返回 NGINX 页面，说明 Pod-to-Pod 基础连通性正常。

### 10.4 验证 Service 与 DNS

```bash
kubectl -n network-test expose pod web --port=80 --name=web
kubectl -n network-test get service web
kubectl -n network-test exec client -- curl -sS --max-time 5 http://web
```

验证集群 DNS：

```bash
kubectl -n network-test exec client -- nslookup web.network-test.svc.cluster.local
```

测试完成后清理：

```bash
kubectl delete namespace network-test
```

### 10.5 查看事件和 CNI 日志

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pods -A -o wide
```

Calico：

```bash
kubectl get pods -n calico-system -o wide
kubectl get tigerastatus
```

Flannel：

```bash
kubectl get pods -n kube-flannel -o wide
kubectl get ds -n kube-flannel
```

## 十一、使用 Calico NetworkPolicy 实现访问控制

Kubernetes NetworkPolicy 依赖网络插件实际执行。仅仅创建 NetworkPolicy 对象，并不保证所有 CNI 都会执行相同的策略；部署前必须确认所选插件的策略能力。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) [Calico 官方快速开始](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

下面创建一个只允许同 Namespace 客户端访问 `web` 的示例。

首先部署应用：

```bash
kubectl create namespace policy-demo

kubectl -n policy-demo create deployment web \
  --image=nginx:stable

kubectl -n policy-demo expose deployment web --port=80
```

创建允许策略：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-web
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 80
```

保存为 `allow-client-to-web.yaml` 并应用：

```bash
kubectl apply -f allow-client-to-web.yaml
kubectl get networkpolicy -n policy-demo
```

创建带有 `role=client` 标签的测试 Pod：

```bash
kubectl -n policy-demo run client \
  --image=curlimages/curl:latest \
  --labels=role=client \
  --restart=Never \
  --command -- sleep 3600
```

NetworkPolicy 是“默认拒绝后逐步放行”的安全模型。实际业务中还需要显式允许 DNS、监控、日志和必要的跨 Namespace 流量，否则应用可能出现“网络策略生效，但业务也无法正常运行”的情况。

清理实验资源：

```bash
kubectl delete namespace policy-demo
rm -f allow-client-to-web.yaml
```

## 十二、常见故障排查

| 现象 | 常见原因 | 排查命令 |
| --- | --- | --- |
| 节点一直 `NotReady` | CNI 未安装、CNI Pod 异常、网段不一致 | `kubectl describe node`、`kubectl get pods -A` |
| CNI Pod `CrashLoopBackOff` | 内核模块缺失、权限、镜像拉取或配置错误 | `kubectl logs`、`journalctl -u kubelet` |
| Pod 没有 IP | CNI 配置未生成、IPAM 失败、CNI 二进制缺失 | `ls /etc/cni/net.d`、`ls /opt/cni/bin` |
| 同节点 Pod 通信正常，跨节点失败 | VXLAN、路由、主机防火墙或云安全组问题 | 检查 backend、路由表和节点间端口 |
| Pod 能访问 IP，但不能访问域名 | CoreDNS、Service、网络策略或上游 DNS 异常 | `kubectl get pods -n kube-system`、`nslookup` |
| Service 不通但 Pod IP 可通 | kube-proxy、Service selector 或服务端口错误 | `kubectl describe service`、`kubectl get endpointslices` |
| NetworkPolicy 不生效 | CNI 不执行策略、selector 写错或策略范围不对 | `kubectl describe networkpolicy`、检查 CNI 能力 |
| 安装第二个 CNI 后网络混乱 | 多套 CNI 同时管理节点 | 备份后清理旧 CNI，必要时重建实验集群 |

### 12.1 先看资源状态，不要先重启

建议按照以下顺序排查：

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

然后查看相关组件日志：

```bash
kubectl logs -n kube-system <pod-name> --tail=200
journalctl -u kubelet -n 200 --no-pager
journalctl -u containerd -n 200 --no-pager
```

### 12.2 检查 CNI 文件是否存在

```bash
find /etc/cni/net.d -maxdepth 1 -type f -print
find /opt/cni/bin -maxdepth 1 -type f -print | head
```

如果 `/etc/cni/net.d` 为空，通常说明 CNI DaemonSet 尚未在该节点生成配置。如果 `/opt/cni/bin` 中缺少所需插件，则需要检查容器运行时和网络插件安装流程。

### 12.3 检查节点路由和接口

```bash
ip addr
ip route
ip link
```

Flannel VXLAN 环境中可以重点观察 VXLAN 接口和相关路由；Calico 环境中则需要结合其封装模式、路由或 BGP 配置检查数据路径。

## 十三、如何选择 Calico 或 Flannel

如果你的目标是第一次让 kubeadm 集群运行起来，Flannel 是比较容易理解的起点。它把重点放在节点子网分配和跨节点流量转发上，适合先建立 Pod 网络的整体概念。

如果你的目标是学习 Kubernetes 网络安全、NetworkPolicy、IPPool、封装模式和更复杂的网络治理，Calico 更值得深入。它的配置对象和能力更多，同时也意味着需要理解更多网络概念。

| 目标 | 建议 |
| --- | --- |
| 课程实验、快速验证 Pod 网络 | Flannel |
| 学习 NetworkPolicy | Calico |
| 需要更细粒度网络隔离 | Calico，或评估其他策略能力更强的 CNI |
| 追求最少配置 | Flannel |
| 多集群、复杂网络、企业生产 | 结合网络拓扑、性能、安全和支持周期进行专项评估 |

没有一个 CNI 能够在所有环境中自动成为最佳选择。选型时应同时考虑节点网络、云平台安全组、IP 地址规划、可观测性、策略能力、升级方式和团队运维经验。

## 十四、升级、切换与清理注意事项

### 14.1 不要直接在线替换生产 CNI

CNI 负责整个集群的 Pod 网络。直接删除旧 CNI、立即应用新 CNI 可能导致现有 Pod 网络中断，甚至影响控制面和 DNS。生产环境应提前设计：

- 兼容性验证和版本矩阵；
- 节点逐批处理和业务迁移；
- CNI 配置备份；
- 回滚路径和维护窗口；
- 对 Service、DNS、Ingress、NetworkPolicy 的回归测试。

### 14.2 学习环境切换 CNI

如果只是个人实验集群，最安全的方式通常是销毁并重新创建集群，而不是在已经运行大量业务的集群中强行切换：

```bash
# 仅示意，执行前请确认这是实验集群
kubeadm reset -f
```

具体清理步骤还包括删除旧 CNI 资源、清理节点上的 CNI 配置和网络接口，并重新初始化集群。不同 CNI 的清理方式不同，必须参考对应项目的卸载文档。

### 14.3 固定版本和来源

博客示例可以使用版本变量，但生产部署应把 manifest、Helm Chart、镜像和 CRD 版本固定下来，并保存经过审核的 YAML。直接使用 `releases/latest` 便于学习，却可能因为上游更新而导致结果变化。

## 十五、学习路线建议

建议按以下顺序学习 Kubernetes 网络：

1. 先使用 Flannel 搭建一个最小 kubeadm 集群，理解 Pod IP、节点子网和跨节点转发。
2. 使用 `kubectl get pods -A -o wide`、`ip route` 和 `ip addr` 建立 Kubernetes 对象与 Linux 网络之间的联系。
3. 部署 CoreDNS 和 Service，理解 Service IP、EndpointSlice 和 DNS 解析。
4. 更换或重建实验集群，安装 Calico，观察 IPPool、DaemonSet 和节点网络配置。
5. 编写 NetworkPolicy，从允许同 Namespace 通信开始，逐步增加 DNS、Ingress、监控和跨 Namespace 规则。
6. 继续学习 kube-proxy、iptables、IPVS、VXLAN、BGP、NetworkPolicy、Ingress 和 Service Mesh。

## 十六、总结

CNI 网络插件是 Kubernetes 集群能够正常运行的基础设施。Kubernetes 定义了网络模型，但具体的 Pod 网络接口、IPAM、跨节点转发和网络策略执行，需要由兼容的网络插件实现。[Kubernetes CNI 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

Flannel 以简单的三层网络和跨节点连通性为重点，适合快速入门；Calico 提供更丰富的网络和策略能力，适合深入学习网络安全和集群治理。无论选择哪一个插件，都应该先完成网段规划，再严格验证 CNI 版本、Kubernetes 版本、容器运行时、内核模块和防火墙配置。

> **一句话记忆：**Kubernetes 负责调度和网络模型，CNI 负责把这个模型真正落到每一个节点和 Pod 上。

## 参考资料

- [Kubernetes 官方：Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)，介绍 CNI 插件要求、容器运行时与网络插件的关系。
- [Kubernetes 官方：Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)，介绍 Pod、Service、Node 地址范围和集群网络模型。
- [Calico 官方：Quickstart Guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)，介绍 Operator、CRD、IPPool 和网络策略配置。
- [Flannel 官方 GitHub 仓库](https://github.com/flannel-io/flannel)，介绍 Flannel 的工作原理、backend 和 Kubernetes 部署方式。

---

> 本文为 Kubernetes 学习博客文章，示例版本和命令仅用于实验环境。正式使用前，请根据目标 Kubernetes 版本、操作系统、容器运行时和网络拓扑完成验证。
