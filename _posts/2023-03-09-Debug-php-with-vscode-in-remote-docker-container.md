---
title: Debug php with vscode in remote docker container
date: 2023-03-09 21:04:32
categories:
- ENV
tags:
- php
- docker
- debug
toc: true
notshow: false
---

本文将介绍如何使用 vscode 调试 docker 中的 php 代码.

# 寻找镜像
Dockerhub 中可以找到一些已经添加了 xdebug 扩展的 php 镜像,例如: 
- [mobtitude/php-xdebug](https://hub.docker.com/r/mobtitude/php-xdebug)

目前 dockerhub 已有的版本如下:
-   mobtitude/php-xdebug:5.6-apache
-   mobtitude/php-xdebug:5.6-cli
-   mobtitude/php-xdebug:5.6-fpm
-   mobtitude/php-xdebug:7.0-apache
-   mobtitude/php-xdebug:7.0-cli
-   mobtitude/php-xdebug:7.0-fpm
-   mobtitude/php-xdebug:7.1-apache
-   mobtitude/php-xdebug:7.1-cli
-   mobtitude/php-xdebug:7.1-fpm
-   mobtitude/php-xdebug:7.2-apache
-   mobtitude/php-xdebug:7.2-cli
-   mobtitude/php-xdebug:7.2-fpm

该项目在 github 上面的 repo 已经可以支持 php 8.1
- [docker-php-xdebug](https://github.com/mobtitude/docker-php-xdebug)

原镜像中 xdebug 配置文件路径为 /usr/local/etc/php/conf.d/xdebug.ini, 在使用的时候可以将其覆盖为自己的配置文件.

# 编写配置
## xdebug.ini
镜像中的 xdbeug 版本都为 2.x, xdebug 版本 2 与版本 3 有所不同.

### version2 
xdebug2.ini
```ini
zend_extension=xdebug.so
xdebug.remote_enable=1
xdebug.remote_handler=dbgp
xdebug.remote_host=host.docker.internal
xdebug.remote_port=9003

xdebug.idekey=PHPSTORM
xdebug.remote_autostart=1
xdebug.auto_trace=1
xdebug.log=/dev/stdout
```
需要注意的是, remote_host 此处填入的 host.docker.internal 表示宿主机, host.docker.internal 在 linux 中可能无法生效, 因此在 docker-compose.yaml 中通常会加入一行 `- "host.docker.internal:host-gateway"` 的配置,具体可以见下面 docker-compose.yaml 的示例.

## docker-compose.yaml

docker-compose.yaml 需要设置两个映射:
1. web 目录
2. xdebug.ini

```yaml
version: '3'
services:
  web:
    platform: linux/amd64
    image: "mobtitude/php-xdebug:5.6-apache"

    volumes:
      - "./:/var/www/html"
      - "./xdebug2.ini:/usr/local/etc/php/conf.d/xdebug.ini"
    ports:
      - "28999:80"

    extra_hosts:
      - "host.docker.internal:host-gateway"
```
extra_hosts 配置就是为了解决 linux 下 host.docker.internal 无效的问题. 原理其实也是添加了一个 host, 并设置为 host-gateway. 但如果直接将 xdebug.remote_host 设置为 host-gateway 是无法成功调试的.

docker 运行起来之后会监听 28999 端口.

## launch.json
使用 vscode 进行调试, 可以在 vscode 中创建 launch.json 文件并添加 php 的配置, 例如:
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch built-in server and debug",
            "type": "php",
            "request": "launch",
            "runtimeArgs": [
                "-S",
                "localhost:8000",
                "-t",
                "."
            ],
            "port": 9003,
            "serverReadyAction": {
                "action": "openExternally"
            }
        },
        {
            "name": "Debug current script in console",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "externalConsole": false,
            "port": 9003
        },
        {
            "name": "Listen for Xdebug on Docker",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/html/":"/home/dr34d/ctf/test/php_deserialize/shell/"
            }
        }
    ]
}
```
由于是远程调试, 需要使用其中的 Listen for Xdebug 配置项. 需要注意的是我们需要在其中添加一个 pathMappings 字段, 用于将目标 web 路径和本地调试的路径对应起来.
```json
"pathMappings": {
    "/var/www/html/":"/home/dr34d/ctf/test/php_deserialize/shell/"
}
```

配置完成之后就可以正常调试了.

PS：现在有了 Dev Container 插件，这种远程调试的方式就没那么方便了。
