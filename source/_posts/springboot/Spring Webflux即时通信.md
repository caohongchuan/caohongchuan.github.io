---
title: Spring WebFlux 实现 websocket
---

# 消息推送

>通过一定的技术标准或协议，在互联网上通过定期传送用户需要的信息来减少信息过载。

消息推送的实现方式：

- `AJAX`长短轮循：客户端不断发起请求，交换数据，客户端主动。
- 基于HTTP协议的`SSE`（Server-Sent Events）技术：实现服务器向客户端实时推送数据的Web技术，服务端主动。`SSE`基于`HTTP协议`，允许服务器将数据以事件流（Event Stream）的形式发送给客户端。客户端通过建立持久的`HTTP连接`，并监听事件流，可以实时接收服务器推送的数据。前端JS通过`EventSource`结合数据。
- `WebSocket`技术：`WebSocket` 是基于 TCP 的一种新的应用层网络协议。它提供了一个[全双工](https://so.csdn.net/so/search?q=全双工&spm=1001.2101.3001.7020)的通道，允许服务器和客户端之间实时双向通信。
- `MQTT`通讯协议

## WebSocket

> websocket是一个新的协议，是直接建立在TCP之上的，不基于HTTP，但握手过程中会使用HTTP。

特点：

* 建立在 TCP 协议之上，服务器端的实现比较容易。
* 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。
* 数据格式比较轻量，性能开销小，通信高效。可以发送文本，也可以发送二进制数据。
* 没有同源限制，客户端可以与任意服务器通信。
* 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

应用场景：

- 即时聊天通信
- 多玩家游戏
- 在线协同编辑/编辑
- 实时数据流的拉取与推送
- 体育/游戏实况
- 实时地图位置

原理图：

![image-20250411204253429](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250411204253429.png)

通过wireshark分析websocket建立过程：

```
// ---------------------------------------- 连接 -------------------------------------------
// TCP握手
127.0.0.1	127.0.0.1	TCP	74	53522 → 8080 [SYN] Seq=0 Win=65495 Len=0 MSS=65495 SACK_PERM TSval=1655867669 TSecr=0 WS=128
127.0.0.1	127.0.0.1	TCP	74	8080 → 53522 [SYN, ACK] Seq=0 Ack=1 Win=65483 Len=0 MSS=65495 SACK_PERM TSval=1655867669 TSecr=1655867669 WS=128
127.0.0.1	127.0.0.1	TCP	66	53522 → 8080 [ACK] Seq=1 Ack=1 Win=65536 Len=0 TSval=1655867669 TSecr=1655867669
//客户端发送HTTP请求携带
127.0.0.1	127.0.0.1	HTTP	293	GET /echo HTTP/1.1 
// ACK确认
127.0.0.1	127.0.0.1	TCP	66	8080 → 53522 [ACK] Seq=1 Ack=228 Win=65280 Len=0 TSval=1655867670 TSecr=1655867670
//服务端返回告诉客户端更换协议为websocket
127.0.0.1	127.0.0.1	HTTP	284	HTTP/1.1 101 Switching Protocols 
// ACK确认
127.0.0.1	127.0.0.1	TCP	66	53522 → 8080 [ACK] Seq=228 Ack=219 Win=65408 Len=0 TSval=1655867863 TSecr=1655867863


// ---------------------------------------- 数据传递 -------------------------------------------
// 客户端发送数据
127.0.0.1	127.0.0.1	WebSocket	84	WebSocket Text [FIN] [MASKED]
// ACK确认
127.0.0.1	127.0.0.1	TCP	66	8080 → 53522 [ACK] Seq=219 Ack=246 Win=65536 Len=0 TSval=1656126026 TSecr=1656126026
// 服务端回现数据给客户端
127.0.0.1	127.0.0.1	WebSocket	86	WebSocket Text [FIN] 
// ACK确认
127.0.0.1	127.0.0.1	TCP	66	53522 → 8080 [ACK] Seq=246 Ack=239 Win=65536 Len=0 TSval=1656126034 TSecr=1656126034


// ---------------------------------------- 关闭连接 -------------------------------------------
// 客户端告知服务端关闭连接
127.0.0.1	127.0.0.1	WebSocket	74	WebSocket Connection Close [FIN] [MASKED]
// 服务端告知客户端关闭连接
127.0.0.1	127.0.0.1	WebSocket	70	WebSocket Connection Close [FIN] 
// 客户端收到服务端断开请求并ACK确认
127.0.0.1	127.0.0.1	TCP	66	53522 → 8080 [ACK] Seq=254 Ack=243 Win=65536 Len=0 TSval=1656183231 TSecr=1656183231
// 服务器发起 TCP 连接关闭，发送 FIN 包
127.0.0.1	127.0.0.1	TCP	66	8080 → 53522 [FIN, ACK] Seq=243 Ack=254 Win=65536 Len=0 TSval=1656183232 TSecr=1656183231
// 客户端响应服务器的 FIN，发送自己的 FIN 包
127.0.0.1	127.0.0.1	TCP	66	53522 → 8080 [FIN, ACK] Seq=254 Ack=244 Win=65536 Len=0 TSval=1656183232 TSecr=1656183232
// 服务器确认客户端的 FIN 包
127.0.0.1	127.0.0.1	TCP	66	8080 → 53522 [ACK] Seq=244 Ack=255 Win=65536 Len=0 TSval=1656183232 TSecr=1656183232
```

创建过程：

1. 三次握手建立TCP连接
2. 客户端发送HTTP请求，请求升级通信协议为websocket
3. 服务端发送HTTP请求，将协议升级为websocket

通信过程：

1. 使用websocket协议通信

关闭过程：

1. 客户端使用websocket协议告知服务端关闭连接
2. 服务端使用websocket协议告知客户端关闭连接
3. 完成TCP的四次挥手



```java
@Configuration
public class WebSocketConfig {

    @Bean
    public WebSocketHandler echoWebSocketHandler() {
        return new EchoWebSocketHandler();
    }

    @Bean
    public WebSocketHandler chatWebSocketHandler() {
        return new ChatWebSocketHandler();
    }

    @Bean
    public SimpleUrlHandlerMapping handlerMapping(WebSocketHandler echoWebSocketHandler, WebSocketHandler chatWebSocketHandler) {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/echo", echoWebSocketHandler);
        map.put("/chat", chatWebSocketHandler);
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1); // 优先级高于其他路由
        mapping.setCorsConfigurations(Map.of("*", new CorsConfiguration().applyPermitDefaultValues()));
        return mapping;
    }
}
```

```java
@Slf4j
public class ChatWebSocketHandler implements WebSocketHandler {
    private final ConcurrentHashMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        log.info("New client connected: {}, Total: {}", session.getId(), sessions.size());
        sessions.put(session.getId(), session);

        Mono<Void> receiveAndBrodcast = session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .flatMap(msg -> {
                    log.info("Received from {}: {}", session.getId(), msg);
                    return Mono.when(
                            sessions.entrySet().stream()
                                    .filter(entry -> !entry.getKey().equals(session.getId()))
                                    .map(entry -> {
                                        WebSocketSession targetSession = entry.getValue();
                                        return targetSession.send(
                                                Mono.just(targetSession.textMessage("Broadcast: " + msg))
                                        );
                                    })
                                    .toList()
                    );
                })
                .doOnError(err -> log.error("Error in session {}: {}", session.getId(), err.getMessage()))
                .then();

        Mono<Void> sendWelcome = session.send(
                Mono.just(session.textMessage("Welcome to broadcast chat!"))
        );

        return Mono.when(sendWelcome, receiveAndBrodcast)
                .doFinally(signal -> {
                    sessions.remove(session.getId());
                    log.info("Client disconnect: {}, Total: {}", session.getId(), sessions.size());
                });
    }
}
```

```java
@Slf4j
public class EchoWebSocketHandler implements WebSocketHandler {
    @Override
    public Mono<Void> handle(WebSocketSession session) {

        return session.send(
                session.receive()
                        .map(msg -> {
                            log.info("Received: {}", msg.getPayloadAsText());
                            return session.textMessage("Echo: " + msg.getPayloadAsText());
                        })
                        .doOnError(e -> System.err.println("Error: " + e.getMessage()))
        );
    }
}
```

