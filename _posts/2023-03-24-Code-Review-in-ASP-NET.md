---
title: ASP.NET 代码审计
date: 2023-03-24 23:15:11
categories:
- ASP.NET
tags:
- ASP.NET
toc: true
---


##  Methodology of Code Review

Code review is a systematic task that requires us to identify security vulnerabilities in existing code. We need to build a model that can be used to guide the proccess of code reviewing.

Below are some of my thoughts.

1. Network protocol
2. Sensitive information leak: include hard code credentials, public-private key, IP, URL and source code etc.
3. Risk of configuration
4. Risk of Component
5. Vulnerabilities of code

    1. Broken Access Control
    2. Injection
    3. etc

‍

##  Learn Basic of ASP.NET

Learn from video is the fast way to improve you operation skill.

* [使用ASP.NET开发Web应用程序](https://www.bilibili.com/video/BV16X4y1V7qH/?p=6&spm_id_from=pageDriver)

‍

###  Official Document

* [ASP.NET overview](https://learn.microsoft.com/en-us/aspnet/overview)
* [IIS configuration](https://learn.microsoft.com/en-us/iis/configuration/)

‍

###  Build a sample application

Create new web application in vs.

![20230216085115-d9391wd](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085115-d9391wd.png)

The visual studio will automatically create a Web Form website

![20230216085242-ffxi89c](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085242-ffxi89c.png)

It cann be noticed that an aspx file is actually composed of two parts, the front-end .aspx file and the back-end .cs file.

 The "CodeFile" specified in the front-end .aspx file refers to the corresponding .cs file.

![20230216085442-479sxxe](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085442-479sxxe.png)

All of .cs file are inherit from the Page class.

![20230216085347-js8o565](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085347-js8o565.png)
‍

![20230216085552-cxd53cv](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085552-cxd53cv.png)

‍
###  Empty web application

Ceate a empty website 

![20230216085659-4t5dfcw](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085659-4t5dfcw.png)

Creat a Web Form

![20230216085815-d667kw7](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085815-d667kw7.png)

This will generate an aspx file.

![20230216085857-z3p014w](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216085857-z3p014w.png)

点击下方的拆分可以同时看到 aspx 前端源码以及设计界面。

![20230216090055-0wabacp](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216090055-0wabacp.png)

Open the tool box

![20230216090219-sqhhgee](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216090219-sqhhgee.png)

We can drag corresponding controls from the toolbox onto the design surface. Here is an TextBox and I modifyied its ID in the properties page.

![20230216091014-ca4ft8t](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216091014-ca4ft8t.png)

Other controls are the same, such as the button below.

![20230216093355-v1xkm7t](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216093355-v1xkm7t.png)

So as the following label.

![20230216093554-mkm8nbo](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216093554-mkm8nbo.png)

Double-clicking on a control in the design surface will take you to the code editor. The `btnSure_Click`​​is the function that is executed when the button is click.

![20230216093706-t4lwssn](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/图片-20230216093706-t4lwssn.png)

We can code whatever within this function.

‍

##  Build Environment

###  ASP.NET Environment

- [Win7下利用IIS自建网站全攻略！包括ASP和ASP.NET动态网站！高手勿入](https://edi.wang/post/2010/7/18/setup-iis-in-windows7-both-asp-and-aspnet)

- [Asp.Net MVC3.0项目部署到Win7 64过程总结](https://developer.aliyun.com/article/343659)

‍

####  SQL Server

- [How to download and install Microsoft SQL Server Management Studio 2008 for SQL Server 2008](https://www.daktronics.com/en-us/support/kb/DD3400198)

- [Microsoft® SQL Server® 2008 R2 SP2 - Express Edition](https://www.microsoft.com/zh-TW/download/details.aspx?id=30438)

- [SQL Server Management Studio 教程一：设置sa用户登录](https://www.cnblogs.com/turnip/p/12145330.html)

‍

###  Tools

####  SAST tools

#####  Fortify SCA

Fortify Static Code Analyzer (SCA) is a software security tool that is used to identify and fix vulnerabilities in software applications. It is designed to analyze the source code of an application and identify potential security flaws before the application is deployed. Fortify SCA provides developers with a comprehensive security analysis of their code, including identifying vulnerabilities, providing remediation guidance, and tracking progress over time. This helps organizations to build more secure applications and reduce the risk of security breaches.

Download links:

> version 22.1

链接：https://pan.baidu.com/s/1K5QdQJCjzeW5_MVfcPR45g?pwd=4j0u  
提取码：4j0u  
--来自百度网盘超级会员V5的分享

> version 20.1.1

链接：https://pan.baidu.com/s/1jleP1gfHUfkqer2kAULYnA?pwd=wcmt  
提取码：wcmt  
--来自百度网盘超级会员V5的分享

解压密码：shungg.cn

‍

#####  dnSpy

dnSpy is a free and open-source .NET debugger and assembly editor. It is used by developers and security researchers to reverse engineer .NET applications and analyze their code. dnSpy can decompile .NET assmblies into C#,Visual Basic and IL code, allowing users to view and modify the code of an application. It also includes a debugger that allow users to stop through the code of an application, set breakpoints, and inpsect variables.

dnSpy supports a wide range of .NET assemblies, including .NET core, .NET Standard, and .NET Framework. 

Download  link:

* https://github.com/dnSpy/dnSpy

‍

#####  dotPeek

dotPeek is a free and lightweight .NET decompiler and code viewer developed by JetBrains. It is used by developers and software engineers to reverse engineer .NET applications and view their source code. dotPeek can decompile .NET assemblies into C#, VB.NET, and IL code, allowing users to easily view and analyze the code of an application.

Download link:

* https://www.jetbrains.com/decompiler/download/

‍

> Both dotPeek and dnSpy are .NET decompilers and code viewers, but they have some differences in terms of features and capabilities.
>
> dotPeek is a lightweight and user-friendly tool that is well-suited for viewing and analyzing code, while dnSpy is a more advanced tool that includes a debugger and is better suited for reverse engineering and modifying code.

‍

####  Front-end code viewing tool.

#####  Visual Studio

Visual Studio provides a range of features and tools that make ASP.NET development easier and more efficient.

1. Visual Design: Visual Studio includes a visual design tool that allows developers to create ASP.NET web pages using drag-and-drop controls. This helps developers create visually appealing and responsive web pages quickly.

In the work of code review, we often face the dilemma of having only .aspx files and binary files. Visual Studio can display front-end pages with only .aspx files, which is very helpful for our auditing of front-end forms and input parameters.

It is recommended to use Visual Studio 2015 or later versions as they can intergrate better with Fortify SCA.

‍

####  Third part component check

* [Dependency Check Guide to Help You Find Vulnerabilities in Open-source Software Components](https://relevant.software/blog/dependency-check-guide-vulnerabilities-open-source-software/)

‍

#####  DependencyCheck

Download link：[DependencyCheck](https://github.com/jeremylong/DependencyCheck)****

**Environment required：**

> 1. To analyze .NET Assemblies the dotnet 6 run time or SDK must be installed.
>
>     * Assemblies targeting other run times can be analyzed - but 6 is required to run the analysis.

**Usgae：**

On *nix

```
$ ./bin/dependency-check.sh -h
$ ./bin/dependency-check.sh --out . --scan [path to jar files to be scanned]
```

On Windows

```
> .\bin\dependency-check.bat -h
> .\bin\dependency-check.bat --out . --scan [path to jar files to be scanned]
```

On Mac with [Homebrew](http://brew.sh/) Note - homebrew users upgrading from 5.x to 6.0.0 will need to run `dependency-check.sh --purge`​.

```
$ brew update && brew install dependency-check
$ dependency-check -h
$ dependency-check --out . --scan [path to jar files to be scanned]
```

‍

‍

####  Other tools

#####  C# Online
- [C#在线编译器 ](http://cs.jsrun.net/)

‍

#####  SonarSource

‍

#####  Security Code Scan

[static code analyzer for .NET](https://security-code-scan.github.io/#IntegrationwithContinuousIntegration(CI)buildsandthird-partytools)

######  Installation

* [Visual Studio extension](https://marketplace.visualstudio.com/items?itemName=JaroslavLobacevski.SecurityCodeScanVS2019). Use the link or open “Tools > Extensions and Updates…” Select “Online” in the tree on the left and search for SecurityCodeScan in the right upper field. Click “Download” and install.
* [NuGet package](https://www.nuget.org/packages/SecurityCodeScan.VS2019/).

  * Right-click on the root item in your solution. Select “Manage NuGet Packages for Solution…”. Select “Browse” on the top and search for SecurityCodeScan.VS2019. Select project you want to install into and click “Install”.
  * Another option is to install the package into all projects in a solution: use “Tools > NuGet Package Manager > Package Manager Console”. Run the command `Get-Project -All | Install-Package SecurityCodeScan.VS2019`​.
* [Stand-alone runner](https://www.nuget.org/packages/security-scan/). Install with `dotnet tool install --global security-scan`​ and run `security-scan /your/solution.sln`​. For older .NET 4.x please use `security-scan4x.zip`​ from [GitHub Releases](https://github.com/security-code-scan/security-code-scan/releases).

‍

‍

#####  reko

- [推荐一款采用 .NET 编写的 反编译到源码工具 Reko](https://www.dongchuanmin.com/net/4568.html)

‍

#####  ReSharper

- [适用于.NET开发者的Visual Studio扩展](https://www.jetbrains.com/zh-cn/resharper/)

‍

###  Debug

dnSpy provides additional debugging functionality. When debugging .net framework projects, you need to drag the DLLs in the /bin directory into dnSpy. You can determine which DLLs are compiled from developer code, which are third-party libraries, and which are system libraries by observing the file names in the /bin directory. 

Here is an example: filenames like App_Web_xxx and App_Code are developer-written code in .net framework projects, App_Code.dll comes from the App_Code directory in the source code, App_Web_xxx comes from other directories, and log4net.dll is a third-party library. DLLs starting with "System" are generally system libraries.

```c#
├── ADO.dll
├── AjaxControlToolkit.dll
├── AntiXSSLibrary.dll
├── Apmtech.dll
├── App_Browsers.dll
├── App_Code.dll
├── App_global.asax.dll
├── App_Web_0pk0kvkf.dll
├── App_Web_0xkl5pmw.dll
├── App_Web_d4msxdcz.dll
├── App_Web_elxhpigl.dll
├── App_Web_fyo2axmt.dll
├── App_Web_jgcwvu0c.dll
├── App_Web_kcgcewvf.dll
├── App_Web_liiehwei.dll
├── App_Web_lond3s1y.dll
├── App_Web_lqvh1efk.dll
├── App_Web_o24jbtmm.dll
├── App_Web_oovjeiib.dll
├── App_Web_r3tukein.dll
├── App_WebReferences.dll
├── App_Web_wcbicfv1.dll
├── App_Web_yjumwbb1.dll
├── App_Web_yqwe0f2h.dll
├── log4net.dll
├── System.Web.Helpers.dll
...
```

After dragging the dll into dnSpy, you can also identify the developer's own code compiled in the .net framework project by version. The version of the developer's own code is generally 0.0.0.0.

You can perform additional debugging of the .net framework in dnSpy by following these steps:

Debug --> Attach to Process --> w3wp.exe --> Attach

‍

####  Cases where breakpoints are invalid

In some cases, breakpoints are set but appear to be invalid in dnSpy. This is often because the opened dll is not loaded into memory. The following steps can help resolve this issue:

1. Search for the class you want to debug in the Search window. Once found, double-click to enter the code page for the class.
2. Check the name of the dll where the class is located, such as App_Web_jgcwvu0c.dll.
3. In the Modules window, search for this dll and double-click to open it. Often, the opened dll is not the one you previously dragged into dnSpy. This newly opened dll is the one that has actually been loaded into memory.
4. Locate the section you want to debug in the newly opened dll, set breakpoints, and attach to debug again.

‍

###  Decompile

You can drag all the dlls to be decompiled into dnSpy, then select all --> Export to Project.

You can also use the dnSpy terminal program to decompile in batches. Place the dlls to be decompiled in a directory, and use `dnSpy.Console.exe -o C:\out\path C:\some\path`​ to decompile them.

‍

##  Vulnerabilitis

###  Vulnerability Samples

* [Security rules](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/security-warnings)
* [SonarQube C# static code analysis rules](https://rules.sonarsource.com/csharp/RSPEC-6096)
‍

###  Labs

* [WebGoat.NET](https://github.com/jerryhoff/WebGoat.NET) - OWASP WebGoat.NET
* [Damn Vulnerable Thick Client App](https://github.com/secvulture/dvta) - DVTA is a Vulnerable Thick Client Application developed in C# .NET
* [ASP.NET Vulnerable Site](http://aspnet.testsparker.com/) - Online .NET application that can be used to practice hacking.

‍

###  Blogs

* [初识.Net审计 ](https://www.cnblogs.com/nice0e3/p/15236334.html)
* [卷入.NET WEB](https://paper.seebug.org/1894/)
* [记一次靠猜的.net代码审计拿下目标](https://www.buaq.net/go-68477.html)
* [Unable to trigger injection flaws for C# with Developer Edition](https://community.sonarsource.com/t/unable-to-trigger-injection-flaws-for-c-with-developer-edition/8619)

‍

###  github repo

* [awesome-dotnet-security](https://github.com/guardrailsio/awesome-dotnet-security)
* [NET-Deserialize](https://github.com/Ivan1ee/NET-Deserialize/blob/master/.NET%E9%AB%98%E7%BA%A7%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%EF%BC%88%E7%AC%AC%E4%B9%9D%E8%AF%BE%EF%BC%89%20BinaryFormatter%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E.pdf)
* [sonar rules C# static code analysis](https://rules.sonarsource.com/csharp/RSPEC-6096)

‍

‍