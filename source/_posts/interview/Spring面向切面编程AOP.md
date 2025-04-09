---
title: Spring面向切面编程(AOP)
category: interview
---

# 面向切面编程(AOP) VS 面向对象编程（OOP）

> OOP面向对象编程，针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分。而AOP则是针对业务处理过程中的切面进行提取，它所面对的是处理过程的某个步骤或阶段，以获得逻辑过程的中各部分之间低耦合的隔离效果。
>
> * OOP：从纵向解决代码重复问题
> * AOP：从横向解决代码重复问题，取代了传统从纵向继承体系重复代码（性能监控、事务管理、缓存，日志等）将业务逻辑与系统处理代码进行解耦。

![image-20250404085620522](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250404085620522.png)

图左结构为正常AOP架构，日志处理会被加入到每一个方法中。图右结构为AOP架构，日志处理会被抽离出来，方法中不需要处理日志，会被AOP以切面的形式添加到每个方法开始之前。

* **连接点（Jointpoint）**：表示需要在程序中插入横切关注点的扩展点，**连接点可能是类初始化、方法执行、方法调用、字段调用或处理异常等等**，*Spring只支持方法执行连接点*，在AOP中表示为**在哪里干**；

* **切入点（Pointcut）**： 选择一组相关连接点的模式，即可以认为连接点的集合，Spring支持perl5正则表达式和AspectJ切入点模式，Spring默认使用AspectJ语法，在AOP中表示为**在哪里干的集合**；

* **通知（Advice）**：在连接点上执行的行为，通知提供了在AOP中需要在切入点所选择的连接点处进行扩展现有行为的手段；包括前置通知（before advice）、后置通知(after advice)、环绕通知（around advice），在Spring中通过代理模式实现AOP，并通过拦截器模式以环绕连接点的拦截器链织入通知；在AOP中表示为**干什么**。

* **切面（Aspect）**：横切关注点的模块化，比如上边提到的日志组件。可以认为是通知和切入点的组合；在Spring中可以使用Schema和@AspectJ方式进行组织实现；在AOP中表示为**在哪干和干什么集合**。

# Spring AOP和AspectJ

> AspectJ是一个Java实现的AOP框架，它能够对Java代码进行AOP编译（一般在编译期进行），让Java代码具有AspectJ的AOP功能（需要特殊的编译器）

> Spring 框架通过定义切面, 通过拦截切点实现了不同业务模块的解耦，这个就叫**面向切面编程 - Aspect Oriented Programming (AOP)**。
>
> AOP的本质也是为了解耦，通过**预编译方式**和**运行期间动态代理**实现程序的统一维护的一种技术。

区别：

* AspectJ是更强的AOP框架，是实际意义的**AOP标准**；

* Spring AOP使用纯Java实现, 它不需要专门的编译过程, 它一个**重要的原则就是无侵入性（non-invasiveness）**。

* Spring小组喜欢@AspectJ注解风格更胜于Spring XML配置，在Spring 2.0使用了和AspectJ 5一样的注解，并使用AspectJ来做切入点解析和匹配。**但是，AOP在运行时仍旧是纯的Spring AOP，并不依赖于AspectJ的编译器或者织入器（weaver）**。

| Spring AOP                                       | AspectJ                                                      |
| ------------------------------------------------ | ------------------------------------------------------------ |
| 在纯 Java 中实现                                 | 使用 Java 编程语言的扩展实现                                 |
| 不需要单独的编译过程                             | 除非设置 LTW，否则需要 AspectJ 编译器 (ajc)                  |
| 只能使用运行时织入                               | 运行时织入不可用。支持编译时、编译后和加载时织入             |
| 功能不强-仅支持方法级编织                        | 更强大 - 可以编织字段、方法、构造函数、静态初始值设定项、最终类/方法等 |
| 只能在由 Spring 容器管理的 bean 上实现           | 可以在所有域对象上实现                                       |
| 仅支持方法执行切入点                             | 支持所有切入点                                               |
| 代理是由目标对象创建的, 并且切面应用在这些代理上 | 在执行应用程序之前 (在运行时) 前, 各方面直接在代码中进行织入 |
| 比 AspectJ 慢多了                                | 更好的性能                                                   |
| 易于学习和应用                                   | 相对于 Spring AOP 来说更复杂                                 |

织入器（weaver）：织入分为动态织入和静态织入，织入可以理解为将切面应用到目标函数中，即定义切面后，需要将切面织入到目标代码内。

* **动态织入**的方式是在运行时动态将要增强的代码织入到目标类中，这样往往是通过动态代理技术完成的，如Java JDK的动态代理(Proxy，底层通过反射实现)或者CGLIB的动态代理(底层通过继承实现)，Spring AOP采用的就是基于运行时增强的代理技术

