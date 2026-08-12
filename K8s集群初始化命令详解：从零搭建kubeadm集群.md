# K8s 集群初始化命令详解：从零搭建 kubeadm 集群

> 本文面向 Kubernetes 初学者，记录如何使用 `kubeadm`、`containerd` 和 Calico 搭建一个用于学习和实验的 Kubernetes 集群。
>
> **适用场景：**个人实验室、虚拟机环境、课程实践和技术学习。
>
> **不适用场景：**生产集群、金融或政企高安全环境。生产环境应使用受支持的操作系统、经过验证的 Kubernetes 版本、可靠的镜像仓库、备份方案和高可用控制平面。

## 一、你将在本文完成什么

完成本文后，你将得到一个包含 1 个控制平面节点和若干工作节点的 Kubernetes 集群，并能够理解以下组件之间的关系：

| 组件 | 作用 |
| --- | --- |
| `kubeadm` | 初始化控制平面，并生成节点加入集群所需的配置和命令 |
| `kubelet` | 运行在每个节点上的 Kubernetes 节点代理，负责管理本机 Pod |
| `kubectl` | Kubernetes 命令行客户端，用于查看和管理集群资源 |
| `containerd` | 容器运行时，负责镜像、容器生命周期和容器进程管理 |
| Calico | CNI 网络插件，负责 Pod 网络和网络策略 |
| Sealos | 可选的一键部署工具，用于简化 kubeadm 集群的安装和运维 |

本文分为两条路线：

1. **手工 kubeadm 路线：**适合学习 Kubernetes 的底层安装流程。
2. **Sealos 路线：**适合快速搭建实验集群，但不建议用它替代对 kubeadm、容器运行时和 CNI 的基础理解。

## 目录

