---
title: SpringBoot启动流程
category: springboot
---

> Spring Boot 是一个开源的 Java 框架，用于简化 Spring 应用程序的开发。它是 Spring 生态系统的一部分，旨在帮助开发人员更快速和方便地创建独立、生产级的 Spring 应用。
>
> * 快速入门：Spring Boot 通过约定优于配置的原则，使开发者可以快速上手，无需繁琐的 XML 配置。
>
> * 自带嵌入式服务器：Spring Boot 支持嵌入式服务器（如 Tomcat Jetty 和 Undertow），开发者无需单独部署应用服务器，可以直接运行 Java 应用。
>
> * 自动配置：Spring Boot 提供了自动配置功能，根据项目的类路径和配置自动设置 Spring 应用的各种组件，减少了大量的配置工作。
>
> * 生产级特性：Spring Boot 内置了许多生产环境所需的功能，如健康检查、指标监控、应用配置管理等。
>
> * 开箱即用：Spring Boot 提供了一系列的 starter 依赖，简化了依赖管理。例如，spring-boot-starter-web 可以帮助快速构建基于 Spring MVC 的 web 应用。
>
> * 支持微服务架构：Spring Boot 适合构建微服务应用，结合 Spring Cloud，能够实现分布式系统的开发。1.

# 1.`@SpringBootApplication`注解

```java
@SpringBootApplication
public class LearnApplication {

    public static void main(String[] args) {
        SpringApplication.run(LearnApplication.class, args);
    }

}
```

SpringBoot启动类中包含`@SpringBootApplication`注解和`SpringApplication.run(LearnApplication.class, args);`方法。SpringBoot的入口是main方法。本章节先讲解`@SpringBootApplication`注解。

> 在 Java 中，**注解（Annotation）** 是一种元数据的形式，可以添加到代码中（例如类、方法、字段等），以提供额外的信息。注解本身并不直接影响代码的执行逻辑，但可以通过反射机制或编译时处理来实现特定的功能。
>
> 注解主要分为：
>
> * **元注解**：用于定义其他注解的行为  （`@Retention`,`@Target`,`@Documented`,`@Inherited`）
>
> * **内置注解**：该类注解的执行器被内置到了Javac中，由Javac负责解析注解。（`@Override`,`@Deprecated`,`@SuppressWarnings`）
>
> * **自定义注解**：使用元注解自定义注解，需要编写*注解处理器*来处理该类注解。自定义注解的处理取决于`@Retention`定义的阶段。
>
>   * `RetentionPolicy.SOURCE`：注解仅保留在源代码中，编译后丢弃。该类注解由编译器或源码级工具在编译时处理。
>
>     注：内置注解也属于该类，但内置注解是嵌入到编译器中的语义分析和语法分析中，属于编译器的一部分。但该类的自定义注解的处理需要通过**注解处理器（Annotation Processor）**处理，在 javac 编译时，注解处理器会扫描源代码中的注解，并根据注解信息执行特定逻辑，例如生成新代码、验证规则或抛出错误，比较常用的就是Lombok的注解。
>
>   * `RetentionPolicy.CLASS`：注解保留在编译后的 .class 文件中，但运行时不可见。也是由**注解处理器（Annotation Processor）**处理。用于生成代码或验证，可用在构建工具中，一般不常用。
>
>   * `RetentionPolicy.RUNTIME`：注解保留到运行时，可通过反射访问。由程序在运行时通过反射动态读取和处理。比较常见的就是各种框架，如SpringBoot。

