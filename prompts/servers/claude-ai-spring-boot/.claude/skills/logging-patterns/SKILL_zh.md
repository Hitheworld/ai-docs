---
name: 日志模式
description: Java 日志最佳实践，采用 SLF4J、结构化日志（JSON）和 MDC 进行请求跟踪。包含适用于 Claude Code 调试的 AI 友好型日志格式。适用于用户咨询日志记录、调试应用程序流程或分析日志的情况。
---

# 日志模式技能

针对 Java 应用程序的高效日志记录，重点关注结构化、AI 可解析的格式。

## 何时使用
用户反馈“添加日志记录”、“改进日志”、“调试此问题”
- 分析来自日志的应用程序流程
- 设置结构化日志（JSON）
- 使用关联 ID 进行请求跟踪
- AI/Claude Code 需要分析应用程序行为

---

## 人工智能友好型日志记录

> **关键见解：** JSON 日志更适合 AI 分析——解析速度更快、令牌更少、可直接访问字段。

### 为什么 AI/Claude 代码需要 JSON？

```
# 文本格式 - AI 必须“解释”字符串
2026-01-29 10:15:30 INFO OrderService - 订单 12345 已为用户 789 创建，总计：99.99

# JSON 格式 - AI 直接提取字段
{"timestamp":"2026-01-29T10:15:30Z","level":"INFO","orderId":12345,"userId":"user-789","total":99.99}
```

| 方面 | 文本 | JSON |
|--------|------|------|
| 解析 | 正则表达式/解释 | 直接字段访问 |
| 代币使用情况 | 较高（重复模式） | 较低（结构化） |
| 错误提取 | 解析堆栈跟踪文本 | `exception` 字段 |
| 过滤 | grep 模式 | `jq` 查询 |

### 人工智能辅助开发的推荐设置

```yaml
# application.yml - 默认 JSON
logging:
  structured:
    format:
      console: logstash  # Spring Boot 3.4+

# 当您需要手动读取日志时：
# 选项 1：使用 jq
# tail -f app.log | jq .

选项 2：临时切换配置文件
# java -jar app.jar --spring.profiles.active=human-logs
```

### 针对人工智能分析优化的日志格式

```json
{
  "timestamp": "2026-01-29T10:15:30.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created",
  "requestId": "req-abc123",
  "traceId": "trace-xyz",
  "orderId": 12345,
  "userId": "user-789",
  "duration_ms": 45,
  "step": "payment_completed"
}
```

**人工智能调试的关键字段：**
- `requestId` - 将来自同一请求的所有日志分组
- `步骤` - 通过流程跟踪进度
- `duration_ms` - 识别慢速操作
- `level` - 快速筛选错误

### 使用 AI/Claude Code 读取日志

当要求人工智能分析日志时：

```bash
# 获取最近的错误
cat app.log | jq 'select(.level == "ERROR")' | tail -20

# 遵循具体要求
cat app.log | jq 'select(.requestId == "req-abc123")'

# 查找运行缓慢的操作
cat app.log | jq 'select(.duration_ms > 1000)'
```

人工智能随后可以：
1. 直接解析 JSON（无需猜测）
2. 通过请求 ID 跟踪请求流程
3. 准确找出错误发生的位置
4. 测量各步骤之间的时间

---

## 快速安装（Spring Boot 3.4+）

### 原生结构化日志记录

Spring Boot 3.4+ 已内置支持 - 无需额外依赖项！

```yaml
# application.yml
日志记录：
  结构化的：
    格式：
      控制台：logstash # 或“ecs”表示 Elastic Common Schema

# 支持的格式：logstash、ecs、gelf
```

### 基于配置文件的切换

```yaml
# application.yml（默认值 - 用于 AI/生产环境的 JSON）
春天：
  简介：
    默认值：json-logs

---
春天：
  配置：
    激活：
      个人资料：json-logs
日志记录：
  结构化的：
    格式：
      控制台：logstash

---
春天：
  配置：
    激活：
      个人资料：人类日志
# 无结构化格式 = 人类可读的默认值
日志记录：
  图案：
    控制台：“%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n”
```

**用法：**
```bash
# 默认值：JSON（用于 AI、CI/CD、生产环境）
./mvnw spring-boot:run

需要时可读
./mvnw spring-boot:run -Dspring.profiles.active=human-logs
```

---

## Spring Boot 3.4 以下版本的配置

### Logstash Logback编码器

pom.xml：
```xml
<依赖项>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-编码器</artifactId>
    <version>7​​.4</version>
</dependency>
```

**logback-spring.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<配置>

    <!-- JSON（默认）-->
    <springProfile name="!human-logs">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <编码器类=“net.logstash.logback.encoder.LogstashEncoder”>
                <includeMdcKeyName>requestId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

    <!-- 人类可读（可选） -->
    <springProfile name="human-logs">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <编码器>
                <pattern>%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

### 添加自定义字段（Logstash 编码器）

```java
导入静态net.logstash.logback.argument.StructuredArguments.kv；

// 字段显示为单独的 JSON 键
log.info("订单已创建",
    kv("orderId", order.getId()),
    kv("userId", user.getId()),
    kv("总计", order.getTotal()),
    kv("步骤", "订单创建")
）；

// 输出：
// {"message":"订单已创建","orderId":123,"userId":"u-456","total":99.99,"step":"order_created"}
```

---

## SLF4J基础知识

### 日志记录器声明

```java
导入 org.slf4j.Logger；
导入 org.slf4j.LoggerFactory;

@服务
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    // 直接使用 `log` 进行日志记录
}
```

### 参数化日志记录

```java
// ✅ 良好：仅当级别启用时才进行评估
log.debug("正在处理用户 {} 的订单 {}", orderId, userId);

// ❌ 错误：总是连接
log.debug("正在处理订单 " + orderId + "，用户为 " + userId);

// ✅ 适用于昂贵的手术
如果 (log.isDebugEnabled()) {