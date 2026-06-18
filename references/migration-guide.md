# Migration Guide: Layered → Hexagonal

## Step 1 — Extract domain models
**Before:**
```java
@Entity
@Table(name = "orders")
public class Order {
  @Id @GeneratedValue private Long id;
  private String status;
  @OneToMany private List<OrderItem> items;
}
```
**After (domain/model/Order.java):**
```java
public class Order {
  private OrderId id;
  private OrderStatus status;
  private List<OrderItem> items;

  public Order(OrderId id, List<OrderItem> items) {
    this.id = Objects.requireNonNull(id);
    this.items = List.copyOf(items);
    this.status = OrderStatus.PENDING;
  }
}
```
**After (adapter/outbound/persistence/entity/OrderEntity.java):**
```java
@Entity
@Table(name = "orders")
public class OrderEntity {
  @Id private String id;
  @Enumerated(EnumType.STRING) private OrderStatus status;
  @OneToMany(cascade = ALL) @JoinColumn(name = "order_id") private List<OrderItemEntity> items;
}
```

## Step 2 — Create Value Objects
Wrap primitives: `String` → `OrderId`, `BigDecimal` → `Money`

## Step 3 — Define inbound ports
Extract interface from existing `OrderService`:
```java
public interface CreateOrderUseCase {
  Order create(CreateOrderCommand command);
}
```

## Step 4 — Define outbound ports
Extract interface from existing `OrderRepository` (remove `@Repository`):
```java
public interface OrderRepository {
  void save(Order order);
  Optional<Order> findById(OrderId id);
}
```

## Step 5 — Wrap services as use cases
Move `OrderServiceImpl` → `application/service/CreateOrderService.java`.
Replace `@Autowired` with constructor injection of ports.

## Step 6 — Create persistence adapter
Create `OrderEntity`, `OrderJpaRepository`, `OrderEntityMapper`, `OrderRepositoryAdapter`.

## Step 7 — Update controllers
Move to `adapter/inbound/rest/controller/`. Inject `CreateOrderUseCase` instead of `OrderService`.

## Step 8 — Add shared layer
Add `ApiResponse`, `GlobalExceptionHandler`, `CorrelationIdFilter`.

## Step 9 — Update tests
- Domain: pure JUnit, no Spring
- Use case: Mockito + inject mocks
- Adapter: `@WebMvcTest` or `@DataJpaTest` with Testcontainers

## Step 10 — Audit
Run the full audit workflow.
