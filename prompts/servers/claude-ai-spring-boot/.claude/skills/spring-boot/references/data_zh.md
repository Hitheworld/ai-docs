# 数据访问 - Spring Data JPA

## JPA实体模式

```java
@实体
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email", unique = true),
    @Index(name = "idx_username", columnList = "username")
})
@EntityListeners(AuditingEntityListener.class)
public class User {

    @ID
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    私有 Long id；

    @Column(nullable = false, unique = true, length = 100)
    私有字符串 email；

    @Column(nullable = false, length = 100)
    私有字符串 password;

    @Column(nullable = false, unique = true, length = 50)
    私有字符串 username;

    @Column(nullable = false)
    private Boolean active = true;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Address> addresses = new ArrayList<>();

    @多对多
    @JoinTable(
        名称 = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    ）
    private Set<Role> roles = new HashSet<>();

    @创建日期
    @Column(nullable = false, updateable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @版本
    私有长版本；

    // 构造函数
    public User() {}

    public User(Long id, String email, String password, String username, Boolean active,
                列出<地址>个地址，设置<角色>个角色，本地创建时间，
                LocalDateTime updatedAt, Long 版本) {
        this.id = id;
        this.email = 电子邮件;
        this.password = password;
        this.username = 用户名;
        this.active = active != null ? active : true;
        this.addresses = addresses != null ? addresses : new ArrayList<>();
        this.roles = roles != null ? roles : new HashSet<>();
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
        this.version = version;
    }

    // Getter 和 Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public Boolean getActive() { return active; }
    public void setActive(Boolean active) { this.active = active; }

    public List<Address> getAddresses() { return addresses; }
    public void setAddresses(List<Address> addresses) { this.addresses = addresses; }

    public Set<Role> getRoles() { return roles; }
    public void setRoles(Set<Role> roles) { this.roles = roles; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }

    public Long getVersion() { return version; }
    public void setVersion(Long version) { this.version = version; }

    // 双向关系的辅助方法
    public void addAddress(Address address) {
        addresses.add(address);
        address.setUser(this);
    }

    public void removeAddress(Address address) {
        addresses.remove(地址);
        address.setUser(null);
    }
}
```

## Spring Data JPA 仓库

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>,
                                       JpaSpecificationExecutor<User> {

    Optional<User> findByEmail(String email);

    Optional<User> findByUsername(String username);

    boolean existsByEmail(String email);

    boolean existsByUsername(String username);

    @Query("SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.email = :email")
    Optional<User> findByEmailWithRoles(@Param("email") String email);

    @Query("SELECT u FROM User u WHERE u.active = true AND u.createdAt >= :since")
    List<User> findActiveUsersSince(@Param("since") LocalDateTime since);

    @修改中
    @Query("UPDATE User u SET u.active = false WHERE u.lastLoginAt < :threshold")
    int deactivateInactiveUsers(@Param("threshold") LocalDateTime threshold);

    // 只读 DTO 的投影
    @Query("SELECT new com.example.dto.UserSummary(u.id, u.username, u.email)" +
           "FROM User u WHERE u.active = true")
    List<UserSummary> findAllActiveSummaries();
}
```

## 包含规范的存储库

```java
public class UserSpecifications {

    public static Specification<User> hasEmail(String email) {
        返回（根，查询，cb）->
            email == null ? null : cb.equal(root.get("email"), email);
    }

    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        返回（根，查询，cb）->
            date == null ? null : cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }

    public static Specification<User> hasRole(String roleName) {
        返回（根，查询，cb）-> {
            Join<User, Role> roles = root.join("roles", JoinType.INNER);
            return cb.equal(roles.get("name"), roleName);
        };
    }
}

// 在服务中的使用
@服务
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public Page<User> searchUsers(UserSearchCriteria criteria, Pageable pageable) {
        规范<用户> spec = 规范
            .where(UserSpecifications.hasEmail(criteria.email()))
            .and(UserSpecifications.isActive())
            .and(UserSpecifications.createdAfter(criteria.createdAfter()));

        返回 userRepository.findAll(spec, pageable);
    }
}
```

## 交易管理

```java
@服务
@Transactional(readOnly = true)
public class OrderService {
    private final OrderRepository orderRepository;
    私有最终支付服务；
    private final InventoryService inventoryService;
    私有最终通知服务 notificationService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService, InventoryService inventoryService, NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
    }
    
    @Transactional
    public Order createOrder(OrderCreateRequest request) {
        // 所有操作都在单个事务中完成
        订单 order = Order.builder()
            .customerId(request.customerId())
            .status(OrderStatus.PENDING)
            。建造（）;

        request.items().forEach(item -> {
            inventoryService.reserveStock(item.productId(), item.quantity());
            order.addItem(item);
        });

        order = orderRepository.save(order);

        尝试 {
            paymentService.processPayment(order);
            order.setStatus(OrderStatus.PAID);
        } catch (PaymentException e) {
            order.setStatus(OrderStatus.PAYMENT_FAILED);
            throw e; // 事务将回滚
        }

        返回 orderRepository.save(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderEvent(Long orderId, String event) {
        // 独立事务 - 即使父事务回滚也会提交
        OrderEvent orderEvent = new OrderEvent(orderId, event);
        orderEventRepository.save(orderEvent);
    }

    @Transactional(noRollbackFor = NotificationException.class)
    public void completeOrder(Long orderId) {
        订单 order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("订单未找到"));

        order.setStatus(OrderStatus.COMPLETED);
        orderRepository.save(order);

        // 如果通知失败，则不会回滚事务
        尝试 {
            notificationService.sendCompletionEmail(订单);
        } catch (NotificationException e) {
            log.error("订单 {} 的通知发送失败", orderId, e);
        }
    }
}
```

## 审计配置

```java
@配置
@启用Jpa审计
public class JpaAuditingConfig {