* ApectJ采用的就是**静态织入**的方式。ApectJ主要采用的是编译期织入，在这个期间使用AspectJ的acj编译器(类似javac)把aspect类编译成class字节码后，在java目标类编译时织入，即先编译aspect类再编译目标类。

# Spring AOP 

> Spring AOP 支持对XML模式和基于@AspectJ注解的两种配置方式。

## XML模式

定义业务实现类`xxxServiceImpl.java`，定义切面类`xxxAspect.java`，编写XML配置文件`xxx.xml`。将业务实现类和切面类注册为bean，然后配置切面（切面中包含切入点和通知类型）。

```xml
<beans>
    <context:component-scan base-package="tech.pdai.springframework" />

    <aop:aspectj-autoproxy/>

    <!-- 目标类 -->
    <bean id="" class="">
        <!-- configure properties of bean here as normal -->
    </bean>

    <!-- 切面 -->
    <bean id="" class="">
        <!-- configure properties of aspect here as normal -->
    </bean>

    <aop:config>
        <!-- 配置切面 -->
        <aop:aspect ref="">
            <!-- 配置切入点 -->
            <aop:pointcut id="pointCutMethod" expression="execution(* xxx.service.*.*(..))"/>
            <!-- 环绕通知 -->
            <aop:around method="doAround" pointcut-ref="pointCutMethod"/>
            <!-- 前置通知 -->
            <aop:before method="doBefore" pointcut-ref="pointCutMethod"/>
            <!-- 后置通知；returning属性：用于设置后置通知的第二个参数的名称，类型是Object -->
            <aop:after-returning method="doAfterReturning" pointcut-ref="pointCutMethod" returning="result"/>
            <!-- 异常通知：如果没有异常，将不会执行增强；throwing属性：用于设置通知第二个参数的的名称、类型-->
            <aop:after-throwing method="doAfterThrowing" pointcut-ref="pointCutMethod" throwing="e"/>
            <!-- 最终通知 -->
            <aop:after method="doAfter" pointcut-ref="pointCutMethod"/>
        </aop:aspect>
    </aop:config>
</beans>
```

## Spring AOP注解

| 注解名称        | 解释                                                         |
| --------------- | ------------------------------------------------------------ |
| @Aspect         | 用来定义一个切面。                                           |
| @pointcut       | 用于定义切入点表达式。在使用时还需要定义一个包含名字和任意参数的方法签名来表示切入点名称，这个方法签名就是一个返回值为void，且方法体为空的普通方法。 |
| @Before         | 用于定义前置通知，相当于BeforeAdvice。在使用时，通常需要指定一个value属性值，该属性值用于指定一个切入点表达式(可以是已有的切入点，也可以直接定义切入点表达式)。 |
| @AfterReturning | 用于定义后置通知，相当于AfterReturningAdvice。在使用时可以指定pointcut / value和returning属性，其中pointcut / value这两个属性的作用一样，都用于指定切入点表达式。 |
| @Around         | 用于定义环绕通知，相当于MethodInterceptor。在使用时需要指定一个value属性，该属性用于指定该通知被植入的切入点。 |
| @After-Throwing | 用于定义异常通知来处理程序中未处理的异常，相当于ThrowAdvice。在使用时可指定pointcut / value和throwing属性。其中pointcut/value用于指定切入点表达式，而throwing属性值用于指定-一个形参名来表示Advice方法中可定义与此同名的形参，该形参可用于访问目标方法抛出的异常。 |
| @After          | 用于定义最终final 通知，不管是否异常，该通知都会执行。使用时需要指定一个value属性，该属性用于指定该通知被植入的切入点。 |
| @DeclareParents | 用于定义引介通知，相当于IntroductionInterceptor。            |

## 定义日志注解

首先定义一个日志注解，该注解用来标记需要被切面处理的方法上。

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

## 定义切面

```java
@EnableAspectJAutoProxy
@Aspect
@Component
public class LoggingAspect {
    @Pointcut("@annotation(top.chc.aspectdemo.aspect.Loggable)")
    public void loggable() {
    }

    @Before("loggable()")
    public void beforeAdvice() {
        System.out.println("Before the method is called");
    }
}
```

其中**切入点**是注解`Loggable`，然后通过`@Before @After`等注解定义**通知**。

## 放置注解

将注解放置到业务代码中

```java
@Service
public class UserServiceImpl {
    @Loggable
    public void getUser(String name){
        System.out.printf("User name: %s\n", name);
    }
}
```

这种方式是侵入式的，会在业务代码中添加了`@Loggable`注解。

可以在定义切面的时候，直接将切入点定义到具体的业务类方法上，而不是注解。

