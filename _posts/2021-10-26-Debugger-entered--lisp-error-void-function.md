---
layout: post
title: "Debugger entered--lisp error void-function"
category: Language
tags: emacs
---


最近在学习使用emacs，尝试了很多配置，最终选定redguardtoo 大神配置，很齐全，很强大。
众所周知，学习emacs，不学elisp不合适，遂学习之。

奈何 hello world上遇到了问题。

问题如下：
声明了个hello函数，执行时候报错 entered--lisp error void-function hello，各种百度，bing，谷歌，无果。。。最后发现是个小问题，记录之
![]({{site.url}}/pics/emacs/1.jpg)

问题原因，hello方法需 **先执行一次，再调用** ！！！
如上所示，需要在第八行右侧括号处，C-x C-e执行，minibuffer中会出现hello标志，此时到第九行hello右侧括号处执行 C-x C-e 会正确执行。
