## 特权

A在访问B时，首先判断B是不是需要特权才能访问，如果是则需要检查A的Access token看有没有那个特权

如果我们需要赋予域用户特权一般都是通过组策略下发。比如说默认情况底下的`Default Domain Controllers Policy(GUID={6AC1786C-016F-11D2-945F-00C04FB984F9})`这条组策略会把SeEnableDelegationPrivilege这个特权赋予`Administrators`

![image-20231127104131777](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127104131777.png)

执行whoami /priv来查看当前用户的特权

![image-20231127104543468](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127104543468.png)

### SeEnableDelegationPrivilege

此策略设置确定哪些用户可以在用户或计算机对象上设置“受信任的委派”设置。

这个权限有关委派，默认情况底下，在域内只有`SeEnableDelegationPrivilege`权限的用户才能设置委派。而这个权限默认域内的`Administrators`组的用户才能拥有，所以我们一般都是使用SeEnableDelegationPrivilege这个权限来留后门。

+ 给域用户添加该特权，这一步需要域管权限

  通过组策略来实现

  ![image-20231127122932151](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127122932151.png)

  或者手动添加用户sid`C:\Windows\SYSVOL\sysvol\hack.com\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit`的GptTmpl.inf里面,需要手动更新组策略设置

  ```
  gpupdate /force
  ```

  此后hacker用户就可以进行设置委派操作了

  接下来使用hacker用户进行委派设置建立后门

+ hacker用户需要拥有SeEnableDelegationPrivilege特权，并且对自己有GenericAll / GenericWrite权限(这个默认是没有的)

  ![image-20231127133949398](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127133949398.png)

  在这里！

+ hacker用户需要有spn,在这之前已经设置好了，有spn的用户才可以进行委派操作

  ![image-20231127134346892](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127134346892.png)

+ hacker的useraccountControl需要有TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION

+ 修改hacker的msDS-AllowedToDelegateTo属性，设置他的委派

  ![image-20231127140722249](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127140722249.png)

  这样就可以直接申请票据模拟administrator访问域控了
  
  ![image-20231127144705421](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127144705421.png)
  
  ![image-20231127144717513](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127144717513.png)

除此之外还有其他很多特权

[官方权限文档](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/user-rights-assignment)

## ACL

### 一些有攻击价值的ACL介绍

（1） 对某些属性的WriteProperty ，有以下属性

- member(bf9679c0-0de6-11d0-a285-00aa003049e2)
- servicePrincipalName(28630EBB-41D5-11D1-A9C1-0000F80367C1)
- GPC-File-Sys-Path(f30e3bc1-9ff0-11d1-b603-0000f80367c1)

（2） 扩展权限有

- User-Force-Change-Password(0299570-246d-11d0-a768-00aa006e0529)

  可以在不知道当前目标用户的密码的情况下更改目标用户的密码

- DS-Replication-Get-Changes(1131f6aa-9c07-11d1-f79f-00c04fc2dcd2) 和 DS-Replication-Get-Changes-All(1131f6ad-9c07-11d1-f79f-00c04fc2dcd2)

  对域对象具有这两个扩展权限的用户具备dcsync 权限

（3） 通用权限有

- WriteDacl
- AllExtendedRights
- WriteOwner
- GenericWrite
- GenericAll
- Full Control

下面逐个演示利用方式

+ AddMembers

  将任意用户，组或计算机添加到目标组

  如果一个用户对一个组有**AddMembers**权限，那么这个用户可以将任何用户加入这个组，从而具备这个组的权限。

  hacker用户具备对Domain Admin这个组的**AddMembers**权限，也就是对member(bf9679c0-0de6-11d0-a285-00aa003049e2) 这个属性的写权限。

  ![image-20231127151749264](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127151749264.png)

  这里为了方便就设置为所有权限了，通过admod将任意用户加进Domain Admin.(这里是hacker)

  ![image-20231127152457275](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127152457275.png)

  ![image-20231127152538231](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127152538231.png)

  之后可以使用hacker用户直接psexec

  ![image-20231127152737271](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127152737271.png)

+ servicePrincipalName(28630EBB-41D5-11D1-A9C1-0000F80367C1)

  写入spn权限，如果对一个对象有写入spn权限，就可以对该对象进行kerberosting

+  GPC-File-Sys-Path(f30e3bc1-9ff0-11d1-b603-0000f80367c1)

  将GPO域GPT链接起来，GPT是组策略具体的策略配置信息，其位于域控制器的SYSVOL共享目录下。如果我们能够控制GPC-File-Sys-Path的话，可以将ad活动目录里面的gpo指向我们自定义的GPT，而GPT里面包含的是组策略具体的策略配置信息，也就是说我们可以修改组策略配置信息的内容。

  通常用来留后门