通过`@SpringBootApplication`注解的源码注释可以得出，其为一个组合注解，组合了`@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`。经过后面的源码分析，可以得出SpringBoot并不是直接处理`@SpringBootApplication`而是处理他的三个组合注解。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
	Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;
    
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;
}
```

* `@SpringBootConfiguration`组合了`@Configuration`，表明是一个Java配置类
* `@EnableAutoConfiguration` 组合了`@AutoConfigurationPackage`和`@Import(AutoConfigurationImportSelector.class)`用于将所有符合配置条件的 Bean 都加载到 IOC 容器中
* `@ComponentScan`指明了包搜索路径以及需要过滤的类

# 2.`SpringApplication.run(LearnApplication.class, args);`

`SpringApplication.run(LearnApplication.class, args);`是整个SpringBoot的入口。查看源码：

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

* 创建`ConfigurableApplicationContext`实例
* 运行该实例

## SpringAplplication构造函数

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //判断WEB应用的类型
    this.properties.setWebApplicationType(WebApplicationType.deduceFromClasspath());
    // 将类型为BootstrapRegistryInitializer的初始化类数组赋值给this.bootstrapRegistryInitializers
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    //将类型为ApplicationContextInitializer的初始化类数组赋值给this.initializers
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //将类型为ApplicationListener的监听类数组赋值给this.listeners
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 从调用栈中拿到main方法所在的类
    this.mainApplicationClass = deduceMainApplicationClass();
}

private Class<?> deduceMainApplicationClass() {
    //使用Java9引入的StackWalker遍历当前线程的调用栈
    return StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE)
        // 找到第一个包含main方法的类并返回它的Class<?>
        .walk(this::findMainClass)
        //找不到返回null
        .orElse(null);
}
private Optional<Class<?>> findMainClass(Stream<StackFrame> stack) {
    return stack.filter((frame) -> Objects.equals(frame.getMethodName(), "main"))
        .findFirst()
        .map(StackWalker.StackFrame::getDeclaringClass);
}
```

SpringApplication 构造函数的参数是资源加载器和主程序源（SpringBoot启动类）

```java
//判断WEB应用的类型
this.properties.setWebApplicationType(WebApplicationType.deduceFromClasspath());
```

`WebApplicationType`有三种类型：

* `NONE`：程序不是WEB应用，不需要启动内置WEB服务器
* `SERVLET`： 程序是WEB应用，需要启动WEB服务器
* `REACTIVE`：程序是响应式WEB应用，启动REACTIVE WEB服务器

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "jakarta.servlet.Servlet",
        "org.springframework.web.context.ConfigurableWebApplicationContext" };

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

判断方法很简单，就是寻找在Classpath加载路径上有某些Class，比如有`DispatcherHandler`且没有`DispatcherServlet`和`ServletContainer`则是响应式WEB应用。如果包含`Servlet` `ConfigurableWebApplicationContext`则是Servlet WEB应用。否则就不是WEB应用。

下面三个赋值中都使用了`getSpringFactoriesInstances()`用于获取Spring工厂对象。即将对应的class文件加载到内存对象。

```java
private <T> List<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, null);
}

private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);
}
```

### `getSpringFactoriesInstances()`

> classpath 是Java的系统变量，Javac通过classpath寻找class文件或依赖包。classpath默认路径是当前目录下（`./`）以及`javac -cp xxx`指定的路径。
>
> 在SpringBoot中，对classpath进行了增强：
>
> * 开发模式：classpath 包含了`target/classes/`(编译前来自于`src/main/resources/`)和Maven从本地仓库中获取的依赖包（`~/.m2/repository/xxx.jar`)。此模式下Maven会解析pom.xml，并自动添加`~/.m2/repository/` 里的 JAR 到 classpath
>
> *  Spring Boot Fat JAR 模式：此时SpringBoot不使用classpath，而是用 `LaunchedURLClassLoader` 来解析 `BOOT-INF/lib/` 里的 JAR。此模式下Maven会将依赖包从`~/.m2/repository/`中直接复制到`target/myapp-1.0.0.jar/BOOT-INF/lib/`中。
>
>   ```bash
>   target/myapp-1.0.0.jar
>    ├── BOOT-INF/classes/        # 编译后的 .class 文件
>    ├── BOOT-INF/lib/            # 所有依赖的 JAR
>    ├── META-INF/MANIFEST.MF     # 启动信息
>    ├── org/springframework/...  # Spring 相关类
>   ```
>
> 注：开发模式中，也可以不使用Maven的启动，直接使用Java命令。编译之后执行`java -cp target/classes:$(mvn dependency:build-classpath) com.example.MainApplication`，使用`-cp`参数将`target/classes`和`~/.m2/repository/`下的依赖包添加到classpath中，也可以正常启动项目。
>
> ```bash
> //maven中查看依赖
> mvn dependency:tree
> //输出依赖的完整路径
> mvn dependency:build-classpath
> ```

了解了classpath工作原理，重点讲解`SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);`该类是利用`classloader`将classpath中的所有满足要求的类都加载进来。

