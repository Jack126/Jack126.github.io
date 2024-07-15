---
layout: post
title: "Linux find指令过滤掉没有查看权限的文件"
category: Linux
tags: Linux
date: 2022-09-01 11:00
---


如题，find指令寻找某个文件时，
如果，
find 路径 -name 文件名
通常会列出很多无权限查看文件，
事实上这些文件也不是我们要找的，
有时候占了很多篇幅影响视线。
```
find 路径 -name 文件名 2>/dev/null
```
就能过滤掉，
也就是说，后面加了：
```
2>/dev/null
```
记录用，如有侵，告之必删 
