---
title: String MVC 原理详解
category: springboot
---

> SpringMVC 是基于 Servlet 为核心，核心组件为DispatcherServlet。其基于Tomcat或Jetty或Undertow或WebSphere。

> 模型(Model)：模型表示应用程序的数据、业务逻辑和规则。它是系统的核心，负责管理数据、状态以及与数据库或外部服务的交互。
>
> 视图(View)：视图负责向用户呈现数据，是用户与应用程序交互的界面。
>
> 控制器(Controller)：控制器是模型和视图之间的中介，负责处理用户输入并协调模型与视图的交互。
>
> 工作原理：
>
> 1. 用户通过视图（如点击按钮、提交表单）与应用程序交互。
> 2. 控制器接收用户输入，解析请求，并调用相应的模型进行数据处理。
> 3. 模型执行必要的业务逻辑，更新数据状态，并通知视图需要更新。
> 4. 视图根据模型的最新数据重新渲染，呈现给用户。
> 5. 循环重复此过程，保持用户界面的动态更新。

# SpringBoot 加载 DispatcherServlet

**若按照XML的形式配置文件，需要在Tomcat中的web.xml中添加DispatcherServlet**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!-- 配置 DispatcherServlet -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- 指定 Spring MVC 配置文件路径（可选） -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring-mvc-config.xml</param-value>
        </init-param>
        <!-- 启动时加载 Servlet -->
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- 映射 DispatcherServlet 处理的 URL 模式 -->
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

此XML配置下，所有的请求都会被传递给`DispatcherServlet`处理。默认 DispatcherServlet 会加载 `WEB-INF/[DispatcherServlet的Servlet名字]-servlet.xml` 配 置 文 件 。 本 示 例 为 `/WEB-INF/spring-mvc-config.xml` 其中会注册一些bean类。

**若按照Java注解配置，SpringBoot会根据自动配置将`DispatcherServlet`传递给内置Tomcat**

在springboot启动过程中，会处理自动配置注解`@EnableAutoConfiguration`，其中通过`@Import(AutoConfigurationImportSelector.class)`注解执行`AutoConfigurationImportSelector.class`的`selectImports()`方法。在此方法中会将路径中所有的`org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件中的配置类读入。其中与MVC的有关的主要类是`org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration`和`org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration`和`org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration`

其中

* `WebMvcAutoConfiguration`中存储Spring Web MVC的一些默认配置；

* `DispatcherServletAutoConfiguration`中注册两个bean类`DispatcherServlet`和`DispatcherServletRegistrationBean` 后者中会注入前者，并在Servlet容器（如Tomcat）启动时会将前者的Sevlet类自动加载。即在内置Servlet容器时会自动执行`DispatcherServletRegistrationBean`的父类方法`ServletContextInitializer`的`onStartup()`方法，将`DispatcherServlet`注册到Servlet容器中； 

  SpringBoot启动过程中，因为是 web servlet 所以执行 `ServletWebServerApplicationContext.onRefresh().createWebServer()` 其中会调用`TomcatServletWebServerFactory.getWebServer()` 其中会创建 Tomcat类并在`prepareContext(tomcat.getHost(), initializers);`中将SpringBoot中的`ServletContextInitializer`注册进Tomcat中。

* `ServletWebServerFactoryAutoConfiguration`会将SpringBoot支持的Servlet容器根据条件自动导入，包括`EmbeddedTomcat` `EmbeddedJetty` `EmbeddedUndertow` 然后注册两个bean类`ServletWebServerFactoryCustomizer`和`ServletWebServerFactoryCustomizer`

# DispatcherServlet 处理请求

DispatcherServlet主要组成：

* HandlerMapping：映射请求
* HandlerAdapter：执行控制器
* ViewResolver：用于解析视图（如Thymeleaf 或 JSP）。
* HttpMessageConverter：支持JSON/XML的序列化和反序列化

当DispatcherServlet注册到Tomcat中后，每当请求进入后会自动执行DispatcherServlet的`Service()`方法，在其父类`FrameworkServlet`中

```java
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    if (HTTP_SERVLET_METHODS.contains(request.getMethod())) {
        super.service(request, response);
    }
    else {
        processRequest(request, response);
    }
}
```

最终都会执行`processRequest(request, response)`方法，其中会执行`doService(request, response)`方法，`DispatcherServlet.doService()`会执行`DispatcherServlet.doDispatch(request, response)`

主要步骤有：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    try {
        try {
            // Determine handler for the current request.
            mappedHandler = getHandler(processedRequest);
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
            // Actually invoke the handler.
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            applyDefaultViewName(processedRequest, mv);
        }
        catch (Exception ex) {
        }
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
    }
}

```

