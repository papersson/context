# Scala 3 Type Implementation Guide

## Translating Domain Patterns to Scala 3

### Simple Types (Constrained Values)
```scala
// Use opaque types for zero-cost wrappers
opaque type CustomerId = String
object CustomerId:
  def apply(value: String): Option[CustomerId] =
    if (value.nonEmpty) Some(value) else None
  extension (id: CustomerId) def value: String = id

// For numeric constraints
opaque type Age = Int
object Age:
  def apply(value: Int): Either[String, Age] =
    if (value >= 0 && value <= 150) Right(value)
    else Left(s"Age must be 0-150, got $value")
```

### Product Types (AND)
```scala
// Use case classes for records
case class Order(
  orderId: OrderId,
  customer: Customer,
  lines: NonEmptyList[OrderLine],  // Use refined types
  total: Money
)

// Make fields required by default, Option only when truly optional
case class Customer(
  id: CustomerId,
  email: Email,
  phone: Option[PhoneNumber]  // Explicitly optional
)
```

### Sum Types (OR)
```scala
// Use enums for simple choices
enum PaymentMethod:
  case CreditCard(number: CardNumber, cvv: CVV)
  case BankTransfer(account: AccountNumber)
  case Cash

// Use sealed traits when you need more control
sealed trait OrderState
case class Unvalidated(data: UnvalidatedData) extends OrderState
case class Validated(data: ValidatedData) extends OrderState
case class Priced(data: PricedData) extends OrderState
```

### State Machines
```scala
// Each state as its own type
case class UnvalidatedOrder(raw: Map[String, String])
case class ValidatedOrder(customer: Customer, items: NonEmptyList[Item])
case class PricedOrder(validated: ValidatedOrder, total: Money)

// Transitions as functions that can only accept correct state
def validate(unvalidated: UnvalidatedOrder): Either[ValidationError, ValidatedOrder]
def price(validated: ValidatedOrder): Either[PricingError, PricedOrder]
// Can't accidentally call price on UnvalidatedOrder!
```

### Making Illegal States Unrepresentable
```scala
// Instead of:
case class Email(address: String, isVerified: Boolean) // Can be invalid!

// Use:
enum EmailStatus:
  case Unverified(email: Email)
  case Verified(email: Email, token: VerificationToken)
// Can't have verified without token!
```

### Workflows with Effects (ZIO)
```scala
import zio._

// Pure domain function
type ValidateOrder = UnvalidatedOrder => Either[ValidationError, ValidatedOrder]

// With effects
type ValidateOrderWithEffects =
  UnvalidatedOrder => ZIO[AddressService & ProductCatalog, ValidationError, ValidatedOrder]

// Implementation
def validateOrder(order: UnvalidatedOrder):
  ZIO[AddressService & ProductCatalog, ValidationError, ValidatedOrder] =
  for
    address <- AddressService.validate(order.address)
    items   <- ZIO.foreach(order.items)(ProductCatalog.validate)
    // ... more validation
  yield ValidatedOrder(...)
```

### Smart Constructors Pattern
```scala
// Make constructor private, provide factory method
final case class Email private (value: String)

object Email:
  private val emailRegex = "^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$".r

  def parse(s: String): Either[String, Email] =
    if (emailRegex.matches(s)) Right(new Email(s))
    else Left(s"Invalid email: $s")

  // For literals known at compile time
  inline def unsafeFrom(inline s: String): Email =
    ${ validateEmailMacro('s) }
```

### Aggregate Boundaries
```scala
// Order is the aggregate root
case class Order private (
  id: OrderId,
  lines: NonEmptyList[OrderLine],
  total: Money
) {
  // All modifications go through the root
  def updateLineQuantity(lineId: LineId, qty: Quantity): Either[OrderError, Order] =
    for
      updatedLines <- lines.findAndUpdate(lineId, _.copy(quantity = qty))
      newTotal = updatedLines.map(_.lineTotal).sum
    yield copy(lines = updatedLines, total = newTotal)
}

// OrderLine can't be modified directly from outside
case class OrderLine private[Order] (
  id: LineId,
  product: ProductId,
  quantity: Quantity,
  price: Money
)
```

### Event Sourcing Pattern
```scala
// Events as ADT
enum OrderEvent:
  case Created(id: OrderId, customer: CustomerId, items: List[Item])
  case ItemAdded(orderId: OrderId, item: Item)
  case ItemRemoved(orderId: OrderId, itemId: ItemId)
  case Submitted(orderId: OrderId, total: Money)

// State reconstruction
def reconstruct(events: List[OrderEvent]): Order =
  events.foldLeft(Order.empty)(_.apply(_))
```

## Key Scala 3 Features for Domain Modeling

1. **Opaque Types**: Zero-cost domain types
2. **Enums**: Concise ADTs with exhaustive matching
3. **Union Types**: `String | Int` for flexible APIs
4. **Intersection Types**: Combining capabilities
5. **Extension Methods**: Add domain operations cleanly
6. **Given/Using**: Dependency injection
7. **Inline**: Compile-time validation
8. **Match Types**: Type-level computation

## Libraries to Consider

- **Refined**: For refined types with compile-time checking
- **Iron**: Lightweight refinement types for Scala 3
- **Chimney**: For transformations between similar types
- **ZIO**: For effect management and dependency injection
- **Cats**: For functional abstractions