- [一、你将在本文完成什么](#一你将在本文完成什么)
- [二、实验环境与版本约定](#二实验环境与版本约定)
- [三、所有节点执行：操作系统预处理](#三所有节点执行操作系统预处理)
- [四、所有节点安装 containerd](#四所有节点安装-containerd)
- [五、所有节点安装 kubeadm、kubelet 和 kubectl](#五所有节点安装-kubeadmkubelet-和-kubectl)
- [六、仅在控制平面节点执行：初始化集群](#六仅在控制平面节点执行初始化集群)
- [七、在工作节点加入集群](#七在工作节点加入集群)
- [八、安装 CNI 网络插件](#八安装-cni-网络插件)
- [九、部署后的验证](#九部署后的验证)
- [十、常见故障排查](#十常见故障排查)
- [十一、学习环境的常用增强配置](#十一学习环境的常用增强配置)
- [十二、可选路线：使用 Sealos 快速部署](#十二可选路线使用-sealos-快速部署)
- [十三、清理实验集群](#十三清理实验集群)
- [十四、完整流程回顾](#十四完整流程回顾)
- [十五、学习路线建议](#十五学习路线建议)
- [十六、安全提醒](#十六安全提醒)
- [参考资料](#参考资料)

## 二、实验环境与版本约定

本文中的命令以 Linux 虚拟机为例，默认使用 root 权限或具有 `sudo` 权限的用户。示例版本采用 Kubernetes `v1.30.0`，你也可以替换为目标环境中经过验证的版本，但必须保证 `kubeadm`、`kubelet`、`kubectl` 和控制平面版本之间兼容。[1]

| 项目 | 示例值 | 说明 |
| --- | --- | --- |
| 控制平面节点 | `192.168.8.30` | 运行 Kubernetes API Server、etcd 等控制面组件 |
| 工作节点 | `192.168.8.31`、`192.168.8.32` | 运行用户业务 Pod |
| Kubernetes 版本 | `v1.30.0` | 学习示例版本，发布前请确认版本支持周期 |
| Pod 网段 | `10.244.0.0/16` | 必须与 CNI 配置一致，且不能与现有网络冲突 |
| Service 网段 | `172.16.0.0/16` | 集群内部 Service 的虚拟 IP 地址池 |
| 控制平面端口 | `6443` | Kubernetes API Server 默认端口 |
| 容器运行时 | `containerd` | Kubernetes 推荐的 CRI 运行时之一 |

> **重要：**请将文中的 IP、版本号、镜像仓库和网段替换成你的实际环境。Pod 网段、Service 网段一旦用于初始化集群，后续修改会涉及集群级迁移，不要直接照抄。

### 2.1 节点要求

所有节点都需要满足以下条件：

- 节点之间网络互通，并且主机名唯一。
- 节点时间同步正常，DNS 或 `/etc/hosts` 能够解析节点名称。
- 所有节点使用兼容的 Linux 内核、容器运行时和 Kubernetes 版本。
- 控制平面节点至少准备 2 个 CPU 和 2 GiB 内存；学习环境建议为控制平面准备 4 GiB 或更多内存。
- 防火墙必须按照 Kubernetes 端口要求放行。为了方便实验可以暂时关闭，但生产环境不应简单关闭防火墙。[2]
- 不要在公网直接暴露 API Server、etcd 或 SSH 端口。

### 2.2 节点角色约定

本文使用 `master` 表示控制平面节点。Kubernetes 官方术语目前更推荐使用 **control plane（控制平面）**，因此命令输出中的角色通常显示为 `control-plane`。

以下操作应在 **所有节点** 执行，除非特别注明：

```text
192.168.8.30  master / control-plane
192.168.8.31  worker
192.168.8.32  worker
```

## 三、所有节点执行：操作系统预处理

### 3.1 设置主机名与 hosts

在每台机器上设置唯一主机名，例如：

```bash
hostnamectl set-hostname master
# worker 节点分别设置为 worker-1、worker-2
```

在所有节点的 `/etc/hosts` 中写入节点映射：

```bash
cat >> /etc/hosts <<'EOF'
192.168.8.30 master
192.168.8.31 worker-1
192.168.8.32 worker-2
EOF
```

检查解析是否正常：

```bash
getent hosts master worker-1 worker-2
```

### 3.2 处理防火墙与 SELinux

学习环境可以暂时停止 firewalld：

```bash
systemctl disable --now firewalld
```

生产环境建议根据 Kubernetes 官方端口清单放行端口，而不是关闭防火墙。[2]

SELinux 建议优先使用发行版和 Kubernetes 官方支持的配置。若只是为了搭建学习环境，可以临时切换为 permissive：

```bash
setenforce 0
getenforce
```

如果确实需要在实验环境永久关闭，再编辑 `/etc/selinux/config`：

```ini
SELINUX=permissive
```

> `setenforce 0` 只对当前运行周期生效；修改 `/etc/selinux/config` 需要重启后才会完整生效。不要把关闭 SELinux 当作生产环境的默认方案。

### 3.3 关闭 swap

传统 kubeadm 部署通常要求关闭 swap；不同 Kubernetes 版本对 swap 的支持能力可能不同。为了减少学习环境中的变量，本文仍然采用关闭 swap 的方式。[1]

```bash
swapoff -a
sed -ri '/\sswap\s/s/^/#/' /etc/fstab
```

验证：

```bash
free -h
swapon --show
```

如果 `swapon --show` 没有输出，说明当前没有启用 swap。

### 3.4 加载内核模块

`overlay` 用于容器镜像存储，`br_netfilter` 用于让网桥流量进入内核网络过滤路径：

```bash
cat >/etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

验证：

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

### 3.5 配置内核网络参数

```bash
cat >/etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

验证：

```bash
sysctl net.bridge.bridge-nf-call-iptables \
       net.bridge.bridge-nf-call-ip6tables \
       net.ipv4.ip_forward
```

预期结果中，相关参数应为 `1`。

## 四、所有节点安装 containerd

### 4.1 安装 containerd

不同发行版的软件包管理命令可能不同。以 RHEL/CentOS 兼容发行版为例：

```bash
yum install -y yum-utils

yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

yum install -y containerd.io
```

如果你的环境无法访问外网，应提前准备离线 RPM 包或配置企业内部镜像仓库，不建议在文档中直接删除所有系统软件源。

### 4.2 生成并修改 containerd 配置

```bash
mkdir -p /etc/containerd
containerd config default >/etc/containerd/config.toml
```

将 cgroup 驱动切换为 systemd：

```bash
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' \
  /etc/containerd/config.toml
```

Kubernetes、kubelet 和 containerd 的 cgroup 驱动应保持一致，否则可能导致节点异常或 `NotReady`。[1] [3]

启动并设置 containerd 开机自启：

```bash
systemctl enable --now containerd
systemctl status containerd --no-pager
```

验证 CRI 是否正常：

```bash
crictl info
```

如果系统尚未安装 `crictl`，可以先跳过该命令，在安装 Kubernetes 工具后再验证。

### 4.3 国内网络环境的镜像说明

早期教程经常通过修改 `config.toml`，将 `registry.k8s.io` 替换成某个镜像代理地址。这类地址可能发生变更，也可能存在同步延迟。更稳妥的做法是使用组织内部镜像仓库，或者在执行部署前通过 `kubeadm config images list` 和 `crictl pull` 验证目标仓库是否可访问。

如需使用 containerd 的镜像仓库配置，建议使用 `certs.d/hosts.toml` 目录方式，并根据当前 containerd 版本文档配置，不要直接复制来源不明的镜像地址。[3]

## 五、所有节点安装 kubeadm、kubelet 和 kubectl

### 5.1 配置 Kubernetes 软件源

创建 `/etc/yum.repos.d/kubernetes.repo`：

```ini
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

> 软件源地址和版本仓库会随 Kubernetes 发布策略变化。发布本文前，请以 [Kubernetes 官方安装文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 为准，不要长期依赖过时的镜像站地址。[1]

### 5.2 安装并锁定版本

```bash
yum install -y kubelet-1.30.0 kubeadm-1.30.0 kubectl-1.30.0 \
  --disableexcludes=kubernetes

yum install -y bash-completion

yum versionlock kubelet kubeadm kubectl 2>/dev/null || true
systemctl enable --now kubelet
```

kubelet 在集群初始化前可能反复报错，这是因为它暂时找不到 API Server；执行 `kubeadm init` 或 `kubeadm join` 后再观察状态。

检查版本：

```bash
kubeadm version
kubelet --version
kubectl version --client
```

## 六、仅在控制平面节点执行：初始化集群

### 6.1 初始化前检查镜像

```bash
kubeadm config images list \
  --kubernetes-version v1.30.0 \
  --image-repository registry.k8s.io
```

如果目标环境无法访问默认仓库，应先准备可用的镜像仓库，并确认每个镜像都能被 containerd 拉取。不要在不知道镜像是否存在的情况下直接替换仓库地址。

### 6.2 执行 kubeadm init

下面是学习环境示例。请替换控制平面地址和版本：

```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=172.16.0.0/16 \
  --kubernetes-version=v1.30.0 \
  --control-plane-endpoint=192.168.8.30:6443 \
  --upload-certs
```

如果你使用经过验证的国内或内部镜像仓库，可以额外指定：

```bash
--image-repository=你的镜像仓库地址
```

参数说明如下：

| 参数 | 含义 |
| --- | --- |
| `--pod-network-cidr` | Pod 网络地址池，必须与后续 CNI 配置一致 |
| `--service-cidr` | Service 虚拟 IP 地址池，不能与主机、Pod 或现有网络冲突 |
| `--kubernetes-version` | 指定控制平面版本，应与安装的 kubeadm 版本兼容 |
| `--control-plane-endpoint` | 控制平面访问地址；高可用场景应填写稳定的 VIP 或负载均衡地址 |
| `--upload-certs` | 将控制平面证书上传到集群，便于加入额外控制平面节点 |

初始化成功后，请保存终端输出中的两类信息：

1. 当前用户配置 `kubectl` 的命令。
2. 工作节点执行的 `kubeadm join` 命令。

### 6.3 配置 kubectl

以当前普通用户执行：

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

验证 API Server：

```bash
kubectl cluster-info
kubectl get nodes
```

此时节点可能显示 `NotReady`，因为还没有安装 CNI 网络插件，这是预期现象。

## 七、在工作节点加入集群

在每个 worker 节点执行初始化输出的 `kubeadm join` 命令。示例中的 token 和 hash 仅用于说明格式，不可直接使用：

```bash
kubeadm join 192.168.8.30:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

如果 token 过期，在控制平面节点重新生成：

```bash
kubeadm token create --print-join-command
```

加入后，在控制平面节点查看：

```bash
kubectl get nodes -o wide
```

## 八、安装 CNI 网络插件

集群必须安装 CNI 插件后，Pod 网络才能正常工作。本文使用 Calico 作为示例，但生产环境应根据网络需求、版本兼容性和网络策略要求选择 CNI。[4]

> **网段一致性：**`kubeadm init` 中的 `--pod-network-cidr` 必须与 Calico 的 IPPool 配置一致。如果不一致，节点可能长期处于 `NotReady`，Pod 也可能无法互通。

部署 Calico 前，请先阅读对应版本的官方安装说明，并确认它支持你的 Kubernetes 版本：

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

如果实验环境无法访问 GitHub，可以先把 YAML 下载到控制平面节点，再执行：

```bash
kubectl apply -f ./calico.yaml
```

查看系统 Pod：

```bash
kubectl get pods -n kube-system -o wide
```

查看节点状态：

```bash
kubectl get nodes
```

最终所有节点应显示为 `Ready`。

## 九、部署后的验证

### 9.1 查看集群与系统组件

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get cs 2>/dev/null || true
```

### 9.2 创建测试 Pod

```bash
kubectl create deployment nginx --image=nginx:stable
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl get deployment,pod,service
```

确认 Pod 已运行：

```bash
kubectl rollout status deployment/nginx
kubectl get pods -o wide
```

测试完成后清理资源：

```bash
kubectl delete service nginx
kubectl delete deployment nginx
```

### 9.3 验证节点间网络

```bash
kubectl run netshoot --image=nicolaka/netshoot --command -- sleep 3600
kubectl get pod netshoot -o wide
kubectl exec -it netshoot -- ip addr
kubectl delete pod netshoot
```

如果镜像无法拉取，应检查镜像仓库、DNS、代理和 containerd 日志，而不是直接反复重启 kubelet。

## 十、常见故障排查

| 现象 | 常见原因 | 排查方向 |
| --- | --- | --- |
| 节点一直 `NotReady` | 未安装 CNI、CNI 网段不匹配、kubelet 或 containerd 异常 | `kubectl describe node`、`kubectl get pods -n kube-system`、`journalctl -u kubelet` |
| `kubeadm init` 拉取镜像失败 | DNS、代理、仓库不可达或镜像不存在 | `kubeadm config images list`、`crictl pull`、`curl` 检查仓库 |
| containerd 启动失败 | 配置文件语法错误、cgroup 配置不正确 | `containerd config dump`、`journalctl -u containerd` |
| Pod 处于 `Pending` | 节点资源不足、污点、调度约束或网络插件未就绪 | `kubectl describe pod`、`kubectl describe node` |
| worker 加入失败 | token 过期、CA hash 错误、6443 不通、主机名重复 | 重新生成 join 命令，检查 `nc -vz 192.168.8.30 6443` |
| kubectl 报权限错误 | kubeconfig 不存在或属主错误 | 检查 `$HOME/.kube/config` 和文件权限 |
| crictl 出现 endpoint 警告 | 没有配置 CRI socket | 创建 `/etc/crictl.yaml` 并指定 containerd socket |

查看关键日志：

```bash
journalctl -u kubelet -n 100 --no-pager
journalctl -u containerd -n 100 --no-pager
kubectl get events -A --sort-by=.lastTimestamp
```

### 10.1 配置 crictl

如果使用 containerd，可以创建：

```bash
cat >/etc/crictl.yaml <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

验证：

```bash
crictl info
crictl ps
crictl images
```

## 十一、学习环境的常用增强配置

### 11.1 kubectl 命令补全

```bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl >/dev/null
source /etc/bash_completion.d/kubectl
```

### 11.2 查看容器运行时资源

Kubernetes 使用 containerd 时，不建议把 `docker` 命令简单别名为 `crictl`。两者命令语义并不完全相同。学习 CRI 时应直接使用：

```bash
crictl ps
crictl images
crictl logs <container-id>
```

### 11.3 切换 kube-proxy 到 IPVS 的说明

小型学习集群通常不需要切换到 IPVS。不同 Kubernetes 版本和内核版本对 kube-proxy、iptables、nftables、IPVS 的支持细节可能不同。只有在明确了解内核模块、kube-proxy 配置和回滚方式后，才建议进行此项实验。

## 十二、可选路线：使用 Sealos 快速部署

Sealos 可以封装 kubeadm、CNI 和相关组件，减少手工安装步骤，适合快速搭建实验环境。它并不能替代对 Kubernetes 基础组件的理解。[5]

### 12.1 安装 Sealos

请从 [Sealos 官方 Releases](https://github.com/labring/sealos/releases) 获取与你的系统架构匹配的版本。不要长期固定使用来源不明的下载地址。

以 Linux amd64 为例，下载后执行：

```bash
tar -xzf sealos_<version>_linux_amd64.tar.gz
sudo install -m 0755 sealos /usr/local/bin/sealos
sealos version
```

### 12.2 使用 Clusterfile 部署

Sealos 部署前需要确认：

- 控制平面和工作节点之间 SSH 互通。
- 节点完成 swap、内核模块、网络和主机名等基础准备。
- 使用密钥认证或安全地传入密码，不要把真实密码提交到 GitHub。
- 所使用的 Kubernetes、Calico 和 Helm 镜像版本彼此兼容。

命令结构示例：

```bash
sealos run <kubernetes-image> <helm-image> <cni-image> \
  --masters 192.168.8.30 \
  --nodes 192.168.8.31,192.168.8.32 \
  -u root
```

为了避免密码出现在 Shell 历史记录中，建议优先使用 SSH 公钥认证。具体参数以当前 Sealos 版本的官方文档为准。

常见管理命令示例：

```bash
sealos add --nodes 192.168.8.33
sealos delete --nodes 192.168.8.33
sealos exec "kubectl get nodes"
```

> `sealos reset` 或删除节点可能造成数据和业务中断。执行前必须确认节点角色、业务状态和备份情况。

## 十三、清理实验集群

如果需要在控制平面节点清理 kubeadm 集群：

```bash
kubeadm reset -f
rm -rf "$HOME/.kube"
```

然后在各节点停止并卸载不再需要的组件。清理操作可能删除集群配置、证书和本地资源，请不要在生产环境直接执行。

## 十四、完整流程回顾

本文的手工路线可以概括为：

1. 为所有节点设置唯一主机名，确保网络和时间同步正常。
2. 处理 swap、SELinux、防火墙、内核模块和转发参数。
3. 在所有节点安装并配置 containerd，统一 cgroup 驱动。
4. 在所有节点安装兼容版本的 kubeadm、kubelet 和 kubectl。
5. 仅在控制平面节点执行 `kubeadm init`。
6. 配置普通用户的 kubeconfig。
7. 在工作节点执行 `kubeadm join`。
8. 安装与 Pod 网段匹配的 CNI 插件。
9. 通过节点、系统 Pod、Service 和测试 Pod 验证集群。
10. 根据需要增加命令补全、日志、监控和镜像仓库配置。

## 十五、学习路线建议

搭建集群只是开始。建议按照以下顺序继续学习：

| 阶段 | 建议主题 |
| --- | --- |
| 基础资源 | Pod、Deployment、ReplicaSet、Service、Namespace |
| 配置管理 | ConfigMap、Secret、环境变量、Volume |
| 调度机制 | requests/limits、节点选择器、亲和性、污点和容忍度 |
| 网络 | Service、CoreDNS、Ingress、NetworkPolicy、CNI |
| 存储 | PV、PVC、StorageClass、CSI |
| 运维 | 日志、监控、滚动更新、回滚、备份和升级 |
| 安全 | RBAC、ServiceAccount、Secret 管理、最小权限原则 |
| 生产实践 | 高可用控制平面、镜像仓库、审计、资源配额和灾备 |

## 十六、安全提醒

本文中的命令包含停止防火墙、修改 SELinux、执行远程安装和重置集群等高权限操作。执行前应确认当前机器不是生产节点，并为重要配置和数据准备备份。

不要在公开仓库中提交以下内容：

- 真实的 SSH 密码、root 密码和 API Token。
- `admin.conf`、私钥、Cookie、证书私钥和 `storage` 文件。
- 包含真实内网拓扑、业务域名或敏感镜像仓库凭据的配置文件。
- 带有真实 `kubeadm join` token 的日志或截图。

## 参考资料

[1]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ "Kubernetes 官方 kubeadm 安装文档"
[2]: https://kubernetes.io/docs/reference/networking/ports-and-protocols/ "Kubernetes 官方端口与协议"
[3]: https://github.com/containerd/containerd/blob/main/docs/getting-started.md "containerd 官方入门文档"
[4]: https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart "Calico 官方 Kubernetes 快速开始"
[5]: https://sealos.io/docs/ "Sealos 官方文档"

---

> 本文定位为学习笔记，示例配置和版本仅用于演示。发布到 GitHub 前，建议在目标操作系统和目标 Kubernetes 版本上完整验证一遍，并在文档顶部注明实际验证环境。
