这个阶段打算把字段挑出来单独介绍一下，顺便介绍介绍委派

## TGS_REQ

![1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/1.png)

TGS_REQ数据包如下

```SHELL
Kerberos
    Record Mark: 1335 bytes
        0... .... .... .... .... .... .... .... = Reserved: Not set
        .000 0000 0000 0000 0000 0101 0011 0111 = Record Length: 1335
    tgs-req
        pvno: 5
        msg-type: krb-tgs-req (12)
        padata: 1 item
            PA-DATA pA-TGS-REQ
                padata-type: pA-TGS-REQ (1)
                    padata-value: 6e8204923082048ea003020105a10302010ea20703050000000000a38204096182040530…
                        ap-req
                            pvno: 5
                            msg-type: krb-ap-req (14)
                            Padding: 0
                            ap-options: 00000000
                                0... .... = reserved: False
                                .0.. .... = use-session-key: False
                                ..0. .... = mutual-required: False
                            ticket
                                tkt-vno: 5
                                realm: HACK.COM
                                sname
                                    name-type: kRB5-NT-SRV-INST (2)
                                    sname-string: 2 items
                                        SNameString: krbtgt
                                        SNameString: hack.com
                                enc-part
                                    etype: eTYPE-AES256-CTS-HMAC-SHA1-96 (18)
                                    kvno: 2
                                    cipher: 41590d9624a82c3a7f3adc91125074691bef1fd2c835edf2542758d2527a716883128737…
                            authenticator
                                etype: eTYPE-ARCFOUR-HMAC-MD5 (23)
                                cipher: 864f8ffbfdb0d3d28609a8f7f406883dd170da239cf2432462e129c294d6b711cc561953…
        req-body
            Padding: 0
            kdc-options: 00000000
                0... .... = reserved: False
                .0.. .... = forwardable: False
                ..0. .... = forwarded: False
                ...0 .... = proxiable: False
                .... 0... = proxy: False
                .... .0.. = allow-postdate: False
                .... ..0. = postdated: False
                .... ...0 = unused7: False
                0... .... = renewable: False
                .0.. .... = unused9: False
                ..0. .... = unused10: False
                ...0 .... = opt-hardware-auth: False
                .... 0... = unused12: False
                .... .0.. = unused13: False
                .... ..0. = constrained-delegation: False
                .... ...0 = canonicalize: False
                0... .... = request-anonymous: False
                .0.. .... = unused17: False
                ..0. .... = unused18: False
                ...0 .... = unused19: False
                .... 0... = unused20: False
                .... .0.. = unused21: False
                .... ..0. = unused22: False
                .... ...0 = unused23: False
                0... .... = unused24: False
                .0.. .... = unused25: False
                ..0. .... = disable-transited-check: False
                ...0 .... = renewable-ok: False
                .... 0... = enc-tkt-in-skey: False
                .... .0.. = unused29: False
                .... ..0. = renew: False
                .... ...0 = validate: False
            cname
                name-type: kRB5-NT-PRINCIPAL (1)
                cname-string: 1 item
                    CNameString: Administrator
            realm: hack.com
            sname
                name-type: kRB5-NT-SRV-INST (2)
                sname-string: 2 items
                    SNameString: krbtgt
                    SNameString: hack.com
            till: Sep 13, 2037 18:48:05.000000000 中国标准时间
            nonce: 1818848256
            etype: 1 item
                ENCTYPE: eTYPE-ARCFOUR-HMAC-MD5 (23)

```



#### msg-type

TGS_REQ对应的就是KRB_TGS_REQ(0x0c)

#### PA-DATA

+ AP_REQ

  该结构体中存放着AS_REP获取到的TGT票据,KDC会校验他来返回TGS票据

+ PA_FOR_USER

   (没有该扩展则没有该字段)

  类型是S4U2SELF

  值是一个唯一的标识符，该标识符指示用户的身份 该唯一标识符由用户名和域名组成

  S4U2proxy 必须扩展PA_FOR_USER结构，指定服务代表某个用户(图片里面是administrator)去请求针对服务自身的kerberos服务票据

+ PA_PAC_OPTIONS

  值是以下flag的组合

  -- Claims(0)

  -- Branch Aware(1)

  -- Forward to Full DC(2)

  -- Resource-based Constrained Delegation (3)

  微软的[MS-SFU 2.2.5](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/aeecfd82-a5e4-474c-92ab-8df9022cf955)， 使用基于资源的约束委派，S4U2proxy 应该 扩展 PA-PAC-OPTIONS 结构

  ```
   PA-PAC-OPTIONS ::= KerberosFlags 
     -- resource-based constrained delegation (3)
  ```

  指定Resource-based Constrained Delegation (3)位

#### REQ-BODY

