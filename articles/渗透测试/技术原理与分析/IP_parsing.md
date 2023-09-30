---
title: IP_parsing
date: 2023-09-03T15:29:52+08:00
updated: 2023-06-30T00:19:13+08:00
categories: 
- 渗透测试
- 技术原理与分析
---

本篇文章是在研究SSRF绕过时，偶然看到的视频，有点感兴趣，所以稍微分析了一下不同环境对于IPv4解析的问题

## Go

GO语言解析IP通常是利用net中的IP解析函数，不同的函数对应着不同的解析方式，这里给出两个函数

```go
==>  ParseIP  <==
import (
	"fmt"
	"net"
	"os"
)
func main() {
	if len(os.Args) != 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s ip.addr\n", os.Args[0])
		os.Exit(1)
	}
	addr := os.Args[1]
	ip := net.ParseIP(addr)
	if ip != nil {
		fmt.Println("The address is", ip.String())
	} else {
		fmt.Println("Invalid address")
	}
	os.Exit(0)
}
//ParseIP函数能够解析正确的10进制IPv4和IPv6地址，以下为函数定义和测试
func ParseIP(s string) IP {
	for i := 0; i < len(s); i++ {
		switch s[i] {
		case '.':
			return parseIPv4(s)
		case ':':
			return parseIPv6(s)
		}
	}
	return nil
}
```



```go
==>IPv4<==
import (
	"fmt"
	"net"
)
func main() {
	octalIP := net.IPv4(0177, 0, 0, 1)
	fmt.Println("IP address:", octalIP.String())
}
//我们得到IP address: 127.0.0.1，证明IPv4函数能够解析8进制编码的IPaddress
```

如果我们想要解析url该怎么解决

```go
import (
    "fmt"
    "net/url"
)

func main() {
    urlString := "http://0177.0.0.1"
    parsedUrl, err := url.Parse(urlString)
    if err != nil {
        panic(err)
    }

    fmt.Println("Scheme:", parsedUrl.Scheme)
    fmt.Println("Host:", parsedUrl.Host)
    fmt.Println("Path:", parsedUrl.Path)
    fmt.Println("Raw Query:", parsedUrl.RawQuery)
    fmt.Println("Fragment:", parsedUrl.Fragment)
}
//得到Host:0177.0.0.1 ;显然，通过url.Parse函数无法直接进行8进制编码的解析
```

## Python

python中并没有能够直接解析8进制IP地址的函数，需要将8进制转换为10进制，再通过tp_bytes()转换为2进制表示法，再使用inet-ntoa()转换为点分十进制表示法

```python
import socket

ip_octal = '0177.0000.00.01'

# Parse IP address
ip_dec = int(ip_octal.replace('.', ''), 8)
ip_bin = ip_dec.to_bytes(4, byteorder='big')
ip_str = socket.inet_ntoa(ip_bin)

# Print results
print('Octal:', ip_octal)
print('Decimal:', ip_dec)
print('Binary:', ' '.join(format(x, '08b') for x in ip_bin))
print('IP address:', ip_str)
#return 127.0.0.1
```

并且python在处理整数时会将前导零忽略

python中处理url,使用urlparse时会直接将hostname解析为字符串，貌似并没有解析8进制IP;经过部分测试，发现Python如果要解释8进制位IP地址，总是需要先进行转换

## Java

```java
import java.net.InetAddress;
import java.net.UnknownHostException;

public class IPParser {

    public static void main(String[] args) {
        String ipAddress = "0177.0.0.1";
        parseIPAddress(ipAddress);
    }

    public static void parseIPAddress(String ipAddress) {
        try {
            // Parse IP address
            InetAddress inetAddress = InetAddress.getByName(ipAddress);

            // Print results
            System.out.println("IP address: " + inetAddress.getHostAddress());
            System.out.println("Hostname: " + inetAddress.getHostName());
            System.out.println("Canonical hostname: " + inetAddress.getCanonicalHostName());
            System.out.println("Is reachable: " + inetAddress.isReachable(5000));
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

===>

```java
IP address: 127.0.0.1
Hostname: localhost
Canonical hostname: localhost
Is reachable: true
```

我们得到了想要的结果，也就是解析了8进制位IP，16进制又如何呢

```
0x7F000001
```

上述程序并没有实现16进制IP的解析，我们在本地是能够ping通的

具体来讲，Java解析IP具有如下步骤

1. 通过调用 `InetAddress.getByName()` 方法，并将IP地址字符串或主机名作为参数传递，返回一个 `InetAddress` 对象。
2. 如果传递的参数是IP地址字符串，Java会首先检查该字符串是否符合IPv4或IPv6地址的格式。如果是IPv4地址，Java会将其转换为32位整数表示，然后将其存储在 `InetAddress` 对象中。如果是IPv6地址，Java会将其转换为128位整数表示，然后将其存储在 `InetAddress` 对象中。
3. 如果传递的参数是主机名，Java会尝试将该主机名解析为一个或多个IP地址。Java会首先查找本地DNS解析器缓存中是否有该主机名的IP地址。如果缓存中没有该主机名的IP地址，Java会将该主机名发送到DNS服务器进行解析，并返回一个或多个IP地址，然后将其存储在 `InetAddress` 对象中。
4. 通过调用 `InetAddress` 类的实例方法，可以获取IP地址、主机名、规范主机名、主机地址等信息

注意上述的第二点，会首先将IPv4转换为32位整数的形式，我们自然可以直接提供32位无符号整数，使其直接进行判断

```
127.0.0.1 ==> 2130706433
```

得到

```shell
IP address: 127.0.0.1
Hostname: 127.0.0.1
Canonical hostname: 127.0.0.1
Is reachable: true
```

## PHP

```php
<?php
$ip_address = '0177.0.0.1';
$binary_ip = inet_pton($ip_address);
echo bin2hex($binary_ip); 
echo "\r\n";
//127.0.0.1 ==>7f000001
$ip_addr=inet_ntop($binary_ip);
echo($ip_addr)

