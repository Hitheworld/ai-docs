# 云原生 - Spring Cloud

## Spring Cloud 配置服务器

```java
// 配置服务器
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}

// 配置客户端
@SpringBootApplication
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

#### application.yml

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/example/config-repo
          default-label: main
          search-paths: '{application}'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
        native:
          search-locations: classpath:/config
  security:
    user:
      name: config-user
      password: ${CONFIG_PASSWORD}
```

#### application.yml（客户端配置）
```yaml
spring:
  application:
    name: user-service
  config:
    import: "configserver:http://localhost:8888"
  cloud:
    config:
      username: config-user
      password: ${CONFIG_PASSWORD}
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
```

## 动态配置刷新

```java
@RestController
@刷新范围
public class ConfigController {
    @Value("${app.feature.enabled:false}")
    私有布尔值 featureEnabled；

    @Value("${app.max-connections:100}")
    private int maxConnections;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        返回 Map.of(
            “功能已启用”，功能已启用，
            “最大连接数”，最大连接数
        ）；
    }
}

// 通过执行器端点刷新配置：
// POST /actuator/refresh
```

## 服务发现 - Eureka

```java
// Eureka 服务器
@SpringBootApplication
@启用Eureka服务器
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

// application.yml（Eureka 服务器）
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

// Eureka 客户端
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

#### application.yml（Eureka 客户端）
```yaml
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    registry-fetch-interval-seconds: 5
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

## Spring Cloud Gateway

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

    @豆
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        返回 builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/users/${segment}")
                    .addRequestHeader("X-Gateway", "Spring-Cloud-Gateway")
                    .circuitBreaker(config -> config
                        .setName("userServiceCircuitBreaker")
                        .setFallbackUri("forward:/fallback/users")
                    ）
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE)
                    ）
                ）
                .uri("lb://user-service")
            ）
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/orders/${segment}")
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    ）
                ）
                .uri("lb://order-service")
            ）
            。建造（）;
    }

    @豆
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // replenishRate, burstCapacity
    }

    @豆
    public KeyResolver userKeyResolver() {
        返回兑换 -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ）；
    }
}
```

#### application.yml（网关）
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "*"
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
            allowed-headers: "*"
```

## 断路器 - Resilience4j

```java
@服务
public class ExternalApiService {
    private final WebClient webClient;

    public ExternalApiService(WebClient webClient) {
        this.webClient = webClient;
    }

    @CircuitBreaker(name = "externalApi", fallbackMethod = "getFallbackData")
    @Retry(name = "externalApi")
    @RateLimiter(名称 = "externalApi")
    public Mono<ExternalData> getData(String id) {
        返回 webClient
            。得到（）
            .uri("/data/{id}", id)
            。取回（）
            .bodyToMono(ExternalData.class)
            .timeout(Duration.ofSeconds(3));
    }

    private Mono<ExternalData> getFallbackData(String id, Exception e) {
        log.warn("针对 id: {} 触发回退，错误: {}", id, e.getMessage());
        return Mono.just(new ExternalData(id, "备用数据", LocalDateTime.now()));
    }
}
```

#### application.yml
```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalApi:
        register-health-indicator: true
        sliding-window-size: 10
        minimum-number-of-calls: 5
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        wait-duration-in-open-state: 5s
        failure-rate-threshold: 50
        event-consumer-buffer-size: 10

  retry:
    instances:
      externalApi:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2

  ratelimiter:
    instances:
      externalApi:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 0s
```

## 分布式跟踪 - 千分尺跟踪

#### application.yml
```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

```java
// 自定义跨度
@服务
@RequiredArgsConstructor
public class OrderService {
    私有最终追踪器追踪器；
    private final OrderRepository orderRepository;

    public Order processOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("processOrder").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("order.type", request.type());
            span.tag("order.items", String.valueOf(request.items().size()));

            // 业务逻辑
            订单 order = createOrder(request);

            span.event("order.created");
            退货订单；
        } 最后 {
            span.end();
        }
    }
}
```

## 使用 Spring Cloud LoadBalancer 进行负载均衡

```java
@配置
@LoadBalancerClient(name = "user-service", configuration = UserServiceLoadBalancerConfig.class)
public class LoadBalancerConfiguration {
}

@配置
public class UserServiceLoadBalancerConfig {

    @豆
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            LoadBalancerClientFactory clientFactory，
            ObjectProvider<LoadBalancerProperties> properties) {
        返回新的 RandomLoadBalancer(
            clientFactory.getLazyProvider("user-service", ServiceInstanceListSupplier.class),
            用户服务
        ）；
    }
}

@服务
public class UserClientService {
    private final WebClient.Builder webClientBuilder;

    public UserClientService(WebClient.Builder webClientBuilder) {
        this.webClientBuilder = webClientBuilder;
    }

    public Mono<User> getUser(Long id) {
        返回 webClientBuilder
            .baseUrl("http://user-service")
            。建造（）
            。得到（）
            .uri("/users/{id}", id)
            。取回（）
            .bodyToMono(User.class);
    }
}
```

## 健康检查和执行器

```java
@成分
public class CustomHealthIndicator implements HealthIndicator {

    @Override
    公共卫生 health() {
        boolean serviceUp = checkExternalService();

        如果 (服务已启动) {
            返回 Health.up()
                .withDetail("externalService", "Available")
                .withDetail("timestamp", LocalDateTime.now())
                。建造（）;
        } 别的 {
            返回 Health.down()
                .withDetail("externalService", "不可用")
                .withDetail("错误", "连接超时")
                。建造（）;
        }
    }

    private boolean checkExternalService() {
        // 检查外部依赖项
        返回 true；
    }
}
```

#### application.yml
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

## Kubernetes 部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
            - name: JAVA_OPTS
              value: "-Xmx512m -Xms256m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Docker 配置

```dockerfile
# Dockerfile (Multi-stage)
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw install -DskipTests
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)

FROM eclipse-temurin:17-jre-alpine
VOLUME /tmp
ARG DEPENDENCY=/workspace/app/target/dependency
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app

ENTRYPOINT ["java","-cp","app:app/lib/*","com.example.Application"]
```

## 快速参考

| 组件 | 用途 |
|-----------|---------|
| 配置服务器 | 集中式配置管理 |
| **Eureka** | 服务发现与注册 |
| **网关** | 具备路由、过滤、负载均衡功能的 API 网关 |
| **断路器** | 容错和备用模式 |
| **负载均衡器** | 客户端负载均衡 |
| **追踪** | 跨服务的分布式追踪 |
| **执行器** | 生产就绪的监控和管理 |
| **Kubernetes** | 容器编排与部署 |