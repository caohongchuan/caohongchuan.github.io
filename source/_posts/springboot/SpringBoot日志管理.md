---
title: SpringBoot日志管理
category: springboot
---

>SpringBoot支持多种日志框架，包括Logback、Log4j2和Java Util Logging（JUL）。默认情况下，如果你使用SpringBoot的starters启动器，它将使用Logback作为日志框架。
>
>* Logback：Logback是SpringBoot默认的日志框架，它是Log4j的继任者，提供了更好的性能和可靠性。可以通过在资源目录下创建一个logback-spring.xml文件来配置Logback。
>* Log4j2：Log4j2是Log4j的升级版，它在性能和功能上都有所提升，支持异步日志和插件机制。如果想在SpringBoot中使用Log4j2，需要添加相应的依赖并在配置文件中指定Log4j2作为日志框架。
>* Java Util Logging（JUL）：JUL是Java SE的默认日志框架，SpringBoot可以配置使用JUL作为日志框架，但一般不推荐使用，因为它的性能和灵活性相对较差。
>
>除了上述日志框架外，SpringBoot还支持SLF4J和Commons Logging这两个日志门面。这些门面可以与多种日志实现进行集成，使得你可以在不改变代码的情况下更换日志框架。
>
>无论使用哪种日志框架，SpringBoot都支持配置将日志输出到控制台或者文件中。可以在application.properties或application.yml配置文件中设置日志级别、输出格式等参数。

# 日志门面

在早期使用日志框架时，应用程序通常需要直接与具体的日志框架进行耦合，这就导致了以下几个问题：

- 代码依赖性

  应用程序需要直接引用具体的日志框架，从而导致代码与日志框架强耦合，难以满足应用程序对日志框架的灵活配置。

- 日志框架不统一

  在使用不同的日志框架时，应用程序需要根据具体的日志框架来编写代码，这不仅会增加开发难度，而且在多种日志框架中切换时需要进行大量的代码改动。

为了解决这些问题，SLF4J提供了一套通用的日志门面接口，让应用程序可以通过这些接口来记录日志信息，而不需要直接引用具体的日志框架。这样，应用程序就可以在不同的日志框架之间进行灵活配置和切换，同时还可以获得更好的性能表现。

依赖：

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.17</version>
</dependency>
```

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass {
    private static final Logger log = LoggerFactory.getLogger(MyClass.class);
    //...
    public static void main(String[] args) {
        log.info("This is an info message.");
    }
}
```

如果引入Lombok，可以使用lombok的注解@Slf4j 代替上面log的创建。

```java 
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class MyClass {
    public static void main(String[] args) {
        log.info("This is an info message.");
    }
}
```

在SpringBoot中，`spring-boot-starter`中包含`spring-boot-starter-logging`，其中会包含`slf4j-api`依赖。

# 日志实现

Logback 和 Log4j2是目前常用的两个日志实现框架。

## Logback

Logback 的核心模块为 logback-classic，它提供了一个 SLF4J 的实现，兼容 Log4j API，可以无缝地替换 Log4j。它自身已经包含了 logback-core 模块，而 logback-core，顾名思义就是 logback 的核心功能，包括日志记录器 Appender  Layout 等。其他 logback 模块都依赖于该模块。

```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.5.18</version>
    <scope>compile</scope>
</dependency>
```

