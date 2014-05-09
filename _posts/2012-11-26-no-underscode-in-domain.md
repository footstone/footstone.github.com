---
layout: post
title: 域名中不能含有下划线
---

###域名中不能含有下划线


小问题引起大问题，所以我觉得值得记录一下。

业务系统的SSO基于Cookie传递票据的方式实现。各个业务系统都在同一个大域下，如a.com。各个子业务系统分别定义自己的二级域名，如crm.a.com,billing.a.com等等。Cookie票据写在a.com下，于是各个子业务系统就都可以读取并认证。

SSO服务一直正常，但是某业务系统却一直无法接入认证。现象是用httpwatch可以看到cookie中的票据，但是程序中就是无法读到。兄弟团队从下到上为此纠结了好几天，最后扯皮推脱到我们这边，毕竟SSO服务是我们提供的，这事还得我们去查。

各种调试检查后发现，原来是因为该业务系统的二级域名中包含有下划线(_)，删除该下划线则一切正常！！

查阅资料后发现，有一个RFC952的规范确实定义了域名中不能含有下划线等字符，只能是字母、数字和短线（-）。

> RFC 952 - 美国国防部互联网主机表规范中的相关条文如下：
> 
> 1. A "name" (Net, Host, Gateway, or Domain name) is a text string up to 24 characters drawn from the alphabet (A-Z), digits (0-9), minus sign (-), and period (.)

