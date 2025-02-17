---
title: 
category: 
---

# HTTPs

HTTPs分为两个阶段：握手阶段，传输阶段。

# 握手阶段

> TCP握手 + TLS\SSL握手

网络协议分为五层，首先完成网络层TCP的三次握手，再完成传输层的TLS\SSL的握手，最后到达应用层的HTTPS

![network structure](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250203171952087.png)

## 1.证书生成

### 生成密钥

`openssl`生成RSA密钥：

```bash
openssl genrsa -out prkey.key 2048
```

生成RSA加密密钥：

```bash
openssl genrsa -des3 -out prkey.key 2048
```

### 从私钥中导出公钥

```bash
openssl rsa -pubout -in prkey.key -out pukey.pem
```

### 创建证书签名请求（CSR）

```bash
openssl req -new -key prkey.key -out cer_request.csr
```

### 自签名证书

`.crt`可以换成`.pem`

```bash
openssl x509 -req -days 365 -in cer_request.csr -signkey prkey.key -out certificate.crt
```

### CA签署证书

`.crt`可以换成`.pem`

```bash
openssl x509 -req -days 365 -in cer_request.csr -CA ca_cert.crt -CAkey ca_key.key -CAcreateserial -out certificate.crt 
```

查看

```bash
openssl x509 -in certificate.crt -noout -text
openssl req -in cer_request.csr -noout -text
```

### 实例

自签名证书有两种类型：

- 自签名证书：自签名证书的`Issuer`和`Subject`是相同的。

- 私有 CA 签名证书：其中CA的证书是自签名证书，server证书是由CA签发的。

  

1. 生成服务器端私钥（server.key)

   ```bash
   以上就是一个模板配置， 哪些是必要配置，哪些是根据自己实际需求，看一眼就知道了。看一个真实的UserDao-Mapper.xml配openssl genrsa -out server.key 2048
   ```

2. 生成服务端签名证书请求（server.csr)

   ```bash
   openssl req -new -config serverssl.cnf -key server.key -out server.csr
   ```

   ```py
   # serverssl.cnf
   [req]
   default_bits = 2048
   prompt = no
   default_md = sha256
   x509_extensions = v3_req
   distinguished_name = dn
    
   [dn]
   C = CN
   ST = SD
   L = JINAN
   O = yingzheng
   OU = yingzhengunit
   emailAddress = yingzhengttt@gmail.com
   CN = yingzhengttt.com
    
   [v3_req]
   subjectAltName = @alt_names
    
   [alt_names]
   DNS.1 = yingzhengttt.com
   DNS.2 = www.yingzhengttt.com
   ```

   其中CN必须与服务器的域名一致。否则浏览器会警告不安全。

3. 生成CA私钥（ca.key)

   ```bash
   openssl genrsa -out ca.key 2048
   ```

4. 生成CA自签名证书（ca.crt）

   ```bash 
   openssl req -new -x509 -days 365 -key ca.key -out ca.crt -subj "/C=CN/ST=SD/L=JINAN/O=yingzheng/OU=yingzhengunit/CN=ROOTcert/emailAddress=yingzhengttt@gmail.com"
   ```

5. 使用CA证书(ca.crt)对服务端证书请求(server.csr)签名，获取服务端签名（server.crt）

   ```bash
   openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt 
   ```

注：本测试在Ubuntu24上的Chrome中，只要是未被公共信任的证书，即使已经加入到系统的信任链中，浏览器也会提示安全风险。但是在Postman中能够正常工作。所以应该是浏览器的安全设置问题。

为了取消报错并在本地使用自签名证书，关闭Chrome的签名认证，在Chrome启动时加入参数 `--ignore-certificate-errors`。 比如在Ubuntu中，更改`/usr/share/applications/google-chrome.desktop`，在这里我使用的是canary所以更改`/usr/share/applications/google-chrome-canary.desktop`

```
Exec=/usr/bin/google-chrome-canary --ignore-certificate-errors %U
```

## 2.SpringBoot启用SSL

将`server.crt`，`server.key`拷贝到resource目录下。

pom.xml

```yaml
server:
    port: 8433
    ssl:
        certificate: "classpath:server.crt"
        certificate-private-key: "classpath:server.key"
    http:
        port: 8080
```

创建`TomcatConfg.java`来创建一个新的Tomcat来接受http请求。

```java
@Configuration
public class TomcatConfig {
    @Value("${server.http.port}")
    private int httpPort;
    @Value("${server.port}")
    private int httpsPort;
    @Bean
    TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint constraint = new SecurityConstraint();
                constraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                constraint.addCollection(collection);
                context.addConstraint(constraint);
            }
        };
        factory.addAdditionalTomcatConnectors(createTomcatConnector());
        return factory;
    }

    @Bean
    protected Connector createTomcatConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setPort(httpPort);
        connector.setSecure(false);
        connector.setRedirectPort(httpsPort);
        return connector;

    }
}
```

并在spring security中配置之允许使用https协议，并将http协议跳转到https协议端口

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .portMapper(portMapper -> portMapper.http(8080).mapsTo(8443))
                .requiresChannel(channel -> channel.anyRequest().requiresSecure())
				...;
        return http.build();
    }

}
```



## Springboot 添加信任证书

> 如果只想添加信任证书能够访问其他https服务，而不是自身启动https，可以只添加证书到信任链中。

### 方法1. 将证书添加到Java信任库

首先获取需要信任的证书`ca.crt`

将`ca.crt`导入到Java信任库中

```bash
keytool -import -trustcacerts -keystore $JAVA_HOME/lib/security/cacerts -noprompt -alias my-cert -file ca.crt
```

### 方法2. 将证书添加到Springboot中

*该方法并没有成功*

创建JKS信任库

```bash
keytool -import -trustcacerts -keystore server.jks -storepass 1234 -alias my-cert -file ca.crt
```

将`server.jks` 添加到resource目录下

在`application.yml`

```yaml
ssl:
    key-store: ""
    key-store-password: ""
    key-store-type: ""
    trust-store: classpath:server.jks
    trust-store-password: 1234
    trust-store-type: JKS
```

*问题：即使将key-store设置为空，但启动的时候springboot仍然去/home/amber/.keystore (No such file or directory)去拿key-store。一直解决不了。如果设置trust-store，springboot就默认启动ssl也会去找key-store，即使设置为空也还是去找。*

