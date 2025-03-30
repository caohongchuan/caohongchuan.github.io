---
title: Nginx 面试合集
category: interview
---

```bash
# install
sudo apt update
sudo apt install nginx

sudo systemctl start nginx
sudo systemctl stop nginx

#平滑启动，不会中断
sudo systemctl reload nginx
#完全重启，会中断
sudo systemctl restart nginx
# 状态
sudo systemctl status nginx
# 开机自启动
sudo systemctl enable nginx
# 关闭开机启动
sudo systemctl disable nginx

ps aux | grep nginx
```

# 1. 什么是Nginx

[Nginx](https://nginx.org/) 是一个高性能的**HTTP服务器**和**反向代理服务器**，它以**轻量级**和**高并发**处理能力而闻名。Nginx 的核心是基于**事件驱动架构**，这使得它在处理大量并发连接时表现出色。它的设计不像传统的服务器那样使用线程处理请求，而是一个更加高级的机制—事件驱动机制，是一种异步事件驱动结构。此外，Nginx 还提供了邮件代理、TCP/UDP 代理服务器的功能，以及强大的**负载均衡**和**缓存机制**。它的模块化设计也使得它能够灵活地适应不同的应用场景。

Nginx 的**反向代理**功能允许它作为前端服务器，接收客户端的请求并将它们转发到后端服务器，同时，Nginx 也能够作为**负载均衡**器，将流量分配到多个后端服务器，这样可以提高网站的可用性和扩展性。Nginx 还支持**静态文件服务**，由于其高效的文件处理能力，它经常被用来作为静态资源的服务器。

# 2.Nginx优点

* **高性能和高并发**：Nginx 能够处理大量的并发连接，而内存消耗相对较小。这使得它成为处理高流量网站的理想选择。
* **高可靠性**： Nginx有健康监听机制，当某服务器损坏后，可自动将请求转移到其他正常的服务器。
* **内存消耗小**：与其他服务器相比，Nginx 在开启多个进程时，内存消耗仍然很低，这对于资源有限的环境非常有用。
* 静态文件处理：Nginx 在处理静态文件方面非常高效，它能够快速地提供图片、CSS、JavaScript 等静态资源。
* **反向代理 负载均衡**：Nginx 可以作为负载均衡器，将流量分配到多个后端服务器，这样可以提高网站的可用性和扩展性。
* 模块化和灵活性：Nginx 的模块化设计使得它能够灵活地适应不同的应用场景，如作为邮件代理、通用 TCP/UDP 代理服务器等。
* 安全性：Nginx 提供了多种安全特性，如防止 DDoS 攻击、限制请求频率等，这有助于提高网站的安全性。

缺点：**动态处理能力**：Nginx 在处理动态内容方面相对较弱，它更适合作为静态资源的服务器和反向代理。对于需要复杂动态处理的应用，可能需要与其他应用服务器（如 PHP、Node.js）配合使用。

# 3. Nginx性能高的原因

* 异步非阻塞事件处理机制：Nginx 使用了 **epoll**（在 Linux 上）模型，这是一种异步非阻塞的事件处理机制，它可以有效地处理大量的并发连接，而不会因为等待 I/O 操作而阻塞。
* 轻量级进程/线程模型：Nginx 使用了轻量级的进程/线程模型，这使得它能够在有限的系统资源下处理大量的并发连接。
* 高效的内存管理：Nginx 在内存管理上非常高效，它使用了自己的内存分配器，这减少了内存碎片和内存泄露的风险。
* 模块化设计：Nginx 的模块化设计使得它能够灵活地添加或移除功能，这样可以确保只有需要的功能被加载，从而减少了资源的消耗。
* 静态文件处理优化：Nginx 对静态文件的处理进行了优化，它能够快速地提供静态资源，而不需要后端服务器的参与。

# 4. Nginx如何处理请求

1. **接收请求**: 当客户端发起一个请求时，Nginx 的工作进程会监听网络端口，接收客户端的连接请求。

2. **建立连接**: Nginx 接收到客户端的连接请求后，会为该连接分配一个连接对象（ngx_connection_t）。连接对象包含连接的状态信息、读写事件处理器等。

3. **读取请求头**: Nginx 从客户端读取请求头信息。请求头包含 HTTP 方法（如 GET POST）、URL HTTP 版本以及各种请求头字段（如 Host User-Agent Content-Length 等）。

4. **解析请求头**: Nginx 解析请求头信息，提取必要的参数，如请求方法、URI Host 等。解析后的请求头信息存储在 ngx_http_request_t 结构体中。

5. **查找匹配的 server 块**: Nginx 根据请求头中的 Host 字段查找匹配的虚拟主机（server 块）。每个虚拟主机可以配置不同的域名和监听端口。

6. **查找匹配的 location 块**: 在找到匹配的虚拟主机后，Nginx 继续查找与请求 URI 匹配的 location 块。location 块定义了如何处理特定路径的请求。

7. **执行处理阶段**: Nginx 的请求处理分为多个阶段，每个阶段可以由多个模块处理。这些阶段包括但不限于：

  * rewrite phase：执行重写规则，如 URL 重写。

  * post rewrite phase：处理重写后的请求。

  * preaccess phase：执行访问控制，如 IP 地址过滤。

  * access phase：执行访问控制，如身份验证。

  * content phase：生成响应内容，如静态文件服务、反向代理、FastCGI 等。

8. **生成响应**: 在 content phase 阶段，Nginx 根据配置生成响应内容。这可能涉及读取静态文件、调用后端服务（如反向代理、FastCGI、uWSGI 等）、生成动态内容等。

9. **发送响应头**: Nginx 将生成的响应头发送回客户端。响应头包含 HTTP 状态码、响应头字段（如 Content-Type、Content-Length 等）。

10. **发送响应体**: Nginx 将生成的响应体发送回客户端。响应体可以是静态文件内容、后端服务返回的数据等。

11. **关闭连接**: 一旦响应发送完毕，Nginx 会关闭连接。如果启用了 keep-alive 连接，则连接可以保持打开状态，用于后续请求。

# 5. 正向代理和反向代理

![](https://img2.baidu.com/it/u=874433948,3077840998&fm=253&fmt=auto&app=138&f=JPEG?w=500&h=518)

都是网络代理服务器：

* 正向代理：
  * 正向代理位于客户端和目标服务器之间。
  * 客户端通过代理服务器向目标服务器发送请求。
  * 目标服务器只能看到代理服务器的 IP 地址，而看不到客户端的真实 IP 地址。
  * 正向代理通常用于客户端访问互联网时，通过代理服务器来访问外部资源，这可以提高安全性和隐私保护。

* 反向代理：

  * 反向代理位于客户端和目标服务器之间，但与正向代理不同，客户端通常不知道反向代理的存在。

  * 客户端向反向代理服务器发送请求，然后反向代理服务器将请求转发到一个或多个后端服务器。

  * 后端服务器处理请求并将响应返回给反向代理服务器，反向代理服务器再将响应返回给客户端。

  * 反向代理可以提高网站的可用性和扩展性，同时也可以提供缓存、负载均衡和 SSL 终端等功能。

  * 隐藏服务器：反向代理服务器可以隐藏后端服务器的存在和特征，这有助于提高安全性，因为外部用户无法直接访问后端服务器。
    负载均衡：反向代理可以作为负载均衡器，将流量分配到多个后端服务器，这样可以提高网站的可用性和扩展性。
    缓存静态内容：反向代理服务器可以缓存静态内容，如图片、CSS 和 JavaScript 文件等，这样可以减少后端服务器的负载并提高响应速度。
    SSL 终端：反向代理服务器可以处理 SSL/TLS 加密，这样可以减轻后端服务器的加密负担。
    压缩和优化：反向代理服务器可以在将内容发送到客户端之前对其进行压缩和优化，这样可以减少带宽消耗并提高响应速度。
    提供额外的安全层：反向代理服务器可以提供额外的安全层，如防火墙、DDoS 防护等。

    ￼
                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

# 6. 使用“反向代理服务器”的优点

* 隐藏服务器：反向代理服务器可以隐藏后端服务器的存在和特征，这有助于提高安全性，因为外部用户无法直接访问后端服务器。

* 负载均衡：反向代理可以作为负载均衡器，将流量分配到多个后端服务器，这样可以提高网站的可用性和扩展性。

* 缓存静态内容：反向代理服务器可以缓存静态内容，如图片、CSS 和 JavaScript 文件等，这样可以减少后端服务器的负载并提高响应速度。

* SSL 终端：反向代理服务器可以处理 SSL/TLS 加密，这样可以减轻后端服务器的加密负担。
  压缩和优化：反向代理服务器可以在将内容发送到客户端之前对其进行压缩和优化，这样可以

  减少带宽消耗并提高响应速度。

* 提供额外的安全层：反向代理服务器可以提供额外的安全层，如防火墙、DDoS 防护等。

# 7. Nginx应用场景

* HTTP 服务器：Nginx 可以作为 HTTP 服务器独立提供 HTTP 服务，适用于静态网站托管。

* 虚拟主机：Nginx 支持虚拟主机功能，可以在一台服务器上托管多个网站，这对于托管提供商来说非常有用。

* 反向代理和负载均衡：Nginx 可以作为反向代理服务器，将请求转发到后端服务器，并支持负载均衡，这对于高流量网站和应用来说非常重要。

* API 网关：Nginx 可以配置为 API 网关，对每个接口服务进行拦截和路由，提供额外的安全层和流量控制。

* 媒体流服务：Nginx 支持媒体流服务，可以用于视频点播和直播服务。

* 邮件代理：Nginx 还可以作为邮件代理服务器，处理邮件传输。

* 缓存服务器：Nginx 可以设置为缓存服务器，缓存静态内容和动态内容，减少后端服务器的负载，提高响应速度。

* SSL 终端：Nginx 可以处理 SSL/TLS 加密，提供 HTTPS 服务，增强数据传输的安全性。

# 8. Nginx目录结构

client_body_temp：用于存储客户端请求的临时文件。
conf：存放 Nginx 的配置文件，包括 nginx.conf 主配置文件和其他配置文件。
fastcgi_temp：用于存储 FastCGI 进程的临时文件。
html：默认的站点目录，通常用于存放静态文件，如 HTML、CSS、JavaScript 文件等。
logs：存放 Nginx 的日志文件，包括访问日志 access.log、错误日志 error.log 和进程 ID 文件 nginx.pid。
proxy_temp：用于存储代理服务器的临时文件。
sbin：存放 Nginx 的可执行文件，如 nginx 命令。
scgi_temp：用于存储 SCGI 进程的临时文件。
uwsgi_temp：用于存储 uWSGI 进程的临时文件。

# 9. Nginx配置文件nginx.conf有哪些属性模块

* http：定义了 HTTP 服务器的配置，包括文件类型、默认类型、连接超时等。

* server：定义了虚拟主机的配置，可以包含多个 server 块，每个块定义了一个虚拟主机的设置。

* location：定义了请求的匹配和处理规则，可以根据 URI、正则表达式等匹配请求，并指定处理方式。

* upstream：定义了负载均衡的配置，可以指定多个后端服务器，并设置负载均衡策略。

* events：定义了事件处理的配置，如工作连接数 worker_connections。

* mail：如果 Nginx 用于邮件代理，这个模块用于配置邮件服务的相关参数。

  ```
  全局模块
  event模块
  http模块
      upstream模块
  
      server模块
          location块
          location块
          ....
      server模块
          location块
          location块
          ...
      ....    
  ```

# 10. 静态文件服务

Nginx 在处理静态文件（如 HTML、CSS、JavaScript 和图像文件）时非常高效。它可以直接从文件系统中读取静态文件并返回给客户端，而不需要经过复杂的处理流程。

* root 指令：指定文件系统上的根目录，用于查找静态资源。
* alias 指令：为静态资源定义一个别名，方便管理和引用。
* autoindex 指令：当请求的 URI 以斜杠 / 结尾时，启用目录索引功能，列出目录下的所有文件。
* sendfile 指令：开启高效传输模式，允许 Nginx 直接在内核空间将文件发送给客户端，提高文件传输效率。
  

```
http{
	server {
	    listen 8081;
	    server_name localhost;
	    root /home/amber/Downloads/photo;
	    autoindex on; # 启用目录列表
	    location / {
			try_files $uri $uri/ =404;
	    }
	}
}
```

# 11. 如何用Nginx解决前端跨域问题

Nginx 可以通过配置 CORS（跨源资源共享）头部来解决前端跨域问题。

在 server 或 location 块中，使用 add_header 指令添加 Access-Control-Allow-Origin 头部，指定允许访问的源。如果需要，还可以添加 Access-Control-Allow-Methods 头部，指定允许的 HTTP 方法。对于需要凭证的请求，可以添加 Access-Control-Allow-Credentials 头部。

> 产生跨域问题的主要原因就在于**同源策略**，为了保证用户信息安全，防止恶意网站窃取数据，同源策略是必须的，否则`cookie`可以共享。由于`http`无状态协议通常会借助`cookie`来实现有状态的信息记录，例如用户的身份/密码等，因此一旦`cookie`被共享，那么会导致用户的身份信息被盗取。
> 同源策略主要是指三点相同，**协议+域名+端口** 相同的两个请求，则可以被看做是同源的，但如果其中任意一点存在不同，则代表是两个不同源的请求，同源策略会限制了不同源之间的资源交互。

```
location / {
    # 允许跨域的请求，可以自定义变量$http_origin，*表示所有
    add_header 'Access-Control-Allow-Origin' *;
    # 允许携带cookie请求
    add_header 'Access-Control-Allow-Credentials' 'true';
    # 允许跨域请求的方法：GET,POST,OPTIONS,PUT
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT';
    # 允许请求时携带的头部信息，*表示所有
    add_header 'Access-Control-Allow-Headers' *;
    # 允许发送按段获取资源的请求
    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
    # 一定要有！！！否则Post请求无法进行跨域！
    # 在发送Post跨域请求前，会以Options方式发送预检请求，服务器接受时才会正式请求
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Max-Age' 1728000;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        # 对于Options方式的请求返回204，表示接受跨域请求
        return 204;
    }
}
```

# 12. Nginx虚拟主机怎么配置

在 Nginx 中配置虚拟主机主要涉及 `server` 块的设置。

* 定义 server 块：每个 server 块定义了一个虚拟主机的配置。
* 设置监听端口：使用 listen 指令设置服务器监听的端口，通常是 80（HTTP）和 443（HTTPS）。
* 设置服务器名称：使用 server_name 指令设置虚拟主机的域名。
* 定义 location 块：在 server 块内部定义 location 块，设置请求的处理规则。
* 设置根目录：使用 root 指令设置网站内容的根目录。
* 设置默认首页：使用 index 指令设置默认首页文件。

`location` 指令在 Nginx 配置中扮演着核心角色，它定义了如何处理进入 Nginx 的 HTTP 请求。location 块可以匹配不同的 URI、正则表达式或指定的字符串，从而允许对特定的请求路径应用不同的处理规则。

* 精确匹配：使用 = 符号进行精确匹配，例如 location = / 匹配根路径。
* 字符串开头匹配：使用 ^~ 符号匹配以特定字符串开头的 URI。
* 正则表达式匹配：使用 ~ 或 ~* 符号进行正则表达式匹配，其中 ~ 是区分大小写的，而 ~* 是不区分大小写的。
* 通用匹配（前缀匹配）：使用 / 符号进行通用匹配，作为最后的选择，如果其他匹配都未成功，请求将被这个 location 块处理。

location 块可以包含多种指令，如 proxy_pass root try_files 等，用于定义请求的处理方式，例如代理到后端服务器、提供静态文件服务或重定向。

# 13. Nginx配置SSL证书

为了缓解后端服务器的压力，加密解密一般都放置在Nginx反向代理中，而后端服务器处于局域网内，不需要加密传输。

1. 先去CA机构或从云控制台中申请对应的`SSL`证书，审核通过后下载`Nginx`版本的证书。
2. 下载数字证书后，完整的文件总共有三个：`.crt .key .pem,`
   * `.crt`：数字证书文件，`.crt`是`.pem`的拓展文件，因此有些人下载后可能没有。`
   * `.key`：服务器的私钥文件，及非对称加密的私钥，用于解密公钥传输的数据。
   * `.pem`：`Base64-encoded`编码格式的源证书文本文件，可自行根需求修改拓展名。

3. 在`Nginx`目录下新建`certificate`目录，并将下载好的证书/私钥等文件上传至该目录。

4. 修改Nginx配置文件：

   ```
   # ----------HTTPS配置-----------
   server {
       # 监听HTTPS默认的443端口
       listen 443;
       # 配置自己项目的域名
       server_name www.xxx.com;
       # 打开SSL加密传输
       ssl on;
       # 输入域名后，首页文件所在的目录
       root html;
       # 配置首页的文件名
       index index.html index.htm index.jsp index.ftl;
       # 配置自己下载的数字证书
       ssl_certificate  certificate/xxx.pem;
       # 配置自己下载的服务器私钥
       ssl_certificate_key certificate/xxx.key;
       # 停止通信时，加密会话的有效期，在该时间段内不需要重新交换密钥
       ssl_session_timeout 5m;
       # TLS握手时，服务器采用的密码套件
       ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
       # 服务器支持的TLS版本
       ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
       # 开启由服务器决定采用的密码套件
       ssl_prefer_server_ciphers on;
   
       location / {
           ....
       }
   }
   
   # ---------HTTP请求转HTTPS-------------
   server {
       # 监听HTTP默认的80端口
       listen 80;
       # 如果80端口出现访问该域名的请求
       server_name www.xxx.com;
       # 将请求改写为HTTPS（这里写你配置了HTTPS的域名）
       rewrite ^(.*)$ https://www.xxx.com;
   }
   ```

# 14. 如何配置限流

Nginx 的限流基于令牌桶算法，主要依赖以下指令：

- **limit_req_zone**：定义限流的共享内存区域和限制规则。
- **limit_req**：在具体的 location 中应用限流规则。

```
# 限制每个 IP 每秒 2 次请求
http {
    limit_req_zone $binary_remote_addr zone=myzone:10m rate=2r/s;

    server {
        location / {
        	#允许额外 5 个突发请求，超限请求会被延迟处理
            limit_req zone=myzone burst=5;
        }
    }
}
```

如果客户端在 1 秒内发送 10 个请求：

- 前 2 个请求立即处理（符合 rate）。
- 接下来的 5 个请求进入队列（burst 容量）。
- 剩余 3 个请求被拒绝（桶满）。
- 队列中的 5 个请求会以每秒 2 个的速率逐步处理。

# 15. 漏桶流算法和令牌桶算法

漏桶算法和令牌桶算法是两种常用的流量整形和速率限制算法，它们在网络中用于控制数据的传输速率。

* 漏桶算法：
  漏桶算法通过一个固定容量的桶和漏水龙头来控制数据流。数据以任意速率进入桶中，然后以固定的速率从桶中流出。如果桶满了，额外的数据将被丢弃。这种机制可以平滑突发流量，确保数据以固定的速率被处理，防止网络拥塞。

* 令牌桶算法：
  令牌桶算法使用一个令牌桶，桶中会以固定的速率生成令牌。每个数据包的发送需要消耗一个令牌。如果桶中没有令牌，数据包将被延迟发送，直到桶中有令牌可用。这种算法允许数据以峰值速率发送，直到令牌耗尽，然后数据发送速率将降至平均速率。

令牌桶算法更适合需要突发传输的应用，因为它允许在令牌充足时快速发送数据。而漏桶算法则更适合于那些需要持续、均匀速率发送数据的应用。

# 16. Nginx 动静分离

动静分离是指将动态内容和静态内容分开处理和存储。

* 性能优化：静态资源如图片 CSS JavaScript 文件等不经常变化，可以由 Nginx 直接提供，这样可以减少后端应用服务器的负载，提高响应速度。
* 缓存策略：静态资源适合设置较长的缓存时间，而动态内容通常需要实时处理，动静分离后可以针对静态资源设置更有效的缓存策略。
* 扩展性：在高流量的情况下，可以通过增加静态资源服务器的数量来提高服务能力，而不用担心影响动态内容的处理。
* 安全性：静态资源通常不需要访问后端数据库或其他安全敏感的服务，将它们与动态内容分离可以减少安全风险。

通过动静分离，可以提高网站的整体性能和可扩展性，同时降低运维成本。
```
server {
    listen 80;
    server_name example.com;
    
    # 静态资源配置
    location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css){
        root    /xxx/static_resources;
        expires 7d;
    }
    
    # 动态资源配置
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

# 17. Nginx负载均衡的算法

Nginx 实现负载均衡主要通过 upstream 模块，该模块允许定义一个后端服务器组，并根据特定的策略将请求分发到这些服务器。以下是 Nginx 支持的一些负载均衡策略：

* 轮询（round-robin）：默认策略，将请求按顺序轮流分配给每个服务器。

* 权重（weight）：通过给每个服务器设置权重来控制流量分配的比例。

  ```
  upstream app1 {
      server 10.10.10.1 weight=3;
      server 10.10.10.2;
  }
  ```

* IP绑定（ip_hash）：通过请求来源的 IP 地址进行哈希，确保同一 IP 的请求总是被分配到同一个服务器。

  ```
  upstream app1 {
      ip_hash;
      server 10.10.10.1;
      server 10.10.10.2;
  }
  
  ```

* 最少连接（least-connected）：每次都找连接数最少的服务器来转发请求。

  ```
  upstream app1 {
      least_conn;
      server 10.10.10.1;
      server 10.10.10.2;
  }
  ```

```
http{
    upstream backend {
        server server1 weight=3;
        server server2 weight=2;
        server server3;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

# 18. Nginx配置高可用性

配置 Nginx 以实现高可用性主要涉及确保 Nginx 能够处理后端服务器的故障，并在必要时将流量重定向到健康的服务器。

* 定义多个后端服务器：在 upstream 块中定义多个服务器，以便在一个服务器失败时有备用服务器可用。
* 设置超时参数：配置 proxy_connect_timeout、proxy_send_timeout 和 proxy_read_timeout 指令，以便在后端服务器无响应时及时失败转移。
* 使用 max_fails 和 fail_timeout：配置 max_fails 指令来设置在多长时间内允许多少次失败，以及 fail_timeout 指令来设置服务器失败后应该被排除在外的时间。

```
http {
    upstream backend {
        server server1;
        server server2;
        server server3;

        max_fails=3;
        fail_timeout=30s;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;
            proxy_connect_timeout 1s;
            proxy_send_timeout 1s;
            proxy_read_timeout 1s;
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        }
    }
}
```

如果后端服务器在 30 秒内失败超过 3 次，它将被认为不可用，并从轮询中排除 30 秒。同时，如果 Nginx 遇到超时或指定的 HTTP 状态码，它将尝试将请求代理到另一个健康的服务器

# 19. 限制IP或浏览器访问

```
# 只允许Chrome浏览器访问
server {
    listen 80;
    server_name example.com;
    
    location / {
        if ($http_user_agent ~ Chrome) {
            return 500;
        }
        proxy_pass http://backend;
    }
}

# ------------------------------------------------------
# 只允许192.168.1.100和192.168.1.101访问
geo $block_ip {
    default 0;
    192.168.1.100 1;
    192.168.1.101 1;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        if ($block_ip) {
            return 403;
        }
        proxy_pass http://backend;
    }
}
```

# 20. 多进程模型 主进程 和 工作进程

Nginx的多进程模型主要由一个主进程（master process）和多个工作进程（worker process）组成。主进程负责管理和监控工作进程，而工作进程负责处理实际的客户端请求。

每个工作进程都是单线程的，这意味着每个工作进程在同一时间只能处理一个客户端请求。这种设计选择主要基于以下原因：

* 轻量级：单线程模型相对于多线程或多进程模型来说更加轻量级，减少了线程切换和进程间通信的开销。
* 可扩展性：通过创建多个工作进程，Nginx能够同时处理多个请求，实现高并发处理能力。每个工作进程之间相互独立，可以并行处理请求，提高系统的吞吐量。
* 高效的事件驱动模型：Nginx使用了高效的事件驱动模型（基于epoll kqueue等），通过异步非阻塞方式处理网络请求，从而避免了线程阻塞和资源浪费。

需要注意的是，尽管每个工作进程是单线程的，但Nginx通过事件驱动和非阻塞I/O的方式能够处理大量并发请求，实现高性能和高吞吐量。这种设计在处理静态内容和反向代理等场景下表现出色，但在涉及大量计算密集型任务的场景下可能会受到性能限制。在这种情况下，通常会将计算任务委托给后端应用服务器来处理。

当Nginx启动时，它会创建一个主进程（master process），主进程主要负责以下任务：

* 读取并解析配置文件：主进程会读取Nginx的主配置文件（nginx.conf），包括全局配置和默认服务器配置。它还可以包含其他辅助配置文件（如conf.d目录下的文件）。
* 启动工作进程：主进程会根据配置文件中指定的工作进程数量，创建相应数量的工作进程。每个工作进程都是一个独立的进程，负责实际处理客户端请求。
* 监控工作进程：主进程会监控工作进程的运行状态，如果工作进程异常退出或终止，主进程会重新启动新的工作进程来替代。
* 处理信号：主进程可以接收系统信号（如重启、停止等），并根据信号进行相应的操作，例如重新加载配置文件或优雅地关闭工作进程。

每个工作进程都是单线程的，每个工作进程通过异步非阻塞方式处理客户端请求。这种事件驱动的模型允许每个工作进程同时处理多个并发请求，而无需为每个请求创建一个新的线程。

工作进程的主要任务包括：

* 接收和处理连接：工作进程通过监听套接字接收客户端的连接请求，并根据配置文件中的服务器块进行请求分发。它会接受请求、解析请求头和请求体，并执行相应的操作。
* 处理请求：工作进程通过事件驱动模型，使用非阻塞I/O方式处理请求。它会执行请求所需的操作，如读取文件、向后端服务器发起代理请求等。
* 响应客户端：工作进程会生成响应数据，并将响应发送回客户端。它会设置响应头、发送响应体，并根据配置进行内容压缩、缓存等操作。
* 日志记录：工作进程会将访问日志和错误日志写入相应的日志文件，记录请求的详细信息和错误信息。

Nginx的多进程单线程模型结合了事件驱动和非阻塞I/O的优势，使得它在高并发场景下能够高效地处理大量的并发请求，同时保持较低的资源消耗。这使得Nginx成为一个流行的高性能Web服务器和反向代理服务器。