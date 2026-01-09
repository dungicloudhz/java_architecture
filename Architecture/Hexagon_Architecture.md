Mô hình **Hexagonal Architecture (HA)** (còn gọi là **Port and Adapters**) là một kiến trúc phần mềm giúp tách biệt **core business logic (domain)** khỏi **các yếu tố hạ tầng (infreastructure)** như database, API, UI, messaging system.

# 1. Ý tưởng cốt lõi
- Ứng dụng được chia thành **vòng tròn lõi** (domain + application service) và **các adapter** ở xung quanh.
- Giao tiếp qua **Port (interfaces)**.
- Các **Adapters** (database, REST API, Kafka, UI, ...) sẽ **cắm** vào các port này.
> Nhờ đó domain không phụ thuộc vào database, framework hay giao tiếp bên ngoài.

# 2. So đồi trực quan (Bằng text)
```bash
                            +----------------+
                            |   REST API     |  <-- Adapter Inbound
                            +----------------+
                                    |
                                (Input Port)
                                    v
    +-------------------------------------------------------------+
    |                      Application Core                      |
    |                                                             |
    |   +----------------+     +----------------+                 |
    |   |  Domain Model  |<--->| ApplicationSvc |                 |
    |   +----------------+     +----------------+                 |
    |          ^                          |                       |
    |          | (Domain logic)           | (Output Port)         |
    |   +----------------+     +----------------+                 |
    |   |  Repository    |<----|   Interface    |                 |
    |   +----------------+     +----------------+                 |
    |                                                             |
    +-------------------------------------------------------------+
                                    ^
                                (Output Port)
                                    |
                        +--------------------------+
                        |   Database / Kafka / ... |
                        +--------------------------+
                            Adapter Outbound
```

# 3. Thành phần chính
## 1. Domain (Core)
- Chứa **entity, value object, domain service**.
- Không phụ thuộc framework (Spring, JPA).
## 2. Application (Use Cases)
- Xử lý luồng nghiệp vụ.
- Định nghĩa **Port (interfaces)** để nói `tôi cần lưu order` hoặc `tôi cần gửi message`.
## 3. Ports
- Input Port: interface cho **service** gọi từ bên ngoài (ví dụ: `OrderUseCase`).
- Output Port: interface cho core gọi ra hạ tầng (ví dụ: `OrderRepositoryPort`).
## 4. Adapters
- Inbound: Controller (REST, GraphQL, CLI).
- Outbound: Repository (JPA, JDBC), Kafka, Email service.

👉 Tóm gọn:
Luồng = User → REST Adapter → Application Use Case → Domain → Repository Adapter (DB) + Domain Event → Publisher Adapter (Kafka).
# 4. Code minh họa
## 1. Domain layer
```java
    // domain/valueobject/BaseId.java
    public abstract class BaseId<T> {
        private final T value;
        protected BaseId(T value) { this.value = java.util.Objects.requireNonNull(value); }
        public T getValue() { return value; }
        @Override public String toString() { return value.toString(); }
    }
    // domain/valueobject/OrderId.java
    public final class OrderId extends BaseId<java.util.UUID> {
        public OrderId(java.util.UUID value) { super(value); }
        public static OrderId newId() { return new OrderId(java.util.UUID.randomUUID()); }
    }

    // domain/valueobject/Money.java
    import java.math.*;

    public final class Money {
        private final BigDecimal amount;
        public Money(BigDecimal amount) {
            this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        }
        public BigDecimal amount() { return amount; }
        public boolean isPositive() { return amount.compareTo(BigDecimal.ZERO) > 0; }
        @Override public String toString() { return amount.toPlainString(); }
    }

    // domain/entity/AggregateRoot.java
    import java.util.*;

    public abstract class AggregateRoot<ID extends BaseId<?>> {
        protected ID id;
        private final List<DomainEvent> events = new ArrayList<>();

        protected void registerEvent(DomainEvent e) { events.add(e); }
        public List<DomainEvent> pullEvents() {
            List<DomainEvent> copy = List.copyOf(events);
            events.clear();
            return copy;
        }
        public ID getId() { return id; }
    }

    // domain/event/DomainEvent.java
    import java.time.Instant;

    public interface DomainEvent {
        String aggregateId();
        Instant occurredOn();
    }

    // domain/event/OrderCreatedEvent.java
    import java.time.Instant;

    public record OrderCreatedEvent(
            String aggregateId,
            String customerId,
            Money price,
            Instant occurredOn
    ) implements DomainEvent {
        public OrderCreatedEvent(String aggregateId, String customerId, Money price) {
            this(aggregateId, customerId, price, Instant.now());
        }
    }

    // domain/entity/Order.java
    public class Order extends AggregateRoot<OrderId> {
        private String customerId;
        private Money price;

        private Order() {}

        public static Order place(String customerId, Money price) {
            if (!price.isPositive()) throw new RuntimeException("Price must be positive");

            Order o = new Order();
            o.id = OrderId.newId();
            o.customerId = customerId;
            o.price = price;
            o.registerEvent(new OrderCreatedEvent(o.id.toString(), customerId, price));
            return o;
        }

        public String getCustomerId() { return customerId; }
        public Money getPrice() { return price; }
    }

```

