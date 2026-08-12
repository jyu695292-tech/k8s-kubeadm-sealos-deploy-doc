# K8s 集群部署初始化命令详解

## **一、手工部署K8s集群**

> 环境：CentOS Linux，kubeadm 部署 K8s 集群前置操作

## 1. 关闭防火墙和 SELinux

```shell
systemctl disable firewalld.service --now
```

- `systemctl`：systemd 系统服务管理工具
- `disable`：设置服务开机禁止自启
- `firewalld.service`：CentOS 默认防火墙服务
- `--now`：立刻停止当前正在运行的服务，无需重启服务器

> 作用：立刻关闭防火墙，并且设置开机不自动启动。K8s 集群内部大量端口通信，测试环境直接关闭防火墙；生产环境建议放行对应端口，不直接关闭。

```shell
vim /etc/selinux/config
SELINUX=disabled
```

- `/etc/selinux/config`：SELinux 永久配置文件
- `SELINUX=disabled`：设置 SELinux 永久关闭

> 该修改写入配置文件，**服务器重启后才完整生效**。SELinux 为 Linux 安全增强模块，会拦截容器文件读写、网络交互，部署 K8s 环境直接关闭。

```shell
setenforce 0
```

- `setenforce`：临时修改 SELinux 运行状态工具
- `0`代表关闭，`1`代表开启

> **临时生效，重启服务器失效**。无需重启机器，即时关闭 SELinux，配合配置文件实现永久关闭。

## 2. 关闭 swap 交换分区

> K8s 硬性强制要求必须关闭 swap，swap 磁盘虚拟内存会严重干扰 kubelet 调度逻辑。

```shell
sed -i '/swap/s/^/#/' /etc/fstab
```

- `sed`：流文本编辑工具
- `-i`：直接修改原始文件
- `/swap/`：匹配文件内包含 swap 的文本行
- `s/^/#/`：在行首添加`#`，将该行注释
- `/etc/fstab`：系统开机自动挂载磁盘配置文件

> 作用：注释 swap 开机挂载条目，保证重启服务器后 swap 不会自动开启。

```
swapoff -a
```

- `swapoff`：关闭 swap 分区命令
- `-a`：关闭全部 swap 设备

> **即时生效，重启失效**。立刻关闭当前机器 swap。

> 验证命令：`free -m`，swap 全部为 0 代表关闭成功。

## 3. 设置系统参数，加载内核模块

```
cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
```

- `/etc/modules-load.d/k8s.conf`：开机自动加载内核模块配置目录
- `cat > 文件 << EOF ... EOF`：here‑document 语法，将模块名称写入配置文件
- `overlay`：容器存储内核模块，containerd 存储驱动依赖
- `br_netfilter`：网桥过滤内核模块，Pod 网桥网络转发依赖

> 效果：服务器开机自动加载这两个内核模块。

```
modprobe overlay
modprobe br_netfilter
```

- `modprobe`：手动加载内核模块命令
- `overlay`：容器存储层内核模块
- `br_netfilter`：网桥过滤模块，让 iptables 可以处理网桥数据包，K8s Service iptables 模式必需。

> 验证：`lsmod | grep -E "overlay|br_netfilter"`

