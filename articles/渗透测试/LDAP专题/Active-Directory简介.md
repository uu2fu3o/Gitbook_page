这个专题主要是对域中ldap这个协议和AD域稍微再深入一些，依旧是基于windows protocl,最好还是翻阅翻阅官方文档

## LDAP简介

轻量级目录访问协议 （LDAP） 是一种用于处理各种目录服务的应用程序协议。目录服务（如 Active Directory）存储用户和帐户信息，以及密码等安全信息。然后，该服务允许与网络上的其他设备共享信息。电子邮件、客户关系经理 （CRM） 和人力资源 （HR） 软件等企业应用程序可以使用 LDAP 来验证、访问和查找信息。   -----来自microsoft

**目录服务**

目录数据库由目录服务数据库和一套访问协议组成

目录服务数据库也是一种数据库，对比于常见的关系型数据库(Mysql登)有以下几个特点

+ 呈树状结构，类似于文件目录
+ 为了查询，浏览和搜索而优化的数据库，读性能强，写性能差，不支持事务处理，会滚等复杂操作

目录结构如下树		------图来源于windows protocol

![1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/2021meiya/1.png)

为了能够访问目录数据库，必须设计一台能够访问目录服务数据库的协议，LDAP是其中一种实现协议。

LDAP之所以被称为轻量目录访问协议，事实上在LDAP使用的是另一种协议 X.500 DAP 协议规范，该协议十分复杂，是一个重量级协议，简化后就得到了LDAP协议，即便是简化后的版本，LDAP依然复杂。

> 介绍一些基本概念
>
> 1.目录树：在一个目录服务系统种，整个目录信息集可以表示为一个目录信息树，树中的每一个节点就是一个条目
>
> 2.条目： 每个条目就是一条记录，每个条目有自己唯一可区别名称（DN）。比如图中的每个圆圈都是一条记录。
>
> 3.DN&RDN： 比如第一个叶子条目,他的唯一可区别名称(DN)为: "uid=bob,ou=people,dc=acme,dc=org",与文件目录的绝对路径非常相似，同样的他的RDN=“uid=bob”
>
> 4.属性： 描述条目具体信息。比如"uid=bill,ou=people,dc=acme,dc=org"，他的属性name=bill,属性age=111,school为xx
>
> ![shuxing](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/shuxing.png)

## Active Directory简介

不同的厂商对AD有不同的实现方式，主要是了解Windows下的AD

![adways](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/adways.png)

Active Directory存储着整个域内所有的计算机，用户等的所有信息。

+ 访问Active Directory

  1.域内的每一台域控都有一份完整的本域AD,通过连接域控的389/636端口(636为LDAPS)来进行连接查看修改

  2.如果用户知道某个对象处于哪个域，也知道对象的标识名，那么通过上面第一种方式搜索对象就非常容易。但是考虑到这种情况，不知道对象所处的域，我们不得不去域林中的每个域搜索。为了解决这个问题，微软提出全局编录服务器(GC，Global Catalog)，  全局编录服务器中除了保存本域中所有对象的所有属性外，还保存林中其它域所有对象的部分属性，这样就允许用户通过全局编录信息搜索林中所有域中对象的信息。也就是说如果需要在整个林中进行搜索，而不单单是在具体的某个域进行搜索的时候，可以连接域控的3268/3269端口。

  ![GC](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/GC.png)

## Naming Context和Application Partitions

### Naming Context

Active Directory具有分布式特性，一个域林中有若干个域，每个域内有若干域控，每个域控必须一个独立的Active Directory.对此，有必要将数据隔离到多个分区中，如果不隔离，则每个域控都必须复制域林里面的所有数据。分区之后就可能够选择性的复制。

这个分区我们称为Naming Context(简称NC),每个NC都有自己的安全边界。

Active Directory预定义了三个NC

> Configuration NC(Configuration NC)
>
> Schema NC(Schema NC)
>
> Domain NC(DomainName NC)

使用ADExplorer连接就能看到

![AD1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD1.png)

后面两个是引用程序分区。

接下里分别介绍这三个分区Naming Context

