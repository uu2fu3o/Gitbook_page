## LDAP高级搜索语法

### LDAP位查找操作

在LDAP 里面，有些字段是位字段，这里以userAccountControl举例

他的属性类位于架构分区的`CN=User-Account-Control,CN=Schema,CN=Configuration,DC=hack,DC=com`

`attributeSyntax`是`2.5.5.9`,`oMSyntax`是`2`。

![image-20231120103440538](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120103440538.png)

查询表可得，该位为32位,integer类型

![image-20231120103540291](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120103540291.png)

之所以说它是个位字段，是因为它是由一个个位组成

![image-20231120103917115](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120103917115.png)

比如说一个账户，他的LOCKOUT，以及NOT_DELEGATED，其他的位都没有，那这个用户的属性`userAccountControl`的值就为0x100000+0x0010。是个32 位 int 类型。

LDAP语法支持按位查找，这种查找方式能够过滤具体条目具体属性的具体位，搜寻语法如下：

```
   <属性名称>：<BitFilterRule-ID> := <十进制比较值>
```

![t01a25c7551c331ecdc](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t01a25c7551c331ecdc.png)

我们最常的是AND ，也就是`1.2.840.113556.1.4.803`

例如想查询哪些对象设置了NOT_DELEGATED,该位对应的10进制比较值位为1048576

因此规则如下

```
(userAccoutControl:1.2.840.113556.1.4.803:=1048576)
```

![image-20231120110158145](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120110158145.png)

上一个条目没有结果，换了一个位查，更清晰一些

adfind有一个比较方便的设置，可以使用参数-bit，会自动识别AND等参数

![image-20231120110528337](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/image-20231120110528337.png)

## LDAP 查找中的objectCategory和objectClass

### objectClass

在对象的objectClass属性中可以看到对象是哪个类的实例，以及所有的父类

所有的类都是`top`类的子类。因此当我们过滤`(objectClass=top)`可以找到域内的所有对象

![image-20231120111135715](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/image-20231120111135715.png)

### objectCategory

在Windows Server 2008之前默认不对objectClass 属性进行索引。

Windows 2000 附带了未索引的objectClass 属性和另一个已建立索引的单值属性，称为objectCategory。

`objectCategory`属性：该属性是一个单值属性。并且建立了索引。其中包含对象是其实例的类或其父类之一的专有名称。

比如`CN=PC,CN=Computers,DC=hack,DC=com,DC.hack.com`的objectCategory为`CN=Computer,CN=Schema,CN=Configuration,DC=hack,DC=com`

![image-20231120112624957](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120112624957.png)

创建对象时，系统会将其objectCategory属性设置为由其对象类的`defaultObjectCategory`属性指定的值。无法更改对象的objectCategory属性。

查看schema中Computer类的defaultObjectCategory的值

![image-20231120112831844](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120112831844.png)

如果我们想查询Computer类下的所有实例，即过滤所有objectCategory值为CN=Computer,CN=Schema,CN=Configuration,DC=hack,DC=com的属性，构造·如下语句

```shell
(objectCategory="CN=Computer,CN=Schema,CN=Configuration,DC=hack,DC=com")
```

![image-20231120113201105](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120113201105.png)

为了更方便的查找，我们可以利用属性lDAPDisplayName，该属性指定该类的显示名。

例如CN=Computer,CN=Schema,CN=Configuratn,DC=hack,DC=com的lDAPDisplayName为computer

![image-20231120113904237](C:\Users\86199\AppData\Roaming\Typora\typora-user-images\image-20231120113904237.png)

LDAP在实现上，支持用类的`lDAPDisplayName`作为搜索条件。所以如果我们想找所有`CN=Computer,CN=Schema,CN=Configuration,DC=hack,DC=lcom`的实例，可以简化为以下过滤规则。

```
(objectCategory=computer)
```

![image-20231120114054692](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120114054692.png)

### objectClass 与objectCategory的结合使用

这里以CN=hacker,CN=Users,DC=hack,DC=com举例，

