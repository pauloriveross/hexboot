# Ports and Use Cases

## Inbound port naming
- `Create{Entity}UseCase` — mutation
- `Get{Entity}UseCase` — single query
- `List{Entity}UseCase` — collection query
- `Update{Entity}UseCase` — partial/full update
- `Delete{Entity}UseCase` — removal
- `Process{Action}UseCase` — non-CRUD action

## Outbound port naming
- `{Entity}Repository` — persistence
- `{Event}Publisher` — messaging
- `{ExternalService}Client` — external API
- `{Domain}LockManager` — distributed locking

## Use case pattern

```java
public class CreateOrderService implements CreateOrderUseCase {
  private final OrderRepository orderRepository;
  private final EventPublisher eventPublisher;
  private final PricingDomainService pricingService;

  public CreateOrderService(OrderRepository orderRepository, EventPublisher eventPublisher,
      PricingDomainService pricingService) {
    this.orderRepository = orderRepository;
    this.eventPublisher = eventPublisher;
    this.pricingService = pricingService;
  }

  @Transactional
  @Override
  public Order create(CreateOrderCommand command) {
    var id = new OrderId(UUID.randomUUID().toString());
    var total = pricingService.calculateTotal(command.items(), command.discount());
    var order = new Order(id, command.items(), total);
    orderRepository.save(order);
    eventPublisher.publish(new OrderCreatedEvent(id, Instant.now()));
    return order;
  }
}
```

## @Transactional rules
- On application use case, NOT on domain
- NOT on interface method (implementation detail)
- Read-only queries: `@Transactional(readOnly = true)`
- Avoid nested transactions: prefer `REQUIRED` (default)

## Command/Query DTOs
- Inbound: `{Action}{Entity}Command` for mutations, `{Entity}Query` for queries
- Immutable `record` with `@Valid` annotations
- Located in `application/port/inbound/` or `adapter/inbound/rest/dto/`
