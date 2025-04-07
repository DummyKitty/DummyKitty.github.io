---
title: 将 gitbook 迁移到 mkdocs
date: 2024-01-02 12:29:48
categories:
- ENV
tags:
- gitbook
- mkdocs
mermaid: true
image:
  path: /assets/images/mkdocs.svg

toc: true
---


# 将 gitbook 项目迁移到 mkdocs
很多参考文档使用 gitbook 搭建，例如 [hacktricks-cloud](https://cloud.hacktricks.xyz/welcome/readme)，由于 gitbook 不开源，如果想要将文档离线部署，就不得不更换其他文档生成工具，如 mkdocs、teedocs、docksify 等。

尝试了 teedoc 和 mkdocs，都存在部分问题：
1. teedoc 对于 markdown 图片链接的解析还存在问题，如果图片名包含空格，则链接中需要包含 `<>` 才能够被 markdown 解析，例如：
    ```bash
    ![](<../.gitbook/assets/image (193).png>)
    ```
    但 teedoc 无法将这种链接转化为图片链接。
    ![20240102224306](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20240102224306.png)
2. gitbook 项目的结构通常在 SUMMARY.md 中定义，原始的 mkdocs 仅能够根据目录结构来定义文档结构，生成的文档结构往往是乱的。 
3. gitbook 有着很多自身专用的语法，其他文档生成工具无法解析。例如：
    {% raw %}
    ```
    {% content-ref url="pentesting-kubernetes-services.md" %}
    ```
    {% endraw %}
4. gitbook 静态资源存储在 .gitbook/assets 中。mkdocs 在 build 时不会自动将 .gitbook 文件夹拷贝过去。

通过查询资料，第二个问题可以通过 mkdocs-literate-nav 插件来解决，第三、四个问题可以通过 gitbook2mkdocs 部分解决。

## mkdocs-literate-nav 插件
安装 mkdocs-literate-nav 插件
```bash
proxychains python -m pip install mkdocs-section-index mkdocs-literate-nav
```
原生的 mkdocs 仅支持 yaml 格式来定义导航列表，例如：
```yaml
nav:
  - Frob: index.md
  - Baz: baz.md
  - Borgs:
    - Bar: borgs/bar.md
    - Foo: borgs/foo.md
``` 
安装 mkdocs-literate-nav 插件后，可以直接使用 SUMMARY.md 来定义。只需要在 plugins 中添加 literate-nav。
```yaml
plugins:
  - search
  - literate-nav:
      nav_file: SUMMARY.md
      tab_length: 2
```
但需要注意的是 mkdocs 中 SUMMARY.md 语法相对 gitbook 还有不同，不支持 `##` 这样的 markdown 标题，同级目录之间不能出现空行。

例如，gitbook 中原始的 SUMMARY.md 如下：
```markdown
## 👽 Welcome!
  * [HackTricks Cloud](README.md)
  * [About the Author](https://book.hacktricks.xyz/welcome/about-the-author)
  * [HackTricks Values & faq](https://book.hacktricks.xyz/welcome/hacktricks-values-and-faq)

## 🏭 Pentesting CI/CD
  * [Pentesting CI/CD Methodology](pentesting-ci-cd/pentesting-ci-cd-methodology.md)
  * [Github Security](pentesting-ci-cd/github-security/README.md)
```
更改为 mkdocs 中的 SUMMARY.md：
```markdown
* 👽 Welcome!
  * [HackTricks Cloud](README.md)
  * [About the Author](https://book.hacktricks.xyz/welcome/about-the-author)
  * [HackTricks Values & faq](https://book.hacktricks.xyz/welcome/hacktricks-values-and-faq)
* 🏭 Pentesting CI/CD
  * [Pentesting CI/CD Methodology](pentesting-ci-cd/pentesting-ci-cd-methodology.md)
  * [Github Security](pentesting-ci-cd/github-security/README.md)
    * [Abusing Github Actions](pentesting-ci-cd/github-security/
```
另外，`tab_length: 2` 表示缩进 2 个空格，默认情况下为 4 个，可以根据原始 SUMMARY.md 格式进行修改。

## gitbook2mkdocs 插件
- [pledra/gitbook2mkdocs](https://github.com/pledra/gitbook2mkdocs)

gitbook2mkdocs 插件并不是特别完善，主要功能有两个：
1. 转换部分 gitbook 中的专用语法。
2. 创建 .gitbook/assets 文件夹的 link：gbassets，并将生成的 html 中的图片连接进行替换。

安装：
```bash
pip3 install git+https://github.com/pledra/gitbook2mkdocs
```

使用方法：只需要在 plugins 中进行添加即可
```yaml
plugins:
  - gitbook2mkdocs
```

注意：
1. windows 下创建 symlink 需要管理员权限，执行 mkdocs build 前需要切换到管理员。
2. gitbook2mkdocs 仅会对 docs/.gitbook/assets 文件夹进行操作，也就是项目文档直接放在 docs 目录下，如果多了一层目录，例如：docs/hacktricks/.gitbook/assets，那么该插件不会对目录进行复制（可以修改考虑源码或者手动将文件夹复制过去）。

# 参考资料
- [MkDocs - Workshop Setup](https://ibm.github.io/workshop-setup/MKDOCS/)
- [mkdocs-literate-nav · PyPI](https://pypi.org/project/mkdocs-literate-nav/)
- [参考 - mkdocs-literate-nav](https://oprypin.github.io/mkdocs-literate-nav/reference.html#wildcards)
- [pledra/gitbook2mkdocs](https://github.com/pledra/gitbook2mkdocs)