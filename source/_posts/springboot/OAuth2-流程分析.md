---
title: Spring OAuth2 Client流程分析
category: SpringBoot
---

> 目标是实现前后端分离（BFF）的OAuth2登陆流程。需要详细分析Spring OAuth2 Client认证流程，才能对后端的认证进行修改满足前后端分离的要求。

# Spring OAuth 配置

## 1. 依赖（Dependency)

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

## 2. OAuth2 Server Configuration (application.yml)

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

## 3. Config `SecurityFilterChain` 

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

![login_window](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250103195400116.png)

我们发现登陆的API为`localhost:8080/oauth2/authorization/github`，**这也是我们前端需要跳转的页面API地址**。

# 授权流程

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

登陆完成后，Github OAuth Server会重定向到`https://github.com/login/oauth/authorize?` 然后Github会根据之前在Spring OAuth中设置的`Authorization callback URL`并跳转到该地址。下图是本文在Github中的设置：

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250219171534862.png" alt="image-20250219171534862" style="zoom:50%;" />

即后续跳转到`https://localhost:8443/login/oauth2/code/github?code=3fd7543e7cf92aab5d53&state=1NiT_GBSj2pOolkHnTHpSft8V9x6uOMTsvYveTHM6GE%3D`

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

# SpringBoot OAuth2 Client前后端分离

经过上述分析，前端只需要访问`localhost:8080/oauth2/authorization/github`即可开始认证过程。只要设置了`http.oauth2Login()`，SpringSecurity 即会保证`/oauth2/authorization/{client}`认证端点对外开放。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorizeRequests -> authorizeRequests
                        .anyRequest().authenticated())
                .oauth2Login(oauth2 -> oauth2
                        .authorizationEndpoint();
        return http.build();
    }
}
```

但前端调用该接口是需要添加参数，告诉后端处理完成之后需要跳转回哪里。所以目前需要在Spring Security OAuth中完成两项工作：

* 修改拦截`localhost:8080/oauth2/authorization/github?origin_url=xxx`的拦截器，将其中的参数`origin_url`取出存到Session中。
* 登陆完成后，从Session取出之前的`origin_url`并将用户从Github拿到的基本信息添加到该URL中进行跳转，返回到原来的页面。

通过上一章节的页面调试，可以看到针对`localhost:8080/oauth2/authorization/github?origin_url=xxx`的处理是过滤连中`OAuth2AuthorizationRequestRedirectFilter`处理的。

当Github根据提前设置的`Authorization callback URL`将code返回时，是过滤链中`OAuth2LoginAuthenticationFilter`处理的，处理过程包括1.接收Github传回的code。2.使用该code向Github发起POST请求来换取Token，最后接收到Token。 3.根据Token来获取Github用户基本信息。

处理完成后跳转回最初的访问地址（此处需要更改为 从Session中拿出`origin_url`并跳转）。

## 设置拦截器，拦截`/oauth2/authorization/github?origin_url=xxx`的参数

> 本文通过重写`AuthorizationRequestRepository`将`origin_url`写入Session。
>
> 也可以通过重写`AuthorizationRequestResolver`来解析`origin_url`，在解析选择哪种数据源的同时将`origin_url`封装到`OAuth2AuthorizationRequest`类里，后续`AuthorizationRequestRepository`会将该类直接放到Session中，最后需要通过获取Session中的`OAuth2AuthorizationRequest`对象，再从里面获取`origin_url`。读者可以自行尝试。

经过调试`OAuth2AuthorizationRequestRedirectFilter`，发现只要是authorization_code认证方式的OAuth请求，都会经过`HttpSessionOAuth2AuthorizationRequestRepository`的处理。

```java
// OAuth2AuthorizationRequestRedirectFilter.java
private void sendRedirectForAuthorization(HttpServletRequest request, HttpServletResponse response,
        OAuth2AuthorizationRequest authorizationRequest) throws IOException {
    if (AuthorizationGrantType.AUTHORIZATION_CODE.equals(authorizationRequest.getGrantType())) {
        this.authorizationRequestRepository.saveAuthorizationRequest(authorizationRequest, request, response);
    }
    this.authorizationRedirectStrategy.sendRedirect(request, response,
            authorizationRequest.getAuthorizationRequestUri());
}
```

其中的`this.authorizationRequestRepository`默认是`HttpSessionOAuth2AuthorizationRequestRepository`。它的源码显示其专门在Session中存储请求信息：

```java
// HttpSessionOAuth2AuthorizationRequestRepository.java
/**
 * An implementation of an {@link AuthorizationRequestRepository} that stores
 * {@link OAuth2AuthorizationRequest} in the {@code HttpSession}.
 */
public final class HttpSessionOAuth2AuthorizationRequestRepository
		implements AuthorizationRequestRepository<OAuth2AuthorizationRequest> {}
