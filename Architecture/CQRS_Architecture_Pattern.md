**CQRS (Command Query Responsibility Segregation)** là pattern **rất mạnh**, nhưng cũng **dễ dùng sai nhất** nếu không hiểu đúng bản chất.

# 1️⃣ CQRS là gì?
**CQRS = Tách riêng:**
- **Command** → Thay đổi trạng thái (WRITE)
- **Query** → Chỉ đọc dữ liệu (READ)

`❌ Không dùng model cho cả read & write`
`✅ Mỗi bên có 1 model riêng`

**Ví dụ đơn giản**
| Hành động         | CQRS Side |
| ----------------- | --------- |
| Tạo đơn hàng      | Command   |
| Huỷ đơn hàng      | Command   |
| Xem danh sách đơn | Query     |
| Xem chi tiết đơn  | Query     |

# 2️⃣ Vấn đề CQRS giải quyết
**❌ CRUD truyền thống**
```bash
Controller
 └─ OrderService
     └─ OrderEntity (JPA)
```
- Entity phình to
- Logic đọc/ghi lẫn lộn
- Performance kém
- Khó scale

✅ **CQRS**
```bash
Write Side                Read Side
──────────               ──────────
Command API              Query API
Domain Model             Read Model (DTO)
Transaction              No Transaction
Strong consistency       Eventual consistency
```

# 3️⃣ CQRS ≠ Event Sourcing (RẤT QUAN TRỌNG)
| CQRS            | Event Sourcing    |
| --------------- | ----------------- |
| Tách read/write | Lưu state = event |
| Dùng DB thường  | Event store       |
| Dễ áp dụng      | Phức tạp          |
| Phổ biến        | Ít hơn            |

👉 **Có thể dùng CQRS mà KHÔNG cần Event Sourcing**

# 4️⃣ Kiến trúc tổng thể CQRS (Spring Boot)
```bash
┌───────────────┐
│ Client        │
└──────┬────────┘
       │
┌──────▼──────────────┐
│ API Layer           │
├─────────┬───────────┤
│ Command │ Query     │
└──────┬─┘   └─┬──────┘
       │       │
┌──────▼───┐ ┌─▼──────────┐
│ Write DB │ │ Read DB    │
└──────┬───┘ └────┬───────┘
       │          │
       └──Event───┘
```

# 5️⃣ Khi nào nên dùng CQRS?
✅ Domain phức tạp
✅ Read nhiều hơn write
✅ Performance cao
✅ Microservices
✅ Event-driven system

❌ CRUD đơn giản
❌ Team mới
❌ App nhỏ

# 6️⃣ Command Side (WRITE)
## 6.1 Command
```java
public class CreateOrderCommand {
    private BigDecimal total;
}
```
👉 Command = **ý định** (intent)
## 6.2 Command Handler
```java
@Service
public class CreateOrderHandler {

    @Transactional
    public void handle(CreateOrderCommand cmd) {
        Order order = Order.create(cmd.getTotal());
        orderRepository.save(order);
    }
}
```
📌 **Không trả dữ liệu**

## 6.3 Domain Model (WRITE)
```java
public class Order {
    private OrderId id;
    private Money total;

    public static Order create(Money total) {
        // validate business rule
        return new Order(total);
    }
}
```
# 7️⃣ Query Side (READ)
## 7.1 Read Model (DTO)
```java
public class OrderView {
    private Long orderId;
    private BigDecimal total;
    private String status;
}
```
📌 **Không phải Entity domain**
## 7.2 Query Handler
```java
@Repository
public class OrderQueryRepository {

    public List<OrderView> findAll() {
        return jdbcTemplate.query(
            "SELECT id, total, status FROM order_view",
            rowMapper
        );
    }
}
```
👉 **Read không chứa business logic**

# 8️⃣ Event kết nối Read & Write
```java
public class OrderCreatedEvent {
    private Long orderId;
    private BigDecimal total;
}
```

**Event Handler cập nhật Read DB**
```java
@Component
public class OrderProjection {

    @Transactional
    @EventListener
    public void handle(OrderCreatedEvent event) {

        jdbcTemplate.update(
            "INSERT INTO order_view VALUES (?, ?, ?)",
            event.getOrderId(),
            event.getTotal(),
            "CREATED"
        );
    }
}
```
📌 **Projection = build read model**

# 9️⃣ CQRS + Outbox (CHUẨN)
```bash
Command
 └─ Save Order + Outbox
        │
        ▼
Kafka / Broker
        │
        ▼
Projection Consumer
 └─ Update Read DB
```

✔ Không mất event
✔ Read model eventually consistent

# 🔟 Eventual Consistency (HIỂU RÕ)
- Write xong ❌ Read chưa thấy ngay
- Delay = vài ms → vài giây
- Chấp nhận được

📌 Phải giải thích cho business

# 1️⃣1️⃣ CQRS với 1 DB hay 2 DB?
| Kiểu           | Khi nào       |
| -------------- | ------------- |
| 1 DB, 2 schema | Monolith      |
| 2 DB           | Microservices |
| Read DB NoSQL  | Read nhiều    |

# 1️⃣2️⃣ CQRS + REST API
```bash
POST   /orders        → Command
DELETE /orders/{id}   → Command
GET    /orders        → Query
GET    /orders/{id}   → Query
```

# 1️⃣3️⃣ Những lỗi CHẾT NGƯỜI ❌
❌ Dùng CQRS cho CRUD đơn giản
❌ Command trả DTO
❌ Query dùng domain entity
❌ Expect strong consistency
❌ Không dùng event → read sync

# 1️⃣4️⃣ CQRS + Saga
- Command → Event
- Saga lắng nghe → gửi Command khác
- Không rollback
```bash
OrderCreated
  ├─ ReserveInventoryCommand
  └─ ProcessPaymentCommand
```
# 1️⃣5️⃣ Best Practices

✅ Command immutable
✅ One command → one aggregate
✅ Read model denormalized
✅ Projection idempotent
✅ Version event schema

# 1️⃣6️⃣ CQRS Flow (TỪNG BƯỚC)

1️⃣ Client → Command API
2️⃣ Validate
3️⃣ Save aggregate
4️⃣ Save outbox event
5️⃣ Commit
6️⃣ Event published
7️⃣ Projection update
8️⃣ Query API trả dữ liệu

# 1️⃣7️⃣ Khi KHÔNG nên dùng CQRS
- App admin CRUD
- MVP
- Team nhỏ
- Không async

# 1️⃣8️⃣ So sánh nhanh
| Pattern        | Mục đích             |
| -------------- | -------------------- |
| CQRS           | Scale read/write     |
| Event Sourcing | Audit, history       |
| Saga           | Transaction phân tán |
| Outbox         | Reliable event       |
