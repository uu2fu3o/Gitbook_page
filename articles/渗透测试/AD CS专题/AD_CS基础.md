## Active Directory 证书服务安装

没什么特别需要注意的地方，一直下一步即可（注意将web注册服务勾选）

![image-20231212140115385](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212140115385.png)

完了配置证书服务

![image-20231212140238086](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212140238086.png)

这里需要选择企业CA

![image-20231212140841888](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212140841888.png)

## Active Directory 证书服务概述

+ 企业PKI

PKI即公钥基本结构，用来实现证书的产生、管理、存储、分发和撤销等功能

+ 证书颁发机构

  证书颁发机构 (CA) 接受证书申请，根据 CA 的策略验证申请者的信息，然后使用其私钥将其数字签名应用于证书。然后 CA 将证书颁发给证书的使用者。此外，CA 还负责吊销证书和发布证书吊销列表 (CRL)。

  企业CA与ADDS服务结合，他的信息存储在ADDS数据库里面(就是LDAP上)

  **企业CA也支持基于证书模板和自动注册证书**

  Certification Authorities 容器对应根 CA 的证书存储。

  ![image-20231212163727387](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212163727387.png)

  

+ 证书注册 Web 服务

  通过此服务，用户和计算机能够通过 Web 服务执行证书注册。 与证书注册策略 Web 服务一起使用时，可在客户端计算机不是域成员或域成员未连接到域时实现基于策略的证书注册。

+ ADCS服务架构

  ![rain1_ice](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/rain1_ice.webp)

ORCA1：首先使用本地管理员部署单机离线的根 CA，配置 AIA 及 CRL，导出根 CA 证书和 CRL 文件

通常这台机器是离线的，或者是不在域内，防止管理员误操作或者攻击导致根CA的改变，又需要重新下发根CA

为了验证由根 CA 颁发的证书，需要使 CRL 验证可用于所有端点，为此将在从属 CA (APP1) 上安装一个 Web 服务器来托管验证内容。根 CA 机器使用频率很低，仅当需要进行添加另一个从属/颁发 CA、更新 CA 或更改 CRL。

APP1：用于端点注册的从属 CA，通常完成以下关键配置

将根CA放入AD配置容器中，允许域客户计算机自动信任根CA证书，不需要通过组策略下发

在离线 ORCA1上申请 APP1 的 CA 证书后，利用传输设备将根 CA 证书和 CRL文件放入 APP1 的本地存储中，使 APP1 对根 CA 证书和根 CA CRL 的迅速直接信任

部署 Web Server 以分发证书和 CRL，设置 CDP 及 AIA

+ Enterprise NTAuth store( 企业NTAuth存储 )

  **NTAuthCertificates** 容器定义了有资格颁发身份验证证书的 CA 证书

  ```
  向 NTAuth 发布/添加证书：
  certutil –dspublish –f IssuingCaFileName.cer NTAuthCA
  
  要查看 NTAuth 中的所有证书：
  certutil –viewstore –enterprise NTAuth
  
  要删除 NTAuth 中的证书：
  certutil –viewdelstore –enterprise NTAuth
  ```

  域内机器在注册表中有一份缓存：

  ```
  HKLM\SOFTWARE\Microsoft\EnterpriseCertificates\NTAuth\Certificates
  ```

  当组策略开启“自动注册证书”，等组策略更新时才会更新本地缓存

+ Certificate Revocation List(证书吊销列表)

  位于CDP中包含了CRL的有关信息，例如 URL (Web Server)或 LDAP 路径 (Active Directory)。

  ![image-20231212161206068](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212161206068.png)

  证书吊销列表 (CRL) 是由颁发相应证书的 CA 发布的已吊销证书列表，将证书与 CRL 进行比较是确定证书是否有效的一种方法

- AIA（Authority Information Access

  容器保存了中间 CA 证书的 AD 对象。中间 CA 是 PKI 树层次结构中根 CA 的 “子代”，因此，此容器的存在是为了帮助验证证书链。与 Certification Authorities 容器一样，每个 CA 在 AIA 容器中表示为一个 AD 对象，其中 `objectClass` 属性设置为 CertificationAuthority，并且 `cACertificate` 属性包含 CA 证书的二进制内容。当有新的 CA 安装时，它的证书则会自动放到 AIA 容器中。这些 CAs 传播到每台机器上的中间证书颁发机构证书存储区。

## 证书模板

```
certtmpl.msc
```

开启本地的证书模板控制台，编辑模板证书

这些模板是注册策略和预定义证书设置的集合，包含诸如 “此证书有效期为多久？”、“证书用于什么？”、“如何指定证书的主题？”、“谁可以申请证书？”，以及许多其他设置。

