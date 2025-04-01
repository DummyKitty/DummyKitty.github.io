---
title: CTF Pyjail 沙箱逃逸原理合集
date: 2023-05-29 05:04:02
categories:
- Python
tags:
- pyjail
image:
  path: /assets/images/pyjail.jpg
toc: true
---

沙箱是一种安全机制，用于在受限制的环境中运行未信任的程序或代码。它的主要目的是防止这些程序或代码影响宿主系统或者访问非授权的数据。

在 Python 中，沙箱主要用于限制 Python 代码的能力，例如，阻止其访问文件系统、网络，或者限制其使用的系统资源。Python 沙箱的实现方式有多种，包括使用 Python 的内置功能（如re模块），使用特殊的 Python 解释器（如PyPy），或者使用第三方库（如RestrictedPython）。

然而，Python 并没有提供内建的、可靠的沙箱机制,并且 Python 的标准库和语言特性提供了很多可以用于逃逸沙箱的方法，因此在实践中创建一个完全安全的 Python 沙箱非常困难。例如，通过`os`模块访问文件系统，通过`subprocess`模块执行外部命令，或者通过`import`语句加载并执行任意 Python 代码。


## 常见沙箱

### exec 执行

Python 的 `exec()` 是一个内建函数，用来执行动态生成的 Python 代码。也就是说，`exec()` 可以执行储存在字符串或对象中的 Python 代码。这是 `exec()` 的基本语法：

```python
exec(object, globals, locals)
```

- `object` 必需参数，是一个字符串，或者是任何可以被 `compile()` 函数转化为代码对象的对象。
- `globals` 可选参数，是一个字典，表示全局命名空间 (全局变量)，如果提供了，则在执行代码中被用作全局命名空间。
- `locals` 可选参数，可以是任何映射对象，表示局部命名空间 (局部变量)，如果被提供，则在执行代码中被用作局部命名空间。如果两者都被忽略，那么在 `exec()` 调用的地方决定执行的命名空间。

exec 还有另外一种写法：
```py
exec command in _global
```
其中 `_global` 为一个字典， 表示 command 将在 `_global` 指定的命名空间中运行。

示例沙箱
```py
print(
    exec(input("code> "), 
         {"__builtins__": {}},
        {"__builtins__": {}}
        )
    )
```

示例题目：
```py
#!/usr/bin/env python2
# -*- coding:utf-8 -*-


def banner():
    print "============================================="
    print "   Simple calculator implemented by python   "
    print "============================================="
    return


def getexp():
    return raw_input(">>> ")


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


def sandbox_filter(command):
    blacklist = ['exec', 'sh', '__getitem__', '__setitem__',
                 '=', 'open', 'read', 'sys', ';', 'os']
    for forbid in blacklist:
        if forbid in command:
            return 0
    return 1


def sandbox_exec(command):      # sandbox user input
    result = 0
    __sandboxed_builtins__ = dict(__builtins__.__dict__)
    __sandboxed_builtins__['__import__'] = _hook_import_    # hook import
    del __sandboxed_builtins__['open']
    _global = {
        '__builtins__': __sandboxed_builtins__
    }
    if sandbox_filter(command) == 0:
        print 'Malicious user input detected!!!'
        exit(0)
    command = 'result = ' + command
    try:
        exec command in _global     # do calculate in a sandboxed environment
    except Exception, e:
        print e
        return 0
    result = _global['result']  # extract the result
    return result


banner()
while 1:
    command = getexp()
    print sandbox_exec(command)

```

### eval 执行
eval 的执行与 exec 基本一致，都可以对命名空间进行限制，例如下面的代码，在这个示例中就是直接将命名空间置空，这样就使得内置的函数都无法使用。
```py
print(
    eval(input("code> "), 
         {"__builtins__": {}},
        {"__builtins__": {}}
        )
    )
```
eval 和 exec 之间也有区别。

#### eval 与 exec 的区别
eval 与 exec 的区别再于 exec 允许 \n 和 ； 进行换行，而 eval 不允许。并且 exec 不会将结果输出出来，而 eval 会。
```py
exec("print('RCE'); __import__('os').system('ls')") #Using ";"
exec("print('RCE')\n__import__('os').system('ls')") #Using "\n"
eval("__import__('os').system('ls')") #Eval doesn't allow ";"
eval(compile('print("hello world"); print("heyy")', '<stdin>', 'exec')) #This way eval accept ";"
__import__('timeit').timeit("__import__('os').system('ls')",number=1)
#One liners that allow new lines and tabs
eval(compile('def myFunc():\n\ta="hello word"\n\tprint(a)\nmyFunc()', '<stdin>', 'exec'))
exec(compile('def myFunc():\n\ta="hello word"\n\tprint(a)\nmyFunc()', '<stdin>', 'exec'))
```
如果加入了 compile 函数则需要按照 compile 函数的模式进行区分。

