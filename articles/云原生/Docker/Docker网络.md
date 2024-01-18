> 主要是端口映射和网络配置方面的东西

## 使用网络

### 外部访问容器

```dockerfile
-P #Docker会随机映射一个端口到内部容器开放的网络端口
docker logs container_name #查看访问记录
-p #可以指定要映射的端口，并且，在一个指定端口上只可以绑定一个容器.
#ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort
```

+ 映射所有接口地址

  ```
  hostPort:containerPort
  ```

  默认绑定本地所有接口上的所有地址

+ 映射指定接口地址

  ```
  ip:hostPort:containerPort
  ```

+ 映射到指定地址的任意端口

  ```
  ip::containerPort
  eg:docker run -d -p 127.0.0.1::80 nginx:latest
  ```

> 查看端口
>
> docker port cd0862f6acfe
>
> 80/tcp -> 0.0.0.0:32768
> 80/tcp -> :::32768

注意：容器有自己的内部网络和 ip 地址（使用 `docker inspect` 查看，Docker 还可以有一个可变的网络配置。） -p可以一次绑定多个端口

> 例如，我不加-p等参数启动一个nginx,通过docker port id查看是没有端口映射的，但是docker ps会看到80/tcp，这说明容器开启了80端口，但是没有映射到本机

### 通过网络使容器互联

+ 新建网络

  ```
  docker network create -d bridge new-net
  #-d指定Docker的网络类型，其中overlay 网络类型用于 Swarm mode
  ```

+ 连接容器

  ```
  docker run -it --rm --name netry1 --network new-net busybox sh
  docker run -it --rm --name netry2 --network new-net busybox sh
  ```

现在两个容器处于互联状态，能够相互ping通

![image-20240113204505934](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240113204505934.png)

> 可能在过程中你会产生向我一样的疑惑，为什么查找不到对应的DNS解析，却能够直接ping容器名
>
> 自定义网络的DNS是内置服务器，由Docker daemon进行管理，并不通过常规的文件存储或记录。这个DNS会自动更新。
>
> 如果你查看DNS文件，会发现
>
> ```
> / # cat /etc/resolv.conf
> search localdomain
> nameserver 127.0.0.11
> options ndots:0
> ```

### 配置DNS

在容器中执行mount,我们能看到挂载信息

```
/dev/sda1 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro)
/dev/sda1 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro)
/dev/sda1 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro)
```

这种机制可以让宿主主机 DNS 信息发生更新后，所有 Docker 容器的 DNS 配置通过 `/etc/resolv.conf` 文件立刻得到更新。

+ 配置全部容器的DNS

  在`/etc/docker/daemon.json` 文件中增加，格式如下

  ```
  {
    "dns" : [
      "114.114.114.114",
      "8.8.8.8"
    ]
  }
  ```

+ 手动指定容器配置

  `-h HOSTNAME` 或者 `--hostname=HOSTNAME` 设定容器的主机名，它会被写到容器内的 `/etc/hostname` 和 `/etc/hosts`。但它在容器外部看不到，既不会在 `docker container ls` 中显示，也不会在其他的容器的 `/etc/hosts` 看到。

  `--dns=IP_ADDRESS` 添加 DNS 服务器到容器的 `/etc/resolv.conf` 中，让容器用这个服务器来解析所有不在 `/etc/hosts` 中的主机名。

  `--dns-search=DOMAIN` 设定容器的搜索域，当设定搜索域为 `.example.com` 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 `host.example.com`。

  > 注意：如果在容器启动时没有指定最后两个参数，Docker 会默认用主机上的 `/etc/resolv.conf` 来配置容器。

## 网络配置

当 Docker 启动时，会自动在主机上创建一个 `docker0` 虚拟网桥，实际上是 Linux 的一个 bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进行转发。

Docker 随机分配一个本地未占用的私有网段中的一个地址给 `docker0` 接口。