```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

- `/etc/sysctl.d/k8s.conf`：内核参数永久配置文件

1. `net.bridge.bridge-nf-call-iptables = 1`：网桥数据包交给 iptables 处理，Service 转发依赖，不开启 Pod 互通异常
2. `net.bridge.bridge-nf-call-ip6tables = 1`：IPv6 网桥数据包交给 ip6tables 处理
3. `net.ipv4.ip_forward = 1`：开启 Linux 主机 IP 转发，实现跨节点 Pod 通信

```
sysctl --system
```

- `sysctl --system`：读取`/etc/sysctl.d/`下全部配置，重载内核参数，即时生效，无需重启。

> 验证命令：`sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward`

------

**以上是集群安装前的操作系统预处理，做完之后，才开始安装 kubeadm、kubelet、containerd。**

# 4. 安装容器运行时 Containerd

> Kubernetes v1.30 推荐使用 Containerd 作为容器运行时，负责镜像拉取、容器创建销毁生命周期管理。

```
yum -y  remove runc
```

- `yum`：CentOS rpm 软件包管理器
- `-y`：全部交互确认自动选 yes
- `remove`：卸载软件
- `runc`：底层容器二进制程序，用于创建进程隔离容器

> 作用：卸载系统自带旧版本 runc，避免版本冲突导致 containerd 异常。后续由 containerd 捆绑安装新版 runc 全权替代。

```
rm -rf /etc/yum.repos.d/Cent*.repo
```

- `rm`：删除文件
- `-r`递归，`-f`强制删除，无警告提示
- `/etc/yum.repos.d/`：yum 软件源存放目录
- `Cent*.repo`：所有 Cent 开头的外网 repo 源文件

> 作用：删除默认外网 yum 源，后续使用本地 DVD 光盘源，避免源冲突。

```
mount /dev/cdrom /media/
```

- `mount`：挂载设备到指定目录
- `/dev/cdrom`：光驱 / 虚拟机 ISO 镜像设备
- `/media/`：挂载访问目录

> 将 ISO 光盘镜像挂载到 /media 目录，读取光盘内 rpm 包；重启挂载失效。

```
cat /etc/fstab  |grep iso9660
```

- `/etc/fstab`：开机自动挂载配置文件
- `cat`读取文件；`|`管道，输出交给后续命令
- `grep iso9660`：过滤光盘文件系统类型的行

输出示例：

```
/dev/cdrom		/media			iso9660 defaults   0 0
```

字段：设备 | 挂载目录 | 文件系统类型 | 挂载参数 | 备份标记 | 自检标记

```
cat /etc/yum.repos.d/dvd.repo 
```

dvd.repo 本地光盘源内容：

```shell
[BaseOS]
name=BaseOS
baseurl=file:///media/BaseOS
gpgcheck=0

[AppStream]
name=AppStream
baseurl=file:///media/AppStream
gpgcheck=0
```

- `[BaseOS]`：仓库 ID 标识
- `name`：仓库描述名称
- `baseurl`：软件包地址，`file://`代表本地文件
- `gpgcheck=0`：关闭 rpm 数字签名校验，本地光盘无密钥，关闭校验防止报错

```
yum install -y yum-utils
```

- `yum‑utils`：yum 工具集，提供`yum‑config‑manager`管理软件源。

