---
title: "搭建github博客"
date: 2015-02-19
tags: blog
---

记录下搭建本地jekyll环境遇到的问题。

<!--more-->

---

* 主要搭建的步骤，参考github[说明](https://help.github.com/articles/using-jekyll-with-pages/)。
* 在安装bundler时，会出现某些包安装失败，可以先按提示尝试手动安装。如果出现nokogiri安装失败，参照[stackoverflow](http://stackoverflow.com/questions/8003523/error-installing-nokogiri-1-5-0-with-rails-3-1-0-and-ubuntu)需要安装完整的rvm依赖。
* 运行Jekyll时出现ExecJS环境问题，参考网上意见，简单的方法就是直接安装Node.js，具体的步骤可以参考[博客](http://blog.csdn.net/kucss/article/details/6832191)。

搭建成功后，运行 `bundle exec jekyll serve` ，即可在 `http://127.0.0.1:4000/blog/` 看到网页。

其他
---

* ubuntu sublime的中文输入问题，参考[博客](http://html5beta.com/page/ubuntu-14-04-install-fcitx-sougoupinyin-sublime-text-3-chinese-input-fix.html)。如果安装的是sublime2，注意要修改 `源（改成2）`  和快捷方式代码中 `Exec值中文件夹sublime_text改成sublime_text_2` 。

* 基本步骤：参考[阮一峰的博客](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)。
* 另一篇详细点的[博客](http://www.zhanxin.info/jekyll/2013-08-07-jekyll-doc-installation.html)
