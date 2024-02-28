## 部署应用程序

K8S通过发布Deployment来进行程序的部署

> **Deployment** 译名为 **部署**。在k8s中，通过发布 Deployment，可以创建应用程序 (docker image) 的实例 (docker container)，这个实例会被包含在称为 **Pod** 的概念中，**Pod** 是 k8s 中最小可管理单元。

创建应用程序实例后，Kubernetes Deployment Controller 会持续监控这些实例。如果运行实例的 worker 节点关机或被删除，则 Kubernetes Deployment Controller 将在群集中资源最优的另一个 worker 节点上重新创建一个新的实例。**这提供了一种自我修复机制来解决机器故障或维护问题。**

### 通过kubectl部署nginx

Deployment通过yaml文件指定，因此我们只需要写一个容器文件即可

> 感觉和docker compose的文件比较像

```yaml
apiVersion: apps/v1 #取决于k8s支持的集群版本
kind: Deployment #该配置的类型，我们使用的是 Deployment
metadata: #下方用来定义Deplyment一些基本属性
  name: nginx-deployment #Deployment名车给
  labels: #标签，为key: value
    app: nginx 
spec: #关于Deployment的描述
  replicas: 1 #运行的所需 Pod 的数量
  selector: #标签选择器，与上面的标签共同作用
    matchLabels:
      app: nginx
  template: #这是选择或创建的Pod的模板
    metadata:
      labels:
        app: nginx
    spec: #期望Pod实现的功能（即在pod中部署）
      containers: #生成的容器，我们采用的服务为docker，因此为docker创建的容器
      - name: nginx #容器名称
        image: nginx:1.7.9 #指定使用的镜像
        ports:
        - containerPort: 80 #设置容器使用的端口
```

> 还有一些其他参数
>
> - replicas： 控制着运行的所需 Pod 的数量
>
> - template： 为 Deployment 创建的 Pod 提供模板
>
> - selector： 用于找出由 Deployment 管辨理的 Pod
>
> - strategy： 提供更新 Deployment 的策略
>
>   具体可以参考[Kubernetes Deployment 详细介绍](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/)

执行命令应用该Deployment

```
kubectl apply -f nginx-deployment.yaml
```

现在你应该能看到一个名为nginx-deployment的Deployment和一个名为 nginx-deployment-xxxxxxx-xxxxx 的 Pod

![image-20240118204836958](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240118204836958.png)

用图形化界面进行部署会更加简单

## Pods&Nodes

### Pods

创建Deployment之后，k8s创建了一个pod（容器组）用来存储应用实例

**Pod 容器组** 是一个k8s中一个抽象的概念，用于存放一组 container（可包含一个或多个 container 容器，即图上正方体)，以及这些 container （容器）的一些共享资源。这些资源包括：

- 共享存储，称为卷(Volumes)
- 网络，每个 Pod（容器组）在集群中有个唯一的 IP，pod（容器组）中的 container（容器）共享该IP地址
- container（容器）的基本信息，例如容器的镜像版本，对外暴露的端口等

> 同一pod中，不一致只有一个服务，例如服务A需要服务B提供数据，由于同一pod中IP地址是共享的，因此服务A想要访问服务B，需要通过localhost+port的形式进行访问(同一pod中，containers的ports不能产生冲突)

通过Deployment创建的pod，在该pod对应的worker所在的节点(Node)发生故障时，会在集群的其他可用Node上运行相同的Pod（从同样的镜像创建 Container，使用同样的配置，IP 地址不同，Pod 名字不同）

> 学到这里先提个问题：ip不同，pod名字不同，那么如果是需要固定的服务该怎么进行访问？到底会不会存在这样的问题？和docker使用自定义网络的形式相同吗？

+ 重要提示：
  - Pod 是一组容器（可包含一个或多个应用程序容器），以及共享存储（卷 Volumes）、IP 地址和有关如何运行容器的信息。
  - 如果多个容器紧密耦合并且需要共享磁盘等资源，则他们应该被部署在同一个Pod（容器组）中。

### Nodes

Pod是依赖于节点运行的，一个节点上可以有多个pod。

Node（节点）是 kubernetes 集群中的计算机，可以是虚拟机或物理机。