```

所以，可以自定义该类来解析Request请求中的参数`origin_url`。查看[Spring Security OAuth2 Client文档](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorization-grants.html#oauth2-client-authorization-code-authorization-request-repository) 可以配置自定义的`AuthorizationRequestRepository`。

```java
@Configuration
@EnableWebSecurity
public class OAuth2ClientSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
            .oauth2Login(oauth2 -> oauth2
                .authorizationEndpoint(endpoint -> endpoint
                    .authorizationRequestRepository(new MyAuthorizationRequestRepository())
                    // ...
                )
            );
			return http.build();
	}
}
```

目的只是为了增加一个解析URL参数，所以核心代码仍然使用它默认的`HttpSessionOAuth2AuthorizationRequestRepository`，只在`saveAuthorizationRequest()`方法中加入对`origin_url`的解析。

定义类`ParseUrlHttpSessionOAuth2AuthorizationRequestRepository`，将`HttpSessionOAuth2AuthorizationRequestRepository`的内容完全复制过来，将Class名字更改，并在`saveAuthorizationRequest()`中加入三行代码：

```java
// set url params to session
String originUrl = request.getParameter("origin_url");
Assert.hasText(originUrl, "authorizationRequest.origin_url cannot be empty");
request.getSession().setAttribute("origin_url", originUrl);
```

解析URL获取`origin_url`，并将URL存入到Session中。

```java
// ParseUrlHttpSessionOAuth2AuthorizationRequestRepository.java
public class ParseUrlHttpSessionOAuth2AuthorizationRequestRepository implements AuthorizationRequestRepository<OAuth2AuthorizationRequest> {
    private static final String DEFAULT_AUTHORIZATION_REQUEST_ATTR_NAME = HttpSessionOAuth2AuthorizationRequestRepository.class
            .getName() + ".AUTHORIZATION_REQUEST";

    private final String sessionAttributeName = DEFAULT_AUTHORIZATION_REQUEST_ATTR_NAME;

    @Override
    public OAuth2AuthorizationRequest loadAuthorizationRequest(HttpServletRequest request) {
		...
    }

    @Override
    public void saveAuthorizationRequest(OAuth2AuthorizationRequest authorizationRequest, HttpServletRequest request,
                                         HttpServletResponse response) {
        Assert.notNull(request, "request cannot be null");
        Assert.notNull(response, "response cannot be null");
        if (authorizationRequest == null) {
            removeAuthorizationRequest(request, response);
            return;
        }
        String state = authorizationRequest.getState();
        Assert.hasText(state, "authorizationRequest.state cannot be empty");

        // set url params to session
        String originUrl = request.getParameter("origin_url");
        Assert.hasText(originUrl, "authorizationRequest.origin_url cannot be empty");
        request.getSession().setAttribute("origin_url", originUrl);

        request.getSession().setAttribute(this.sessionAttributeName, authorizationRequest);
    }

    @Override
    public OAuth2AuthorizationRequest removeAuthorizationRequest(HttpServletRequest request,
                                                                 HttpServletResponse response) {
		...
    }


    private String getStateParameter(HttpServletRequest request) {
        return request.getParameter(OAuth2ParameterNames.STATE);
    }

