# 1. Mục tiêu của Clean Architecture
Clean Architecture (Robert C.Martin - Uncle Bob) được sinh ra để giải quyết vấn đề **phụ thuộc** trong hệ thống lớn:
- **Business logic (Domain)** không phụ thuộc vào **framework** (Spring Boot, Hibernate, Kafka...).
- **Dễ test** (test business thuần mà không cần database, server thật).
- **Dễ mở rộng, thay đổi**: đổi framework, đổi database, thêm adapter... không ảnh hưởng domain.
> Nguyên tắc: **Dependencies always point inward** - phụ thuộc luôn hướng vào **Core Domain**.

# 2. Các vòng tròn trong Clean Architecture
Hình dung có 4 vòng tròn (từ trong ra ngoài):
```pgsql
    +---------------------------+
    |       Frameworks          |
    | (Spring Boot, Hibernate)  |
    +---------------------------+
    |        Adapters           |
    | (RestController, RepoImpl)|
    +---------------------------+
    |   Application Services    |
    | (Use Cases, Orchestrator) |
    +---------------------------+
    |        Domain             |
    | (Entities, Business Rule) |
    +---------------------------+
```
- **Domain (Entities)**: Nơi chứa business logic thuần (class, entity, value object). Không biết gì về DB, REST hay framework.
- **Application (Use Cases)**: Chứa logic ứng dụng (dịch vụ gọi domain, xử lý flow). Biết domain, nhưng không biết adapter nào (DB, REST...).
- **Adapters (Infrastructure + Interface)**: Triển khai chi tiết (ví dụ: Repository implement bằng JPA, REST, Controller, Kafka consumer).
- **Frameworks**: Spring Boot, Hibernate, Kafka lib... chỉ là "plugin" cung cấp công cụ.

# 3. Cấu trúc thư mục Spring Boot theo Clean Architecture
Ví dụ: `order-service`
```bash
    order-service/
    └── src/main/java/com/example/order
        ├── domain/               # Core business
        │    ├── model/           # Entities, Value Objects
        │    └── service/         # Business rules
        │
        ├── application/          # Use Cases
        │    ├── port/            # Interface (in/out ports)
        │    └── service/         # Use case implementation
        │
        ├── infrastructure/       # Adapter implement
        │    ├── persistence/     # JPA/Hibernate repo impl
        │    ├── web/             # REST Controller
        │    └── config/          # Spring configuration
        │
        └── OrderServiceApplication.java  # Entry point
```