?>
```

显然inet_pton无法转化8进制IP，inet_ntop也抛弃了这个选项

```php
<?php
$ip = '0177.0.0.1';
$long = ip2long($ip);

// 输出转换前的IP地址和long整数
echo "转换前的IP地址：$ip\r\n";
echo "转换前的long整数：$long\r\n";

// 将long整数转换为IP地址
$ip_from_long = long2ip($long);

// 输出转换后的IP地址
echo "从long整数转换回来的IP地址：$ip_from_long";
?>
```

我们能得到这样的结果

```
转换前的IP地址：0177.0.0.1
转换前的long整数：
从long整数转换回来的IP地址：0.0.0.0
```

从PHP 7.2.0开始，`ip2long()`函数不再支持将8进制IP地址作为参数传入。因此，如果使用`ip2long()`函数将8进制IP地址转换为long整数，会返回`false`。显然没有达到我们预期的效果，再来看看其他函数呢

```php
<?php
$ip = '0177.0.0.1';
if (filter_var($ip, FILTER_VALIDATE_IP)) {
    echo 'IP地址：' . $ip;
} else {
    echo '不是一个合法的IP地址。';
}
?>
```

我们能得到这不是一个合法的IP,高版本的php不再支持转换8和16进制ip

## windows ping

```shell
==>192.168.121.1<==
192 ==>0300//8进制
168 ==>0250
1 ==> 001
121 ==>0x79//此处开始16进制
```

```shell
ping 0300.0250.0x79.001
正在 Ping 192.168.121.1 具有 32 字节的数据:
来自 192.168.121.1 的回复: 字节=32 时间<1ms TTL=128
来自 192.168.121.1 的回复: 字节=32 时间<1ms TTL=128
来自 192.168.121.1 的回复: 字节=32 时间<1ms TTL=128
来自 192.168.121.1 的回复: 字节=32 时间<1ms TTL=128
```

使用ping工具，我们能够ping通8进制与16进制混合表示的ip地址,ping为什么能够这样进行解析呢？

ping命令在接收IP地址时，实际上是通过将输入的字符串解析为二进制格式的IP地址来进行处理，因此，ping命令可以接收8进制和16进制混合使用的IP地址，并将其转换为正确的点分十进制IP地址进行使用

## Some interesting

当我们在网页中输入2130706433，chrome会为我们自动解析成目标IP，也就是127.0.0.1

我们知道的是，2130706433其实是127.0.0.1的32位无符号整数表示

如果我们用8进制呢，又或者是16进制呢

```
=== Class C ===
Octal:		0177.00.00.01
Decimal:	127.0.0.1
Hexadecimal:	0x7f.0x0.0x0.0x1
=== Class B ===
Octal:		0177.00.01
Decimal:	127.0.1
Hexadecimal:	0x7f.0x0.0x1
=== Class A ===
Octal:		0177.01
Decimal:	127.1
Hexadecimal:	0x7f.0x1
=== Class (whole network) ===
Octal:		017700000001
Decimal:	2130706433
Hexadecimal:	0x7f000001
```

经过测试，无论我们采取什么样的形式去访问，http://ip这样的形式都会直接进行解析

### about class A,B,C

A,B,C指的是在引入CIDR之前对IPv4进行的早期分配方案将IP地址分为A类，B类，C类，虽然现在已经不常使用了，但是我们可以来看看

```
127.0.0.1
class C: 127.0.0.1
class B: 127.0.1
class A: 127.1
```

我们使用http://127.1类似的形式，仍然能在chrome中访问到我们的本地站点，这也正好解释了我们在SSRF绕过中能够使用像127.1这样的形式绕过限制，1其实是24位数字，127.1是127.0.0.1的A类表示法

### 关于前导零

IP地址是否允许拥有无限多个前导0?前导零可能会导致IP地址被当成8进制解析，但现在很多语言已经抛弃了8进制，16进制的解析方式，并将前导0视为10进制

```
001.002.003.004
```

这样的IP普遍认为是有效的，它与1.2.3.4在二进制编码下相同，如果是 这样的形式呢

```
0000000001.0000000002.0000000003.000000004
```

对上面这一段IP，我们在windows ping中仍然能得到1.2.3.4的正确结果，但在python中就会导致ip过长等问题，go中的net.IPv4却还能够正确解析

### 关于ipv4映射ipv6

在ssrf的绕过中，我们看到了这样的格式

```
http://[0:0:0:0:0:ffff:127.0.0.1]
http://[::ffff:127.0.0.1]
```

事实上这是一个ipv4映射ipv6的格式,这种地址格式可以用来在IPv6网络中访问IPv4地址

IPv4映射IPv6地址的格式为：`::ffff:w.x.y.z`，其中，`w.x.y.z`表示IPv4地址的四段数字，转换为16进制后就是IPv6地址的后4个字节。在这个例子中，IPv4地址127.0.0.1转换为16进制就是0x7f000001，将其对应到IPv6地址的后4个字节就是0:0:0:0:0:ffff:7f00:0001，可以缩写为`::ffff:127.0.0.1`

这样的格式不再支持对127.0.0.1进行更改

## 参考链接

https://news.ycombinator.com/item?id=25545967

https://news.ycombinator.com/item?id=25545967
