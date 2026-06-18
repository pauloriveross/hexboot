---
name: hexboot
description: >
  Creates, modifies, migrates, and audits production-ready Spring Boot
  microservices using Hexagonal Architecture (Ports & Adapters). Use when
  the user asks to build/create/generate a Spring Boot microservice, add
  features to an existing one, audit/refactor/review architecture, or
  mentions: hexagonal architecture, ports and adapters, DDD, clean
  architecture, onion architecture, spring boot, microservice.
license: MIT
metadata:
  author: hexboot
  java: "17+"
  spring-boot: "3.x"
---

# hexboot — Spring Boot Hexagonal Microservices

## BEFORE ANY ACTION

Ask the developer:
- Base package: `{basePackage}`
- Service name: `{serviceName}`
- Java version: 17 / 21 / 23
- Module layout: single-module / multi-module
- Database: Postgres / MySQL / H2 / MongoDB / none
- API docs: OpenAPI yes/no
- Security: JWT / OAuth2 / Basic / none
- Messaging: Kafka / RabbitMQ / none
- Docker: yes/no
- Tests: unit / unit+integration / none

Wait for answers. Generate ONLY what was selected.

---

## 1. CREATE MICROSERVICE

### 1.1 Directory structure

```
{serviceName}/
├── pom.xml
├── Dockerfile                            (if Docker=yes)
├── docker-compose.yml                    (if Docker=yes)
└── src/
    ├── main/java/{basePackagePath}/
    │   ├── {serviceName}Application.java
    │   ├── domain/
    │   │   ├── model/
    │   │   ├── vo/
    │   │   ├── service/
    │   │   └── event/                    (if messaging)
    │   ├── application/
    │   │   ├── port/
    │   │   │   ├── inbound/
    │   │   │   └── outbound/
    │   │   └── service/
    │   ├── adapter/
    │   │   ├── inbound/
    │   │   │   ├── rest/
    │   │   │   │   ├── dto/
    │   │   │   │   ├── mapper/
    │   │   │   │   └── controller/
    │   │   │   ├── security/             (if Security!=none)
    │   │   │   └── messaging/            (if messaging)
    │   │   └── outbound/
    │   │       ├── persistence/
    │   │       │   ├── entity/
    │   │       │   ├── mapper/
    │   │       │   └── repository/
    │   │       ├── client/               (if needed)
    │   │       └── messaging/            (if messaging)
    │   └── shared/
    │       ├── dto/
    │       ├── exception/
    │       └── config/
    └── test/java/{basePackagePath}/
```

### 1.2 Layer rules (NON-NEGOTIABLE)

