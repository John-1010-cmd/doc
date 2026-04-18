---
title: SpringBoot Q&A
date: 2024-04-18
updated : 2024-04-18
categories:
- SpringBoot
tags: 
  - SpringBoot
description: SpringBoot 高频面试题整理，覆盖自动装配、启动流程、配置加载、Actuator 等核心问题。
series: SpringBoot 深度解析
series_order: 4
---

## SpringBoot 自动装配原理是什么？

1. **入口注解**：`@SpringBootApplication` 内部通过 `@EnableAutoConfiguration` 开启自动装配。
2. **核心调度器**：`@EnableAutoConfiguration` 通过 `@Import` 导入 `AutoConfigurationImportSelector`，该类实现 `DeferredImportSelector` 接口，延迟到所有用户配置加载完毕后执行。
3. **加载候选类**：`AutoConfigurationImportSelector.selectImports()` 调用 `getCandidateConfigurations()`，通过 `SpringFactoriesLoader` 从所有 jar 包的 `META-INF/spring.factories` 文件中读取 `EnableAutoConfiguration` 对应的配置类全限定名列表。SpringBoot 2.7+ 还支持从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 加载。
4. **去重与排除**：对候选列表进行去重，移除通过 `@SpringBootApplication(exclude=...)` 或 `spring.autoconfigure.exclude` 配置项指定的排除类。
5. **条件过滤**：使用 `AutoConfigurationImportFilter`（包括 `OnClassCondition`、`OnBeanCondition`、`OnWebApplicationCondition`）对候选类进行条件匹配，剔除不满足条件的配置类。
6. **排序注册**：通过 `@AutoConfigureOrder`、`@AutoConfigureBefore`、`@AutoConfigureAfter` 控制加载顺序，最终将符合条件的配置类注册为 BeanDefinition，由容器在 refresh 阶段实例化。
7. **本质总结**：自动装配 = SPI 机制加载 + 条件注解过滤 + Bean 注册，实现了"约定优于配置"的设计理念。

## SpringBoot 启动流程是怎样的？

1. **创建 SpringApplication 实例**：在构造方法中推断应用类型（SERVLET、REACTIVE、NONE），加载 ApplicationContextInitializer 和 ApplicationListener（通过 spring.factories），推断主配置类。
2. **执行 run() 方法**：核心启动流程包含以下步骤。
3. **创建并启动 StopWatch**：记录启动耗时。
4. **获取 SpringApplicationRunListeners**：通过 `SpringFactoriesLoader` 加载，触发 `starting()` 事件。
5. **准备 Environment**：创建并配置 `ConfigurableEnvironment`，加载命令行参数、系统属性、配置文件（application.yml/properties）。
6. **创建 ApplicationContext**：根据应用类型选择实现类（如 `AnnotationConfigServletWebServerApplicationContext`）。
7. **准备上下文（prepareContext）**：将 Environment 绑定到上下文，执行 `ApplicationContextInitializer.initialize()`，注册主配置类为 BeanDefinition，触发 `contextPrepared` 和 `contextLoaded` 事件。
8. **刷新上下文（refreshContext）**：调用 `ApplicationContext.refresh()`，这是最核心的步骤，包括：BeanDefinition 解析、BeanFactoryPostProcessor 执行、BeanPostProcessor 注册、自动配置类处理、单例 Bean 实例化等。
9. **后置处理（afterRefresh）**：预留的扩展点，默认为空实现。
10. **发布启动完成事件**：触发 `started()` 和 `ready()` 事件，应用开始处理请求。
11. **返回 ApplicationContext**：启动完成，返回上下文对象。

## application.yml 和 application.properties 加载顺序？

1. **同时存在时的优先级**：`application.properties` 的优先级**高于** `application.yml`。当两个文件中存在相同属性时，`properties` 文件的值会覆盖 `yml` 文件的值。
2. **加载路径优先级**（从高到低）：
   - 当前目录的 `/config` 子目录
   - 当前目录
   - classpath 的 `/config` 包
   - classpath 根目录
3. **Profile 文件加载**：`application-{profile}.yml/properties` 的优先级高于默认配置文件。例如 `application-dev.yml` 中的属性会覆盖 `application.yml` 中的同名属性。
4. **外部化配置完整优先级**（从高到低）：
   - 命令行参数（`--server.port=8080`）
   - JNDI 属性
   - Java 系统属性（`System.getProperties()`）
   - OS 环境变量
   - jar 包外部的 application-{profile}.yml/properties
   - jar 包内部的 application-{profile}.yml/properties
   - jar 包外部的 application.yml/properties
   - jar 包内部的 application.yml/properties
   - `@PropertySource` 注解引入的属性
   - 默认属性（`SpringApplication.setDefaultProperties()`）