+ sname

  ```
              sname
                  name-type: kRB5-NT-SRV-INST (2)
                  sname-string: 2 items
                      SNameString: krbtgt
                      SNameString: hack.com
  ```

  请求的服务名称，TGS票据是使用该服务用户的ntlm-hash进行加密的。如果请求的服务是krbtgt，那么拿到的TGS可以当作TGT使用

+ AdditonTicket

  附加票据，在S4U2proxy请求里面，既需要正常的TGT，也需要S4U2self阶段获取到的TGS，那么这个TGS就添加到AddtionTicket里面。

## TGS_REP

数据包如下

```shell
Kerberos
    Record Mark: 1322 bytes
        0... .... .... .... .... .... .... .... = Reserved: Not set
        .000 0000 0000 0000 0000 0101 0010 1010 = Record Length: 1322
    tgs-rep
        pvno: 5
        msg-type: krb-tgs-rep (13)
        crealm: HACK.COM
        cname
            name-type: kRB5-NT-PRINCIPAL (1)
            cname-string: 1 item
                CNameString: Administrator
        ticket
            tkt-vno: 5
            realm: HACK.COM
            sname
                name-type: kRB5-NT-SRV-INST (2)
                sname-string: 2 items
                    SNameString: krbtgt
                    SNameString: hack.com
            enc-part
                etype: eTYPE-ARCFOUR-HMAC-MD5 (23)
                kvno: 2
                cipher: 5f794a8a8a3035ecfd4269c30b0dc26908dd06d39db80434f39252f3502c553ef3e525dc…
        enc-part
            etype: eTYPE-ARCFOUR-HMAC-MD5 (23)
            cipher: dd63c996246084df190c9e6bbf542d82f06930524922b943d30fcdbc92df84c8a57cba74…
```

#### msg_type

AS_REQ的响应body对应的就是KRB_TGS_REQ(0x0d)

#### ticket

返回的TGS票据，使用用户申请的对应服务的ntlmhash加密,用户无法读取其中的内容，如果我们拥有服务的hash，我们就能自己制作一个ticket.这里会产生两个安全问题，一是白银票据，而是kerberosting，即hash的破解

#### enc_part

并不是ticket内的enc_part，这一部用户可以解密，对应的session_key在AS_REP阶段就传给了用户，而这一部分内容里面包含了新的session_key等信息，新的session_key用来作为下一阶段的认证密钥

## S4U

### S4U2SELF

根据微软官方的介绍

S4U2self 扩展允许服务代表用户获取自身的服务票证。使用用户的名称和域名向 KDC 标识用户。或者，可以根据用户的证书标识用户。Kerberos 票证授予服务 （TGS） 交换请求和响应消息（KRB_TGS_REQ和KRB_TGS_REP）与两种新数据结构之一一起使用。当通过用户名和领域名称向 KDC 标识用户时，将使用新的 PA-FOR-USER 数据结构。另一种结构 PA-S4U-X509-USER 在向 KDC 提供用户证书以获取授权信息时使用。通过代表用户获取自身的服务票证，服务在票证中接收用户的授权数据。

![s4u2self](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/s4u2self.png)

> 用户会通过其他协议（例如NTLM或基于表单的身份验证）对服务进行身份验证，因此他们不会将TGS发送给服务。在这种情况下，服务可以调用`S4U2Self`来要求身份验证服务为其自身的任意用户生成一个TGS，然后可以在调用S4U2Proxy时将其用作凭证

需要注意的是: **服务代表用户获得针对服务自身的kerberos票据这个过程，服务是不需要用户的凭据的**

**条件**

+ TGT可转发

  ![TGTfoward](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/TGTfoward.png)

+ 服务配置了约束委派

+ 服务请求了可转发选项

  ![forwardoption](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/forwardoption.png)

则TGS必须将**票证标志** 字段设置为可转发，需要注意的是，如果用户的UserAccountControl字段中设置了USER_NOT_DELEGATED位,那么返回的TGS是永远也没法转发的。

![2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/2.png)

也就是上图的敏感账户不能被委派，需要关闭。

![s4u2self2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/s4u2self2.png)

如上图S4U2SELF的请求过程，服务代替用户去请求服务，事实上从KDC请求TGS票据回来(需要有服务用户向KDC请求的TGT票据)，这也说明了为什么S4U2SELF这个请求过程是不需要用户凭据的。

通过TGS_REQ和TGS_REQ的流量可以看出，TGS_REQ由hacker向hacker这个服务进行申请，到KDC后，通过S4U2SELF代替administrator请求一个对应服务的TGS票据

![s4u2self3](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/s4u2self3.png)

### S4U2Proxy

