---
title: Spring面试合集
category: interview
---



## Spring 持久化框架

Spring提供了**JDBC（Java Database Connectivity）**，**JPA (Java Persistence API)**，**MyBatis**。  

**JDBC（Java Database Connectivity）** 是Java的标准数据库访问技术。它直接与数据库交互，允许Java应用程序执行SQL语句并与数据库进行通信。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
    <version>3.4.2</version>
</dependency>
```

**JPA** 是一个Java的持久化API，用于管理Java对象与数据库之间的映射。它提供了对象关系映射（ORM）的功能，允许开发者以面向对象的方式处理数据。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <version>3.4.2</version>
</dependency>
```

**MyBatis** 是一个支持普通SQL查询、存储过程和高级映射的持久层框架。它避免了几乎所有的JDBC代码和手动设置参数以及获取结果集。MyBatus是**半自动ORM**：MyBatis不像JPA那样完全抽象掉SQL，它允许开发者写SQL语句，同时提供了映射文件来定义SQL语句与Java对象的映射关系。

就目前版本的mybatis中包含了`spring-boot-starter-jdbc`，直接使用JDBC的Datasource和数据库连接池。

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.4</version>
</dependency>
```



## SpringBoot 数据库连接池



## Spring IOC 循环依赖
