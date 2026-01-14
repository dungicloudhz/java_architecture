# 1️⃣ Event-Driven Architecture là gì?
**Event-Driven Architecture (EDA)** là kiến trúc trong đó: `Hệ thống phản ứng lại các "sự kiện" (event) đã xảy ra trong quá khứ`

**Event là gì?**
Event = **một sự thật đã xảy ra**
```bash
OrderCreated
PaymentCompleted
UserRegistered
```
👉 Event **KHÔNG phải lệnh**
❌ CreateOrderCommand
✅ OrderCreatedEvent

**Ví dụ đời thực**
- Bạn đặt hàng → **OrderCreated**
- Hệ thống thanh toán nghe → trừ tiền
- Hệ thống email nghe → gửi mail
- Hệ thống kho nghe → trừ tồn

⛔ Không hệ thống nào gọi trực tiếp hệ thống nào

# 2️⃣ Vì sao dùng Event-Driven?
| Vấn đề            | Giải pháp EDA                |
| ----------------- | ---------------------------- |
| Coupling chặt     | Producer không biết consumer |
| Khó mở rộng       | Thêm consumer không sửa code |
| Khó scale         | Mỗi consumer scale riêng     |
| Business phức tạp | Tách side-effects            |

# 3️⃣ Event-Driven vs REST truyền thống
❌ **REST Coupling**
```bash
OrderService
 ├─ gọi PaymentService
 ├─ gọi EmailService
 └─ gọi InventoryService
```
➡️ **Tightly coupled (Liên kết chặt chẽ) – domino failure**

✅ **Event-Driven**
```bash
OrderService
 └─ publish OrderCreatedEvent

PaymentService  ┐
EmailService    ├─ subscribe OrderCreatedEvent
InventoryService┘
```
➡️ **Loose coupling (Khớp nối lỏng lẻo)**

# 4️⃣ Các thành phần chính của EDA
```bash
Event Producer  →  Event Broker  →  Event Consumer
```
| Thành phần | Ví dụ                       |
| ---------- | --------------------------- |
| Producer   | OrderService                |
| Broker     | Kafka / RabbitMQ / AWS SNS  |
| Consumer   | Payment / Email / Inventory |

# 5️⃣ Các kiểu Event trong Spring Boot
## 1. Domain Event (DDD)
- Xảy ra **trong domain**
- Gắn với **Aggregate Root**
- Không phụ thuộc framework

```java
public class OrderCreatedEvent {
    private final Long orderId;
    private final BigDecimal total;
}
// thể hiện đầy đủ code ở phía dưới
```

## 2. Application Event (Spring)
- Dùng Spring `ApplicationEventPublisher`
- In-process (KHÔNG qua message broker)

```java
publisher.publishEvent(new OrderCreatedEvent(...));
```

## 3. Integration Event
- Gửi qua Kafka/ RabitMQ
- Dùng cho microservice

# 6️⃣ Kiến trúc chuẩn (DDD + Clean + EDA)
```bash
domain
 ├─ model
 ├─ event
 └─ repository

application
 ├─ service
 ├─ command
 └─ event-handler

infrastructure
 ├─ messaging (Kafka/Rabbit)
 └─ persistence
```
📌 **Không phụ thuộc vào Spring**

# 7️⃣ Ví dụ CHUẨN: Order → Event
## 7.1 Domain Event
```java
// domain/event/OrderCreatedEvent.java
public class OrderCreatedEvent {
    private final Long orderId;
    private final BigDecimal total;

    public OrderCreatedEvent(Long orderId, BigDecimal total) {
        this.orderId = orderId;
        this.total = total;
    }

    public Long getOrderId() { return orderId; }
    public BigDecimal getTotal() { return total; }
}
```

## 7.2 Aggregate Root phát event
```java
// domain/model/Order.java
public class Order {

    private Long id;
    private BigDecimal total;
    private final List<Object> domainEvents = new ArrayList<>();

    public static Order create(BigDecimal total) {
        Order order = new Order();
        order.total = total;
        order.domainEvents.add(
            new OrderCreatedEvent(order.id, total)
        );
        return order;
    }

    public List<Object> getDomainEvents() {
        return domainEvents;
    }
}
```
## 7.3 Application Service
```java
@Service
public class OrderService {

    private final OrderRepository repository;
    private final ApplicationEventPublisher publisher;

    public OrderService(OrderRepository repository,
                        ApplicationEventPublisher publisher) {
        this.repository = repository;
        this.publisher = publisher;
    }

    @Transactional
    public void createOrder(BigDecimal total) {
        Order order = Order.create(total);
        repository.save(order);

        order.getDomainEvents()
             .forEach(publisher::publishEvent);
    }
}
```
📌 **Transaction Boundary rất quan trọng**

# 8️⃣ Event Listener (Consumer)
```java
@Component
public class PaymentListener {

    @EventListener
    public void handle(OrderCreatedEvent event) {
        System.out.println(
            "Thanh toán cho order " + event.getOrderId()
        );
    }
}
```

## 9️⃣ Async Event (Quan trọng)
```java
@EnableAsync
@Configuration
public class AsyncConfig {}

@Async
@EventListener
public void handle(OrderCreatedEvent event) {
    // chạy thread riêng
}
```
📌 **Không block transaction**

# 🔟 Event với Kafka (Microservices)
**Producer**
```java
kafkaTemplate.send("order.created", event);
```

**Consumer**
```java
@KafkaListener(topics = "order.created")
public void consume(OrderCreatedEvent event) {}
```

# 1️⃣1️⃣ Transactional Event Listener (RẤT QUAN TRỌNG)
```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
public void handle(OrderCreatedEvent event) {
    // chỉ chạy khi DB commit thành công
}
```
👉 Tránh gửi event khi transaction rollback

# 1️⃣2️⃣ Các Pattern quan trọng
| Pattern              | Mục đích                   |
| -------------------- | -------------------------- |
| Domain Event         | Tách business logic        |
| Outbox Pattern       | Tránh mất event            |
| Saga                 | Xử lý transaction phân tán |
| CQRS                 | Tách read/write            |
| Eventual Consistency | Nhất quán cuối             |

# 1️⃣3️⃣ Sai lầm thường gặp ❌
❌ Dùng event thay cho mọi thứ
❌ Event mang quá nhiều data
❌ Business logic nằm trong listener
❌ Không xử lý retry / idempotency
❌ Gửi event trước commit DB

# 1️⃣4️⃣ Khi nào KHÔNG nên dùng EDA?
- CRUD đơn giản
- App nhỏ
- Không cần scale
- Team chưa quen async

# 1️⃣5️⃣ Lộ trình học khuyên dùng (cho bạn)
1️⃣ Spring ApplicationEvent
2️⃣ Domain Event + DDD
3️⃣ Async & Transactional Event
4️⃣ Kafka/RabbitMQ
5️⃣ Outbox Pattern
6️⃣ Saga + CQRS