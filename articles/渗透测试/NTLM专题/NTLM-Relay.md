上一章简单介绍了NTLM的身份验证方式，现在我们知道NTLM是客户端和服务端之间的认证，那么NTLM中继我们就作为中间人，客户端认为我们是服务端，服务端认为我们是客户端。

如下的中继流程

![ntlm_relay_basic](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/ntlm_relay_basic.png)

在服务端的眼中，攻击者与客户端无异，因此，当我们完成上述流程，服务器认为攻击者身份验证成功，然后攻击者可以代表 原客户端 在服务器上执行操作。

每种协议更详细的介绍，可以参考这篇文章[ntlm-realy](https://en.hackndo.com/ntlm-relay/)

## Relay2SMB

能直接relay到smb服务器，是最直接最有效的方法。可以直接控制该服务器(包括但不限于在远程服务器上执行命令，上传exe到远程命令上执行，dump 服务器的用户hash等等等等)。

realy2smb主要有两种场景 ，分为工作组环境和域环境，这里重点介绍域环境下realy2smb

在开始攻击之前，先简单介绍一下smb协议(权当复习)

### SMB

SMB（ServerMessage Block）通信协议是微软（Microsoft）和英特尔(Intel)在1987年制定的协议，主要是作为Microsoft网络的通讯协议。SMB 是在会话层（session layer）和表示层（presentation layer）以及小部分应用层（application layer）的协议。SMB使用了NetBIOS的应用程序接口 （ApplicationProgram Interface，简称API），一般端口使用为139，445。

> SMB是应用层（和表示层）协议，使用C/S架构，其工作的端口与其使用的协议有关

当远程连接计算机访问共享资源时有两种方式：

- 共享计算机地址\共IP享资源路径
- 共享计算机名\共享资源路径

**域环境**

域环境底下域用户的账号密码Hash保存在域控的 ntds.dit里面。如下没有限制域用户登录到某台机子，那就可以将该域用户Relay到别人的机子，或者是拿到域控的请求，将域控Relay到普通的机子，比如域管运维所在的机子。(为啥不Relay到其他域控，因为域内就域控默认开启smb签名)

### 利用

以下演示在没有特殊说明情况都是从域控relay到其他机器上

+ impacket下的smbrelayx.py

![smbrealyx](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/smbrealyx.png)

+ impacket 的底下的`ntlmrelayx.py`

![ntlmrelayx](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/ntlmrelayx.png)

当域控访问攻击机时会relay到PC上执行命令，自带的一个权限提升

+ Responder下的MultiRelay.py

  ![multirelay](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/multirelay.png)

接下来演示从域机器relay到域机器

经过不断地测试，发现有以下条件：

> 域用户不被限制登录到目标机器
>
> 域用户对目标机器有管理员权限

满足以上两点即可relay,说到这里想起来一篇文章的总结

- 域普通用户 != 中继
- 域管 == 中继
- 域普通用户+域管理员组 == 中继

除了域管理员组换成目标机器的管理员组，大概是相同的

攻击与域控relay无异，只需要在已控机器上访问攻击机即可

![u2u](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/u2u.png)

20为win7，登录用户为hacker,hacker在win10管理员组中

## Relay2EWS

使用工具：[ntlmRelaytoEWS](https://github.com/Arno0x/NtlmRelayToEWS)

Exchange的认证也是支持NTLM SSP的。我们可以relay的Exchange，从而收发邮件，代理等等。在使用outlook的情况下还可以通过homepage或者下发规则达到命令执行的效果。而且这种Relay还有一种好处，将Exchange开放在外网的公司并不在少数，我们可以在外网发起relay

这里先贴贴别的师傅的截图和代码

```HTML
<html>
<head>
<meta http-equiv="Content-Language" content="en-us">
<meta http-equiv="Content-Type" content="text/html; charset=windows-1252">
<title>Outlook</title>
<script id=clientEventHandlersVBS language=vbscript>
<!--
 Sub window_onload()
     Set Application = ViewCtl1.OutlookApplication
     Set cmd = Application.CreateObject("Wscript.Shell")
     cmd.Run("calc")
 End Sub
-->

</script>
</head>

<body>
 <object classid="clsid:0006F063-0000-0000-C000-000000000046" id="ViewCtl1" data="" width="100%" height="100%"></object>
</body>
</html>
```

 [来源](https://daiker.gitbook.io/windows-protocol/ntlm-pian/6#2.-relay2ews)

等换了域控再写一下

## Relay2LDAP 

protocol上提供了三个通用性的思路，根据权限的不同来区分

![123353](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/123353.png)

### 高权限用户

如果NTLM发起用户在以下用户组

- Enterprise admins
- Domain admins
- Built-in Administrators
- Backup operators
- Account operators

那么就可以将任意用户拉进该组，从而使该用户称为高权限用户，比如域管

### Write Acl 权限

如果发起者对`DS-Replication-GetChanges(GUID: 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2)`和`DS-Replication-Get-Changes-All(1131f6ad-9c07-11d1-f79f-00c04fc2dcd2)`有write-acl 权限，那么就可以在该acl里面添加任意用户，从而使得该用户可以具备dcsync的权限

这个案例的典型例子就是Exchange Windows Permissions组。

### 普通用户权限

对于一个普通的域用户，可能会需要用到基于资源的约束委派，但前提是这个普通域用户能够修改目标主机的属性，能够创建机器账户。

满足这两个条件就能够使用资源约束委派了。详细可以看委派攻击的地方

## 参考链接

[protocol](https://daiker.gitbook.io/windows-protocol/ntlm-pian/6#3.-relay2ldap)

[LM-Hash、NTLM-Hash、Net-NTLMv1、Net-NTLMv2详解](http://d1iv3.me/2018/12/08/LM-Hash、NTLM-Hash、Net-NTLMv1、Net-NTLMv2详解/)

[The NTLM Authentication Protocol and Security Support Provider](http://davenport.sourceforge.net/ntlm.html)
