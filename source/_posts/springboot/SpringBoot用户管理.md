---
title: SpringBoot用户管理
category: springboot
---

# 用户密码登陆

**注册**：

1. 前端：用户输入用户名和密码，通过 HTTPS 发送到后端。
2. 后端：验证输入，生成盐值，哈希密码，存储用户名和哈希值到数据库。
3. 数据库：保存哈希值（和盐值）。
4. 后端：返回注册成功响应。

**登录**：

1. 前端：发送用户名和密码（HTTPS）。
2. 后端：查询数据库，验证密码哈希。
3. 后端：验证通过后返回令牌，前端保存令牌用于后续请求。

## HTTP 数据类型

在HTTP协议中，请求和响应的内容可以根据`Content-Type`头部进行分类：

### 1. **表单数据（Form Data）**

- **`application/x-www-form-urlencoded`**
  默认的表单提交格式，数据编码为键值对（如 `key1=value1&key2=value2`）。

  ```http
  POST /submit HTTP/1.1
  Content-Type: application/x-www-form-urlencoded
  
  name=John&age=30
  ```

- **`multipart/form-data`**
  用于文件上传或表单中包含二进制数据，数据分多部分

  ```http
  POST /upload HTTP/1.1
  Content-Type: multipart/form-data; boundary=----boundary
  
  ----boundary
  Content-Disposition: form-data; name="file"; filename="example.txt"
  Content-Type: text/plain
  
  (文件二进制数据)
  ----boundary--
  ```

### 2. **JSON（JavaScript Object Notation）**

- **`application/json`**
  用于传输结构化JSON数据，常见于REST API。

  ```http
  POST /api/data HTTP/1.1
  Content-Type: application/json
  
  {"name": "John", "age": 30}
  ```

------

### 3. **XML（eXtensible Markup Language）**

- **`application/xml`** 或 **`text/xml`**
  用于传输XML格式数据。

  ```http
  POST /api/data HTTP/1.1
  Content-Type: application/xml
  
  <user><name>John</name><age>30</age></user>
  ```

------

### 4. **纯文本（Plain Text）**

- **`text/plain`**
  普通文本内容，无格式。

  ```http
  POST /log HTTP/1.1
  Content-Type: text/plain
  
  This is a log message.
  ```

------

### 5. **二进制数据（Binary Data）**

- **`application/octet-stream`**
  未知或任意二进制数据（如文件下载）。

  ```http
  HTTP/1.1 200 OK
  Content-Type: application/octet-stream
  Content-Disposition: attachment; filename="example.zip"
  
  (文件二进制数据)
  ```

------

### 6. **其他常见类型**

- **`application/javascript`**
  JavaScript代码。
- **`text/html`**
  HTML网页内容。
- **`image/png`**, **`image/jpeg`**
  图片资源。
- **`application/pdf`**
  PDF文档。

注：关键头部字段：

- **`Content-Type`**：定义请求/响应的主体类型（如 `Content-Type: application/json`）。
- **`Accept`**：客户端声明期望的响应类型（如 `Accept: application/json`）。

# 前端处理用户信息

- 密码输入：
  - 用户在前端（如浏览器、移动端）通过表单输入密码。
  - 密码字段使用 `<input type="password">`，避免明文显示。
- 不存储明文：
  - 前端不保存明文密码，仅在用户提交时处理。
- 加密传输：
  - 使用 **HTTPS**（TLS/SSL）确保密码在网络传输中加密，防止中间人攻击。
  - 前端将密码作为请求体的一部分（如 JSON）发送到后端。
- 不预处理密码：
  - 前端不应对密码进行哈希或加密，直接发送明文密码（通过 HTTPS），由后端负责安全处理。哈希在前端可能导致盐值暴露或被拦截。

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Login Page</title>
        <style>
            html {
                font-family: Arial, sans-serif;
                height: 100vh;
                width: 100vw;
                color: white;
                background-color: #333333;
            }
            body {
                margin: 0;
                height: 100%;
                width: 100%;
                display: flex;
                align-items: center;
                justify-content: center;
            }
            .login_container {
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: center;
                background: rgba(255, 255, 255, 0.2); /* 半透明背景 */
                backdrop-filter: blur(10px); /* 模糊强度 */
                border: 2px solid rgba(255, 255, 255, 0.3); /* 轻边框增强效果 */
                width: 60%;
                border-radius: 10px;
            }
            form {
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: center;
                gap: 20px;
                padding: 20px;
            }
            .input_group {
                width: 100%;
                display: flex;
                flex-direction: row;
                align-items: center;
                justify-content: flex-end;
                gap: 10px;
            }
        </style>
    </head>
    <body>
        <div class="login_container">
            <h1>Login</h1>
            <form action="https://localhost:8083/login" method="post">
                <div class="input_group">
                    <label for="email">Email:</label>
                    <input
                        type="email"
                        id="email"
                        name="email"
                        pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$"
                        title="输入正确邮箱地址"
                        required
                    />
                </div>
                <div class="input_group">
                    <label for="password">Password:</label>
                    <input
                        type="password"
                        id="password"
                        name="password"
                        pattern="(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[A-Za-z\d]{8,}"
                        title="密码需至少8位,包含大小写字母和数字"
                        required
                    />
                </div>
                <input type="submit" value="Login" />
            </form>
        </div>

        <script></script>
    </body>
</html>

```

# 第三方登陆

在一个 Spring Boot 应用中同时实现 **OAuth2 Client**（用于第三方登录，如 Google） 和 **OAuth2 Authorization Server**（用于生成自定义 JWT 和 Refresh Token）。

1. **OAuth2 Client**：处理第三方登录，获取用户 Access Token 和用户信息。
2. **OAuth2 Authorization Server**：接收 OAuth2 Client 传递的信息，生成 JWT 和 Refresh Token，并支持 Refresh Token 刷新流程。
3. **整合**：在一个 Spring Boot 应用中运行这两部分，确保它们协同工作。