    private OAuth2AuthorizationRequest getAuthorizationRequest(HttpServletRequest request) {
 	...
    }
}
```

## 设置登陆成功后跳转回制定URL(`origin_url`)





```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.oauth2Login(oauth2 -> oauth2
                        .authorizationEndpoint(authorization -> authorization
                                .authorizationRequestRepository(new ParseUrlHttpSessionOAuth2AuthorizationRequestRepository()))
                        .successHandler(
                                (request, response, authentication) -> {
                                    String originUrl = (String) request.getSession().getAttribute("origin_url");
                                    response.sendRedirect(originUrl);
                                })
                )
        );
        return http.build();
    }

}
```





```
2025-02-19T13:38:16.582+08:00  INFO 28513 --- [on(4)-127.0.0.1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2025-02-19T13:38:16.583+08:00  INFO 28513 --- [on(4)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2025-02-19T13:38:16.586+08:00  INFO 28513 --- [on(4)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2025-02-19T13:43:35.633+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Trying to match request against DefaultSecurityFilterChain defined as 'securityFilterChain' in [class path resource [top/chc/usermanager/config/SecurityConfig.class]] matching [any request] and having filters [DisableEncodeUrl, ChannelProcessing, WebAsyncManagerIntegration, SecurityContextHolder, HeaderWriter, Cors, Logout, OAuth2AuthorizationRequestRedirect, OAuth2LoginAuthentication, BasicAuthentication, SecurityContextHolderAwareRequest, AnonymousAuthentication, ExceptionTranslation, Authorization] (1/1)
2025-02-19T13:43:35.634+08:00 DEBUG 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Securing GET /oauth2/authorization/github?origin_url=localhost:8443
2025-02-19T13:43:35.635+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking DisableEncodeUrlFilter (1/14)
2025-02-19T13:43:35.638+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking ChannelProcessingFilter (2/14)
2025-02-19T13:43:35.638+08:00 DEBUG 28513 --- [nio-8443-exec-2] o.s.s.w.a.c.ChannelProcessingFilter      : Request: filter invocation [GET /oauth2/authorization/github?origin_url=localhost:8443]; ConfigAttributes: [REQUIRES_SECURE_CHANNEL]
2025-02-19T13:43:35.638+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking WebAsyncManagerIntegrationFilter (3/14)
2025-02-19T13:43:35.640+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking SecurityContextHolderFilter (4/14)
2025-02-19T13:43:35.641+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking HeaderWriterFilter (5/14)
2025-02-19T13:43:35.642+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking CorsFilter (6/14)
2025-02-19T13:43:35.643+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking LogoutFilter (7/14)
2025-02-19T13:43:35.643+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.s.w.a.logout.LogoutFilter            : Did not match request to Or [Ant [pattern='/logout', GET], Ant [pattern='/logout', POST], Ant [pattern='/logout', PUT], Ant [pattern='/logout', DELETE]]
2025-02-19T13:43:35.644+08:00 TRACE 28513 --- [nio-8443-exec-2] o.s.security.web.FilterChainProxy        : Invoking OAuth2AuthorizationRequestRedirectFilter (8/14)
2025-02-19T13:43:35.687+08:00 DEBUG 28513 --- [nio-8443-exec-2] o.s.s.web.DefaultRedirectStrategy        : Redirecting to https://github.com/login/oauth/authorize?response_type=code&client_id=Ov23lisFyLM8ev2mgeX1&scope=read:user&state=1NiT_GBSj2pOolkHnTHpSft8V9x6uOMTsvYveTHM6GE%3D&redirect_uri=https://localhost:8443/login/oauth2/code/github
2025-02-19T13:44:50.058+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Trying to match request against DefaultSecurityFilterChain defined as 'securityFilterChain' in [class path resource [top/chc/usermanager/config/SecurityConfig.class]] matching [any request] and having filters [DisableEncodeUrl, ChannelProcessing, WebAsyncManagerIntegration, SecurityContextHolder, HeaderWriter, Cors, Logout, OAuth2AuthorizationRequestRedirect, OAuth2LoginAuthentication, BasicAuthentication, SecurityContextHolderAwareRequest, AnonymousAuthentication, ExceptionTranslation, Authorization] (1/1)
2025-02-19T13:44:50.058+08:00 DEBUG 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Securing GET /login/oauth2/code/github?code=3fd7543e7cf92aab5d53&state=1NiT_GBSj2pOolkHnTHpSft8V9x6uOMTsvYveTHM6GE%3D
2025-02-19T13:44:50.058+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking DisableEncodeUrlFilter (1/14)
2025-02-19T13:44:50.059+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking ChannelProcessingFilter (2/14)
2025-02-19T13:44:50.059+08:00 DEBUG 28513 --- [io-8443-exec-10] o.s.s.w.a.c.ChannelProcessingFilter      : Request: filter invocation [GET /login/oauth2/code/github?code=3fd7543e7cf92aab5d53&state=1NiT_GBSj2pOolkHnTHpSft8V9x6uOMTsvYveTHM6GE%3D]; ConfigAttributes: [REQUIRES_SECURE_CHANNEL]
2025-02-19T13:44:50.059+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking WebAsyncManagerIntegrationFilter (3/14)
2025-02-19T13:44:50.059+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking SecurityContextHolderFilter (4/14)
2025-02-19T13:44:50.059+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking HeaderWriterFilter (5/14)
2025-02-19T13:44:50.060+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking CorsFilter (6/14)
2025-02-19T13:44:50.060+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking LogoutFilter (7/14)
2025-02-19T13:44:50.061+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.s.w.a.logout.LogoutFilter            : Did not match request to Or [Ant [pattern='/logout', GET], Ant [pattern='/logout', POST], Ant [pattern='/logout', PUT], Ant [pattern='/logout', DELETE]]
2025-02-19T13:44:50.061+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking OAuth2AuthorizationRequestRedirectFilter (8/14)
2025-02-19T13:44:50.062+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.security.web.FilterChainProxy        : Invoking OAuth2LoginAuthenticationFilter (9/14)
2025-02-19T13:44:50.102+08:00 TRACE 28513 --- [io-8443-exec-10] o.s.s.authentication.ProviderManager     : Authenticating request with OAuth2LoginAuthenticationProvider (1/3)
2025-02-19T13:44:51.534+08:00 TRACE 28513 --- [io-8443-exec-10] s.CompositeSessionAuthenticationStrategy : Preparing session with ChangeSessionIdAuthenticationStrategy (1/1)
2025-02-19T13:44:51.539+08:00 DEBUG 28513 --- [io-8443-exec-10] .s.ChangeSessionIdAuthenticationStrategy : Changed session id from 2cb1fec3-dba6-4a8f-8376-40934da49d8d
2025-02-19T13:44:51.541+08:00 DEBUG 28513 --- [io-8443-exec-10] w.c.HttpSessionSecurityContextRepository : Stored SecurityContextImpl [Authentication=OAuth2AuthenticationToken [Principal=Name: [42151586], Granted Authorities: [[OAUTH2_USER, SCOPE_read:user]], User Attributes: [{login=caohongchuan, id=42151586, node_id=MDQ6VXNlcjQyMTUxNTg2, avatar_url=https://avatars.githubusercontent.com/u/42151586?v=4, gravatar_id=, url=https://api.github.com/users/caohongchuan, html_url=https://github.com/caohongchuan, followers_url=https://api.github.com/users/caohongchuan/followers, following_url=https://api.github.com/users/caohongchuan/following{/other_user}, gists_url=https://api.github.com/users/caohongchuan/gists{/gist_id}, starred_url=https://api.github.com/users/caohongchuan/starred{/owner}{/repo}, subscriptions_url=https://api.github.com/users/caohongchuan/subscriptions, organizations_url=https://api.github.com/users/caohongchuan/orgs, repos_url=https://api.github.com/users/caohongchuan/repos, events_url=https://api.github.com/users/caohongchuan/events{/privacy}, received_events_url=https://api.github.com/users/caohongchuan/received_events, type=User, user_view_type=private, site_admin=false, name=Zheng Ying, company=null, blog=blog.caohongchuan.com, location=null, email=null, hireable=null, bio=Java/nlp, twitter_username=null, notification_email=null, public_repos=13, public_gists=0, followers=0, following=0, created_at=2018-08-06T17:32:16Z, updated_at=2025-02-19T05:44:49Z, private_gists=0, total_private_repos=2, owned_private_repos=2, disk_usage=28310, collaborators=0, two_factor_authentication=true, plan={name=free, space=976562499, collaborators=0, private_repos=10000}}], Credentials=[PROTECTED], Authenticated=true, Details=WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=2cb1fec3-dba6-4a8f-8376-40934da49d8d], Granted Authorities=[OAUTH2_USER, SCOPE_read:user]]] to HttpSession [org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper$HttpSessionWrapper@66bfb1a7]
2025-02-19T13:44:51.542+08:00 DEBUG 28513 --- [io-8443-exec-10] .s.o.c.w.OAuth2LoginAuthenticationFilter : Set SecurityContextHolder to OAuth2AuthenticationToken [Principal=Name: [42151586], Granted Authorities: [[OAUTH2_USER, SCOPE_read:user]], User Attributes: [{login=caohongchuan, id=42151586, node_id=MDQ6VXNlcjQyMTUxNTg2, avatar_url=https://avatars.githubusercontent.com/u/42151586?v=4, gravatar_id=, url=https://api.github.com/users/caohongchuan, html_url=https://github.com/caohongchuan, followers_url=https://api.github.com/users/caohongchuan/followers, following_url=https://api.github.com/users/caohongchuan/following{/other_user}, gists_url=https://api.github.com/users/caohongchuan/gists{/gist_id}, starred_url=https://api.github.com/users/caohongchuan/starred{/owner}{/repo}, subscriptions_url=https://api.github.com/users/caohongchuan/subscriptions, organizations_url=https://api.github.com/users/caohongchuan/orgs, repos_url=https://api.github.com/users/caohongchuan/repos, events_url=https://api.github.com/users/caohongchuan/events{/privacy}, received_events_url=https://api.github.com/users/caohongchuan/received_events, type=User, user_view_type=private, site_admin=false, name=Zheng Ying, company=null, blog=blog.caohongchuan.com, location=null, email=null, hireable=null, bio=Java/nlp, twitter_username=null, notification_email=null, public_repos=13, public_gists=0, followers=0, following=0, created_at=2018-08-06T17:32:16Z, updated_at=2025-02-19T05:44:49Z, private_gists=0, total_private_repos=2, owned_private_repos=2, disk_usage=28310, collaborators=0, two_factor_authentication=true, plan={name=free, space=976562499, collaborators=0, private_repos=10000}}], Credentials=[PROTECTED], Authenticated=true, Details=WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=2cb1fec3-dba6-4a8f-8376-40934da49d8d], Granted Authorities=[OAUTH2_USER, SCOPE_read:user]]
```



