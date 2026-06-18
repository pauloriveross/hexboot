# Domain Patterns

## Aggregate rules
- Root entity with globally unique identity
- Only root has repository
- Access child entities only through root
- One aggregate = one transaction boundary

## Value Object rules
- Immutable: `record` or `final class` with no setters
- `equals()` and `hashCode()` based on all attributes
- Self-validating in constructor
- No identity field

```java
public record Money(BigDecimal amount, Currency currency) {
  public Money {
    if (amount == null || currency == null) throw new IllegalArgumentException("amount and currency required");
    if (amount.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("amount must be non-negative");
  }
  public Money add(Money other) {
    if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("currency mismatch");
    return new Money(this.amount.add(other.amount), this.currency);
  }
}
```

## Value Object types
- `OrderId`, `CustomerId`, `PaymentId` (wrapped UUID/String)
- `Money` (BigDecimal + Currency)
- `Email`, `PhoneNumber`, `Address`
- `OrderStatus` (enum)

## Domain Service
- Stateless, injectable via constructor
- Orchestrates multiple aggregates
- Name: verb + noun + `DomainService`

```java
public class PricingDomainService {
  public Money calculateTotal(List<OrderItem> items, Discount discount) {
    var subtotal = items.stream().map(OrderItem::subtotal).reduce(Money.ZERO, Money::add);
    return discount.apply(subtotal);
  }
}
```

## Domain Event
```java
public record OrderCreatedEvent(String aggregateId, Instant occurredOn) implements DomainEvent {}
```
