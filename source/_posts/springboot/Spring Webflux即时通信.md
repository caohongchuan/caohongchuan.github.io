---
title: Spring WebFlux 实现 websocket
---

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
    public WebSocketHandler notifyWebSocketHandler() {
        return new NotifyWebSocketHandler();
    }

    @Bean
    public SimpleUrlHandlerMapping handlerMapping(WebSocketHandler echoWebSocketHandler,
                                                  WebSocketHandler chatWebSocketHandler,
                                                  WebSocketHandler notifyWebSocketHandler) {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/echo", echoWebSocketHandler);
        map.put("/chat", chatWebSocketHandler);
        map.put("/notify", notifyWebSocketHandler); // 新增通知端点

        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1);
        mapping.setCorsConfigurations(Map.of("*", new CorsConfiguration().applyPermitDefaultValues()));
        System.out.println("WebSocket mappings registered: " + map.keySet());
        return mapping;
    }

    @Bean
    public WebSocketHandlerAdapter webSocketHandlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

```java
package top.chc.imserver;

import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.time.Duration;

public class NotifyWebSocketHandler implements WebSocketHandler {
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Flux<String> notifications = Flux.interval(Duration.ofSeconds(2))
            .map(i -> "Notification " + i + " at " + System.currentTimeMillis());

        return session.send(notifications.map(session::textMessage));
    }
}
```

