---
title: Spring Session
category: Spring
---

# Spring Session

> Spring Session 是 Spring 框架的一个项目，旨在提供会话管理的解决方案。它可以与各种后端存储（如内存、数据库、Redis 等）集成。

* SecurityContextHolderFilter中的SecurityContextRepository，存储用户信息上下文，其中一种实现方式就是HttpSessionSecurityContextRepository，而且该方式是默认的实现方式。
* AbstractAuthenticationProcessingFilter中，当用户登录成功，也是会调用this.securityContextRepository.saveContext(context, request, response);存储上下文到session中。