他的`objectClass`是`top,person,organizationalPerson,user`。

他的`objectCategory`是`person`。

一个对象的`objectClass` 是一个类的对象类，以及这个对象类的所有父类。

一个对象的`objectCategory` 是一个类的对象类或者这个对象类的所有父类。

所以说一个对象的`objectCategory` 必定是`objectClass` 中的其中一个。

user，person，organizationalPerson类将其defaultObjectCategory设置为person。这允许像（objectCategory= person）这样的搜索过滤器通过单个查询定位所有这些类的实例。user类的继承关系如下

```shell
top => person => organizationalPerson => user
```

`person`,`organizationalPerson`,`user`都将其defaultObjectCategory设置为person。因此我们可以先过滤。

```
(objectCategory=person)
```

加上objectClass

(&(objectCategory=person)(objectclass=user))

![image-20231120125951927](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120125951927.png)

关于为什么使用这样的查询方式，是为了获取更快的查询速度。事实上单独查询的结果是一致的（对于当前情况），使用obejctCategory可以先划分一个较大的组，再通过objectclass进行精确匹配

![image-20231120130108339](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120130108339.png)

## 组

### 组介绍

组，这个概念并不陌生，例如域管就是一个组。组按照用途划分可以分为安全组和通讯组。通讯组，例如邮件组，将若干个人划分到一个通讯组，给这个通讯组发件，那组内用户都能收到。但是通讯组不能控制对资源的访问。

安全组是权限的集合。将用户拉入拥有特殊权限的安全组中，用户就拥有了组内所设置的特殊权限。

安全组可以根据作用范围划分为。

- 全局组 (Global group)
- 通用组(Universal group)
- 域本地组(Domain Local group)

### 查询组

所有的组都是`group`类的实例，

我们可以用`(objectClass=group)`或者`(objectCategory=group)`来过滤组。

![image-20231120131636684](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120131636684.png)

为什么不一起使用，能够精确匹配的情况下，用其中一个就好了。

组的类型由属性`groupType`决定，属性`groupType`是一个位字段

![image-20231120131834164](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120131834164.png)

GROUP_TYPE_BUILTIN_LOCAL_GROUP：指定系统创建的组。

GROUP_TYPE_ACCOUNT_GROUP：指定全局组。

GROUP_TYPE_RESOURCE_GROUP：指定域本地组。

GROUP_TYPE_UNIVERSAL_GROUP：指定通用组。

GROUP_TYPE_APP_BASIC_GROUP：Active Directory 不使用此类型的组。此常量包含在本文档中，因为 Active Directory 在处理 groupType 属性时使用此常量的值

GROUP_TYPE_APP_QUERY_GROUP：Active Directory 不使用此类型的组。本文档中包含此常量，因为 Active Directory 在处理 groupType 属性时使用此常量的值

GROUP_TYPE_SECURITY_ENABLED：指定启用安全性的组。

+ 域内所有全局组

  ![image-20231120132353552](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120132353552.png)

+ 所有域本地组

  ![image-20231120132433219](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120132433219.png)

+ 所有域通用组

  ![image-20231120132506626](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120132506626.png)

+ 域内所有安全组，包括全局组，通用组，域本地组

  ![image-20231120133142138](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120133142138.png)

+ 域内的所有通讯组，不属于安全组的组都是通讯组

  ![image-20231120133305163](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120133305163.png)

+ 域内系统组

  ![image-20231120133341744](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120133341744.png)

### 组范围

可以看如下表格

| 组类型   | 可以授予权限                         | 可包含                                                       | 可包含于                                                     | 成员是否在全局编录复制 |
| -------- | ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------- |
| 全局组   | 在同一林中或信任域或林中的任何域上。 | 来自同一域的帐户。 来自同一域的其他全局组                    | 来自同一林中任何域的通用组。 来自同一域的其他全局组。 来自同一林中任何域或任何信任域的域本地组。 | 无                     |
| 通用组   | 在同一林或信任林中的任何域上。       | 来自同一林中任何域的帐户。 来自同一林中任何域的全局组。 来自同一林中任何域的其他通用组。 | 同一林中的其他通用组。 在同一个林或信任林中域本地组。        | 是                     |
| 域本地组 | 在同一个域中                         | 来自任何域或任何受信任域的帐户。 来自任何域或任何受信任域的全局组。 来自同一林中任何域的通用组。 来自同一域的其他域本地组。 | 来自同一域的其他域本地组。                                   | 无                     |

