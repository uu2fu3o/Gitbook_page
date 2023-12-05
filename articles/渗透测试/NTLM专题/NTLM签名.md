这一篇主要是讲讲为了防止ntlm relay而做的一些防护措施即签名问题。

## 签名方式

简单来说客户端会和服务端商量使用哪一串字符进行加密，如果我们得不到这串字符，就只能进行流量的转发，而无法进行relay

签名的整个过程中有3个key

#### exported_session_key

如下该key的加密规则

```python
def get_random_export_session_key():
    return os.urandom(16)
```

 这个key是随机数。如果开启签名的话，客户端和服务端是用这个做为key进行签名的。

#### key_exchange_key

这个key使用用户密码，Server Challenge,Client Challenge经过一定运算得到(借鉴师傅的图)

![t01f6642281b8157af5](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/t01f6642281b8157af5.png)

#### encrypted_random_session_key

当开启签名时，会使用exported_session_key作为密钥，并且使用key_exchange_key与之运算得到encrypted_random_session_key

具体的加密流程如下：

![jiami](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/jiami.png)

服务端得到encrypted_random_session_key后，与key_exchange_key再次进行运算，得到随机数key,encrypted_random_session_key这个session_key在流量中是明文显示的，我们可以得到，但即便如此，我们也无法对其进行利用，，因为我们没办法拿到key_session_key，就没法得到随机key,也就无法对流量进行加解密，也就不能进行relay

## SMB签名

在一个默认的域环境中，域控制器开启smb签名，域成员机器并没有开启

![smb](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/smb.png)

（在实际测试时我的域控制器实际上并没有启用这个选项）

## LDAP签名

在默认情况底下，ldap服务器就在域控里面，而且默认策略就是协商签名。而不是强制签名。也就是说是否签名是有客户端决定的。服务端跟客户端协商是否签名。(客户端分情况，如果是smb协议的话，默认要求签名的，如果是webadv或者http协议，是不要求签名的)

微软公司于 2019-09-11 日发布相关通告称微软计划于 2020 年 1 月发布安全更新。为了提升域控制器的安全性，该安全更新将强制开启所有域控制器上 LDAP channel binding 与 LDAP signing 功能。

![ldap](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/ldap.png)

## SMB签名流量分析

接下来抓取具体的流量对SMB签名进行简单分析，有助于理解这个过程

![smb2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/smb2.png)

上图为服务端向客户端返回chanllege阶段，在这个过程中是不需要签名的，签名是针对会话产生，在会话过程中之前如何确定两个服务器之间的会话是否需要签名，这一步在**NTLM协商期间完成**，二是允许指示签名是**必需的**、**可选的**还是**禁用的**。这是在客户端和服务器级别完成的设置。

![askkk1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/askkk1.png)

如图的flag标识位，当flag值为1时，标识客户端支持签名，但这并不意味着一定要使用签名，同理服务端也是这样。

![enables](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/enables.png)

![origin](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/origin.png)

这段信息来源于标记部分，是在最初的地方对SMB签名进行了标识，PC表示不需要签名，但是它可以处理带签名的数据包

同理可以查看DC

![respoinse](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ntlm/respoinse.png)

DC表示不仅支持签名，而且需要签名，此后会话过程中的SMB数据包将会被签名。

另外一些细节会在相关漏洞中提到。

### 参考链接：

[windows protocol](https://daiker.gitbook.io/windows-protocol/ntlm-pian/7#1.-guan-yu-qian-ming-de-yi-dian-xi-jie)

[ntlm relay](https://www.geekby.site/2021/09/ntlm-relay/#43-smb--ntlm)

[ntlm_relay_by_hackndo](https://en.hackndo.com/ntlm-relay/)