* `mappedHandler = getHandler(processedRequest);` 通过`handlerMappings`寻找符合条件的handler（即Controller）
* `getHandlerAdapter(mappedHandler.getHandler());` 找到适配`handler`的适配器`HandlerAdapter`
* `ha.handle(processedRequest, response, mappedHandler.getHandler());` 将 `handler` 传入适配的`HandlerAdapter`并执行其`hanlder()`方法，适配器调用处理器方法（如 `@RequestMapping` 方法），生成 `ModelAndView`
* `processDispatchResult()`：通过 `ViewResolver` 解析视图 和 统一处理异常，生成错误响应。

整体处理流程：

1. **Servlet 容器（Tomcat）的请求转发：**

- **Tomcat 接收请求**：客户端请求首先由 Tomcat 的 `Connector` 接收，解析 HTTP 协议后生成 `HttpServletRequest` 和 `HttpServletResponse` 对象。

- **Servlet 匹配**：Tomcat 根据 URL 映射规则，将请求转发到对应的 `Servlet`（此处为 `DispatcherServlet`）。

  

2.  **`DispatcherServlet` 的初始化流程：**

- **`service()` 方法**：Tomcat 调用 `DispatcherServlet` 的 `service()` 方法（继承自 `HttpServlet`）。

- **`doService()` 方法**：`DispatcherServlet` 重写了 `doService()`，在其中初始化一些上下文（如 `LocaleContext`、`RequestAttributes`），然后调用 `doDispatch()`。

  

3. **`doDispatch()` 的核心逻辑：**

   `doDispatch()` 负责以下关键步骤：

   1. **处理 Multipart 请求**（如文件上传）。
   2. **获取处理器链**（`HandlerExecutionChain`），包含目标处理器和拦截器。
   3. **调用拦截器的 `preHandle()`**。
   4. **执行处理器方法**（如 `@Controller` 中的方法），生成 `ModelAndView`。
   5. **调用拦截器的 `postHandle()`**。
   6. **处理结果**（渲染视图或处理异常）。

   

4. **视图渲染与响应写入**

- **`processDispatchResult()`**：在 `doDispatch()` 的末尾，调用此方法处理视图渲染或异常。
  - **视图渲染**：通过 `ViewResolver` 解析视图，调用 `View.render()` 将模型数据写入响应。
  - **异常处理**：通过 `HandlerExceptionResolver` 生成错误响应。

# 请求类型

## GET

```java
@RequestMapping(value="/create", method = RequestMethod.GET)
```

```java
@GetMapping("/create")
```

## POST

```java
@RequestMapping(value="/create", method = RequestMethod.POST)
```

```java
@PostMapping("/create")
```



其他请求类型限制：

```java
@RequestMapping(value = "/user", params = "id") // 必须有id参数
@RequestMapping(value = "/user", params = "!id") // 不能有id参数
@RequestMapping(value = "/user", params = "id=123") // id参数必须为123

@RequestMapping(value = "/user", headers = "content-type=text/*") // 匹配特定Content-Type
@RequestMapping(value = "/user", headers = "!X-Custom-Header") // 不能包含特定头

@RequestMapping(value = "/user", consumes = "application/json") // 只处理Content-Type为JSON的请求
@RequestMapping(value = "/user", produces = "application/json") // 只产生JSON响应
```

# 请求数据格式

## Form格式

前端代码

```html
<form action="/login" method="post">
    <label for="username">账号:</label>
    <input type="text" id="username" name="username" required>
    <label for="password">密码:</label>
    <input type="password" id="password" name="password" required>
    <button type="submit">登录</button>
</form>
```

传输数据

```
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=user123&password=pass123
```

后端代码