#### compile
compile() 函数是一个内置函数，它可以将源码编译为代码或 AST 对象。编译的源码可以是普通的 Python 代码，也可以是 AST 对象。如果它是一个普通的 Python 代码，那么它必须是一个字符串。如果它是一个 AST 对象，那么它将被编译为一个代码对象。

compile() 函数的基本语法如下：
```py
compile(source, filename, mode, flags=0, dont_inherit=False, optimize=-1)
```
- source：要编译的源代码。它可以是普通的 Python 代码，或者是一个 AST 对象。如果它是普通的 Python 代码，那么它必须是一个字符串。
- filename：源代码的文件名。如果源代码没有来自文件，你可以传递一些可识别的值。
- mode：源代码的种类。可以是 'exec'，'eval' 或 'single'。'exec' 用于模块、脚本或者命令行，'eval' 用于简单的表达式，'single' 用于单一的可执行语句。
- flags 和 dont_inherit：这两个参数用于控制编译源代码时的标志和是否继承上下文。它们是可选的。
- optimize：用于指定优化级别。默认值为 -1。

mode 参数决定了如何编译输入的源代码。具体来说，它有三个可能的值： 'exec'，'eval' 和 'single'。
- 'exec'： exec 方式就类似于直接使用 exec 方法，可以处理换行符，分号，import语句等。
- 'eval'： eval 方式也就类似于直接使用 eval，只能处理简单的表达式，不支持换行、分号、import 语句
- 'single'：这个模式类似于 'exec'，但是它只用于执行单个语句(可以在语句中添加换行符等)。


### 基于 audit hook 的沙箱 
Python 3.8 中引入的一种 audit hook 的新特性。审计钩子可以用来监控和记录 Python 程序在运行时的行为，特别是那些安全敏感的行为，如文件的读写、网络通信和动态代码的执行等。

`sys.addaudithook(hook)` 的参数 `hook` 是一个函数，它的定义形式为 `hook(event: str, args: tuple)`。其中，`event` 是一个描述事件名称的字符串，`args` 是一个包含了与该事件相关的参数的元组。

一旦一个审计钩子被添加，那么在解释器运行时，每当发生一个与安全相关的事件，就会调用该审计钩子函数。`event` 参数会包含事件的描述，`args` 参数则包含了事件的相关信息。这样，审计钩子就可以根据这些信息进行审计记录或者对某些事件进行阻止。

注意，由于 `sys.addaudithook()` 主要是用于增加审计和安全性，一旦一个审计钩子被添加，它不能被移除。这是为了防止恶意代码移除审计钩子以逃避审计。

举例来说，我们可以定义这样的一个审计钩子，使得每次触发 `open` 事件都会打印一条消息：

```python
import sys

def audit_hook(event, args):
    if event == 'open':
        print(f'Opening file: {args}')

sys.addaudithook(audit_hook)
```

然后，如果运行 `open('myfile.txt')`，就会得到 `Opening file: ('myfile.txt',)` 这样的输出。

在一些沙箱的题目中也会使用这种方式，例如：
```py
    ...
def my_audit_hook(event, _):
    BALCKED_EVENTS = set({'pty.spawn', 'os.system', 'os.exec', 'os.posix_spawn','os.spawn','subprocess.Popen'})
    if event in BALCKED_EVENTS:
        raise RuntimeError('Operation banned: {}'.format(event))
    ...

sys.addaudithook(my_audit_hook)

if __name__ == '__main__':
    main()
```
这个沙箱对`'pty.spawn', 'os.system', 'os.exec', 'os.posix_spawn','os.spawn','subprocess.Popen'` 这些函数进行了限制，一旦调用则抛出异常.

#### sys.addaudithook 构建沙箱
上面的示例就是一个黑名单:
```py
    ...
def my_audit_hook(event, _):
    BALCKED_EVENTS = set({'pty.spawn', 'os.system', 'os.exec', 'os.posix_spawn','os.spawn','subprocess.Popen'})
    if event in BALCKED_EVENTS:
        raise RuntimeError('Operation banned: {}'.format(event))
    ...

sys.addaudithook(my_audit_hook)
if __name__ == '__main__':
    main()
```