+ Configuration NC(Configuration NC)

  根据微软官方的说法配置分区包含复制拓扑和其他配置数据，这些数据必须在整个林中复制。林中的每个域控制器都具有同一配置分区的副本。

  意思是讲这个分区是林配置信息的主要储存库。包含了有关站点，服务，分区和ADSchema的信息，并且每个域控上都有相同的一份配置NC。

  配置NC的根位于配置容器中，该容器是林根域的子容器。例如，`test.local`林将为`CN=Configuration,DC=test,DC=local`

  看下这个NC的顶级容器

  ![AD2](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/AD2.png)

  | RDN                              | 说明                                                         |
  | -------------------------------- | ------------------------------------------------------------ |
  | CN=DisplaySpecifiers             | 定义了Active Directory管理单元的各种显示格式                 |
  | CN=Extended-Rights               | 扩展权限对象的容器，我们将在域内ACL那篇文章里面详解          |
  | CN=ForestUpdates                 | 包含用于表示森林状态和与域功能级别更改的对象                 |
  | CN=Partitions                    | 包含每个Naming Context，Application Partitions以及外部LDAP目录引用的对象 |
  | CN=Physical Locations            | 包含位置对象，可以将其与其他对象关联 以表示该对象的位置。    |
  | CN=Services                      | 存储有关服务的配置信息，比如文件复制服务                     |
  | CN=Sites                         | 包含所有站点拓扑和复制对象                                   |
  | CN=WellKnown Security Principals | 包含常用的外部安全性主题的对象，比如Anonymous，Authenticated Users，Everyone等等 |

+ Schema NC(Schema NC)

  架构分区包含 classSchema 和 attributeSchema 对象，这些对象定义林中可以存在的对象类型。林中的每个域控制器都具有同一架构分区的副本。

+ Domain NC(DomainName NC)

  域分区包含与本地域关联的目录对象，例如用户和计算机。一个域可以有多个域控制器，一个林可以有多个域。每个域控制器存储其本地域的域分区的完整副本，但不存储其他域的域分区的副本。

  域NC的根由域的专有名称(DN)表示，比如DC.hack.com 就表示为 dc=DC,dc=hack,dc=com.域内的所有计算机，所有用户的具体信息都储存在AD下，也就是储存在AD的域NC里面。我们查看时也是选用这个NC。

  来看下域NC的顶级容器

  ![AD3](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/AD3.png)

  | RDN                          | 说明                                                     |
  | ---------------------------- | -------------------------------------------------------- |
  | CN=Builtin                   | 内置本地安全组的容器，包括管理员，域用户和账号操作员等等 |
  | CN=Computers                 | 机器用户的容器，包括加入域的所有机器                     |
  | OU=Domain Controllers        | 域控制器的容器，包括域内所有域控                         |
  | CN=ForeignSecurityPrincipals | 代表域中来自森林外部域的组中的成员                       |
  | CN=Keys                      | Server 2016之后才有，关键凭证对象的默认容器              |
  | CN=Managed Service Accounts  | 托管服务帐户的容器。                                     |
  | CN=System                    | 各种预配置对象的容器。包括信任对象，DNS对象和组策略对象  |
  | CN=TPM Devices               | 可信平台模块(TPM)密钥的恢复信息的容器。                  |
  | CN=Users                     | 用户和组对象的默认容器                                   |

### Application Partitions

win server开始，微软允许用户自定义分区来扩展Naming Context的概念。Application Partitions就是NC的一个扩展。本质上仍然属于NC。使用该功能，管理员可以自己创建分区(后称为区域)，用来将数据存储在他们选择的特定域控制器上，Application Partitions主要有以下特点：

+ Naming Context是微软预定义的，用户不可以定义自己的Naming Context。而如果用户想要定义一个分区，可以通过Application Partitions。虽然微软也预置了两个Application Partitions，但是Application Partitions的设计更多是为了让用户可以自定义自己的数据。设计Application Partitions最大的用途就是，让用户自己来定义分区。

  ![AD4](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldap/AD4.png)