```java
@PostMapping("/login_form")
public ResponseEntity<String> loginForm(@ModelAttribute LoginRequest loginRequest) {
    String username = loginRequest.getUsername();
    String password = loginRequest.getPassword();

    // 模拟验证逻辑（实际中应调用服务层验证）
    if (username != null && password != null && !username.isEmpty() && !password.isEmpty()) {
        // 假设验证通过
        return ResponseEntity.success("登录成功，用户: " + username);
    } else {
        return ResponseEntity.badRequest("账号或密码无效");
    }
}
```

## JSON格式

JavaScript代码

```javascript
fetch('/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        username: 'user123',
        password: 'pass123'
    })
})
.then(response => response.text())
.then(data => console.log(data));
```

传输数据

```
POST /login HTTP/1.1
Content-Type: application/json

{"username":"user123","password":"pass123"}
```

后端代码

```java
@PostMapping("/login")
public ResponseEntity<String> login(@RequestBody LoginRequest loginRequest) {
    String username = loginRequest.getUsername();
    String password = loginRequest.getPassword();

    // 模拟验证逻辑（实际中应调用服务层验证）
    if (username != null && password != null && !username.isEmpty() && !password.isEmpty()) {
        // 假设验证通过
        return ResponseEntity.success("登录成功，用户: " + username);
    } else {
        return ResponseEntity.badRequest("账号或密码无效");
    }
}
```

# 请求路径格式

## 普通 URL 路径映射

`@RequestMapping(value={"/test1", "/user/create"})`

## URI 模板模式映射

* `@RequestMapping(value="/users/{userId}")`
*  `@RequestMapping(value="/users/{userId}/create")` 
* `@RequestMapping(value="/users/{userId}/topics/{topicId}")` 

需要通过`@PathVariable`来获取URL中参数。

```java
@RestController
@RequestMapping("/api")
public class ExampleController {

    @GetMapping("/users/{id}")
    public String getUserById(@PathVariable Long id) {
        return "User ID: " + id;
    }
    
    @GetMapping("/users/{userId}/orders/{orderId}")
    public String getOrderDetails(@PathVariable Long userId, @PathVariable Long orderId) {
        return "User: " + userId + ", Order: " + orderId;
    }
}
```

## Ant 风格的 URL 路径映射

* `@RequestMapping(value="/users/**")`：可以匹配`/users/abc/abc`，但`/users/123`将会被【URI模板模式映射 中的`/users/{Id}`模式优先映射到】
* `@RequestMapping(value="/product?")`：可匹配`/product1`或`/producta`，但不匹配`/product`或`/productaa`
* `@RequestMapping(value="/product*")`：可匹配`/productabc`或`/product`，但不匹配`/productabc/abc`
* `@RequestMapping(value="/product/*")`：可匹配`/product/abc`，但不匹配`/productabc`
* `@RequestMapping(value="/products/**/{productId}")`：可匹配`/products/abc/abc/123`或`/products/123`，也就是Ant风格和URI模板变量风格可混用

## 	正则表达式风格的 URL 路径映射

`@RequestMapping(value="/products/{categoryCode:\\d+}-{pageNumber:\\d+}")`：可 以 匹 配 `/products/123-1`，但不能匹配`/products/abc-1`，这样可以设计更加严格的规则。

`@RequestMapping("/{textualPart:[a-z-]+}-{numericPart:[\\d]+}")`



# SpringMVC注解

## **控制器相关注解**

- `@Controller`
  - 标记一个类为 Spring MVC 控制器，负责处理 HTTP 请求。
  - 示例：@Controller public class MyController { ... }
- `@RestController`
  - 组合注解，等价于 @Controller + @ResponseBody，表示控制器方法返回的对象直接序列化为 JSON 或 XML。
  - 示例：@RestController public class ApiController { ... }
- `@Component`
  - 通用注解，可用于控制器类，纳入 Spring 容器管理（通常搭配其他注解使用）。
- `@RequestMapping`
  - 映射 HTTP 请求到控制器方法或类，支持 GET、POST 等方法。
  - 属性：value（路径）、method（请求方法）、produces（响应类型）、consumes（请求类型）。
  - 示例：@RequestMapping(value = "/home", method = RequestMethod.GET)