kubernetes master 会根据每个 Node（节点）上可用资源的情况，自动调度 Pod（容器组）到最佳的 Node（节点）上。

每个 Kubernetes Node（节点）至少运行：

- Kubelet，负责 master 节点和 worker 节点之间通信的进程；管理 Pod（容器组）和 Pod（容器组）内运行的 Container（容器）。
- 容器运行环境（如Docker）负责下载镜像、创建和运行容器等。

## 公布应用程序

### Kubernetes Service（服务）概述

我们知道当pod运行的Node故障时，Deployment可以通过创建新的Pod来动态地将群集调整回原来的状态，以使应用程序保持运行。

但是在这个过程中pod的ip和name发生了变化。由于 Kubernetes 集群中每个 Pod（容器组）都有一个唯一的 IP 地址（即使是同一个 Node 上的不同 Pod），我们需要一种机制，为前端系统屏蔽后端系统的 Pod（容器组）在销毁、创建过程中所带来的 IP 地址的变化。

> 这部分正好解决前面提出的问题

Service API 是 Kubernetes 的组成部分，它是一种抽象，帮助你将 Pod 集合在网络上公开出去。 每个 Service 对象定义端点的一个逻辑集合（通常这些端点就是 Pod）以及如何访问到这些 Pod 的策略。

Service（服务）使 Pod（容器组）之间的相互依赖解耦（原本从一个 Pod 中访问另外一个 Pod，需要知道对方的 IP 地址）。一个 Service（服务）选定哪些 **Pod（容器组）** 通常由 **LabelSelector(标签选择器)** 来决定。

通过设置配置文件中的 spec.type 字段的值，可以以不同方式向外部暴露应用程序：

- **ClusterIP**（默认）

  在群集中的内部IP上公布服务，这种方式的 Service（服务）只在集群内部可以访问到

- **NodePort**

  使用 NAT 在集群中每个的同一端口上公布服务。这种方式下，可以通过访问集群中任意节点+端口号的方式访问服务 `<NodeIP>:<NodePort>`。此时 ClusterIP 的访问方式仍然可用。

- **LoadBalancer**

  在云环境中（需要云供应商可以支持）创建一个集群外部的负载均衡器，并为使用该负载均衡器的 IP 地址作为服务的访问地址。此时 ClusterIP 和 NodePort 的访问方式仍然可用。

### 服务和标签

![image-20240119134252039](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240119134252039.png)

这里用kuboard的图片，方便理解学习

Service 将外部请求路由到一组 Pod 中，它提供了一个抽象层，使得 Kubernetes 可以在不影响服务调用者的情况下，动态调度容器组（在容器组失效后重新创建容器组，增加或者减少同一个 Deployment 对应容器组的数量等）

> 意思是在node在down以后，对于新建的pod，仍然位于服务中，解决了之前的问题
>
> 那么我们该如何确定服务位置，以及服务的所包含的pod呢？

Service使用 [Labels、LabelSelector(标签和选择器) (opens new window)](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels)匹配一组 Pod。Labels（标签）是附加到 Kubernetes 对象的键/值对，其用途有多种：

- 将 Kubernetes 对象（Node、Deployment、Pod、Service等）指派用于开发环境、测试环境或生产环境
- 嵌入版本标签，使用标签区别不同应用软件版本
- 使用标签对 Kubernetes 对象进行分类

对于上图

- Deployment B 含有 LabelSelector 为 app=B 通过此方式声明含有 app=B 标签的 Pod 与之关联
- 通过 Deployment B 创建的 Pod 包含标签为 app=B
- Service B 通过标签选择器 app=B 选择可以路由的 Pod

> 包含的关系，service通过标签选择器来选择pod

### 为nginx-deployment创建服务

我们之前所用的文件如下

```yaml
metadata:	#译名为元数据，即Deployment的一些基本属性和信息
  name: nginx-deployment	#Deployment的名称
  labels:	#标签，可以灵活定位一个或多个资源，其中key和value均可自定义，可以定义多组
    app: nginx	#为该Deployment设置key为app，value为nginx的标签
```

> 通过该deployment创建的pod,都含有app=nginx这个标签

现在我们创建一个nginx-service.yaml文件来配置该deployment的服务

