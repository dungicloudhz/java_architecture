**DDD (Domain-Driven Design)** là **đỉnh cao tư duy thiết kế nghiệp vụ**, nhưng chỉ "phát huy sức mạnh" khi hiểu **đúng bản chất**, không học theo kiểu thuộc khái niệm.

Tôi sẽ hướng dẫn bạn **từ tư duy → khái niệm → code Java thực tế → sai lầm thường gặp**, theo chuẩn **DDD hiện đại + Spring Boot**.

# 1️⃣ DDD là gì?
**DDD không phải là framework, không phải kiến trúc code**.
DDD là **phương pháp thiết kế phần mềm xoay quanh nghiệp vụ**.

`"Code phải phản ánh đúng ngôn ngữ & luật nghiệp vụ" - Eric Evans`

DDD giải quyết **1 vấn đề duy nhất**: `Làm sao để code không phản bội Business`

# 2️⃣ Khi nào NÊN / KHÔNG NÊN dùng DDD

✅ NÊN dùng khi
- Nghiệp vụ phức tạp
- Nhiều rule, trạng thái, quy trình
- Team lớn (dev ↔ business)
- Hệ thống sống lâu dài

❌ KHÔNG NÊN dùng khi
- CRUD đơn giản
- MVP, admin tool
- App nhỏ, dealine gấp

👉 **Cho bài toán khó, không phải mọi bài toán**

# 3️⃣ 3 trụ cột của DDD
## 1️⃣ Ubiquitous Language (QUAN TRỌNG NHẤT)
- Dev & Business dùng **chung 1 ngôn ngữ**
- Tên class, method, biến = thuật ngữ nghiệp vụ

❌ `processOrder()`
✅ `confirmPayment()`

## 2️⃣ Bounded Context
👉 **Mỗi nghiệp vụ có một ngữ cảnh riêng**
| Context  | Ý nghĩa    |
| -------- | ---------- |
| Order    | Đơn hàng   |
| Payment  | Thanh toán |
| Shipping | Giao hàng  |

📌 **Cùng 1 từ, nghĩa khác nhau**

## 3️⃣ Model nghiệp vụ
- Entity
- Value Object
- Aggregate
- Domain Service
- Domain Event

## 4️⃣ Entity vs Value Object
**Entity**
- Có **identity**
- Thay đổi trạng thái

```java
public class Order {
    private OrderId id;
    private OrderStatus status;
}
```

**Value Object**
- **Không có Identity**
- **Immutable**
- So sánh bằng giá trị

```java
public record Money(BigDecimal value) {
    public Money {
        if (value.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException();
    }
}
```
📌 **80% bug nghiệp vụ do thiếu Value Object**

## 5️⃣ Aggregate & Aggregate Root (LINH HỒN DDD)
👉 **Aggregate = ranh giới consistency (tính nhất quán)**
```css
Order (Aggregate Root)
 ├── OrderItem
 └── ShippingAddress
```

**Aggregate Root**:
- Là **entry point duy nhất**
- Không cho truy cập thẳng entity con

```java
public class Order {
    private List<OrderItem> items;

    public void addItem(ProductId productId, int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException();
        items.add(new OrderItem(productId, quantity));
    }
}
```
❌ `orderItem.setQuantity()` từ bên ngoài

## 6️⃣ Domain Service (KHI NÀO CẦN)
👉 Khi logic:
- Không thuộc về 1 entity
- Dùng nhiều aggregate

```java
public class PricingService {
    public Money calculateTotal(List<OrderItem> items) {
        // logic nghiệp vụ
    }
}
```
📌 **Domain Service KHÔNG phụ thuộc framework**

## 7️⃣ Application Service (Use Case)
👉 **Điều phối**, KHÔNG chứa logic nghiệp vụ
```java
public class PlaceOrderService {

    private final OrderRepository orderRepository;

    public void place(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.customerId());
        order.addItem(cmd.productId(), cmd.quantity());

        orderRepository.save(order);
    }
}
```

## 8️⃣ Repository (DDD đúng nghĩa)
❌ **Repository không phải DAO**
```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

@Component
public class OrderRepositoryImpl implements OrderRepository {

    private final SpringDataOrderRepository jpaRepo;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = OrderMapper.toJpa(order);
        jpaRepo.save(entity);
    }
}

@Entity
@Table(name = "orders")
public class OrderJpaEntity {
    @Id
    @GeneratedValue
    private Long id;
    private BigDecimal total;
}
```
📌 Repository trả về **Aggregate**, không trả entity con

## 9️⃣ Domain Event (DDD hiện đại)
👉 Điều gì đó xảy ra trong Domain
```java
public record OrderPaidEvent(OrderId orderId) {}
```

**Phát event**
```java
public void pay() {
    status = OrderStatus.PAID;
    domainEvents.add(new OrderPaidEvent(id));
}
```
📌 Event = **quá khứ**, KHÔNG phải command

## 🔟 DDD Tactical vs Strategic
**Tactical DDD (Chiến thuật CODE)**
- Entity
- VO
- Aggregate
- Repository
- Domain Service
- Event

**Strategic DDD (KIẾN TRÚC)**
- Bounded Context
- Context Map
- Anti-Corruption Layer
- Shared Kernel

👉 **90% dev chỉ học Tactical → FAIL**

## 1️⃣1️⃣ DDD + Clean + Hex (BEST PRACTICE)
```pgsql
Domain (DDD)
 ├── Aggregate
 ├── VO
 ├── Domain Service
 ├── Domain Event
Application
 ├── Use Case
Ports
 ├── Repository
 ├── External Service
Adapters
 ├── REST
 ├── Persistence
Infrastructure
```
📌 **DDD sống chung với Domain Layer**

## 1️⃣2️⃣ Sai lầm kinh điển (RẤT HAY GẶP)
❌ Entity = JPA Entity
❌ Service chứa business logic
❌ Không có Value Object
❌ Aggregate quá lớn
❌ Transaction xuyên nhiều Aggregate
❌ “DDD cho CRUD”

## 1️⃣3️⃣ Checklist DDD đúng chuẩn
✔ Business rule nằm trong Domain
✔ Domain không phụ thuộc Spring
✔ Method đặt tên theo nghiệp vụ
✔ Có Aggregate Root
✔ Có Value Object
✔ Application Service mỏng
✔ Repository trả Aggregate