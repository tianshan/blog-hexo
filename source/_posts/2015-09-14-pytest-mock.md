---
title: "pytest的mocking模块"
date: 2015-09-14
tags: pytest
---

在用tox集成py.test时遇到个问题，一个测试脚本需要测试运行流程的正确性，但某个测试函数会调用外部的ceph的命令，又会产生一些其他的依赖，而这些函数不是测试重点，就可以使用mock模块，用一个其他函数来替换有依赖的函数，两个函数输入一致，但是这个函数可以不做任何处理，所以就跳过了依赖调用。

<!--more-->

具体场景是这样的，有个函数需要调用ceph命令获取磁盘，ceph命令又会依赖python-rados，而这个函数这一步并不需要去做检测。所以可以用另一个函数替换，如下：

mock有很多实现，有python的独立安装包[mock](http://www.voidspace.org.uk/python/mock/)等，pytest通过monkeypatch参数实现了这一功能，不用额外依赖。


```python
# 因为是替换的class中的函数，所以会有self参数
def mock_get_blocks(self):
    return '/dev/sda'

# 注意，这里需要参数monkeypatch，pytest会自动处理
def test_run(monkeypatch):
    # 这里会替换 MyClass.get_blocks()函数
    monkeypatch.setattr(MyClass, 'get_blocks', mock_get_blocks)
    # 如果不想import MyClass类，可以通过传入raising来跳过检测
    monkeypatch.setattr(MyClass, 'get_blocks', mock_get_blocks, raising=False)
```

接下来就可以愉快的使用 `tox -e py27` 来做集成测试了。

```ini
# tox集成pytest的简单示例
[testenv:py27]
deps =
    pytest
commands =
    py.test tests/ {posargs}
```

目前应该用场景比较简单，后面有更复杂的话再记录。

参考

* pytest文档对monkeypatch的简单介绍：<https://pytest.org/latest/monkeypatch.html>  
* tox集成pytest的样例：<http://tox.readthedocs.org/en/latest/example/pytest.html>