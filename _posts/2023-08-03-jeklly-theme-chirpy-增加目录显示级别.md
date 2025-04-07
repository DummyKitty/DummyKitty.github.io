---
title: Jekyll-theme-chirpy 主题增加目录展示级别
date: 2023-07-30 10:23:57
categories:
- ENV
tags:
- Jekyll
toc: true
mermaid: true
image:
  path: /assets/images/Jekyll-theme-chirpy.jpg
---


Jekyll-theme-chirpy 主题默认只能展示 h2, h3，如果想要显示 h2,h3,h4, 可以参考：[Ability to control/increase the Heading levels suppported by TOC Plugin · Issue #1023 · cotes2020/jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/issues/1023) 中  @otzslayer 的回复：

> I've been modifying it myself so that H4 appears in the ToC, but I'm not sure if there's a problem.

> [otzslayer@ 2128083](https://github.com/otzslayer/otzslayer.github.io/commit/2128083b857df302ffab55494c9312213656ff40)

> This is the commit I pushed to use, and it looks fine on my blog at the moment.

> https://otzslayer.github.io/cs224w/2023/06/16/cs224w-05.html#%EC%A0%84%ED%86%B5%EC%A0%81%EC%9D%B8-gnn-%EB%A0%88%EC%9D%B4%EC%96%B4-1-gcn

这位老哥的 commit 值接照搬就可以同时显示 h2, h3, h4。

如果想要同时显示 h1,h2,h3,h4，可以在上面修改的基础之上，将 h1 加进去，需要改动三个文件：

toc.html：
{% raw %}
```html
{% assign enable_toc = false %}
{% if site.toc and page.toc %}
  {% if page.content contains '<h1' or page.content contains '<h2' or page.content contains '<h3' or page.content contains '<h4' %}
    {% assign enable_toc = true %}
  {% endif %}
{% endif %}

{% if enable_toc %}
  <div id="toc-wrapper" class="ps-0 pe-4 mb-5">
    <div class="panel-heading ps-3 pt-2 mb-2">{{- site.data.locales[include.lang].panel.toc -}}</div>
    <nav id="toc"></nav>
  </div>
{% endif %}
```
{% endraw %}

toc.js
{% raw %}
```js
export function toc() {
  // if (document.querySelector('#core-wrapper h2,#core-wrapper h3')) {
    if (
      document.querySelector('#core-wrapper h1,#core-wrapper h2,#core-wrapper h3,#core-wrapper h4')
    ) {
    // see: https://github.com/tscanlin/tocbot#usage
    tocbot.init({
      tocSelector: '#toc',
      contentSelector: '.post-content',
      ignoreSelector: '[data-toc-skip]',
      // headingSelector: 'h2, h3',
      headingSelector: 'h1, h2, h3, h4',
      orderedList: false,
      scrollSmooth: false
    });
  }
}
```
{% endraw %}

post.scss 再加一层。
{% raw %}
```css
    ul {
      a {
        padding-left: 2rem;
      }
      ul {
        a {
          padding-left: 3rem;
        }
        ul {
          a {
            padding-left: 4rem;
          }
        }
      }
    }
```
{% endraw %}
## 参考
- [Ability to control/increase the Heading levels suppported by TOC Plugin · Issue #1023 · cotes2020/jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/issues/1023)