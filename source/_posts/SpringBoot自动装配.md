---
title: SpringBoot自动装配原理
date: 2024-03-02
updated : 2024-03-02
categories:
- SpringBoot
tags: 
  - SpringBoot
  - 原理
description: "深入解析 SpringBoot 自动装配原理，从 @SpringBootApplication 到 spring.factories 的完整链路。"
series: SpringBoot 深度解析
series_order: 1
---

## 从 @SpringBootApplication 说起

每一个 SpringBoot 应用的入口都离不开如下代码：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@SpringBootApplication` 看似简单，实际上是一个**组合注解**。打开其源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
    }
)
public @interface SpringBootApplication {
    // ...
}
```

它由三个核心注解组成：

1. **@SpringBootConfiguration** — 本质上是 `@Configuration`，表明当前类是配置类，Spring 容器会识别并处理其中定义的 `@Bean` 方法。
2. **@EnableAutoConfiguration** — 自动装配的核心入口，负责触发自动配置类的加载与过滤。
3. **@ComponentScan** — 组件扫描，默认扫描当前包及其子包下的 `@Component`、`@Service`、`@Repository` 等注解标记的类。同时通过 `excludeFilters` 排除了 `TypeExcludeFilter` 和 `AutoConfigurationExcludeFilter`。

理解这三个注解的分工，就理解了 SpringBoot 启动的第一步：**先扫描用户自定义的 Bean，再触发自动配置的加载**。

## @EnableAutoConfiguration 工作流程

`@EnableAutoConfiguration` 是自动装配的触发器，其源码如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

关键点：

- **@AutoConfigurationPackage** — 将主配置类所在的包作为自动配置的根包注册到 `AutoConfigurationPackages` 中。
- **@Import(AutoConfigurationImportSelector.class)** — 通过 `@Import` 导入 `AutoConfigurationImportSelector`，这是自动装配的核心执行者。

### 执行时序

1. Spring 在解析 `@EnableAutoConfiguration` 注解时，触发 `AutoConfigurationImportSelector` 的加载。
2. `AutoConfigurationImportSelector` 实现 `DeferredImportSelector` 接口，这意味着它的执行时机晚于普通的 `ImportSelector`，确保所有用户自定义的配置先被处理。
3. 在 `selectImports()` 方法中，通过 SpringFactoriesLoader 加载所有候选自动配置类。
4. 经过条件过滤后，将符合条件的自动配置类返回给 Spring 容器进行注册。

## AutoConfigurationImportSelector 源码分析

`AutoConfigurationImportSelector` 是整个自动装配流程的核心调度器。下面逐步分析其关键方法。