**domain/** — Pure Java. ZERO framework imports. No Spring, JPA, Jackson.
- `domain/model/`: Business entities with invariants in constructors
- `domain/vo/`: Value objects with equals/hashCode, immutability
- `domain/service/`: Stateless domain services for cross-entity logic
- `domain/event/`: Domain event records implementing `DomainEvent` interface

**application/** — Depends ONLY on domain.
- `application/port/inbound/`: Interfaces that define what the app does
- `application/port/outbound/`: Interfaces that define what the app needs from outside
- `application/service/`: Use case implementations. `@Transactional` goes HERE if needed

**adapter/** — Implements ports. Depends on application + domain + frameworks.
- `adapter/inbound/rest/`: `@RestController`, DTOs, mappers. Calls inbound ports via constructor injection
- `adapter/outbound/persistence/`: `@Entity`, `*JpaRepository`, mappers. Implements outbound ports

**shared/** — Cross-cutting: exceptions, config, ApiResponse.

### 1.3 Naming convention (NON-NEGOTIABLE)

| Layer | Example | Suffix |
|-------|---------|--------|
| Domain model | `Order` | No suffix |
| Value Object | `OrderId`, `Money` | Semantic |
| Domain event | `OrderCreatedEvent` | `*Event` |
| Inbound port | `CreateOrderUseCase` | `*UseCase` |
| Outbound port | `OrderRepository`, `EventPublisher` | `*Repository`, `*Publisher` |
| Use case | `CreateOrderService` | `*Service` |
| JPA entity | `OrderEntity` | `*Entity` |
| JPA repository | `OrderJpaRepository` | `*JpaRepository` |
| Inbound DTO | `CreateOrderRequest`, `OrderResponse` | `*Request`, `*Response` |
| Mapper | `OrderMapper` | `*Mapper` |
| Controller | `OrderController` | `*Controller` |

### 1.4 Code templates

#### ApiResponse (shared/dto/ApiResponse.java)
```java
public record ApiResponse<T>(boolean success, T data, String message, int code) {
  public static <T> ApiResponse<T> ok(T data) {
    return new ApiResponse<>(true, data, "Success", 200);
  }
  public static <T> ApiResponse<T> created(T data) {
    return new ApiResponse<>(true, data, "Created", 201);
  }
  public static <T> ApiResponse<T> error(String message, int code) {
    return new ApiResponse<>(false, null, message, code);
  }
}
```

#### Domain model (domain/model/Order.java)
```java
public class Order {
  private final OrderId id;
  private OrderStatus status;
  private final Money total;
  private final List<OrderItem> items;

  public Order(OrderId id, List<OrderItem> items, Money total) {
    if (items == null || items.isEmpty()) throw new IllegalArgumentException("Items required");
    this.id = Objects.requireNonNull(id);
    this.items = List.copyOf(items);
    this.total = Objects.requireNonNull(total);
    this.status = OrderStatus.PENDING;
  }

  public OrderId id() { return id; }
  public OrderStatus status() { return status; }
  public Money total() { return total; }
  public List<OrderItem> items() { return items; }
}
```

#### Inbound port (application/port/inbound/CreateOrderUseCase.java)
```java
public interface CreateOrderUseCase {
  Order create(CreateOrderCommand command);
}
```

#### Outbound port (application/port/outbound/OrderRepository.java)
```java
public interface OrderRepository {
  Optional<Order> findById(OrderId id);
  void save(Order order);
}
```

#### Use case (application/service/CreateOrderService.java)
```java
public class CreateOrderService implements CreateOrderUseCase {
  private final OrderRepository orderRepository;
  private final EventPublisher eventPublisher;

  public CreateOrderService(OrderRepository orderRepository, EventPublisher eventPublisher) {
    this.orderRepository = orderRepository;
    this.eventPublisher = eventPublisher;
  }

  @Transactional
  @Override
  public Order create(CreateOrderCommand command) {
    var id = new OrderId(UUID.randomUUID().toString());
    var order = new Order(id, command.items(), command.total());
    orderRepository.save(order);
    eventPublisher.publish(new OrderCreatedEvent(id));
    return order;
  }
}
```

#### REST controller (adapter/inbound/rest/OrderController.java)
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
  private final CreateOrderUseCase createOrderUseCase;

  public OrderController(CreateOrderUseCase createOrderUseCase) {
    this.createOrderUseCase = createOrderUseCase;
  }

  @PostMapping
  public ResponseEntity<ApiResponse<OrderResponse>> create(@Valid @RequestBody CreateOrderRequest request) {
    var command = OrderMapper.toCommand(request);
    var order = createOrderUseCase.create(command);
    return ResponseEntity.status(201).body(ApiResponse.created(OrderMapper.toResponse(order)));
  }
}
```

#### Global exception handler (shared/exception/GlobalExceptionHandler.java)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<ApiResponse<Void>> handleBadArgument(IllegalArgumentException ex) {
    return ResponseEntity.badRequest().body(ApiResponse.error(ex.getMessage(), 400));
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ApiResponse<Void>> handleGeneric(Exception ex) {
    return ResponseEntity.status(500).body(ApiResponse.error("Internal error", 500));
  }
}
```

#### Domain event base (domain/event/DomainEvent.java)
```java
public interface DomainEvent {
  String aggregateId();
  Instant occurredOn();
}
```

#### Event publisher port (application/port/outbound/EventPublisher.java)
```java
public interface EventPublisher {
  void publish(DomainEvent event);
}
```

#### Correlation ID filter (shared/config/CorrelationIdFilter.java)
```java
@Component
@Order(1)
public class CorrelationIdFilter extends OncePerRequestFilter {
  private static final String HEADER = "X-Correlation-Id";

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws ServletException, IOException {
    var correlationId = request.getHeader(HEADER);
    if (correlationId == null || correlationId.isBlank()) correlationId = UUID.randomUUID().toString();
    MDC.put("correlationId", correlationId);
    response.setHeader(HEADER, correlationId);
    try { chain.doFilter(request, response); } finally { MDC.clear(); }
  }
}
```

### 1.5 Generated files based on answers

| Answer | Files generated |
|--------|----------------|
| DB=Postgres | `application-postgres.yml`, `postgresql` dependency |
| DB=Mongo | `spring-boot-starter-data-mongodb`, Mongo config |
| Security=JWT | `security.md` reference: JwtFilter, SecurityConfig, @PreAuthorize on inbound ports |
| Messaging=Kafka | `EventPublisher` impl using KafkaTemplate, consumer adapter |
| Docker=yes | Multi-stage `Dockerfile`, `docker-compose.yml` with service + DB |
| Tests=unit+integration | `*Test.java` for use case (Mockito) + controller (MockMvc) + persistence (Testcontainers) |
| API docs=yes | OpenAPI config bean, springdoc-openapi dependency |
| Resilience=yes | Resilience4j config, `@CircuitBreaker` on outbound adapter |

---

## 2. MODIFY MICROSERVICE

### 2.1 Detect project structure
- Scan existing packages to determine single vs multi-module, base package, used frameworks
- If NOT hexagonal → reject with: "This project uses traditional layering. Run MIGRATE workflow first."
- If hexagonal → proceed

### 2.2 Ask what to add
- "New domain entity with full CRUD?"
- "New endpoint in existing entity?"
- "New external client adapter?"
- "New messaging producer/consumer?"

### 2.3 Order of operations (MANDATORY)
1. Domain: model + VO + event (if applicable)
2. Application: inbound port + outbound port
3. Application: use case implementation
4. Adapter: persistence entity + repository + mapper
5. Adapter: REST DTO + mapper + controller
6. Adapter: messaging (if applicable)
7. Shared: update exception handler if needed
8. Tests

Never skip steps. Never put domain logic in adapter.

### 2.4 Transaction rules
- `@Transactional` on application use case class/method
- NOT on domain service
- NOT on adapter
- Exception: `@Transactional` on `*JpaRepository` is inherited from Spring Data, that's fine

### 2.5 Security rules
- `@PreAuthorize` on inbound port interface
- `SecurityFilterChain` bean in `shared/config/SecurityConfig.java`
- JWT validation in `adapter/inbound/security/JwtAuthenticationFilter.java`

---

## 3. MIGRATE MICROSERVICE (Layered → Hexagonal)

For existing projects using Controller → Service → Repository pattern.

### 3.1 Pre-flight check
Run these greps to assess current state:
```
grep -r "@Service" src/main/java/ | wc -l
grep -r "@Autowired" src/main/java/ | wc -l
grep -r "import org.springframework" src/main/java/*/domain/ 2>/dev/null
```

### 3.2 Migration steps (each is independent, can stop at any step)

**Step 1 — Extract domain models:**
- Move POJOs without framework annotations to `domain/model/`
- Remove `@Entity`, `@Table`, `@Column`, `@Id` → those stay in new entity classes

**Step 2 — Create Value Objects:**
- Wrap primitive identifiers → `domain/vo/{EntityName}Id.java`
- Wrap Money, Email, Phone, etc.

**Step 3 — Define inbound ports:**
- Extract existing service interfaces → `application/port/inbound/`

**Step 4 — Define outbound ports:**
- Extract repository interfaces → `application/port/outbound/`
- Remove `@Repository` annotation (it's an implementation detail)

**Step 5 — Wrap services as use cases:**
- Move service implementations to `application/service/`
- They now implement inbound ports and inject outbound ports

**Step 6 — Create persistence adapter:**
- New `@Entity` classes in `adapter/outbound/persistence/entity/`
- New `*JpaRepository` in `adapter/outbound/persistence/repository/`
- New mapper (Entity ↔ Domain) in `adapter/outbound/persistence/mapper/`
- Outbound port implementation that delegates to JpaRepository

**Step 7 — Update controllers:**
- Move to `adapter/inbound/rest/controller/`
- Inject inbound ports instead of services directly
- Add DTO classes, mappers, `@Valid` validation

**Step 8 — Add shared components:**
- `ApiResponse<T>`, `GlobalExceptionHandler`, `CorrelationIdFilter`

**Step 9 — Fix tests:**
- Domain tests: pure JUnit, no Spring context
- Use case tests: Mockito only, inject mocked outbound ports
- Adapter tests: Spring Boot test with sliced context

**Step 10 — Verify:**
Run full audit (see section 4).

---

## 4. AUDIT MICROSERVICE

Run against any existing microservice. Each check outputs exact location and fix.

### 4.1 Architecture scan

```
CHECK 1 — Framework imports in domain
  CMD:   grep -rn "import org.springframework\|import jakarta\.persistence\|import com\.fasterxml" src/main/java/*/domain/
  PASS:  No output
  FAIL:  [CRITICAL] Domain layer imports framework. Move to adapter.

CHECK 2 — @Entity in domain
  CMD:   grep -rn "@Entity\|@Table\|@Column" src/main/java/*/domain/
  PASS:  No output
  FAIL:  [CRITICAL] JPA annotation in domain. Create separate *Entity class in adapter.

