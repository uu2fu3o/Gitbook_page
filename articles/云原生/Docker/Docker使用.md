>  记录搭建过程以及一些使用命令

## 安装

+ 一键部署

  ```shell
  curl -fsSL https://get.docker.com/ | sh
  or
  wget -qO- https://get.docker.com/ | sh
  ```

+ 安装docker-compose

  ```shell
  sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  sudo chmod +x /usr/local/bin/docker-compose
  docker-compose --version
  ```

注意：如果你像我一样使用的是kali,可能会出现无法使用上述命令或下载速度太慢的情况。kali只需要换源,下载即可

```
apt-get install docker.io docker-compose
```

## 使用

+ 拉取镜像

  ```
  docker pull ubuntu[:tag]
  ```

+ 根据指定镜像创建容器并启动

  ```
  docker run -d -t ubuntu
  #-d后台运行容器并返回其id
  ```

+ 列出镜像

  ```
  docker image ls
  ```

+ 查看所有容器

  ```
  docker ps -a
  ```

  否则显示已启动的容器

+ 进入容器

  ```
  docker exec -it id bash
  ```

  只推荐使用exec进行操作

+ 容器的导入和导出

  ```shell
  docker export 5af9292b4f0f > ubuntu.tar 
  #执行容器快照的导出操作
  cat ubuntu.tar | docker import - test/ubuntu:v1.0
  #从快照重新导入为镜像
  docker import http://example.com/exampleimage.tgz example/imagerepo
  #或者通过指定URL或某个目录来写入
  ```

+ 有关数据卷

  ```shell
  docker volume create my-vol
  #创建一个数据卷
  docker volume ls
  #查看所有数据卷
  docker volume inspect my-vol
  #查看指定数据卷的信息
  eg:
  docker run -d -P \
      --name web \
      # -v my-vol:/usr/share/nginx/html \
      --mount source=my-vol,target=/usr/share/nginx/html \
      nginx:latest
  #创建名为web的容器并加载数据卷到容器的/usr/share/nginx/html目录
  docker inspect web
  #查看web容器的具体信息，其中包括了数据卷信息
  docker volume rm my-vol
  #删除数据卷
  ```

  如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

+ 有关目录挂载

  使用 `--mount` 标记可以指定挂载一个本地主机的目录到容器中去。

  ```shell
   docker run -d -P \
      --name web \
      # -v /src/webapp:/usr/share/nginx/html \
      --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html[,readonly] \
      nginx:latest
  #加载主机的 /src/webapp 目录到容器的 /usr/share/nginx/html目录.本地目录的路径必须是绝对路径
  #Docker 挂载主机目录的默认权限是 读写，用户也可以通过增加 readonly 指定为只读
  ```

  `--mount` 标记也可以从主机挂载单个文件到容器中

  ```
  docker run --rm -it \
     # -v $HOME/.bash_history:/root/.bash_history \
     --mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history \
     ubuntu:18.04 \
     bash
  ```
  
  此后能记录在容器输入过的命令。

## 网络配置

其中有些命令选项只有在 Docker 服务启动的时候才能配置，而且不能马上生效。

- `-b BRIDGE` 或 `--bridge=BRIDGE` 指定容器挂载的网桥
- `--bip=CIDR` 定制 docker0 的掩码
- `-H SOCKET...` 或 `--host=SOCKET...` Docker 服务端接收命令的通道
- `--icc=true|false` 是否支持容器之间进行通信
- `--ip-forward=true|false` 请看下文容器之间的通信
- `--iptables=true|false` 是否允许 Docker 添加 iptables 规则
- `--mtu=BYTES` 容器网络中的 MTU

下面2个命令选项既可以在启动服务时指定，也可以在启动容器时指定。在 Docker 服务启动的时候指定则会成为默认值，后面执行 `docker run` 时可以覆盖设置的默认值。

- `--dns=IP_ADDRESS...` 使用指定的DNS服务器
- `--dns-search=DOMAIN...` 指定DNS搜索域

最后这些选项只有在 `docker run` 执行时使用，因为它是针对容器的特性内容。

- `-h HOSTNAME` 或 `--hostname=HOSTNAME` 配置容器主机名
- `--link=CONTAINER_NAME:ALIAS` 添加到另一个容器的连接
- `--net=bridge|none|container:NAME_or_ID|host` 配置容器的桥接模式
- `-p SPEC` 或 `--publish=SPEC` 映射容器端口到宿主主机
- `-P or --publish-all=true|false` 映射容器所有端口到宿主主机

## 卸载

+ 容器的删除操作

  ```
  docker rm [id] #删除指定容器
  docker rm $(docker ps -q -f status=exited) #删除所有已退出的容器
  docker rm $(docker ps -a -q) #删除所有已停止的容器
  docker container rm $(docker container ps -aq) #删除所有容器
  ```

