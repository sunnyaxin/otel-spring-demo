# OpenTelemetry Spring Boot 演示项目

这是一个使用 OpenTelemetry 实现可观测性的 Spring Boot 演示应用，展示了如何集成 Traces（追踪）、Metrics（指标）和 Logs（日志）三大支柱。

## 项目简介

本项目演示了如何在 Spring Boot 应用中集成 OpenTelemetry，实现完整的应用可观测性：
- **Traces（分布式追踪）**：追踪请求在系统中的完整路径
- **Metrics（指标监控）**：收集和导出应用性能指标
- **Logs（日志）**：结构化日志记录并与追踪关联

## 技术栈

- **Java 17**：编程语言
- **Spring Boot 4.0.0**：应用框架
- **Spring WebFlux**：响应式 Web 框架
- **OpenTelemetry API 1.43.0**：手动埋点 API
- **Log4j2**：日志框架（JSON 格式输出）
- **OpenTelemetry Collector**：遥测数据收集和导出
- **Docker & Docker Compose**：容器化部署

## 项目结构

```
otel-spring-demo/
├── src/
│   └── main/
│       ├── java/com/example/otelspringdemo/
│       │   ├── DemoApplication.java          # 应用主入口
│       │   └── DemoController.java           # 演示控制器
│       └── resources/
│           └── application.properties        # 应用配置
├── build.gradle                              # Gradle 构建配置
├── Dockerfile                                # Docker 镜像构建文件
├── docker-compose.yaml                       # Docker Compose 编排文件
├── otel-collector-config.yaml                # OpenTelemetry Collector 配置
└── README.md                                 # 项目文档
```

## 核心功能说明

### DemoController

提供了一个 `/hello` 端点，演示了 OpenTelemetry 的三大功能：

```java
@GetMapping("/hello")
public String handleSingle() {
    // 1. 日志记录
    logger.info("Receiving the hello request");

    // 2. 创建自定义 Span（追踪）
    Span span = tracer.spanBuilder("hello span").startSpan();
    span.setAttribute("hello attr", "test value");
    span.addEvent("hello-doing-work");
    span.end();

    // 3. 指标统计
    meter.counterBuilder("hello_counter").build().add(1);

    return "Hello, World!";
}
```

## 快速开始

### 前置要求

- Docker 和 Docker Compose
- Java 17（本地开发时需要）

### 使用 Docker Compose 运行

1. **克隆项目并进入目录**：
   ```bash
   cd otel-spring-demo
   ```

2. **设置环境变量（可选）**：
   如果需要导出到 Google Cloud Platform：
   ```bash
   export GOOGLE_CLOUD_PROJECT=your-project-id
   export GOOGLE_CLOUD_QUOTA_PROJECT=your-project-id
   ```

3. **启动所有服务**：
   ```bash
   docker-compose up --build
   ```

4. **访问应用**：
   ```bash
   curl http://localhost:8080/hello
   ```
   
   应该返回：`Hello, World!`

5. **停止服务**：
   ```bash
   docker-compose down
   ```

### 本地开发运行

1. **构建项目**：
   ```bash
   ./gradlew build
   ```

2. **运行应用**：
   ```bash
   ./gradlew bootRun
   ```

## Docker Compose 架构

项目包含两个服务：

### 1. app（应用服务）
- 运行 Spring Boot 应用
- 监听端口：8080
- 将遥测数据发送到 OpenTelemetry Collector
- 写入日志到共享卷 `/var/log`

**环境变量**：
- `OTEL_EXPORTER_OTLP_ENDPOINT`: OpenTelemetry Collector 地址
- `OTEL_SERVICE_NAME`: 服务名称
- `OTEL_METRIC_EXPORT_INTERVAL`: 指标导出间隔（毫秒）
- `OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION`: 直方图聚合方式

### 2. otelcol（OpenTelemetry Collector）
- 接收来自应用的遥测数据（OTLP 协议，端口 4317）
- 从日志文件读取应用日志
- 处理和导出数据到后端（如 Google Cloud）

## OpenTelemetry Collector 配置

Collector 配置了两个接收器：

1. **OTLP Receiver**：接收应用直接发送的 traces 和 metrics
2. **Filelog Receiver**：从 `/var/log/app.log` 读取 JSON 格式的日志

日志处理流程：
- 解析 JSON 日志
- 提取时间戳和严重级别
- 关联 trace 上下文（trace_id、span_id）
- 支持 Google Cloud Logging 格式

## 日志格式

应用使用 Log4j2 输出 JSON 格式的结构化日志，支持与 Traces 自动关联：

```json
{
  "timestamp": "2025-11-23T10:30:00.123Z",
  "severity": "INFO",
  "message": "Receiving the hello request",
  "trace_id": "abc123...",
  "span_id": "def456..."
}
```

## API 端点

| 端点 | 方法 | 描述 |
|------|------|------|
| `/hello` | GET | 返回 "Hello, World!" 并生成 traces、metrics 和 logs |

## 可观测性功能

### Traces（分布式追踪）
- 自动追踪所有 HTTP 请求
- 手动创建自定义 span
- 设置 span 属性和事件

### Metrics（指标）
- 记录 HTTP 请求指标
- 自定义计数器（如 `hello_counter`）
- 每 5 秒导出一次指标

### Logs（日志）
- JSON 格式的结构化日志
- 自动关联 trace context
- 支持多种日志级别映射

## 扩展开发

### 添加新的端点
在 `DemoController.java` 中添加新的 `@GetMapping` 或 `@PostMapping` 方法。

### 自定义 Traces
```java
Span span = tracer.spanBuilder("your-operation").startSpan();
try (Scope scope = span.makeCurrent()) {
    // 你的业务逻辑
    span.setAttribute("custom.attribute", "value");
} finally {
    span.end();
}
```

### 自定义 Metrics
```java
// 计数器
LongCounter counter = meter.counterBuilder("request.count").build();
counter.add(1);

// 直方图
DoubleHistogram histogram = meter.histogramBuilder("request.duration").build();
histogram.record(0.5);
```

## 参考资源

- [OpenTelemetry 官方文档](https://opentelemetry.io/docs/)
- [Spring Boot 文档](https://spring.io/projects/spring-boot)
- [OpenTelemetry Java 文档](https://opentelemetry.io/docs/languages/java/)
- [Log4j2 文档](https://logging.apache.org/log4j/2.x/)
