---
title: Flask 中的 XSS
date: 2023-07-30 10:23:57
categories:
- Python
tags:
- xss
mermaid: true
image:
  path: /assets/images/flask.png
toc: true
---

在 Flask 应用中，XSS 攻击通常发生在模板引擎渲染阶段，特别是在使用 {% raw %}{{ ... }}{% endraw %} 语法输出用户数据的时候，下面是 flask 导致 XSS 的利用场景。
- 使用危险模板渲染函数
  - render_template_string
  - flask.Markup 
- 绕过模板引擎
  - 路由信息带入返回值
- 模板语法的错误使用
  - safe 过滤器
  - 关闭 autoescape
  - 不带引号的模板变量渲染到 HTML 属性中
  - 模板变量渲染到 href 属性中
  - 模板变量渲染到 `<script>` 标签中
- 异常处理导致的 XSS

## 使用危险模板渲染函数

### 使用 render_template_string 进行渲染
render_template_string 函数用于直接渲染传递的字符串作为模板，使用这个函数可以从用户输入中动态创建模板，但用户输入值接拼接进模板会造成 SSTI 或 XSS。
```py
@web.route('/test1', methods=['GET'])
def test1():
    return render_template_string("<div>%s</div>" % request.args.get("name"))
```
例如输入：
```
name=<script>alert(%27XSS%27)</script>
```
### 使用 flask.Markup 进行渲染 
在 Flask 中，为了防止 XSS，模板引擎会默认对输出进行转义，但 flask.Markup 会将字符串以 HTML 原始形式输出，不会进行转义，在一些情况下会导致 XSS。

```python
from flask import Markup

@web.route('/test3', methods=['GET'])
def test3():
    template_string = '<h1><script>alert("XSS");</script></h1>'
    return Markup(template_string)
```
## 绕过模板引擎

### 路由信息带入返回值
直接从路由中获取输入来返回内容，这种方式会绕过模板渲染引擎，不会进行任何转义，例如：
```python
@app.route("/index/<msg>")
def index(msg):
  return "Hello! " + msg
```

## 模板语法的错误使用

### 使用了 safe 过滤器
safe 过滤器会禁用 HTML 转义，容易导致 XSS，例如 TFC CTF 2023 BABY DUCKY NOTES 中的漏洞代码：
{% raw %}
```py
        {% for post in posts %}
        <li>
            <div class="blog_post">
                <div class="container_copy">
                  <h1> {{post.get('title')}} </h1>
                  <h3> {{post.get('username')}} </h3>
                  <p> {{post.get('content') | safe}} </p>
                </div>
            </div>
        </li>
        {% endfor %} 
```
{% endraw %}
题目在 content 属性中使用了 safe 过滤器，导致内容不会被转义，从而产生 XSS。

### 关闭 autoescape
在 Flask 中，autoescape 是模板引擎的一个配置选项，用于控制模板渲染时的自动转义行为。默认情况下，autoescape 设置为 True，即自动对输出进行转义。如果将 autoescape 设置为 False ，就有可能造成 XSS。

设置 autoescape 的方式有两种：
1. 应用全局配置： 可以在 Flask 应用的配置中设置 autoescape，这会影响所有的模板渲染。
    ```py
        app = Flask(__name__)
        app.config['TEMPLATES_AUTO_RELOAD'] = True  # 全局设置自动转义
    ```


2. 模板级别配置： 可以在单个模板文件中设置 autoescape，这会仅影响当前模板的渲染。
    {% raw %}
    ```html
        {%- autoescape false %}
        <p>{{ user_input }}</p> 
        {%- endautoescape %}
    ```
    {% endraw %}
### 不带引号的模板变量渲染到 HTML 属性中
如果模板变量需要拼接到 html 标签的属性中，但又没有加上引号，就有可能造成标签属性的注入，例如：
{% raw %}
```html
<div class={{ classes }}></div>
```
{% endraw %}
### 模板变量渲染到 href 属性中
如果模板变量需要拼接到 html 标签的属性中，并且加上链引号，在属性为 href 时仍有可能造成 XSS。
{% raw %}
```html
<a href="{{ link }}"></a>
```
{% endraw %}
原因在于 href 值可以接收 javascript:URI