当创建一个 Docker 容器的时候，同时会创建了一对 `veth pair` 接口（当数据包发送到一个接口时，另外一个接口也可以收到相同的数据包）。这对接口一端在容器内，即 `eth0`；另一端在本地并被挂载到 `docker0` 网桥，名称以 `veth` 开头（例如 `vethAQI2QT`）。通过这种方式，主机可以跟容器通信，容器之间也可以相互通信。Docker 就创建了在主机和所有容器之间一个虚拟共享网络。

![image-20240113212928943](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2024/image-20240113212928943.png)

图片来自[Docker-从入门到实践](https://yeasy.gitbook.io/docker_practice/advanced_network)

### 容器访问控制

主要通过linux上的iptables防火墙来进行管理和实现。

+ 访问外部网络

  依靠于本地系统的转发支持

  ```
  ┌──(root㉿kali)-[~]
  └─# sysctl net.ipv4.ip_forward 
  net.ipv4.ip_forward = 1
  ```

  如果为1，则说明系统支持转发

  为0需要手动开启

  ```
  sysctl -w net.ipv4.ip_forward=1
  ```

> 如果在启动 Docker 服务的时候设定 `--ip-forward=true`, Docker 就会自动设定系统的 `ip_forward` 参数为 1

+ 容器之间相互访问

  需要以下支持：

  1.容器的网络拓扑是否已经互联。默认情况下，所有容器都会被连接到 `docker0` 网桥上。

  2.本地系统的防火墙软件 -- `iptables` 是否允许通过。

> 和之前的通过网络使容器互联性质相同，只不过这里使用的是默认的docker0网桥
>
> 是否允许通过是指是否允许docker容器之间的互相通信，关于这个可以通过iptables -L来查看

+ 容器之间的访问端口

  所有端口：

  当启动 Docker 服务（即 dockerd）的时候，默认会添加一条转发策略到本地主机 iptables 的 FORWARD 链上。策略为通过（`ACCEPT`）还是禁止（`DROP`）取决于配置`--icc=true`（缺省值）还是 `--icc=false`。当然，如果手动指定 `--iptables=false` 则不会添加 `iptables` 规则。

  可见，默认情况下，不同容器之间是允许网络互通的。如果为了安全考虑，可以在 `/etc/docker/daemon.json` 文件中配置 `{"icc": false}` 来禁止它。

  当仅需要访问指定端口：

  指定-icc=false关闭网络访问，通过 `--link=CONTAINER_NAME:ALIAS` 选项来访问容器的开放端口

  > 在docker1.9之后就不再推荐使用--link,容器之间的访问最好还是通过新建网络的形式

### 端口映射

+ 容器访问外部

  之前提到过，容器访问外部依赖于系统的转发，查看iptables的nat规则

  ```
  Chain POSTROUTING (policy ACCEPT)
  target     prot opt source               destination         
  MASQUERADE  0    --  172.18.0.0/16        0.0.0.0/0           
  MASQUERADE  0    --  172.17.0.0/16        0.0.0.0/0           
  ```

  这条规则使用`MASQUERADE`目标，在所有来自`172.17.0.0/16`网络并且目的地不在`172.17.0.0/16`网络内的流量上执行源地址转换（SNAT）

  这使得docker容器的ip地址看起来像主机的IP地址，也就是伪装。MASQUERADE 跟传统 SNAT 的好处是它能动态从网卡获取地址。

+ 外部访问容器实现

  外部访问容器依赖于在docker run时映射的端口,查看iptables中的nat规则，会添加如下

  ```
  DNAT       6    --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
  ```
  
  > 0.0.0.0，意味着将接受主机来自所有接口的流量,可以自定义更严格的规则
  >
  > 如果希望永久绑定到某个固定的 IP 地址，可以在 Docker 配置文件 `/etc/docker/daemon.json` 中添加如下内容
  >
  > ```
  > {
  >   "ip": "0.0.0.0"
  > }
  > ```

### 网桥配置

我们知道docker服务默认存在一个网桥dokcer0,它在内核层连通了其他的物理或虚拟网卡，这就将所有容器和本地主机都放到同一个物理网络.

我们可以在启动docker服务时自定义这个网桥

```
--bip=CIDR IP 地址加掩码格式，例如 192.168.1.5/24
--mtu=BYTES 覆盖默认的 Docker mtu 配置 // MTU（接口允许接收的最大传输单元），通常是 1500 Bytes
```

可以使用 `brctl show` 来查看网桥和端口连接信息

```
docker0         8000.0242cb1a498f       no              veth8871dc2
```

每次创建新的容器，会从可用地址当中选取一个分配给容器的eth0端口。使用本地主机上 `docker0` 接口的 IP 作为所有容器的默认网关。

+ 自定义网桥

  ```bash
  brctl addbr bridge1 #创建一个网桥
  ip addr add 192.168.5.1/24 dev bridge1 #添加地址块
  ip link set dev bridge1 up #启动该网桥
  ip addr show bridge0 #确认
  //
  36: bridge1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
      link/ether 86:84:b8:5d:dd:ba brd ff:ff:ff:ff:ff:ff
      inet 192.168.5.1/24 scope global bridge1
         valid_lft forever preferred_lft forever
      inet6 fe80::8484:b8ff:fe5d:ddba/64 scope link proto kernel_ll 
         valid_lft forever preferred_lft forever
  //
  如果你需要删除网桥，首先需要停止该网桥上的服务以及网桥
  ip link set dev bridge1 down
  brctl delbr bridge1
  ```

  > 如果你希望docker一直使用同一个网桥，可以在配置文件`/etc/docker/daemon.json` 中添加如下内容
  >
  > ```
  > {
  >   "bridge": "bridge0",
  > }
  > ```

注意：在docker内部修改网络配置文件，在容器终止或重启后并不会保存下来

### 配置网络代理

目的时为了加速镜像拉取构建和使用，一般不怎么用到这个:)

