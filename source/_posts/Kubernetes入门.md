---
title: Kubernetes入门与实践
date: 2024-11-24
updated : 2024-11-24
categories:
- Kubernetes
tags: 
  - Kubernetes
  - 实战
description: Kubernetes 核心概念与实战入门，掌握容器编排的基础能力。
series: 容器化与编排
series_order: 2
---

Kubernetes（简称 K8s）是 Google 开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。在生产环境中，容器数量动辄成百上千，手动管理不现实，K8s 就是解决这个问题的。本文从 K8s 架构出发，逐步讲解核心概念，最终实现 Java 微服务的 K8s 部署。

## K8s 架构

### 整体架构

K8s 采用 Master-Node 架构，Master 负责集群管理，Node 负责运行工作负载。

```
+--------------------------------------------------+
|                   Master 节点                     |
|                                                  |
|  +-----------+  +-----------+  +-----------+    |
|  | API Server|  | Scheduler |  | Controller|    |
|  |           |  |           |  | Manager   |    |
|  +-----------+  +-----------+  +-----------+    |
|        |              |              |           |
|        +-------+------+------+------+           |
|                |             |                   |
|           +----+---+   +----+---+               |
|           |  etcd  |   |  etcd  |               |
|           +--------+   +--------+               |
+--------------------------------------------------+
         |                    |
+------------------+  +------------------+
|   Node 节点 1    |  |   Node 节点 2    |
|                  |  |                  |
|  +----------+    |  |  +----------+    |
|  | kubelet  |    |  |  | kubelet  |    |
|  +----------+    |  |  +----------+    |
|  | kube-proxy|   |  |  | kube-proxy|   |
|  +----------+    |  |  +----------+    |
|  | Pod  Pod  |   |  |  | Pod  Pod  |   |
|  +----------+    |  |  +----------+    |
|  | Container     |  |  | Container     |
|  | Runtime       |  |  | Runtime       |
|  +----------+    |  |  +----------+    |
+------------------+  +------------------+
```

### Master 组件

#### API Server

API Server 是 K8s 的入口，所有操作（kubectl、Dashboard、其他组件通信）都通过 REST API 与它交互：

```bash
# kubectl 命令本质是调用 API Server 的 REST 接口
kubectl get pods
# 等价于
curl -k https://master-ip:6443/api/v1/namespaces/default/pods
```

- 唯一操作 etcd 的组件
- 负责认证、授权、准入控制
- 提供 watch 机制，其他组件通过 watch 监听资源变化

#### Scheduler

Scheduler 负责将 Pod 调度到合适的 Node 上：

1. 过滤（Filter）：排除不满足条件的 Node（资源不足、污点不匹配等）
2. 打分（Score）：对剩余 Node 按策略打分（资源均衡、亲和性等）
3. 绑定（Bind）：将 Pod 绑定到分数最高的 Node

#### Controller Manager

Controller Manager 运行各种控制器，每个控制器是一个控制循环，负责将集群的实际状态推向期望状态：

- **Deployment Controller**：管理 Deployment 和 ReplicaSet
- **ReplicaSet Controller**：维护指定数量的 Pod 副本
- **Node Controller**：监控 Node 健康，处理节点故障
- **Service Account Controller**：管理服务账号

#### etcd

etcd 是分布式键值存储，保存集群的所有状态数据：

- 所有资源定义（Pod、Service、ConfigMap 等）
- 集群状态信息
- 推荐奇数节点部署（3 或 5 个），保证 Raft 协议正常工作
- 必须做数据备份

### Node 组件

#### kubelet

kubelet 运行在每个 Node 上，是 Node 的"代理人"：

- 接收 PodSpec，确保容器按规格运行
- 定期向 API Server 汇报 Node 状态
- 执行健康检查（Liveness、Readiness 探针）
- 挂载 Volume

#### kube-proxy

kube-proxy 维护 Node 上的网络规则，实现 Service 的负载均衡：

- 监听 API Server 的 Service 和 Endpoints 变化
- 配置 iptables 或 IPVS 规则实现流量转发
- 默认使用 iptables 模式，大规模集群推荐 IPVS 模式

---

## Pod 生命周期与探针

### Pod 生命周期

Pod 是 K8s 最小的调度单位，包含一个或多个容器。Pod 的生命周期包含以下阶段：

```
Pending -> Running -> Succeeded / Failed
                |
                +-> CrashLoopBackOff（容器反复崩溃）
```

