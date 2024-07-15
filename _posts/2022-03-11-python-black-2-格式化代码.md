---
layout: post
title: "python-black 2 格式化代码"
category: Language
tags: emacs
date: 2022-03-11 12:00
---


### 缘由：
目前使用emacs敲python代码，每次都需要手动使用black命令格式化 git status下的修改文件，遂改写一个black 命令 black2（站在巨人肩膀上，感谢psf)。废话不多说，上代码


```
#!/usr/local/opt/python@3.9/bin/python3.9
# -*- coding: utf-8 -*-

import os
from time import sleep

cmd = "git status | grep 'modified'"
a = os.popen(cmd).read()
b = a.replace("modified:", "").splitlines()
for i in b:
    os.popen("black " + i.lstrip())
    sleep(1)
```

代码原理很简单，把git status中筛出来的文件做一次black  格式化。这样一个black2 命令即可执行多个文件的批量格式化（再次感谢psf）
把该文件放在 计算机变量中（mac 可放在 /usr/local/bin/black2 跟black 同级），chmod +x 加执行权限即可

### PS
需提前 pip install black （此为肩膀）