![image-20231212170038046](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231212170038046.png)

接下来介绍一些属性的用途

+ 常规：证书的有效实践

  ![image-20231213094512770](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213094512770.png)

+ 请求处理：证书的目的和导出私钥的能力（可通过mimikatz导出不可导出的证书）

  ![image-20231213094628023](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213094628023.png)

+ 加密：使用的加密服务提供程序(CSP)和最小密钥大小

  ![image-20231213094753944](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213094753944.png)

+ subject：它指示如何构建证书的专有名称：来自请求中用户提供的值，或来自请求证书的域主体的身份。这个需要注意的是，默认勾选了`在使用者名称中说那个电子邮件名`，当用户去申请的时候，如果用户的LDAP属性没有mail就会申请失败。

  ![image-20231213095348552](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213095348552.png)

+ 发布要求：CA证书经理程序批准

  **这个值得注意，就算用户有注册权限(在ACL里面体现)，但是证书模板勾选了CA证书管理程序批准，也得得到证书管理的批准才能申请证书，如果没有勾选这个，那有权限就可以直接申请证书了。**

  ![image-20231213095953462](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213095953462.png)

+ 安全描述符：证书模板的 ACL，包括具有注册到模板所需的扩展权限的主体的身份。

+ 扩展：包含在证书中的 X509v3 扩展列表及其重要性（包括`KeyUsage`和`ExtendedKeyUsages`）

  这里有几个关键的地方，我们主要利用的点也是这里

  **Extended Key Usages (EKUs)**：扩展密钥用法，描述证书将如何使用的对象标识符（OID）。常见的 EKU OID 包括：

  - 代码签名（OID 1.3.6.1.5.5.7.3.3）：证书用于签署可执行代码。
  - 加密文件系统（OID 1.3.6.1.4.1.311.10.3.4）：证书用于加密文件系统。
  - 安全电子邮件（OID 1.3.6.1.5.5.7.3.4）：证书用于加密电子邮件。
  - 客户端身份验证（OID 1.3.6.1.5.5.7.3.2）：证书用于身份验证到另一个服务器。
  - 智能卡登录（OID 1.3.6.1.4.1.311.20.2.2）：证书用于智能卡认证。
  - 服务器认证（OID 1.3.6.1.5.5.7.3.1）：证书用于识别服务器（例如，HTTPS 证书）。

  ![image-20231213100853153](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213100853153.png)

  当特定的 EKU 出现在证书中时，该证书允许被用于对 AD 进行身份验证。根据 Specterops 的研究，拥有以下 OID 的证书可以用于身份验证：

  | Description                   | OID                    |
  | ----------------------------- | ---------------------- |
  | Client Authentication         | 1.3.6.1.5.5.7.3.2      |
  | PKINIT Client Authentication* | 1.3.6.1.5.2.3.4        |
  | Smart Card Logon              | 1.3.6.1.4.1.311.20.2.2 |
  | Any Purpose                   | 2.5.29.37.0            |
  | SubCA（子CA）                 | (no EKUs)              |

  **默认情况下，AD CS 部署中不存在 OID 1.3.6.1.5.2.3.4，因此需要⼿动添加，但它确实适⽤于客户端⾝份验证。**

  此外，Specterops 发现可以滥⽤的另⼀个 EKU OID 是**证书请求代理** OID 1.3.6.1.4.1.311.20.2.1。除⾮设置了特定限制，否则具有此 OID 的证书可⽤于代表其他用户申请证书。

  如图这些OID

  ![image-20231213102137051](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213102137051.png)

  当然这些证书模板同样被保存在AD目录中

  ```
  CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=loser,DC=com,ECHANGE2013.loser.com
  ```

  属性pKIExtendedKeyUsage定义了该证书使包含OID

  ![image-20231213104004019](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213104004019.png)

## 证书注册

### 注册流程

![image-20231213111203742](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213111203742.png)

1.客户端生成一个证书申请文件

2.客户端把证书申请文件发送给CA，选择一个证书模板

3.CA会判断证书模板是否存在，根据模板判断用户是否有权限申请证书。证书模板会决定证书的主题名是什么，证书的有效时间是多久，证书用于干啥。是不是需要证书管理员批准。

4.CA会使用自己的私钥来签署证书。签署完的证书可以在颁发列表里面看到

### 注册权限

用户不一定从每个定义的证书模板中获取证书。模板由网络管理员创建，再由企业CA发布，客户能够注册。AD CS 在 AD 中将企业 CA 注册为 `objectClass` 属性为 `pKIEnrollmentService` 的对象。AD CS 通过将模板的名称添加到对象的 `certificateTemplates` 属性来指定在企业 CA 上所启用的证书模板