## 2. Application layer
```java
    // application/port/in/CreateOrderUseCase.java
    public interface CreateOrderUseCase {
        OrderId handle(CreateOrderCommand cmd);
    }

    // application/dto/CreateOrderCommand.java
    public record CreateOrderCommand(String customerId, BigDecimal price) {}

    // application/port/out/OrderRepositoryPort.java
    public interface OrderRepositoryPort {
        Order save(Order order);
    }

    // application/port/out/DomainEventPublisher.java
    import java.util.Collection;

    public interface DomainEventPublisher {
        void publish(Collection<? extends DomainEvent> events);
    }

    // application/service/CreateOrderService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class CreateOrderService implements CreateOrderUseCase {
        private final OrderRepositoryPort repository;
        private final DomainEventPublisher publisher;

        public CreateOrderService(OrderRepositoryPort repository, DomainEventPublisher publisher) {
            this.repository = repository;
            this.publisher = publisher;
        }

        @Override @Transactional
        public OrderId handle(CreateOrderCommand cmd) {
            Order order = Order.place(cmd.customerId(), new Money(cmd.price()));
            repository.save(order);
            publisher.publish(order.pullEvents());
            return order.getId();
        }
    }

```
## 3. Adapters
### Inbound (REST)
```java
    // adapter/in/web/OrderController.java
    import org.springframework.http.ResponseEntity;
    import org.springframework.web.bind.annotation.*;

    @RestController
    @RequestMapping("/orders")
    public class OrderController {
        private final CreateOrderUseCase useCase;

        public OrderController(CreateOrderUseCase useCase) { this.useCase = useCase; }

        @PostMapping
        public ResponseEntity<String> create(@RequestBody OrderRequest req) {
            OrderId id = useCase.handle(new CreateOrderCommand(req.customerId(), req.price()));
            return ResponseEntity.ok(id.toString());
        }
    }

    record OrderRequest(String customerId, java.math.BigDecimal price) {}

```

### Outbound (Persistence - JPA)
```java
    // adapter/out/persistence/OrderJpaEntity.java
    import jakarta.persistence.*;
    import java.math.BigDecimal;

    @Entity @Table(name="orders")
    class OrderJpaEntity {
        @Id String id;
        String customerId;
        BigDecimal price;
        // getters/setters
    }

    // adapter/out/persistence/OrderRepositoryAdapter.java
    import org.springframework.stereotype.Repository;

    @Repository
    class OrderRepositoryAdapter implements OrderRepositoryPort {
        private final SpringDataOrderRepository repo;
        OrderRepositoryAdapter(SpringDataOrderRepository repo) { this.repo = repo; }

        @Override
        public Order save(Order domain) {
            OrderJpaEntity e = new OrderJpaEntity();
            e.setId(domain.getId().toString());
            e.setCustomerId(domain.getCustomerId());
            e.setPrice(domain.getPrice().amount());
            repo.save(e);
            return domain;
        }
    }

    interface SpringDataOrderRepository extends org.springframework.data.jpa.repository.JpaRepository<OrderJpaEntity, String> {}

```
### Outbound (Persistence - JPA)
```java
    // adapter/out/event/LoggingDomainEventPublisher.java
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.stereotype.Component;
    import java.util.Collection;

    @Component
    class LoggingDomainEventPublisher implements DomainEventPublisher {
        private static final Logger log = LoggerFactory.getLogger(LoggingDomainEventPublisher.class);

        @Override
        public void publish(Collection<? extends DomainEvent> events) {
            events.forEach(e -> log.info("Publishing domain event: {}", e));
            // Thực tế: gửi Kafka, RabbitMQ, hoặc ghi Outbox
        }
    }

```

