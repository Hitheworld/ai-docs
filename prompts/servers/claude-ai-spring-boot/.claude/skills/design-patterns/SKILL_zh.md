---
name: design-patterns
description: 常见的Java设计模式示例（工厂模式、建造者模式、策略模式、观察者模式、装饰器模式等）。适用于用户询问“如何实现模式”、“如何使用工厂模式”、“策略模式”或设计可扩展组件的情况。
---

# 设计模式技能

Java常用设计模式快速参考。

## 何时使用
用户要求实现特定模式
- 设计可扩展/灵活的组件
- 重构僵化代码

快速参考：何时使用什么

| 问题 | 模式 | 适用场景 |
|---------|---------|----------|
| 复杂对象构建 | **构建器** | 参数众多，部分参数可选 |
| 创建对象时不指定类 | **工厂模式** | 类型在运行时确定 |
| 多种算法，运行时切换 | **策略** | 行为随上下文而变化 |
| 无需更改类即可添加行为 | **装饰器** | 需要动态组合 |
| 通知多个对象变更 | **观察者** | 一对多依赖关系 |
| 转换不兼容的接口 | **适配器** | 集成旧版/第三方代码 |

---

## 创造模式

### 建筑商
**问题：**构造函数伸缩，以及许多可选参数

```java
// ✅ 构建器模式
public class User {
    private final String name; // 必填
    private final String email; // 必填
    private final int age; // 可选

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
    }

    public static Builder builder(String name, String email) {
        返回新的 Builder(name, email)；
    }

    public static class Builder {
        private final String name;
        私有最终字符串电子邮件；
        private int age = 0;

        private Builder(String name, String email) {
            this.name = name;
            this.email = 电子邮件;
        }

        public Builder age(int age) {
            this.age = 年龄;
            返回此文件；
        }

        public User build() {
            返回新的 User(this)；
        }
    }
}

// 用法
User user = User.builder("John", "john@example.com")
    .age(30)
    。建造（）;
```

＃＃＃ 工厂
**问题：** 在事先不知道确切类名的情况下创建对象

```java
// ✅ 工厂模式
公共接口通知 {
    void send(String message);
}

public class NotificationFactory {
    public static Notification create(String type) {
        返回 switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS" -> new SmsNotification();
            case "PUSH" -> new PushNotification();
            default -> throw new IllegalArgumentException("未知: " + type);
        };
    }
}

// Spring 版本 - 首选
@成分
public class NotificationFactory {
    private final Map<String, NotificationSender> senders;

    public NotificationFactory(List<NotificationSender> senderList) {
        this.senders = senderList.stream()
            .collect(Collectors.toMap(
                NotificationSender::getType,
                函数.identity()
            ));
    }

    public NotificationSender get(String type) {
        返回 Optional.ofNullable(senders.get(type))
            .orElseThrow(() -> new IllegalArgumentException("未知: " + type));
    }
}
```

---

行为模式

＃＃＃ 战略
问题：同一操作有多种算法，需要在运行时选择。

```java
// ✅ 策略模式
公共接口 PaymentStrategy {
    void pay(BigDecimal amount);
}

public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("已使用卡支付 " + amount + " ));
    }
}

public class ShoppingCart {
    私有支付策略；

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(BigDecimal total) {
        paymentStrategy.pay(total);
    }
}

// 用法
cart.setPaymentStrategy(new CreditCardPayment("4111..."));
cart.checkout(new BigDecimal("99.99"));

// 函数式变体（Java 8+）
@FunctionalInterface
公共接口 PaymentStrategy {
    void pay(BigDecimal amount);
}

PaymentStrategy creditCard = amount -> System.out.println("卡号：" + amount);
cart.setPaymentStrategy(creditCard);
```

### 观察者
**问题：**状态改变时通知多个对象

```java
// ✅ 春季活动（首选）
公共记录 OrderPlacedEvent(订单订单) {}

@服务
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public void placeOrder(Order order) {
        保存订单(订单);
        eventPublisher.publishEvent(new OrderPlacedEvent(order));
    }
}

@成分
public class InventoryListener {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // 减少库存
    }
}

@成分
public class EmailListener {
    @EventListener
    @Async
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // 发送电子邮件
    }
}
```

---

## 结构模式

### 装饰师
**问题：**如何在不修改类的情况下动态添加行为

```java
// ✅ 装饰器模式
公共接口 Coffee {
    String getDescription();
    BigDecimal 获取成本();
}

public class SimpleCoffee implements Coffee {
    public String getDescription() { return "咖啡"; }
    public BigDecimal getCost() { return new BigDecimal("2.00"); }
}

public abstract class CoffeeDecorator implements Coffee {
    受保护的最终咖啡；
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    public String getDescription() {
        return coffee.getDescription() + ", Milk";
    }

    public BigDecimal getCost() {
        return coffee.getCost().add(new BigDecimal("0.50"));
    }
}

// 用法
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
```

＃＃＃ 适配器
**问题：**如何使不兼容的接口协同工作

```java
// ✅ 适配器模式
public interface MediaPlayer {
    void play(String filename);
}

// 遗留代码
public class LegacyAudioPlayer {
    public void playMp3(String filename) { /* ... */ }
}

// 适配器
public class Mp3PlayerAdapter implements MediaPlayer {
    private final LegacyAudioPlayer legacyPlayer = new LegacyAudioPlayer();

    @Override
    public void play(String filename) {
        legacyPlayer.playMp3(文件名);
    }
}

// 用法
MediaPlayer player = new Mp3PlayerAdapter();
player.play("song.mp3");
```

---

## 图案选择指南

| 情况 | 模式 |
|-----------|---------|
| 对象创建很复杂 | 构建器，工厂 |
| 需要动态添加功能 | 装饰器 |
算法的多种实现方式 | 策略 |
| 对状态变化做出反应 | 观察者 |
| 与原有代码集成 | 适配器 |

## 应避免的反模式

反模式 | 问题 | 更佳解决方法 |
|--------------|---------|-----------------|
单例模式滥用 | 全局状态，难以测试 | 依赖注入 |
| 工厂无处不在 | 过度设计 | 如果类型已知，则简单地使用“新建” |
| 深层装饰器链 | 难以调试 | 支持组合，保持链短 |