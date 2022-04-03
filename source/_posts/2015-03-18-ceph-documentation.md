---
title: "Ceph文档编译"
date: 2015-03-18
tags: ceph
---

Ceph的文档可以从代码中直接编译，主要使用了两个工具`doxygen`和``sphinx`。doxygen会生成xml和一些图表，sphinx会在这些基础上再生成完整的网页。

<!--more-->

文档的编译过程如下：

1.克隆代码

    git clone https://github.com/ceph/ceph.git

2.安装工具

    编译所需的以来可以在文件`doc_deps.deb.txt`中找到。

3.利用doxygen生成xml，执行命令

    doxygen Doxyfile

`Doxyfile`为根目录配置文件名，可以通过`doxygen -g`来生成一个新的文件，查看每个选项的含义。注意，需要在根目录预先新建`build-doc`目录。
执行完后，build-doc中就会多了doxygen目录，里面有xml文件。

4.利用sphinx生成文档。进入build-doc目录（因为sphinx会依赖doxygen的生成，不进目录会提示找不到xml文件）

    cd build-doc
    sphinx-build -b html ../doc html

`doc`为sphinx配置文件目录。详细说明见[sphinx命令](http://zh-sphinx-doc.readthedocs.org/en/latest/invocation.html)

目前有个问题，生成的html有些链接不带后缀`.html`，阅览不太方便，还没解决。