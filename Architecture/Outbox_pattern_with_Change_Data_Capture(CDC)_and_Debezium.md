**Outbox Pattern** là **mảnh ghép quan trọng nhất** khi bạn làm **Event-Driven Architecture với Spring Boot + Kafka/RabbitMQ**.

# 1️⃣ Vấn đề Outbox Pattern giải quyết là gì?
❌ Cách làm sai (rất phổ biến)
```java
@Transactional
public void createOrder() {
    orderRepository.save(order);     // DB
    kafkaTemplate.send("order", evt); // Kafka
}
```

❌ Có thể xảy ra
- DB commit thành công ❌ Kafka gửi thất bại → **Mất event**
- Kafka gửi OK ❌ DB rollback → **Event ma**
- Crash giữa chừng → **Inconsistent state (Không nhất quán)**

➡️ **Không có Atomicty giữa DB & Message Broker**

# 2️⃣ Outbox Pattern là gì?
**Ghi event vào DB (Outbox table) trong CÙNG transaction với business data**, sau đó **một process khác** đọc Outbox và gửi sang message broker.

✔ DB là single source of truth
✔ Không mất event
✔ Phù hợp Microservices

# 3️⃣ Kiến trúc tổng thể
```css
┌────────────┐
│ Order API  │
└─────┬──────┘
      │
┌─────▼──────────────┐
│ Transaction        │
│  - orders table    │
│  - outbox table    │
└─────┬──────────────┘
      │
┌─────▼──────────────┐
│ Outbox Processor   │ (Scheduler / CDC)
│  - đọc outbox      │
│  - gửi Kafka       │
└─────┬──────────────┘
      │
┌─────▼──────────────┐
│ Kafka / RabbitMQ   │
└────────────────────┘
```

# 4️⃣ Khi nào bắt buộc dùng Outbox?
✅ Microservices
✅ Event Integration
✅ Kafka / RabbitMQ
✅ Không được mất event

❌ In-memory event
❌ Monolith nhỏ

# 5️⃣ Outbox Table (CỰC KỲ QUAN TRỌNG)
**Thiết kế chuẩn**
```sql
CREATE TABLE outbox_event (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(50),
    aggregate_id VARCHAR(50),
    event_type VARCHAR(100),
    payload JSONB,
    status VARCHAR(20),
    created_at TIMESTAMP
);
```
| Field          | Ý nghĩa                 |
| -------------- | ----------------------- |
| aggregate_type | Order                   |
| aggregate_id   | orderId                 |
| event_type     | OrderCreated            |
| payload        | JSON event              |
| status         | PENDING / SENT / FAILED |

# 6️⃣ Domain Event (DDD)
```java
public class OrderCreatedEvent {
    private Long orderId;
    private BigDecimal total;
}
```
👉 **Domain không biết Kafka, DB, JSON**

# 7️⃣ Outbox Entity
```java
@Entity
@Table(name = "outbox_event")
public class OutboxEvent {

    @Id
    private UUID id;

    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Column(columnDefinition = "jsonb")
    private String payload;

    private String status;
    private LocalDateTime createdAt;
}
```

# 8️⃣ Application Service (CORE)
```java
@Transactional
public void createOrder(BigDecimal total) {

    Order order = Order.create(total);
    orderRepository.save(order);

    OrderCreatedEvent event =
        new OrderCreatedEvent(order.getId(), total);

    OutboxEvent outbox = new OutboxEvent(
        UUID.randomUUID(),
        "Order",
        order.getId().toString(),
        "OrderCreated",
        objectMapper.writeValueAsString(event),
        "PENDING",
        LocalDateTime.now()
    );

    outboxRepository.save(outbox);
}
```
🔥 **Business data + Outbox cùng transaction**

# 9️⃣ Outbox Processor (Scheduler)
```java
@Scheduled(fixedDelay = 5000)
@Transactional
public void processOutbox() {

    List<OutboxEvent> events =
        outboxRepository.findByStatus("PENDING");

    for (OutboxEvent event : events) {
        try {
            kafkaTemplate.send(
                "order.created",
                event.getAggregateId(),
                event.getPayload()
            );
            event.setStatus("SENT");
        } catch (Exception e) {
            event.setStatus("FAILED");
        }
    }
}
```
📌 Có thể retry

# 🔟 Đảm bảo Idempotency (BẮT BUỘC)
**Consumer phải xử lý trùng event**
```java
@KafkaListener(topics = "order.created")
public void consume(String payload) {

    if (processedEventRepo.exists(eventId)) {
        return;
    }

    // xử lý
    processedEventRepo.save(eventId);
}
```

# 1️⃣1️⃣ Transaction + Outbox Flow (TỪNG BƯỚC)
1️⃣ Client gọi API
2️⃣ DB transaction mở
3️⃣ Lưu Order
4️⃣ Lưu Outbox Event
5️⃣ Commit DB
6️⃣ Scheduler đọc Outbox
7️⃣ Gửi Kafka
8️⃣ Consumer xử lý

➡️ **Eventual Consistency**

# 1️⃣2️⃣ 2 Cách triển khai Outbox
1️⃣ Polling (Spring Scheduler)

✔ Dễ làm
❌ Delay
❌ Load DB

2️⃣ CDC (Debezium) – PRO

✔ Real-time
✔ Scale tốt
❌ Setup phức tạp

```bash
Postgres → Debezium → Kafka
```

# 1️⃣3️⃣ Outbox + Clean Architecture
```bash
domain
 └─ event

application
 └─ service

infrastructure
 ├─ persistence
 └─ messaging
```
📌 **Outbox nằm infrastructure**

# 1️⃣4️⃣ Những lỗi CHẾT NGƯỜI ❌
❌ Publish Kafka trong transaction
❌ Không retry
❌ Không idempotent
❌ Event chứa entity JPA
❌ Không cleanup outbox

# 1️⃣5️⃣ Best Practices

✅ Event bất biến (immutable)
✅ JSON schema versioning
✅ Retry + DLQ
✅ Batch sending
✅ Cleanup job (xóa SENT)

# 1️⃣6️⃣ So sánh nhanh
| Cách                       | An toàn         |
| -------------------------- | --------------- |
| DB + Kafka trực tiếp       | ❌               |
| TransactionalEventListener | ❌ (distributed) |
| **Outbox Pattern**         | ✅               |

# 1️⃣7️⃣ Khi kết hợp với Saga / CQRS
- Outbox = **Event transport an toàn**
- Saga = **Orchestration / Choreography**
- CQRS = **Read model async**