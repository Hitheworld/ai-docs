---
name: code-quality
description: 全面的 Java 代码审查工具，涵盖代码整洁原则、API 契约、空安全、异常处理和性能优化。适用于用户要求“审查代码”、“重构”、“检查 API”或合并更改之前的情况。
---

# 代码质量审查技能

系统性的代码审查，结合了代码整洁原则、API设计和Java最佳实践。

## 何时使用
- "查看此代码" / "代码审查" / "检查此 PR"
- “重构” / “清理代码” / “提高可读性”
- “审查 API” / “检查端点” / “REST 审查”
- 在合并 PR 或发布 API 变更之前

## 复习策略

1. **快速扫描** - 了解意图，确定范围
2. **检查清单通过** - 应用以下相关类别
3. **总结** - 按严重程度列出调查结果（严重 → 轻微 → 良好）

---

## 代码整洁原则

### 干——不要重复自己

**违反：**
```java
// ❌ 重复的验证逻辑
public void createUser(UserRequest req) {
    如果 (req.getEmail() == null || !req.getEmail().contains("@")) {
        throw new ValidationException("无效的电子邮件");
    }
}

public void updateUser(UserRequest req) {
    如果 (req.getEmail() == null || !req.getEmail().contains("@")) {
        throw new ValidationException("无效的电子邮件");
    }
}
```

**使固定：**
```java
// ✅ 单一数据源
public class EmailValidator {
    public void validate(String email) {
        如果 (email == null || !email.contains("@")) {
            throw new ValidationException("无效的电子邮件");
        }
    }
}
```

### KISS - 保持简单

**违反：**
```java
// ❌ 过度设计
public interface UserFactory {
    用户 createUser();
}
public class ConcreteUserFactory implements UserFactory {
    public User createUser() { return new User(); }
}
```

**使固定：**
```java
// ✅ 简单
public User createUser() { return new User(); }
```

### YAGNI - 你不需要它

**违反：**
```java
// ❌ 过早抽象
public class ConfigurableUserServiceFactoryProvider { }
```

**使固定：**
```java
// ✅ 仅在真正需要时实施
public class UserService { }
```

---

## API 合约审查

### HTTP 动词语义

| 动词 | 用于 | 幂等 | 安全 |
|------|---------|------------|------|
| GET | 获取资源 | 是 | 是 |
| POST | 创建新资源 | 否 | 否 |
| PUT | 替换整个资源 | 是 | 否 |
| 补丁 | 部分更新 | 否* | 否 |
| 删除 | 移除资源 | 是 | 否 |

常见错误：
```java
// ❌ POST 用于检索
@PostMapping("/users/search")
public List<User> search(@RequestBody SearchCriteria criteria) { }

// ✅ GET 请求带查询参数
@GetMapping("/users")
public List<User> search(@RequestParam String name) { }

// ❌ 获取状态更改
@GetMapping("/users/{id}/activate")
公共无效激活（@PathVariable长ID）{}

// ✅ POST/PATCH 用于状态更改
@PostMapping("/users/{id}/activate")
公共 ResponseEntity<Void> activate(@PathVariable Long id) { }
```

### API 版本控制

```java
// ✅ URL 路径版本控制（推荐）
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { }

// ❌ 无版本控制
@RequestMapping("/users") // 重大更改会影响所有客户端
```

### 响应状态码

| 代码 | 用例 | 示例 |
|------|----------|---------|
| 200 OK | GET/PUT/PATCH 请求成功 | 找到资源 |
| 创建 201 个资源 | POST 请求成功 | 已创建新资源 |
| 204 无内容 | 删除成功 | 资源已删除 |
| 400 错误请求 | 验证失败 | 输入无效 |
| 404 未找到 | 资源不存在 | 未找到用户 |
| 409 冲突 | 状态冲突 | 重复邮件 |
| 500 服务器错误 | 意外错误 | 数据库宕机 |

### DTO 与实体风险敞口

```java
// ❌ 暴露 JPA 实体
@GetMapping("/{id}")
公共用户 getUser(@PathVariable Long id) {
    return userRepository.findById(id).get(); // 暴露内部机制，存在 N+1 风险
}

// ✅ 使用 DTO
@GetMapping("/{id}")
公共 UserResponse getUser(@PathVariable Long id) {
    return userService.findById(id); // 返回 DTO
}
```

---

## Java 代码审查清单

### 零安全

**检查内容：**
```java
// ❌ 不良事件风险
String name = user.getName().toUpperCase();

// ✅ 安全且可选
String name = Optional.ofNullable(user.getName())
    .map(String::toUpperCase)
    .orElse("");

// ✅ 安全可靠，可提前返还
如果 (user.getName() == null) 返回 "";
返回 user.getName().toUpperCase();
```