CHECK 3 — @Transactional in domain
  CMD:   grep -rn "@Transactional" src/main/java/*/domain/
  PASS:  No output
  FAIL:  [CRITICAL] Transactional in domain. Move to application service.

CHECK 4 — Business logic in adapter
  CMD:   grep -rn "if.*||.*for.*||.*while" src/main/java/*/adapter/*/controller/
  PASS:  No complex logic in controllers
  FAIL:  [HIGH] Controller contains business logic. Extract to use case.

CHECK 5 — Service injecting another service directly
  CMD:   grep -rn "@Autowired" src/main/java/*/application/ | grep "Service"
  PASS:  No direct service-to-service injection
  FAIL:  [HIGH] Service injected into another service. Use a port.

CHECK 6 — Missing global exception handler
  CMD:   grep -rn "@RestControllerAdvice" src/main/java/
  PASS:  Found
  FAIL:  [HIGH] Add GlobalExceptionHandler in shared/exception/

CHECK 7 — Missing ApiResponse
  CMD:   grep -rn "ApiResponse" src/main/java/*/shared/
  PASS:  Found
  FAIL:  [MEDIUM] Add ApiResponse record in shared/dto/

CHECK 8 — Missing @Valid on request DTOs
  CMD:   grep -rn "@Valid" src/main/java/*/adapter/*/controller/
  PASS:  Found
  FAIL:  [MEDIUM] Add @Valid to controller @RequestBody parameters.

CHECK 9 — Ports in adapter package
  CMD:   grep -rn "interface.*UseCase\|interface.*Repository\|interface.*Publisher" src/main/java/*/adapter/
  PASS:  No output
  FAIL:  [MEDIUM] Port interface found in adapter package. Move to application/port/.

