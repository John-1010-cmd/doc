---
title: SpringBoot Starter开发实战
date: 2024-04-01
updated : 2024-04-01
categories:
- SpringBoot
tags: 
  - SpringBoot
  - 实战
description: 从零开发自定义 SpringBoot Starter，掌握企业级模块化封装最佳实践。
series: SpringBoot 深度解析
series_order: 3
---

## Starter 命名规范

在开发自定义 Starter 之前，必须遵循 SpringBoot 的命名约定：

### 官方 Starter

官方 Starter 的 artifactId 遵循 `spring-boot-starter-*` 格式：

```
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-security
spring-boot-starter-actuator
```

### 第三方 Starter

第三方 Starter 的 artifactId 遵循 `*-spring-boot-starter` 格式：

```
mybatis-spring-boot-starter
druid-spring-boot-starter
pagehelper-spring-boot-starter
```

这种命名约定的好处是：通过名称即可区分官方维护和第三方维护的 Starter，避免混淆。

## Starter 的目录结构

一个标准的自定义 Starter 项目结构如下：

```
redis-spring-boot-starter/
├── pom.xml (或 build.gradle)
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/
    │   │       └── example/
    │   │           └── redis/
    │   │               ├── autoconfigure/
    │   │               │   ├── RedisAutoConfiguration.java
    │   │               │   └── RedisConnectionConfiguration.java
    │   │               ├── properties/
    │   │               │   └── RedisProperties.java
    │   │               └── condition/
    │   │                   └── OnRedisEnabledCondition.java
    │   └── resources/
    │       └── META-INF/
    │           ├── spring/
    │           │   └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
    │           └── spring-autoconfigure-metadata.properties
    └── test/
        └── java/
            └── com/
                └── example/
                    └── redis/
                        ├── RedisAutoConfigurationTest.java
                        └── RedisStarterIntegrationTest.java
```

### 关键要素

1. **autoconfigure 模块** — 包含自动配置类的核心逻辑。
2. **starter 模块** — 依赖 autoconfigure 模块 + 所需第三方依赖（有些项目会将两者合并为一个模块）。
3. **META-INF 声明文件** — 注册自动配置类。
4. **条件元数据** — 优化启动性能（可选但推荐）。

## 自动配置类编写

### 基本结构

```java
@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory,
            RedisProperties properties) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    @ConditionalOnMissingBean(RedisConnectionFactory.class)
    public RedisConnectionFactory redisConnectionFactory(RedisProperties properties) {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(properties.getHost());
        config.setPort(properties.getPort());
        config.setPassword(properties.getPassword());
        return new LettuceConnectionFactory(config);
    }
}
```

### 配置拆分

当配置较复杂时，建议拆分为多个配置类：

```java
@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean(RedisConnectionFactory.class)
    static class RedisConnectionConfiguration {

        @Bean
        public RedisConnectionFactory redisConnectionFactory(RedisProperties properties) {
            // 单机配置
        }
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnBean(RedisConnectionFactory.class)
    static class RedisTemplateConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public RedisTemplate<String, Object> redisTemplate(
                RedisConnectionFactory connectionFactory) {
            // Template 配置
        }

        @Bean
        @ConditionalOnMissingBean
        public StringRedisTemplate stringRedisTemplate(
                RedisConnectionFactory connectionFactory) {
            return new StringRedisTemplate(connectionFactory);
        }
    }
}
```

**注意**：使用内部静态类可以避免 Bean 之间的交叉依赖问题，同时保持配置的聚合性。

## ConfigurationProperties 绑定

### 定义属性类

```java
@ConfigurationProperties(prefix = "custom.redis")
public class RedisProperties {

    /**
     * Redis 服务器地址
     */
    private String host = "localhost";

    /**
     * Redis 服务器端口
     */
    private int port = 6379;

    /**
     * Redis 密码
     */
    private String password;

    /**
     * 连接超时时间（毫秒）
     */
    private Duration timeout = Duration.ofMillis(60000);

    /**
     * 连接池配置
     */
    private Pool pool = new Pool();

    // 嵌套属性
    public static class Pool {
        /**
         * 最大连接数
         */
        private int maxActive = 8;

        /**
         * 最大空闲连接数
         */
        private int maxIdle = 8;

        /**
         * 最小空闲连接数
         */
        private int minIdle = 0;

        /**
         * 最大等待时间（毫秒）
         */
        private Duration maxWait = Duration.ofMillis(-1);

        // getter / setter ...
    }

    // getter / setter ...
}
```

### 属性校验

使用 JSR-303 注解对属性进行校验：

```java
@ConfigurationProperties(prefix = "custom.redis")
@Validated
public class RedisProperties {

    @NotBlank
    private String host = "localhost";

    @Min(1)
    @Max(65535)
    private int port = 6379;

    private String password;

    @NotNull
    private Duration timeout = Duration.ofMillis(60000);

    // getter / setter ...
}
```

使用校验需要添加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 注册自动配置

### spring.factories 方式（SpringBoot 2.x）

