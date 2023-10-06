### 基础名词

kerberos可以简单分为三个部分：

用户(client)

服务(server)

KDC(Key Distribution Center,密钥分发中心)，KDC又包含AS和TGS

AS(Authentication Server)：身份认证服务

TGS(Ticket Granting Server)：票据授予服务

TGT(Ticket Granting Ticket)：由身份认证服务授予的票据，用于身份认证，存储在内存，默认有效期为10小时

### 认证流程

在介绍kerberos认证流程之前，先来看看kerberos数据库

所有使用kerberos协议的用户和网络服务，在他们添加进kerberos系统中时，都会根据自己当前的密码（用户密码，人为对网络服务随机生成的密码）生成一把密钥存储在kerberos数据库中，且kerberos数据库也会同时保存用户的基本信息（例如用户名，用户IP地址等）和网络服务的基本信息（IP，Server Name）

对于kerberos的认证流程，可以分成三次通信阶段来理解

#### 第一次通信

![第一次通信](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E4%BF%A1.jpg)

+ AS_REQ：Client向KDC以明文的形式发起请求，请求内容包含通过client密码hash加密的用户名，IP和时间戳

+ AS_REP：AS接收请求，先前往kerberos认证数据库中寻找是否存在该用户，此时只会查找是否有相同用户名的用户，并不判断身份的可靠性，如果有则继续下一步，没有则直接失败(或者说是使用client密码hash进行解密，解密成功则返回TGT等)

+ 返回内容：返回内容分为两部分

  第一部分为使用krbtgt用户NTLM HASH加密的TGT票据(Ticket Granting Ticket，票据凭证授权)。TGT包含CT_SK(session_key)，PAC(特权属证书)等，PAC包含Client的相关权限信息，如SID,所在的组等。PAC用于验证用户权限，只有KDC能制作和查看PAC。

  CT_SK用于客户端和TGS间进行通信

  第二部分为客户端密钥加密的一段内容，包含CT_SK,客户端即将访问的TGS的Name,TGT有效时间等

#### 第二次通信

客户端接收到AS传递的内容后，此时客户端会用自己的密钥将第二部分内容进行解密，分别获得时间戳，自己将要访问的TGS的信息，和用于与TGS通信时的密钥CT_SK。首先他会根据时间戳判断该时间戳与自己发送请求时的时间之间的差值是否大于5分钟，如果大于五分钟则认为该AS是伪造的，认证至此失败。如果时间戳合理，客户端便准备向TGS发起请求。

![第二次通信](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/%E7%AC%AC%E4%BA%8C%E6%AC%A1%E9%80%9A%E4%BF%A1.png)

+ TGS_REQ：Client将TGT原封不动的传递给TGS，顺便带上要访问的服务IP，并且传递以CT_SK加密的自身信息

+ TGS_REP：TGS使用krbtgt用户的NTLM HASH对TGT进行解密，解密后会根据时间戳判断通信是否延时，没有则继续下一步，使用CT_SK解密客户端自身的信息，与TGT中的用户信息进行对比，无误则返回响应内容

+ 返回内容：

  第一部分：返回使用服务NTLM HASH加密的TGS票据(ST)并带上PAC，CS_SK.注意，在认证过程中不论用户有没有服务的访问权限，只要TGT正确，就会返回ST。

  第二部分：返回CT_SK加密的ST有效时间，时间戳，CS_SK等

#### 第三次通信

Client接收到了来自TGS的响应，并使用本地缓存的CT_SK解密了第二部分的内容，检查时间戳后取出CS_SK向服务端发起请求

![第三次通信](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/%E7%AC%AC%E4%B8%89%E6%AC%A1%E9%80%9A%E4%BF%A1.png)

+ AP_REQ：Client使用ST去请求服务，并带上使用CS_SK加密的客户端信息等
+ AP_REP：服务端接收来自客户端的请求，并用自己的NTLM-HASH进行解密，取出CS_SK解密第二部分内容，获取到TGS认证的客户端信息和客户端带来的信息，对比无误后，就会将PAC交给KDC解密，KDC由此判断Client是否具有访问服务的权限。如果没有PAC就不会去KDC求证。

**注意：**

上述认证过程中提到的session_key都在认证过程结束后立即释放，并且只使用一次，下一次认证时会产生新的session_key

### 攻击手段

如果是看完了上述认证流程，大概是能猜到攻击出现在哪几个地方，kerberos的攻击手法可以大致分为三种，分别出现在第一次通信，第二次通信和PAC安全问题上

```shell
AS_REQ&ASR_EP过程
|_ 域内用户枚举
|_ 密码喷洒攻击
|_ AS-REP Roasting攻击
|_ 黄金票据
TGS_REQ&TGS_REP阶段攻击
|_ Kerberosast攻击
|_ 白银票据
|_ 委派攻击
		 |_ 非约束委派
		 |_ S4U协议
		 		 |_约束委派
		 		 |_基于资源的约束委派
PAC安全问题
|_ MS14068
|_ CVE-2021-4227&CVE-2021-42287
```

#### 参考链接：

https://seevae.github.io/2020/09/12/%E8%AF%A6%E8%A7%A3kerberos%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B/

https://www.cnblogs.com/bmjoker/p/10723432.html