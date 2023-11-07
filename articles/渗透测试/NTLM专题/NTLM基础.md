## LM HASH & NTLM HASH

这部分的内容在之前的文章就已经介绍过，就不再赘述，详细可以参看以前的文章

[PTH](https://uu2fu3o.gitbook.io/articles/articles/shen-tou-ce-shi/nei-wang-ti-xi-jian-she/ha-xi-chuan-di-gong-ji-pth)

## NTLM身份验证

之前介绍的不够详细，再看看

![123](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/123.png)

1.用户登录客户端电脑

2.(type 1)客户端向服务器发送type 1(协商)消息,它主要包含客户端支持和服务器请求的功能列表。

3.(type 2)服务器用type 2消息(质询)进行响应，这包含服务器支持和同意的功能列表。但是，最重要的是，它包含服务器产生的Challenge。

4.(type 3)客户端用type 3消息(身份验证)回复质询。用户接收到步骤3中的challenge之后，使用用户hash与challenge进行加密运算得到response，将response,username,challeng发给服务器。消息中的response是最关键的部分，因为它们向服务器证明客户端用户已经知道帐户密码。

5.服务器拿到type 3之后，使用challenge和用户hash进行加密得到response2与type 3发来的response进行比较。如果用户hash是存储在域控里面的话，那么没有用户hash，也就没办法计算response2。也就没法验证。这个时候用户服务器就会通过netlogon协议联系域控，建立一个安全通道,然后将type 1,type 2，type3 全部发给域控(这个过程也叫作Pass Through Authentication认证流程)

6.域控使用challenge和用户hash进行加密得到response2，与type 3的response进行比较

## Net-ntlm Hash

在type3中的响应，有六种类型的响应

- LM(LAN Manager)响应 - 由大多数较早的客户端发送，这是“原始”响应类型。

- NTLM v1响应 - 这是由基于NT的客户端发送的，包括Windows 2000和XP。

- NTLMv2响应 - 在Windows NT Service Pack 4中引入的一种较新的响应类型。它替换启用了 NTLM版本2的系统上的NTLM响应。

- LMv2响应 - 替代NTLM版本2系统上的LM响应。

- NTLM2会话响应 - 用于在没有NTLMv2身份验证的情况下协商NTLM2会话安全性时，此方案会更改LM NTLM响应的语义。

- 匿名响应 - 当匿名上下文正在建立时使用; 没有提供实际的证书，也没有真正的身份验证。“存 根”字段显示在类型3消息中。

  这六种使用的加密流程一样，都是前面我们说的Challenge/Response 验证机制,区别在Challenge和加密算法不同。

  这里我们侧重讲下NTLM v1响应和NTLMv2响应

- v2是16位的Challenge，而v1是8位的Challenge

具体的计算也在PTH一篇中讲过了，了解即可

## SSP & SSPI

SSPI是一个软件接口。分布式编程库（如 RPC）可以将其用于经过身份验证的通信。一个或多个软件模块提供实际的身份验证功能。每个模块（称为安全支持提供程序 （SSP））都作为动态链接库 （DLL） 实现。

```
SSPI(Security Support Provider Interface)
这是 Windows 定义的一套接口，此接口定义了与安全有关的功能函数， 用来获得验证、信息完整性、信息隐私等安全功能，就是定义了一套接口函数用来身份验证，签名等，但是没有具体的实现。
SSP(Security Support Provider)
​ SSPI 的实现者，对SSPI相关功能函数的具体实现。微软自己实现了如下的 SSP，用于提供安全功能：
NTLM SSP
Kerberos
Cred SSP
Digest SSP
Negotiate SSP
Schannel SSP
Negotiate Extensions SSP
PKU2U SSP
```

**SSP**可用作后门

## LmCompatibilityLevel

此安全设置确定网络登录使用的质询/响应身份验证协议。此选项会影响客户端使用的身份验证协议的等级、协商的会话安全的等级以及服务器接受的身份验证的等级，其设置值如下:

```
发送 LM NTLM 响应: 客户端使用 LM 和 NTLM 身份验证，而决不会使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证。
发送 LM & NTLM - 如果协商一致，则使用 NTLMv2 会话安全: 客户端使用 LM 和 NTLM 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证。
仅发送 NTLM 响应: 客户端仅使用 NTLM 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证。
仅发送 NTLMv2 响应: 客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证。
仅发送 NTLMv2 响应\拒绝 LM: 客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器拒绝 LM (仅接受 NTLM 和 NTLMv2 身份验证)。
仅发送 NTLMv2 响应\拒绝 LM & NTLM: 客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器拒绝 LM 和 NTLM (仅接受 NTLMv2 身份验证)。
默认值:
Windows 2000 以及 Windows XP: 发送 LM & NTLM 响应
Windows Server 2003: 仅发送 NTLM 响应
Windows Vista、Windows Server 2008、Windows 7 以及 Windows Server 2008 R2及以上: 仅发送 NTLMv2 响应
```

[详细介绍](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/network-security-lan-manager-authentication-level)

## 相关安全问题

### 利用NTLM收集信息

[c#版本的Ssmb_version](https://www.zcgonvh.com/post/CSharp_smb_version_Detection.html)

在type2返回challenge的过程中，同时返回了目标主机的版本，主机名等信息，ntlm是一个嵌入式的协议，消息的传输依赖于使用ntlm的上层协议，比如SMB,LDAP,HTTP等。我们以SMB为例。在目标主机开放了445或者139的情况，通过给服务器发送一个type1的请求，然后解析type2的响应。就可以收集到一些信息。

上面链接的脚本拿过来编译一下基本就能使用

msf中也有类似的模块auxiliary/scanner/smb/smb_version

![1234](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/1234.png)

### NTLM RELAY

见下一篇