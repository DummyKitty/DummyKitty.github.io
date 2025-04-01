---
title: Python inf 特性
date: 2023-07-27 10:23:57
categories:
- Python
tags:
- inf
image:
  path: /assets/images/python.png
toc: true
---

# Python 中的 inf 利用
在 Python 中，inf 表示正无穷大（positive infinity）。inf 一个特殊的浮点数值，用于表示没有上限的数值。在一些 Python 应用中，如果某些逻辑是通过用户的输入来做计算的，那么输入 inf 就有可能造成一些安全问题。

## CTF 示例：waiting an eternity 
AmateursCTF 2023 中 waiting an eternity 这道题基于这个背景。
> web/waiting-an-eternity
> 
> voxal
>
> 508 solves / 125 points
> 
> My friend sent me this website and said that if I wait long enough, I could get and flag! Not that I need a flag or anything, but I've been waiting a couple days and it's still asking me to wait. I'm getting a little impatient, could you help me get the flag?

访问这道题的链接，会显示 just wait an eternity。

![20230727212224](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230727212224.png)

响应报文的 set-cookie 头中会给出一个连接：url=/secret-site?secretcode=5770011ff65738feaf0c1d009caffb035651bb8a7e16799a433a301c0756003a

访问这个链接之后会响应在这个页面等待的时间。

![20230727212419](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230727212419.png)

可以看到 Cookie 中有一个 time 参数，猜测回显的时间是为当前时间减去 Cookie 中输入的 time，比如：`current_time - time`

从响应的报文中可以看到目标是一个 gunicorn 服务，结合题目要求需要等待一个永恒的时间，且在 Python 中任何数字减去 -inf 会得到一个无穷大的数，构造 time=-inf 即可得到 flag。
 
![20230727212810](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230727212810.png)

## 参考
- [CTFtime.org / AmateursCTF 2023 / waiting an eternity / Writeup](https://ctftime.org/writeup/37678)
- [les-amateurs/AmateursCTF-Public](https://github.com/les-amateurs/AmateursCTF-Public)