![image-20231213141124236](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213141124236.png)

AD CS 使用两个安全描述符定义注册权限：一个在证书模板 AD 对象上，另一个在企业 CA 本身上。

对于证书模板，模板的 DACL 中的以下 ACE 可能会导致主体具有注册权限：

- ACE 为主体授予证书注册（Certificate-Enrollment）扩展权限。这个 ACE 授予主体 `RIGHT_DS_CONTROL_ACCESS` 访问权限，其中 `ObjectType` 设置为 `0e10c968-78fb-11d2-90d4-00c04f79dc5547`。此 GUID 对应于 Certificate-Enrollment 权限。
- ACE 为主体授予证书自动注册 （Certificate-AutoEnrollment）扩展权限。这个 ACE 授予主体 `RIGHT_DS_CONTROL_ACCESS` 访问权限，其中 `ObjectType` 设置为 `a05b8cc2-17bc-4802-a710-e7c15ab866a249`。此 GUID 对应于 Certificate-AutoEnrollment 扩展权限。
- ACE 为主体授予所有扩展（ExtendedRights）权限。这个 ACE 启⽤ `RIGHT_DS_CONTROL_ACCESS` 访问权限，其中 `ObjectType` 设置为 `00000000- 0000-0000-0000-000000000000`。此 GUID 对应于 ExtendedRights 权限。
- ACE 为主体授予完全控制（FullControl/GenericAll）权限。这个 ACE 启用 FullControl/GenericAll 访问权限。

执行以下命令，查看指定主体对指定证书模板所拥有的 ACE，如下图所示。

```powershell
Import-Module ActiveDirectory
cd AD:
$Acl = Get-Acl 'CN=User,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=loser,DC=com'
$Acl.Access.Count
$Acl.Access | where IdentityReference -match 'Domain Users'
```

![image-20231213142702749](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213142702749.png)

通过图形化界面设置证书模板的权限

![image-20231213143000001](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213143000001.png)

勾选注册权限，则这个组内的用户都会拥有注册权限所有的这些安全设置都将在 CA 服务器上的注册表 

```
HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration\<CA NAME>中设置键值 Security 
```

在企业CA上，查看CA对于用户访问权限的设置

![image-20231213151859563](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213151859563.png)

### 注册方式

如果企业 CA 和证书模板的安全描述符都授予客户端证书注册权限，则客户端可以请求证书。客户端可以根据 AD CS 环境的配置以不同⽅式请求证书：

1.通过web服务进行申请，需要安装web服务组件

![image-20231213150802793](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213150802793.png)

2.通过本地GUI进行申请

启动 `certmgr.msc`（用于申请用户证书）或 `certlm.msc`（用于申请计算机证书），右键单击 “个人”，选择 “所有任务”，选择 “申请新证书”

![image-20231213150941157](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231213150941157.png)

3.使用 Windows 客户端证书注册协议（MS-WCCE），这是一组分布式组件对象模型（DCOM）接口，可与包括注册在内的各种 AD CS 功能进行交互。DCOM 服务器默认在所有 AD CS 服务器上启用，这也是客户端申请证书的最常用方法

4.通过 ICertPassage 远程协议（MS-ICPR），一种可以在命名管道或 TCP/IP 上运行的 RPC 协议。

5.与证书注册服务（CES）交互。要使用此功能，服务器需要安装 “证书注册 Web 服务” 角色。启用后，用户可以通过 `https://<CESSERVER>/<CANAME>_CES_Kerberos/service.svc` 访问 Web 服务以请求证书。此服务与证书注册策略（CEP）服务（通过证书注册策略 Web 服务角色安装）协同工作，客户端使用该服务在 URL `https://<CEPSERVER>/ADPolicyProvider_CEP_Kerberos/service.svc` 中列出证书模板

6.使用网络设备注册服务。要使用它，服务器需要安装 “网络设备注册服务” 角色，它允许客户端（即网络设备）通过简单证书注册协议（SCEP）获取证书。启用后，管理员可以在 URL `http://<NDESSERVER>/CertSrv/mscep_admin/` 中获取一次性密码（OTP）。然后管理员可以将 OTP 提供给网络设备，该设备将使用 SCEP 通过 URL `http://NDESSERVER/CertSrv/mscep/` 请求证书。

还可以使用内置的 `certreq.exe` 命令或 PowerShell 的 `Get-Certificate` Cmdlet 进行证书注册。在非 Windows 机器上，客户端可以使用基于 HTTP 的接口来申请证书。

