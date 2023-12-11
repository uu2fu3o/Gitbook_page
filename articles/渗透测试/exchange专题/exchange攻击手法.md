## 前言

大概复现一些常见的exchange攻击手法，从定位，无用户攻击，低权限和高权限来分别讨论

## 定位

### 在外网搜索Exchange

通过fofa进行搜索,给出一个大致的格式

```
app="Microsoft-Exchange-2016"
app="Microsoft-Exchange-2016-CU5"
```

如果要定位到特定的版本，修改2016和CU5的位置即可

### 如何在域内快速定位到Exchange

**端口探测**

Exchange的端口一些特征，通过nmap扫描可以看出exchange 接口会暴露在80端口，同时25/587/2525等端口上会有SMTP服务

![image-20231205214217694](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231205214217694.png)

**SPN**

查询域内的SPN

![image-20231205220031699](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231205220031699.png)

**特殊域名**

```
https://autodiscover.domain.com/autodiscover/autodiscover.xml
https://owa.domian/owa/
https://mail.domain.com/
https://webmail.domain.com/
```

**使用工具**

https://github.com/vysec/checkO365

**确定版本**

确定好exchange页面的位置之后，访问owa和ecp从html来确定当前站点exchange的版本

从开发者页面搜索favicon.ico

![image-20231205224000491](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231205224000491.png)

有具体的版本号，根据该版本号进行查询