### 模板变量渲染到 `<script>` 标签中
如果模板变量值接插入到 script 标签中，会造成 JavaScript 代码的注入。例如：
{% raw %}
```js
<script>var name = {{ name }};</script>
```
{% endraw %}
## 异常处理造成的 XSS
当服务器发生异常并返回错误信息给客户端时，如果这些错误信息未经过适当的转义，就有可能导致 XSS。但异常处理需要与服务本身的逻辑结合分析，因此没有一个较为固定的形式，一般可以从异常处理的代码片段开始着手。

以 TFC CTF 2023 DUCKY NOTES: PART 3 为例：
- [TFC CTF 2023](https://ctf.thefewchosen.com/challenges)
- [CTFtime.org / TFC CTF 2023](https://ctftime.org/event/2034)
- [archives](https://drive.google.com/file/d/1boBv7M-jrORcnUsso41acBwOTj7XDjSp/view)

这道题提供了一个类似向管理员发送帖子的页面。其中存在如下的一个异常处理函数。
```py
@app.errorhandler(Exception)
def handle_error(e):
    if not e.args or "username" not in e.args[0].keys():
        return e, 500
    
    error_date = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    post = e.args[0]
    log_message = f'"{error_date}" {post["username"]} {post["content"]}'

    with open(f"{app.config['LOG_DIR']}{error_date}.txt", 'w') as f:
        f.write(log_message)

    return log_message, 500
```
这个函数会将用户输入的 content 值接打印出来，没有进行任何转义，那么就有可能造成 XSS。

为了利用这个异常处理，还需要找到一个可以触发异常的地方。作为客户端输入，python 中可利用的一些异常大多为：
1. KeyError（键错误）： 当访问字典中不存在的键时引发的错误。（用户输入的键名被应用使用）
2. FileNotFoundError（文件未找到错误）： 在尝试打开不存在的文件时引发的错误。
3. ValueError（值错误）： 当函数接收到正确类型的参数，但参数值不合适时引发的错误。
4. TypeError（类型错误）： 当操作或函数应用于不支持的数据类型时引发的错误。
5. NameError（名称错误）： 当尝试访问一个不存在的变量或名称时引发的错误。

因为 handle_error 函数处理的异常信息并不是系统异常，因此利用点出现在题目自生抛出的异常中，搜索 Exception 就可以发现两处抛出异常的地方。
- posts_view 函数。
- posts 函数。

**最终这道题的思路是触发 posts 函数中的 KeyError**。

在创建帖子时如果输入的 title 如果为 null。
```json
{
    "title":null,
    "content":"<img src=x onerror=alert(document.domain)>",
    "hidden":false
}
```
则可以将 tilte 值赋值为 None，导致插入 SQLite 数据库中的值变为 NULL。当管理员进行查看时，需要从数据库中取出数据，代码如下：

```py
@web.route('/posts/', methods=['GET'])
@auth_required
def posts(username):
    if username != 'admin':
        return jsonify('You must be admin to see all posts!'), 401

    frontend_posts = []
    posts = db_get_all_users_posts()

    for post in posts:
        try:
            frontend_posts += [{'username': post['username'], 
                                'title': post['title'], 
                                'content': post['content']}]
        except:
            raise Exception(post)

    return render_template('posts.html', posts=frontend_posts)    

```
db_get_all_users_posts 会从数据库中获取所有记录。

```py
def db_get_all_users_posts():
    con = sqlite3.connect('database/data.db')

    posts = query(con, 'SELECT users.username as username, title, content, hidden from posts INNER JOIN users ON users.id = posts.user_id ')
    return posts
```
但其中的 query 函数对 sqlite fectchall 进行了包装，过滤了其中为 None 的部分，导致 title 为 None 时，整个 title 键值对缺失。
```py
def query(con, query, args=(), one=False):
    c = con.cursor()
    c.execute(query, args)
    rv = [dict((c.description[idx][0], value)
        for idx, value in enumerate(row) if value != None) for row in c.fetchall()]
    return (rv[0] if rv else None) if one else rv
```
最终引发 KeyError。



## 参考资料
- [XSS prevention for Flask - Semgrep](https://semgrep.dev/docs/cheat-sheets/flask-xss/)
- [TFC CTF 2023 Writeups - はまやんはまやんはまやん](https://blog.hamayanhamayan.com/entry/2023/07/30/202849)