- `@GetMapping`、`@PostMapping`、`@PutMapping`、`@DeleteMapping`、`@PatchMapping`
  - @RequestMapping 的快捷方式，分别对应 HTTP 的 GET、POST、PUT、DELETE 和 PATCH 请求。
  - 示例：@GetMapping("/users")

## **请求参数与路径处理注解**

- `@PathVariable`
  - 从 URL 路径中提取变量，绑定到方法参数。
  - 示例：@GetMapping("/user/{id}") public String getUser(@PathVariable Long id)
- `@RequestParam`
  - 从请求参数（查询字符串或表单数据）中提取值，绑定到方法参数。
  - 属性：name（参数名）、required（是否必须）、defaultValue（默认值）。
  - 示例：@RequestParam(value = "name", defaultValue = "Guest") String name
- `@RequestBody`
  - 将 HTTP 请求的正文（如 JSON 或 XML）绑定到方法参数，通常用于 RESTful API。
  - 示例：@PostMapping("/user") public void saveUser(@RequestBody User user)
- `@RequestHeader`
  - 从 HTTP 请求头中提取值，绑定到方法参数。
  - 示例：@RequestHeader("User-Agent") String userAgent
- `@CookieValue`
  - 从请求的 Cookie 中提取值，绑定到方法参数。
  - 示例：@CookieValue("sessionId") String sessionId
- `@MatrixVariable`
  - 从 URL 路径中的矩阵变量（如 /path;key=value）提取值。
  - 示例：@GetMapping("/data/{path}") public String getMatrix(@MatrixVariable String key)

## **响应处理注解**

- `@ResponseBody`
  - 表示方法返回值直接作为 HTTP 响应正文，通常序列化为 JSON 或 XML。
  - 示例：@ResponseBody public User getUser()
- `@ResponseStatus`
  - 指定控制器方法或异常处理方法的 HTTP 状态码。
  - 示例：@ResponseStatus(HttpStatus.CREATED)
- `@ModelAttribute`
  - 将方法返回值或参数绑定到模型对象，供视图使用；也可用于方法级，预填充模型数据。
  - 示例：@ModelAttribute("user") public User getUser()

## **异常处理注解**

- `@ExceptionHandler`
  - 标记方法用于处理特定异常，限定在控制器内部。
  - 示例：@ExceptionHandler(NullPointerException.class) public ResponseEntity handleException()
- `@ControllerAdvice`
  - 定义全局异常处理、模型增强或绑定器，作用于所有控制器。
  - 示例：@ControllerAdvice public class GlobalExceptionHandler { ... }

## **跨域与配置相关注解**

- `@CrossOrigin`
  - 启用跨域资源共享（CORS），可用于类或方法级别。
  - 属性：origins（允许的域名）、methods（允许的请求方法）。
  - 示例：@CrossOrigin(origins = "http://example.com")
- `@SessionAttributes`
  - 指定模型属性存储在会话（Session）中，作用于类级别。
  - 示例：@SessionAttributes("user")
- `@InitBinder`
  - 标记方法用于自定义数据绑定或验证逻辑，作用于控制器内部。
  - 示例：@InitBinder public void initBinder(WebDataBinder binder)

## **其他高级注解**

- `@SessionAttribute`
  - 从会话中获取属性值，绑定到方法参数。
  - 示例：@SessionAttribute("user") User user
- `@RequestAttribute`
  - 从请求属性中获取值，绑定到方法参数。
  - 示例：@RequestAttribute("data") String data
- `@EnableWebMvc`
  - 启用 Spring MVC 配置，通常用于自定义 MVC 配置类。
  - 示例：@EnableWebMvc @Configuration public class WebConfig { ... }

# 建议

对于前后端分离的项目可以选择RESTful风格的API，使用JSON作为传递类型。通过`@RequestBody`（将HTTP的body使用JSON反序列化为Java对象） 和 `@ResponseBody` （将返回数据使用JSON序列化并赋值到HTTP的body中）加载请求和响应。

对于文件等二进制传输，使用`Multipart/form-data`格式传输：

前端上传代码：

```react
const formData = new FormData();
formData.append('username', 'john');
formData.append('avatar', fileInput.files[0]); // 文件

// 使用 Fetch API 发送
fetch('/api/upload', {
  method: 'POST',
  body: formData
  // 不需要设置 Content-Type，浏览器会自动设置
});
```