```shell
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

- `yum‑config‑manager`：yum 源管理工具
- `--add‑repo`：新增 yum 软件仓库

> 添加阿里云 docker-ce 网络源，源内包含 containerd.io 软件包，该步骤需要外网。

```
yum install -y containerd.io
```

> 从阿里云源安装 containerd，自动生成 systemd 服务文件。

```
mkdir -p /etc/containerd
```

- `mkdir`创建目录
- `-p`：目录存在不报错，不存在则创建，containerd 配置存放目录。

```
containerd config default > /etc/containerd/config.toml
```

- `containerd config default`：输出 containerd 默认完整 toml 格式配置模板
- `>`输出重定向，输出写入配置文件

> 安装完 containerd 无配置文件，必须生成模板，否则服务启动失败。

```
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g'  /etc/containerd/config.toml
```

- `sed -i`直接修改文件
- `s/旧值/新值/g`全局替换

> K8s 强制要求`SystemdCgroup=true`，cgroup 驱动必须和 kubelet 保持一致；false 为 cgroupfs 驱动，不一致会造成节点 NotReady。

```
sed -i 's|registry.k8s.io|registry.aliyuncs.com/google_containers|g' /etc/containerd/config.toml
```

> 镜像地址替换，国内环境把 k8s 官方镜像仓库替换阿里云镜像代理，解决镜像拉取失败。

```
systemctl enable --now containerd
```

- `enable`设置开机自启
- `--now`立刻启动服务

> 即时启动 containerd 并且配置开机自动运行。

> 验证命令：`systemctl status containerd`，看到`active (running)`代表运行正常。

------

## 整体流程总结

1. 卸载旧 runc，规避版本冲突；后续由 containerd 捆绑安装新版 runc 全权替代。
2. 删除外网 yum 源，配置本地 DVD 光盘 yum 源，实现离线基础软件安装；
3. 安装 yum 工具，添加阿里云 docker-ce 互联网源，用于安装 containerd；
4. yum 安装 containerd.io；
5. 生成 containerd 配置模板，修改 cgroup 驱动为 systemd，配置国内镜像加速；
6. 启动 containerd 服务，设置开机自启。

> 高频故障点：
>
> 1. 不生成`config.toml`，containerd 启动失败；
> 2. `SystemdCgroup=false`未修改，cgroup 驱动不匹配，K8s 节点 NotReady。

> 两个源区分：
>
> - dvd.repo：本地光盘源，系统基础软件，离线可用
> - docker‑ce.repo：阿里云网络源，下载 containerd，需要外网

## 5. 安装和配置 kubernetes

```shell
cat kubernetes.repo 
```

kubernetes.repo 文件完整内容：

```shell
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key
```

- `[kubernetes]`：仓库 ID，yum 仓库唯一标识
- `name=Kubernetes`：仓库自定义描述名称
- `baseurl`：软件包下载地址，使用**阿里云 K8s 国内镜像源**，替代国外 google 官方源，解决国内下载慢、访问失败问题
- `enabled=1`：启用该 yum 仓库；`1`开启，`0`关闭
- `gpgcheck=1`：开启 RPM 包数字签名校验，校验软件包完整性、防止包被篡改
- `gpgkey`：签名公钥文件地址，用来校验软件包合法性

>
> 该 repo 文件放置在 `/etc/yum.repos.d/` 目录下，yum 才可以识别这个软件仓库。

```shell
yum install -y kubelet-1.30.0 kubeadm-1.30.0 kubectl-1.30.0 --disableexcludes=kubernetes
```

- `yum install -y`：yum 安装软件，全部交互确认自动 yes
- `kubelet‑1.30.0`：节点核心代理程序，运行在所有 master、worker 节点，跟容器运行时交互，管理本机 Pod，**集群每一台机器都必须安装**
- `kubeadm‑1.30.0`：集群部署工具，只用来初始化集群、节点加入集群，集群搭建完成后日常很少使用
- `kubectl‑1.30.0`：集群客户端命令行工具，用来给 apiserver 发送指令，管理集群资源；可以安装在任意机器操作集群
- `--disableexcludes=kubernetes`：禁用 yum 的排除机制，防止系统配置屏蔽 k8s 相关软件，避免安装失败

>
> ⚠️ 三个组件**版本必须严格统一**，这里锁定 v1.30.0，版本不一致会集群异常。

```shell
systemctl enable --now kubelet
```

- `systemctl`：systemd 系统服务管理工具
- `enable`：设置 kubelet 开机自启
- `--now`：立刻启动 kubelet 服务

>
> kubelet 启动之后此时状态会持续报错，属于正常现象：**集群还没有初始化，kubelet 找不到 apiserver，等待 kubeadm init 完成之后才会正常工作**。

## 6. 初始化 kubernetes

>
> 注意：所有节点做完前置环境预处理，**仅 master 控制平面节点执行 kubeadm init**，worker 节点不要执行。

```shell
kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=172.16.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.30.0  \
--control-plane-endpoint="192.168.8.30:6443" \
--upload-certs
```

`kubeadm init`：初始化 K8s 控制平面（master 节点），生成集群所有核心组件配置、证书，拉取控制平面组件镜像。

参数逐条解析：

1. `--pod-network-cidr=10.244.0.0/16`：Pod 网络网段，**Calico 网络插件默认使用该网段**，网段一旦初始化完成不可修改。Pod 的 IP 全部从这个网段分配。
2. `--service-cidr=172.16.0.0/16`：Service 虚拟服务网段，集群内部 Service 的 IP 地址池，不能和主机、Pod 网段冲突。
3. `--image-repository registry.aliyuncs.com/google_containers`：镜像仓库地址，使用阿里云镜像代理，拉取 apiserver、etcd、scheduler 等组件镜像，规避访问 google 镜像失败。
4. `--kubernetes-version v1.30.0`：指定集群版本，必须和 kubeadm/kubelet/kubectl 安装版本保持一致。
5. `--control-plane-endpoint="192.168.8.30:6443"`：控制平面访问地址，填写 master 节点内网 IP，6443 是 apiserver 默认端口。多 master 高可用场景这里填写 VIP 负载均衡地址。
6. `--upload‑certs`：把控制平面证书上传到集群 secret 资源，用于多 master 节点扩容，把证书自动分发到其他控制节点。

>
> kubeadm init 执行成功后输出两段关键信息：
>
>
> 1. kubeconfig 配置文件拷贝提示；
> 2. worker 节点加入集群的`kubeadm join`命令（包含 token、hash 值）。

```shell
mkdir -p $HOME/.kube
```

- `mkdir -p`：目录不存在创建，存在不报错；
- `$HOME/.kube`：当前用户家目录下.kube 文件夹，用来存放 kubectl 访问集群的配置文件 config。

```shell
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