### selectImports 方法

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationEntry autoConfigurationEntry =
        getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(
        autoConfigurationEntry.getConfigurations());
}
```

`isEnabled()` 方法会检查 `spring.boot.enableautoconfiguration` 配置项，默认为 `true`。可以通过设置该属性为 `false` 来关闭自动装配。

### getAutoConfigurationEntry 方法

这是核心入口方法：

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(
        AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 1. 获取注解属性（exclude、excludeName）
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 2. 从 spring.factories 加载候选配置类
    List<String> configurations = getCandidateConfigurations(
        annotationMetadata, attributes);
    // 3. 去重
    configurations = removeDuplicates(configurations);
    // 4. 获取需要排除的配置类
    Set<String> exclusions = getExclusions(
        annotationMetadata, attributes);
    // 5. 校验排除类
    checkExcludedClasses(configurations, exclusions);
    // 6. 移除排除类
    configurations.removeAll(exclusions);
    // 7. 条件过滤
    configurations = getConfigurationClassFilter().filter(configurations);
    // 8. 触发自动配置导入事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

### getCandidateConfigurations 方法

```java
protected List<String> getCandidateConfigurations(
        AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = new ArrayList<>(
        SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
        .forEach(configurations::add);
    Assert.notEmpty(configurations,
        "No auto configuration classes found in META-INF/spring.factories "
        + "nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports.");
    return configurations;
}
```

从这里可以看到，SpringBoot 2.7+ 同时支持两种加载方式：

1. **SpringFactoriesLoader** — 从 `META-INF/spring.factories` 中加载。
2. **ImportCandidates** — 从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中加载（SpringBoot 2.7 新增，SpringBoot 3.0 成为主要方式）。

## spring.factories 与 AutoConfiguration Imports

### 传统方式：spring.factories

在 SpringBoot 2.7 之前，自动配置类的声明统一放在 jar 包的 `META-INF/spring.factories` 文件中：

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.autoconfigure.MyServiceAutoConfiguration,\
  com.example.autoconfigure.MyRepositoryAutoConfiguration
```

这个机制基于 Java 的 **SPI（Service Provider Interface）** 思想，通过 `SpringFactoriesLoader` 工具类读取所有 jar 包下的 `spring.factories` 文件，合并为完整列表。

### 新方式：AutoConfiguration.imports

从 SpringBoot 2.7 开始，引入了新的配置文件路径：

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

每行一个全限定类名：

```
com.example.autoconfigure.MyServiceAutoConfiguration
com.example.autoconfigure.MyRepositoryAutoConfiguration
```

SpringBoot 3.0 中，`spring.factories` 中的自动配置声明已被**完全废弃**，仅保留 `AutoConfiguration.imports` 方式。

### spring-autoconfigure-metadata

除了配置类列表，SpringBoot 还使用 `spring-autoconfigure-metadata.properties` 文件来优化自动配置的加载效率：

```properties
com.example.autoconfigure.MyServiceAutoConfiguration.ConditionalOnClass=com.example.MyService
com.example.autoconfigure.MyServiceAutoConfiguration.AutoConfigureAfter=com.example.autoconfigure.CoreAutoConfiguration
```

这些元数据允许 SpringBoot 在**不实际加载配置类**的情况下进行条件预筛选，显著提升启动性能。

## AutoConfigurationPackages 机制

`@AutoConfigurationPackage` 注解的作用是将主配置类所在的包注册到一个静态 holder 中：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    // ...
}
```

`Registrar` 实现了 `ImportBeanDefinitionRegistrar`，在容器启动时将主类所在的包名注册为一个 Bean：

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
        register(registry, new PackageImports(metadata)
            .getPackageNames().toArray(new String[0]));
    }
}
```

这个机制让其他自动配置类可以通过 `AutoConfigurationPackages.get()` 获取到应用的基础包路径，用于 JPA Entity 扫描、MyBatis Mapper 扫描等场景。

## 条件过滤链

从 `spring.factories` 加载到的候选配置类通常有上百个，但并非所有都会生效。SpringBoot 通过一套**条件过滤机制**进行筛选。

### 过滤流程

`getConfigurationClassFilter().filter(configurations)` 方法触发的过滤链如下：

```java
private ConfigurationClassFilter getConfigurationClassFilter() {
    if (this.configurationClassFilter == null) {
        List<AutoConfigurationImportFilter> filters =
            SpringFactoriesLoader.loadFactories(
                AutoConfigurationImportFilter.class, this.beanClassLoader);
        this.configurationClassFilter =
            new ConfigurationClassFilter(this.beanClassLoader, filters);
    }
    return this.configurationClassFilter;
}
```

### 核心过滤器

SpringBoot 内置了三个关键的 `AutoConfigurationImportFilter` 实现：

#### 1. OnClassCondition

对应 `@ConditionalOnClass` 和 `@ConditionalOnMissingClass`。

检查目标类是否存在于 classpath 中。这是**最常用的过滤条件**，因为大多数自动配置都依赖于特定的第三方库。

```java
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {
    // 只有 classpath 中存在 DataSource 类时才会加载
}
```

SpringBoot 通过 `spring-autoconfigure-metadata.properties` 中的元数据预判 `OnClassCondition`，避免提前实例化配置类。

#### 2. OnBeanCondition

对应 `@ConditionalOnBean`、`@ConditionalOnMissingBean` 和 `@ConditionalOnSingleCandidate`。

检查 Spring 容器中是否已存在特定类型的 Bean。`@ConditionalOnMissingBean` 是**用户覆盖默认配置**的核心机制——当用户自定义了某个 Bean 后，自动配置中的默认实现就不会再创建。

```java
@Bean
@ConditionalOnMissingBean
public MyService myService() {
    return new DefaultMyService();
}
```