```java
public static SpringFactoriesLoader forDefaultResourceLocation(@Nullable ClassLoader classLoader) {
    return forResourceLocation(FACTORIES_RESOURCE_LOCATION, classLoader);
}
public static SpringFactoriesLoader forResourceLocation(String resourceLocation, @Nullable ClassLoader classLoader) {
    Assert.hasText(resourceLocation, "'resourceLocation' must not be empty");
    //获取classloader
    ClassLoader resourceClassLoader = (classLoader != null ? classLoader :
            SpringFactoriesLoader.class.getClassLoader());
    //从缓存中查看该classloader是否存在，如果没有创建一个cache[classloader]=hashmap<string, SpringFactoriesLoader> 
    //并将该hashmap返回给loaders
    Map<String, SpringFactoriesLoader> loaders = cache.computeIfAbsent(
            resourceClassLoader, key -> new ConcurrentReferenceHashMap<>());
    //获取loaders["META-INF/spring.factories"]，如果不存在，创建一个新的SpringFactoriesLoader
    //参数1:classloader
    //参数2:factories : Map<String, List<String>>
    return loaders.computeIfAbsent(resourceLocation, key ->
            new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)));
}
```

其中`FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"`，ClassLoader是默认的类加载器 `Thread.currentThread().getContextClassLoader();` 通过调试`classloader = ClassLoaders$AppClassLoader`

`loadFactoriesResource()`会将符合条件的类加载进来。

```java
//classloader = ClassLoaders$AppClassLoader
//resourceLocation = "META-INF/spring.factories"
protected static Map<String, List<String>> loadFactoriesResource(ClassLoader classLoader, String resourceLocation) {
    	//result中包含[初始化类型，对应需要加载的所有初始化类]
		Map<String, List<String>> result = new LinkedHashMap<>();
		try {
            //classLoader将classpath路径下的所有的resourceLocation文件都加载到urls中
            //urls中包含了所有META-INF/spring.factories的文件路径
			Enumeration<URL> urls = classLoader.getResources(resourceLocation);
			while (urls.hasMoreElements()) {
                //根据urls中的文件路径加载到UrlResource中
                //举个例子 resource=jar:file:/home/amber/.m2/repository/org/springframework/boot/spring-boot/3.4.3/spring-boot-3.4.3.jar!/META-INF/spring.factories
				UrlResource resource = new UrlResource(urls.nextElement());
                //将文件中的所有属性加载到properties中
                //Properties 是一个 ConcurrentHashMap<Object, Object>
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                //遍历所有属性
				properties.forEach((name, value) -> {
                    //以逗号分割，将value字符串分割为String[]数组
					String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String) value);
                    //将{name: ArrayList<String>{}}放入到result中,ArrayList的长度就是value数组的长度
					List<String> implementations = result.computeIfAbsent(((String) name).trim(),
							key -> new ArrayList<>(factoryImplementationNames.length));
                    //将factoryImplementationNames流化，将每一个元素都trim后再将每一个元素添加到implementations中
					Arrays.stream(factoryImplementationNames).map(String::trim).forEach(implementations::add);
				});
			}
            //对每一种初始化类型中的所有初始化类去重
			result.replaceAll(SpringFactoriesLoader::toDistinctUnmodifiableList);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" + resourceLocation + "]", ex);
		}
    	//返回一个不可以修改的Map，最终赋值给了SpringFactoriesLoader.factories
		return Collections.unmodifiableMap(result);
	}
```

```properties
# file:/home/amber/.m2/repository/org/springframework/boot/spring-boot/3.4.3/spring-boot-3.4.3.jar!/META-INF/spring.factories
# Logging Systems
org.springframework.boot.logging.LoggingSystemFactory=\
org.springframework.boot.logging.logback.LogbackLoggingSystem.Factory,\
org.springframework.boot.logging.log4j2.Log4J2LoggingSystem.Factory,\
org.springframework.boot.logging.java.JavaLoggingSystem.Factory

# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
.....
```

到目前已经通过`SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader())`得到了`SpringFactoriesLoader`对象，其中包含了`SpringFactoriesLoader.classLoader`,`SpringFactoriesLoader.factories`。后续调用了该对象的`load(type, argumentResolver)`方法

