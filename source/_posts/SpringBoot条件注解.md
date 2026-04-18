---
title: SpringBoot条件注解详解
date: 2024-03-15
updated : 2024-03-15
categories:
- SpringBoot
tags: 
  - SpringBoot
  - 原理
description: 全面解析 SpringBoot 条件注解体系，掌握按条件注册 Bean 的核心机制。
series: SpringBoot 深度解析
series_order: 2
---

## @Conditional 根注解原理

SpringBoot 条件注解体系的根基是 Spring Framework 4.0 引入的 `@Conditional` 注解：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Conditional {
    Class<? extends Condition>[] value();
}
```

它接受一个 `Condition` 实现类数组，在 Bean 注册时通过 `Condition.matches()` 方法判断是否满足条件：

```java
@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

`ConditionContext` 提供了对 Spring 容器的完整访问能力，包括：

- **BeanDefinitionRegistry** — 注册表，用于查询已注册的 Bean 定义。
- **ConfigurableListableBeanFactory** — Bean 工厂，用于查询已存在的 Bean 实例。
- **Environment** — 环境变量和配置属性。
- **ResourceLoader** — 资源加载器。
- **ClassLoader** — 类加载器。

SpringBoot 在此基础上封装了一套**开箱即用的条件注解**，分为以下几大类别。

## Class Conditions

### @ConditionalOnClass

当指定的类存在于 classpath 时，才加载当前配置类或注册当前 Bean。

```java
@Configuration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

**原理**：`OnClassCondition` 通过 `ClassLoader` 尝试加载指定类，如果抛出 `ClassNotFoundException` 则条件不满足。底层使用 `ClassUtils.isPresent()` 方法：

```java
protected ConditionOutcome getOutcome(String[] autoConfigurationClasses,
        AutoConfigurationMetadata autoConfigurationMetadata) {
    // 从 spring-autoconfigure-metadata 中读取类条件
    // 避免提前加载配置类本身
}
```

### @ConditionalOnMissingClass

与 `@ConditionalOnClass` 相反，当指定的类**不存在**于 classpath 时才生效。

```java
@Configuration
@ConditionalOnMissingClass("com.fasterxml.jackson.databind.ObjectMapper")
public class GsonAutoConfiguration {
    // 当 Jackson 不存在时，使用 Gson 作为 JSON 序列化方案
}
```

### 注意事项

- `value` 属性接受 Class 数组，`name` 属性接受全限定类名字符串数组。
- 当类路径中可能不存在目标类时，应使用 `name` 属性避免 `ClassNotFoundException`：
  ```java
  @ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver")
  ```
- 多个类之间是 **AND** 关系，即所有类都必须满足条件。

## Bean Conditions

### @ConditionalOnBean

当 Spring 容器中已存在指定类型的 Bean 时才生效。

```java
@Bean
@ConditionalOnBean(DataSource.class)
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}
```

属性说明：

| 属性 | 说明 |
|------|------|
| `value` / `type` | 指定 Bean 的类型 |
| `name` | 指定 Bean 的名称 |
| `annotation` | 指定 Bean 上的注解 |
| `search` | 搜索策略：`CURRENT`（当前容器）、`ANCESTORS`（父容器）、`ALL`（所有容器） |

### @ConditionalOnMissingBean

当 Spring 容器中**不存在**指定类型的 Bean 时才生效。这是 SpringBoot 自动配置中**最常用的条件注解**，也是"用户优先"原则的核心实现。

```java
@Configuration
public class HttpMessageConvertersAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public HttpMessageConverters messageConverters(
            ObjectProvider<HttpMessageConverter<?>> converters) {
        return new HttpMessageConverters(converters.orderedStream()
            .collect(Collectors.toList()));
    }
}
```

**设计意图**：自动配置提供默认实现，但允许用户通过自定义 Bean 覆盖默认行为。这被称为 **"Back Off"** 机制。

### @ConditionalOnSingleCandidate

当容器中存在且仅有一个指定类型的 Bean，或者存在多个但有且仅有一个标记为 `@Primary` 时才生效。

```java
@Bean
@ConditionalOnSingleCandidate(DataSource.class)
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    factory.setDataSource(dataSource);
    return factory.getObject();
}
```

### Bean Conditions 的匹配陷阱

使用 Bean Conditions 时需要注意** Bean 注册顺序**的问题：

```java
// 错误示例：Bean A 依赖 Bean B 的存在性，但 B 的配置可能在 A 之后处理
@Configuration
public class ConfigA {
    @Bean
    @ConditionalOnMissingBean(ServiceB.class)
    public ServiceB serviceB() {
        return new DefaultServiceB();
    }
}

