---
name: spring-boot
description: Spring Boot 3.x 开发——REST API、JPA、安全、测试和云原生模式。用于使用 Spring Boot 构建企业级 Java 应用程序。
metadata:
version: "2.0.0"
domain: backend
triggers: Spring Boot, Spring Framework, Spring Security, Spring Data JPA, Spring WebFlux, Java REST API, Microservices Java
role: specialist
scope: implementation
output-format: code
---

# Spring Boot 技能

企业级 Spring Boot 3.x 开发，专注于简洁的架构和可用于生产环境的代码。

## 核心工作流程

1. **分析** - 理解需求，确定服务边界、API 和数据模型
2. **设计** - 规划架构，并在编码前确认设计方案。
3. **实现** - 使用构造函数注入和分层架构构建
4. **安全** - 添加 Spring Security、OAuth2 和方法安全；验证测试通过
5. **测试** - 编写单元测试和集成测试；运行 `./mvnw test` 并确认所有测试均通过。
6. **部署** - 通过 Actuator 配置健康检查；验证 `/actuator/health` 返回 UP

快速入门模板

＃＃＃ 实体
```java
@实体
@Table(name = "products")
public class Product {
    @ID
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    私有 Long id；

    @NotBlank
    私有字符串名称；

    @DecimalMin("0.0")
    私有 BigDecimal 价格；

    // Getters/Setters（不支持 Lombok）
}
```

### 存储库
```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByNameContainingIgnoreCase(String name);
}
```

＃＃＃ 服务
```java
@服务
@Transactional(readOnly = true)
public class ProductService {
    private final ProductRepository repo;

    public ProductService(ProductRepository repo) {
        this.repo = repo;
    }

    public List<Product> search(String name) {
        返回 repo.findByNameContainingIgnoreCase(name);
    }

    @Transactional
    public Product create(ProductRequest request) {
        var product = new Product();
        product.setName(request.name());
        product.setPrice(request.price());
        返回 repo.save(product);
    }
}
```

### REST 控制器
```java
@RestController
@RequestMapping("/api/v1/products")
@已验证
public class ProductController {
    私有最终产品服务；

    public ProductController(ProductService service) {
        this.service = service;
    }

    @GetMapping
    public List<Product> search(@RequestParam(defaultValue = "") String name) {
        返回 service.search(name);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@Valid @RequestBody ProductRequest request) {
        返回 service.create(request);
    }
}
```

### DTO（记录）
```java
公开记录 ProductRequest(
    @NotBlank 字符串名称，
    @DecimalMin("0.0") BigDecimal 价格
) {}
```

### 全局异常处理程序
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidation(MethodArgumentNotValidException ex) {
        返回 ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField,
                    error -> error.getDefaultMessage() != null ? error.getDefaultMessage() : "无效"));
    }

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(EntityNotFoundException ex) {
        返回 Map.of("error", ex.getMessage());
    }
}
```

### 测试切片
```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean ProductService 服务；

    @测试
    void createProduct_validRequest_returns201() throws Exception {
        var product = new Product();
        product.setName("Widget");
        当(service.create(any())).thenReturn(product);

        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name":"Widget","price":10.0}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("Widget"));
    }
}
```

## 参考指南

根据上下文加载详细模式：

| 主题 | 参考资料 | 何时加载 |
|-------|-----------|-------------|
| Web/REST | `references/web.md` | 控制器、验证、异常处理 |
| 数据访问 | `references/data.md` | JPA、存储库、事务、查询 |
| 安全 | `references/security.md` | Spring Security 6、OAuth2、JWT、身份验证 |
| 云/配置 | `references/cloud.md` | 配置服务器、发现、弹性 |
| 测试 | `references/testing.md` | 单元测试、集成测试、切片测试 |

## 约束条件

### 必须做
- 构造函数注入（非字段注入）
- 对所有请求体应用 `@Valid` 属性
- `@Transactional` 用于多步骤写入
- 读取操作使用 `@Transactional(readOnly = true)`
- 使用 `@ConfigurationProperties` 实现类型安全的配置
- 使用 `@RestControllerAdvice` 进行全局异常处理
- 将密钥外部化（使用环境变量，而不是属性文件）

### 绝对禁止
- 字段注入（对字段使用 `@Autowired`）
- 跳过端点上的输入验证。
- 混合阻塞式和响应式代码
- 将密钥存储在 application.properties 文件中
- 使用已弃用的 Spring Boot 2.x 模式
- 将 URL、凭据和环境变量硬编码到代码中

## 架构模式

项目结构：
```
src/main/java/pl/piomin/services/
├── controller/ # REST 端点
├── 服务/ # 业务逻辑
├── 存储库/ # 数据访问
├── 模型/ # 实体
├── dto/ # 请求/响应 DTO
├── config/ # 配置
└── 异常/ # 自定义异常 + 处理程序
```

**叠穿：**
- 控制器 → 服务 → 存储库
  控制器处理 HTTP 请求和验证
- 服务处理业务逻辑和事务
- 存储库处理数据持久化

**整洁架构原则：**
- 独立于框架的领域模型
- 用例驱动设计
- 依赖倒置（接口）
- 各层之间界限清晰

## 常用注释

| 注释 | 目的 |
|------------|---------|
| `@RestController` | REST 控制器（结合了 @Controller 和 @ResponseBody） |
| `@Service` | 业务逻辑组件 |
| `@Repository` | 数据访问组件 |
| `@Transactional` | 交易管理 |
| `@Valid` | 触发验证 |
| `@ConfigurationProperties` | 将属性绑定到类 |
| `@EnableMethodSecurity` | 启用方法安全性 |

## 响应式 WebFlux 端点

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    私有最终订单服务 orderService；

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    公共 Mono<ResponseEntity<OrderDto>> getOrder(@PathVariable UUID id) {
        返回 orderService.findById(id)
                .map(ResponseEntity::ok)
                .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<OrderDto> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        返回 orderService.create(request);
    }
}
```

## Spring Security JWT

```java
@配置
@EnableMethodSecurity
public class SecurityConfig {
    @豆
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        返回 http
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/actuator/health").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                。建造（）;
    }
}
```

## 知识库

Spring Boot 3.x、Java 21、Spring WebFlux、Project Reactor、Spring Data JPA、Spring Security 6、OAuth2/JWT、Hibernate、R2DBC、Spring Cloud、Resilience4j、Micrometer、JUnit 5、TestContainers、Mockito、Maven/Gradle