```java
public <T> List<T> load(Class<T> factoryType, @Nullable FailureHandler failureHandler) {
    return load(factoryType, null, failureHandler);
}

public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver,
        @Nullable FailureHandler failureHandler) {

    Assert.notNull(factoryType, "'factoryType' must not be null");
    //构造函数中已经把{初始化类型：[对应类型的实现类数组]}放入到factories中
    //取出初始化类型为factoryType的类数组
    List<String> implementationNames = loadFactoryNames(factoryType);
    logger.trace(LogMessage.format("Loaded [%s] names: %s", factoryType.getName(), implementationNames));
    //结果是一个类型为T的列表
    List<T> result = new ArrayList<>(implementationNames.size());
    FailureHandler failureHandlerToUse = (failureHandler != null) ? failureHandler : THROWING_FAILURE_HANDLER;
    //遍历所有的实现类
    for (String implementationName : implementationNames) {
        //将实现类实例化
        T factory = instantiateFactory(implementationName, factoryType, argumentResolver, failureHandlerToUse);
        //将实例化的实现类放入到result中
        if (factory != null) {
            result.add(factory);
        }
    }
    //根据@Order @Priority对实现类进行排序
    AnnotationAwareOrderComparator.sort(result);
    return result;
}

private List<String> loadFactoryNames(Class<?> factoryType) {
    return this.factories.getOrDefault(factoryType.getName(), Collections.emptyList());
}

@Nullable
protected <T> T instantiateFactory(String implementationName, Class<T> type,
        @Nullable ArgumentResolver argumentResolver, FailureHandler failureHandler) {

    try {
        //使用class.forName(xxx, false, zzz)将xxx类加载到元数据区，只类加载不执行类初始化
        Class<?> factoryImplementationClass = ClassUtils.forName(implementationName, this.classLoader);
        Assert.isTrue(type.isAssignableFrom(factoryImplementationClass), () ->
                "Class [%s] is not assignable to factory type [%s]".formatted(implementationName, type.getName()));
        //创建一个用于初始化factoryImplementationClass类的内部创建类（该类包含了factoryImplementationClass的构造方法）
        FactoryInstantiator<T> factoryInstantiator = FactoryInstantiator.forClass(factoryImplementationClass);
        // 处理构造函数所需的参数，使用this.constructor.newInstance(args)创建factoryImplementationClass类
        return factoryInstantiator.instantiate(argumentResolver);
    }
    catch (Throwable ex) {
        failureHandler.handleFailure(type, implementationName, ex);
        return null;
    }
}
```

到目前为止，得出了`getSpringFactoriesInstances(xxx.class)`方法是将初始化类型为xxx.class的类从`META-INF/spring.factories`中提取出来，并返回该类型的所有的实现类，返回的是一个xxx.class类型的数组。

`ConfigurableApplicationContext`后续的三个赋值都是利用`getSpringFactoriesInstances(xxx.class)`将初始化类数组赋值给类成员，最后一步是从调用栈中获取main方法所在的类。

# 2. 执行`SpringApplication`的run方法

`SpringApplication`构造函数中都是一些赋值操作，run方法才是真正启动SpringBoot项目。

```java
public ConfigurableApplicationContext run(String... args) {
    // 初始化启动时钟
    Startup startup = Startup.create();
    // 检查properties中是否注册关闭钩子
    if (this.properties.isRegisterShutdownHook()) {
        //启 Shutdown Hook注册机制，保证Spring Boot应用在JVM关闭时能够正确释放资源（如数据库连接池、线程池等）。
        SpringApplication.shutdownHook.enableShutdownHookAddition();
    }
    // 创建引导上下文
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    // 创建应用上下文
    ConfigurableApplicationContext context = null;
    // 将环境变量中java.awt.headless设置为原值或者默认true
    configureHeadlessProperty();
    // 用getSpringFactoriesInstances()方法获取了所有类型为
    // SpringApplicationRunListener.class的启动监听类，
    // 将其封装在SpringApplicationRunListeners类中
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 将bootstrapContext传递给所有的启动监听类
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        // 解析命令行传入的参数，并封装到ApplicationArguments
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        // 打印Banner 根据条件 判断打印到log还是console，是使用默认值还是指定banner
        Banner printedBanner = printBanner(environment);
        // 根据WebApplicationType创建 应用程序上下文
        // 获取META-INF/spring.factories中的ApplicationContextFactory接口实现类
        // ReactiveWebServerApplicationContextFactory 创建了 ReactiveWebServerApplicationContext
        // ServletWebServerApplicationContextFactory 创建了 ServletWebServerApplicationContext
        context = createApplicationContext();
        // 将applicationStartup赋值给context
        // applicationStartup 用于收集和分析应用的启动性能数据，帮助开发者优化 Spring Boot 启动过程
        context.setApplicationStartup(this.applicationStartup);
        // 准备 应用程序上下文
        // 将参数绑定到应用程序上下文（ApplicationContext）为应用程序启动运行做好准备
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 启动 应用上下文 （SpringBoot核心启动方法）
        refreshContext(context);
        // 刷新之后需要的处理，默认是空
        afterRefresh(context, applicationArguments);
        //SpringBoot启动
        startup.started();
        //日志	
        if (this.properties.isLogStartupInfo()) {
            new StartupInfoLogger(this.mainApplicationClass, environment).logStarted(getApplicationLog(), startup);
        }
        // 将ConfigurableApplicationContext传递给所有的启动监听类，表明started
        listeners.started(context, startup.timeTakenToStarted());
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        throw handleRunFailure(context, ex, listeners);
    }
    try {
        if (context.isRunning()) {
            listeners.ready(context, startup.ready());
        }
    }
    catch (Throwable ex) {
        throw handleRunFailure(context, ex, null);
    }
    return context;
}
```

