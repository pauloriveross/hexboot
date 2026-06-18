# Adapters

## REST adapter

### Controller pattern
```java
@RestController
@RequestMapping("/api/v1/{entities}")
public class {Entity}Controller {
  private final Create{Entity}UseCase createUseCase;
  private final Get{Entity}UseCase getUseCase;

  public {Entity}Controller(Create{Entity}UseCase createUseCase, Get{Entity}UseCase getUseCase) {
    this.createUseCase = createUseCase; this.getUseCase = getUseCase;
  }

  @PostMapping
  public ResponseEntity<ApiResponse<{Entity}Response>> create(@Valid @RequestBody {Entity}Request request) {
    var command = {Entity}Mapper.toCommand(request);
    var entity = createUseCase.create(command);
    return ResponseEntity.status(201).body(ApiResponse.created({Entity}Mapper.toResponse(entity)));
  }

  @GetMapping("/{id}")
  public ResponseEntity<ApiResponse<{Entity}Response>> get(@PathVariable String id) {
    var result = getUseCase.get(new {Entity}Id(id));
    return result.map(e -> ResponseEntity.ok(ApiResponse.ok({Entity}Mapper.toResponse(e))))
        .orElse(ResponseEntity.notFound().build());
  }
}
```

### DTO ↔ Domain mapping (manual, no MapStruct to avoid extra dep)

```java
public class {Entity}Mapper {
  public static Create{Entity}Command toCommand({Entity}Request request) {
    var items = request.items().stream()
      .map(i -> new OrderItem(new ProductId(i.productId()), i.quantity(), new Money(i.price(), Currency.getInstance("USD"))))
      .toList();
    var total = new Money(request.total(), Currency.getInstance("USD"));
    return new Create{Entity}Command(items, total);
  }

  public static {Entity}Response toResponse({Entity} entity) {
    return new {Entity}Response(entity.id().value(), entity.status().name(), entity.total().amount());
  }
}
```

## Persistence adapter

### JPA entity
```java
@Entity
@Table(name = "orders", indexes = @Index(columnList = "status"))
public class OrderEntity {
  @Id private String id;
  @Enumerated(EnumType.STRING) private OrderStatus status;
  @OneToMany(cascade = ALL, fetch = EAGER) @JoinColumn(name = "order_id")
  private List<OrderItemEntity> items;

  protected OrderEntity() {}
  public OrderEntity(String id, OrderStatus status, List<OrderItemEntity> items) {
    this.id = id; this.status = status; this.items = items;
  }
}
```

### Entity ↔ Domain mapper
```java
public class OrderEntityMapper {
  public OrderEntity toEntity(Order domain) {
    var items = domain.items().stream().map(i -> new OrderItemEntity(...)).toList();
    return new OrderEntity(domain.id().value(), domain.status(), items);
  }
  public Order toDomain(OrderEntity entity) {
    var items = entity.getItems().stream().map(i -> new OrderItem(...)).toList();
    return new Order(new OrderId(entity.getId()), items, new Money(entity.getTotal(), Currency.getInstance("USD")));
  }
}
```

### JPA repository
```java
public interface OrderJpaRepository extends JpaRepository<OrderEntity, String> {
  @EntityGraph(attributePaths = "items")
  Optional<OrderEntity> findWithItemsById(String id);
}
```

### Outbound port implementation
```java
public class OrderRepositoryAdapter implements OrderRepository {
  private final OrderJpaRepository jpaRepository;
  private final OrderEntityMapper mapper;

  public OrderRepositoryAdapter(OrderJpaRepository jpaRepository, OrderEntityMapper mapper) {
    this.jpaRepository = jpaRepository; this.mapper = mapper;
  }

  @Override
  public Optional<Order> findById(OrderId id) {
    return jpaRepository.findById(id.value()).map(mapper::toDomain);
  }

  @Override
  public void save(Order order) {
    jpaRepository.save(mapper.toEntity(order));
  }
}
```

## External HTTP client adapter

```java
public class PaymentClientAdapter implements PaymentClient {
  private final RestTemplate restTemplate;

  public PaymentClientAdapter(RestTemplate restTemplate) {
    this.restTemplate = restTemplate;
  }

  @Override
  @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
  public PaymentResult charge(Money amount) {
    var response = restTemplate.postForEntity("/api/charge", new ChargeRequest(amount), ChargeResponse.class);
    return response.getStatusCode().is2xxSuccessful() ? PaymentResult.success() : PaymentResult.failed();
  }

  public PaymentResult fallback(Money amount, Throwable t) {
    MDC.put("fallback", "paymentService");
    return PaymentResult.failed();
  }
}
```

## Messaging adapter (Kafka)

### Producer
```java
@Component
public class KafkaEventPublisher implements EventPublisher {
  private final KafkaTemplate<String, Object> kafka;

  public KafkaEventPublisher(KafkaTemplate<String, Object> kafka) { this.kafka = kafka; }

  @Override
  public void publish(DomainEvent event) {
    kafka.send(event.aggregateId(), event);
  }
}
```

### Consumer
```java
@Component
public class OrderEventConsumer {
  @KafkaListener(topics = "order-events")
  public void onOrderCreated(OrderCreatedEvent event) {
    // delegate to use case
  }
}
```
