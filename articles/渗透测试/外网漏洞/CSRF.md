---
title: CSRF
date: 2023-09-03T15:29:52+08:00
updated: 2023-06-03T17:22:12+08:00
categories: 
- 渗透测试
- 外网漏洞
---

# Cross-site request forgery

## 前言

在XSS学习中，已经简单介绍过CSRF的原理以及简单利用，本篇主要讲述如何检测和防御CSRF攻击

## CSRF检测

### http referer 

通过检测数据包中的referer字段判断是否有CSRF漏洞

具体方法可以直接置空referer字段进行发包，观察是否正常发包并且回应；

通常referer字段会被检测，我们可以伪造域看是否能够绕过检测

```
eg:
检测内容为uu2fu3o.com
我们可以修改为uu2fu3o.com.test.com or uu2fu3o.com/test.com，通过伪造域名来绕过
```

## token

如果一个数据包没有referer也没有token通常存在csrf漏洞，token以参数的形式加入到http请求包中，用户在登陆后获得一个随机的token并且添加到session中，以后的每次请求都会从session中取出该token作为参数添加，服务端验证token是否为用户请求；

同样的，在数据包中，我们可以置空token或者是删除该参数，观察是否有效

## CSRF绕过

### 更改请求方法

服务端有可能会忘记对请求方法做限制，如果能够不限制请求方法访问，通常具有csrf漏洞

例如一个post提交的请求包

```
POST /change_passwd Http
.......
newpasswd=newpasswd
```

我们可以把他改为GET方法进行提交

```
GET /change_passwd?newpasswd=newpasswd Http
```

如果能够请求成功，即使设置了token也不一定能防御csrf

### 对token进行测试

当有token时我们可以采取以下的方式验证token是否有效

置空token或是删除token参数

```
POST /change_passwd Http
....
newpasswd=newpasswd&token=
或是
newpasswd=newpasswd
```

如果验证内容只是token是否存在于服务端，我们可以尝试使用自己的token

```
POST /change_passwd Http
....
newpasswd=newpasswd&token=ur_token
```

session固定

当一个站点使用双token的形式进行验证时，通常不会验证token的真实性，只是验证token是否相同，我们可以采取session固定对受害者进行攻击，这样受害者就会使用我们伪造的假token进行请求，从而被我们劫持

此时执行

```
POST /change_passwd 
....
Cookie: token=fake_token
....
newpasswd=newpasswd&token=fake_token
```

### 对referer字段进行测试

直接置空或者是移除referer字段进行发包验证

```
<meta name =“referrer”content =“no-referrer”>
```

将该内容添加到漏洞页面进行测试

使用二级域名绕过检测

## 防御CSRF

验证referer字段是否来自规定网址，并且只允许相应字段，这涉及到正则表达式的绕过

验证token，验证token的合法性，并且注意防止黑客伪造token

限制请求方法，这个会被经常忽略

在http头中自定义属性并进行验证

直接使用具有csrf防御功能的框架

### 参考链接

https://pdai.tech/md/develop/security/dev-security-x-csrf.html#csrf-%E5%A6%82%E4%BD%95%E6%94%BB%E5%87%BB

https://www.freebuf.com/articles/web/254501.html

https://xz.aliyun.com/t/6176

https://www.cnblogs.com/echojson/p/12805102.html  >>会话固定攻击流程及原理