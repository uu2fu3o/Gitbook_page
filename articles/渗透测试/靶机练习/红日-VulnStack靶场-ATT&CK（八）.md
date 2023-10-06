## 环境搭建

给web添加一张通向外面的网卡，推荐使用vmnet0桥接，一层网络一层网络地搭建

![net](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/net.jpg)

## 第一层网络

+ 获取ip，进行扫描

```
web : 192.168.0.101
```

使用nmap进行端口扫描，一些比较重要的内容如下

![web-nmap](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/web-nmap.png)

访问http服务的位置，打开页面只能发现一张图片，没有什么有价值的信息，继续向下，访问4848端口，发现是一个后台的登录界面，采用Oracle的框架，用AWVS进行扫描，图方便，我把其他几个http服务的端口一起加到了扫描目标里面，除了4848其他几个都是connect error

![classfish](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/classfish.png)

前前后后就这几个高危漏洞，选第一个来看看

官方给的漏洞库：

https://github.com/rapid7/metasploitable3/wiki/Vulnerabilities

使用默认密码admin:sploit可以成功登录后台

![fishbackdoor](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/fishbackdoor.png)

找找看有没有可以利用的地方，glassfish的后台存在一个上传war包getshell的地方，将大马打包，配置到目标应用下，访问即可

访问格式http://ip:8181/name/木马

![大马](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/%E5%A4%A7%E9%A9%AC.png)

用cs生成后门文件，并上传到目标机器，执行，即可上线![web1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/web1.png)

+ 内网信息收集，存活主机扫描

拿到第一台机器的shell之后，确认不在云上容器，就可以开始信息收集了，先提权，由于没什么补丁，直接内核就能提权成功，开始信息收集。我们当前拿到的两个shell用户都不在域内，犹豫是2008版，还能抓到明文密码，用mimikatz来dump一下，看看能不能抓到明文密码

![user-password](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/prive-long-long/user-password.png)

抓到除了vagrant等几个用户的密码
