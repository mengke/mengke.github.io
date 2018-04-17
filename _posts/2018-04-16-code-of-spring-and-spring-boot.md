---
layout: post
title: "Code of Spring and Spring Boot"
category: Spring
tags: ["spring"]
---

基于Spring Boot 2.0.x和Spring 5.0.x

## Spring Boot启动流程

1. 检测Spring应用类型(Reactive-Web, Non-Web以及Web-Servlet)
2. 通过spring.factories文件加载并实例化注册的`org.springframework.context.ApplicationContextInitializer`s
3. 通过spring.factories文件加载并实例化注册的`org.springframework.context.ApplicationListener`s
4. 确定Spring Boot工程的Main Class
5. 通过spring.factories文件加载并实例化注册的`org.springframework.boot.SpringApplicationRunListener`, 并分发`ApplicationStartingEvent`事件给`ApplicationListener`s
6. 根据应用类型创建`org.springframework.core.env.ConfigurableEnvironment`实例, 分发`ApplicationEnvironmentPreparedEvent`事件给`ApplicationListener`s
7. 创建`org.springframework.context.ConfigurableApplicationContext`实例
8. Apply `ApplicationContextInitializer`s
9. `SpringApplicationRunListener`s#contextPrepared(空方法)
10. load(annotated main class)
11. 分发`ApplicationPreparedEvent`事件给`ApplicationListener`s
12. context.refresh
13. 由ApplicationContext分发`ApplicationStartedEvent`事件
14. 由ApplicationContext分发分发`ApplicationReadyEvent`事件
15. 如果应用在启动过程中有错误发生, 则分发`ApplicationFailedEvent`事件给`ApplicationListener`s

## Spring Boot预置的spring.factories

Spring Boot通过`org.springframework.core.io.support.SpringFactoriesLoader#loadFactories(Class clazz, ClassLoader loader)`来获取某个类别下的实例

### ApplicationContextInitializer

* spring-boot
    + `org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer`
    + `org.springframework.boot.context.ContextIdApplicationContextInitializer`
    + `org.springframework.boot.context.config.DelegatingApplicationContextInitializer`
    + `org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer`
- - -
* spring-boot-autoconfigure
    + `org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer`
    + `org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener`
- - -
* 如果引入了spring-boot-devtools, 还包括以下`ApplicationContextInitializer`
    + `org.springframework.boot.devtools.restart.RestartScopeInitializer`

### ApplicationListener

* spring-boot
    + `org.springframework.boot.ClearCachesApplicationListener`
    + `org.springframework.boot.builder.ParentContextCloserApplicationListener`
    + `org.springframework.boot.context.FileEncodingApplicationListener`
    + `org.springframework.boot.context.config.AnsiOutputApplicationListener`
    + `org.springframework.boot.context.config.ConfigFileApplicationListener`
    + `org.springframework.boot.context.config.DelegatingApplicationListener`
    + `org.springframework.boot.context.logging.ClasspathLoggingApplicationListener`
    + `org.springframework.boot.context.logging.LoggingApplicationListener`
    + `org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener`
- - -
* spring-boot-autoconfigure
    + `org.springframework.boot.autoconfigure.BackgroundPreinitializer`
- - -
* 如果引入了spring-boot-devtools, 还包括以下`ApplicationListener`
    + `org.springframework.boot.devtools.restart.RestartApplicationListener`

### SpringApplicationRunListener
* `org.springframework.boot.context.event.EventPublishingRunListener`


## Spring Boot启动过程中部分重要的事件以及处理过程

### `ApplicationStartingEvent`

* `LoggingApplicationListener`: 构建日志系统
* `BackgroundPreinitializer`: 后台启动一个线程, 用来预先初始化一些服务
* `RestartApplicationListener`: 初始化Restarter

### `ApplicationEnvironmentPreparedEvent`

* `ConfigFileApplicationListener`: 载入注册于spring.factories的`org.springframework.boot.env.EnvironmentPostProcessor`s, 主要用于在`context.refresh`之前自定义应用的`Environment`
* `DelegatingApplicationListener`: 在本阶段(EnvironmentPrepared)载入注册于`context.listener.classes`属性下的`ApplicationListener`s, 并将后续事件分发给他们
* `LoggingApplicationListener`: 初始化日志系统, 并注册Shutdown Hook
* `ConfigFileApplicationListener`: 将默认属性优先级将为最低

### `ApplicationPreparedEvent`(contextLoaded)
* `LoggingApplicationListener`: 将日志系统注册到`BeanFactory`

### `ApplicationStartedEvent`(contextRefreshed)

### `ApplicationReadyEvent`(main class run)

* `BackgroundPreinitializer`: 如果预载服务的线程任务没有结束, 阻塞主线程

## `ApplicationContextInitializer`初始化过程

* `ConfigurationWarningsApplicationContextInitializer`: 对有问题的`BeanDefinitionRegistry`进行提示
* `ContextIdApplicationContextInitializer`: 创建一个ContextId, 并将其注册到`BeanFactory`
* `DelegatingApplicationContextInitializer`: 初始化其他注册在`context.initializer.classes`的`ApplicationContextInitializer`
* `ServerPortInfoApplicationContextInitializer`: 将自身作为一个`ApplicationListener`注册到`ApplicationContext`, 用于监听`WebServerInitializedEvent`事件
* `SharedMetadataReaderFactoryContextInitializer`: 创建一个`ConfigurationClassPostProcessor`(用于处理标注`@Configuration`的类)和Spring Boot共享的一个MetadataReader, 用于读取类的元信息
- - -
* `ConditionEvaluationReportLoggingListener`: 注册一个`ApplicationListener`用于在contextRefreshed的时候打印Auto Configuration的信息
* `RestartScopeInitializer`: 注册一个`RestartScope`到`BeanFactory`

## ApplicationContext.refresh