白名单比黑名单的限制更大,下面的沙箱只允许 input exec compile 等函数的调用.
```py
...
def my_audit_hook(my_event, _):
    WHITED_EVENTS = set({'builtins.input', 'builtins.input/result', 'exec', 'compile'})
    if my_event not in WHITED_EVENTS:
        raise RuntimeError('Operation not permitted: {}'.format(my_event))
    ...
if __name__ == "__main__":
    sys.addaudithook(my_audit_hook)
    main()
```
一般的 payload 无法使用:
```py
> __import__('ctypes').CDLL(None).system('ls /'.encode())
Operation not permitted: import
> [ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("ls")
Operation not permitted: os.system
```
#### PySys_AddAuditHook 构建沙箱


### 基于 AST 的沙箱
Python 的抽象语法树（AST，Abstract Syntax Tree）是一种用来表示 Python 源代码的树状结构。在这个树状结构中，每个节点都代表源代码中的一种结构，如一个函数调用、一个操作符、一个变量等。Python 的 ast 模块提供了一种机制来解析 Python 源代码并生成这样的抽象语法树。
下面是Python `ast`模块的一些常见节点类型：

- `ast.Module`: 表示一个整个的模块或者脚本。
- `ast.FunctionDef`: 表示一个函数定义。
- `ast.AsyncFunctionDef`: 表示一个异步函数定义。
- `ast.ClassDef`: 表示一个类定义。
- `ast.Return`: 表示一个return语句。
- `ast.Delete`: 表示一个del语句。
- `ast.Assign`: 表示一个赋值语句。
- `ast.AugAssign`: 表示一个增量赋值语句，如`x += 1`。
- `ast.For`: 表示一个for循环。
- `ast.While`: 表示一个while循环。
- `ast.If`: 表示一个if语句。
- `ast.With`: 表示一个with语句。
- `ast.Raise`: 表示一个raise语句。
- `ast.Try`: 表示一个try/except语句。
- `ast.Import`: 表示一个import语句。
- `ast.ImportFrom`: 表示一个from...import...语句。
- `ast.Expr`: 表示一个表达式。
- `ast.Call`: 表示一个函数调用。
- `ast.Name`: 表示一个变量名。
- `ast.Attribute`: 表示一个属性引用，如`x.y`。

以上列举的只是`ast`模块中一部分的节点类型，还有很多其他类型的节点。更详细的列表可以在Python官方文档的`ast`模块部分找到。

一些 CTF 题目就采用了检查 AST 节点构建沙箱, 下面是一个示例. 在这个示例中, verify_secure 函数对 compile 之后的代码进行校验, 禁止 ast.Import|ast.ImportFrom|ast.Call 这三类操作, 这样一来就无法导入模块和执行函数.

```python
import ast
import sys
import os

def verify_secure(m):
  for x in ast.walk(m):
    match type(x):
      case (ast.Import|ast.ImportFrom|ast.Call):
        print(f"ERROR: Banned statement {x}")
        return False
  return True

abspath = os.path.abspath(__file__)
dname = os.path.dirname(abspath)
os.chdir(dname)

print("-- Please enter code (last line must contain only --END)")
source_code = ""
while True:
  line = sys.stdin.readline()
  if line.startswith("--END"):
    break
  source_code += line

tree = compile(source_code, "input.py", 'exec', flags=ast.PyCF_ONLY_AST)
if verify_secure(tree):  # Safe to execute!
  print("-- Executing safe code:")
  compiled = compile(source_code, "input.py", 'exec')
  exec(compiled)
```

### 基于 opcode 的沙箱
#### Python 字节码与操作码

Python 是一种 解释型语言，这意味着 Python 代码在执行之前会被编译器编译为一种中间表示形式，称为 字节码（Bytecode）。字节码是一种低级的、与平台无关的表示形式，它是 Python 虚拟机（Python Virtual Machine, PVM）可以直接执行的指令集。

1. 什么是字节码？
   1. 字节码 是 Python 源代码编译后的中间表示形式，它由一系列的指令组成，每条指令都有一个对应的操作码（opcode）。
   2. 字节码是 Python 虚拟机执行程序的基础。Python 虚拟机读取字节码并逐条解释和执行。
   3. 字节码是与平台无关的，这意味着同样的字节码可以在不同的系统上运行，只要有相应的 Python 解释器。

字节码是 Python 中的一种抽象层次，介于源代码和机器代码之间。它不是机器代码，因此不能直接在硬件上运行，而是由 Python 虚拟机解释执行。

2. 什么是操作码（Opcode）？
   1. 操作码（Opcode, Operation Code）==是字节码中的一条指令==，用于告诉 Python 虚拟机应该执行什么操作。
   2. 每个操作码对应一个特定的操作，例如加载常量、调用函数、加法运算等。
   3. 操作码通常由一个整数表示，并且每个操作码都有一个对应的名称。