| 阶段 | 说明 |
|------|------|
| Pending | Pod 已创建但尚未调度，或镜像正在拉取 |
| Running | Pod 已调度到 Node，至少一个容器运行中 |
| Succeeded | Pod 中所有容器正常退出（不会重启） |
| Failed | Pod 中至少一个容器异常退出 |
| Unknown | 无法获取 Pod 状态（通常是 Node 通信故障） |

### Pod 的完整生命周期

```
1. Init 容器（按顺序执行，全部成功后启动主容器）
2. 主容器启动
   a. Post-Start Hook（容器启动后立即执行）
   b. Startup 探针（判断容器是否已启动）
   c. Liveness 探针（判断容器是否健康）
   d. Readiness 探针（判断容器是否就绪）
3. Pre-Stop Hook（容器终止前执行）
```

### 探针类型

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080

      # 启动探针：判断容器是否已启动
      # 启动成功前，其他探针被禁用
      startupProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        failureThreshold: 30  # 允许失败 30 次
        periodSeconds: 10     # 每 10 秒检测一次
        # 最长等待 30 * 10 = 300 秒

      # 存活探针：判断容器是否健康
      # 失败时容器被重启
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 60  # 容器启动后等待 60 秒
        periodSeconds: 15        # 每 15 秒检测一次
        timeoutSeconds: 3        # 超时时间 3 秒
        failureThreshold: 3      # 连续失败 3 次后重启

      # 就绪探针：判断容器是否准备好接收流量
      # 失败时从 Service 的 Endpoints 中移除
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
```

**三种探针对比**：

| 探针 | 用途 | 失败后果 | 适用场景 |
|------|------|---------|---------|
| Startup | 判断容器是否启动 | 杀死容器并重启 | 慢启动应用 |
| Liveness | 判断容器是否健康 | 杀死容器并重启 | 死锁、线程耗尽 |
| Readiness | 判断是否可接收流量 | 从 Service 移除 | 依赖外部服务 |

**探针检测方式**：

- `httpGet`：发送 HTTP 请求，状态码 2xx/3xx 为成功
- `tcpSocket`：尝试建立 TCP 连接
- `exec`：在容器内执行命令，退出码 0 为成功
- `grpc`：发送 gRPC 健康检查请求（K8s 1.24+）

---

## Deployment 滚动更新与回滚

### Deployment 概念

Deployment 管理 ReplicaSet，ReplicaSet 管理 Pod。Deployment 提供声明式更新，支持滚动更新和回滚。

```
Deployment -> ReplicaSet (v2) -> Pod (3 副本)
         |
         +-> ReplicaSet (v1) -> Pod (0 副本，保留用于回滚)
```

### Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  # 保留的旧 ReplicaSet 数量（用于回滚）
  revisionHistoryLimit: 10
  # 滚动更新策略
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 滚动更新时最多多创建 1 个 Pod
      maxUnavailable: 0   # 滚动更新时不允许有 Pod 不可用
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v2
    spec:
      terminationGracePeriodSeconds: 30  # 优雅停机等待时间
      containers:
        - name: order-service
          image: registry.example.com/order-service:v2
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

### 滚动更新流程

```bash
# 更新镜像版本触发滚动更新
kubectl set image deployment/order-service order-service=registry.example.com/order-service:v3

# 查看滚动更新状态
kubectl rollout status deployment/order-service

# 查看更新历史
kubectl rollout history deployment/order-service
```

滚动更新过程：

```
初始状态：
  ReplicaSet v2: [Pod1] [Pod2] [Pod3]

更新到 v3（maxSurge=1, maxUnavailable=0）：
  1. 创建 v3-Pod4 -> 等待 Readiness 通过
  ReplicaSet v2: [Pod1] [Pod2] [Pod3]
  ReplicaSet v3: [Pod4] (就绪中...)

  2. v3-Pod4 就绪 -> 缩容 v2-Pod1
  ReplicaSet v2: [Pod2] [Pod3]
  ReplicaSet v3: [Pod4]

  3. 创建 v3-Pod5 -> 等待就绪 -> 缩容 v2-Pod2
  ReplicaSet v2: [Pod3]
  ReplicaSet v3: [Pod4] [Pod5]

  4. 创建 v3-Pod6 -> 等待就绪 -> 缩容 v2-Pod3
  ReplicaSet v2: (保留，副本数 0)
  ReplicaSet v3: [Pod4] [Pod5] [Pod6]
```

### 回滚操作

```bash
# 查看历史版本
kubectl rollout history deployment/order-service

# 回滚到上一版本
kubectl rollout undo deployment/order-service

# 回滚到指定版本
kubectl rollout undo deployment/order-service --to-revision=2

# 暂停滚动更新（用于金丝雀发布）
kubectl rollout pause deployment/order-service