## 创建引导程序上下文`createBootstrapContext()`

引导程序上下文的作用：**在 `SpringApplication` 启动过程中，提前注册某些组件，以便在应用上下文（`ApplicationContext`）正式创建时使用**。

`DefaultBootstrapContext` 存在的时间非常短，它只在 `SpringApplication.run()` 方法执行的早期阶段存在，等 `ApplicationContext` 初始化完成后，它就会被丢弃。

引导上下文主要用于支持 **Spring Cloud** 的配置加载机制（例如 Spring Cloud Config）。它并不是 Spring Boot 核心的一部分，而是 Spring Cloud 提供的高级功能。

```java
DefaultBootstrapContext bootstrapContext = createBootstrapContext();

private DefaultBootstrapContext createBootstrapContext() {
    // 创建引导上下文
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    // 构造函数中已经将BootstrapRegistryInitializer类型的初始化类型数组赋值给了this.bootstrapRegistryInitializers
    // 遍历初始化类型数组，并执行每一个初始化类型的initialize方法，将引导上下文DefaultBootstrapContext赋值给它
    this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
    return bootstrapContext;
}
```

## 监听类`SpringApplicationRunListeners`

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);
    argumentResolver = argumentResolver.and(String[].class, args);
    // 通过getSpringFactoriesInstances()获取所有接口为SpringApplicationRunListener.class的实现类
    List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,
            argumentResolver);
    SpringApplicationHook hook = applicationHook.get();
    SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;
    if (hookListener != null) {
        listeners = new ArrayList<>(listeners);
        listeners.add(hookListener);
    }
    // 将监听类List和applicationStartup传递給SpringApplicationRunListeners
    // 其中this.applicationStartup是一个性能分析工具，默认状态下，它什么都没做，是一个空实现。
    return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);
}
```

比如`starting()`方法赋值监听名称为`"spring.boot.application.starting"`并将bootstrapContext通过`listener.starting()`赋值给每一个监听类。

```java
class SpringApplicationRunListeners {

	private final Log log;

	private final List<SpringApplicationRunListener> listeners;

	private final ApplicationStartup applicationStartup;

	SpringApplicationRunListeners(Log log, List<SpringApplicationRunListener> listeners,
			ApplicationStartup applicationStartup) {
		this.log = log;
		this.listeners = List.copyOf(listeners);
		this.applicationStartup = applicationStartup;
	}

	void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {
		doWithListeners("spring.boot.application.starting", (listener) -> listener.starting(bootstrapContext),
				(step) -> {
					if (mainApplicationClass != null) {
						step.tag("mainApplicationClass", mainApplicationClass.getName());
					}
				});
	}