+ Application Partitions可以存储动态对象。动态对象是具有生存时间(TTL) 值的对象，该值确定它们在被Active Directory自动删除之前将存在多长时间。也就说Application Partitions可以给数据设置个TTL，时间一到，Active Directory就删除该数据。

通过ntdsutil创建Application Partitions

![AD6](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD6.png)

添加成功。

或许你会想知道该扩展的实际意义

> Application Partitions在Windows域环境中允许将特定应用程序或服务的目录数据分离出来，并将其复制到其他域控制器，从而实现分布式应用程序、提高性能和可用性，并满足特定的业务需求。这种机制提供了更灵活和可定制的方式来管理和复制目录数据。

## Schema NC

这条NC内包含了架构信息，定义了AD中的类和属性，所以会先介绍LDAP中的类和继承

### LDAP中的类和继承

+ 类和实例

  域内每一个条目都是类的实例。类是一组属性的集合

  例如现有域内机器PC "CN=PC,CN=Computer,DC=hack,DC=com"在AD里面是一个条目，里面有众多属性描述条目具体信息

  ![AD7](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD7.png)

  或许你会好奇，values中并不止Computer一个值，为什么只说该条目是Computer类的实例化。这些值用于描述对象的多个方面和功能。条目可以同时是多个类的实例，这称为多继承。
  
  类是可继承的。子类继承父类的所有属性，Top类是所有类的父类。这也就是解释为什么我们看到的values并非只有computer.objectClass保存了类继承关系。`user`是`organizationPerson`的子类，`organizationPerson`是`person`的子类，`person`是`top`的子类。
  
  ![AD8](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD8.png)
  
  + 类的分类
  
    类分为三种类型
  
    + 结构类（Structural）
  
      结构类规定了对象实例的基本属性，每个条目属于且仅属于一个结构型对象类。前面说过域内每个条目都是类的实例，这个类必须是结构类。只有结构类才有实例。例如Computer
  
    + 抽象类(Abstract)
  
      抽象类型是结构类或其他抽象类的父类，它将对象属性中公共的部分组织在一起。跟面对对象里面的抽象方法一样，他没有实例，只能充当结构类或者抽象类的父类。比如说top 类。注意抽象类只能从另一个抽象类继承
  
    + 辅助类(Auxiliary）
  
      辅助类型规定了对象实体的扩展属性。虽然每个条目只属于一个结构型对象类，但可以同时属于多个辅助型对象类。注意辅助类不能从结构类继承

### Schema NC中的类

通过ADExplorer来查看Schema NC中的类。

之前说过域的每一条目都是一个结构类的实例化。而在Schema NC中，每一个条目都是一个类，例如之前说到过的Computer.

在domain中，条目"CN=PC,CN=Computer,DC=hack,DC=com"是computer的实例化，Computer类在Schema NC中作为条目

"CN=Computer,CN=Schema,CN=Configuration,DC=hack,DC=com"存在

![AD9](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD9.png)

+ 每个条目都是一个类的实例，这对Schema NC中的类条目同样适用，Computer类是classSchema。(`CN=Class-Schema,CN=Schema,CN=Configuration,DC=test,DC=local`)。所有的类条目都是classSchema类的实例

  ![AD10](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD10.png)

+ 名称是Computer(通过adminDescription，adminDisplayName，cn，name属性)

+ 属性defaultSecurityDescriptor表明，当创建Computer类的实例是，如果没有指定ACL，就已该属性的值作为默认ACL。

  ![AD11](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD11.png)

  注意与属性nTSecurityDescriptor区分，nTSecurityDescriptor是这个条目的ACL，而defaultSecurityDescriptor是实例默认的ACL。

  当我们创建Computer类的实例，Computer类有属性nTSecurityDescriptor和属性defaultSecurityDescriptor。如果我们不指定实例的ACL，那么创建实例的ACL即Computer类的属性defaultSecurityDescriptor的值。

+ 属性rDNAttID表明通过LDAP连接到类的实例的时候，使用的两个字母的前缀用过是cn。所以实例CN=PC,CN=Computers,DC=hack,DC=com使用的前缀是CN。来看另外一条类条目OU=Domain Controllers,DC=com,DC=com。使用的前缀是OU，它是类`organizationalUnit`的实例。来看该类。

  ![AD12](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD12.png)

​		该类对应的该属性为ou,因此该类的实例前缀为ou.

+ 属性objectClassCategory定义了类的类型

  为1代表结构类

  2为抽象类

  3为辅助类

+ 属性subClassOf表示了该条目的父类

  ![AD13](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD13.png)

  例如这里的Computer类的父类为user

+ systemPossSuperior约束了他的实例只能创建在这三个类

  container`,`organizationalUnit`,`domainDNS的实例底下。

  ![AD14](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD14.png)

  例如：Computer类的一个实例"CN=PC,CN=Computers,DC=hack,DC=com",位于条目“CN=Computers,DC=hack,DC=com”下

  通过查看该条目的objectClass能发现，它是container的实例，而container位于上面提到的属性底下，显然成立。

  ![AD15](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD15.png)

+ 最后一点也是最核心的，我们来讲下他的实例是怎么获取到基本属性的。

  + 这个类没有属性`systemMustContain`和`MustContain`，这两个属性定义了强制属性

  + 该类包含systemMayContain`和`MayContain，为可选属性

    ![AD16](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD16.png)

    ![AD17](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD17.png)

    上面四个属性里面的属性集合是这个类独有的属性集合，由于类是可继承的。因此，一个类的属性集合里面出了前面四个属性的值，还有可能来自父类和辅助类。

    + 辅助类可通过systemAuxiliaryClass查看，显然Computer没有辅助类

    + 通过subClass查看父类，父类为user,通过查看user的辅助类和父类，如此递归直到top

      ![AD18](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD18.png)

      ![AD19](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD19.png)

      用Active DirectorySchema 查看，能看到属性的类型是可选或强制，以及是从哪个源类继承来的

### Schema NC中的属性

每个属性都是一个条目，是类`attributeSchema`的实例

在域内的所有属性必须在这里定义，而这里的条目，最主要的是限定了属性的语法定义。其实就是数据类型，比如 Boolean类型，Integer类型等。

以条目CN=System-Flags,CN=Schema,CN=Configuration,DC=hack,DC=com为例

他的`attributeSyntax`是2.5.5.9

![AD20](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD20.png)

属性System-Flags是类attributeSchema的实例

![AD21](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD21.png)

![attributeSyntax](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/attributeSyntax.png)

## 搜索Active Directory

基础的操作，查询目录搜索要求的数据。查询目录需要指定两个元素。

+ BaseDN

+ 过滤规则

简单介绍语法问题

### Base DN

![AD22](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/AD22.png)

Base DN指定了树的根，例如这里的"DC= hack,DC=com"就是以DC=hack,DC=com为根向下搜索

### 过滤规则

LDAP 搜索过滤器语法有以下子集：

- 用与号 (&) 表示的 AND 运算符。
- 用竖线 (|) 表示的 OR 运算符。
- 用感叹号 (!) 表示的 NOT 运算符。
- 用名称和值表达式的等号 (=) 表示的相等比较。
- 用名称和值表达式中值的开头或结尾处的星号 (*) 表示的通配符。

下面举几个例子

- (uid=testuser)

  匹配 uid 属性为testuser的所有对象

- (uid=test*)

  匹配 uid 属性以test开头的所有对象

- (!(uid=test*))

  匹配 uid 属性不以test开头的所有对象

- (&(department=1234)(city=Paris))

  匹配 department 属性为1234且city属性为Paris的所有对象

- (|(department=1234)(department=56*))

  匹配 department 属性的值刚好为1234或者以56开头的所有对象。

一个需要注意的点就是运算符是放在前面的，跟我们之前常规思维的放在中间不一样

## 参考链接

[Windows Protocol](https://daiker.gitbook.io/windows-protocol/ldap-pian/8#0x06-sou-suo-active-directory)

https://learn.microsoft.com/en-us/windows/win32/ad/active-directory-domain-services