#### 3. OnWebApplicationCondition

对应 `@ConditionalOnWebApplication` 和 `@ConditionalOnNotWebApplication`。

检查当前应用是否为 Web 应用，以及使用的是哪种 Web 容器（Servlet 或 Reactive）。

### 过滤的执行顺序

```
候选配置类（约 130+）
       |
       v
  OnClassCondition 过滤
  （剔除 classpath 缺少依赖的配置类）
       |
       v
  OnBeanCondition 过滤
  （剔除容器中已存在对应 Bean 的配置类）
       |
       v
  OnWebApplicationCondition 过滤
  （剔除不匹配当前应用类型的配置类）
       |
       v
  最终生效的配置类（约 20-30 个）
```

## 自动装配的加载时序

自动配置类之间可能存在依赖关系，SpringBoot 提供了三种排序注解来控制加载顺序：

### @AutoConfigureBefore / @AutoConfigureAfter

```java
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfigureBefore(HibernateJpaAutoConfiguration.class)
public class MyJpaAutoConfiguration {
    // ...
}
```

### @AutoConfigureOrder

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class CoreAutoConfiguration {
    // ...
}
```

### 完整时序图

```
SpringApplication.run()
       |
       v
  createApplicationContext()
       |
       v
  prepareContext()
       |
       v
  refreshContext()
       |
       v
  invokeBeanFactoryPostProcessors()
       |
       v
  处理 @Configuration 类
       |
       v
  解析 @EnableAutoConfiguration
       |
       v
  AutoConfigurationImportSelector.selectImports()
       |
       v
  getCandidateConfigurations()
  （从 spring.factories / AutoConfiguration.imports 加载）
       |
       v
  去重 + 排除
       |
       v
  条件过滤（OnClass -> OnBean -> OnWebApplication）
       |
       v
  排序（@AutoConfigureOrder / Before / After）
       |
       v
  注册到 BeanDefinitionRegistry
       |
       v
  实例化 Bean（lazy / eager 取决于作用域）
```

## 如何自定义自动装配

### 创建自定义自动配置类

```java
@AutoConfiguration
@ConditionalOnClass(MyFeature.class)
@EnableConfigurationProperties(MyFeatureProperties.class)
public class MyFeatureAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyFeatureService myFeatureService(MyFeatureProperties properties) {
        return new MyFeatureService(properties);
    }
}
```

### 注册自动配置

**方式一：spring.factories（SpringBoot 2.x）**

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyFeatureAutoConfiguration
```

**方式二：AutoConfiguration.imports（SpringBoot 2.7+ / 3.x）**

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyFeatureAutoConfiguration
```

### 添加条件元数据（可选但推荐）

```properties
# META-INF/spring-autoconfigure-metadata.properties
com.example.MyFeatureAutoConfiguration.ConditionalOnClass=com.example.MyFeature
com.example.MyFeatureAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 通过 application.yml 控制

用户可以通过配置文件禁用特定的自动配置：

```yaml
spring:
  autoconfigure:
    exclude:
      - com.example.MyFeatureAutoConfiguration
```

或在主类上通过注解排除：

```java
@SpringBootApplication(exclude = {MyFeatureAutoConfiguration.class})
```

也可以使用 `@ConditionalOnProperty` 让用户通过配置开关控制：

```java
@AutoConfiguration
@ConditionalOnProperty(
    prefix = "my.feature",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true
)
public class MyFeatureAutoConfiguration {
    // ...
}
```

## 总结

SpringBoot 自动装配的核心链路可以概括为：

1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`。
2. `@EnableAutoConfiguration` 通过 `@Import` 导入 `AutoConfigurationImportSelector`。
3. `AutoConfigurationImportSelector` 从 `spring.factories` 或 `AutoConfiguration.imports` 加载所有候选配置类。
4. 经过去重、排除、条件过滤、排序后，将最终符合条件的配置类注册到容器。
5. 容器在 refresh 阶段实例化这些 Bean，完成自动装配。

理解这条链路，就掌握了 SpringBoot"约定优于配置"的底层实现。在实际开发中，无论是排查自动配置问题，还是开发自定义 Starter，都需要回到这条链路上定位关键节点。