例如，LOAD_CONST 是一个常见的操作码，它用于从常量池中加载一个常量值到栈中。

3. 字节码的工作流程
   1. 编译阶段：
      1. 当你运行 Python 脚本时，Python 首先将源代码编译为字节码。这个过程发生在后台，通常你不需要手动干预。
      2. 编译后的字节码可以存储在 .pyc 文件中（位于 `__pycache__` 目录下），以便下次运行时加快启动速度。
   2. 执行阶段：
      1. 编译后的字节码会被交给 Python 虚拟机（PVM），虚拟机会逐条解释和执行这些指令。
      2. 每条指令都对应一个具体的操作，如变量赋值、函数调用、循环跳转等。

#### 示例：从源代码到字节码
让我们通过一个简单的例子来看看如何查看 Python 源代码被编译成字节码：

```python
def add(a, b):
    return a + b
```
你可以使用 dis 模块来查看这个函数对应的字节码：

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
```
输出：
```py
  3           0 RESUME                   0

  4           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE
```
1. LOAD_FAST 0 (a)：将局部变量 a 加载到栈中。
2. LOAD_FAST 1 (b)：将局部变量 b 加载到栈中。
3. BINARY_OP：将栈顶两个元素相加，并将结果压回栈中。
4. RETURN_VALUE：从栈顶弹出值并作为函数返回值。


每条字节码指令都有一个对应的操作码。操作码是一个整数，表示特定的操作，而字节码则是这些操作组合起来形成的一系列指令。

1. 操作码（opcode） 是单个指令，例如 LOAD_FAST, BINARY_ADD, RETURN_VALUE 等等。
2. 字节码 是这些操作组合起来形成的一段程序，它包含了多个操作符以及它们可能需要的数据或参数。
#### 枚举所有操作码
我们可以通过 dis.opname 和 dis.opmap 来查看所有可用的操作码及其对应关系：

```py
import dis

# 打印所有可用的操作码名称
print(dis.opname)

