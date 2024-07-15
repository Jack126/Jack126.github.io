---
layout: post
title: "SqlSession was not registered for synchronization because synchronization is not active事务开启失败"
category: Language
tags: JAVA
date: 2021-11-26 12:00
---

学习ssm初始阶段，遇到一个mybatis-config 打印logImpl 报错问题，如下：
![]({{site.url}}/pics/java/1.png)
spring-sevice.xml 配置如下
![]({{site.url}}/pics/java/2.png)

解决方法：
在在service层里依赖注入进去@Transactional
![]({{site.url}}/pics/java/3.png)

最后运行如下：
![]({{site.url}}/pics/java/4.png)
