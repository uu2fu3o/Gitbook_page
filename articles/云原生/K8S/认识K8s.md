docker  --> 单机

k8s --> 多机管理

## K8S的需求

传统部署 :无法做到环境隔离

|

虚拟化部署:实现了环境隔离，但是资源占用大，启动速度慢

|

容器化部署(docker):实现了环境隔离，由于命名空间等技术的利用，资源分配合理，启动速度较快，但是存在寿命周期短，需要经常重启的情况

|

K8S：由于容器的寿命比较短暂，需要经常调试环境，而重新打包部署容器比较麻烦，又会存在一系列问题，包括但不限于网络，数据同步等，因此才有了K8S来对容器进行部署和管理

## K8S的特点

自我修复：对容器进行监测，出现问题就在原有无问题容器基础上进行复制启动，出现问题的容器进行抛弃(也可能是重启?)

弹性伸缩：容器数量的控制

自动部署和回滚：通过配置文件进行自动的容器构建，对容器的回滚更新

服务发现和负载均衡：默认方案

机密和配置管理：对敏感数据或其他进行配置管理

存储编排：虚拟磁盘与物理磁盘

批处理：批量任务实现

## K8S组件

### Master组件

Master组件是集群的控制平台

- master 组件负责集群中的全局决策（例如，调度）
- master 组件探测并响应集群事件（例如，当 Deployment 的实际 Pod 副本数未达到 `replicas` 字段的规定时，启动一个新的 Pod）

#### kube-apiserver

此 master 组件提供 Kubernetes API。这是Kubernetes控制平台的前端（front-end）

kubectl / kubernetes dashboard / kuboard 等Kubernetes管理工具就是通过 kubernetes API 实现对 Kubernetes 集群的管理。

#### etcd

etcd 是 Kubernetes 的核心组件之一，它是一个开源的分布式统一键值存储系统，用于分布式系统或计算机集群的共享配置、服务发现和的调度协调。etcd 通过 Raft 一致性算法处理日志复制以保证强一致性。Kubernetes 的所有集群数据都可以保存在 etcd 中。

#### kube-scheduler

此 master 组件监控所有新创建尚未分配到节点上的 Pod，并且自动选择为 Pod 选择一个合适的节点去运行

#### kube-controller-manager

此 master 组件运行了所有的控制器

逻辑上来说，每一个控制器是一个独立的进程，但是为了降低复杂度，这些控制器都被合并运行在一个进程里。

kube-controller-manager 中包含的控制器有：

- 节点控制器： 负责监听节点停机的事件并作出对应响应
- 副本控制器： 负责为集群中每一个 副本控制器对象（Replication Controller Object）维护期望的 Pod 副本数
- 端点（Endpoints）控制器：负责为端点对象（Endpoints Object，连接 Service 和 Pod）赋值
- Service Account & Token控制器： 负责为新的名称空间创建 default Service Account 以及 API Access Token

####  cloud-controller-manager

cloud-controller-manager 中运行了与具体云基础设施供应商互动的控制器。（默认不安装）

### Node组件

Node 组件运行在每一个节点上（包括 master 节点和 worker 节点），负责维护运行中的 Pod 并提供 Kubernetes 运行时环境。

#### kubelet

每个节点上都存在的代理程序，用来确保pod中的容器处于运行状态。也就是自动检测的功能，Kubelet不管理不是通过 Kubernetes 创建的容器。

#### kube-proxy

每个节点上都存在的网络代理程序，是实现 Kubernetes Service 概念的重要部分。

kube-proxy 在节点上维护网络规则。这些网络规则使得您可以在集群内、集群外正确地与 Pod 进行网络通信。

#### 容器引擎

负责运行容器

### Addons

Addons 使用 Kubernetes 资源（DaemonSet、Deployment等）实现集群的功能特性。由于他们提供集群级别的功能特性，addons使用到的Kubernetes资源都放置在 `kube-system` 名称空间下。

#### DNS

除了DNS Addon ，k8s集群还必须包含Cluster DNS.

Cluster DNS 是一个 DNS 服务器，是对您已有环境中其他 DNS 服务器的一个补充，存放了 Kubernetes Service 的 DNS 记录。

WEB UI(Dashboard)

集群的图形化管理界面





