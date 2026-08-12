# Kubernetes 核心资源与调度机制学习指南

> 本文从 Pod 出发，逐步理解静态 Pod、Init 容器、Deployment、Service、Ingress、DaemonSet，以及节点选择、亲和性、污点和容忍度等调度机制。
>
> 这不是一份“只记命令”的速查表，而是一篇试图回答 **Kubernetes 为什么这样设计** 的学习博客：Pod 为什么需要控制器管理？Service 如何找到不断变化的 Pod？Ingress 为什么不能单独工作？调度器如何决定一个 Pod 应该运行在哪个节点？
>
> **适合读者：**正在学习 Kubernetes、准备 CKA 实验、希望从资源对象和控制器角度建立完整知识体系的读者。

## 文章目录

- [一、先建立 Kubernetes 的整体认识](#一先建立-kubernetes-的整体认识)
- [二、Kubernetes 对象与声明式管理](#二kubernetes-对象与声明式管理)
- [三、Pod：最小可部署单元](#三pod最小可部署单元)
  - [3.1 Pod 的核心理解](#31-pod-的核心理解)
  - [3.2 使用命令创建 Pod](#32-使用命令创建-pod)
  - [3.3 使用 YAML 描述 Pod](#33-使用-yaml-描述-pod)
  - [3.4 Pod 生命周期与重启策略](#34-pod-生命周期与重启策略)
  - [3.5 静态 Pod](#35-静态-pod)
  - [3.6 Init 容器](#36-init-容器)
  - [3.7 Labels 与 Annotations](#37-labels-与-annotations)
- [四、Deployment：管理无状态应用](#四deployment管理无状态应用)
  - [4.1 Deployment、ReplicaSet 与 Pod 的关系](#41-deploymentreplicaset-与-pod-的关系)
  - [4.2 创建和查看 Deployment](#42-创建和查看-deployment)
  - [4.3 滚动更新](#43-滚动更新)
  - [4.4 版本历史与回滚](#44-版本历史与回滚)
- [五、Service：给变化中的 Pod 提供稳定入口](#五service给变化中的-pod-提供稳定入口)
  - [5.1 为什么需要 Service](#51-为什么需要-service)
  - [5.2 Service 的端口关系](#52-service-的端口关系)
  - [5.3 ClusterIP 与 NodePort](#53-clusterip-与-nodeport)
  - [5.4 Service 排错思路](#54-service-排错思路)
- [六、Ingress：七层 HTTP/HTTPS 路由](#六ingress七层-httphttps-路由)
- [七、DaemonSet：在节点维度运行 Pod](#七daemonset在节点维度运行-pod)
- [八、调度：决定 Pod 应该运行在哪里](#八调度决定-pod-应该运行在哪里)
  - [8.1 nodeSelector：最简单的节点选择](#81-nodeselector最简单的节点选择)
  - [8.2 节点亲和性与反亲和性](#82-节点亲和性与反亲和性)
  - [8.3 Taints 与 Tolerations](#83-taints-与-tolerations)
  - [8.4 调度失败时如何分析](#84-调度失败时如何分析)
- [九、常用查询与排错命令](#九常用查询与排错命令)
- [十、从资源对象到控制器的理解总结](#十从资源对象到控制器的理解总结)
- [十一、适合 CKA 的练习顺序](#十一适合-cka-的练习顺序)
- [参考资料](#参考资料)

## 一、先建立 Kubernetes 的整体认识

Kubernetes 不只是“运行容器的工具”，更像一个围绕 **期望状态** 工作的分布式控制系统。用户通过 YAML 或命令告诉 API Server：“我希望集群中存在 3 个带有某些标签的 Nginx Pod”，随后控制器、调度器和 kubelet 协同工作，把当前状态逐渐调整到期望状态。[1]

可以用下面的关系理解一项典型的无状态应用：

```text
Deployment
    │ 管理版本和副本
    ▼
ReplicaSet
    │ 保证副本数量
    ▼
Pod
    │ 运行一个或多个容器
    ▼
Container
```

访问路径则通常是：

```text
客户端
  │
  ▼
Ingress / Gateway      七层路由，可选
  │
  ▼
Service                稳定的服务入口
  │
  ▼
Pod                    实际处理请求的后端实例
```

这里有一个非常重要的设计思想：**Pod 是相对短暂的，Service 和控制器负责隐藏 Pod 的不稳定性。** Pod 可能因为发布、故障、驱逐或节点异常而被替换，但客户端不应该依赖某个具体 Pod 的名称或 IP 地址。[2]

## 二、Kubernetes 对象与声明式管理

Kubernetes 中的 Pod、Deployment、Service、Ingress 和 DaemonSet 都是 API 对象。对象通常包含以下几类信息：

| 字段 | 作用 |
| --- | --- |
| `apiVersion` | 指明资源使用的 API 版本 |
| `kind` | 指明资源类型，例如 `Pod`、`Deployment` |
| `metadata` | 存放名称、Namespace、Labels、Annotations 等元数据 |
| `spec` | 用户声明的期望状态 |
| `status` | Kubernetes 控制面和节点反馈的当前状态 |

例如，用户写入 `spec.replicas: 3`，表达的是“希望有 3 个副本”，而不是手工启动 3 个容器。控制器会持续比较 `spec` 与实际状态，并在副本不足时创建 Pod、在副本过多时删除 Pod。

常见操作方式如下：

```bash
kubectl apply -f app.yaml
kubectl get pods
kubectl describe pod <pod-name>
kubectl delete -f app.yaml
```

其中，`apply` 更接近声明式管理：它将本地配置应用到集群；`describe` 适合查看事件和调度失败原因；`delete` 会删除该 YAML 所管理的资源。

## 三、Pod：最小可部署单元

### 3.1 Pod 的核心理解

Pod 是 Kubernetes 中最小的可部署计算单元。一个 Pod 可以包含一个或多个容器，这些容器共享网络命名空间、存储卷以及生命周期上下文。[2]

> **Pod 不是容器。** Pod 是容器的编排和运行边界。多个容器只有在需要紧密协作、共享网络和共享存储时，才适合放在同一个 Pod 中。

同一个 Pod 内的容器：

- 共享同一个 Pod IP；
- 可以通过 `localhost` 互相访问；
- 可以挂载并共享同一个 Volume；
- 通常被共同调度到同一个节点；
- 共享 Pod 的生命周期边界，但每个容器仍然有自己的进程和状态。

最常见的模式是 **一个 Pod 运行一个主容器**。Sidecar、代理、日志采集器等辅助容器属于更复杂的场景，不应该仅仅为了“多运行几个容器”而把无关应用放到同一个 Pod 中。

### 3.2 使用命令创建 Pod

```bash
kubectl run web01 --image=nginx:stable
kubectl get pods
kubectl get pods -o wide
```

查看 Namespace：

```bash
kubectl get namespaces
kubectl get pods -n kube-system
```

进入 Pod 内部：

```bash
kubectl exec -it web01 -- /bin/bash
```

如果 Pod 中存在多个容器，需要通过 `-c` 指定容器：

```bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```

命令中的 `--` 用于分隔 kubectl 自身参数和要在容器中执行的命令。某些精简镜像不包含 Bash，此时应使用 `/bin/sh`。

> 直接创建的 Pod 适合学习和临时调试，不适合作为长期运行的业务管理方式。生产应用通常应由 Deployment、StatefulSet、DaemonSet 或 Job 等工作负载控制器创建。

### 3.3 使用 YAML 描述 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web01
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - name: http
          containerPort: 80
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

字段可以这样理解：

| 字段 | 技术含义 |
| --- | --- |
| `apiVersion: v1` | Pod 使用核心 API 组的 v1 版本 |
| `kind: Pod` | 资源类型是 Pod |
| `metadata.name` | Pod 名称，在同一个 Namespace 内唯一 |
| `metadata.labels` | 可被 Service、选择器和控制器使用的标签 |
| `spec.containers` | Pod 中需要运行的容器列表 |
| `dnsPolicy: ClusterFirst` | 优先通过集群 DNS 解析，常用于普通 Pod |
| `restartPolicy` | 定义 Pod 中容器退出后的重启策略 |

`containerPort` 主要是文档和元数据声明，并不会自动把端口暴露到集群外部。真正提供访问入口，通常还需要配置 Service 或 Ingress。

### 3.4 Pod 生命周期与重启策略

Pod 的 `restartPolicy` 常见取值如下：

| 取值 | 行为 | 典型场景 |
| --- | --- | --- |
| `Always` | 容器退出后持续重启，默认值 | 长期运行的业务容器 |
| `OnFailure` | 容器以非 0 状态退出时重启 | Job 或批处理任务 |
| `Never` | 容器退出后不重启 | 一次性实验或调试 |

需要区分 **容器重启** 与 **Pod 重建**：kubelet 可以在同一个 Pod 内重启容器，但如果 Pod 被删除、节点故障或控制器执行更新，Kubernetes 可能创建一个新的 Pod。新 Pod 通常会拥有新的 IP 和 UID。[2]

### 3.5 静态 Pod

静态 Pod 不由 API Server 直接创建，而是由某个节点上的 kubelet 读取本地 manifest 并负责运行。kubelet 默认会监视：

```text
/etc/kubernetes/manifests/
```

例如，可以在节点上创建：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
    - name: nginx
      image: nginx:stable
```

保存到 `/etc/kubernetes/manifests/static-web.yaml` 后，kubelet 会读取并启动它。静态 Pod 只运行在创建 manifest 的那个节点上；当文件被删除时，kubelet 通常也会停止对应 Pod。[3]

使用 kubeadm 部署的控制平面组件，例如 kube-apiserver、kube-controller-manager、kube-scheduler 和本地 etcd，通常就是静态 Pod。API Server 中可能可以看到它们的 mirror Pod，但真正的运行依据仍然是节点上的本地 manifest。

### 3.6 Init 容器

Init 容器在应用容器启动前按顺序执行。只有所有 Init 容器都成功退出，业务容器才会启动；如果某个 Init 容器失败，kubelet 会依据 Pod 的重启策略重新执行它。[4]

典型用途包括：等待依赖服务、生成配置文件、执行初始化脚本或准备共享 Volume。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: wait-for-api
      image: curlimages/curl:latest
      command:
        - sh
        - -c
        - |
          until curl -fsS http://api.default.svc.cluster.local/healthz; do
            echo "waiting for api"
            sleep 2
          done
  containers:
    - name: web
      image: nginx:stable
```

Init 容器与普通容器的核心区别是：它们是**前置阶段**，不是与业务容器并行运行的 Sidecar。不要把“启动依赖等待”简单写成固定 `sleep 30`，更可靠的做法是轮询依赖服务的健康接口，并设置超时和失败处理。

### 3.7 Labels 与 Annotations

Label 和 Annotation 都属于对象元数据，但用途不同：

| 类型 | 用途 | 能否用于选择器 |
| --- | --- | --- |
| Label | 对资源进行分类、分组和匹配 | 可以 |
| Annotation | 保存说明、配置或工具读取的附加信息 | 不可以 |

例如，Service 通过 Label Selector 找到后端 Pod：

```yaml
selector:
  app: web
```

而 Annotation 常用于 Ingress Controller、监控系统或其他控制器读取额外配置：

```yaml
metadata:
  annotations:
    example.com/owner: platform-team
```

Labels 影响资源关联关系，修改时需要格外谨慎；如果 Deployment 的选择器与 Pod 模板标签不匹配，控制器就无法正确管理副本。

## 四、Deployment：管理无状态应用

### 4.1 Deployment、ReplicaSet 与 Pod 的关系

Deployment 通常用于管理无状态应用。它通过 ReplicaSet 管理 Pod 副本，并提供扩缩容、滚动更新、版本历史和回滚能力。[5]

```text
Deployment
    │ 管理发布版本
    ▼
ReplicaSet
    │ 保证期望副本数
    ▼
Pod
```

用户通常不需要直接操作 ReplicaSet。Deployment 修改 Pod 模板后，控制器会创建新的 ReplicaSet，让新旧 ReplicaSet 按更新策略逐渐完成替换。

### 4.2 创建和查看 Deployment

生成一个不立即创建资源的 YAML 模板：

```bash
kubectl create deployment web09 \
  --image=nginx:stable \
  --dry-run=client \
  -o yaml > web09-deployment.yaml
```

应用并查看：

```bash
kubectl apply -f web09-deployment.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=web09
kubectl describe deployment web09
```

一个更完整的 Deployment 示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web09
  labels:
    app.kubernetes.io/name: web09
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: web09
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web09
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - name: http
              containerPort: 80
          env:
            - name: APP_ENV
              value: learning
```

`selector.matchLabels` 必须能够匹配 `template.metadata.labels`。可以把它理解为 Deployment 对“哪些 Pod 属于我”的判断条件。创建后，Deployment 的 selector 通常不应随意修改，因为这会改变控制器的管理边界。

### 4.3 滚动更新

Deployment 默认使用 RollingUpdate 策略。修改 Pod 模板，例如更新镜像：

```bash
kubectl set image deployment/web09 nginx=nginx:1.27
kubectl rollout status deployment/web09
```

常用滚动更新配置：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  revisionHistoryLimit: 5
```

| 参数 | 作用 |
| --- | --- |
| `maxSurge` | 更新期间允许超过 `replicas` 的最大 Pod 数量 |
| `maxUnavailable` | 更新期间允许不可用的最大 Pod 数量 |
| `minReadySeconds` | Pod Ready 后持续多长时间才被视为可用 |
| `revisionHistoryLimit` | 保留多少个旧 ReplicaSet，供回滚使用 |

例如，`replicas: 3`、`maxSurge: 1`、`maxUnavailable: 0` 表示更新时最多运行 4 个 Pod，并尽量保证至少 3 个副本可用。实际发布速度还会受到镜像拉取、Readiness Probe 和资源配额影响。

### 4.4 版本历史与回滚

```bash
kubectl rollout history deployment/web09
kubectl rollout history deployment/web09 --revision=2
kubectl rollout undo deployment/web09
kubectl rollout undo deployment/web09 --to-revision=2
```

查看状态：

```bash
kubectl get deployment web09
kubectl get rs
kubectl get pods -l app.kubernetes.io/name=web09
```

删除 Deployment 时，通常由它管理的 ReplicaSet 和 Pod 也会被级联删除：

```bash
kubectl delete deployment web09
```

## 五、Service：给变化中的 Pod 提供稳定入口

### 5.1 为什么需要 Service

Pod 会被创建、删除和替换，Pod IP 因此不适合作为稳定的访问地址。Service 提供一个稳定的虚拟 IP 和 DNS 名称，并通过 Selector 找到一组后端 Pod。[6]

```text
客户端
  │ 访问 Service 的 ClusterIP 或 DNS
  ▼
Service
  │ 通过 selector 维护 EndpointSlice
  ▼
多个 Pod
```

Service Controller 会持续根据 Selector 更新 EndpointSlice。客户端不需要知道后端 Pod 当前有哪些实例，也不需要跟踪 Pod IP 的变化。[6]

### 5.2 Service 的端口关系

一个 Service 端口定义通常包含三层概念：

| 字段 | 含义 |
| --- | --- |
| `port` | Service 对外提供的端口 |
| `targetPort` | 转发到后端 Pod 的端口 |
| `nodePort` | NodePort 类型在每个节点监听的端口 |

示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web09
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: web09
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

`port` 和 `targetPort` 可以不同。例如 Service 使用 8080，而后端容器监听 80：

```yaml
ports:
  - port: 8080
    targetPort: 80
```

### 5.3 ClusterIP 与 NodePort

#### ClusterIP

ClusterIP 是默认类型，只能从集群网络内部访问：

```bash
kubectl expose deployment web09 \
  --name=web09 \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP

kubectl get service web09
kubectl get endpointslice -l kubernetes.io/service-name=web09
```

也可以使用 YAML：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web09
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: web09
  ports:
    - port: 80
      targetPort: 80
```

#### NodePort

NodePort 会在每个节点暴露一个端口，外部客户端可以通过 `节点 IP:nodePort` 访问 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web09-nodeport
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: web09
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f web09-nodeport.yaml
kubectl get service web09-nodeport
```

NodePort 的默认端口范围通常是 `30000-32767`，但具体范围以 API Server 的配置为准。NodePort 适合实验和简单场景；生产环境还需要考虑负载均衡、入口控制器和安全策略。

Service 还包括 LoadBalancer 和 ExternalName 等类型。LoadBalancer 通常依赖云平台或外部负载均衡实现，ExternalName 则通过 DNS 名称指向集群外部服务。

#### kube-proxy 的转发模式

Service 的虚拟 IP 转发通常由 kube-proxy 协助完成。不同集群可能配置为 iptables、IPVS 或其他实现。查看 kube-proxy 配置：

```bash
kubectl -n kube-system get configmap kube-proxy -o yaml
```

如果环境使用 IPVS，可以在节点上查看规则：

```bash
ipvsadm -L -n
```

不能简单地认为 IPVS 在所有环境中都一定更好。实际效果取决于 Kubernetes 版本、内核、规则规模、网络模式和运维能力。

### 5.4 Service 排错思路

Service 不通时，应按“选择器 → EndpointSlice → Pod 就绪 → 端口 → 网络路径”的顺序排查：

```bash
kubectl get service web09 -o yaml
kubectl get endpointslice -l kubernetes.io/service-name=web09 -o yaml
kubectl get pods -l app.kubernetes.io/name=web09 --show-labels
kubectl describe service web09
```

如果 EndpointSlice 为空，优先检查 Service Selector 是否与 Pod Labels 完全匹配，以及 Pod 是否通过 Readiness Probe。Service 没有后端时，继续检查 kube-proxy 往往不能解决根本问题。

## 六、Ingress：七层 HTTP/HTTPS 路由

Service 主要提供四层服务入口，而 Ingress 用于根据域名和路径完成 HTTP/HTTPS 路由。Ingress 本身只是 API 对象，必须先安装并运行 Ingress Controller，规则才会被真正执行。[7]

> **理解重点：**创建 Ingress YAML 不等于集群已经具备 HTTP 入口。还需要一个 Ingress Controller，例如 ingress-nginx，并且要让域名解析到 Controller 的入口地址。

示例：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: www.example.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web09
                port:
                  number: 80
```

查看资源：

```bash
kubectl get ingress
kubectl describe ingress web-ingress
```

本地实验可以修改 `/etc/hosts`，把域名解析到 Ingress Controller 所在节点或入口地址：

```text
192.168.8.30 www.example.test
```

然后测试：

```bash
curl -H 'Host: www.example.test' http://192.168.8.30/
```

生产环境还需要配置 TLS、证书管理、访问日志、限流、超时和真实客户端 IP。对于新项目，也应了解 Gateway API，它提供了比 Ingress 更丰富的路由模型。[6]

## 七、DaemonSet：在节点维度运行 Pod

DaemonSet 确保符合条件的每个节点运行一份 Pod。新增节点后，控制器可以自动在新节点创建 Pod；节点被移除后，对应 Pod 也会被清理。[8]

DaemonSet 适合节点级代理，例如：

- 日志采集器；
- 节点监控 Agent；
- CNI 网络组件；
- 存储或安全代理。

最小示例：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox:stable
          command: ["sh", "-c", "while true; do sleep 3600; done"]
```

需要注意，DaemonSet 并不意味着“无条件在每一个节点运行”。节点选择器、Node Affinity、Taint、Toleration 和资源条件都可能影响它的调度范围。系统级 DaemonSet 往往需要配置对控制平面节点污点的容忍度。

查看 DaemonSet：

```bash
kubectl get daemonset -A
kubectl describe daemonset node-agent -n kube-system
```

## 八、调度：决定 Pod 应该运行在哪里

kube-scheduler 会根据资源、约束、亲和性、污点容忍度和其他调度条件，为尚未绑定节点的 Pod 选择合适的节点。[9]

调度规则可以分成两类：

| 类型 | 含义 |
| --- | --- |
| 硬约束 | 不满足时不能调度，例如 `nodeSelector` 或 required affinity |
| 软偏好 | 尽量满足，但没有符合节点时仍可能调度，例如 preferred affinity |

调度并不是简单的“随机找一台机器”。Pod 先要通过节点过滤条件，再根据优先级和评分选择更合适的节点。

### 8.1 nodeSelector：最简单的节点选择

查看节点标签：

```bash
kubectl get nodes --show-labels
```

给节点添加标签：

```bash
kubectl label node node1 disk=ssd
kubectl label node node2 workload=web
```

删除标签时，在键后加 `-`：

```bash
kubectl label node node1 disk-
```

Pod 使用 `nodeSelector`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-on-ssd
spec:
  nodeSelector:
    disk: ssd
  containers:
    - name: nginx
      image: nginx:stable
```

`nodeSelector` 是最简单的硬性匹配方式。Pod 只有在节点同时具备指定标签时，才有资格被调度到该节点。[9]

### 8.2 节点亲和性与反亲和性

Node Affinity 比 `nodeSelector` 表达能力更强，可以表达“必须满足”和“尽量满足”两类规则：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disk
                operator: In
                values:
                  - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: workload
                operator: In
                values:
                  - web
  containers:
    - name: nginx
      image: nginx:stable
```

| 规则 | 调度行为 |
| --- | --- |
| `requiredDuringSchedulingIgnoredDuringExecution` | 必须满足，否则 Pod 无法调度 |
| `preferredDuringSchedulingIgnoredDuringExecution` | 尽量满足，不满足时仍允许调度 |

`IgnoredDuringExecution` 表示 Pod 已经运行后，即使节点标签发生变化，规则通常不会因此自动驱逐该 Pod。Node Affinity 适合表达节点属性；Pod Affinity 和 Anti-Affinity 则可以根据其他 Pod 的标签控制共置或分散。[9]

例如，反亲和可以尽量避免同一个应用的多个副本落在同一节点，从而减少单节点故障带来的影响。

### 8.3 Taints 与 Tolerations

Taint 加在节点上，表达“这个节点不希望某些 Pod 进入”；Toleration 写在 Pod 上，表达“这个 Pod 可以接受某种 Taint”。两者配合使用，形成节点排斥机制。[10]

设置污点：

```bash
kubectl taint nodes node1 performance=low:PreferNoSchedule
kubectl taint nodes node2 dedicated=control-plane:NoSchedule
```

删除某个键对应的污点：

```bash
kubectl taint nodes node1 performance-
```

三种常见 effect：

| Effect | 行为 |
| --- | --- |
| `PreferNoSchedule` | 尽量不调度到该节点，但不是绝对禁止 |
| `NoSchedule` | 没有匹配容忍度的新 Pod 不会被调度到该节点 |
| `NoExecute` | 没有匹配容忍度的既有 Pod 也可能被驱逐 |

Pod 增加容忍度：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-demo
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: control-plane
      effect: NoSchedule
  containers:
    - name: nginx
      image: nginx:stable
```

需要明确：**Toleration 只表示“允许进入”，不表示“必须调度到该节点”。** 如果希望 Pod 必须去某类节点，还需要配合 `nodeSelector` 或 Node Affinity。

`NoExecute` 还可以设置 `tolerationSeconds`，让 Pod 在容忍一段时间后被驱逐：

```yaml
tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

### 8.4 调度失败时如何分析

当 Pod 处于 `Pending`，不要只执行 `kubectl get pods`。应查看 Pod 事件：

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

常见原因包括：

- `nodeSelector` 或 required affinity 没有匹配节点；
- 节点存在 `NoSchedule` 或 `NoExecute` 污点；
- Pod 请求的 CPU、内存超过节点可分配资源；
- PVC 没有绑定；
- 命名空间资源配额不足；
- 节点处于 `NotReady`；
- 镜像问题导致 Pod 已调度但容器无法启动。

要区分“调度失败”和“容器启动失败”：

```text
Pending                 可能尚未找到合适节点
ContainerCreating       已调度，正在创建容器或挂载网络/存储
ImagePullBackOff        已调度，但镜像拉取失败
CrashLoopBackOff        容器启动后反复退出
```

## 九、常用查询与排错命令

### 9.1 API、资源和标签

```bash
kubectl api-versions
kubectl api-resources
kubectl get all -A
kubectl get pods -A -o wide
kubectl get pods --show-labels
kubectl get nodes --show-labels
```

### 9.2 查看对象详细信息

```bash
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
kubectl describe service <service-name>
kubectl describe ingress <ingress-name>
kubectl describe node <node-name>
```

`describe` 的 Events 区域经常能直接暴露调度、挂载、镜像、探针和网络问题。

### 9.3 YAML 与字段检查

```bash
kubectl get deployment web09 -o yaml
kubectl get service web09 -o yaml
kubectl get pod <pod-name> -o yaml
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.affinity
```

### 9.4 应用和删除资源

```bash
kubectl apply -f app.yaml
kubectl diff -f app.yaml
kubectl delete -f app.yaml
```

`kubectl diff` 可以在应用前查看本地 YAML 与集群现状之间的差异，适合减少误操作。

## 十、从资源对象到控制器的理解总结

学习这些资源时，不要把它们看成彼此孤立的 YAML 文件。它们实际上共同构成了 Kubernetes 的控制闭环：

| 学习对象 | 应该建立的理解 |
| --- | --- |
| Pod | 应用运行的最小逻辑主机，不是长期稳定的服务器 |
| Init Container | 应用启动前的顺序化准备阶段 |
| Static Pod | kubelet 直接管理的节点本地工作负载 |
| Deployment | 面向无状态应用的版本、副本和发布控制器 |
| ReplicaSet | Deployment 用来维持 Pod 副本数量的中间控制器 |
| Service | 用稳定入口解耦客户端与短暂的 Pod 实例 |
| Ingress | 通过域名和路径把 HTTP/HTTPS 流量路由到 Service |
| DaemonSet | 以节点为维度部署节点级代理 |
| Label | 建立资源关联和选择器匹配关系 |
| Annotation | 为控制器、工具和运维系统提供附加元数据 |
| nodeSelector/Affinity | 表达 Pod 对节点属性的要求或偏好 |
| Taint/Toleration | 表达节点对 Pod 的排斥以及 Pod 的接纳能力 |

真正理解 Kubernetes 的标志，不是能够背出多少命令，而是能够从“期望状态、控制器、标签选择器、调度约束和事件”这几个角度解释资源为什么处于当前状态。

## 十一、适合 CKA 的练习顺序

可以按照下面的顺序在实验集群中练习：

1. 使用 `kubectl run` 创建 Pod，查看日志、进入容器并删除 Pod。
2. 使用 YAML 创建 Pod，练习 Labels、环境变量、Init Container 和容器端口声明。
3. 创建 Deployment，调整副本数，修改镜像并观察滚动更新。
4. 查看 ReplicaSet 和 rollout history，执行版本回滚。
5. 为 Deployment 创建 ClusterIP Service，检查 Selector 和 EndpointSlice。
6. 创建 NodePort Service，从节点外部访问应用。
7. 安装 Ingress Controller，配置基于域名的 Ingress 路由。
8. 创建 DaemonSet，观察新增节点后的自动部署行为。
9. 给节点打标签，使用 `nodeSelector` 和 Node Affinity 控制调度。
10. 给节点设置不同 effect 的 Taint，并为 Pod 增加 Toleration。
11. 故意制造错误，例如写错标签、设置不存在的节点标签或提高资源请求，再使用 `describe` 和 Events 分析问题。

## 十二、总结

Kubernetes 的核心能力来自多个组件之间的协作：API Server 保存对象，控制器观察对象并推动状态变化，Scheduler 负责为待调度 Pod 选择节点，kubelet 在节点上落实运行状态，网络和存储插件为 Pod 提供基础设施能力。

从学习路径看，Pod 是理解 Kubernetes 的起点，但不是终点。只有把 Pod 放进 Deployment，放到 Service 后面，再结合 Ingress、DaemonSet 和调度约束，才能真正理解 Kubernetes 如何把“一个容器”变成“可扩展、可更新、可发现、可调度的应用系统”。

> **核心记忆：**Pod 负责运行，控制器负责维持，Service 负责发现，Ingress 负责路由，Scheduler 负责放置，kubelet 负责落实。

## 参考资料

1. [Kubernetes 官方：Objects in Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
2. [Kubernetes 官方：Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
3. [Kubernetes 官方：Create static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
4. [Kubernetes 官方：Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
5. [Kubernetes 官方：Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
6. [Kubernetes 官方：Service](https://kubernetes.io/docs/concepts/services-networking/service/)
7. [Kubernetes 官方：Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
8. [Kubernetes 官方：DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
9. [Kubernetes 官方：Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
10. [Kubernetes 官方：Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)