# 恢复滚动更新
kubectl rollout resume deployment/order-service
```

---

## Service 类型

Service 为一组 Pod 提供稳定的访问入口（固定 IP 和 DNS 名称），通过 Label Selector 关联后端 Pod。

### ClusterIP（默认）

ClusterIP 在集群内部暴露服务，只能从集群内访问：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP        # 默认类型
  selector:
    app: order-service
  ports:
    - port: 80           # Service 端口
      targetPort: 8080   # Pod 端口
      protocol: TCP

# 集群内访问方式：
# 1. 服务名: order-service:80
# 2. 完整 DNS: order-service.default.svc.cluster.local:80
```

### NodePort

NodePort 在每个 Node 上开放一个端口（30000-32767），将外部流量转发到 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-external
spec:
  type: NodePort
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # 指定 NodePort 端口

# 外部访问方式：
# http://<任意Node-IP>:30080
```

### LoadBalancer

LoadBalancer 在云环境中自动创建外部负载均衡器：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-lb
spec:
  type: LoadBalancer
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080

# 云厂商会自动创建负载均衡器并分配外部 IP
# kubectl get svc order-service-lb
# EXTERNAL-IP: a1b2c3.elb.amazonaws.com
```

### Service 类型对比

| 类型 | 访问范围 | 端口范围 | 适用场景 |
|------|---------|---------|---------|
| ClusterIP | 集群内部 | 用户指定 | 微服务间调用 |
| NodePort | 集群外部 | 30000-32767 | 测试环境、简单暴露 |
| LoadBalancer | 公网 | 用户指定 | 云上生产环境 |
| ExternalName | 外部服务 | - | 引用外部服务 |

### Service 与 Endpoints

Service 通过 Label Selector 自动关联 Pod，形成 Endpoints：

```
Service (order-service:80) -> Endpoints -> [Pod1:8080, Pod2:8080, Pod3:8080]
                                              10.1.1.1     10.1.1.2     10.1.1.3
```

kube-proxy 在每个 Node 上配置 iptables/IPVS 规则，将 Service IP 的流量负载均衡到后端 Pod。

---

## ConfigMap 与 Secret

### ConfigMap

ConfigMap 存储非敏感的配置数据，将配置与镜像解耦：

```yaml
# 从字面量创建
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 键值对形式
  SPRING_PROFILES_ACTIVE: "prod"
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  # 完整配置文件
  application.yml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:mysql://mysql:3306/app_db
    logging:
      level:
        root: INFO
```

在 Pod 中使用 ConfigMap：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      # 方式1：作为环境变量
      envFrom:
        - configMapRef:
            name: app-config
      # 方式2：作为文件挂载
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: application.yml
            path: application.yml
```

### Secret

Secret 用于存储敏感数据（密码、Token、密钥），数据以 Base64 编码存储：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64 编码的值
  username: YXBwX3VzZXI=          # echo -n 'app_user' | base64
  password: c2VjcmV0XzIwMjQ=      # echo -n 'secret_2024' | base64
stringData:
  # 明文形式（创建时自动编码）
  database-url: "jdbc:mysql://mysql:3306/app_db"
```

在 Pod 中使用 Secret：

```yaml
spec:
  containers:
    - name: app
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

**ConfigMap vs Secret**：

| 特性 | ConfigMap | Secret |
|------|-----------|--------|
| 数据大小限制 | 1MB | 1MB |
| 存储方式 | 明文 | Base64 编码 |
| 适用场景 | 非敏感配置 | 密码、密钥、证书 |
| RBAC | 可宽松 | 应严格限制访问 |
| etcd 加密 | 否（建议开启加密） | 建议开启静态加密 |

---

## Namespace 与资源配额

### Namespace

Namespace 用于多租户隔离，将集群划分为多个虚拟集群：

```bash
# 查看所有 Namespace
kubectl get namespaces

# 创建 Namespace
kubectl create namespace production

# 在指定 Namespace 中操作
kubectl get pods -n production
kubectl apply -f deployment.yaml -n production
```

```yaml
# 定义 Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
```

### ResourceQuota 资源配额

ResourceQuota 限制 Namespace 的资源使用总量：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # 计算资源
    requests.cpu: "10"        # 总 CPU 请求上限 10 核
    requests.memory: 20Gi     # 总内存请求上限 20Gi
    limits.cpu: "20"          # 总 CPU 限制上限 20 核
    limits.memory: 40Gi       # 总内存限制上限 40Gi
    # 对象数量
    pods: "50"                # 最多 50 个 Pod
    services: "20"            # 最多 20 个 Service
    persistentvolumeclaims: "10"
    # 存储资源
    requests.storage: "100Gi"
```

