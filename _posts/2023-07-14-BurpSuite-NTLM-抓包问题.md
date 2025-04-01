---
title: BurpSuite NTLM 抓包问题
date: 2023-07-14 05:34:21
categories:
- Exchange
tags:
- NTLM
image:
  path: /assets/images/burpsuite.png
toc: true
---


使用 burpsuite 抓取 exchange ntlm 认证报文时存在无法响应的情况，一挂代理就无法正常响应。

## 更改 HTTP 协议版本
抓包时发现 burp 会自动使用 http/2 版本的协议，使用 http/1.1 版本协议进行发送时可以得到响应，burpsuite 中修改 http 协议版本有两个地方：
1. Repeater 中的 inspector 框，选择 Request Attributes，将 protocol 选择为 http/1.1。这个方法仅在重放单个包时有效。
2. Project Options 中将 HTTP/2 配置中的 "Default to HTTP/2 if the server supports it" 取消勾选，这样 burpsuite 都会使用 http/1.1 来发送。

设置默认使用 http/1.1 后，能够得到响应，但 ntml 总是无法认证成功。
## burpsuite 对 NTLM 认证的支持
在用 burpsuite 作为代理时， ntlm 总是无法认证成功，不挂代理就可以，查阅资料发现是 burpsuite 对 NTLM 认证支持有问题：[技术讨论之Exchange后渗透分析-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1651853)，文章作者使用 fiddler 来配合抓包。

官方文档也提供了一种方式来进行 ntlm 认证，可参考 [Configuring NTLM with Burp Suite - PortSwigger](https://portswigger.net/support/configuring-ntlm-with-burp-suite)，选择 Project Options --> Connections 中的 Platform Authentication，选择 override user options 选项，添加凭证，其中可以选择 NTLMv2 或者 NTLMv1，配置完后，burpsuite 会在发送报文时自动完成 ntlm 认证过程。

还有一种方式是通过配置 burpsuite 的上游代理为 fiddler 来发送报文，具体可参考：[NTLM认证失效时，如何使用Fiddler配合Burp Suite进行渗透测试？-阿里云开发者社区](https://developer.aliyun.com/article/218500) 

## 参考
- [Working with HTTP/2 in Burp Suite - PortSwigger](https://portswigger.net/burp/documentation/desktop/http2#changing-the-default-protocol)
- [技术讨论之Exchange后渗透分析-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1651853)
- [Configuring NTLM with Burp Suite - PortSwigger](https://portswigger.net/support/configuring-ntlm-with-burp-suite)