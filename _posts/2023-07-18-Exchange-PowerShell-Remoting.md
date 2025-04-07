---
title: Exchange PowerShell Remoting
date: 2023-07-14 05:34:21
categories:
- Exchange
tags:
- Exchange
mermaid: true
image:
  path: /assets/images/exchange.jpg
toc: true
---


## Exchange 测试环境搭建
参考资料：
- [Exchange系列文章——Exchange2019部署安装 - 小贝笔记](https://www.xiaobei.one/archives/775.html)

安装步骤除了文章中描述的以外，还需要安装一个 URL 重写模块，Exchange server 2019 CU 11 以上需要安装。

- 切换域管理员或拥有Exchange管理权限的用户登录并安装必备组件（此次实验环境用域管理员登录）
  - [.NET Framework 4.8](https://go.microsoft.com/fwlink/?linkid=2088631)
  - [Visual C++ Redistributable Package for Visual Studio 2012](https://www.microsoft.com/download/details.aspx?id=30679)
  - [Visual C++ Redistributable Package for Visual Studio 2013](https://support.microsoft.com/help/4032938/update-for-visual-c-2013-redistributable-package)
  - [Unified Communications Managed API 4.0](https://www.microsoft.com/download/details.aspx?id=34992)
  - 通过Power Shell安装Exchange必备的Windows组件
    ```powershell
    Install-WindowsFeature Server-Media-Foundation, NET-Framework-45-Features, RPC-over-HTTP-proxy, RSAT-Clustering, RSAT-Clustering-CmdInterface, RSAT-Clustering-Mgmt, RSAT-Clustering-PowerShell, WAS-Process-Model, Web-Asp-Net45, Web-Basic-Auth, Web-Client-Auth, Web-Digest-Auth, Web-Dir-Browsing, Web-Dyn-Compression, Web-Http-Errors, Web-Http-Logging, Web-Http-Redirect, Web-Http-Tracing, Web-ISAPI-Ext, Web-ISAPI-Filter, Web-Lgcy-Mgmt-Console, Web-Metabase, Web-Mgmt-Console, Web-Mgmt-Service, Web-Net-Ext45, Web-Request-Monitor, Web-Server, Web-Stat-Compression, Web-Static-Content, Web-Windows-Auth, Web-WMI, Windows-Identity-Foundation, RSAT-ADDS
    ```
  - 安装 URL 重写模块，CU 11 以上需要安装：[URL Rewrite : The Official Microsoft IIS Site](https://www.iis.net/downloads/microsoft/url-rewrite)



## Exchange Powershell

### PowerShell Remoting Protocol
PSRP（PowerShell Remoting Protocol）是一个位于 WSMan/WinRM 协议之上的协议，旨在与 PowerShell 实例远程交互。

协议的具体内容可参考官方文档：[[MS-PSRP]: PowerShell Remoting Protocol](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-psrp/602ee78e-9a19-45ad-90fa-bb132b7cecec)，**其中包含了协议具体设计以及数据结构、序列化与反序列化等内容**。

pypsrp 是基于 PSRP 协议的一个 python 实现，github 链接：[jborean93/pypsrp: PowerShell Remoting Protocol for Python](https://github.com/jborean93/pypsrp)

下面是一个使用示例，基于 PowerShell Remoting Protocol 可以远程访问 PowerShell，需要注意的是，此时访问的是本地 PowerShell，而不是 Exchange PowerShell

```py
from pypsrp.powershell import PowerShell, RunspacePool
from pypsrp.wsman import WSMan

wsman = WSMan("localhost", username="DR34D\Administrator", password="xxxx",
              cert_validation=False,port=5985,ssl=False)

with RunspacePool(wsman) as pool:
    ps = PowerShell(pool)
    ps.add_cmdlet("Invoke-Expression").add_parameter("Command", "Get-PSDrive -Name C")
    ps.add_cmdlet("Out-String").add_parameter("Stream")
    ps.invoke()
    print("\n".join(ps.output))
```

### Exchange Management Shell
Exchange Server 提供了一个 Exchange Management Shell，其本质是一个 PowerShell 沙箱，提供了用于管理 Exchange Server 的一系列命令。其实现也是依赖 WinRM 服务，底层同样是 PSRP。

Exchange Management Shell 支持的命令可参考官方文档：[ExchangePowerShell Module](https://learn.microsoft.com/en-us/powershell/module/exchange/?view=exchange-ps)

### Exchange PowerShell Remoting
Exchange Management Shell 是 Exchange Server 服务器中内置的工具，而 Exchange PowerShell Remoting 可以看作是一个远程使用 Exchange Management Shell 的一个服务。

使用管理员账户登陆 /ecp，找到"服务器" --> "虚拟目录"，可以看到一个 PowerShell 节点。

![20230717210726](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230717210726.png)

该虚拟目录的内部 URL 为 FQDN/powershell，默认情况下外部 URL 为空，身份认证中提供了 Kerberos 和 Basic 两种认证方式，但默认情况下没有开启，因此从外部无法正常使用该功能。

![20230717210830](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230717210830.png)

Exchange PowerShell Remoting 这个功能的本义是方便不同 Exchange Server 之间调用 PowerShell 命令实现管理的自动化，不应该由外部直接访问，一旦滥用会造成很多安全问题。在 Exchange 诸多历史漏洞中，例如 ProxyShell、ProxyNotShell，都是通过 SSRF 访问到后端 /powershell 节点进行利用。

分析 ProxyShell、ProxyNotShell 漏洞时我就在疑惑，与 /powershell 节点交互的数据包格式是怎么来的，是否有比较快捷的方式获取呢？

默认情况下，Kerberos 和 Basic 两种认证方式没有开启，如果将其开启，则支持从外部直接访问 /powershell，这就为抓包带来了便利。

![20230717214714](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230717214714.png)

测试脚本可以使用 pysprp 库，下面是一个使用示例：
```python
from pypsrp.powershell import PowerShell, RunspacePool
from pypsrp.wsman import WSMan

## import logging
## logging.basicConfig(level=logging.DEBUG)

wsman = WSMan(
    server="192.168.137.129",
    username="DR34D\\Administrator",
    password="xxxxx",
    ssl=False,
    port=80,
    auth="basic",
    encryption="never",
    path="PowerShell",
)

with RunspacePool(wsman, configuration_name="Microsoft.Exchange") as pool:
    ps = PowerShell(pool)
    ps.add_cmdlet("Get-Mailbox").invoke()
    out = "\n".join([str(s) for s in ps.output])
    print(out)
```
如果要配合 burpsuite 进行抓包，只要设置环境变量即可，如下所示：
```py
import os

os.environ['http_proxy'] = 'http://localhost:8080'
os.environ['https_proxy'] = 'http://localhost:8080'
```


总结来看，Exchange PowerShell Remoting 与正常 Remote PowerShell 有所不同，主要有以下几点：
1. 访问 URL 不同，WinRM 通常监听 5985 与 5986 端口（SSL）,需要访问 /wsman，而 Exchange 访问的是 80 端口的 /powershell。
2. 认证方式不同，正常情况下 WinRM 可以使用 Kerberos 认证方式，而 Exchange 默认情况下 /powershell 节点与 /owa 认证集成到了一起， OWA 认证之后得到的 Token，可以直接用于访问 /powershell，当然，Exchange 也支持开启 Kerberos 和 Basic 认证，但默认情况下不会开启。


## 参考
- [针对Exchange的攻击方式 - 跳跳糖](https://tttang.com/archive/1487/)
- [渗透基础——Exchange Autodiscover的使用](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-Exchange-Autodiscover%E7%9A%84%E4%BD%BF%E7%94%A8)
- [【技术原创】渗透技巧——远程访问Exchange Powershell](https://www.4hou.com/posts/zlj2)
- [ExchangePowerShell Module](https://learn.microsoft.com/en-us/powershell/module/exchange/?view=exchange-ps)
- [PowerShell Remoting Protocol](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-psrp/602ee78e-9a19-45ad-90fa-bb132b7cecec)
- [Exchange系列文章——Exchange2019部署安装 - 小贝笔记](https://www.xiaobei.one/archives/775.html)