---
title: 7+弱口令后台getshell
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-27T17:24:29+08:00
categories: 
- 渗透测试
- 漏洞复现
- Tomcat
---

## Tomcat7+ 弱口令 && 后台getshell漏洞

**漏洞影响版本：**

apache tomcat 7+

**漏洞产生原因：**

Tomcat支持在后台部署war文件，可以直接将webshell部署到web目录下。其中，欲访问后台，需要对应用户有相应权限。

Tomcat7+权限分为：

- manager（后台管理）
  - manager-gui 拥有html页面权限
  - manager-status 拥有查看status的权限
  - manager-script(管理器脚本) 拥有text接口的权限，和status权限
  - manager-jmx 拥有jmx权限，和status权限
- host-manager（虚拟主机管理）
  - admin-gui 拥有html页面权限
  - admin-script 拥有text接口权限

根据官方的解释，当一个用户拥有gui权限时，对于script和jmx的权限给予需要谨慎，当用户知道能够调用什么命令时，能够执行很多事情，当我们拥有manager-gui权限时，我们就可以通过访问/manager/html来进行弱密码测试进入后台，当拥有其他权限时上传war包getshell

权限管理的详细介绍：https://tomcat.apache.org/tomcat-8.5-doc/manager-howto.html#Configuring_Manager_Application_Access

```xml
<tomcat-users xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd" version="1.0">
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<user username="tomcat" password="tomcat" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>
</tomcat-users>
```

根据服务器下用户的配置文件，可以看到该用户具有所有权限，并且密码为tomcat,利用该用户就能够getshell,需要注意的是,tomcat8默认是没有用户的，需要管理用手动修改$CATALINA_BASE/conf/tomcat-users.xml文件来添加用户和权限

**为什么是war包**

war包可以理解为javaweb包，jar为java包，war包包括写的代码编译成的class文件，依赖的包，配置文件，所有的网站页面，包括html，jsp等等，可以直接作为webapp项目使用，至于为什么一定是war,tomcat要求部署webapp时需要的就是war包

### 漏洞利用

找一个jsp大马，压缩为war包，直接在app处上传

![tomcat-war](E:\笔记软件\笔记\渗透测试\漏洞复现\tomcat-war.png)

成功后访问该项目中的jsp代码即可

![tomcat-war2](E:\笔记软件\笔记\渗透测试\漏洞复现\tomcat-war2.png)

可以利用该位置反弹shell

**利用msf**

模块选择exploit/multi/http/tomcat_mgr_upload

根据show options设置目标和端口号

```
set rhosts 192.168.121.140
set rport 8080
set HttpPassword tomcat 
set HttpUsername tomcat 
```

![tomcat-war3](E:\笔记软件\笔记\渗透测试\漏洞复现\tomcat-war3.png)

如图成功拿到反向shell,之后升级即可

**修复建议**

1、在系统上以低权限运行Tomcat应用程序。创建一个专门的 Tomcat服务用户，该用户只能拥有一组最小权限（例如不允许远程登录）

2、增加对于本地和基于证书的身份验证，部署账户锁定机制（对于集中式认证，目录服务也要做相应配置）。在CATALINA_HOME/conf/web.xml文件设置锁定机制和时间超时限制。

3、以及针对manager-gui/manager-status/manager-script等目录页面设置最小权限访问限制。