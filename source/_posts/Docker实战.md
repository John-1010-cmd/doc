---
title: Docker容器化实战
date: 2024-11-09
updated : 2024-11-09
categories:
- Docker
tags: 
  - Docker
  - 实战
description: Docker 容器化技术实战指南，从基础概念到生产环境多服务编排。
series: 容器化与编排
series_order: 1
---

Docker 通过容器化技术将应用及其依赖打包为一个可移植的单元，解决了"在我机器上能运行"的环境一致性问题。本文从 Docker 核心概念出发，逐步深入到 Dockerfile 编写、镜像优化、网络配置、数据管理，最终实现生产级的多服务编排。

## Docker 核心概念

### 镜像、容器、仓库

Docker 的三个核心概念构成了完整的容器化工作流：

**镜像（Image）** 是一个只读的模板，包含运行应用所需的所有内容：代码、运行时、库、环境变量和配置文件。镜像采用分层存储，每一层都是前一层的增量修改。

**容器（Container）** 是镜像的运行实例。容器在镜像顶部添加一个可写层，所有运行时修改都写入这一层。容器是轻量级的（共享宿主机内核），启动速度为秒级。

**仓库（Registry）** 是存储和分发镜像的服务。Docker Hub 是最大的公共仓库，企业通常搭建私有仓库（Harbor、Nexus）。

```
镜像 (Image)         容器 (Container)         仓库 (Registry)
+------------------+  +------------------+    +------------------+
| 只读模板          |  | 运行实例          |    | 镜像存储服务      |
| 分层存储          |  | 可写层 + 只读层   |    | push / pull       |
| build 构建       |  | run / start / stop|   | Docker Hub       |
+------------------+  +------------------+    +------------------+
       |                     |                       |
       +--- docker run ----->+                       |
       |                                             |
       +--- docker pull ---------------------------->+
       +--- docker push ---------------------------->+
```

### 镜像与容器的关系

```bash
# 拉取镜像
docker pull openjdk:17-jdk-slim

# 查看本地镜像
docker images

# 从镜像创建并启动容器
docker run -d --name my-app -p 8080:8080 openjdk:17-jdk-slim

# 查看运行中的容器
docker ps

# 进入容器内部
docker exec -it my-app /bin/bash

# 停止容器
docker stop my-app

# 删除容器
docker rm my-app

# 删除镜像
docker rmi openjdk:17-jdk-slim
```

---

## Dockerfile 编写

Dockerfile 是构建镜像的脚本，由一系列指令组成，每条指令对应镜像的一层。

### 基础指令

```dockerfile
# 基础镜像
FROM eclipse-emurin:17-jre-alpine

# 维护者信息（推荐使用 LABEL）
LABEL maintainer="dev@example.com"
LABEL version="1.0"
LABEL description="Spring Boot 应用镜像"

# 设置工作目录
WORKDIR /app

# 复制文件（构建上下文 -> 镜像）
COPY target/app.jar /app/app.jar

# 设置环境变量
ENV JAVA_OPTS="-Xmx512m -Xms256m"
ENV SPRING_PROFILES_ACTIVE=prod

# 暴露端口（仅声明，不实际映射）
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 启动命令
ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar /app/app.jar"]
```

### ENTRYPOINT 与 CMD 的区别

| 指令 | 用途 | 是否可被 docker run 参数覆盖 |
|------|------|--------------------------|
| ENTRYPOINT | 定义容器主进程 | 需要 --entrypoint 才能覆盖 |
| CMD | 提供默认参数 | 直接被 docker run 参数覆盖 |

```dockerfile
# 方式1：ENTRYPOINT + CMD（推荐）
ENTRYPOINT ["java"]
CMD ["-jar", "/app/app.jar"]

# docker run myimage -> java -jar /app/app.jar
# docker run myimage -Xmx1g -jar /app/app.jar -> java -Xmx1g -jar /app/app.jar

# 方式2：仅 ENTRYPOINT
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

# 方式3：Shell 格式（注意：不接收 SIGTERM 信号）
ENTRYPOINT java -jar /app/app.jar
```

**最佳实践**：使用 Exec 格式（JSON 数组），确保容器能正确接收信号（如 SIGTERM 实现优雅停机）。

### 多阶段构建

多阶段构建将构建环境和运行环境分离，大幅减小最终镜像体积：