# 打印某个特定操作符名称对应的 opcode 值
print(dis.opmap['LOAD_CONST'])  # 输出: 100
```
从结果上可以看到，`dis.opname` 是一个包含所有操作码（opcode）名称的列表，长度为 256。这是因为 Python 的字节码使用 1字节（8位） 来表示操作码，因此最多可以有 256 个不同的操作码（从 0 到 255）。

然而，并不是所有的操作码都被使用，某些操作码是==保留的==或==未定义的==，这些位置通常显示为 `<number>` 的形式。比如 240 到 254 是未定义或保留的操作码。

所有可用的操作码如下：
```py
{'CACHE': 0, 'POP_TOP': 1, 'PUSH_NULL': 2, 'NOP': 9, 'UNARY_POSITIVE': 10, 'UNARY_NEGATIVE': 11, 'UNARY_NOT': 12, 'UNARY_INVERT': 15, 'BINARY_SUBSCR': 25, 'GET_LEN': 30, 'MATCH_MAPPING': 31, 'MATCH_SEQUENCE': 32, 'MATCH_KEYS': 33, 'PUSH_EXC_INFO': 35, 'CHECK_EXC_MATCH': 36, 'CHECK_EG_MATCH': 37, 'WITH_EXCEPT_START': 49, 'GET_AITER': 50, 'GET_ANEXT': 51, 'BEFORE_ASYNC_WITH': 52, 'BEFORE_WITH': 53, 'END_ASYNC_FOR': 54, 'STORE_SUBSCR': 60, 'DELETE_SUBSCR': 61, 'GET_ITER': 68, 'GET_YIELD_FROM_ITER': 69, 'PRINT_EXPR': 70, 'LOAD_BUILD_CLASS': 71, 'LOAD_ASSERTION_ERROR': 74, 'RETURN_GENERATOR': 75, 'LIST_TO_TUPLE': 82, 'RETURN_VALUE': 83, 'IMPORT_STAR': 84, 'SETUP_ANNOTATIONS': 85, 'YIELD_VALUE': 86, 'ASYNC_GEN_WRAP': 87, 'PREP_RERAISE_STAR': 88, 'POP_EXCEPT': 89, 'STORE_NAME': 90, 'DELETE_NAME': 91, 'UNPACK_SEQUENCE': 92, 'FOR_ITER': 93, 'UNPACK_EX': 94, 'STORE_ATTR': 95, 'DELETE_ATTR': 96, 'STORE_GLOBAL': 97, 'DELETE_GLOBAL': 98, 'SWAP': 99, 'LOAD_CONST': 100, 'LOAD_NAME': 101, 'BUILD_TUPLE': 102, 'BUILD_LIST': 103, 'BUILD_SET': 104, 'BUILD_MAP': 105, 'LOAD_ATTR': 106, 'COMPARE_OP': 107, 'IMPORT_NAME': 108, 'IMPORT_FROM': 109, 'JUMP_FORWARD': 110, 'JUMP_IF_FALSE_OR_POP': 111, 'JUMP_IF_TRUE_OR_POP': 112, 'POP_JUMP_FORWARD_IF_FALSE': 114, 'POP_JUMP_FORWARD_IF_TRUE': 115, 'LOAD_GLOBAL': 116, 'IS_OP': 117, 'CONTAINS_OP': 118, 'RERAISE': 119, 'COPY': 120, 'BINARY_OP': 122, 'SEND': 123, 'LOAD_FAST': 124, 'STORE_FAST': 125, 'DELETE_FAST': 126, 'POP_JUMP_FORWARD_IF_NOT_NONE': 128, 'POP_JUMP_FORWARD_IF_NONE': 129, 'RAISE_VARARGS': 130, 'GET_AWAITABLE': 131, 'MAKE_FUNCTION': 132, 'BUILD_SLICE': 133, 'JUMP_BACKWARD_NO_INTERRUPT': 134, 'MAKE_CELL': 135, 'LOAD_CLOSURE': 136, 'LOAD_DEREF': 137, 'STORE_DEREF': 138, 'DELETE_DEREF': 139, 'JUMP_BACKWARD': 140, 'CALL_FUNCTION_EX': 142, 'EXTENDED_ARG': 144, 'LIST_APPEND': 145, 'SET_ADD': 146, 'MAP_ADD': 147, 'LOAD_CLASSDEREF': 148, 'COPY_FREE_VARS': 149, 'RESUME': 151, 'MATCH_CLASS': 152, 'FORMAT_VALUE': 155, 'BUILD_CONST_KEY_MAP': 156, 'BUILD_STRING': 157, 'LOAD_METHOD': 160, 'LIST_EXTEND': 162, 'SET_UPDATE': 163, 'DICT_MERGE': 164, 'DICT_UPDATE': 165, 'PRECALL': 166, 'CALL': 171, 'KW_NAMES': 172, 'POP_JUMP_BACKWARD_IF_NOT_NONE': 173, 'POP_JUMP_BACKWARD_IF_NONE': 174, 'POP_JUMP_BACKWARD_IF_FALSE': 175, 'POP_JUMP_BACKWARD_IF_TRUE': 176}
```

具体每种操作码对应的含义可以查阅官方文档，每个 python 版本间略有差异。
- [dis — Disassembler for Python bytecode — Python 3.12.7 documentation](https://docs.python.org/3.12/library/dis.html#opcode-IMPORT_NAME)

#### 沙箱示例

以 LACTF 2023 中的 Pycjail 为例，这道题定义了一个 banned 黑名单：
```py
banned = ["IMPORT_NAME", "MAKE_FUNCTION"]
for k in opcode.opmap:
    if (
        ("LOAD" in k and k != "LOAD_CONST")
        or "STORE" in k
        or "DELETE" in k
        or "JUMP" in k
    ):
        banned.append(k)
banned = {opcode.opmap[x] for x in banned}

...
elif any(code[i] in banned for i in range(0, len(code), 2)):
    print("banned opcode >:(")
```
- 将 IMPORT_NAME、MAKE_FUNCTION 以及包含 LOAD、STORE、DELETE、JUMP 的操作码（LOAD_CONST 除外）存放进黑名单。
- 通过遍历用户的输入的字节码来判断是否存在黑名单操作码。


## 参考资料
- [Python沙箱逃逸](https://www.cnblogs.com/h0cksr/p/16189741.html)
- [Python沙箱逃逸小结](https://www.mi1k7ea.com/2019/05/31/Python%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E5%B0%8F%E7%BB%93/)
- [Bypass Python sandboxes](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes)
- [[PyJail] python沙箱逃逸探究·上（HNCTF题解 - WEEK1）](https://zhuanlan.zhihu.com/p/578986988)
- [[PyJail] python沙箱逃逸探究·中（HNCTF题解 - WEEK2）](https://zhuanlan.zhihu.com/p/579057932)
- [[PyJail] python沙箱逃逸探究·下（HNCTF题解 - WEEK3）](https://zhuanlan.zhihu.com/p/579183067)
- [audited2](https://ctftime.org/writeup/31883)
- [[GCTF 2022] Treebox | /dev/ur4ndom - Robin Jadoul](https://ur4ndom.dev/posts/2022-07-04-gctf-treebox/)