+ 为dockerd创建配置文件夹

  ```
  mkdir -p /etc/systemd/system/docker.service.d
  ```

+ 为 dockerd 创建 HTTP/HTTPS 网络代理的配置文件，文件路径是 /etc/systemd/system/docker.service.d/http-proxy.conf 。并在该文件中添加相关环境变量。

  ```
  [Service]
  Environment="HTTP_PROXY=http://proxy.example.com:8080/"
  Environment="HTTPS_PROXY=http://proxy.example.com:8080/"
  Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
  ```

> 需要重启docker服务

+ 为容器设置网络代理

  ```
  更改 docker 客户端配置：创建或更改 ~/.docker/config.json，并在该文件中添加相关配置
  {
   "proxies":
   {
     "default":
     {
       "httpProxy": "http://proxy.example.com:8080/",
       "httpsProxy": "http://proxy.example.com:8080/",
       "noProxy": "localhost,127.0.0.1,.example.com"
     }
   }
  }
  ```

  或者在docker run时指定环境变量

  ```
  --env:
  HTTP_PROXY="http://proxy.example.com:8080/"
  HTTPS_PROXY="http://proxy.example.com:8080/"
  NO_PROXY="localhost,127.0.0.1,.example.com"
  ```

+ 为docker build过程设置代理

  ```
  --build-arg指定相关环境变量
  
  docker build \
      --build-arg "HTTP_PROXY=http://proxy.example.com:8080/" \
      --build-arg "HTTPS_PROXY=http://proxy.example.com:8080/" \
      --build-arg "NO_PROXY=localhost,127.0.0.1,.example.com" .
      
  在dockerfile中指定代理
  ENV HTTP_PROXY="http://proxy.example.com:8080/"
  #其余参数同环境变量
  ```

> 关于docker的点到点链接，网上有很多例子，最推荐的方法还是通过自定义网络进行连接，需要注意的是自定义网络的方法不同于默认的网桥，默认的网桥虽然实现了容器之间的通信，这种点到点的通信仅限于容器之间，不通过网桥而直接通信，对于容器与容器直接更加效率。