```java
@EnableAspectJAutoProxy
@Aspect
@Component
public class LogAspect {
    private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);

    // 定义切点（这里以controller包下的所有方法为例）
    @Pointcut("execution(* com.example.controller..*.*(..))")
    public void logPointcut() {}

    // 前置通知：在方法执行前记录日志
    @Before("logPointcut()")
    public void logBefore(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        Object[] args = joinPoint.getArgs();
        
        logger.info("方法开始执行: {}.{}", className, methodName);
        logger.info("入参: {}", args);
    }

    // 后置通知：在方法执行后记录日志
    @AfterReturning(pointcut = "logPointcut()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        logger.info("方法执行结束: {}.{}", className, methodName);
        logger.info("返回结果: {}", result);
    }

    // 异常通知：在方法抛出异常时记录日志
    @AfterThrowing(pointcut = "logPointcut()", throwing = "exception")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable exception) {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        logger.error("方法执行异常: {}.{}", className, methodName);
        logger.error("异常信息: ", exception);
    }

    // 环绕通知：可以控制方法执行的整个过程
    @Around("logPointcut()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        // 执行目标方法
        Object result = joinPoint.proceed();
        
        long timeCost = System.currentTimeMillis() - startTime;
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        logger.info("方法: {}.{} 执行耗时: {}ms", className, methodName, timeCost);
        
        return result;
    }
}
```

`@Pointcut`：定义切入点，可以使用不同的表达式

- `execution(* com.example..*.*(..))`：匹配指定包下的所有方法
- `within(com.example.controller.*)`：匹配指定包下的所有类
- `@annotation(com.example.annotation.Log)`：匹配带有特定注解的方法

```java
// 任意公共方法的执行：
execution（public * *（..））

// 任何一个名字以“set”开始的方法的执行：
execution（* set*（..））

// AccountService接口定义的任意方法的执行：
execution（* com.xyz.service.AccountService.*（..））

// 在service包中定义的任意方法的执行：
execution（* com.xyz.service.*.*（..））

// 在service包或其子包中定义的任意方法的执行：
execution（* com.xyz.service..*.*（..））

// 在service包中的任意连接点（在Spring AOP中只是方法执行）：
within（com.xyz.service.*）

// 在service包或其子包中的任意连接点（在Spring AOP中只是方法执行）：
within（com.xyz.service..*）

// 实现了AccountService接口的代理对象的任意连接点 （在Spring AOP中只是方法执行）：
this（com.xyz.service.AccountService）// 'this'在绑定表单中更加常用

// 实现AccountService接口的目标对象的任意连接点 （在Spring AOP中只是方法执行）：
target（com.xyz.service.AccountService） // 'target'在绑定表单中更加常用

// 任何一个只接受一个参数，并且运行时所传入的参数是Serializable 接口的连接点（在Spring AOP中只是方法执行）
args（java.io.Serializable） // 'args'在绑定表单中更加常用; 请注意在例子中给出的切入点不同于 execution(* *(java.io.Serializable))： args版本只有在动态运行时候传入参数是Serializable时才匹配，而execution版本在方法签名中声明只有一个 Serializable类型的参数时候匹配。

// 目标对象中有一个 @Transactional 注解的任意连接点 （在Spring AOP中只是方法执行）
@target（org.springframework.transaction.annotation.Transactional）// '@target'在绑定表单中更加常用

// 任何一个目标对象声明的类型有一个 @Transactional 注解的连接点 （在Spring AOP中只是方法执行）：
@within（org.springframework.transaction.annotation.Transactional） // '@within'在绑定表单中更加常用

// 任何一个执行的方法有一个 @Transactional 注解的连接点 （在Spring AOP中只是方法执行）
@annotation（org.springframework.transaction.annotation.Transactional） // '@annotation'在绑定表单中更加常用

// 任何一个只接受一个参数，并且运行时所传入的参数类型具有@Classified 注解的连接点（在Spring AOP中只是方法执行）
@args（com.xyz.security.Classified） // '@args'在绑定表单中更加常用

// 任何一个在名为'tradeService'的Spring bean之上的连接点 （在Spring AOP中只是方法执行）
bean（tradeService）

// 任何一个在名字匹配通配符表达式'*Service'的Spring bean之上的连接点 （在Spring AOP中只是方法执行）
bean（*Service）
    
// &&：要求连接点同时匹配两个切入点表达式
// ||：要求连接点匹配任意个切入点表达式
// !:：要求连接点不匹配指定的切入点表达式
```

总结：

- Spring AOP比完全使用AspectJ更加简单， 因为它不需要引入AspectJ的编译器/织入器到你开发和构建过程中。 如果你**仅仅需要在Spring bean上通知执行操作，那么Spring AOP是合适的选择**。
- 如果你需要通知domain对象或其它没有在Spring容器中管理的任意对象，那么你需要使用AspectJ。
- 如果你想通知除了简单的方法执行之外的连接点（调用连接点、字段get或set的连接点等等）， 也需要使用AspectJ。