	void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
		doWithListeners("spring.boot.application.environment-prepared",
				(listener) -> listener.environmentPrepared(bootstrapContext, environment));
	}

	void contextPrepared(ConfigurableApplicationContext context) {
		doWithListeners("spring.boot.application.context-prepared", (listener) -> listener.contextPrepared(context));
	}

	void contextLoaded(ConfigurableApplicationContext context) {
		doWithListeners("spring.boot.application.context-loaded", (listener) -> listener.contextLoaded(context));
	}

	void started(ConfigurableApplicationContext context, Duration timeTaken) {
		doWithListeners("spring.boot.application.started", (listener) -> listener.started(context, timeTaken));
	}

	void ready(ConfigurableApplicationContext context, Duration timeTaken) {
		doWithListeners("spring.boot.application.ready", (listener) -> listener.ready(context, timeTaken));
	}

	void failed(ConfigurableApplicationContext context, Throwable exception) {
		doWithListeners("spring.boot.application.failed",
				(listener) -> callFailedListener(listener, context, exception), (step) -> {
					step.tag("exception", exception.getClass().toString());
					step.tag("message", exception.getMessage());
				});
	}

	private void callFailedListener(SpringApplicationRunListener listener, ConfigurableApplicationContext context,
			Throwable exception) {
		try {
			listener.failed(context, exception);
		}
		catch (Throwable ex) {
			if (exception == null) {
				ReflectionUtils.rethrowRuntimeException(ex);
			}
			if (this.log.isDebugEnabled()) {
				this.log.error("Error handling failed", ex);
			}
			else {
				String message = ex.getMessage();
				message = (message != null) ? message : "no error message";
				this.log.warn("Error handling failed (" + message + ")");
			}
		}
	}

	private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction) {
		doWithListeners(stepName, listenerAction, null);
	}
	
    //每一个方法都调用了doWithListeners()
    //参数1：步骤名称
    //参数2：消费型函数式接口，它可以接收一个 SpringApplicationRunListener 对象，并对其执行某些操作。
	private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
			Consumer<StartupStep> stepAction) {
        // 性能监听，默认是空函数
		StartupStep step = this.applicationStartup.start(stepName);
        // listeners是SpringApplicationRunListener对象的List
        // 对listeners中的每一个SpringApplicationRunListener对象执行listenerAction方法。
		this.listeners.forEach(listenerAction);
		if (stepAction != null) {
			stepAction.accept(step);
		}
		step.end();
	}
}

```

## 准备环境 `prepareEnvironment()`

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 根据应用类型创建对应的环境类
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 将命令行的参数添加到Springboot环境变量中
    // 配置spring.profiles.active
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 将 ConfigurationPropertySources 附加到 Environment，让 Spring Environment 识别 ConfigurationPropertySource
    ConfigurationPropertySources.attach(environment);
    // 告诉所有的监听类，环境准备阶段
    listeners.environmentPrepared(bootstrapContext, environment);
    ApplicationInfoPropertySource.moveToEnd(environment);
    DefaultPropertiesPropertySource.moveToEnd(environment);
    Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
            "Environment prefix cannot be set via properties.");
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
        environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    // 在构造函数中已经赋值，此时拿出webapplication的类型
    WebApplicationType webApplicationType = this.properties.getWebApplicationType();
    // 根据不同的webapplication类型创建不同的ConfigurableEnvironment
    // DefaultApplicationContextFactory
    // ServletWebServerApplicationContextFactory
    // ReactiveWebServerApplicationContextFactory
    // 会从META/spring.factories中获取ApplicationContextFactory并根据webApplicationType选取对应的类
    ConfigurableEnvironment environment = this.applicationContextFactory.createEnvironment(webApplicationType);
    if (environment == null && this.applicationContextFactory != ApplicationContextFactory.DEFAULT) {
        environment = ApplicationContextFactory.DEFAULT.createEnvironment(webApplicationType);
    }
    return (environment != null) ? environment : new ApplicationEnvironment();
}
```

## 准备应用上下文 `prepareContext()`