```yaml
apiVersion: v1
kind: Service 	#类型为Service
metadata:
  name: nginx-service #服务名称
  labels:
    app: nginx  #Service 自己的标签
spec:
  selector: #标签选择器配置
    app: nginx  #包含标签app: nginx的pod
  ports:
  - name: nginx-port #端口名字
    protocol: TCP #协议
    port: 80 #集群内的其他容器组可通过 80 端口访问 Service 
    nodePort: 32600 #通过任意节点的 32600 端口访问 Service
    targetPort: 80 #将请求转发到匹配 Pod 的 80 端口
  type: NodePort #Serive的类型，ClusterIP/NodePort/LoaderBalancer

```

执行命令

```
kubectl apply -f nginx-service.yaml
```

现在我们可以通过任意一个node来访问nginx服务

![image-20240119141728198](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240119141728198.png)

> 或许单纯这样看配置文件并不好理解端口的映射关系，以及为什么能够从任意一个Node去访问服务
>
> 1. `port`：这是服务的公开端口，集群中的其他Pod通过此端口访问该服务。示例中，其他Pod可以通过80端口访问该服务。
> 2. `nodePort`：这是你的服务在节点上的公开端口。这意味着服务可以通过这个端口在集群外部访问。示例中，可以通过任何节点的32600端口访问该服务。
> 3. `targetPort`：这是服务将流量转发到的Pod内的端口。示例中，请求将被转发到匹配Pod的80端口。
> 4. `NodePort`：这代表服务在每个节点的一个静态端口（nodePort）上暴露出来。你可以通过 `<NodeIP>:<NodePort>` 从集群的外部访问这个服务。

当然一个服务不可能只有一个端口，也不一定只有一个pod,如下，对服务设置多端口

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

**为 Service 使用多个端口时，必须为所有端口提供名称，以使它们无歧义**

> Pod 中的端口定义是有名字的，你可以在 Service 的 `targetPort` 属性中引用这些名字
>
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx
>   labels:
>     app.kubernetes.io/name: proxy
> spec:
>   containers:
>   - name: nginx
>     image: nginx:stable
>     ports:
>       - containerPort: 80
>         name: http-web-svc
> 
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx-service
> spec:
>   selector:
>     app.kubernetes.io/name: proxy
>   ports:
>   - name: name-of-service-port
>     protocol: TCP
>     port: 80
>     targetPort: http-web-svc
> ```

## 伸缩应用程序

### Scaling（伸缩）应用程序

对于出现较大流量时采用的策略，**伸缩** 的实现可以通过更改 nginx-deployment.yaml 文件中部署的 replicas（副本数）来完成

```
spec:
  replicas: 2    #使用该Deployment创建两个应用程序实例
```

> 这样做的好处是，外部在访问service服务时，会将流量转发到多个负载上减轻压力

## 执行滚动更新

**Rolling Update滚动更新** 通过使用新版本的 Pod 逐步替代旧版本的 Pod 来实现 Deployment 的更新，从而实现零停机。新的 Pod 将在具有可用资源的 Node（节点）上进行调度。

> Kubernetes 更新多副本的 Deployment 的版本时，会逐步的创建新版本的 Pod，逐步的停止旧版本的 Pod，以便使应用一直处于可用状态。这个过程中，Service 能够监视 Pod 的状态，将流量始终转发到可用的 Pod 上。
>
> (显然这样更新，deployment创建实例不能为1)

默认情况下，**Rolling Update 滚动更新** 过程中，Kubernetes 逐个使用新版本 Pod 替换旧版本 Pod（最大不可用 Pod 数为 1、最大新建 Pod 数也为 1）。这两个参数可以配置为数字或百分比。在Kubernetes 中，更新是版本化的，任何部署更新都可以恢复为以前的（稳定）版本。

> 简单来说，创一替一，注意新建的pod拥有一个新的ip

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
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
        image: nginx:1.8.0 #替换为新版本
```

执行

```
kubectl apply -f nginx-deployment.yaml
```

随后开始滚动更新

```
watch kubectl get pods -l app=nginx
```

使用该命令可以看到更新的整个过程

## 参考链接

[kubernetes](https://kubernetes.io/)

[kuboard](https://kuboard.cn/learning/k8s-bg/architecture/com-m-n.html#apiserver-to-kubelet)