@Configuration
public class ConfigB {
    @Bean
    public ServiceB customServiceB() {
        return new CustomServiceB();
    }
}
```

为了避免这种顺序问题，建议：

1. 将 `@ConditionalOnBean` / `@ConditionalOnMissingBean` 尽量用在 `@AutoConfiguration` 类上，而非普通 `@Configuration` 类。
2. 使用 `@AutoConfigureBefore` / `@AutoConfigureAfter` 明确指定配置类之间的顺序。
3. 在自动配置元数据中声明条件信息，便于预过滤。

## Property Conditions

### @ConditionalOnProperty

根据 Spring Environment 中的属性值来决定是否生效。

```java
@Bean
@ConditionalOnProperty(
    prefix = "app.cache",
    name = "type",
    havingValue = "redis",
    matchIfMissing = false
)
public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
    return RedisCacheManager.create(factory);
}
```

属性说明：

| 属性 | 说明 |
|------|------|
| `prefix` | 属性前缀 |
| `name` | 属性名（可与 prefix 组合） |
| `havingValue` | 期望的属性值 |
| `matchIfMissing` | 当属性不存在时是否匹配，默认 `false` |

### 匹配规则

1. 如果仅指定 `name`，属性存在即匹配（值不为 `"false"`、`"off"`、`"0"`）。
2. 如果指定 `havingValue`，属性值必须等于 `havingValue` 才匹配。
3. 如果 `matchIfMissing = true`，属性不存在时也算匹配。

```java
// 属性存在且值不为 false/off/0 即可
@ConditionalOnProperty("app.feature.enabled")

// 属性值必须等于 true
@ConditionalOnProperty(
    prefix = "app.feature",
    name = "enabled",
    havingValue = "true"
)

// 属性不存在时默认启用
@ConditionalOnProperty(
    prefix = "app.feature",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true
)
```

### 多属性组合

```java
@ConditionalOnProperty(
    prefix = "app.security",
    name = {"enabled", "auth-type"}
)
// 当 app.security.enabled 和 app.security.auth-type 都存在时才匹配
```

## Resource Conditions

### @ConditionalOnResource

当指定的资源文件存在于 classpath 中时才生效。

```java
@Configuration
@ConditionalOnResource(resources = "classpath:hibernate.properties")
public class HibernateConfig {
    // 只有 classpath 下存在 hibernate.properties 时才加载
}
```

`resources` 属性支持以下前缀：

- `classpath:` — 类路径资源
- `file:` — 文件系统资源
- `url:` — URL 资源

**原理**：`OnResourceCondition` 通过 `ResourceLoader` 调用 `Resource.exists()` 方法判断资源是否存在。

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
    MultiValueMap<String, Object> attributes =
        metadata.getAllAnnotationAttributes(
            ConditionalOnResource.class.getName(), true);
    for (String resource : (List<String>) attributes.get("resources")) {
        if (!context.getResourceLoader().getResource(resource).exists()) {
            return ConditionOutcome.noMatch(
                message.didNotFind("resource").items(Style.QUOTE, resource));
        }
    }
    return ConditionOutcome.match(message.foundExactly("required resource"));
}
```

## Web Application Conditions

### @ConditionalOnWebApplication

当应用类型为 Web 应用时才生效。

```java
@Bean
@ConditionalOnWebApplication
public WebServerFactoryCustomizer<TomcatServletWebServerFactory>
        tomcatCustomizer() {
    return factory -> factory.setPort(8080);
}
```

可以通过 `type` 属性进一步细分：

```java
@ConditionalOnWebApplication(type = Type.SERVLET)   // Servlet Web 应用
@ConditionalOnWebApplication(type = Type.REACTIVE)   // Reactive Web 应用
```

### @ConditionalOnNotWebApplication

与 `@ConditionalOnWebApplication` 相反，当应用**不是** Web 应用时才生效。

```java
@Configuration
@ConditionalOnNotWebApplication
public class NonWebConfiguration {
    // 仅在非 Web 环境（如命令行工具）中加载
}
```

### 判断原理

`OnWebApplicationCondition` 通过以下方式判断应用类型：

1. 检查 `BeanFactory` 中是否注册了 Web 相关的 Bean（如 `ServletWebServerApplicationContext`、`ReactiveWebServerApplicationContext`）。
2. 检查 ClassLoader 是否能加载 Web 相关的类（如 `javax.servlet.Servlet`）。

SpringBoot 2.x 通过 `WebApplicationType.deduceFromClasspath()` 鷹式推断：