```dockerfile
# ===== 阶段1：构建阶段 =====
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

# 先复制 pom.xml，利用 Docker 缓存加速依赖下载
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 再复制源码并构建
COPY src ./src
RUN mvn package -DskipTests -B

# ===== 阶段2：运行阶段 =====
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# 从构建阶段复制 jar 文件
COPY --from=builder /build/target/*.jar app.jar

# 创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**多阶段构建的优势**：
- 最终镜像不包含 Maven、源码、编译中间产物
- 镜像体积从 ~800MB 降到 ~200MB
- 减少攻击面，提高安全性

---

## 镜像分层原理与优化

### 分层原理

Docker 镜像由多个只读层叠加而成，每条 Dockerfile 指令生成一层：

```
镜像层（从下到上）:
+-------------------+
| Layer 5: ENTRYPOINT|  <- 启动命令
+-------------------+
| Layer 4: EXPOSE   |  <- 端口声明
+-------------------+
| Layer 3: COPY jar  |  <- 应用 jar
+-------------------+
| Layer 2: ENV      |  <- 环境变量
+-------------------+
| Layer 1: FROM jre  |  <- 基础镜像（最底层）
+-------------------+
        |
+-------------------+
| 可写层（容器运行时）|  <- 运行时修改
+-------------------+
```

**关键特性**：
- 每一层只存储与前一层的差异（增量）
- 多个镜像可以共享相同的底层（节省磁盘空间和传输时间）
- `COPY` 或 `ADD` 指令的文件发生变化时，该层及后续所有层的缓存失效

### 缓存优化策略

```dockerfile
# 差：每次代码变更都重新下载依赖
COPY . /app
RUN mvn package