- 域本地组(Domain Local group)

顾名思义，就是本域内的本地组。不适用于林，适用于本域。可包含林内的账户，通用组，全局组。其他域内的通用组要在本域拥有权限，一般都是加入这个域的域本地组。比如说一个林里面，只有林根域有`Enterprise Admins`这个组，这是个通用组。然后其他子域 的域本地组`Administrators`会把林根域的`Enterprise Admins`加进里面，所以林根域的`Enterprise Admins`组用户才在整个林内具备管理员权限。如果想要一个只允许访问同一个域中的资源的组，那么使用域本地组即可。

- 通用组(Universal group)

上面已经简单提过了通用组，典型例子是`Enterprise Admins`这个组。在林的场景下比较有用。组内成员会在GC内复制。如果你想要一个可以访问林中任何东西的组，并且可以在林中包含任何账户，请使用通用组。

- 全局组 (Global group)

全局组比较复杂，前面说了。在单域内用域本地组，在林中使用通用组。全局组应该说是一种比较折中的方案，他可以在林中使用，但是只能包含本域内的账户。全局组的使用范围是本域以及受信任关系的其他域。最为常见的全局组是Domain Admin，也就是我们常说的域管。因为全局组只能包含本域内账户，因此来自一个域的账户不能嵌套在另一个域中的全局组中，这就是为什么来自同一个域的用户不符合在外部域中的域管的成员资格（由于其全局范围的影响)。

### 常见组介绍

- Administrators

域本地组。具备系统管理员的权限，拥有对整个域最大的控制权，可以执行整个域的管理任务。Administrators包括`Domain Admins`和`Enterprise Admins`。

- Domain Admins

全局组。我们常说的域管组。默认情况下，域内所有机器会把Domain Admins加入到本地管理员组里面。

- Enterprise Admins

通用组。在林中，只有林根域才有这个组，林中其他域没有这个组，但是其他域默认会把这个组加入到本域的Administrators里面去。

- Domain Users

全局组。包括域中所有用户帐户,在域中创建用户帐户后，该帐户将自动添加到该组中。默认情况下，域内所有机器会把Domain Users加入到本地用户组里面，也就是为什么默认情况底下，啥都不配置。域用户可以登录域内任何一台普通成员机器。

### AGDLP策略

安全组是权限的集合，所以在微软的建议中，并不建议给赋予单个用户权限，而是赋予一个组权限，然后将成员拉近组

A表示用户账号，Account

G表示全局组，Global group 

U表示通用组，Universal Group

L表示本地组， local group

DL表示域本地组，Domain local group

P表示资源权限，Resource Permissions

#### 常见的几种权限划分方式

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t011b7307be86d9898d.jpg)

AGP，将用户账户添加到全局组，然后赋予全局组权限

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t0144ea8fd9fdcfd13c.jpg)

AGLP，将用户账户添加到全局组，将全局组添加到本地组， 然后赋予本地组权限

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t019d2b13a12e69594c.jpg)

ADLP 将用户账户添加到域本地组，然后赋予域本地组权限

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t01b30769e58bc30d1b.jpg)

AGDLP，将用户账户添加到全局组，将全局组添加到域本地组， 然后赋予域本地组权限

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t01ca031e9e49db2bf2.jpg)

AGUDLP，将用户账户添加到全局组，将全局组添加到通用组，将通用组添加到域本地组， 然后赋予域本地组权限

![img](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/t01785b14c5c4f449c4.jpg)