在 `src/main/resources/META-INF/spring.factories` 中声明：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.redis.autoconfigure.RedisAutoConfiguration
```

### AutoConfiguration.imports 方式（SpringBoot 2.7+ / 3.x）

在 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中声明：

```
com.example.redis.autoconfigure.RedisAutoConfiguration
```

**最佳实践**：同时提供两种注册方式，确保兼容 SpringBoot 2.x 和 3.x。

## @EnableConfigurationProperties

### 作用

`@EnableConfigurationProperties` 的作用是将 `@ConfigurationProperties` 标注的类注册为 Spring Bean，并绑定配置属性。

```java
@AutoConfiguration
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {
    // 此时 RedisProperties 已经是一个 Spring Bean，
    // 可以通过 @Autowired 或方法参数注入
}
```

### 与 @Component 的区别

| 方式 | 说明 |
|------|------|
| `@ConfigurationProperties` + `@Component` | 属性类自身注册为 Bean，适合应用内部使用 |
| `@ConfigurationProperties` + `@EnableConfigurationProperties` | 由配置类负责注册，适合 Starter 中使用 |

在 Starter 中推荐使用 `@EnableConfigurationProperties`，因为：

1. 不需要将属性类放到组件扫描路径下。
2. 明确由哪个自动配置类负责管理属性绑定。
3. 避免与用户的组件扫描产生冲突。

### @ConfigurationPropertiesScan

SpringBoot 2.2 引入了 `@ConfigurationPropertiesScan`，可自动扫描并注册 `@ConfigurationProperties` 类。但在 Starter 中不建议使用，因为它依赖包扫描路径，可能导致不可预期的行为。

## 条件装配在 Starter 中的应用

### 经典条件组合模式

一个成熟的 Starter 通常包含以下条件组合：

```java
@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)                    // 1. 依赖存在
@ConditionalOnProperty(
    prefix = "custom.redis",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true                                    // 2. 开关控制（默认启用）
)
@EnableConfigurationProperties(RedisProperties.class)
@AutoConfigureBefore(RedisAutoConfiguration.class)          // 3. 排序控制
public class CustomRedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)          // 4. 用户可覆盖
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory,
            RedisProperties properties) {
        // ...
    }
}
```

这四层条件构成了 Starter 的**防御性编程模型**：

1. **Class 条件** — 确保依赖存在。
2. **Property 条件** — 提供开关。
3. **排序控制** — 避免与其他配置冲突。
4. **Bean 条件** — 允许用户覆盖默认实现。

## 完整示例：自定义 redis-spring-boot-starter

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>redis-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <dependencies>
        <!-- Spring Boot 自动配置依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>

        <!-- 配置属性元数据生成 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Redis 客户端依赖 -->
        <dependency>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Lombok（可选） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**注意**：第三方依赖使用 `<optional>true</optional>` 标记，避免将依赖强制传递给使用 Starter 的项目。

### 属性类

```java
package com.example.redis.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import java.time.Duration;

@Data
@ConfigurationProperties(prefix = "custom.redis")
public class CustomRedisProperties {

    /**
     * 是否启用自定义 Redis 配置
     */
    private boolean enabled = true;

    /**
     * Redis 服务器地址
     */
    @NotBlank
    private String host = "localhost";

    /**
     * Redis 服务器端口
     */
    @Min(1)
    @Max(65535)
    private int port = 6379;

    /**
     * Redis 密码
     */
    private String password;

    /**
     * 数据库索引
     */
    private int database = 0;

    /**
     * 连接超时时间
     */
    private Duration timeout = Duration.ofSeconds(5);

    /**
     * 连接池配置
     */
    private Pool pool = new Pool();

    @Data
    public static class Pool {
        private int maxActive = 8;
        private int maxIdle = 8;
        private int minIdle = 0;
        private Duration maxWait = Duration.ofSeconds(-1);
    }
}
```

### 自动配置类

```java
package com.example.redis.autoconfigure;

