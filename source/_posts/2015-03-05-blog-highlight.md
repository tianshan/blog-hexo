---
title: jekyll 语法高亮
date: 2015-03-05
tags: blog
---

记录折腾博客语法高亮的问题。

<!--more-->

博客解析换成kramdown后，语法高亮的问题。

配置，在 `_config.yml` 中添加如下。使用kramdown解析页面，使用pygments来语法高亮。

```
markdown: kramdown
highlighter: pygments
```

kramdown的配置[解释](http://kramdown.gettalong.org/options.html)

pygments支持的语法高亮格式，注意去掉{和%之间的空格。

~~~python
def xx():
    code here
~~~

pygments支持的[语言列表](http://pygments.org/languages/)


krmadown 支持和github一样的语法高亮，用三个 ```，但是需要安装coderay，而github pages上不支持coderay，所以该方式无法搞定，可行的解决方法是上传本地编译好的html。如果是本地或者自己的空间，可以安装coderay。

    gem install coderay