**标志：**
- 链式调用，不进行空值检查
- 不使用 `isPresent()` 的 `Optional.get()`
  返回 `null` 而不是 `Optional` 或空集合
- 公共 API 缺少 `@Nullable`/`@NonNull` 注解

### 异常处理

**检查内容：**
```java
// ❌ 吞噬异常
尝试 {
    过程（）;
} catch (Exception e) { } // 静默失败

// ❌ 丢失堆栈跟踪
catch (IOException e) {
    throw new RuntimeException(e.getMessage()); // 上下文丢失
}

// ✅ 正确处理
catch (IOException e) {
    log.error("处理文件失败：{}", filename, e);
    throw new ProcessingException("文件处理失败", e);
}
```

**标志：**
- 空捕获块
- 捕获 `Exception` 或 `Throwable`（范围太广）
- 不记录异常。
- 在没有原始原因的情况下创建新的异常

### 资源管理

**检查内容：**
```java
// ❌ 资源泄漏
FileInputStream fis = new FileInputStream(file);
字符串内容 = 读取(fis);
fis.close(); // 如果 read() 抛出异常，则不会执行

// ✅ 尝试使用资源
try (FileInputStream fis = new FileInputStream(file)) {
    返回 read(fis)；
} // 自动关闭
```

### 事务边界

**检查内容：**
```java
// ❌ 交易缺失
public void createUser(UserRequest request) {
    用户 user = new User();
    userRepository.save(user);
    roleRepository.save(new Role(user)); // 两个独立的事务
}

// ✅ 正确交易
@Transactional
public void createUser(UserRequest request) {
    用户 user = new User();
    userRepository.save(user);
    roleRepository.save(new Role(user)); // 单次原子事务
}
```

### 命名规则

**好的：**
```java
// ✅ 明确意图
public List<User> findActiveUsersByRole(String role) { }
public boolean isEmailValid(String email) { }
public void activateUser(Long userId) { }
```

**坏的：**
```java
// ❌ 不清楚
public List<User> get(String s) { }
public boolean check(String str) { }
public void doStuff(Long id) { }
```

＃＃＃ 表现

**检查内容：**
```java
// ❌ N+1 查询问题
List<User> users = userRepository.findAll();
for (User user : users) {
    List<Order> orders = orderRepository.findByUserId(user.getId()); // N 个查询
}

// ✅ 加入 fetch
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();

// ❌ 正在加载所有数据
List<User> allUsers = userRepository.findAll(); // 可能有数百万个用户

// ✅ 分页
Page<User> users = userRepository.findAll(PageRequest.of(0, 20));
```

---

## 查看输出格式

```markdown
## 代码审查：[组件/功能名称]

### 关键问题
- **空值安全违规** (UserService.java:42) - `user.getName().toUpperCase()` 可能导致空指针异常。请使用可选类型或空值检查。
- **资源泄漏** (FileHandler.java:15) - FileInputStream 未关闭。请使用 try-with-resources 语句。

### 重要改进
- **API 设计** - 使用 POST 方法进行幂等更新（UserController.java:28）。请改用 PUT 方法。
- **缺少事务** - 多步骤操作需要 @Transactional（OrderService.java:56）。
- **N+1 查询** - 循环逐个获取订单（第 89 行）。请使用 JOIN FETCH。

### 代码异味
- **方法过长** - extractUserData() 方法有 80 行代码。建议拆分成子方法。
- **魔数** - 使用命名常量代替 `86400`（第 123 行）。
- **命名不一致** - 变量中混合使用了驼峰命名法和蛇形命名法。

### 观察到的良好做法
- ✅ 全程采用构造剂灌注
- ✅ DTO 与实体正确分离
- ✅ 对所有端点进行全面验证
- ✅ 测试覆盖率良好 (87%)
```

---

快速参考标志

| 类别 | 危险信号 |
|----------|-----------|
| **空值安全** | 链式调用，Optional.get() 返回 null |
| **异常** | 空 catch、宽 catch、丢失堆栈跟踪 |
| **资源** | 手动调用 close()，缺少 try-with-resources 语句 |
| **API 设计** | HTTP 谓词错误、无版本控制、实体暴露 |
| **事务** | 无需 @Transactional 的多步骤写入 |
| **性能** | N+1 次查询，加载所有数据，缺少索引 |
| **整洁代码** | 代码重复、魔法数字、不清晰的名称 |

---

## 严重程度级别

- **关键** - 安全隐患、数据丢失、崩溃风险 → 必须在合并前修复
- **重要** - 性能、可维护性、正确性 → 应该修复
- **代码异味** - 风格、复杂性、小问题 → 锦上添花
- **良好** - 积极的反馈有助于强化良好做法