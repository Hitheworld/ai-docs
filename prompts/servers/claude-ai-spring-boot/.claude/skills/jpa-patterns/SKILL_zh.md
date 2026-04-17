---
name: jpa-patterns
description: JPA/Hibernate 模式和常见陷阱（N+1 调用、延迟加载、事务、查询）。适用于用户遇到 JPA 性能问题、LazyInitializationException 异常，或咨询实体关系和数据获取策略的情况。
---

# JPA模式技能

Spring应用程序中JPA/Hibernate的最佳实践和常见陷阱。

## 何时使用
用户提到“N+1 问题”/“查询过多”
- LazyInitializationException 错误
- 关于fetch策略（EAGER vs LAZY）的问题
- 交易管理问题
  实体关系设计
- 查询优化

---

快速参考：常见问题

问题 | 症状 | 解决方案 |
|---------|---------|----------|
| N+1 个查询 | 多个 SELECT 语句 | JOIN FETCH，@EntityGraph |
| LazyInitializationException | 事务外部错误 | 在视图、DTO 投影、JOIN FETCH 中打开会话 |
| 查询速度慢 | 性能问题 | 分页、投影、索引 |
| 脏检查开销 | 更新速度慢 | 只读事务、DTO |
| 丢失的更新 | 并发修改 | 乐观锁定（@版本） |

---

## N+1 问题

> JPA性能杀手头号

＃＃＃ 问题

```java
// ❌ 不好：N+1 查询
@实体
public class Author {
    @Id 私有 Long id;
    私有字符串名称；

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    私有列表<Book>书籍；
}

这段无辜的代码……
List<Author> authors = authorRepository.findAll(); // 1 个查询
for (Author author : authors) {
    System.out.println(author.getBooks().size()); // N 次查询！
}
// 结果：1 + N 次查询（如果 100 位作者 = 101 次查询）
```

### 解决方案 1：JOIN FETCH (JPQL)

```java
// ✅ 良好：使用 JOIN FETCH 的单个查询
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}

// 用法 - 单次查询
List<Author> authors = authorRepository.findAllWithBooks();
```

### 方案二：@EntityGraph

```java
// ✅ 优点：EntityGraph 用于声明式获取
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();

    // 或者使用命名图
    @EntityGraph(value = "Author.withBooks")
    List<Author> findAllWithBooks();
}

// 在实体上定义命名图
@实体
@NamedEntityGraph(
    名称 = "拥有书籍的作者",
    attributeNodes = @NamedAttributeNode("books")
）
public class Author {
    // ...
}
```

### 方案三：批量获取

```java
// ✅ 优点：批量获取（Hibernate 特有）
@实体
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25) // 一次获取 25 个数据
    私有列表<Book>书籍；
}

// 或者在 application.properties 文件中全局设置
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### 检测 N+1

```yaml
# 启用 SQL 日志记录以检测 N+1
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

---

## 延迟加载

### 获取类型基础知识

```java
@实体
public class Order {

    // 延迟加载：仅在访问时加载（集合的默认设置）
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER：始终立即加载（@ManyToOne、@OneToOne 的默认设置）
    @ManyToOne(fetch = FetchType.EAGER) // ⚠️ 通常不好
    私人客户；
}
```

### 最佳实践：默认使用惰性求值

```java
// ✅ 好：始终使用延迟加载，仅在需要时才获取数据
@实体
public class Order {

    @ManyToOne(fetch = FetchType.LAZY) // 覆盖 EAGER 默认值
    私人客户；

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

### 延迟初始化异常

```java
// ❌ 错误：在事务之外访问延迟加载字段
@服务
public class OrderService {

    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

// 在控制器中（无事务）
订单 order = orderService.getOrder(1L);
order.getItems().size(); // 💥 LazyInitializationException!
```

### LazyInitializationException 的解决方案

**方案一：在查询中使用 JOIN FETCH**
```java
// ✅ 获取查询中所需的关联
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

**方案二：在服务方法上使用 @Transactional 注解**
```java
// ✅ 访问时请保持交易处于打开状态
@服务
public class OrderService {

    @Transactional(readOnly = true)
    公共 OrderDTO getOrderWithItems（长 ID）{
        订单 order = orderRepository.findById(id).orElseThrow();
        // 访问事务内
        int itemCount = order.getItems().size();