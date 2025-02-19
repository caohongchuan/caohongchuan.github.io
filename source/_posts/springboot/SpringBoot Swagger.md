---
title: SpringBoot Swagger
category: springboot
---

# 依赖

SpringBoot依赖[springdoc-openapi](https://springdoc.org/)对接口API进行注解

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.8.5</version>
</dependency>
```

配置信息`application.yml`中添加swagger-ui

```yaml
springdoc:
    swagger-ui:
      path: swagger-ui.html
```

启动项目后就可以通过 `https://localhost:8443/swagger-ui.html` 或者`https://localhost:8443/swagger-ui/index.html#/`访问后端接口信息。

# Swagger 注解

[Swagger Annotation Tutorial](https://blog.csdn.net/qq_52774158/article/details/131081371)

## @Tag

API分组

```java
@Tag(name = "OAuth2 Authorization", description = "OAuth2 登陆接口")
```

## @Operation

用于描述API操作

* summary：操作的摘要信息。
* description：操作的详细描述。
* tags：指定 @Tag 注解的对象数组，用于将操作归类到特定的分组。
* parameters：指定 @Parameter 注解的对象数组，用于描述操作的输入参数
* responses：指定 @ApiResponse 注解的对象数组，用于描述操作的响应结果。
* requestBody：指定 @RequestBody 注解的对象，用于描述操作的请求体。

```java
@Operation(summary = "跳转到 OAuth2 授权页面", description = "两次重定向，跳转到Github登陆页面，登陆后与后端交换Token，最后跳转到origin_url")
```

## @Parameter

```java
@Parameter(name = "origin_url", required = true, description = "OAuth2流程结束后会跳转到该URL")
```

## @ApiResponse

```java
@ApiResponse(responseCode = "301", description = "两次跳转到Github登陆窗口")
```

## @OpenAPIDefinition

```java
@OpenAPIDefinition(
        info = @Info(
                title = "User Manager API",
                version = "1.0.0",
                description = "用户信息管理API",
                contact = @Contact(
                        name = "yingzheng",
                        email = "yingzhengttt@gmail.com"
                )
        )
)
```