- `/etc/kubernetes/admin.conf`：kubeadm init 生成的集群管理员完整权限配置文件，包含 apiserver 地址、证书、密钥。
- 复制到当前用户目录，`kubectl`默认读取 `~/.kube/config`，实现可以直接执行 kubectl 命令操作集群。
- `-i`：覆盖文件前提示。

```shell
chown $(id -u):$(id -g) $HOME/.kube/config
```

- `id -u` 获取当前用户 UID；`id -g`获取当前用户 GID
- `chown` 修改文件属主属组；

>
> 作用：把 config 文件归属改成当前普通用户，避免只能 root 用户使用 kubectl，普通用户执行 kubectl 报权限拒绝。

## 7. 将 worker 节点加入 kubernetes 集群

> 该命令在**worker 工作节点执行**，master 节点不要执行。
> kubeadm init 输出的 join 命令里面 token、sha256 哈希每一套集群都不相同，文档示例只是演示。

```shell
kubeadm join 192.168.8.30:6443 --token yfjrsi.60lmqgzobp0b8al3 --discovery-token-ca-cert-hash sha256:f4e3c924918545abfd8148e71a377e086d591c6a59856a496b4b33e0987d9c18
```

- `kubeadm join`：节点加入集群命令
- `192.168.8.30:6443`：master 节点 apiserver 地址端口，worker 节点去找控制平面注册自己
- `--token xxx`：加入集群的身份令牌，有效期默认 24 小时；过期之后节点无法加入集群
- `--discovery‑token‑ca‑cert‑hash sha256:xxx`：CA 证书哈希校验值，worker 节点校验 master 证书合法性，防止接入伪造集群。

>
> token 过期解决方案：master 节点执行
>
>
> ```shell
> kubeadm token create --print-join-command
> ```
>
>
> 直接输出完整可用的 kubeadm join 命令，复制到 worker 节点执行。

## 8. 在 master 节点上安装 calico 网络插件

>
> K8s 集群初始化完成后**必须部署网络插件**，否则 Pod 之间网络不通，节点状态为 NotReady。Calico 是主流网络 CNI 插件，负责 Pod 网络、网络策略。

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

- `kubectl apply -f`：加载 yaml 资源清单，向集群创建 / 更新资源
- 直接网络下载 calico 官方 yaml 清单，部署 Calico 全套 Pod 资源。