# 5. Sơ đồ luồng
## 1. Cấu trúc class (Hexagonal view)
```java
                    +--------------------+
                    |   OrderController  |
                    | (Inbound Adapter)  |
                    +---------+----------+
                            |
                            v
                    +--------------------+
                    |  CreateOrderUseCase|<-------------------+
                    | (Input Port)       |                    |
                    +---------+----------+                    |
                            |                                 |
                            v                                 |
                    +--------------------+                    |
                    | CreateOrderService |                    |
                    | (Application Layer)|                    |
                    +----+-----------+---+                    |
                          |           |                       |
            (save order)  |           | (publish events)      |
                          v           v                       |
         +-------------------+   +----------------------+     |
         |OrderRepositoryPort|   |DomainEventPublisher  |     |
         |(Output Port)      |   |(Output Port)         |     |
         +---------+---------+   +----------+-----------+     |
                   |                        |                 |
                   v                        v                 |
     +--------------------------+  +----------------------+   |
     |OrderRepositoryAdapter    |  |LoggingDomainEvent... |   |
     |(Outbound Adapter - DB)   |  |(Outbound Adapter -   |   |
     +-----------+--------------+  | Event Bus / Log)     |   |
                 |                 +----------------------+   |
                 v                                            |
    +--------------------------+                              |
    |SpringDataOrderRepository |                              |
    |(JPA Repository)          |                              |
    +--------------------------+                              |
                                                              |
    +---------------------------------------------------------+
    |                        Domain                          |
    |                                                        |
    |   +-----------+        +--------------------+          |
    |   | OrderId   |        | Order              |          |
    |   +-----------+        | (Aggregate Root)   |          |
    |                        | - OrderId id       |          |
    |   +-----------+        | - Money price      |          |
    |   | Money     |        | - CustomerId       |          |
    |   +-----------+        |                    |          |
    |                        | place() -> event   |          |
    |   +-----------+        +--------------------+          |
    |   | DomainEvent|---<   | OrderCreatedEvent  |          |
    |   +-----------+        +--------------------+          |
    +---------------------------------------------------------+

```
## 2. Luồng xử lý `POST /orders`
```
    User -> OrderController -> CreateOrderService -> Order.place()
                                        |               |
                                        |               v
                                        |        tạo OrderCreatedEvent
                                        |
                                        v
                            OrderRepositoryAdapter.save(order)
                                        |
                                        v
                            SpringDataOrderRepository.save()

    Sau khi lưu xong:
    CreateOrderService -> LoggingDomainEventPublisher.publish(events)

    Trả về OrderId -> OrderController -> User
```
# 6. Giải thích
## 1. Domain Layer (trái tim của hệ thống)
**`BaseId<T>`**
- **Vai trò**: lớp cha cho tất cả các ID (OrderId, CustomerId, …).
- **Ý nghĩa**: thay vì dùng **`String`**/**`UUID`** trần, ta gói nó lại để type-safe (tránh nhầm lẫn giữa OrderId và CustomerId).
- **Liên hệ**: **`OrderId`** kế thừa từ đây.
___
**`OrderId`**
- **Vai trò**: đại diện cho khóa chính của **`Order`**.
- **Ý nghĩa**: sinh ID mới bằng **`UUID.randomUUID()`**.
- **Liên hệ**: **`Order`** giữ một **`OrderId`**.
___
**`Money`**
- **`Vai trò`**: Value Object cho tiền tệ.
- **`Ý nghĩa`**: luôn chuẩn hóa số tiền với scale = 2, có method check dương/so sánh.
- **`Liên hệ`**: **`Order`** dùng **`Money`** để lưu giá trị đơn hàng.
___
**`AggregateRoot<ID>`**
- **`Vai trò`**: lớp cơ sở cho mọi Aggregate Root.
- **`Ý nghĩa`**: giữ danh sách **`DomainEvent`** được phát sinh, cho phép **`registerEvent()`** và **`pullEvents()`**.
- **`Liên hệ`**: **`Order`** kế thừa từ đây.
___
**`DomainEvent`**
- **`Vai trò`**: interface chung cho mọi event trong domain.
- **`Ý nghĩa`**: bắt buộc mỗi event phải có **`aggregateId()`** và **`occurredOn()`**.
- **`Liên hệ`**: **`OrderCreatedEvent`** implements interface này.
___
**`OrderCreatedEvent`**
- **`Vai trò`**: sự kiện miền khi một đơn hàng mới được tạo.
- **`Ý nghĩa`**: chứa **`aggregateId`** (OrderId), **`customerId`**, **`price`**, và thời điểm xảy ra.
- **`Liên hệ`**: **`Order`** sẽ **`registerEvent(new OrderCreatedEvent(...))`** khi tạo thành công.
___
**`Order`**
- **`Vai trò`**: Aggregate Root của bounded context “Ordering”.
- **`Ý nghĩa`**:
    - Có **`OrderId`**, **`customerId`**, **`Money price`**.
    - Có factory method **`place(customerId, price)`**: tạo đơn hàng mới, validate (giá > 0).
    - Khi hợp lệ → tạo **`OrderCreatedEvent`**.
- **`Liên hệ`**:
    - Kế thừa **`AggregateRoot<OrderId>`**.
    - Sinh event **`OrderCreatedEvent`**.
    - Được dùng trong Application Service.
___
## 2. Application Layer (luồng nghiệp vụ)
**`CreateOrderCommand`**
- **`Vai trò`**: DTO cho input của use case “Create Order”.
- **`Ý nghĩa`**: gom dữ liệu từ bên ngoài (**`customerId`**, **`price`**) để service xử lý.
- **`Liên hệ`**: Controller tạo command này → truyền cho **`CreateOrderService`**.
___
**`CreateOrderUseCase`**
- **`Vai trò`**: input port (interface).
- **`Ý nghĩa`**: định nghĩa hợp đồng: tạo order thì phải nhận **`CreateOrderCommand`** và trả **`OrderId`**.
- **`Liên hệ`**: **`CreateOrderService`** implement interface này.
___
**`OrderRepositoryPort`**
- **`Vai trò`**: output port (interface).
- **`Ý nghĩa`**: domain cần lưu order nhưng không quan tâm DB. Port này chính là “ổ cắm”.
- **`Liên hệ`**: **`OrderRepositoryAdapter`** (adapter JPA) hiện thực interface này.
___
**`DomainEventPublisher`**
- **`Vai trò`**: output port.
- **`Ý nghĩa`**: domain muốn phát sự kiện nhưng không quan tâm Kafka hay RabbitMQ. Chỉ biết mình có “publisher”.
- **`Liên hệ`**: **`LoggingDomainEventPublisher`** hiện thực interface này.
___
**`CreateOrderService`**
- **`Vai trò`**: Use Case Service (ứng dụng).
- **`Logic`**:
    **1.** Nhận **`CreateOrderCommand`**.
    **2.** Gọi **`Order.place(...)`** để tạo order (domain xử lý rule).
    **3.** Gọi **`repository.save(order)`** để lưu vào DB.
    **4.** Gọi **`publisher.publish(order.pullEvents())`** để phát event.
    **5.** Trả về **`OrderId`**.
- **`Liên hệ`**:
    - Input port: **`CreateOrderUseCase`**.
    - Output ports: **`OrderRepositoryPort`**, **`DomainEventPublisher`**.

## 3. Adapters Layer
### Inbound (REST API)
**`OrderController`**
- **`Vai trò`**: REST Controller, đầu vào của hệ thống.
- **`Logic`**:
    - Nhận JSON request **`{customerId, price}`**.
    - Tạo **`CreateOrderCommand`**.
    - Gọi **`useCase.handle(command)`**.
    - Trả về **`OrderId`**.
- **`Liên hệ`**: gọi vào **`CreateOrderService`**.
### Outbound (Persistence - JPA)
**`OrderJpaEntity`**
- **`Vai trò`**: entity ánh xạ DB.
- **`Ý nghĩa`**: JPA cần annotation (**`@Entity`**, **`@Table`**).
- **`Liên hệ`**: **`OrderRepositoryAdapter`** map **`Order`** → **`OrderJpaEntity`**.
**`OrderRepositoryAdapter`**
- **`Vai trò`**: hiện thực **`OrderRepositoryPort`**.
- **`Logic`**:
    - Convert domain **`Order`** sang **`OrderJpaEntity`**.
    - Lưu qua **`SpringDataOrderRepository`**.
    - Trả lại domain **`Order`**.
- **`Liên hệ`**: gọi bởi **`CreateOrderService`**.
**`SpringDataOrderRepository`**
- **`Vai trò`**: extends **`JpaRepository`**.
- **`Ý nghĩa`**: cho phép CRUD với DB.
- **`Liên hệ`**: được **`OrderRepositoryAdapter`** sử dụng.
### Outbound (Event Publisher)
**`LoggingDomainEventPublisher`**
- **`Vai trò`**: hiện thực **`DomainEventPublisher`**.
- **`Logic`**:
    - Nhận danh sách event từ domain.
    - Log ra console (demo).
    - (Thực tế: gửi Kafka, RabbitMQ, Outbox…).
- **`Liên hệ`**: được **`CreateOrderService`** gọi sau khi lưu DB.

## 4. Luồng tổng thể
1. **`User`** gọi API **`POST /orders`**.
2. **`OrderController`** nhận → tạo **`CreateOrderCommand`** → gọi **`CreateOrderService`**.
3. **`CreateOrderService`**:
    - Gọi **`Order.place()`** trong Domain.
    - Domain kiểm tra logic, tạo event **`OrderCreatedEvent`**.
    - Gọi **`OrderRepositoryAdapter.save()`** để lưu DB.
    - Gọi **`LoggingDomainEventPublisher.publish()`** để phát event.
    - Trả về **`OrderId`**.
4. **`OrderController`** trả response **`200 OK`** với **`orderId`**.