用户到代理服务 （S4U2proxy） 扩展提供代表用户获取另一个服务的服务票证的服务。此功能称为约束委派。Kerberos 票证授予服务 （TGS） 交换请求和响应消息（KRB_TGS_REQ和KRB_TGS_REP）与新的 CNAME-IN-ADDL-TKT 和S4U_DELEGATION_INFO数据结构一起使用。第二个服务通常是代表第一个服务执行某些工作的代理，并且代理在用户的授权上下文中执行该工作。

![S4U2Proxy](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/S4U2Proxy.png)

在(1)中，服务1向KDC获取可转发的TGS票据，并将返回的服务票证作为下一步请求中的AddtionTicket，需要满足以下条件

+ 拥有来在用户的授权(S4U2SELE阶段获得的票据)，将其填入AddtionTicket.

  ![addtionticket](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/addtionticket.png)

+ 在请求的kdc-options中设置了CNAME-IN-ADDL-TKT标志

+ 服务请求了可转发选项
+ 服务1 有到服务2的约束委派，将服务2的SPN放在sname里面

满足以上条件，将会返回服务票证。可转发标志将在服务票证中设置

![s4u2proxy2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/s4u2proxy2.png)

如上图所示，hacker--> cifs/DC  administrator@hack.com --> cifs/DC.ticket 该过程在KDC完成

### S4U2PROXY生成的票据是否可转发问题

关于这个问题可以通过AddtionTicket中的票据是否能够转发来分析

+ AddtionTicket中的票据可转发

  如果AddtionTicket里面的票据是可转发的，只要KDC Options里面置forwarable位，那么返回的票据必须置为可转发的

  例子和上文的相同，不再贴图

+ AddtionTicket中的票据不可转发

  ![noforwardtgs](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/noforwardtgs.png)

如图所示生成的不可转发TGS，将其导出用于下一步。

如果AddtionTicket中的服务票据未设置为可转发的，则KDC必须返回状态为STATUS_NO_MATCH的KRB-ERR-BADOPTION（error-code 13）选项。除了一种情况之外，就是配置了服务1到服务2 的基于资源的约束委派，且PA-PAC-OPTION设置了Resource-Based Constrained Delegation标志位(这一例外的前提是S4U2SELF阶段模拟的用户没被设置为对委派敏感，对委派敏感的判断在S4U2SELF阶段，而不是S4U2PROXY阶段)。

> 通过powerview或ActiveDirectory模块来修改msDS-AllowedToActOnBehalfOfOtherIdentity属性配置基于资源的约束委派

```shell
Set-ADComputer PC -PrincipalsAllowedToDelegateToAccount hacker
Get-ADComputer PC -Properties PrincipalsAllowedToDelegateToAccount
```

![ziyuanweipai1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/ziyuanweipai1.png)

> PA-PAC-OPTION设置了Resource-Based Constrained Delegation标志位

此时发包，得到的票据是可转发的

![kezhuanfa](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/kezhuanfa.png)

### 总结一下S4U

s4u2self像一个代理协议，不需要用户凭证就能拿到自身服务的TGS，但是需要服务账户。s4u2proxy则是以service1向KDC请求service2的TGS票据，这个过程会判断msDS-AllowedToDelegateTo的SPN来确定是否可以申请到service2的TGS,基于资源的约束委派则是看msDS-AllowedToActOnBehalfOfOtherIdentity。

可结合下图看整个流程，方便理解

![s4u](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/s4u.png)

## 委派

在Windows 2000 Server首次发布Active Directory时，Microsoft必须提供一种简单的机制来支持用户通过Kerberos向Web Server进行身份验证并需要代表该用户更新后端数据库服务器上的记录的方案。这通常称为“ Kerberos双跳问题”，并且要求进行委派，以便Web Server在修改数据库记录时模拟用户。需要注意的一点是接受委派的用户只能是**服务账户**或者**计算机用户**。

### 非约束委派

非约束委派配置如下

![feiyueshuweipai](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/feiyueshuweipai.png)

如果服务hacker配置了非约束的委派，那hacker就可接受任何用户的委派去请求其他服务。协议层来讲，服务hacker获取到用户的可转发TGT票据(TGS票据内)，这一步是通过AP_REQ来完成的。并将其缓存到lsass进程当中，方便以后使用。服务hacker通过获取到票据受用户委派模拟用户去访问service2.

![image001](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/image001.png)

如图，接下来通过jackson -> hacker -> cifs/PC 来模拟完整流程，如下图

![非约束委派](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/%E9%9D%9E%E7%BA%A6%E6%9D%9F%E5%A7%94%E6%B4%BE.png)

三个请求分别对应了jackson请求TGT，jackson请求hacker服务的TGS，hacker服务被委派请求cifs/PC服务

![非约束委派2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/%E9%9D%9E%E7%BA%A6%E6%9D%9F%E5%A7%94%E6%B4%BE2.png)

