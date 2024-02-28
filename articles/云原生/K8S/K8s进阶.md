# 架构

## 节点

Kubernetes中节点（node）指的是一个工作机器。

不同的集群中，节点可能是虚拟机也可能是物理机。每个节点都由 master 组件管理，并包含了运行 Pod（容器组）所需的服务。这些服务包括：

- [容器引擎](https://kuboard.cn/learning/k8s-bg/component.html#容器引擎)
- kubelet
- kube-proxy 

### 节点状态

```
kubectl get nodes -o wide #查看所有节点的列表
kubectl describe node <your-node-name> #查看节点状态以及节点的其他详细信息
```

节点的状态包含如下信息：

- Addresses
- Conditions
- Capacity and Allocatable
- Info

依次看一下

#### Addresses

依据你集群部署的方式（在哪个云供应商部署，或是在物理机上部署），Addesses 字段可能有所不同

```
Addresses:
  InternalIP:  192.168.59.132
  Hostname:    k8s-worker1
```

- HostName： 在节点命令行界面上执行 `hostname` 命令所获得的值。启动 kubelet 时，可以通过参数 `--hostname-override` 覆盖
- ExternalIP：通常是节点的外部IP（可以从集群外访问的内网IP地址；上面的例子中，此字段为空）
- InternalIP：通常是从节点内部可以访问的 IP 地址

#### Conditions

描述了节点的状态

| Node Condition    | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| OutOfDisk         | 如果节点上的空白磁盘空间不够，不能够再添加新的节点时，该字段为 `True`，其他情况为 `False` |
| Ready             | 如果节点是健康的且已经就绪可以接受新的 Pod。则节点Ready字段为 `True`。`False`表明了该节点不健康，不能够接受新的 Pod。 |
| MemoryPressure    | 如果节点内存紧张，则该字段为 `True`，否则为`False`           |
| PIDPressure       | 如果节点上进程过多，则该字段为 `True`，否则为 `False`        |
| DiskPressure      | 如果节点磁盘空间紧张，则该字段为 `True`，否则为 `False`      |
| NetworkUnvailable | 如果节点的网络配置有问题，则该字段为 `True`，否则为 `False`  |

![image-20240119192449438](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240119192449438.png)

> 某些情况下（例如，节点网络故障），apiserver 不能够与节点上的 kubelet 通信，删除 Pod 的指令不能下达到该节点的 kubelet 上，直到 apiserver 与节点的通信重新建立，指令才下达到节点。这意味着，虽然对 Pod 执行了删除的调度指令，但是这些 Pod 可能仍然在失联的节点上运行。

#### Capacity and Allocatable（容量和可分配量）

```
Capacity:
  cpu:                8
  ephemeral-storage:  19946096Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3963536Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  18382322044
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3861136Ki
  pods:               110
```

容量和可分配量（Capacity and Allocatable）描述了节点上的可用资源的情况：

- CPU
- 内存
- 该节点可调度的最大 pod 数量

Capacity 中的字段表示节点上的资源总数，Allocatable 中的字段表示该节点上可分配给普通 Pod 的资源总数。

#### Info

描述了节点的基本信息，例如：

- Linux 内核版本
- Kubernetes 版本（kubelet 和 kube-proxy 的版本）
- Docker 版本
- 操作系统名称

这些信息由节点上的 kubelet 收集。

```
System Info:
  Machine ID:                 cd3ad9b41d3a4c2496395d89f2504ffe
  System UUID:                776f4d56-6491-392e-b44f-ccfbecbae69f
  Boot ID:                    468ad415-b607-4f5a-bea3-10c0f2e903d8
  Kernel Version:             6.5.0-14-generic
  OS Image:                   Ubuntu 22.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://24.0.5
  Kubelet Version:            v1.22.7
  Kube-Proxy Version:         v1.22.7
```

### 节点管理

如果成功搭建了k8s，并且将节点加入了集群当中。很显然，节点并不是由k8s创建的，而是由我们自己创建并加入的。

向 Kubernetes 中创建节点时，仅仅是创建了一个描述该节点的 API 对象。节点 API 对象创建成功后，Kubernetes将检查该节点是否有效。

> Kubernetes 将保留无效的节点 API 对象，并不断地检查该节点是否有效。除非您使用 `kubectl delete node my-first-k8s-node` 命令删除该节点。

#### 节点控制器（Node Controller）

节点控制器是一个负责管理节点的 Kubernetes master 组件

首先，节点控制器在注册节点是为节点分配CIDR地址块

第二，通过cloud-controller-manager检查节点列表中每一个节点对象对应的虚拟机是否可用。在云环境中，只要节点状态异常，节点控制器检查其虚拟机在云供应商的状态，如果虚拟机不可用，自动将节点对象从 APIServer 中删除。

第三，节点控制器监控节点的健康状况。当节点变得不可触达时（例如，由于节点已停机，节点控制器不再收到来自节点的心跳信号），节点控制器将节点API对象的 `NodeStatus` Condition 取值从 `NodeReady` 更新为 `Unknown`；然后在等待 `pod-eviction-timeout` 时间后，将节点上的所有 Pod 从节点驱逐。

- 默认40秒未收到心跳，修改 `NodeStatus` Condition 为 `Unknown`；
- 默认 `pod-eviction-timeout` 为 5分钟
- 节点控制器每隔 `--node-monitor-period` 秒检查一次节点的状态

#### 节点自注册（Self-Registration）

如果 kubelet 的启动参数 `--register-node`为 true（默认为 true），kubelet 会尝试将自己注册到 API Server。kubelet自行注册时，将使用如下选项：

- `--kubeconfig`：向 apiserver 进行认证时所用身份信息的路径
- `--cloud-provider`：向云供应商读取节点自身元数据
- `--register-node`：自动向 API Server 注册节点
- `--register-with-taints`：注册节点时，为节点添加污点（逗号分隔，格式为 <key>=<value>:<effect>
- `--node-ip`：节点的 IP 地址
- `--node-labels`：注册节点时，为节点添加标签
- `--node-status-update-frequency`：向 master 节点发送心跳信息的时间间隔

如果 [Node authorization mode (opens new window)](https://kubernetes.io/docs/reference/access-authn-authz/node/)和 [NodeRestriction admission plugin (opens new window)](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)被启用，kubelet 只拥有创建/修改其自身所对应的节点 API 对象的权限。

#### 手动管理节点

集群管理员可以创建和修改节点API对象。

如果管理员想要手工创建节点API对象，可以将 kubelet 的启动参数 `--register-node` 设置为 false。

不设置这个参数，管理员能够修改的地方有

- 增加/减少标签
- 标记节点为不可调度（unschedulable）

节点的标签与 Pod 上的节点选择器（node selector）配合，可以控制调度方式，例如，限定 Pod 只能在某一组节点上运行。

执行如下命令可将节点标记为不可调度（unschedulable），此时将阻止新的 Pod 被调度到该节点上，但是不影响任何已经在该节点上运行的 Pod。这在准备重启节点之前非常有用。

```
kubectl cordon $NODENAME
```

#### 节点容量（Node Capacity）

节点API对象中描述了节点的容量（Capacity），例如，CPU数量、内存大小等信息。通常，节点在向 APIServer 注册的同时，在节点API对象里汇报了其容量（Capacity）。

在查看节点描述时就能看到

## 集群内通信

### Master-Node之间的通信

Kubernetes集群和Master节点（实际上是 apiserver）之间的通信路径。

#### Cluster to Master

所有从集群访问 Master 节点的通信，都是针对 apiserver 的（没有任何其他 master 组件发布远程调用接口）。API 服务器被配置为在一个安全的 HTTPS 端口（通常为 443）上监听远程连接请求， 并启用一种或多种形式的客户端[身份认证](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/)机制。 一种或多种客户端[鉴权机制](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/)应该被启用， 特别是在允许使用[匿名请求](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#anonymous-requests) 或[服务账户令牌](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#service-account-tokens)的时候。

想要连接到 API 服务器的 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 可以使用服务账号安全地进行连接。 当 Pod 被实例化时，Kubernetes 自动把公共根证书和一个有效的持有者令牌注入到 Pod 里。 `kubernetes` 服务（位于 `default` 名字空间中）配置了一个虚拟 IP 地址（默认为 10.96.0.1）， 用于（通过 `kube-proxy`）转发请求到 API 服务器的 HTTPS 末端。

#### Master to Cluster

从 master（apiserver）到Cluster存在着两条主要的通信路径：

- apiserver 访问集群中每个节点上的 kubelet 进程
- 使用 apiserver 的 proxy 功能，从 apiserver 访问集群中的任意节点、Pod、Service

##### apiserver to kubelet

- 抓取 Pod 的日志
- 通过 `kubectl exec -it` 指令获得容器的命令行终端
- 提供 `kubectl port-forward` 功能（提供 kubelet 的端口转发功能。）

这些连接终止于 kubelet 的 HTTPS 末端。 默认情况下，API 服务器不检查 kubelet 的服务证书。这使得此类连接容易受到中间人攻击， 在非受信网络或公开网络上运行也是 **不安全的**。

为了对这个连接进行认证，使用 `--kubelet-certificate-authority` 标志给 API 服务器提供一个根证书包，用于 kubelet 的服务证书。

如果无法实现这点，又要求避免在非受信网络或公共网络上进行连接，可在 API 服务器和 kubelet 之间使用 [SSH 隧道](https://kubernetes.io/zh-cn/docs/concepts/architecture/control-plane-node-communication/#ssh-tunnels)。

最后，应该启用 [Kubelet 认证/鉴权](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-authn-authz/) 来保护 kubelet API

##### apiserver to nodes, pods, services

从 API 服务器到节点、Pod 或服务的连接默认为纯 HTTP 方式，因此既没有认证，也没有加密。 这些连接可通过给 API URL 中的节点、Pod 或服务名称添加前缀 `https:` 来运行在安全的 HTTPS 连接上。 不过这些连接既不会验证 HTTPS 末端提供的证书，也不会提供客户端证书。 因此，虽然连接是加密的，仍无法提供任何完整性保证。 这些连接 **目前还不能安全地** 在非受信网络或公共网络上运行。

> 实际上，API服务器并不直接与pod进行通信，而是通过kubelet和kube-proxy进行通信。当你创建、更新或删除一个pod时，这个操作会首先发送到API服务器，然后再被转发给对应的kubelet进行处理。API服务器也负责接收和存储kubelet上报的Pod状态信息。

##### SSH

Kubernetes 支持使用 [SSH 隧道](https://www.ssh.com/academy/ssh/tunneling)来保护从控制面到节点的通信路径。 在这种配置下，API 服务器建立一个到集群中各节点的 SSH 隧道（连接到在 22 端口监听的 SSH 服务器） 并通过这个隧道传输所有到 kubelet、节点、Pod 或服务的请求。 这一隧道保证通信不会被暴露到集群节点所运行的网络之外。

> SSH 隧道目前已被废弃。除非你了解个中细节，否则不应使用。 [Konnectivity 服务](https://kubernetes.io/zh-cn/docs/concepts/architecture/control-plane-node-communication/#konnectivity-service)是 SSH 隧道的替代方案。

## 控制器

在机器人技术和自动化领域，控制回路（Control Loop）是一个非终止回路，用于调节系统状态。

我们会对其设置一个期望状态(Desired State)，对于实例本身来说是当前状态(Current State)。需求是自动将当前状态变得接近期望状态。

在 Kubernetes 中，控制器通过监控[集群](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-cluster) 的公共状态，并致力于将当前状态转变为期望的状态。

### 控制器模式

一个控制器至少追踪一种类型的 Kubernetes 资源。这些 [对象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/#kubernetes-objects) 有一个代表期望状态的 `spec` 字段。 该资源的控制器负责确保其当前状态接近期望状态。

控制器可能会自行执行操作；在 Kubernetes 中更常见的是一个控制器会发送信息给 [API 服务器](https://kubernetes.io/zh-cn/docs/concepts/overview/components/#kube-apiserver)，这会有副作用。 具体可参看后文的例子。

#### 通过 API 服务器来控制

Job Controller是k8s的一个内置控制器，内置控制器通过和集群 API 服务器交互来管理状态。

Job 是一种 Kubernetes 资源，它运行一个或者多个 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)， 来执行一个任务然后停止。 （一旦[被调度了](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/)，对 `kubelet` 来说 Pod 对象就会变成了期望状态的一部分）。

Job并不自己运行Pod，当Job获取到新任务时，它会保证一组 Node 节点上的 `kubelet` 可以运行正确数量的 Pod 来完成工作。Job通知API服务器创建还是移除Pod。控制面当中的其他组件会根据消息完成pod的创建或移除。

创建新 Job 后，所期望的状态就是完成这个 Job。Job 控制器会让 Job 的当前状态不断接近期望状态：创建为 Job 要完成工作所需要的 Pod，使 Job 的状态接近完成。

控制器也会更新配置对象。例如：一旦 Job 的工作完成了，Job 控制器会更新 Job 对象的状态为 `Finished`。

[关于Job](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/job/)

#### 直接控制

有些比较特殊的控制器需要对集群以外的东西进行修改

例如，如果你使用一个控制回路来保证集群中有足够的 [节点](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)，那么控制器就需要当前集群外的 一些服务在需要时创建新节点。和外部状态交互的控制器从 API 服务器获取到它想要的状态，然后直接和外部系统进行通信 并使当前状态更接近期望状态。

> 这里的重点是，控制器做出了一些变更以使得事物更接近你的期望状态， 之后将当前状态报告给集群的 API 服务器。 其他控制回路可以观测到所汇报的数据的这种变化并采取其各自的行动。

### 期望状态&当前状态

Kubernetes 采用了系统的云原生视图，并且可以处理持续的变化。

在任务执行时，集群随时都可能被修改，并且控制回路会自动修复故障。 这意味着很可能集群永远不会达到稳定状态。

只要集群中的控制器在运行并且进行有效的修改，整体状态的稳定与否是无关紧要的。

#### 设计

作为设计原则之一，Kubernetes 使用了很多控制器，每个控制器管理集群状态的一个特定方面。 最常见的一个特定的控制器使用一种类型的资源作为它的期望状态， 控制器管理控制另外一种类型的资源向它的期望状态演化。

> 可能存在多种控制器可以创建或更新相同类型的 API 对象。为了避免混淆，Kubernetes 控制器在创建新的 API 对象时，会将该对象与对应的控制 API 对象关联，并且只关注与控制对象关联的那些对象。
>
> - 例如，Deployment 和 Job，这两类控制器都创建 Pod。Job Controller 不会删除 Deployment Controller 创建的 Pod，因为控制器可以通过标签信息区分哪些 Pod 是它创建的。

### 运行控制器的方式

Kubernetes 内置一组控制器，运行在 [kube-controller-manager](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-controller-manager/) 内。 这些内置的控制器提供了重要的核心功能。

Deployment 控制器和 Job 控制器是 Kubernetes 内置控制器的典型例子。 Kubernetes 允许你运行一个稳定的控制平面，这样即使某些内置控制器失败了， 控制平面的其他部分会接替它们的工作。

你会遇到某些控制器运行在控制面之外，用以扩展 Kubernetes。 或者，如果你愿意，你也可以自己编写新控制器。 你可以以一组 Pod 来运行你的控制器，或者运行在 Kubernetes 之外。 最合适的方案取决于控制器所要执行的功能是什么。

# 操作Kubenets

## Kubernetes 对象

### 理解 Kubernetes 对象

在 Kubernetes 系统中，**Kubernetes 对象**是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。 具体而言，它们描述了如下信息：

- 哪些容器化应用正在运行（以及在哪些节点上运行）
- 可以被应用使用的资源
- 关于应用运行时行为的策略，比如重启策略、升级策略以及容错策略

Kubernetes 对象是一种“意向表达（Record of Intent）”。一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。通过创建对象，你本质上是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的**期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api)。

#### 对象规约（Spec）与状态（Status)

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置

+ spec(规约) ：对于具有 `spec` 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： **期望状态（Desired State）**。
+ Status:描述了对象的**当前状态（Current State）**，它是由 Kubernetes 系统和组件设置并更新的。

> 举个简单的例子，我们设置spec为三个实例，当其中一个实例down掉之后，k8s会帮我们修正，起一个新的实例，将状态恢复到3

#### 描述kubernetes对象

创建 Kubernetes 对象时，必须提供对象的 `spec`，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。

使用API创建对象，我们一般通过yaml格式的文件，k8s会自动将清单转化为json或者其他支持的文件，看下官方给出的对象的例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 告知 Deployment 运行 2 个与该模板匹配的 Pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```

显然这是一个deployment对象

#### 必需字段

在想要创建的 Kubernetes 对象所对应的清单（YAML 或 JSON 文件）中，需要配置的字段如下：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一标识对象的一些数据，包括一个 `name` 字符串、`UID` 和可选的 `namespace`
- `spec` - 你所期望的该对象的状态

 [Kubernetes API 参考](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/)

## 管理k8s对象

> 关于这部分，感觉都是一些命令使用的关系
>
> 注意一点，k8s环境中最好只是用一种方式进行对象的创建

## 对象名称和ID

集群中的每一个[对象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/#kubernetes-objects)都有一个[**名称**](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/#names)来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个 [**UID**](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/#uids) 来标识在整个集群中的唯一性。

> pod和deployment都可以为同一个名称，这种非唯一的属性Kubernetes 提供了[标签（Label）](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)和 [注解（Annotation）](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/annotations/)机制。

### 名称

客户端提供的字符串，引用资源 URL 中的对象，如`/api/v1/pods/some name`。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

**名称在同一资源的所有 [API 版本](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)中必须是唯一的。 这些 API 资源通过各自的 API 组、资源类型、名字空间（对于划分名字空间的资源）和名称来区分。 换言之，API 版本在此上下文中是不相关的。**

> 当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

比较常见的四种资源命名约束:DNS子域名，RFC1123标签名，RFC1035标签名，路径分段名称

详细了解[名称](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/)

### UID

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 UID，它旨在区分类似实体的历史事件。

Kubernetes UID 是全局唯一标识符（也叫 UUID）。 UUID 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667。

## 名称空间

NameSpace负责提供一种机制，将同一集群中的资源划分为相互隔离的组。 同一名字空间内的资源名称要唯一，但跨名字空间时没有这个要求。名字空间作用域仅针对带有名字空间的[对象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/#kubernetes-objects)， （例如 Deployment、Service 等），这种作用域对集群范围的对象 （例如 StorageClass、Node、PersistentVolume 等）不适用。

> 大概知道是什么概念就好，和linx的命名空间还算相似，这里就只看看怎么创建和使用了

### Use

查看namespace

```
kubectl get namespace
```

设置

 使用--namespace参数

```
kubectl run nginx --image=nginx --namespace=<名字空间名称>
kubectl get pods --namespace=<名字空间名称>
```

设置名字空间偏好

你可以永久保存名字空间，以用于对应上下文中所有后续 kubectl 命令。

```
kubectl config set-context --current --namespace=<名字空间名称>
# 验证
kubectl config view --minify | grep namespace:
```

### 名字空间和 DNS

创建一个服务，k8s会添加一条对应的DNS

该条目的形式是 `<服务名称>.<名字空间名称>.svc.cluster.local`，这意味着如果容器只使用 `<服务名称>`，它将被解析到本地名字空间的服务。这对于跨多个名字空间（如开发、测试和生产） 使用相同的配置非常有用。如果你希望跨名字空间访问，则需要使用完全限定域名（FQDN）。

### 并非所有对象都在名字空间中

底层资源， 例如[节点](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)和[持久化卷](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)不属于任何名字空间。

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：

```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

## 标签和选择器

标签（Label）是附加在Kubernetes对象上的一组名值对，其意图是按照对用户有意义的方式来标识Kubernetes对象，同时，又不对Kubernetes的核心逻辑产生影响。标签可以用来组织和选择一组Kubernetes对象。您可以在创建Kubernetes对象时为其添加标签，也可以在创建以后再为其添加标签。每个Kubernetes对象可以有多个标签，同一个对象的标签的 Key 必须唯一，例如：

```
metadata:
  labels:
    key1: value1
    key2: value2
```

> 标签注意名称即可，主要看标签选择器

### 标签选择器

标签不一定是唯一的。通常来讲，会有多个Kubernetes对象包含相同的标签。通过使用标签选择器（label selector），用户/客户端可以选择一组对象。标签选择器（label selector）是 Kubernetes 中最主要的分类和筛选手段。

#### 基于等式的选择方式

可以使用三种操作符 `=`、`==`、`!=`。前两个操作符含义是一样的，都代表相等，后一个操作符代表不相等。例如：

```
# 选择了标签名为 `environment` 且 标签值为 `production` 的Kubernetes对象
environment = production
# 选择了标签名为 `tier` 且标签值不等于 `frontend` 的对象，以及不包含标签 `tier` 的对象
tier != frontend
```

也可以使用逗号分隔的两个等式 `environment=production,tier!=frontend`，此时将选中所有 `environment` 为 `production` 且 `tier` 不为 `frontend` 的对象。

#### 基于集合的选择方式

Set-based 标签选择器可以根据标签名的一组值进行筛选。支持的操作符有三种：`in`、`notin`、`exists`。例如：

```
# 选择所有的包含 `environment` 标签且值为 `production` 或 `qa` 的对象
environment in (production, qa)
# 选择所有的 `tier` 标签不为 `frontend` 和 `backend`的对象，或不含 `tier` 标签的对象
tier notin (frontend, backend)
# 选择所有包含 `partition` 标签的对象
partition
# 选择所有不包含 `partition` 标签的对象
!partition
```

可以组合多个选择器，用 `,` 分隔，`,` 相当于 `AND` 操作符。

### API

#### 查询

LIST 和 WATCH 操作时，可指定标签选择器作为查询条件，以筛选指定的对象集合。两种选择方式都可以使用，但是要符合 URL 编码，例如：

- 基于等式的选择方式： `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
- 基于集合的选择方式： `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

或者

```
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

####  Kubernetes对象引用

介绍服务对对象的引用

Service 中通过 `spec.selector` 字段来选择一组 Pod，并将服务请求转发到选中的 Pod 上。

```
selector:
  component: redis
```

# 容器

## 容器镜像

容器镜像通常会被赋予 `pause`、`example/mycontainer` 或者 `kube-apiserver` 这类的名称。 镜像名称也可以包含所在仓库的主机名。例如：`fictional.registry.example/imagename`。 还可以包含仓库的端口号，例如：`fictional.registry.example:10443/imagename`。

如果你不指定仓库的主机名，Kubernetes 认为你在使用 Docker 公共仓库。

在镜像名称之后，你可以添加一个**标签（Tag）**（与使用 `docker` 或 `podman` 等命令时的方式相同）。 使用标签能让你辨识同一镜像序列中的不同版本。

## 容器环境

Kubernetes 的容器环境给容器提供了几个重要的资源：

- 文件系统，其中包含一个[镜像](https://kubernetes.io/zh-cn/docs/concepts/containers/images/) 和一个或多个的[卷](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/)
- 容器自身的信息
- 集群中其他对象的信息

### 容器信息

一个容器的 **hostname** 是该容器运行所在的 Pod 的名称。通过 `hostname` 命令或者调用 libc 中的 [`gethostname`](https://man7.org/linux/man-pages/man2/gethostname.2.html) 函数可以获取该名称。

Pod 名称和命名空间可以通过 [下行 API](https://kubernetes.io/zh-cn/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) 转换为环境变量。

Pod 定义中的用户所定义的环境变量也可在容器中使用，就像在 container 镜像中静态指定的任何环境变量一样。

### 集群信息

创建容器时正在运行的所有服务都可用作该容器的环境变量。 这里的服务仅限于新容器的 Pod 所在的名字空间中的服务，以及 Kubernetes 控制面的服务。

对于名为 **foo** 的服务，当映射到名为 **bar** 的容器时，定义了以下变量：

```
FOO_SERVICE_HOST=<其上服务正运行的主机>
FOO_SERVICE_PORT=<其上服务正运行的端口>
```

## Runtime Class

RuntimeClass 是一个用于选择容器运行时配置的特性，容器运行时配置用于运行 Pod 中的容器。

[Runtime Class](https://kubernetes.io/zh-cn/docs/concepts/containers/runtime-class/)

## 容器生命周期

> 沟槽的hook

[钩子](https://kubernetes.io/zh-cn/docs/concepts/containers/container-lifecycle-hooks/)

### 事件处理

用到了上面的两个钩子函数，直接看例子

```
lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

Kubernetes 在容器启动后立刻发送 postStart 事件，但是并不能确保 postStart 事件处理程序在容器的 EntryPoint 之前执行。postStart 事件处理程序相对于容器中的进程来说是异步的（同时执行），然而，Kubernetes 在管理容器时，将一直等到 postStart 事件处理程序结束之后，才会将容器的状态标记为 Running。

Kubernetes 在决定关闭容器时，立刻发送 preStop 事件，并且，将一直等到 preStop 事件处理程序结束或者 Pod 的 `--grace-period` 超时，才删除容器。

# 工作负载

## Pod容器组

有几个需要注意的点

Pod 为其成员容器提供了两种类型的共享资源：网络和存储

- 网络 Networking

  每一个 Pod 被分配一个独立的 IP 地址。Pod 中的所有容器共享一个网络名称空间：

  - 同一个 Pod 中的所有容器 IP 地址都相同
  - 同一个 Pod 中的不同容器不能使用相同的端口，否则会导致端口冲突
  - 同一个 Pod 中的不同容器可以通过 localhost:port 进行通信
  - 同一个 Pod 中的不同容器可以通过使用常规的进程间通信手段，例如 SystemV semaphores 或者 POSIX 共享内存

  TIP

  不同 Pod 上的两个容器如果要通信，必须使用对方 Pod 的 IP 地址 + 对方容器的端口号 进行网络通信

+ 存储 Storage

  Pod 中可以定义一组共享的数据卷。Pod 中所有的容器都可以访问这些共享数据卷，以便共享数据。Pod 中数据卷的数据也可以存储持久化的数据，使得容器在重启后仍然可以访问到之前存入到数据卷中的数据。

## 控制器

Kubernetes 通过引入 Controller（控制器）的概念来管理 Pod 实例。在 Kubernetes 中，您应该始终通过创建 Controller 来创建 Pod，而不是直接创建 Pod。**控制器可以提供如下特性：**

- 水平扩展（运行 Pod 的多个副本）
- rollout（版本更新）
- self-healing（故障恢复） 例如：当一个节点出现故障，控制器可以自动地在另一个节点调度一个配置完全一样的 Pod，以替换故障节点上的 Pod。

> 重点介绍Deployment

### Deployment

Deployment 是最常用的用于部署无状态服务的方式。Deployment 控制器使得您能够以声明的方式更新 Pod（容器组）和 ReplicaSet（副本集）。

+ 创建

  直接看例子

  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels:
      app: nginx
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
          - containerPort: 80
  ```

  - 将创建一个名为 nginx-deployment 的 Deployment（部署），名称由 `.metadata.name` 字段指定
  - 该 Deployment 将创建 3 个 Pod 副本，副本数量由 `.spec.replicas` 字段指定
  - `.spec.selector` 字段指定了 Deployment 如何找到由它管理的 Pod。此案例中，我们使用了 Pod template 中定义的一个标签（app: nginx）。对于极少数的情况，这个字段也可以定义更加复杂的规则
  - .template 字段包含了如下字段：
    - `.template.metadata.labels` 字段，指定了 Pod 的标签（app: nginx）
    - `.template.spec.containers[].image` 字段，表明该 Pod 运行一个容器 `nginx:1.7.9`
    - `.template.spec.containers[].name` 字段，表明该容器的名字是 `nginx`

+ 更新

  当且仅当 Deployment 的 Pod template（`.spec.template`）字段中的内容发生变更时（例如标签、容器的镜像被改变），Deployment 的发布更新（rollout）将被触发。Deployment 中其他字段的变化（例如修改 .spec.replicas 字段）将不会触发 Deployment 的发布更新（rollout）

+ 回滚

  ```
  kubectl rollout history deployment.v1.apps/nginx-deployment #检查历史版本
  kubectl rollout undo deployment.v1.apps/nginx-deployment #回滚到上一个版本
  或者指定版本
  kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
  ```

+ 伸缩

  ```
  kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
  在启用自动伸缩的情况下，可以基于最大最小之间进行伸缩
  kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
  ```

### StatefulSet

StatefulSet 顾名思义，用于管理 Stateful（有状态）的应用程序。

StatefulSet 管理 Pod 时，确保其 Pod 有一个按顺序增长的 ID。

与 [Deployment](https://kuboard.cn/learning/k8s-intermediate/workload/wl-deployment/) 相似，StatefulSet 基于一个 Pod 模板管理其 Pod。与 Deployment 最大的不同在于 StatefulSet 始终将一系列不变的名字分配给其 Pod。这些 Pod 从同一个模板创建，但是并不能相互替换：每个 Pod 都对应一个特有的持久化存储标识。

同其他所有控制器一样，StatefulSet 也使用相同的模式运作：用户在 StatefulSet 中定义自己期望的结果，StatefulSet 控制器执行需要的操作，以使得该结果被达成。

#### 使用场景

对于有如下要求的应用程序，StatefulSet 非常适用：

- 稳定、唯一的网络标识（dnsname）
- 每个Pod始终对应各自的存储路径（PersistantVolumeClaimTemplate）
- 按顺序地增加副本、减少副本，并在减少副本时执行清理
- 按顺序自动地执行滚动更新

如果一个应用程序不需要稳定的网络标识，或者不需要按顺序部署、删除、增加副本，您应该考虑使用 Deployment 这类无状态（stateless）的控制器。

# 服务发现，负载均衡，网络

## Service

service存在的意义是为了解决pod不断变化的ip的问题(访问问题)

Service 是 Kubernetes 中的一种服务发现机制：

- Pod 有自己的 IP 地址
- Service 被赋予一个唯一的 dns name
- Service 通过 label selector 选定一组 Pod
- Service 实现负载均衡，可将请求均衡分发到选定这一组 Pod 中

### 虚拟ip和服务代理

Kubernetes 集群中的每个节点都运行了一个 `kube-proxy`，负责为 Service（ExternalName 类型的除外）提供虚拟 IP 访问。

#### Iptables 代理模式 （默认模式）

在 iptables proxy mode 下：

- kube-proxy 监听 kubernetes master 以获得添加和移除 Service / Endpoint 的事件
- kube-proxy 在其所在的节点（每个节点都有 kube-proxy）上为每一个 Service 安装 iptable 规则
- iptables 将发送到 Service 的 ClusterIP / Port 的请求重定向到 Service 的后端 Pod 上
  - 对于 Service 中的每一个 Endpoint，kube-proxy 安装一个 iptable 规则
  - 默认情况下，kube-proxy 随机选择一个 Service 的后端 Pod

### 使用自定义IP

创建 Service 时，如果指定 `.spec.clusterIP` 字段，可以使用自定义的 Cluster IP 地址。该 IP 地址必须是 APIServer 中配置字段 `service-cluster-ip-range` CIDR 范围内的合法 IPv4 或 IPv6 地址，否则不能创建成功。

可能用到自定义 IP 地址的场景：

- 想要重用某个已经存在的 DNS 条目
- 遗留系统是通过 IP 地址寻址，且很难改造

### 服务发现

Kubernetes 支持两种主要的服务发现模式：

- 环境变量
- DNS

#### 环境变量

kubelet 查找有效的 Service，并针对每一个 Service，向其所在节点上的 Pod 注入一组环境变量。支持的环境变量有：

- [Docker links 兼容 (opens new window)](https://docs.docker.com/network/links/)的环境变量
- {SVCNAME}_SERVICE_HOST 和 {SVCNAME}_SERVICE_PORT
  - Service name 被转换为大写
  - 小数点 `.` 被转换为下划线 `_`

例如，Service `redis-master` 暴露 TCP 端口 6379，其 Cluster IP 为 10.0.0.11，对应的环境变量如下所示：

```text
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

TIP

如果要在 Pod 中使用基于环境变量的服务发现方式，必须先创建 Service，再创建调用 Service 的 Pod。否则，Pod 中不会有该 Service 对应的环境变量。

如果使用基于 DNS 的服务发现，您无需担心这个创建顺序的问题

#### DNS

CoreDNS 监听 Kubernetes API 上创建和删除 Service 的事件，并为每一个 Service 创建一条 DNS 记录。集群中所有的 Pod 都可以使用 DNS Name 解析到 Service 的 IP 地址。

例如，名称空间 `my-ns` 中的 Service `my-service`，将对应一条 DNS 记录 `my-service.my-ns`。 名称空间 `my-ns` 中的Pod可以直接 `nslookup my-service` （`my-service.my-ns` 也可以）。其他名称空间的 Pod 必须使用 `my-service.my-ns`。`my-service` 和 `my-service.my-ns` 都将被解析到 Service 的 Cluster IP。

Kubernetes 同样支持 DNS SRV（Service）记录，用于查找一个命名的端口。假设 `my-service.my-ns` Service 有一个 TCP 端口名为 `http`，则，您可以 `nslookup _http._tcp.my-service.my-ns` 以发现该Service 的 IP 地址及端口 `http`

对于 `ExternalName` 类型的 Service，只能通过 DNS 的方式进行服务发现。参考 [Service/Pod 的 DNS](https://kuboard.cn/learning/k8s-intermediate/service/dns.html)

### Service/Pod的DNS

Kubernetes 集群中运行了一组 DNS Pod，配置了对应的 Service，并由 kubelete 将 DNS Service 的 IP 地址配置到节点上的容器中以便解析 DNS names。

集群中的每一个 Service（包括 DNS 服务本身）都将被分配一个 DNS name。默认情况下，客户端 Pod 的 DNS 搜索列表包括 Pod 所在的名称空间以及集群的默认域。例如：

假设名称空间 `bar` 中有一个 Service 名为 `foo`：

- 名称空间 `bar` 中的 Pod 可以通过 `nslookup foo` 查找到该 Service
- 名称空间 `quux` 中的 Pod 可以通过 `nslookup foo.bar` 查找到该 Service

# 存储

## 数据卷Volume

![image-20240121204759836](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240121204759836.png)

在 Kubernetes 里，Volume（数据卷）存在明确的生命周期（与包含该数据卷的容器组相同）。因此，Volume（数据卷）的生命周期比同一容器组中任意容器的生命周期要更长，不管容器重启了多少次，数据都能被保留下来。当然，如果容器组退出了，数据卷也就自然退出了。此时，根据容器组所使用的 Volume（数据卷）类型不同，数据可能随数据卷的退出而删除，也可能被真正持久化，并在下次容器组重启时仍然可以使用。

> 总之，数据卷是定义在容器组当中的

## 数据卷挂载

挂载是指将定义在 Pod 中的数据卷关联到容器，同一个 Pod 中的同一个数据卷可以被挂载到该 Pod 中的多个容器上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
        readOnly: false
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
        readOnly: false
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

> 一个 LAMP（Linux Apache Mysql PHP）应用的 Pod 使用了一个共享数据卷，HTML 内容映射到数据卷的 `html` 目录，数据库的内容映射到了 `mysql` 目录
>
> 有时候我们需要在同一个 Pod 的不同容器间共享数据卷。使用 `volumeMounts.subPath` 属性，可以使容器在挂载数据卷时指向数据卷内部的一个子路径，而不是直接指向数据卷的根路径。
