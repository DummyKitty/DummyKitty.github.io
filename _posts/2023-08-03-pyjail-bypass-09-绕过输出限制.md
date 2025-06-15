---
title: pyjail bypass-09 绕过输出限制
date: 2023-08-03 10:23:57
last_modified_at: 2025-02-25
categories:
- Python
tags:
- pyjail
mermaid: true
image:
  path: /assets/images/pyjail.jpg
toc: true
---
> - [pyjail 沙箱逃逸原理](/posts/python-沙箱逃逸原理/)
> - [绕过删除模块或方法](/posts/pyjail-bypass-01-绕过删除模块或方法/)
- [绕过基于字符串匹配的过滤](/posts/pyjail-bypass-02-字符串变换绕过/)
- [绕过命名空间限制](/posts/pyjail-bypass-03-绕过命名空间限制/)
- [绕过长度限制](/posts/pyjail-bypass-04-绕过多行限制/)
- [绕过多行限制](/posts/pyjail-bypass-05-绕过长度限制/)
- [变量覆盖与函数篡改](/posts/pyjail-bypass-06-变量覆盖与函数篡改/)
- [绕过 audit hook](/posts/pyjail-bypass-07-绕过-audit-hook/)
- [绕过 AST 沙箱](/posts/pyjail-bypass-08-绕过-AST-沙箱/)
- [绕过输出限制](/posts/pyjail-bypass-09-绕过输出限制/)
- [绕过 opcode 沙箱](/posts/pyjail-bypass-10-绕过-opcode-沙箱/)
- [利用生成器栈](/posts/pyjail-bypass-11-利用生成器栈/)

## PyJail 没有输出的场景
在 Python 中使用 exec 函数执行代码时，默认情况下没有输出，如果想要再 exec 中打印结果，就需要在执行代码块时假如 print。

以 AmateursCTF 2023 的一道题目为例，题目的源码如下：
```py
#!/usr/local/bin/python
from flag import flag

for _ in [flag]:
    while True:
        try:
            code = ascii(input("Give code: "))
            if "flag" in code or "e" in code or "t" in code or "\\" in code:
                raise ValueError("invalid input")
            exec(eval(code))
        except Exception as err:
            print(err)
```

在这道题中，首先通过 ascii 将输入进行转化，使用 ascii 后，即使 unicode，也会被转化为 \\u00xx 的形式。然后判断输入中是否出现了 flag、e、t、以及 \\。这样的过滤条件基本将 unicode 绕过的方式给限制住了。过滤了 e 和 t， print、help 等输出函数也会被过滤， 而题目使用 exec 来执行 python 代码，因此除了绕过过滤之外，还需要考虑如何获取输出。

注意到这道题添加了一个异常处理，如果 exec 中出现错误，则会将错误信息打印出来，借助异常处理的输出，就可以将 Python 中的一些内部变量给带出来。

## 利用异常处理
作为客户端输入，结合当前读取变量的场景，python 中可利用的一些异常大多为：
- KeyError（键错误）： 当访问字典中不存在的键时引发的错误。（用户输入的键名被应用使用）
- FileNotFoundError（文件未找到错误）： 在尝试打开不存在的文件时引发的错误。
- ValueError（值错误）： 当函数接收到正确类型的参数，但参数值不合适时引发的错误。

这道题中 _ 与 flag 的值一致，因此我们只需要获取变量 _ 就可以获取 flag。

### KeyError
KeyError 出现在访问字典中不存在的键，利用时，可以随便构造一个字典，然后以需要读取的变量作为键名传进去。比如在这道题中输入：
```py
Give code: {"1":"2"}[_]
'flag{xxxx}'
```

### FileNotFoundError
FileNotFoundError 出现在找不到指定文件时，将需要读取的变量名传入文件操作函数就可以触发异常。例如 file(python2)、open 等。

但由于题目过滤了 e，这些函数都无法使用，如果需要测试的话可以将过滤的语句删除掉。
```py
Give code: open(_)
[Errno 2] No such file or directory: 'flag{xxxx}'
```

### ValueError
ValueError 比较好利用，只需要将需要读取的变量，传入一个函数，该函数的参数类型与这个要读取的变量不一致即可，例如：

```py
Give code: int(_)
ValueError: invalid literal for int() with base 10: 'flag{xxxx}'
```

当然这里过滤了 t，int 函数无法使用，可以去寻找一些别的函数。
### Popen.returncode
在 aliyunCTF 2025 ezoj 这道题中，给出了一个使用 subprocess.Popen 执行 python 脚本，但无回显的情况，
```py
        process = subprocess.Popen(
            ["python3", code_filename],
            stdin=infile,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
        )

        try:
            stdout, stderr = process.communicate(timeout=5)
        except subprocess.TimeoutExpired:
            process.kill()
            raise OJTimeLimitExceed

        if process.returncode != 0:
            raise OJRuntimeError(process.returncode)
```
但是题目给了一个 OJRuntimeError，并且传入了 returncode 属性。returncode 属性用于保存子进程退出时返回的退出码，反映了子进程是在正常结束还是在运行过程中出现异常。

这道题会将错误码发送给客户端：
```py
    except OJRuntimeError as e:
        return {"status": "RE", "message": f"Runtime Error: ret={e.args[0]}"}
```

returncode 的可能取值有以下几种
- None：表示子进程尚未终止，此时 returncode 还没有被赋值。
- 0：表示子进程成功结束，没有发生错误。
- 正整数：表示子进程执行时出现了错误，返回码通常会反映错误类型或状态码。
- 负整数：（仅在 POSIX 系统中）表示子进程被某个信号强制终止，其数值通常为 -N，其中 N 是引起终止的信号编号。

由于 ascii 也是 0-255，借助这个 returncode 就可以实现回显，但 returncode 仅有一位，所以需要逐位回显。

```py
import sys
...

content_len = len(content)
if {loc} < content_len:
    sys.exit(content[{loc}])
else:
    sys.exit(255)
```

## 参考
- [ctf-writeup/AmateursCTF 2023/Censorship at main · daffainfo/ctf-writeup](https://github.com/daffainfo/ctf-writeup/tree/main/AmateursCTF%202023/Censorship)