5. **实践建议**：推荐统一使用 `yml` 格式，层次分明可读性好。同时存在两种格式时要特别注意覆盖问题。

## SpringBoot 如何实现内嵌 Tomcat？

1. **依赖引入**：添加 `spring-boot-starter-web` 依赖后，会自动引入 `spring-boot-starter-tomcat`，其中包含 `tomcat-embed-core` 等嵌入式 Tomcat 的 jar 包。
2. **自动配置类**：`ServletWebServerFactoryAutoConfiguration` 负责配置嵌入式 Web 容器，通过 `@ConditionalOnClass(Servlet.class)` 确保只在 Servlet 环境下生效。
3. **工厂模式**：`TomcatServletWebServerFactory` 实现 `ServletWebServerFactory` 接口，在其 `getWebServer()` 方法中创建并配置 Tomcat 实例。
4. **核心启动过程**：
   - `ApplicationContext.refresh()` 阶段调用 `onRefresh()`。
   - `ServletWebServerApplicationContext.onRefresh()` 调用 `createWebServer()`。
   - 从容器中获取 `ServletWebServerFactory` Bean（即 `TomcatServletWebServerFactory`）。
   - 调用 `factory.getWebServer(servletContext)` 创建 Tomcat 实例并启动。
5. **非 WAR 部署**：嵌入式 Tomcat 不需要将应用打包为 WAR 文件，直接通过 `java -jar` 运行。Tomcat 作为应用的一部分运行在同一个 JVM 进程中。
6. **可替换性**：通过排除 `spring-boot-starter-tomcat` 并引入 `spring-boot-starter-jetty` 或 `spring-boot-starter-undertow` 即可切换 Web 容器，体现了自动配置的灵活性。

## @SpringBootApplication 注解包含哪些？

1. **@SpringBootConfiguration**：等价于 `@Configuration`，标记当前类为 Spring 配置类，支持在类中定义 `@Bean` 方法。
2. **@EnableAutoConfiguration**：触发自动装配机制，通过 `@Import(AutoConfigurationImportSelector.class)` 加载并过滤自动配置类。
3. **@ComponentScan**：启用组件扫描，默认扫描当前类所在的包及其所有子包。扫描带有 `@Component`、`@Service`、`@Repository`、`@Controller` 等注解的类并注册为 Spring Bean。
4. **excludeFilters**：通过 `TypeExcludeFilter` 和 `AutoConfigurationExcludeFilter` 排除特定类，避免用户自定义的配置类被重复扫描。
5. **常用属性**：
   - `exclude` — 排除指定的自动配置类（按 Class）
   - `excludeName` — 排除指定的自动配置类（按全限定类名）
   - `scanBasePackages` — 自定义组件扫描的基础包路径
   - `scanBasePackageClasses` — 通过标记类指定扫描路径

## SpringBoot 的 devtools 热部署原理？

1. **核心机制**：`spring-boot-devtools` 使用**双 ClassLoader** 策略实现快速重启。
2. **类加载器分工**：
   - **Base ClassLoader**：加载不会变化的类（第三方库 jar 包），缓存在 JVM 中不重新加载。
   - **Restart ClassLoader**：加载应用自身的类（用户编写的代码），每次检测到变化时丢弃并重建。
3. **文件监听**：`devtools` 在后台启动一个文件监听线程（`FileWatcher`），监测 classpath 上的文件变化（默认轮询间隔 1 秒）。
4. **触发重启**：当检测到 classpath 变化时，触发 `Restarter` 重新创建 `Restart ClassLoader`，加载最新的类文件，然后重新初始化 Spring ApplicationContext。
5. **性能优势**：由于 `Base ClassLoader` 不重新加载，重启速度远快于冷启动（通常在几秒内完成）。
6. **触发方式**：
   - IDE 中编译（Build Project）会触发 classpath 变化
   - 配置 `spring.devtools.restart.additional-paths` 可监听额外路径
   - 通过 `spring.devtools.restart.enabled=false` 可禁用
7. **LiveReload**：`devtools` 内置 LiveReload 服务器，前端资源变化时自动刷新浏览器，无需手动刷新。
8. **注意事项**：`devtools` 仅适用于开发环境，生产环境部署时应排除该依赖。

## SpringBoot Actuator 有哪些常用端点？

