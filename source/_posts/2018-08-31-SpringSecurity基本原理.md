---
title: SpringSecurity基本原理
date: 2018-08-31 17:25:56
tags: [Spring, Java]
keywords: SpringSecurity基本原理
---
*SpringSecurity* 是一个能够基于 *Spring* 的企业应用系统提供声明式的安全框架<!--more-->
### 它能做什么
> __认证:__ 识别当前用户是谁
> __授权:__ 判断当前用户是否有权限进行相关操作
> __安全防护:__ 防止伪造身份

如果要对 Web 资源进行保护，最好的方法莫过于 Filter, 要项对方法调用进行保护，最好的方法莫过于AOP。

### 它是如何实现的
下面来简单的介绍一个 *SpringSecurity 基本原理* :
当一个请求要访问我们受 `SpringSecurity` 保护的 Web 资源的时候, 会先经过我们在 *Spring* 中设置的关于 *SpringSecurity* 的多层过滤器链,(如 `UsernamePasswordAuthenticationFilter`, `BasicAuthenticationFilter` 等),一个过滤器一个过滤器往下走,每通过一个过滤器的认证就会添加在请求中添加一个标记。如下图所示:
![](filter.png)
然后勒，这些 Filter 怎么和我们的数据库联系在一起勒？
	这些 *Filter* 并不直接处理用户的认证，也不直接处理用户的授权，而是把它们交给认证管理器和决策管理器