如上图，hacker向cifs/PC请求服务，返回jackson对cifs/PC的TGS票据，hacker成功模拟用户申请到了票据。（左为TGS_REQ）

配置了非约束委派的用户的userAccountControl 属性有个FLAG位 TrustedForDelegation。

### 约束委派

其实约束委派在上文的S4U2PROXY中就已经提到过，s4u2self和s4u2proxy是kerberos协议中的一组扩展，被用在约束委派当中。为服务配置约束委派后，约束委派将限制指定服务器可以代表用户执行的服务。这需要域管理员特权(其实严谨一点是SeEnableDelegation特权，该特权很敏感，通常仅授予域管理员)才能为服务配置域帐户，并且将帐户限制为单个域。

配置了约束委派的用户如下

![hacker](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/hacker.png)

设置了约束委派的hacker服务，可以接手任何用户的委派去请求特定的服务，例如cifs/PC.具体过程是收到用户的请求之后，首先代表用户获得针对服务自身的可转发的kerberos服务票据(S4U2SELF)，拿着这个票据向KDC请求访问特定服务的可转发的TGS(S4U2PROXY)，并且代表用户访问特定服务，而且只能访问该特定服务。

相较于非约束委派，约束委派最大的区别也就是配置的时候选择某个特定的服务，而不是所有服务。

配置了约束委派的用户的userAccountControl 属性有个FLAG位 TrustedToAuthForDelegation 。约束的资源委派，除了配置TRUSTED_TO_AUTH_FOR_DELEGATION 之外，还有个地方是存储对哪个spn 进行委派的，位于msDS-AllowedToDelegateTo

![useraccount](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/useraccount.png)

![msds'](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/msds'.png)

还是抓一个详细流程来看看

![约束委派流程](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/%E7%BA%A6%E6%9D%9F%E5%A7%94%E6%B4%BE%E6%B5%81%E7%A8%8B.png)

如图所示的三个流程分别对应

+ 服务hacker向KDC请求自身的TGT票据（可转发）
+ 服务hacker通过s4u2self模拟用户请求自身的TGS票据（可转发）
+ 服务hacker受用户委派模拟用户，通过s4u2proxy请求服务cifs/PC

### 基于资源的约束委派

基于资源的约束委派只能在运行Windows Server 2012 R2和Windows Server 2012的域控制器上配置，但可以在混合模式林中应用。

这种约束委派的风格与传统约束委派非常相似，但配置相反。从帐户A到帐户B的传统约束委派在msDS-AllowedToDelegateTo属性中的帐户A上配置，并定义从A到B的“传出”信任，而在msDS-AllowedToActOnBehalfOfOtherIdentity属性中的帐户B上配置基于资源的约束委派，并定义从A到B的“传入”信任，见下图。

![t01fe78dcd0b3af3aa4](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/t01fe78dcd0b3af3aa4.jpg)

需要serviceA配置如下

![service1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/service1.png)

serviceB如下

![serviceB](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/serviceB.png)

需要配置serviceB指向A的资源委派,已在上文中提到，不再赘述

现抓包分析流程

![基于资源的约束委派](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/%E5%9F%BA%E4%BA%8E%E8%B5%84%E6%BA%90%E7%9A%84%E7%BA%A6%E6%9D%9F%E5%A7%94%E6%B4%BE.png)

如图所示的三个流程，分别对应

+ 服务hacker(serviceA)向KDC请求自身的TGT票据 (可转发)

+ 服务hacker向KDC请求自身的TGS票据

  由于服务hacker没有配置TrustedToAuthForDelegation位和msDS-AllowedToDelegateTo 字段，因此返回的TGS票据是不可转发的

  如图

  ![step2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/step2.png)

  > （这一步就区别传统的约束委派，在S4U2SELF里面提到，返回的TGS可转发的一个条件是服务1配置了传统的约束委派，kdc会检查服务1 的TrustedToAuthForDelegation位和msDS-AllowedToDelegateTo  这个字段，由于基于资源的约束委派，是在服务2配置，服务2的msDS-AllowedToActOnBehalfOfOtherIdentity属性配置了服务1的sid）

+ 服务hacker代替用户去请求服务，获取指定服务的TGS票据，并且由于配置了基于资源的约束委派，申请到的TGS票据是可转发的

  如图：

  ![step3](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/step3.png)

## 参考链接

https://daiker.gitbook.io/windows-protocol/kerberos/2#3.-enc_part

https://shu1l.github.io/2020/08/05/kerberos-yu-wei-pai-gong-ji-xue-xi/#%E9%9D%9E%E7%BA%A6%E6%9D%9F%E6%80%A7%E5%A7%94%E6%B4%BE

https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a