CHECK 10 — Circular package dependency
  CMD:   Examine package structure. domain/ must NOT import application/ or adapter/.
  PASS:  domain is pure
  FAIL:  [CRITICAL] Domain depends on outer layer. Fix imports.

CHECK 11 — @PreAuthorize on controller vs port
  CMD:   grep -rn "@PreAuthorize" src/main/java/*/adapter/
  PASS:  No @PreAuthorize in adapter (good, it's on port)
  FAIL:  [MEDIUM] @PreAuthorize on controller. Move to inbound port interface.

CHECK 12 — No @Repository on port interfaces
  CMD:   grep -rn "@Repository" src/main/java/*/application/
  PASS:  No output
  FAIL:  [LOW] @Repository on port interface. Remove - it's an implementation detail.
```

### 4.2 Optimization scan

```
CHECK 13 — N+1 queries
  CMD:   grep -rn "@EntityGraph\|JOIN FETCH\|@Query" src/main/java/*/adapter/*/persistence/
  PASS:  Fetch strategies defined
  FAIL:  [MEDIUM] Missing fetch plans. Add @EntityGraph or JOIN FETCH to repository queries.

CHECK 14 — Missing indexes
  CMD:   grep -rn "@Table" -A5 src/main/java/*/adapter/*/entity/ | grep "@Index"
  PASS:  Indexes present
  FAIL:  [LOW] No @Index on entity. Add @Table(indexes = ...).