将参数绑定到应用程序上下文（ApplicationContext），为应用程序启动运行做好准备。

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    // environment赋值给context
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    addAotGeneratedInitializerIfNecessary(this.initializers);
    applyInitializers(context);
    //触发所有SpringApplicationRunListerner监听器的contextPrepared
    listeners.contextPrepared(context);
    // 发送事件 当 bootstrapContext关闭且ApplicationContext准备好后发送该事件
    bootstrapContext.close(context);
    //日志处理
    if (this.properties.isLogStartupInfo()) {
        logStartupInfo(context.getParent() == null);
        logStartupInfo(context);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    // 通过 应用上下文 获取BeanFactory
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 通过 BeanFactory 注册单例对象 springApplicationArguments
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        // 通过 BeanFactory 注册单例对象 springBootBanner
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
        autowireCapableBeanFactory.setAllowCircularReferences(this.properties.isAllowCircularReferences());
        if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
            listableBeanFactory.setAllowBeanDefinitionOverriding(this.properties.isAllowBeanDefinitionOverriding());
        }
    }
    // 懒加载 给应用上下文添加懒加载BeanFactory处理类
    if (this.properties.isLazyInitialization()) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // 
    if (this.properties.isKeepAlive()) {
        context.addApplicationListener(new KeepAlive());
    }
    // 
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    if (!AotDetector.useGeneratedArtifacts()) {
        // Load the sources
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        load(context, sources.toArray(new Object[0]));
    }
    //触发所有SpringApplicationRunListerner监听器的contextLoaded
    listeners.contextLoaded(context);
}
```

## 启动应用上下文`refreshContext()`

```java
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.properties.isRegisterShutdownHook()) {
        shutdownHook.registerApplicationContext(context);
    }
    refresh(context);
}
protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
}
```

其中`applicationContext`有两种类：

* `ReactiveWebServerApplicationContext`

* `ServletWebServerApplicationContext`

他们的共同抽象父类`AbstractApplicationContext`，主要的refresh()方法也是在这个抽象父类中定义的。

`refresh()` 是 Spring 容器启动的核心方法，负责加载 Bean、初始化 BeanFactory、注册监听器、完成 Bean 初始化，并最终发布 `ContextRefreshedEvent` 事件

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    this.startupShutdownLock.lock();
    try {
        this.startupShutdownThread = Thread.currentThread();

        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // Prepare this context for refreshing.
        // 检查SpringBoot中的环境变量
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 从子类中获取beanfactory，默认是ConfigurableListableBeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        //为 ConfigurableListableBeanFactory配置一些基础设置和功能，以确保它能够正确地创建和管理 Bean
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 允许子类对beanFactory中的bean在初始化之前修改bean
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // Invoke factory processors registered as beans in the context.
            // 处理注解的主要方法
            invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
            // 注册BeanPostProcessors的bean类
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // Initialize message source for this context.
            // 添加消息bean
            initMessageSource();

            // Initialize event multicaster for this context.
            // 添加事件传播bean
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 处理子类的刷新，如Servlet的启动Tomcat
            onRefresh();

            // Check for listener beans and register them.
            // 注册监听bean
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 初始化所有的 单例bean
            // 并将bean锁住
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (RuntimeException | Error ex ) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            contextRefresh.end();
        }
    }
    finally {
        this.startupShutdownThread = null;
        this.startupShutdownLock.unlock();
    }
}
```



```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    // 为 BeanFactory 设置当前的类加载器,以便在加载 Bean 定义时使用
    beanFactory.setBeanClassLoader(getClassLoader());
    // 用于支持 Spring 表达式语言（SpEL），例如在 @Value 注解中解析表达式。
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 用于处理资源（如 Resource 类型）的注入和解析
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 实现了 ApplicationContextAware 接口的 Bean 能够感知到当前的 ApplicationContext
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 配置 BeanFactory 忽略一些特定的 Aware 接口的依赖自动注入,因为这些接口的实现通常由 Spring 框架自动处理。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 为一些常见的类型注册默认的单例 Bean, 可以通过依赖注入直接获取对应的实例
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
}
```

## Bean扫描注册`invokeBeanFactoryPostProcessors()`

通过`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());`按优先级调用所有注册的 `BeanFactoryPostProcessor` 和 `BeanDefinitionRegistryPostProcessor`，完成对 Bean 定义的修改。

该方法主要完成BeanDefinition的扫描、解析、注册：

1.实例化实现了BeanDefinitionRegistryPostProcessor接口的Bean

2.调用实现了BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry()方法

3.实例化实现了BeanFactoryPostProcessor接口的Bean

4.调用实现了BeanFactoryPostProcessor的postProcessBeanFactory()方法

```java
public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // WARNING: Although it may appear that the body of this method can be easily
    // refactored to avoid the use of multiple loops and multiple lists, the use
    // of multiple lists and multiple passes over the names of processors is
    // intentional. We must ensure that we honor the contracts for PriorityOrdered
    // and Ordered processors. Specifically, we must NOT cause processors to be
    // instantiated (via getBean() invocations) or registered in the ApplicationContext
    // in the wrong order.
    //
    // Before submitting a pull request (PR) to change this method, please review the
    // list of all declined PRs involving changes to PostProcessorRegistrationDelegate
    // to ensure that your proposal does not result in a breaking change:
    // https://github.com/spring-projects/spring-framework/issues?q=PostProcessorRegistrationDelegate+is%3Aclosed+label%3A%22status%3A+declined%22

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();

    if (beanFactory instanceof BeanDefinitionRegistry registry) {
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor registryProcessor) {
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, for example, replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
```



