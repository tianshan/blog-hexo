---
title: "python pdb调试"
date: 2015-08-26
tags:
- python
- pdb
---

简单记录pdb的使用

```python
# 通过再代码中设置
import pdb
pdb.set_trace() # 代码会停留再改语句，然后进入调试模式
# 或者
python -m pdb *.py
```

基本命令

命令  | 用途 | 说明
---- | ---- | ---
break 或 b    | 设置断点 | b *.py:line or b *.function()
continue 或 c | 继续执行程序 | 
list 或 l     | 查看当前行的代码段
step 或 s     | 进入函数
return 或 r   | 执行代码直到从当前函数返回
exit 或 q     | 中止并退出
next 或 n     | 执行下一行
p             | 打印变量的值
help          | 帮助

第一条命令后继续按enter，会重复上一条命令。
在运行中可以动态改变变量的值。
