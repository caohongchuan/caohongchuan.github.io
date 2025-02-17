---
title: OAuth2 流程分析
category: Spring
---

# Spring OAuth2 Client流程分析

> 目标是实现前后端分离（BFF）的OAuth2登陆流程。需要详细分析Spring OAuth2 Client认证流程，才能对后端的认证进行修改满足前后端分离的要求。

## Spring OAuth 配置

### 1. 依赖（Dependency)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 2. OAuth2 Server Configuration (application.yml)

```yaml
spring:
    security:
        oauth2:
            client:
                registration:
                    github:
                        client-id: Ov23lisFyLM8ev2mgeX1
                        client-secret: da386df22b42d1374216995f47c3d05d441b3da6
                    google:
                        client-id: google-client-id
                        client-secret: google-client-secret
                    facebook:
                        client-id: okta-client-id
                        client-secret: okta-client-secret

#为了更好的了解后端认证流程，将TRACE log打印出来                        
logging:
    level:
        org:
            springframework:
                security: TRACE
```

### 3. Config `SecurityFilterChain` 

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorizeRequests -> authorizeRequests
                        .anyRequest().authenticated())
                .oauth2Login(Customizer.withDefaults());

        return http.build();
    }

}
```

到目前为止，默认的Spring security OAuth2已经配置完成了。启动项目并访问`localhost:8080` 会自动跳转到登陆窗口 `localhost:8080/login`

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250103195400116.png" alt="image-20250103195400116" style="zoom:50%;" />

我们发现登陆的API为`localhost:8080/oauth2/authorization/github`，**这也是我们前端需要跳转的页面API地址**。

## 授权流程

> 当访问 `localhost:8080` 跳转到 `localhost:8080/login` 是由于在 SecurityFilterChain 中所有的页面都需要认证，当访问`\`时，被`AuthorizationFilter`抛出异常，并被`ExceptionTranslationFilter`中的`LoginUrlAuthenticationEntryPoint`处理 并重定向到 `localhost:8080/login`。 访问`localhost:8080/login`的时候，会被`DefaultLoginPageGeneratingFilter`拦截并返回Login页面。
>
> 该过程在前端的核心认证过程中用不到，就不展开了。但在规范API的时候可以自定义AuthenticationEntryPoint，将错误API统一返回为JSON格式，而不是返回`/login`页面。

### 1. 访问`localhost:8080/oauth2/authorization/github`

当浏览器访问该地址后会自动跳转到Github的Login页面，此时后端日志停在了`OAuth2AuthorizationRequestRedirectFilter (7/16)`，也就是请求被该类处理了并发起了Redirecting to ` https://github.com/login/oauth/authorize?`的页面跳转。	

![image-20250103202726604](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250103202726604.png)

通过前端调试Network可以看到，当访问`localhost:8080/oauth2/authorization/github`后端会发送重定向请求，将Response的Location设置为`https://github.com/login/oauth/authorize?`并进行跳转。

![image-20250103203153119](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250103203153119.png)

访问`https://github.com/login/oauth/authorize?`后再次跳转到`https://github.com/login?` 

![](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250103213105419.png)

登陆完成后，Github OAuth Server会重定向到`http://localhost:8080/login/oauth2/code/github?code=96176784ee27955498f1&state=f7SZ6frH0MDPHI_HlJaJWOWp-8lUd5bcJWEV7WndZX0%3D`。

此时后端日志中可以看到过滤链停在了`OAuth2LoginAuthenticationFilter (8/16)`，该类会接受code和state参数并进行处理。

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250104112126452.png" alt="image-20250104112126452" style="zoom: 67%;" />

在该类结束的时候会重定向到`/`页面。

```
2025-01-03T23:55:11.589+08:00 DEBUG 46026 --- [nio-8080-exec-4] o.s.s.web.DefaultRedirectStrategy        : Redirecting to /
2025-01-03T23:55:11.590+08:00 TRACE 46026 --- [nio-8080-exec-4] o.s.s.w.header.writers.HstsHeaderWriter  : Not injecting HSTS header since it did not match request to [Is Secure]
```

进入到类`OAuth2LoginAuthenticationFilter`，发现请求会进入到`attemptAuthentication()`方法中。该类主要完成以下步骤：

* `redirect_uri`的请求会被过滤器`OAuth2LoginAuthenticationFilter`拦截处理
* 向Github发送`{token-uri}/login/oauth/access_token`请求并获取token
* 将获取的token封装到`OAuth2AuthenticationToken`中并返回该值
* 设置Session
* `SecurityContextHolder` 存储 `OAuth2AuthenticationToken`
* 从Session拿出之前访问的受限地址并重定向到该地址。