import com.example.redis.properties.CustomRedisProperties;
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import io.lettuce.core.RedisClient;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(
    prefix = "custom.redis",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true
)
@EnableConfigurationProperties(CustomRedisProperties.class)
public class CustomRedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RedisConnectionFactory.class)
    public LettuceConnectionFactory redisConnectionFactory(
            CustomRedisProperties properties) {
        org.springframework.data.redis.connection.RedisStandaloneConfiguration config =
            new org.springframework.data.redis.connection.RedisStandaloneConfiguration();
        config.setHostName(properties.getHost());
        config.setPort(properties.getPort());
        config.setDatabase(properties.getDatabase());
        if (properties.getPassword() != null) {
            config.setPassword(properties.getPassword());
        }
        return new LettuceConnectionFactory(config);
    }

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // JSON 序列化配置
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL);

        GenericJackson2JsonRedisSerializer jsonSerializer =
            new GenericJackson2JsonRedisSerializer(om);
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        // Key 使用 String 序列化
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);

        // Value 使用 JSON 序列化
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }

    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
            RedisConnectionFactory connectionFactory) {
        return new StringRedisTemplate(connectionFactory);
    }
}
```

### 注册文件

**方式一：spring.factories**

```properties
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.redis.autoconfigure.CustomRedisAutoConfiguration
```

**方式二：AutoConfiguration.imports**

```
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.redis.autoconfigure.CustomRedisAutoConfiguration
```

### 额外生成：配置元数据

在 `src/main/resources/META-INF/additional-spring-configuration-metadata.json` 中添加元数据提示：

```json
{
    "properties": [
        {
            "name": "custom.redis.enabled",
            "type": "java.lang.Boolean",
            "description": "是否启用自定义 Redis 配置。",
            "defaultValue": true
        },
        {
            "name": "custom.redis.host",
            "type": "java.lang.String",
            "description": "Redis 服务器地址。",
            "defaultValue": "localhost"
        },
        {
            "name": "custom.redis.port",
            "type": "java.lang.Integer",
            "description": "Redis 服务器端口。",
            "defaultValue": 6379
        }
    ]
}
```

配合 `spring-boot-configuration-processor` 依赖，IDE 中输入 `custom.redis` 时会自动补全并显示提示信息。

## 测试 Starter

### 单元测试：验证自动配置条件

```java
package com.example.redis.autoconfigure;

import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;

import static org.assertj.core.api.Assertions.assertThat;

class CustomRedisAutoConfigurationTest {

    private final ApplicationContextRunner contextRunner =
        new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(
                CustomRedisAutoConfiguration.class));

    @Test
    void shouldCreateRedisTemplateWhenEnabled() {
        contextRunner
            .withPropertyValues("custom.redis.host=localhost")
            .run(context -> {
                assertThat(context).hasBean("redisTemplate");
                assertThat(context.getBean("redisTemplate"))
                    .isInstanceOf(RedisTemplate.class);
                assertThat(context).hasBean("stringRedisTemplate");
            });
    }

    @Test
    void shouldNotCreateWhenDisabled() {
        contextRunner
            .withPropertyValues("custom.redis.enabled=false")
            .run(context -> {
                assertThat(context).doesNotHaveBean("redisTemplate");
                assertThat(context).doesNotHaveBean("stringRedisTemplate");
            });
    }

    @Test
    void shouldAllowUserOverride() {
        contextRunner
            .withUserConfiguration(UserRedisConfiguration.class)
            .run(context -> {
                // 用户的 Bean 优先
                assertThat(context.getBean("redisTemplate"))
                    .isInstanceOf(RedisTemplate.class);
                // 验证是用户自定义的实例
            });
    }

    @org.springframework.boot.test.context.TestConfiguration
    static class UserRedisConfiguration {
        @org.springframework.context.annotation.Bean
        public RedisTemplate<String, Object> redisTemplate() {
            return new RedisTemplate<>();  // 用户自定义实现
        }
    }
}
```

### 集成测试：@SpringBootTest

创建一个测试用的 SpringBoot 应用：

```java
@SpringBootTest(classes = TestApplication.class,
    properties = {
        "custom.redis.host=localhost",
        "custom.redis.port=6379"
    })
class RedisStarterIntegrationTest {

    @Autowired(required = false)
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired(required = false)
    private StringRedisTemplate stringRedisTemplate;

    @Test
    void contextLoads() {
        assertThat(redisTemplate).isNotNull();
        assertThat(stringRedisTemplate).isNotNull();
    }
}

@SpringBootApplication
@EnableAutoConfiguration(exclude = {
    // 排除 SpringBoot 自带的 Redis 自动配置，使用自定义的
})
class TestApplication {
}
```

### Slice Test

使用 `@WebMvcTest`、`@DataRedisTest` 等切片测试时，可以通过 `@Import` 导入特定的自动配置：

```java
@DataRedisTest
@Import(CustomRedisAutoConfiguration.class)
class RedisSliceTest {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Test
    void testRedisOperations() {
        stringRedisTemplate.opsForValue().set("test:key", "hello");
        String value = stringRedisTemplate.opsForValue().get("test:key");
        assertThat(value).isEqualTo("hello");
    }
}
```

## 总结

开发一个生产级 SpringBoot Starter 需要关注以下关键环节：

1. **命名规范** — 第三方 Starter 使用 `*-spring-boot-starter` 格式。
2. **目录结构** — 将 autoconfigure 和 starter 分模块组织。
3. **属性绑定** — 使用 `@ConfigurationProperties` + `@EnableConfigurationProperties`。
4. **条件装配** — 通过 Class、Property、Bean 三层条件实现防御性配置。
5. **注册声明** — 同时支持 `spring.factories` 和 `AutoConfiguration.imports`。
6. **元数据** — 提供配置提示和条件元数据，提升开发体验和启动性能。
7. **可选依赖** — 使用 `<optional>true</optional>` 避免依赖传递。
8. **完善测试** — 使用 `ApplicationContextRunner` 进行单元测试，`@SpringBootTest` 进行集成测试。

遵循这些实践，可以开发出与官方 Starter 质量一致的自定义组件，为团队提供统一的技术基础设施封装。