logback 可以通过 XML 或者 Groovy 配置。以 XML 配置为例，logback 的 XML 配置文件名称通常为 `logback.xml` 或者 `logback-spring.xml`。在 Spring Boot 中，需要放置在 classpath 的根目录下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <!-- 定义通用变量 -->
    <property name="LOG_PATH" value="logs"/>
    <property name="LOG_FILE_NAME" value="application"/>
    <property name="PATTERN" value="%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n"/>

    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 文件输出，按日期滚动 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${LOG_FILE_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天滚动，历史日志文件名带日期 -->
            <fileNamePattern>${LOG_PATH}/${LOG_FILE_NAME}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>30</maxHistory> <!-- 保存 30 天 -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize> <!-- 单文件最大 100MB -->
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 开发环境配置 -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- 生产环境配置 -->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>

    <!-- 默认配置（未指定 profile 时使用） -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```

也可以在`application.yml`或者`application.properties`中简单配置日志：

```yaml
logging:
  level:
    root: INFO
    org.springframework.web: DEBUG
    com.yourpackage: DEBUG
  file:
    name: myapp.log
    path: /var/log
  logback:
    rollingpolicy:
      max-file-size: 10MB
      max-history: 30
```

注：如果`application.yml`和`logback-spring.xml`同时存在，`logback-spring.xml`会覆盖`application.yml`

## log4j 2

The Apache Log4j SLF4J 2.0 API binding to Log4j 2 Core

```xml
<!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-slf4j2-impl -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j2-impl</artifactId>
    <version>2.24.3</version>
    <scope>compile</scope>
</dependency>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="INFO" monitorInterval="30">
  <Properties>
    <Property name="logPath">logs</Property>
  </Properties>
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{ISO8601} [%t] %-5level %logger{36} - %msg%n" />
    </Console>
    <RollingFile name="File" fileName="${logPath}/example.log"
                 filePattern="${logPath}/example-%d{yyyy-MM-dd}-%i.log">
      <PatternLayout pattern="%d{ISO8601} [%t] %-5level %logger{36} - %msg%n"/>
      <Policies>
        <SizeBasedTriggeringPolicy size="10 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="4"/>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Logger name="com.zhanfu.child" level="DEBUG">
      <AppenderRef ref="File"/>
      <AppenderRef ref="Console"/>
    </Logger>
    <Logger name="com.zhanfu" level="INFO">
      <AppenderRef ref="File"/>
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="WARN">
      <AppenderRef ref="Console" />
    </Root>
  </Loggers>
</Configuration>
```

# 日志级别

* TRACE：是最低级别的日志记录，用于输出最详细的调试信息，通常用于开发调试目的。在生产环境中，应该关闭 TRACE 级别的日志记录，以避免输出过多无用信息。
* DEBUG：是用于输出程序中的一些调试信息，通常用于开发过程中。像 TRACE 一样，在生产环境中应该关闭 DEBUG 级别的日志记录。
* INFO：用于输出程序正常运行时的一些关键信息，比如程序的启动、运行日志等。通常在生产环境中开启 INFO 级别的日志记录。
* WARN：是用于输出一些警告信息，提示程序可能会出现一些异常或者错误。在应用程序中，WARN 级别的日志记录通常用于记录一些非致命性异常信息，以便能够及时发现并处理这些问题。
* ERROR：是用于输出程序运行时的一些错误信息，通常表示程序出现了一些不可预料的错误。在应用程序中，ERROR 级别的日志记录通常用于记录一些致命性的异常信息，以便能够及时发现并处理这些问题。

```java
public class Main {
    private static final Logger log = LoggerFactory.getLogger(Main.class);
    public static void main(String[] args) {
        log.trace("This is a Main trace message.");
        log.debug("This is a Main debug message.");
        log.info("This is a Main info message.");
        log.warn("This is a Main warn message.");
        log.error("This is a Main error message.");
        Slave.main(args);
    }
}
```

# SpringBoot不同开发环境

- `application-dev.properties`（开发环境）
- `application-test.properties`（测试环境）
- `application-prod.properties`（生产环境）

然后在的`application.properties`或`application.yml`中指定激活的配置文件：

```properties
# application.properties
spring.profiles.active=dev
```

或者在启动命令行中

```bash
java -jar yourapp.jar --spring.profiles.active=dev
```

设定好开发环境后，`logback-spring.xml`就可以根据不同的开发环境选择日志配置：

```xml
<!-- 开发环境配置 -->
<springProfile name="dev">
    <root level="DEBUG">
        <appender-ref ref="CONSOLE"/>
    </root>
</springProfile>

<!-- 生产环境配置 -->
<springProfile name="prod">
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</springProfile>

<!-- 默认配置（未指定 profile 时使用） -->
<root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
</root>

```

*注：只有`logback-spring.xml`支持`<springProfile>`等命令，`logback.xml`不支持。*