举个例子：有A,B两个域，A中的用户和B的用户想要访问B中的服务，选择在B中建立一个DL，并将用户全部拉进来，此时AB中的用户可以对B中的服务进行访问。此时，A如果新增用户想要访问B中服务，但由于只有B对该DL有控制权限，A中新用户无法直接加入该DL，可以采取在B中建立G，A,B分别建立一个DL，并将两个DL拉入B新建立的G中利用权限继承来解决这个问题。

### 查询组内用户以及用户所属的组

group4是group2组内的成员

group4中会有一个属性memberOf来标识它属于group2

![image-20231120205245943](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120205245943.png)

group2中会有标识member属性标识group2的组成员

![image-20231120205337748](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120205337748.png)

重新配置一下目录，有以下结构

```
group1 => group2 => group4
		  group2 => edd
group3 => edd
```

+ 查询group2有哪些成员

  即查询group2的member属性

  ![image-20231120210731894](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120210731894.png)

  或者通过过滤memberof为group2条目

  ![image-20231120211142344](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120211142344.png)

+ 查询edd属于哪个组

  即查询member属性为edd的对象

  ![image-20231120211358304](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120211358304.png)

  或者查询edd对象的memberOf属性值

  ![image-20231120211451541](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120211451541.png)

  

+ 查看`group1`有哪些成员，这些成员如果是组，就继续查下去，直到非组成员为止

  即采用递归查询，根据LDAP的特性，可以用位操作符的一部分。`BitFilterRule-ID` 为`1.2.840.113556.1.4.1941`.在adfind 里面可以用INCHAIN简化

  现在查询group1的成员，以及成员的成员

  ![image-20231120212205487](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120212205487.png)

  可通过过滤memberof属性进行查看

+  查看用户edd属于哪些组，这些组又属于哪些组，如此往上递归，知道这个组不属于其他组为止

  ![image-20231120212457165](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120212457165.png)

## 组织单位(Organization Unit)

### OU介绍

组织单位(Organization Unit)，简称OU，是一个容器对象，将域中的对象组织成逻辑组，帮助网络管理员简化管理组。组织单位包含下列类型的对象：用户，计算机，工作组，打印机，安全策略，其他组织单位等。可以在组织单位基础上部署组策略，统一管理组织单位中的域对象。 在企业域环境里面，我们经常看到按照部分划分的一个个OU。

![image-20231120213249562](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120213249562.png)

### OU跟容器的区别

组织单位（OU）是专用容器，与常规容器的区别在于管理员可以将组策略应用于OU，然后系统将其下推到OU中的所有计算机。您不能将组策略应用于容器。需要注意的是`Domain Computers`是一个普通容器，而`Domain Controllers`是一个OU，因此可以可以将组策略应该于`Domain Controllers`，不可以将组策略应用于`Domain Computers`。

![image-20231120213747669](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120213747669.png)

这里的domain controllers指的是根域的OU，并不是users下的，users下的domain controllers是组

### OU跟组的区别

组织单位跟组是两个完全不同的概念。很多人经常会把这两个弄混。组是权限的集合。OU是管理对象的集合

将用户拉近组，用户就拥有了组所拥有的特殊权限。将用户拉进OU，就能对用户下发组策略统一管理。

因此，组是管理的集合，OU是被管理的集合

### OU委派

考虑这样一种需求，如果我们想允许某个用户把其他用户拉近OU，而不赋予这个用户域管权限，我们可以在这个OU给这个用户委派 添加成员的权限。组织单位的委派其实就是赋予某个域内用户对OU的某些管理权限。这些权限体现在ACL里面。

![image-20231120215216507](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120215216507.png)

### 查询OU

所有的OU都是`organizationalUnit`类的实例，

我们可以用`(objectClass=organizationalUnit)`或者`(objectCategory=organizationalUnit)`来过滤OU。

![image-20231120215325511](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120215325511.png)

![image-20231120215430655](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120215430655.png)

查询OU里面的账户，可以指定BaseDN为OU就行

![image-20231120215636511](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/image-20231120215636511.png)