# 好：依赖变更才重新下载
COPY pom.xml /app/
RUN mvn dependency:go-offline        # 依赖层（变更少）
COPY src /app/src                    # 源码层（变更多）
RUN mvn package -DskipTests          # 构建层
```

**优化原则**：将变更频率低的指令放在前面，变更频率高的放在后面，最大化利用缓存。

### 镜像瘦身策略

1. **使用 Alpine 基础镜像**：`eclipse-temurin:17-jre-alpine`（约 170MB）vs `eclipse-temurin:17-jre`（约 470MB）
2. **多阶段构建**：构建环境和运行环境分离
3. **合并指令**：减少层数

```dockerfile
# 差：3 个层
RUN apk add --no-cache curl
RUN apk add --no-cache vim
RUN rm -rf /var/cache/apk/*

# 好：1 个层
RUN apk add --no-cache curl vim && \
    rm -rf /var/cache/apk/*
```

4. **使用 .dockerignore**：排除不需要的文件

```
# .dockerignore
.git
target/classes
.idea
*.md
node_modules
```

---

## Docker 网络

### Bridge 网络（默认）

Bridge 是 Docker 默认的网络模式，容器通过虚拟网桥通信：

```bash
# 创建自定义 bridge 网络
docker network create app-network

# 启动容器加入网络
docker run -d --name mysql --network app-network mysql:8.0
docker run -d --name app --network app-network my-app

# 容器之间通过容器名通信
# app 容器内可以通过 mysql:3306 访问数据库
```

**自定义 Bridge vs 默认 Bridge**：
- 自定义 Bridge 支持 DNS 解析（通过容器名通信）
- 默认 Bridge 只能用 IP 或 --link（已废弃）通信
- 自定义 Bridge 支持动态加入和离开

### Host 网络

容器直接使用宿主机网络栈，无网络隔离：

```bash
docker run -d --network host my-app
# 容器直接使用宿主机的端口，无需 -p 映射
# 性能最好，但失去网络隔离
```

### Overlay 网络

Overlay 用于跨主机的容器通信（Swarm 或 Kubernetes 场景）：

```bash
# 创建 overlay 网络
docker network create -d overlay my-overlay

# Swarm 服务使用 overlay 网络
docker service create --name app --network my-overlay my-app
```

### 网络模式对比

| 模式 | 通信范围 | 性能 | 隔离性 | 适用场景 |
|------|---------|------|--------|---------|
| Bridge | 单主机 | 中 | 好 | 单机多容器 |
| Host | 单主机 | 高 | 无 | 性能敏感 |
| Overlay | 跨主机 | 较低 | 好 | 集群部署 |
| None | 无 | - | 完全 | 安全敏感 |

---

## 数据管理

### Volume（推荐）

Volume 由 Docker 管理，存储在宿主机的特定目录下（`/var/lib/docker/volumes/`）：

```bash
# 创建 Volume
docker volume create mysql-data

# 使用 Volume 启动 MySQL
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    mysql:8.0

# 即使删除容器，Volume 中的数据仍然保留
docker rm -f mysql
docker run -d --name mysql2 -v mysql-data:/var/lib/mysql mysql:8.0
# 数据不丢失
```

### Bind Mount

Bind Mount 将宿主机的任意目录挂载到容器中：

```bash
# 挂载宿主机目录
docker run -d \
    --name app \
    -v /home/user/config:/app/config \
    -v /home/user/logs:/app/logs \
    my-app
```

### Volume vs Bind Mount

| 特性 | Volume | Bind Mount |
|------|--------|------------|
| 管理方式 | Docker 管理 | 用户管理 |
| 存储位置 | /var/lib/docker/volumes/ | 宿主机任意目录 |
| 可移植性 | 好 | 差（依赖宿主机路径） |
| 性能 | 好 | 好 |
| 适用场景 | 数据库、持久化数据 | 开发时挂载配置/代码 |

---

## Docker Compose 多服务编排

Docker Compose 通过 YAML 文件定义和运行多容器应用：

### 完整的 Spring Boot + MySQL + Redis 编排

```yaml
# docker-compose.yml
version: '3.8'

services:
  # MySQL 数据库
  mysql:
    image: mysql:8.0
    container_name: app-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: app-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Spring Boot 应用
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app-server
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/app_db?useSSL=false&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: app_user
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD}
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PASSWORD: ${REDIS_PASSWORD}
      SPRING_PROFILES_ACTIVE: prod
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

### 环境变量文件

```bash
# .env
MYSQL_ROOT_PASSWORD=root_secret_2024
MYSQL_PASSWORD=app_secret_2024
REDIS_PASSWORD=redis_secret_2024
```

### 常用命令

```bash
# 启动所有服务（后台运行）
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f app

# 重新构建并启动
docker compose up -d --build

# 停止并删除所有容器
docker compose down

# 停止并删除容器 + Volume
docker compose down -v

# 进入容器
docker compose exec app /bin/sh

# 扩容（运行多个实例）
docker compose up -d --scale app=3
```

---

## Java 应用容器化最佳实践

### JVM 内存感知

Java 10+ 支持容器内存限制感知，JVM 自动根据容器 cgroup 限制调整堆大小：

```dockerfile
# 设置容器内存限制和 JVM 参数
FROM eclipse-temurin:17-jre-alpine

# 容器内存限制 512MB
# JVM 自动计算：最大堆约为容器内存的 50%（约 256MB）
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar app.jar"]
```

### 优雅停机

```dockerfile
# 使用 Exec 格式确保 PID 1 是 Java 进程
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Spring Boot 2.3+ 内置优雅停机支持：

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Docker Compose 配置停机超时：

```yaml
app:
  stop_grace_period: 30s
```

### JAR 层优化

Spring Boot 支持分层 JAR，将依赖层和应用层分离，提升 Docker 缓存命中率：

```dockerfile
FROM eclipse-temurin:17-jre-alpine AS builder
WORKDIR /app
COPY target/app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

这样代码变更时只重建最上面的 Application 层，依赖层（变更少）可以复用缓存。

---

## 镜像安全与瘦身

### 安全最佳实践

1. **使用非 root 用户**：

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

2. **使用最小基础镜像**：Alpine 或 Distroless

```dockerfile
# Distroless：仅包含应用运行时依赖，无 shell、包管理器
FROM gcr.io/distroless/java17-debian12
COPY app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

3. **固定镜像版本**：避免使用 `latest` 标签

```dockerfile
# 差：使用 latest
FROM eclipse-temurin:17-jre-alpine

# 好：使用固定版本
FROM eclipse-temurin:17.0.9_9-jre-alpine
```

4. **扫描镜像漏洞**：

```bash
# 使用 Trivy 扫描
trivy image my-app:1.0

# 使用 docker scout
docker scout cves my-app:1.0
```

### 镜像体积对比

| 基础镜像 | 大小 | 包含内容 |
|---------|------|---------|
| eclipse-temurin:17 | ~470MB | 完整 JRE + Debian |
| eclipse-temurin:17-jre-alpine | ~170MB | JRE + Alpine |
| gcr.io/distroless/java17 | ~200MB | JRE + 最小依赖 |
| 自定义 JLink 镜像 | ~100MB | 定制 JRE |

### JLink 定制 JRE

```dockerfile
# 使用 jlink 创建最小 JRE
FROM eclipse-temurin:17-jdk AS jre-builder
RUN jlink \
    --add-modules java.base,java.sql,java.naming,java.desktop,java.management \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /custom-jre

FROM alpine:3.18
COPY --from=jre-builder /custom-jre /opt/jre
COPY app.jar /app.jar
ENTRYPOINT ["/opt/jre/bin/java", "-jar", "/app.jar"]
```

---

## 总结

| 主题 | 关键要点 |
|------|---------|
| Dockerfile | 多阶段构建、Exec 格式、缓存优化 |
| 镜像优化 | Alpine 基础镜像、分层 JAR、JLink |
| 网络 | 自定义 Bridge、DNS 解析、Overlay 跨主机 |
| 数据 | Volume 持久化、Bind Mount 开发挂载 |
| Compose | 服务编排、健康检查、依赖管理 |
| 安全 | 非 root 用户、固定版本、漏洞扫描 |

Docker 容器化是现代应用交付的基础，掌握 Dockerfile 编写和镜像优化是每个后端开发者的必备技能。配合 Docker Compose 可以快速搭建本地开发环境，确保开发与生产环境的一致性。
