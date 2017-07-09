---
title: github pages 域名配置
date: 2015-04-28
tags: blog
---

为github pags配置域名，主要步骤参考[https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/](https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/)

<!--more-->

* 域名配置

官方建议用CNAME。添加一条记录，例如www指向`tianshan.github.io`，生效后，github会自动做解析跳转，例如`tianshan.github.io`会跳转到`www.quts.me`，其他非User pages库也会对应的连接，比如`tianshan.github.io/blog/`跳转到`www.quts.me/blog` 。

* repository的CNAME配置

在库中根目录添加CNAME文件，其中添加域名记录。
注意：这个地方好像只有User Pages需要添加，内容就是域名地址，例如www.quts.me。而其他的库，并不需要，github会自动跳转到域名+库名的链接。
如果其他库(例如:blog)，想映射到不带后缀的域名（例如:blog.quts.me），这样就需要CNAME文件了，而且_config.yml中的baseurl也需要对应修改。这样会导致原来的tianshan.github.io/blog出问题，不过问题也不大。