# 注解处理

 `prepareBeanFactory` 阶段，Spring 会为 BeanFactory 添加一些默认的处理器，这些处理器会影响后续注解的处理。例如：

- 添加 `ApplicationContextAwareProcessor`，用于处理 Aware 接口（如 ApplicationContextAware）。

- 添加 `BeanNameAware` 和 `BeanFactoryAware` 的支持。

  

`invokeBeanFactoryPostProcessors` 阶段的核心处理器是 `ConfigurationClassPostProcessor`，它实现了 `BeanFactoryPostProcessor` 接口，负责解析配置类及其相关注解。它的执行逻辑包括：

- 解析主配置类（例如带有 `@SpringBootApplication` 的类）。
- 处理 `@ComponentScan`，扫描并注册组件。
- 处理 `@Import`，加载导入的配置类或执行 ImportSelector/ImportBeanDefinitionRegistrar。
- 处理 `@Bean`，将配置类中的 @Bean 方法注册为 Bean 定义。



`registerBeanPostProcessors` 阶段，Spring 会注册所有 `BeanPostProcessor`，这些处理器负责处理与 Bean 初始化和生命周期相关的注解。典型注解包括：

- **`@Autowired`** 和 **@Inject**：由 `AutowiredAnnotationBeanPostProcessor` 处理，用于依赖注入。
- `@Value`：由 `AutowiredAnnotationBeanPostProcessor` 处理，用于属性值注入。
- **`@PostConstruct`** 和 **@PreDestroy**：由 `CommonAnnotationBeanPostProcessor` 处理，用于生命周期回调。
- **`@Resource`**：由 `CommonAnnotationBeanPostProcessor` 处理，用于 JSR-250 注解的依赖注入。



`finishBeanFactoryInitialization` 阶段，Spring 会实例化所有单例 Bean，并触发 BeanPostProcessor 的处理逻辑。这一阶段会实际执行依赖注入、初始化回调等操作。典型注解包括：

- **`@Autowired`**：注入依赖。
- **`@PostConstruct`**：执行初始化方法。
- **`@EventListener`**：由 EventListenerMethodProcessor 处理，注册事件监听器。



Spring Boot 还扩展了许多自定义注解，这些注解的处理逻辑分布在不同的阶段。例如：

- **`@ConditionalOnClass`,`@ConditionalOnMissingBean` 等条件注解**：由 ConditionEvaluator 在 `invokeBeanFactoryPostProcessors` 阶段处理，用于决定是否加载某些配置类或 Bean。
- **`@SpringBootApplication`**：由 `ConfigurationClassPostProcessor` 在 `invokeBeanFactoryPostProcessors` 阶段处理，分解为 `@Configuration` `@ComponentScan` 和 `@EnableAutoConfiguration`。
- **`@RestController`** 和 **`@RequestMapping`**：由 Spring MVC 相关的 BeanPostProcessor（如 RequestMappingHandlerMapping）在 `finishBeanFactoryInitialization` 阶段处理，用于注册控制器和映射请求路径。

| 注解类型                 | 处理阶段                                                     | 核心处理器                           |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------ |
| @Configuration           | invokeBeanFactoryPostProcessors                              | ConfigurationClassPostProcessor      |
| @ComponentScan           | invokeBeanFactoryPostProcessors                              | ConfigurationClassPostProcessor      |
| @Import                  | invokeBeanFactoryPostProcessors                              | ConfigurationClassPostProcessor      |
| @EnableAutoConfiguration | invokeBeanFactoryPostProcessors                              | AutoConfigurationImportSelector      |
| @ConditionalOnClass 等   | invokeBeanFactoryPostProcessors                              | ConditionEvaluator                   |
| @Bean                    | invokeBeanFactoryPostProcessors                              | ConfigurationClassPostProcessor      |
| @Autowired               | registerBeanPostProcessors / finishBeanFactoryInitialization | AutowiredAnnotationBeanPostProcessor |
| @Value                   | registerBeanPostProcessors / finishBeanFactoryInitialization | AutowiredAnnotationBeanPostProcessor |
| @PostConstruct           | registerBeanPostProcessors / finishBeanFactoryInitialization | CommonAnnotationBeanPostProcessor    |
| @EventListener           | finishBeanFactoryInitialization                              | EventListenerMethodProcessor         |
| @RestController          | finishBeanFactoryInitialization                              | RequestMappingHandlerMapping         |
| @RequestMapping          | finishBeanFactoryInitialization                              | RequestMappingHandlerMapping         |
