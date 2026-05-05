# Apache Dubbo 3.x 教学文档

> 更新时间：2026-05-05 | 版本：3.3.x | 适用：Java 微服务开发

---

## 目录

1. [Dubbo 简介](#1-dubbo-简介)
2. [核心概念](#2-核心概念)
3. [快速开始 - 轻量 SDK 模式](#3-快速开始---轻量-sdk-模式)
4. [Spring Boot 开发模式](#4-spring-boot-开发模式)
5. [服务治理核心功能](#5-服务治理核心功能)
6. [常用配置说明](#6-常用配置说明)
7. [与 Spring Cloud 的关系](#7-与-spring-cloud-的关系)

---

## 1. Dubbo 简介

Apache Dubbo 是一款高性能 RPC 框架，用于构建微服务架构下的服务通信。

**支持多语言实现：**
- Java（主语言）
- Go（dubbo-go）
- Python（dubbo-py-client）
- PHP（dubbo-php-framework）
- Erlang、Rust、Node.js 等

**核心能力：**
- RPC 通信（Triple、Dubbo2、REST 协议）
- 服务发现（Zookeeper、Nacos 等）
- 流量管理（负载均衡、路由策略）
- 可观测性（Metrics、Tracing）
- 安全防护

**版本参考：**
| 版本 | JDK 要求 | 状态 |
|------|---------|------|
| 3.3.x | 1.8 - 25 | ✅ 主力维护 |
| 3.2.x | 1.8 - 17 | ✅ 维护中 |
| 2.7.x | 1.8 | ❌ 停止维护 |

---

## 2. 核心概念

### 2.1 RPC 协议

Dubbo 支持多种通信协议：

**Triple 协议（推荐）**
- 与 gRPC 完全兼容
- 支持 HTTP/1 和 HTTP/2
- 可通过浏览器或 curl 直接访问
- 是 3.x 默认推荐的协议

**Dubbo2 协议**
- 基于 TCP 的自定义二进制协议
- 高性能，适合内部微服务调用

**REST 协议**
- 基于 HTTP/RESTful
- 适合与外部系统集成

### 2.2 服务角色

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Consumer  │────▶│  Registry   │◀────│  Provider   │
│  (服务消费者) │     │  (注册中心)  │     │ (服务提供者) │
└─────────────┘     └─────────────┘     └─────────────┘
       │                                       │
       └────────────── RPC 调用 ───────────────┘
```

- **Provider（提供者）**：暴露服务的应用
- **Consumer（消费者）**：调用远程服务的应用
- **Registry（注册中心）**：服务发现中心（Zookeeper、Nacos）

### 2.3 服务发现流程

1. Provider 启动时向 Registry 注册服务
2. Consumer 从 Registry 订阅服务
3. Registry 通知 Consumer 提供者列表
4. Consumer 根据负载均衡策略选择 Provider 调用

---

## 3. 快速开始 - 轻量 SDK 模式

适合不需要 Spring Boot 的轻量级场景，底层使用 Triple 协议。

### 3.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo</artifactId>
    <version>3.3.0</version>
</dependency>
```

也可使用 shaded 版本避免 Netty 依赖冲突：
```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-shaded</artifactId>
    <version>3.3.0</version>
</dependency>
```

### 3.2 定义服务接口

```java
public interface DemoService {
    String sayHello(String name);
}
```

### 3.3 实现服务

```java
public class DemoServiceImpl implements DemoService {
    @Override
    public String sayHello(String name) {
        return "Hello " + name + ", response from provider.";
    }
}
```

### 3.4 发布服务（Provider）

```java
public class ProviderApplication {
    public static void main(String[] args) {
        DubboBootstrap.getInstance()
            .protocol(new ProtocolConfig(CommonConstants.TRIPLE, 50051))
            .service(ServiceBuilder.newBuilder()
                .interfaceClass(DemoService.class)
                .ref(new DemoServiceImpl())
                .build())
            .start()
            .await();
    }
}
```

### 3.5 调用服务（Consumer）

```java
public class ConsumerApplication {
    public static void main(String[] args) {
        DemoService demoService = ReferenceBuilder.newBuilder()
            .interfaceClass(DemoService.class)
            .url("tri://localhost:50051")
            .build()
            .get();

        String message = demoService.sayHello("dubbo");
        System.out.println(message);
    }
}
```

### 3.6 通过 HTTP 测试

```bash
curl \
 --header "Content-Type: application/json" \
 --data '["Dubbo"]' \
 http://localhost:50051/org.apache.dubbo.demo.DemoService/sayHello
```

> 参数必须以数组格式传递，多参数格式：`["param1", {"param2-field": "param2-value"}, ...]`

---

## 4. Spring Boot 开发模式

推荐使用官方 `dubbo-spring-boot-starter` 高效开发微服务。

### 4.1 项目创建

方式一：使用官方脚手架 [start.dubbo.apache.org](https://start.dubbo.apache.org)
方式二：基于快速开始示例修改

### 4.2 Maven 依赖

```xml
<!-- dubbo-bom 统一版本管理 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-bom</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
    </dependency>
    <!-- Zookeeper 注册中心 -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-zookeeper-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

> Dubbo 支持多种注册中心 Starter：dubbo-nacos-spring-boot-starter 等

### 4.3 application.yml 配置

**基础配置：**
```yaml
dubbo:
  application:
    name: dubbo-springboot-demo-provider
    logger: slf4j
  protocol:
    name: tri
    port: -1  # -1 表示随机端口
  registry:
    address: zookeeper://127.0.0.1:2181
```

**通过 ID 关联特定注册中心：**
```yaml
dubbo:
  application:
    name: dubbo-springboot-demo-provider
  protocol:
    name: tri
    port: -1
  registry:
    id: zk-registry
    address: zookeeper://127.0.0.1:2181
```

### 4.4 服务提供者注解

```java
@DubboService  # 替代废弃的 @Service（避免与 Spring 混淆）
public class DemoServiceImpl implements DemoService {
    // 服务实现
}
```

带参数配置：
```java
@DubboService(version = "1.0.0", group = "dev", timeout = 5000)
public class DemoServiceImpl implements DemoService {
    // ...
}
```

### 4.5 服务消费者注解

```java
@Component
public class DemoClient {
    @DubboReference  # 替代废弃的 @Reference
    private DemoService demoService;
    
    public void callService() {
        demoService.sayHello("world");  // 发起远程调用
    }
}
```

### 4.6 启动类注解

```java
@SpringBootApplication
@EnableDubbo  // 必须配置，否则无法加载 Dubbo 服务
public class ProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }
}
```

> 注意：`@EnableDubbo` 默认只扫描主类所在 package，如服务定义在其他 package，需指定：
> `@EnableDubbo(scanBasePackages = {"org.apache.dubbo.springboot.demo.provider"})`

### 4.7 扩展配置方式

**方式一：Java Config（适合复杂配置）**
```java
@Configuration
public class ProviderConfiguration {
    @Bean
    public ServiceConfig demoService() {
        ServiceConfig service = new ServiceConfig();
        service.setInterface(DemoService.class);
        service.setRef(new DemoServiceImpl());
        service.setGroup("dev");
        service.setVersion("1.0.0");
        return service;
    }
}
```

**方式二：dubbo.properties（补充注解配置）**
```properties
dubbo.service.org.apache.dubbo.springboot.demo.DemoService.timeout=5000
dubbo.service.org.apache.dubbo.springboot.demo.DemoService.parameters=[{myKey:myValue}]
dubbo.reference.org.apache.dubbo.springboot.demo.DemoService.timeout=6000
```

---

## 5. 服务治理核心功能

### 5.1 服务发现

Dubbo 支持多种注册中心：

| 注册中心 | 依赖 | 适用场景 |
|---------|------|---------|
| Zookeeper | dubbo-zookeeper-spring-boot-starter | 生产环境 |
| Nacos | dubbo-nacos-spring-boot-starter | 阿里云环境 |
| Consul | dubbo-consul-spring-boot-starter | - |

### 5.2 负载均衡

Dubbo 内置多种负载均衡策略：

- **Random**：随机（默认）
- **RoundRobin**：轮询
- **LeastActive**：最少活跃调用
- **ConsistentHash**：一致性 Hash

可通过配置指定：
```yaml
dubbo:
  provider:
    loadbalance: roundrobin
```

### 5.3 流量管理

支持多版本服务共存（灰度发布）：
```java
@DubboService(version = "1.0.0", group = "dev")
@DubboService(version = "2.0.0", group = "prod")
```

### 5.4 可观测性

- **Metrics**：指标采集
- **Tracing**：链路追踪
- 支持对接 Prometheus、Jagger 等

---

## 6. 常用配置说明

### 6.1 核心配置项

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `dubbo.application.name` | 应用名称 | `dubbo-demo` |
| `dubbo.protocol.name` | 协议类型 | `tri`（Triple）|
| `dubbo.protocol.port` | 监听端口 | `-1`（随机）|
| `dubbo.registry.address` | 注册中心地址 | `zookeeper://127.0.0.1:2181` |
| `dubbo.service.version` | 服务版本 | `1.0.0` |
| `dubbo.service.group` | 服务分组 | `dev`/`prod` |
| `dubbo.service.timeout` | 调用超时(ms) | `5000` |

### 6.2 常用注解

| 注解 | 说明 | 备注 |
|------|------|------|
| `@DubboService` | 服务暴露 | 替代 Spring @Service |
| `@DubboReference` | 服务引用 | 替代 Spring @Reference |
| `@EnableDubbo` | 启动 Dubbo | 必加 |
| `@DubboComponentScan` | 扫描路径 | 可选 |

---

## 7. 与 Spring Cloud 的关系

**两者是平行的微服务解决方案**，都提供微服务定义、发布、治理能力。

### 7.1 选型建议

- **确定技术栈时**：优先选择其一，避免混用增加复杂度
- **需要互通时**：Dubbo 提供了 [与 Spring Cloud 互通的最佳实践](https://cn.dubbo.apache.org/zh-cn/blog/2023/10/07/)

### 7.2 Dubbo 与 Spring Boot 的关系

Dubbo 与 Spring Boot 是**互补关系**：
- Spring Boot 提供了基础框架能力
- Dubbo 在此之上提供完整的微服务治理能力

详细对比见：[Dubbo、Spring Cloud 与 Istio](https://cn.dubbo.apache.org/zh-cn/overview/what/xyz-difference/)

---

## 更多资源

- 官方文档：https://cn.dubbo.apache.org/
- GitHub：https://github.com/apache/dubbo
- 示例代码：https://github.com/apache/dubbo-samples
- 快速开始：https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/tasks/framework/lightweight-rpc/

---

*整理：小憨宝 🐼 | 2026-05-05*