CHECK 15 — Hardcoded configs
  CMD:   grep -rn "localhost:5432\|jdbc:h2" src/main/resources/
  PASS:  Values are in properties
  FAIL:  [MEDIUM] Hardcoded connection strings. Move to application.yml profiles.

CHECK 16 — Connection pool default config
  CMD:   grep -rn "hikari\|maximum-pool-size" src/main/resources/
  PASS:  Pool configured
  FAIL:  [LOW] No connection pool config. Add HikariCP settings to application.yml.
```

### 4.3 Production-readiness scan

```
CHECK 17 — Actuator health
  CMD:   grep -rn "spring-boot-starter-actuator" pom.xml
  PASS:  Dependency present
  FAIL:  [MEDIUM] Add spring-boot-starter-actuator and configure health endpoints.

CHECK 18 — OpenAPI docs
  CMD:   grep -rn "springdoc\|openapi" pom.xml
  PASS:  Dependency present
  FAIL:  [MEDIUM] Add springdoc-openapi and config bean.

CHECK 19 — MDC correlation ID
  CMD:   grep -rn "CorrelationIdFilter\|MDC" src/main/java/
  PASS:  Filter present
  FAIL:  [MEDIUM] Add CorrelationIdFilter for request tracing.

CHECK 20 — Error codes in responses
  CMD:   grep -rn "public static.*error" src/main/java/*/shared/
  PASS:  ApiResponse has error codes
  FAIL:  [LOW] Add error code mapping to GlobalExceptionHandler.

CHECK 21 — Dockerfile
  CMD:   ls Dockerfile 2>/dev/null
  PASS:  Exists
  FAIL:  [LOW] No Dockerfile. Add multi-stage build.

CHECK 22 — Secrets in application.yml
  CMD:   grep -rn "password.*=.*[a-zA-Z]\|api.key\|secret" src/main/resources/ --include="*.yml"
  PASS:  No hardcoded secrets
  FAIL:  [CRITICAL] Secrets found. Use env vars or ${...} placeholders.
```

### 4.4 Output format

For each failed check, output:
```
[{SEVERITY}] {description}
  File: {file:line}
  Fix:  {exact instruction or code}
```

---

## 5. ADRs (Architecture Decision Records)

On CREATE, also generate:

```
docs/adr/001-use-hexagonal-architecture.md
docs/adr/002-use-spring-boot.md
docs/adr/003-use-{database}.md
```

Template:
```markdown
# ADR-{N}: {title}

**Context:** {why this decision was needed}
**Decision:** {what was decided}
**Consequences:** {trade-offs, implications}
```
