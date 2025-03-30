---
title: SpringBoot Bean加载
category: springboot
---

> 了解Java注解原理后，可知除了内置注解(@Override等)，其他的通过元注解编写的注解需要自行编写注解处理器并让JVM加载这些处理器以用于解析注解。
>
> 目前为止，SpringBoot中使用了大量的注解来简化开发，如启动类的`@SpringBootApplication`，自定义bean的`@Configuration` `@bean`这些自定义注解都需要SpringBoot框架去解析。本文将通过SpringBoot加载Bean来讲解注解的处理。

# SpringBoot启动时Bean的加载过程

```
|--LearnApplication //启动类
|----SpringApplication.run()
|------refreshContext(context);
|--------applicationContext.refresh(); //此处以ServletWebServerApplicationContext为例子
|----------AbstractApplicationContext.refresh() //实际调用的是该抽象类的方法
|------------invokeBeanFactoryPostProcessors(beanFactory); //本方法是扫描注解加载Bean的核心方法
|--------------PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```

通过对SpringBoot启动流程的分析，扫描注解加载Bean类的核心方法是`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());`

# `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`详解

> 这个方法写的是真垃圾，看它在注解中直接加入issue链接就知道它写的多乱了。

该方法执行 `BeanFactoryPostProcessor` 及其子接口 `BeanDefinitionRegistryPostProcessor` 的关键方法。

* **执行 `BeanDefinitionRegistryPostProcessor`**
  - 先执行 `PriorityOrdered` 的 `BeanDefinitionRegistryPostProcessor`（按优先级排序）
  - 再执行 `Ordered` 的 `BeanDefinitionRegistryPostProcessor`
  - 最后执行普通的 `BeanDefinitionRegistryPostProcessor`

* **执行 `BeanFactoryPostProcessor`**

  - 先执行 `PriorityOrdered` 的 `BeanFactoryPostProcessor`

  - 再执行 `Ordered` 的 `BeanFactoryPostProcessor`

  - 最后执行普通的 `BeanFactoryPostProcessor`

*注：在 Spring 中，`BeanFactoryPostProcessor` 用于 **在 Bean 实例化之前** 修改 **BeanDefinition**，而 `BeanDefinitionRegistryPostProcessor` 则 **可以额外注册新的 BeanDefinition**。*

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

    // 如果 BeanFactory 是 BeanDefinitionRegistry，则执行 BeanDefinitionRegistryPostProcessor
    if (beanFactory instanceof BeanDefinitionRegistry registry) {
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
		
         // 先处理方法参数传入的 BeanFactoryPostProcessors（非 Spring 扫描的）
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor registryProcessor) {
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // 处理BeanDefinitionRegistryPostProcessor
        // 这里会处理内置于SpringBoot的ConfigurationClassPostProcessor
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 首先处理实现PriorityOrdered的BeanDefinitionRegistryPostProcessors
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        //排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        //调用postProcessBeanDefinitionRegistry方法
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        // 首先处理实现 Ordered 的BeanDefinitionRegistryPostProcessors
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        //调用postProcessBeanDefinitionRegistry方法
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();

        // 最后 执行普通的 BeanDefinitionRegistryPostProcessor
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
        // 调用 BeanDefinitionRegistryPostProcessor 和 BeanFactoryPostProcessor 的postProcessBeanFactory()方法
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // 如果 BeanFactory 不是 BeanDefinitionRegistry，则直接执行普通的 BeanFactoryPostProcessor 的postProcessBeanFactory()方法
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 处理 Spring 容器注册的 BeanFactoryPostProcessor
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

# `ConfigurationClassPostProcessor`

>`ConfigurationClassPostProcessor` 是 **Spring 内部关键的 `BeanFactoryPostProcessor`**，它的主要作用是 **解析 `@Configuration`、`@ComponentScan`、`@Import`、`@Bean` 等注解**，并将解析后的 **BeanDefinition** 注册到 `BeanFactory` 中。

Spring 在 `invokeBeanFactoryPostProcessors()` 时执行所有 `BeanFactoryPostProcessor`，其中 `ConfigurationClassPostProcessor` 是 **Spring 内置的 `BeanFactoryPostProcessor`，优先执行**。

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
```

通过`beanFactory.getBeanNamesForType()`来获取`BeanDefinitionRegistryPostProcessor.class`的后处理类。调试后会得到`postProcessorNames=org.springframework.context.annotation.internalConfigurationAnnotationProcessor`然后将这个名字传入`beanFactory.getBean()`并获取它的bean类型，得到的bean类就是`ConfigurationClassPostProcessor`。后续会调用该类的`postProcessBeanDefinitionRegistry()`方法

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);

    processConfigBeanDefinitions(registry);
}

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    // 获取所有注册的bean类型
    String[] candidateNames = registry.getBeanDefinitionNames();
	//遍历所有的bean类，寻找有@Configuration注解的类
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        // 如果bean类中包含@Configuration注解则将该类加入到configCandidates List中
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }

    // 根据@Order排序
    configCandidates.sort((bd1, bd2) -> {
        int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
        int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
        return Integer.compare(i1, i2);
    });

    // Detect any custom bean name generation strategy supplied through the enclosing application context
    SingletonBeanRegistry singletonRegistry = null;
    if (registry instanceof SingletonBeanRegistry sbr) {
        singletonRegistry = sbr;
        if (!this.localBeanNameGeneratorSet) {
            BeanNameGenerator generator = (BeanNameGenerator) singletonRegistry.getSingleton(
                    AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
            if (generator != null) {
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }
    }

    if (this.environment == null) {
        this.environment = new StandardEnvironment();
    }

    // Parse each @Configuration class
    // 通过ConfigurationClassParser解析@Configuration的类
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = CollectionUtils.newHashSet(configCandidates.size());
    do {
        StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);
        processConfig.tag("classCount", () -> String.valueOf(configClasses.size())).end();

        candidates.clear();
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = Set.of(candidateNames);
            Set<String> alreadyParsedClasses = CollectionUtils.newHashSet(alreadyParsed.size());
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (singletonRegistry != null && !singletonRegistry.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
        singletonRegistry.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    // Store the PropertySourceDescriptors to contribute them Ahead-of-time if necessary
    this.propertySourceDescriptors = parser.getPropertySourceDescriptors();

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory cachingMetadataReaderFactory) {
        // Clear cache in externally provided MetadataReaderFactory; this is a no-op
        // for a shared cache since it'll be cleared by the ApplicationContext.
        cachingMetadataReaderFactory.clearCache();
    }
}
```