>
> 问题：[raw.githubusercontent.com](https://link.wtturl.cn/?target=https%3A%2F%2Fraw.githubusercontent.com&scene=im&aid=497858&lang=zh)国内访问经常失败，同时 calico 镜像国外仓库无法拉取，节点会一直 NotReady。

```
kubectl apply -f  /root/calico-ucloud.yaml
```

- 将修改镜像地址为国内镜像的 calico‑ucloud.yaml 上传到 master 节点本地`/root`目录，读取本地文件部署，规避 github 网络访问失败。

### 集群状态校验

```shell
kubectl get nodes
```

输出示例：

```
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   65m   v1.30.13
node1    Ready    <none>          62m   v1.30.0
node2    Ready    <none>          62m   v1.30.0
```

- `STATUS:Ready`：代表节点就绪，可以调度 Pod；
- `ROLES`：master 节点标记 control‑plane；worker 节点无角色显示`<none>`；

>
> 全部节点状态为 Ready，代表整套 K8s 集群安装完成。

---

## 整体流程总结

1. 配置阿里云 kubernetes yum 软件源，安装 kubelet、kubeadm、kubectl，版本严格统一；
2. 开启 kubelet 开机自启，初始化 master 控制平面，指定 pod、service 网段、国内镜像仓库；
3. 拷贝 admin.conf 配置文件，授权普通用户可以执行 kubectl；
4. worker 节点执行 kubeadm join 注册加入集群；token 过期使用`kubeadm token create --print-join-command`重新生成；
5. 部署 Calico CNI 网络插件，解决 Pod 网络通信；
6. `kubectl get nodes`查看全部节点状态全部 Ready，集群部署完毕。

>
> 高频故障点
>
>
> 1. kubeadm init 的 pod‑network‑cidr 必须和 calico 使用网段一致，否则网络插件异常；
> 2. token 过期 worker 节点无法加入集群；
> 3. 没有部署 CNI 网络插件，节点一直 NotReady；
> 4. kubelet/kubeadm/kubectl 三者版本不一致，集群启动报错

>
> 

## 9. 扩展优化配置

>
> 下述配置需要在集群所有 Master、Worker 节点执行，属于部署完成后运维优化配置，非集群部署必需步骤

### 9.1 配置 kubectl 命令 Tab 自动补全

```
kubectl completion bash > /etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubectl
```

作用：配置完成后，输入 kubectl 部分指令按下 Tab，自动补齐子命令、Pod 名称、命名空间，降低手动输入出错概率。

### 9.2 配置 docker 命令别名适配 containerd 运行时

```
vim  /root/.bashrc
```

在文件内写入别名配置：

```
alias docker='crictl'
```

重载环境变量配置，当前终端即刻生效：

```
source /root/.bashrc
```

>
> 集群采用 containerd 作为容器运行时，未部署 Docker 组件，运维习惯使用`docker ps`、`docker images`指令，配置别名后输入 docker 等价调用 crictl 工具。

### 9.3 消除 crictl 弃用端点警告

执行别名 docker 指令会弹出如下告警：

```
WARN[0000] image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
```

原因：未手动指定容器运行时套接字，工具会逐个扫描多组地址，该扫描方式已经废弃。新建`crictl.yaml`固定通信端点：

```
cat > /etc/crictl.yaml << EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
pull-image-on-create: false
EOF
```

参数释义：

- runtime-endpoint、image-endpoint：指定对接 containerd 的套接字文件；若集群使用 CRI‑O 运行时，则替换为`unix:///run/crio/crio.sock`
- timeout：操作超时时长

### 9.4 可选：kube‑proxy 负载均衡由 iptables 切换 IPVS 模式

适用场景：小规模测试集群无需改动；企业多节点高并发业务集群推荐切换，解决 iptables 规则堆积造成转发性能衰减。

#### 9.4.1 所有节点安装 IPVS 管理工具

```
yum install -y ipvsadm
```

#### 9.4.2 写入开机自加载 IPVS 内核模块

```
cat > /etc/modules-load.d/ipvs.conf << EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
```

模块作用：

- ip_vs：IPVS 核心负载均衡模块
- ip_vs_rr/wrr/sh：对应不同调度策略内核模块
- nf_conntrack：会话连接跟踪模块

#### 9.4.3 修改 kube‑proxy 全局配置

```
kubectl edit cm kube-proxy -n kube-system
```

定位配置片段，修改工作模式与调度算法：

```
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"
```

mode 取值：iptables/ipvs；scheduler 调度策略：rr 轮询、wrr 加权轮询、lc 最少连接。

#### 9.4.4 重建 kube‑proxy Pod 加载新配置

```
kubectl delete pods -n kube-system -l k8s-app=kube-proxy
```

#### 9.4.5 校验运行状态

查看 Pod 运行状态：

```
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

查看内核 IPVS 转发规则，确认配置生效：

```
ipvsadm -L -n
```

生效示例输出：

```
TCP  192.168.8.30:30080 rr
  -> 10.244.104.3:80              Masq    1      0          0         
  -> 10.244.104.4:80              Masq    1      0          0         
  -> 10.244.166.132:80            Masq    1      0          0
```

# 二、使用 sealos 工具快速部署（可选）

>
> sealos 是 k8s 集群一键部署工具，底层依然封装 kubeadm，简化大量手工操作，适合快速搭建集群；节点之间需要 root 密码互通，所有节点提前完成系统基础预处理（关闭 swap、防火墙、内核参数）。

## 1. 下载 sealos 工具

```
wget https://github.com/labring/sealos/releases/download/v5.0.1/sealos_5.0.1_linux_amd64.tar.gz
tar -xzf sealos_5.0.1_linux_amd64.tar.gz -C /usr/bin
```

- `wget`：下载 sealos5.0.1 版本 linux 二进制压缩包；
- `tar -xzf`：解压压缩包；`‑C /usr/bin`把二进制程序解压到系统环境变量目录，任意位置可以直接执行`sealos`命令。

## 2. 安装 kubernetes

```
vim k8s.sh
```

脚本`k8s.sh`写入内容：

```
sealos run registry.cn‑shanghai.aliyuncs.com/labring/kubernetes:v1.30.0  registry.cn‑shanghai.aliyuncs.com/labring/helm:v3.16.2 registry.cn‑shanghai.aliyuncs.com/labring/calico:v3.28.1 --masters 192.168.8.30 --nodes 192.168.8.31,192.168.8.32 -u root -p redhat
```

```
sh k8s.sh
```

参数解释：

- `sealos run`：执行集群安装；全部镜像使用阿里云 labring 国内镜像，规避外网拉取失败；
- `kubernetes:v1.30.0`：k8s 集群版本；`helm`包管理工具；`calico`内置 CNI 网络插件；
- `--masters`：指定 master 节点 IP，多主用逗号分隔；
- `--nodes`：指定 worker 节点 IP，多个节点逗号分隔；
- `-u root`：目标节点登录用户名；`‑p redhat`目标节点 root 密码；

>
> sealos 会通过 ssh 远程推送文件、安装组件，自动完成 kubeadm init、calico 部署，无需手动一条条敲 kubeadm 命令。
> 终端输出安装完成提示，代表集群部署成功。

## 3. 自定义 service 和 pod 网络（可选）

sealos 支持通过`Clusterfile`配置文件修改集群参数，不直接在 run 命令写大量参数。

```
sealos gen registry.cn-shanghai.aliyuncs.com/labring/kubernetes:v1.30.0  registry.cn-shanghai.aliyuncs.com/labring/helm:v3.16.2 registry.cn-shanghai.aliyuncs.com/labring/calico:v3.28.1 --masters 192.168.8.30 --nodes 192.168.8.31,192.168.8.32 -u root -p redhat > Clusterfile
```

- `sealos gen`：生成集群配置模板 Clusterfile，输出重定向存入文件；

```
vim Clusterfile
```

编辑 Clusterfile，修改 pod 网段、service 网段等网络参数。

```
sealos apply -f Clusterfile
```

- `sealos apply -f`：加载 Clusterfile 配置，应用到集群。

## 4. 将一个新节点加入 k8s 集群

```
kubeadm token create --print-join-command
```

输出示例：

```
kubeadm join apiserver.cluster.local:6443 --token o5gfkp.k69lc0oa0vdtx4py --discovery-token-ca-cert-hash sha256:c5df557da4065f21b19fd69f0e0404d08eb4ec91b13513bd0a57cd4e7279bf7e
```

>
> sealos 部署出来的集群，新增节点依旧可以使用原生 kubeadm join 方式入网；生成 join 命令复制到新增节点执行。

## 5. 扩展

```
kubectl completion bash > /etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubectl
```

配置 kubectl bash 命令补全，按 Tab 自动补全命令。

```
vim  /root/.bashrc
```

写入别名：

```
alias docker='crictl'
```

```
source /root/.bashrc
```

>
> 给 crictl 设置 docker 别名，适配旧运维使用习惯，和手动 kubeadm 部署功能完全一致。

## 6. 高阶管理

```
#增加节点
sealos add --nodes 192.168.64.21,192.168.64.19 
```

- `sealos add --nodes`：一键新增 worker 节点，填入新增节点 IP，不需要手动执行 kubeadm join。

```
#删除节点
sealos delete --nodes 192.168.64.21,192.168.64.19
```

- 删除 worker 节点，集群剔除该节点，清理节点上 k8s 相关组件。

```
#删除master节点
sealos delete --masters 192.168.64.21,192.168.64.19
```

- 删除控制平面 master 节点，高可用集群使用，注意保留至少一台 master。

```
#清理集群
sealos reset
```

- 本机重置 k8s 集群，清空所有 k8s 相关配置、容器、镜像。

```
#在所有节点执行命令
sealos exec "cat /etc/hosts"
```

- `sealos exec`：向全部集群节点远程执行同一条 shell 命令，批量运维。

## 7. containerd 镜像加速器配置

>
> containerd 新版本推荐使用`certs.d`目录方式配置镜像加速，不再直接修改 config.toml 写入镜像地址。

```
cat /etc/containerd/config.toml |grep -B1 'config_path'
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
```

- `config_path = "/etc/containerd/certs.d"`：代表镜像加速配置全部读取该目录下文件。

>
> 规则：针对哪个镜像仓库加速，就在`certs.d`下创建对应名称文件夹，内部固定文件名`hosts.toml`。

### docker.io 镜像加速配置

```
cat /etc/containerd/certs.d/docker.io/hosts.toml
```

```
server = "https://docker.io"
[host."https://fb273a16b77a4b0f8e84856a8043410d.mirror.swr.myhuaweicloud.com"]
  capabilities = ["pull", "resolve"]
```

- `server="https://docker.io"`：原始镜像源地址；
- `host."华为镜像加速地址"`：配置镜像代理后端；
- `capabilities = ["pull", "resolve"]`：允许拉取镜像、解析镜像。

测试加速器是否生效：

```
ctr --debug image pull docker.io/library/nginx:latest --hosts-dir /etc/containerd/certs.d
```

- `ctr`为 containerd 自带客户端；`--debug`输出详细调试日志，可以观察是否走了配置的加速地址。

### k8s 官方镜像 registry.k8s.io 加速配置

```
cat /etc/containerd/certs.d/registry.k8s.io/hosts.toml
```

```
server = "https://k8s.io"
[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
```

>
> 针对 k8s 官方镜像配置道客镜像加速，解决国内无法拉取 k8s 组件镜像问题。

>
> 注意：修改完 certs.d 下配置**不需要重启 containerd 服务**，配置即时生效；旧版本方式是修改 config.toml，新版本推荐 certs.d 目录结构。