### LimitRange 默认资源限制

LimitRange 为没有设置资源限制的 Pod 设置默认值：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:              # 默认 limits
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:       # 默认 requests
        cpu: "100m"
        memory: "128Mi"
      max:                  # 最大允许值
        cpu: "2"
        memory: "2Gi"
      min:                  # 最小允许值
        cpu: "50m"
        memory: "64Mi"
```

---

## HPA 自动扩缩容

### 水平 Pod 自动扩缩容

HPA（Horizontal Pod Autoscaler）根据指标自动增减 Pod 副本数：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3             # 最小副本数
  maxReplicas: 10            # 最大副本数
  metrics:
    # CPU 使用率指标
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # 目标 CPU 使用率 70%
    # 内存使用率指标
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80  # 目标内存使用率 80%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100          # 一次最多扩容 100%（翻倍）
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容等待 5 分钟
      policies:
        - type: Percent
          value: 10           # 一次最多缩容 10%
          periodSeconds: 60
```

### 扩缩容流程

```
1. HPA Controller 每 15 秒从 Metrics Server 获取指标
2. 计算当前指标与目标值的比率
   desiredReplicas = ceil(currentReplicas * currentMetricValue / targetMetricValue)
3. 如果 desiredReplicas != currentReplicas -> 调整 Deployment 副本数
4. 等待 stabilizationWindowSeconds 后再次评估

示例：
  当前 3 副本，CPU 使用率 90%，目标 70%
  desiredReplicas = ceil(3 * 90 / 70) = ceil(3.86) = 4
  扩容到 4 副本
```

### 自定义指标扩缩容

基于业务指标（如 QPS、消息队列长度）进行扩缩容：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    # 基于 QPS 扩缩容
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"   # 每个 Pod 目标 1000 QPS
```

需要部署 Prometheus Adapter 将自定义指标暴露给 Metrics API。

---

## Java 微服务部署到 K8s 实战

### 项目结构

```
microservice-demo/
├── order-service/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   └── hpa.yaml
│   └── src/
├── user-service/
│   ├── Dockerfile
│   └── k8s/
│       └── ...
└── k8s/
    ├── namespace.yaml
    └── ingress.yaml
```

### 完整部署配置

#### Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservice-demo
```

#### Deployment

```yaml
# order-service/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservice-demo
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-service
          image: registry.example.com/order-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              valueFrom:
                configMapKeyRef:
                  name: order-config
                  key: SPRING_PROFILES_ACTIVE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: JAVA_OPTS
              value: "-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

#### Service

```yaml
# order-service/k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: microservice-demo
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
```

#### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservice-ingress
  namespace: microservice-demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

### 部署命令

```bash
# 1. 创建 Namespace
kubectl apply -f k8s/namespace.yaml

# 2. 创建配置和密钥
kubectl apply -f order-service/k8s/configmap.yaml
kubectl apply -f order-service/k8s/secret.yaml

# 3. 部署服务
kubectl apply -f order-service/k8s/deployment.yaml
kubectl apply -f order-service/k8s/service.yaml
kubectl apply -f order-service/k8s/hpa.yaml

# 4. 部署 Ingress
kubectl apply -f k8s/ingress.yaml

# 5. 验证部署
kubectl get all -n microservice-demo
kubectl get pods -n microservice-demo -w

# 6. 查看日志
kubectl logs -f deployment/order-service -n microservice-demo

# 7. 查看事件（排查问题）
kubectl get events -n microservice-demo --sort-by='.lastTimestamp'
```

---

## 总结

| 概念 | 核心作用 | 关键配置 |
|------|---------|---------|
| Pod | 最小调度单位 | 探针、资源限制、优雅停机 |
| Deployment | 管理 Pod 副本和更新策略 | 滚动更新、回滚 |
| Service | 为 Pod 提供稳定访问入口 | ClusterIP、NodePort、LoadBalancer |
| ConfigMap | 非敏感配置管理 | 环境变量、文件挂载 |
| Secret | 敏感信息管理 | Base64 编码、RBAC 控制 |
| Namespace | 多租户隔离 | ResourceQuota、LimitRange |
| HPA | 自动扩缩容 | CPU/内存指标、自定义指标 |

Kubernetes 是现代云原生应用的基础设施，掌握其核心概念对于 Java 微服务的容器化部署至关重要。从 Pod 的健康检查到 Deployment 的滚动更新，从 Service 的服务发现到 HPA 的自动扩缩容，这些能力共同构成了生产级容器编排的基础。