[microsoft](https://learn.microsoft.com/zh-cn/Exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)

**IP泄露**

抓包以下接口包，将HTTP版本改为1.0，并删除HOST头，就会暴露exchange ip，有时会暴露内网IP

```
/Microsoft-Server-ActiveSync/default.eas
/Microsoft-Server-ActiveSync
/Autodiscover/Autodiscover.xml
/Autodiscover
/Exchange
/Rpc
/EWS/Exchange.asmx
/EWS/Services.wsdl
/EWS
/ecp
/OAB
/OWA
/aspnet_client
/PowerShell
```

对于在内网的 exchange不太实用，对于外网会比较实用一些，这里出去打野测试了一下

![123](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/123.png)

不过在打野途中也遇到了无法返回IP的情况，目前猜测是因为server2019的原因，不过这里也没有返回具体的版本

**泄露exchange服务器信息**

```
nmap host  -p 443 --script http-ntlm-info --script-args http-ntlm-info.root=/rpc/rpcproxy.dll -Pn
```

![image-20231206160623999](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231206160623999.png)

这个扫描在域外也可以进行，能够返回exchange服务器的信息以及域的部分信息

## 无用户攻击

即外围打点的操作，核心思想是获得邮箱用户进行深度攻击

**爆破**

Exchange服务有几个比较重要的接口比如owa,ecp，以及一些可以用来爆破的接口

常见可爆破接口

```
/Autodiscover/Autodiscover.xml  # 自 Exchange Server 2007 开始推出的一项自动服务,用于自动配置用户在Outlook中邮箱的相关设置,简化用户登录使用邮箱的流程。
/Microsoft-Server-ActiveSync/default.eas
/Microsoft-Server-ActiveSync    # 用于移动应用程序访问电子邮件
/Autodiscover
/Rpc/                           # 早期的 Outlook 还使用称为 Outlook Anywhere 的 RPC 交互
/EWS/Exchange.asmx
/EWS/Services.wsdl
/EWS/                           # Exchange Web Service,实现客户端与服务端之间基于HTTP的SOAP交互
/OAB/                           # 用于为Outlook客户端提供地址簿的副本,减轻 Exchange 的负担
/owa                            # Exchange owa 接口,用于通过web应用程序访问邮件、日历、任务和联系人等
/ecp                            # Exchange 管理中心,管理员用于管理组织中的Exchange 的Web控制台
/Mapi                           # Outlook连接 Exchange 的默认方式,在2013和2013之后开始使用,2010 sp2同样支持
/powershell                     # 用于服务器管理的 Exchange 管理控制台
```

![image-20231206164253327](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231206164253327.png)

[RULER](https://github.com/sensepost/ruler/releases/tag/2.4.1)

**密码喷洒**

[MailSniper](https://github.com/dafthack/MailSniper)

```powershell
对OWA进行爆破
Import-Module .\MailSniper.ps1Invoke-PasswordSprayOWA -ExchHostname OWAHOST -UserList .\user.txt -Password password -Threads 1 -Domain domainname -OutFile out.txt -Verbose 
对EWS进行爆破
Invoke-PasswordSprayEWS -ExchHostname EWSHOST -UserList .\user.txt -Password password -Threads 1 -Domain domainname -OutFile out.txt -Verbose
```

**CVE-2021-26855+CVE-2021-27065**

CVE-2021-26855是一个SSRF漏洞，可以在无用户的情况下绕过Exchange的身份验证

CVE-2021-27065是一个任意文件写入漏洞，这个漏洞要求一个邮件服务器的账户，而ssrf正好提供了这一点

两个漏洞结合，能够达到getshell的目的

+ 漏洞影响版本

  Exchange Server 2019 < 15.02.0792.010

  Exchange Server 2019 < 15.02.0721.013

  Exchange Server 2016 < 15.01.2106.013

  Exchange Server 2013 < 15.00.1497.012

可以适用一键式的脚本直接getshell

![image-20231206220234713](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231206220234713.png)

[tools](https://github.com/ZephrFish/Exch-CVE-2021-26855)

## Exchange后渗透

获取用户一个用户之后，可以对邮件列表进行检索获取敏感信息，方便下一步渗透

MailSniper

```shell
#检索hacker@loser.com收件箱文件夹里内容含有 pass,在启用remote参数后会弹出一个输入框输入邮箱票据
Invoke-SelfSearch -Mailbox hacker@loser.com -Terms *pass* -Folder 收件箱 -ExchangeVersion Exchange2016 -remote -OutputCsv out.csv
```

![image-20231211105904411](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231211105904411.png)

**获取全局地址列表**

我们获得一个合法用户的凭据以后，就可以通过获取全局地址表来获取所有邮箱地址

```
Get-GlobalAddressList -ExchHostname 192.168.30.88 -UserName hacker -ExchangeVersion Exchange2016 -Password UU2FU3O@ADMIN
```

![image-20231211160117124](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231211160117124.png)

**查找存在缺陷的用户邮箱权限委派**

一个用户的邮箱是可以被委托给其他人的

![image-20231211195057394](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231211195057394.png)

这里将loser用户的邮箱委派给其他所有人，通常情况下A对B的邮箱有读写权限，如果我们有A的凭据就可以检索B的邮箱

```
Invoke-OpenInboxFinder -ExchangeVersion Exchange2016 -ExchHostname 192.168.30.88 -Mailbox loser@loser.com -remote
```

![image-20231211195635879](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231211195635879.png)

**通过Exchange用户组进行域提权**

概括来讲就是利用WriteDacl，exchange windows permissions组的用户拥有writeDACL权限，Exchange Trusted Subsystem 是 Exchange Windows Permission 的成员，能继承writedacl权限，有这个权限后就能使用dcsync导出所有用户hash。

同样拥有此权限的还有Organization Management组，这个组可以修改其他exchange组的用户信息，所以当然也可以修改Exchange Trusted Subsystem组的成员信息，比如向里面加一个以获得的用户。

**攻击outlook客户端**

OUTLOOK 客户端有一个 规则与通知 的功能，通过该功能可以使outlook客户端在指定情况下执行指定的指令。若我们获得某用户的凭证，可以通过此功能设置“用户收到含指定字符的邮件时 执行指定的指令比如clac.exe”，当用户登录outlook客户端并访问到此邮件时，它的电脑便会执行calc.exe。
但是，当触发动作为启动应用程序时，只能直接调用可执行程序，如启动一个exe程序，但无法为应用程序传递参数，想要直接上线，我们可以将EXE放到某共享目录下，或者直接上传到用户的机器。

## 参考链接

https://tttang.com/archive/1487/#toc_exchange_1

https://rain1-lce.gitbook.io/web/windowsad-ji-chu/exchange-pian

https://3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-%E4%BD%BF%E7%94%A8Exchange%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%AD%E7%89%B9%E5%AE%9A%E7%9A%84ACL%E5%AE%9E%E7%8E%B0%E5%9F%9F%E6%8F%90%E6%9D%83