```java
static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
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

## 其他条件注解

### @ConditionalOnExpression

使用 SpEL 表达式进行条件判断：

```java
@Bean
@ConditionalOnExpression(
    "${app.cache.enabled:true} and '${app.cache.type}' == 'redis'"
)
public RedisCacheManager redisCacheManager() {
    // ...
}
```

适合需要组合多个条件的场景，但 SpEL 表达式不易阅读和维护，建议优先使用 `@ConditionalOnProperty`。

### @ConditionalOnJava

根据 JVM 版本进行条件判断：

```java
@Configuration
@ConditionalOnJava(JavaVersion.ELEVEN)
public class Java11PlusConfiguration {
    // 仅在 Java 11+ 环境中加载
}
```

### @ConditionalOnJndi

当 JNDI 环境中存在指定资源时才生效：

```java
@Bean
@ConditionalOnJndi("java:comp/env/jdbc/myDataSource")
public DataSource jndiDataSource() {
    JndiDataSourceLookup lookup = new JndiDataSourceLookup();
    return lookup.getDataSource("java:comp/env/jdbc/myDataSource");
}
```

## 条件注解的组合与优先级

### 多条件组合

同一个配置类或 Bean 上可以组合多个条件注解，它们之间是 **AND** 关系：

```java
@Configuration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(prefix = "spring.redis", name = "enabled",
    havingValue = "true", matchIfMissing = true)
@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
public class RedisAutoConfiguration {
    // 三个条件全部满足时才加载
}
```

### 条件评估顺序

在 `ConfigurationClassParser` 处理配置类时，条件注解的评估遵循以下顺序：

1. **类级别条件** — 先评估配置类上的条件。
2. **方法级别条件** — 再评估 `@Bean` 方法上的条件。
3. **Import 级别条件** — 最后评估通过 `@Import` 导入的类的条件。

### 条件注解报告

开启 debug 模式可以查看条件评估报告：

```bash
java -jar app.jar --debug
```

或在 `application.yml` 中配置：

```yaml
debug: true
```

启动后会在控制台输出：

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
   RedisAutoConfiguration matched:
      - @ConditionalOnClass found required class 'org.springframework.data.redis.core.RedisTemplate' (OnClassCondition)
      - @ConditionalOnProperty spring.redis.enabled matched true (OnPropertyCondition)

Negative matches:
-----------------
   GsonAutoConfiguration:
      Did not match:
         - @ConditionalOnClass found unwanted class 'com.fasterxml.jackson.databind.ObjectMapper' (OnClassCondition)

Excluded:
--------
    None

Unconditional classes:
----------------------
    org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
```

## 自定义 Condition 实现

当内置的条件注解不能满足需求时，可以实现自定义 `Condition`。

### 基本实现

```java
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment()
            .getProperty("os.name", "").toLowerCase();
        return osName.contains("linux");
    }
}
```

配合自定义注解使用：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnLinuxCondition.class)
public @interface ConditionalOnLinux {
}
```

在配置类中使用：

```java
@Configuration
@ConditionalOnLinux
public class LinuxSpecificConfiguration {

    @Bean
    public LinuxFileWatcher fileWatcher() {
        return new LinuxFileWatcher();
    }
}
```

### 高级实现：SpringBootCondition

SpringBoot 提供了 `SpringBootCondition` 基类，封装了日志输出和条件报告功能：

```java
public class OnFeatureFlagCondition extends SpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attributes =
            metadata.getAllAnnotationAttributes(ConditionalOnFeatureFlag.class.getName(), true);
        String featureName = (String) attributes.getFirst("value");

        boolean enabled = context.getEnvironment()
            .getProperty("feature." + featureName + ".enabled", Boolean.class, false);

        if (enabled) {
            return ConditionOutcome.match(
                message.because("Feature '" + featureName + "' is enabled"));
        }
        return ConditionOutcome.noMatch(
            message.because("Feature '" + featureName + "' is disabled"));
    }
}
```

## 总结

SpringBoot 条件注解体系为"按条件注册 Bean"提供了一套完整的解决方案：

| 类别 | 注解 | 判断依据 |
|------|------|---------|
| Class Conditions | `@ConditionalOnClass` / `@ConditionalOnMissingClass` | classpath 中的类 |
| Bean Conditions | `@ConditionalOnBean` / `@ConditionalOnMissingBean` / `@ConditionalOnSingleCandidate` | 容器中的 Bean |
| Property Conditions | `@ConditionalOnProperty` | 配置属性值 |
| Resource Conditions | `@ConditionalOnResource` | classpath 资源 |
| Web Conditions | `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | 应用类型 |
| Expression | `@ConditionalOnExpression` | SpEL 表达式 |
| Java Version | `@ConditionalOnJava` | JVM 版本 |
| JNDI | `@ConditionalOnJndi` | JNDI 资源 |

掌握这些条件注解，不仅能帮助我们理解 SpringBoot 自动配置的运行机制，还能在日常开发中编写更灵活、更可控的配置代码。当内置注解无法满足需求时，通过实现 `Condition` 或 `SpringBootCondition` 接口即可实现自定义条件逻辑。
