# 1️⃣ Bản chất: Clean vs Hexagonal
**Clean Architecture (Uncle Bob)**
👉 **Triết lý:**
- Business ở trung tâm
- Dependency chỉ hướng **vào trong**
- Tách **Entity - Use Case - Adapter - Framework**

**Hexagonal Architecture (Ports & Adapters)**
👉 **Cách kết nối:**
- Business giao tiếp với thế giới ngoài qua **Ports**
- Thế giới ngoài kết nối vào qua **Adapter**
- Không quan tâm Web / DB / Queue là gì

📌 **Clean = “WHAT”**
📌 **Hexagonal = “HOW”**

➡️ Kết hợp = **kiến trúc cực kỳ mạnh**

# 2️⃣ Sơ đồ Clean + Hexagonal

```css
            ┌──────────────┐
            │   Web API    │  ← REST, GraphQL
            └──────▲───────┘
                   │
        ┌──────────┴──────────┐
        │   Inbound Adapter   │  ← Controller
        └──────────▲──────────┘
                   │ (Input Port)
        ┌──────────┴──────────┐
        │     Use Case        │  ← Application Service
        └──────────▲──────────┘
                   │ (Output Port)
        ┌──────────┴──────────┐
        │  Outbound Adapter   │  ← DB, Kafka, REST
        └──────────▲──────────┘
                   │
            ┌──────┴─────────┐
            │ Infrastructure │
            └────────────────┘
```

# 3️⃣ Mapping các khái niệm (CỰC QUAN TRỌNG)
| Clean Architecture | Hexagonal           |
| ------------------ | ------------------- |
| Use Case           | Application Service |
| Interface          | Port                |
| Adapter            | Adapter             |
| Framework          | Infrastructure      |
| Controller         | Inbound Adapter     |
| Repository Impl    | Outbound Adapter    |

# 4️⃣ Cấu trúc thư mục CHUẨN (Java)
```bash
src/main/java/com.example.order
├── domain
│   └── model
│       └── Order.java
│
├── application
│   ├── port
│   │   ├── in
│   │   │   └── CreateOrderUseCase.java
│   │   └── out
│   │       └── OrderRepositoryPort.java
│   └── service
│       └── CreateOrderService.java
│
├── adapter
│   ├── in
│   │   └── web
│   │       └── OrderController.java
│   └── out
│       └── persistence
│           ├── OrderJpaEntity.java
│           ├── SpringDataOrderRepository.java
│           └── OrderRepositoryAdapter.java
│
└── config
    └── BeanConfig.java
```
📌 **Tên folder phản ánh đúng vai trò**

# 5️⃣ Domain (Business Core)
```java
public class Order {
    private final Long id;
    private final BigDecimal total;

    public Order(Long id, BigDecimal total) {
        if (total.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Total must be > 0");
        }
        this.id = id;
        this.total = total;
    }

    public Long getId() { return id; }
    public BigDecimal getTotal() { return total; }
}
```
✔ Không Spring
✔ Không JPA

# 6️⃣ Input Port (Use Case Interface)
```java
public interface CreateOrderUseCase {
    Order create(BigDecimal total);
}
```
👉 **Inbound port**

# 7️⃣ Output Port (Repository Port)
```java
public interface OrderRepositoryPort {
    Order save(Order order);
}
```
👉 Outbound Port

# 8️⃣ Application Service (Use Case Impl)
```java
public class CreateOrderService implements CreateOrderUseCase {

    private final OrderRepositoryPort orderRepository;

    public CreateOrderService(OrderRepositoryPort orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public Order create(BigDecimal total) {
        Order order = new Order(null, total);
        return orderRepository.save(order);
    }
}
```
📌 **Service chỉ biết PORT**

# 9️⃣ Inbound Adapter – Controller
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;

    public OrderController(CreateOrderUseCase createOrderUseCase) {
        this.createOrderUseCase = createOrderUseCase;
    }

    @PostMapping
    public OrderResponse create(@RequestBody CreateOrderRequest req) {
        Order order = createOrderUseCase.create(req.total());
        return new OrderResponse(order.getId(), order.getTotal());
    }
}
```
# 🔟 Outbound Adapter – Persistence
**JPA Entity**
```java
@Entity
@Table(name = "orders")
public class OrderJpaEntity {
    @Id
    @GeneratedValue
    private Long id;
    private BigDecimal total;
}
```

**Spring Data Repository**
```java
public interface SpringDataOrderRepository
        extends JpaRepository<OrderJpaEntity, Long> {
}
```

**Adapter Implementation**
```java
@Component
public class OrderRepositoryAdapter implements OrderRepositoryPort {

    private final SpringDataOrderRepository jpaRepo;

    public OrderRepositoryAdapter(SpringDataOrderRepository jpaRepo) {
        this.jpaRepo = jpaRepo;
    }

    @Override
    public Order save(Order order) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setTotal(order.getTotal());

        OrderJpaEntity saved = jpaRepo.save(entity);
        return new Order(saved.getId(), saved.getTotal());
    }
}
```
# 1️⃣1️⃣ Dependency Injection
```java
@Configuration
public class BeanConfig {

    @Bean
    CreateOrderUseCase createOrderUseCase(OrderRepositoryPort repo) {
        return new CreateOrderService(repo);
    }
}
```
# 1️⃣2️⃣ Vì sao kết hợp Clean + Hexagonal là BEST PRACTICE?
✅ Business độc lập hoàn toàn
✅ Thay DB / API / Message Broker không ảnh hưởng core
✅ Test Use Case không cần Spring
✅ Rất hợp Microservices + CQRS
✅ Đúng tư duy kiến trúc hiện đại

# 1️⃣3️⃣ Test Use Case (Không Spring)
```java
class FakeOrderRepository implements OrderRepositoryPort {
    @Override
    public Order save(Order order) {
        return new Order(1L, order.getTotal());
    }
}

@Test
void createOrder_success() {
    CreateOrderUseCase useCase =
        new CreateOrderService(new FakeOrderRepository());

    Order order = useCase.create(BigDecimal.TEN);

    assertEquals(BigDecimal.TEN, order.getTotal());
}
```

# 1️⃣4️⃣ So sánh nhanh
| Kiến trúc   | Mạnh ở điểm                |
| ----------- | -------------------------- |
| Layered     | CRUD nhỏ                   |
| Clean       | Business logic             |
| Hexagonal   | Giao tiếp                  |
| Clean + Hex | **Hệ thống lớn, bền vững** |

# 🚀 Hướng tiếp theo (rất đáng học)

Bạn muốn tôi đi tiếp theo hướng nào?

1️⃣ Clean + Hex + Spring Boot project hoàn chỉnh
2️⃣ Clean + Hex + JPA/Hibernate nâng cao
3️⃣ Clean + Hex + Microservices (REST + Kafka)
4️⃣ Clean + Hex + CQRS
5️⃣ So sánh Clean + Hex vs DDD