CA 颁发证书后，可以通过 `certsrv.msc` 吊销颁发的证书。默认情况下，AD CS 使用证书吊销列表（CRL）分发吊销的证书信息，它们基本上只是每个被撤销证书的序列号的列表。

## 窃取证书

在我们可控的计算机上可能会存在一些证书，这些证书有可能是用客户端身份验证，有可能是CA证书，用以信任其他证书的。我们可以将这些证书导出来。首先需要知道的是，证书分为私钥可导出以及私钥不可导出，私钥可导出的证书我们直接通过图形化界面进行导出就可以了，这是最简单快捷的方法，而私钥不可导出的证书需要使用mimikatz

### 从系统存储导出证书

windows自带命令certutil能够查看当前计算机上的证书，默认查看计算机证书，通过指定参数-user查看用户证书，通过指定-store来查看存储分区，如CA,root,My分别对应中间证书机构`,`个人证书`,`受信任的根证书颁发机构。

(图形化查看用户证书是命令是`certmgr.msc`，图形化查看计算机证书的命令是`certlm.msc`)

通过certutil -store My来查看个人证书

![image-20231227153327843](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227153327843.png)

接下来导出证书，找到我们想导出的证书的hash

如果只是导出证书，不导出密钥

```
certutil -user -store My d950bcdd9e12d8d89b447416490e81a6718711e5 c:\Users\Administrator\Desktop\test1.cer
```

如果要导出的证书包含私钥

```
certutil -user -exportPFX d950bcdd9e12d8d89b447416490e81a6718711e5 c:\Users\Administrator\Desktop\test1.pfx
```

导出过程中需要输入密码，这个密码用于导入证书到计算机时使用

![image-20231227154732329](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227154732329.png)

如果是私钥不可导出的证书，在执行对应命令时会出现这样的情况，这里以计算机证书举例

![image-20231227201924337](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227201924337.png)

```
certutil -exportPFX ee27965b1367f2bb1479fc1ed4dde5dcaa6e2b14 c:\Users\Administrator\Desktop\machine.pfx
```

![image-20231227202118646](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227202118646.png)

这时候我们就需要使用mimikatz了，mimikatz的`crypto::capi` 和 `crypto::cng` 命令可以 Patch CAPI 和 CNG 以允许导出私钥。`crypto::capi` 在当前进程中 Patch CAPI，而 `crypto::cng` 可以 Patch lsass.exe 的内存。此后用`crypto::certificates`进行导出（需要注意的是，这个模块默认情况下也是不支持导出私钥不可导出类型）

![image-20231227203020792](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227203020792.png)

使用crypto::capi进行patch

![image-20231227203218239](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227203218239.png)

现在可以顺利导出了

![image-20231227203628232](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227203628232.png)

### 从文件系统搜索证书

有时证书和它们的私钥只是散落的分布在文件系统上，不需要从系统存储中提取它们。例如，我们在文件共享、管理员的下载文件夹、源代码存储库和服务器的文件系统以及许多其他地方中看到了导出的证书及其私钥。

下面是一些后缀

```
.key 		只包含私钥
crt/cer 	只包含公钥
csr 		证书申请文件，不包含公钥也不包含私钥
pfx,pem,p12 包含公私钥，最关键的
```

## 申请可用于kerberos认证的证书

在证书模板的位置提到过有几个比较关键的扩展，想要向kerberos进行申请，证书必须具有5个关键扩展的其中之一

![image-20231227212019924](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227212019924.png)

一个用户想要申请证书，需要通过两个权限校验

+ 在CA上具有请求证书的权限，这个默认都有
+ 在证书模板上有注册权限

我们之前提到过，有的证书模板可能设置了发布要求需要管理员审批，我们需求的模板最好是不需要管理员进行审核的。

那么综上，能定位到用户/计算机这两个证书模板

+ 这两个模板有什么优点？

  1.他们的扩展属性都有客户端身份验证

  2.用户证书默认所有域用户都可以注册

  ![image-20231227213350320](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227213350320.png)

  3.计算机证书模板默认所有计算机都可以注册

  ![image-20231227213433055](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227213433055.png)

  4. 这两个证书模板在注册时不需要经过管理员的审核

+ 当我们获取到对应的用户凭据后，如何去申请证书呢

  通过web页面进行申请，访问https://CA/certsrv,提交用户证书申请即可

  ![image-20231227214406849](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227214406849.png)

  通过`certmgr.msc`申请，适用于在域内的情况

  ![image-20231227214904279](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231227214904279.png)