+ 镜像的删除操作

  ```
  docker image rm id
  ```

  注意：一个镜像可以对应多个标签，因此当我们删除了所指定的标签后，可能还有别的标签指向了这个镜像，如果是这种情况，那么 `Delete` 行为就不会发生。所以并非所有的 `docker image rm` 都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已。

  镜像的删除也是分层的，当某一层完全没有复用才会被删除。

  如果有容器在使用该镜像，该镜像也是不会被删除的

## Docker Compose

基于service和project的集成管理，多个服务容器为一个project

> 介绍几个常用的命令，主要还是看懂compose模板文件

`-f, --file FILE` 指定使用的 Compose 模板文件，默认为 `docker-compose.yml`，可以多次指定。

`-p, --project-name NAME` 指定项目名称，默认将使用所在目录名称作为项目名。

`--verbose` 输出更多调试信息。

`-v, --version` 打印版本并退出。

+ build

  指定dockerfile所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径）。 `Compose` 将会利用它自动构建这个镜像，然后使用这个镜像。
  
  可以使用context指定dockerfile所在文件夹的路径
  
  使用 `dockerfile` 指令指定 `Dockerfile` 文件名。
  
  使用 `arg` 指令指定构建镜像时的变量
  
  使用 `cache_from` 指定构建镜像的缓存 #这个参数用于加速构建，指定镜像为缓存可以直接从该镜像继续构建
  
  ```
  version: '3'
  services:
  
    webapp:
      build:
        context: ./dir  //文件夹
        dockerfile: Dockerfile-alternate //dockerfile_name
        args:
          buildno: 1
        cache_from:
      	- alpine:latest
      	- corp/web_app:3.14
  ```

+ command

  ```
  //覆盖容器启动后的默认命令
  command: echo "1111111"
  ```

+ container_name

  ```
  //指定容器名称。默认为项目名称_服务名称_序号
  container_name: docker_web_container
  ```

  > 指定容器名称后，该服务将无法进行扩展（scale），因为 Docker 不允许多个容器具有相同的名称。

+ depends_on

  解决依赖问题，例如数据库需要先启动的需求

  ```
  services:
  	web:
  		build: .
  		depends_on:
  			- db
  			- redis
  	redis:
  		image: redis
  	
  	db:
  		image: mysql
  ```

> ​	注意：`web` 服务不会等待 `redis` `db` 「完全启动」之后才启动。

+ environment

  设置环境变量，支持数组或字典

  ```
  environment:
    RACK_ENV: development
    SESSION_SECRET:
  
  environment:
    - RACK_ENV=development
    - SESSION_SECRET
  ```

  > 注意：如果变量名或值中出现布尔词汇，需要双引号避免被解析
  >
  > ```
  > y|Y|yes|Yes|YES|n|N|no|No|NO|true|True|TRUE|false|False|FALSE|on|On|ON|off|Off|OFF
  > ```

+ expose

  ```
  //暴露端口，和dockerfile中的expose并没有什么区别，都值声明开放端口，具体的映射还需要自己去设置
  expose:
   - "3000"
   - "8000"
  ```

+ network_mode

  ```
  设置网络模式，和docker run 的--network一样
  network_mode: "bridge"
  network_mode: "host"
  network_mode: "none"
  network_mode: "service:[service name]"
  network_mode: "container:[container name/id]"
  ```

+ networks

  ```shell
  //配置容器连接的网络
  version: '3'
  
  services:
    service1:
      image: image1
      networks:
        - my_net
  
    service2:
      image: image2
      networks:
        - my_net
  
  networks:
    my_net:
      external: true //用于声明my_net是在该compose文件之外创建的网络
  ```

+ ports

  与expose不同的地方在于可以设定需要映射的外部端口

  使用宿主端口：容器端口 `(HOST:CONTAINER)` 格式，或者仅仅指定容器的端口（宿主将会随机选择端口）都可以。

  ```
  ports:
   - "3000" //随机映射主机的端口
   - "8000:8000"
   - "49100:22"
   - "127.0.0.1:8001:8001"
  ```

  > 注意：端口号小于60需要使用双引号进行包裹，否则会被自动解析为60进制

+ secerts

  存储敏感数据，某些服务的密码

+ volumes

  ```c
  //数据卷挂载的路径可以设置为宿主机路径(HOST:CONTAINER)或者数据卷名称(VOLUME:CONTAINER)，并且可以设置访问模式 （HOST:CONTAINER:ro）。该指令中路径支持相对路径。
  volumes:
   - /var/lib/mysql
   - cache/:/tmp/cache
   - ~/configs:/etc/configs/:ro
  //如果路径为数据卷名称，必须在文件中配置数据卷。
  version: "3"
  
  services:
    my_src:
      image: mysql:8.0
      volumes:
        - mysql_data:/var/lib/mysql
  
  volumes:
    mysql_data:  
  ```

## 参考链接：

https://yeasy.gitbook.io/docker_practice/advanced_network/quick_guide

https://wiki.teamssix.com/cloudnative/