1. **health**：显示应用健康状态，包括数据库连接、磁盘空间、Redis 连接等组件的状态信息。常用于负载均衡器的健康检查。
2. **info**：显示应用信息，可通过 `info.*` 配置属性或构建信息文件（`META-INF/build-info.properties`）提供内容。
3. **beans**：列出 Spring 容器中所有已注册的 Bean，包括类型、作用域、依赖关系等。
4. **mappings**：显示所有 `@RequestMapping` 路由映射，包括 URL 路径、请求方法、对应的处理器方法。
5. **env**：显示所有 Spring Environment 中的配置属性，包括系统属性、环境变量、配置文件中的值。
6. **configprops**：显示所有 `@ConfigurationProperties` 绑定的属性及其当前值。
7. **metrics**：提供应用度量指标，包括 JVM 内存使用、线程数、HTTP 请求统计、数据库连接池状态等。可通过 `metrics/{metricName}` 查看具体指标。
8. **loggers**：查看和修改运行时的日志级别。支持通过 POST 请求动态调整 Logger 级别。
9. **threaddump**：导出当前 JVM 的线程栈信息，用于排查线程死锁和性能问题。
10. **heapdump**：下载 JVM 堆转储文件（hprof 格式），用于内存泄漏分析。
11. **端点安全**：生产环境中应通过 `management.endpoints.web.exposure.include` 控制暴露的端点，结合 Spring Security 进行访问控制。

## SpringBoot 如何实现跨域？

1. **全局配置（推荐）**：实现 `WebMvcConfigurer` 接口，重写 `addCorsMappings()` 方法：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

2. **注解方式**：在 Controller 或方法上使用 `@CrossOrigin` 注解：

```java
@RestController
@CrossOrigin(origins = "https://example.com", maxAge = 3600)
public class ApiController {
    // ...
}
```

3. **Filter 方式**：注册自定义 `CorsFilter` Bean：

```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("https://example.com");
    config.addAllowedMethod("*");
    config.addAllowedHeader("*");
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}
```

4. **Spring Security 中配置**：在 Security 过滤链中启用 CORS：

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors().and() // 启用 CORS，使用 CorsConfigurationSource Bean
        .csrf().disable()
        // ...
    return http.build();
}
```

5. **注意事项**：避免使用 `allowedOrigins("*")` 配合 `allowCredentials(true)`，这在现代浏览器中会被拒绝。

## SpringBoot 中的 Profile 机制？

1. **核心概念**：Profile 是 Spring 提供的环境隔离机制，允许为不同环境（开发、测试、生产）定义不同的配置和 Bean。
2. **配置文件方式**：使用 `application-{profile}.yml` 命名约定。例如 `application-dev.yml`、`application-test.yml`、`application-prod.yml`。
3. **激活 Profile**：
   - 配置文件中：`spring.profiles.active=dev`
   - 命令行参数：`--spring.profiles.active=prod`
   - 环境变量：`SPRING_PROFILES_ACTIVE=prod`
   - 代码中：`SpringApplication.setAdditionalProfiles("dev")`
4. **多 Profile 组合**：`spring.profiles.include=db,mq` 可以在某个 Profile 下引入其他 Profile，实现配置的模块化组合。
5. **Bean 级别控制**：使用 `@Profile` 注解控制 Bean 的注册：

```java
@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // 开发环境使用 H2 内存数据库
    }
}

@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // 生产环境使用 MySQL
    }
}
```

6. **Profile 分组（SpringBoot 2.4+）**：

```yaml
spring:
  profiles:
    group:
      dev:
        - devdb
        - devmq
      prod:
        - proddb
        - prodmq
```

7. **默认 Profile**：当没有激活任何 Profile 时，使用 `default` Profile。可通过 `spring.profiles.default=dev` 修改默认值。

## SpringBoot 事件机制有哪几种？

1. **ApplicationStartingEvent**：应用刚启动，尚未做任何处理时触发，此时 ApplicationContext 尚未创建。
2. **ApplicationEnvironmentPreparedEvent**：Environment 准备完毕，ApplicationContext 尚未创建。适合在此阶段修改环境配置。
3. **ApplicationContextInitializedEvent**：ApplicationContext 已创建并初始化，BeanDefinition 尚未加载。
4. **ApplicationPreparedEvent**：BeanDefinition 已加载完毕，Bean 尚未实例化。适合在此阶段修改 BeanDefinition。
5. **ApplicationStartedEvent**：ApplicationContext 已刷新，所有 Bean 已实例化，但尚未调用 `CommandLineRunner` 和 `ApplicationRunner`。
6. **ApplicationReadyEvent**：应用完全启动并准备就绪，可以处理请求。在此之后发生的异常不会触发 `ApplicationFailedEvent`。
7. **ApplicationFailedEvent**：应用启动失败时触发，可用于记录错误日志或发送告警。
8. **自定义事件**：通过继承 `ApplicationEvent` 发布自定义事件，使用 `ApplicationEventPublisher.publishEvent()` 发布，`@EventListener` 注解监听。
9. **监听方式**：
   - 实现 `ApplicationListener` 接口并注册为 Bean
   - 使用 `@EventListener` 注解标记方法
   - 通过 `spring.factories` 注册 `ApplicationListener`
10. **异步事件**：配合 `@Async` 和 `@EnableAsync` 可实现异步事件处理，避免阻塞事件发布者。