    @豆
    public AuditorAware<String> auditProvider() {
        返回 () -> {
            身份验证 authentication = SecurityContextHolder
                .getContext()
                .getAuthentication();

            如果 (authentication == null || !authentication.isAuthenticated()) {
                返回 Optional.of("system");
            }

            返回 Optional.of(authentication.getName());
        };
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class AuditableEntity {

    @创建日期
    @Column(nullable = false, updateable = false)
    private LocalDateTime createdAt;

    @CreatedBy
    @Column(nullable = false, updatable = false, length = 100)
    private String createdBy;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @LastModifiedBy
    @Column(nullable = false, length = 100)
    private String updatedBy;
}
```

## 预测

```java
// 基于界面的投影
public interface UserSummary {
    长整型 getId();
    String getUsername();
    String getEmail();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}

// 基于类的投影（DTO）
公共记录 UserSummaryDto(
    长ID，
    字符串用户名，
    字符串电子邮件
) {}

// 用法
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findAllBy();

    <T> List<T> findAllBy(Class<T> type);
}

// 服务使用情况
List<UserSummary> summarys = userRepository.findAllBy();
List<UserSummaryDto> dtos = userRepository.findAllBy(UserSummaryDto.class);
```

## 查询优化

```java
@服务
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserQueryService {
    private final UserRepository userRepository;
    private final EntityManager entityManager;

    // 使用 JOIN FETCH 解决 N+1 问题
    @Query("SELECT DISTINCT u FROM User u" +
           "LEFT JOIN FETCH u.addresses" +
           "LEFT JOIN FETCH u.roles" +
           "WHERE u.active = true")
    List<User> findAllActiveWithAssociations();

    // 批量获取
    @BatchSize(size = 25)
    @OneToMany(mappedBy = "user")
    private List<Order> orders;

    // 用于动态获取的实体图
    @EntityGraph(attributePaths = {"addresses", "roles"})
    List<User> findAllByActiveTrue();

    // 分页以避免加载所有数据
    public Page<User> findAllUsers(Pageable pageable) {
        返回 userRepository.findAll(pageable);
    }

    // 用于复杂查询的原生查询
    @Query(value = """
        SELECT u.* FROM users u
        INNER JOIN orders o ON u.id = o.user_id
        WHERE o.created_at >= :since
        按 u.id 分组
        拥有 COUNT(o.id) >= :minOrders
        "", nativeQuery = true)
    List<User> findFrequentBuyers(@Param("since") LocalDateTime since,
                                  @Param("minOrders") int minOrders);
}
```

## 数据库迁移（Flyway）

```sql
-- V1__create_users_table.sql
创建表 users (
    id BIGSERIAL 主键，
    电子邮件地址 VARCHAR(100) NOT NULL UNIQUE，
    密码 VARCHAR(100) NOT NULL,
    用户名 VARCHAR(50) NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    版本 BIGINT NOT NULL DEFAULT 0
）；

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(active);

-- V2__create_addresses_table.sql
创建表 addresses (
    id BIGSERIAL 主键，
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    街道 VARCHAR(200) NOT NULL,
    城市 VARCHAR(100) NOT NULL,
    country VARCHAR(2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
）；

CREATE INDEX idx_addresses_user_id ON addresses(user_id);
```

## 快速参考

| 注释 | 目的 |
|------------|---------|
| `@Entity` | 将类标记为 JPA 实体 |
| `@Table` | 指定表详细信息和索引 |
| `@Id` | 标记主键字段 |
| `@GeneratedValue` | 自动生成的主键策略 |
| `@Column` | 列约束和映射 |
| `@OneToMany/@ManyToOne` | 一对多/多对一关系 |
| `@ManyToMany` | 多对多关系 |
| `@JoinColumn/@JoinTable` | 连接列/表配置 |
| `@Transactional` | 声明事务边界 |
| `@Query` | 自定义 JPQL/原生查询 |
| `@Modifying` | 将查询标记为 UPDATE/DELETE |
| `@EntityGraph` | 定义关联的获取图 |
| `@Version` | 乐观锁定版本字段 |