+ User-Force-Change-Password(0299570-246d-11d0-a768-00aa006e0529)

  在不知道当前目标用户的密码的情况下更改目标用户的密码。强制密码更改

  ![image-20231127171102242](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127171102242.png)

  通过admod进行强制密码更改

  ```
  admod -b CN=Administrator,CN=Users,DC=hack,DC=com unicodepwd::123!@#qazwsx -optenc
  ```

  通过修改后的密码去psexec

+ Dcsync

  最常用的，只需要具备以下两个权限就可以进行dcsync

  ```
  'DS-Replication-Get-Changes'     = 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2
  'DS-Replication-Get-Changes-All' = 1131f6ad-9c07-11d1-f79f-00c04fc2dcd2
  ```

  ![image-20231127172559906](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127172559906.png)

  常见的有mimikatz的dcsync以及screatdump支持远程提供用户账户来dump,这里不做演示

+ WriteDACL

  将新ACE写入目标对象的DACL的功能。例如，攻击者可以向目标对象DACL写入新的ACE，从而使攻击者可以“完全控制”目标对象。

  ![image-20231127204026372](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127204026372.png)

  这一条可以通过admod往上添加，或者使用图形化界面,在实际测试时使用admod没有成功，初步认为是分号的问题

  ```
  admod -b dc=hack,dc=com "nTSecurityDescriptor:+:(OA;;CR;1131f6ad-9c07-11d1-f79f-00c04fc2dcd2;;S-1-5-21-754643614-3937478331-2139222398-1116)"
  ```

  如果成功添加该ACE，则对应用户拥有dcsync权限

+ AllExtendedRights

  所有扩展权限。比如，User-Force-Change-Password权限。

+  WriteOwner

  这个权限这个修改Owner为自己。

  而Owner 默认拥有WriteDacl 和 RIGHT_READ_CONTROL权限。因此我们就可以利用WriteDacl的利用方式。

  ![image-20231127211528905](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127211528905.png)

如果我们是域管用户，把所有者改为hacker,hacker就具备了writeDacl权限，例如给hacker用户添加修改密码的权限

+ GenericWrite

可以修改所有参数，因此包括对某些属性的WriteProperty，比如member。

+ GenericAll

这包括riteDacl和WriteOwner，WRITE_PROPERTY等权限。随便找一个利用就行了。

+ Full Control

这个权限就具备以上所有的权限，随便挑一个特殊权限的攻击方式进行攻击就行了。

## AdminSDHolder

AdminSDHolder是位于Active Directory中的系统分区CN=AdminSDHolder,CN=System,DC=hack,DC=com的一个对象

![image-20231127212952520](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127212952520.png)

他会作为域内某些特权组的安全模版。所谓安全模版，就是说有一个进程(SDProp),每隔60分钟运行一次，将这个对象的ACL复制到某些特权组成员的对象的ACL里面去。 这些特权组和用户默认有 · Account Operators · Administrator · Administrators · Backup Operators · Domain Admins · Domain Controllers · Enterprise Admins · Krbtgt · Print Operators · Read-only Domain Controllers · Replicator · Schema Admins · Server Operators

属性adminCount在Active Directory中标记特权组和用户，对于特权组和用户，该属性将设置为1。通过查看adminCount设置为1的所有对象，可以找到所有的特权组和用户。 但值得注意的是。一旦用户从特权组中删除，他们仍将adminCount值保持为1，但Active Directory不再将其视为受保护的对象。因此通过admincount=1匹配到的所有对象，不一定都是特权组

![image-20231127213852259](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127213852259.png)

对于这个作用我们可以有比较直接的利用，添加一条ACE，使hacker用户对该对象有完全控制权。

由于这个ACL过个60分钟会同步到特权组和用户，这个特权组和用户包括域管，所以hacker对域管已经有完全控制的权限了，达到了后门的目的。

![image-20231127214340533](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231127214340533.png)

默认时间可以在注册表当中修改

```
HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters\AdminSDProtectFrequency
```

默认60分钟且该项目不存在，修改需要新增

## 参考链接：

[windows protocol](https://daiker.gitbook.io/windows-protocol/ldap-pian/12)

[microsoft权限&属性](https://learn.microsoft.com/zh-cn/windows/win32/adschema/extended-rights)