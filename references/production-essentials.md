# Production Essentials

## Resilience4j

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
```

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> circuitBreakerCustomizer() {
  return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
    .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
    .build());
}
```

## Flyway migrations
- `src/main/resources/db/migration/V1__init.sql`
- `V{version}__{description}.sql` naming
- `spring.flyway.enabled=true`

```sql
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    status VARCHAR(20) NOT NULL,
    total DECIMAL(19,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_orders_status ON orders(status);
```

## Testcontainers integration test
```java
@SpringBootTest
@Testcontainers
class OrderRepositoryAdapterTest {
  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

  @DynamicPropertySource
  static void properties(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", postgres::getJdbcUrl);
    r.add("spring.datasource.username", postgres::getUsername);
    r.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired private OrderRepositoryAdapter adapter;

  @Test
  void shouldSaveAndFindOrder() {
    var order = new Order(new OrderId("1"), List.of(), new Money(BigDecimal.TEN, Currency.getInstance("USD")));
    adapter.save(order);
    var found = adapter.findById(new OrderId("1"));
    assertThat(found).isPresent();
  }
}
```

## Actuator
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
```

## Connection pool config
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      idle-timeout: 30000
      connection-timeout: 2000
```
