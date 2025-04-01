---
title: Debug python with vscode in remote docker container
date: 2023-04-09 21:04:32
categories:
- ENV
tags:
- python
- docker
- debug
toc: true
notshow: true
---


假如我们需要调试某个 Docker container 中的 python 程序，并且这个 container 位于 VPS 上，这时候应该怎么配置呢？

## 安装 vscode 插件： python、Remote-SSH。
python 插件用于调试 python 脚本，Remote-SSH 用来连接到目标 VPS.

## 源码中引入 ptvsd 库
ptvsd 是一个用于 Python 的调试工具，允许使用调试器远程调试 Python 代码，支持 Visual Studio Code、PyCharm 等编辑器的集成。

使用 ptvsd，我们可以在程序的任何位置设置断点，检查代码状态和变量值，并单步调试代码。这种调试方式可以帮助我们更快地诊断代码问题，提高开发效率。

在待调试的脚本中添加如下的代码，其中端口设置为 4000,这表示程序在运行起来后会监听 4000 端口等待 IDE 的附加调试。
```py
import ptvsd

ptvsd.enable_attach(address=('0.0.0.0', 4000), redirect_output=True)
ptvsd.wait_for_attach()
```

## vscode 添加调试配置
在 vscode 中创建调试配置文件 launch.json。添加 Python Remote Attach 然后根据需要进行修改，例如连接目标 host 以及端口 port。

这里的 host 需要设置为容器的 ip 地址，port 也就是上面使用 ptvsd 监听的端口。pathMappings 用于映射本地与容器中的脚本路径。

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Remote Attach",
            "type": "python",
            "request": "attach",
            "connect": {
                "host": "172.22.0.3",
                "port": 4000
            },
            "pathMappings": [
                {
                    "localRoot": "/root/ctf/damCTF_2023/thunderstruck/dist/src",
                    "remoteRoot": "/chal"
                },
                {
                    "localRoot": "/usr/lib/python3/dist-packages",
                    "remoteRoot": "/usr/local/lib/python3.10/site-packages"
                }
            ],
            "justMyCode": false
        }
    ]
}
```
## 运行
在容器中运行脚本后会开启监听，在 vscode 中启动调试并设置断点，就可以开始调试了。