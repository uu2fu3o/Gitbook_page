## 写在前面

这个专题是基于exchange2016+winserver2016搭建的，所以后面的介绍包括攻击手法都基于这个

## 快速上手

**exchange体系结构**

![72f56401-0a52-43d0-9d3d-03e84f2f93ba](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/ldapp/72f56401-0a52-43d0-9d3d-03e84f2f93ba.png)

**邮件服务器**

- 邮箱服务器包含用于路由邮件的传输服务。
- 邮箱服务器包含处理、呈现和存储数据的邮箱数据库。
- 邮箱服务器包含客户端访问服务，这些服务接受所有协议的客户端连接。 这些前端服务负责将连接路由或*代理*到邮箱服务器上的相应后端服务。 客户端不直接连接到后端服务。
- 在 Exchange 2016 中，邮箱服务器包含向邮箱提供语音邮件和其他电话服务功能的统一消息 (UM) 服务。

**客户端访问服务器**

负责认证、重定向、代理来自外部不同客户端的访问请求，主要包含客户端访问服务（Client Access service）和前端传输服务（Front End Transport service）两大组件。

**边缘传输服务器**

- 边缘传输服务器处理 Exchange 组织的所有外部邮件流。
- 边缘传输服务器通常安装在外围网络中，并订阅到内部 Exchange 组织。 当 Exchange 组织接收和发送邮件时，EdgeSync 同步进程会向边缘传输服务器提供收件人信息和其他配置信息。
- 当 Exchange 组织接收和发送邮件时，边缘传输服务器会提供反垃圾邮件规则和邮件流规则。
- 您可以使用 Exchange 命令行管理程序 来管理边缘传输服务器。

**接口和协议**

+ owa

  owa即 outlook web app,即outlook的网页版。（outlook是exchange的客户端软件，许多电脑都有所预装）
  地址一般为 http://域名/owa

+ ecp

  `Exchange` 管理中心，管理员用于管理组织中的`Exchange`的`Web`控制台

+ EWS

  `Exchange Web Service` ,实现客户端与服务端之间基于`HTTP`的`SOAP`交互

+ OAB

  用于为 `Outlook` 客户端提供地址簿的副本，减轻`Exchange`的负担

+ Rpc

  早期的`Outlook`还使用称为 `Outlook` `Anywhere` 的 `RPC` 交互

+ mapi

  `Outlook`连接 `Exchange` 的默认方式，在`2013`和`2013`之后开始使用，`2010 sp2`同样支持

+ Autodiscover

  自 `Exchange Server 2007` 开始推出的一项自动服务，用于自动配置用户在 `Outlook` 中邮箱的相关设置，简化用户登录使用邮箱的流程

+ PowerShell

  用于服务器管理的 `Exchange` 管理控制台

+ Microsoft-Server-ActiveSync

  用于移动应用程序访问电子邮件

+ GAl

  GAL即全局地址表（global address list）

  记录了域中用户的基本信息与其邮箱地址，以形成域用户与邮箱用户之间的关联。
  在渗透中可以通过GAL来获取所有邮箱地址。