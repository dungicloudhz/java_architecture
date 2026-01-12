# 1. Mục tiêu của Clean Architecture
Clean Architecture (Robert C.Martin - Uncle Bob) được sinh ra để giải quyết vấn đề **phụ thuộc** trong hệ thống lớn:
- **Business logic (Domain)** không phụ thuộc vào **framework** (Spring Boot, Hibernate, Kafka...).
- **Dễ test** (test business thuần mà không cần database, server thật).
- **Dễ mở rộng, thay đổi**: đổi framework, đổi database, thêm adapter... không ảnh hưởng domain.
`Nguyên tắc: **Dependencies always point inward** - phụ thuộc luôn hướng vào **Core Domain**`.

# 2. Các vòng tròn trong Clean Architecture
Hình dung có 4 vòng tròn (từ trong ra ngoài):
```pgsql
    +---------------------------+
    |       Frameworks          |   ← Spring, Hibernate, DB, Web
    | (Spring Boot, Hibernate)  |
    +---------------------------+
    |        Adapters           |   ← DTO, Mapper, Repository Impl
    | (RestController, RepoImpl)|
    +---------------------------+
    |   Application Services    |   ← Application logic
    | (Use Cases, Orchestrator) |
    +---------------------------+
    |        Domain             |   ← Business rules
    | (Entities, Business Rule) |
    +---------------------------+
```
- **Domain (Entities)**: Nơi chứa business logic thuần (class, entity, value object). Không biết gì về DB, REST hay framework.
- **Application (Use Cases)**: Chứa logic ứng dụng (dịch vụ gọi domain, xử lý flow). Biết domain, nhưng không biết adapter nào (DB, REST...).
- **Adapters (Infrastructure + Interface)**: Triển khai chi tiết (ví dụ: Repository implement bằng JPA, REST, Controller, Kafka consumer).
- **Frameworks**: Spring Boot, Hibernate, Kafka lib... chỉ là "plugin" cung cấp công cụ.

## 1. Entities (Domain Layer)
- Chứa **Business rule cốt lõi** 
- Không phụ thuộc Spring / JPA
- KHÔNG annotation như `@Entity`

**Ví dụ:**
```java
public class User {
    private final Long id;
    private final String email;

    public User(Long id, String email) {
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        this.id = id;
        this.email = email;
    }

    public Long getId() { return id; }
    public String getEmail() { return email; }
}
```
✅ Có logic nghiệp vụ
❌ Không JPA, Không lombok

## 2. Use Case (Application Layer)
- Điều phối nghiệp vụ
- Gọi **Interface**, không gọi implementation
- Mỗi use case = 1 hành động

**Interface Repository**
```java
public interface UserRepository {
    User save(User user);
    boolean existsByEmail(String email);
}
```

**Use Case: Create User**
```java
public class CreateUserUseCase {

    private final UserRepository userRepository;

    public CreateUserUseCase(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User execute(String email) {
        if (userRepository.existsByEmail(email)) {
            throw new RuntimeException("Email already exists");
        }

        User user = new User(null, email);
        return userRepository.save(user);
    }
}
```
📌 **Use case không biết DB là gì**

## 3. Interface Adapter
Chuyển đổi dữ liệu giữa:
- Web ↔ Use Case
- DB ↔ Entity

**DTO**
```java
public record CreateUserRequest(String email) {}
public record UserResponse(Long id, String email) {}
```
**Mapper**
```java
public class UserMapper {
    public static UserResponse toResponse(User user) {
        return new UserResponse(user.getId(), user.getEmail());
    }
}
```
## 4. Frameworks & Drivers (Spring Boot)
**JPA Entity**
```java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id
    @GeneratedValue
    private Long id;

    private String email;
}
```
## 4. Frameworks & Drivers (Spring Boot)
**JPA Entity**
```java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id
    @GeneratedValue
    private Long id;

    private String email;
}
```
**Spring Data Repository**
```java
public interface JpaUserRepository
        extends JpaRepository<UserJpaEntity, Long> {
    boolean existsByEmail(String email);
}
```
**Repository Implementation (Adapter)**
```java
@Component
public class UserRepositoryImpl implements UserRepository {

    private final JpaUserRepository jpaRepo;

    public UserRepositoryImpl(JpaUserRepository jpaRepo) {
        this.jpaRepo = jpaRepo;
    }

    @Override
    public User save(User user) {
        UserJpaEntity entity = new UserJpaEntity();
        entity.setEmail(user.getEmail());

        UserJpaEntity saved = jpaRepo.save(entity);
        return new User(saved.getId(), saved.getEmail());
    }

    @Override
    public boolean existsByEmail(String email) {
        return jpaRepo.existsByEmail(email);
    }
}
```
📌 **Không thuộc Domain, không ngược lại**

## 4. Controller (Outer Layer)
```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final CreateUserUseCase createUserUseCase;

    public UserController(CreateUserUseCase createUserUseCase) {
        this.createUserUseCase = createUserUseCase;
    }

    @PostMapping
    public UserResponse create(@RequestBody CreateUserRequest request) {
        User user = createUserUseCase.execute(request.email());
        return UserMapper.toResponse(user);
    }
}
```
## 5. Cấu trúc thư mục CHUẨN
```bash
src/main/java
└── com.example.app
    ├── domain
    │   └── user
    │       ├── User.java
    │       └── UserRepository.java
    ├── application
    │   └── user
    │       └── CreateUserUseCase.java
    ├── adapter
    │   ├── web
    │   │   └── UserController.java
    │   └── persistence
    │       ├── UserRepositoryImpl.java
    │       ├── UserJpaEntity.java
    │       └── JpaUserRepository.java
    └── config
        └── BeanConfig.java
```

## 6. Dependency Injection (BeanConfig)
```java
@Configuration
public class BeanConfig {

    @Bean
    CreateUserUseCase createUserUseCase(UserRepository userRepository) {
        return new CreateUserUseCase(userRepository);
    }
}
```

## 7. Clean Architecture vs Layered Architecture
| Layered                           | Clean                            |
| --------------------------------- | -------------------------------- |
| Controller → Service → Repository | Controller → UseCase → Interface |
| Phụ thuộc Spring                  | Không phụ thuộc framework        |
| Test khó                          | Test dễ                          |
| Business bị phân tán              | Business tập trung               |

## 8. Test Use Case cực dễ
```java
@Test
void createUser_success() {
    UserRepository fakeRepo = new InMemoryUserRepository();
    CreateUserUseCase useCase = new CreateUserUseCase(fakeRepo);

    User user = useCase.execute("test@gmail.com");

    assertNotNull(user);
}
```
👉 **Không cần Spring, không DB**

## 9. Sai lầm phổ biến (RẤT HAY GẶP)
❌ Domain có `@Entity`
❌ Use Case gọi `JpaRepository`
❌ Controller xử lý nghiệp vụ
❌ Entity = DTO
❌ Lạm dụng Lombok

## 10. Khi nào NÊN dùng Clean Architecture?
✅ Dự án lớn
✅ Microservices
✅ Nhiều business rules
❌ CRUD nhỏ → dùng Layered Architecture