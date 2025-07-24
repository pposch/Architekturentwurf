### Teil B – Domain Design: Buchungsservice

#### 1. Wichtige Entities, Value Objects und Aggregate Root

```mermaid
classDiagram
    class Booking {
        +Booking Id
        +Customer CustomerId
        +Room RoomId
        +DateTime From
        +DateTime To
        +BookingStatus Status
        +decimal TotalAmount
        +static Booking ReserveRoom(CustomerId, RoomId, DateTime, DateTime)
        +void Cancel(string reason)
    }
    class Room
    class Customer
    class Money {
        +decimal Amount
        +string Currency
    }
    Booking o-- Room
    Booking o-- Customer
    Booking o-- Money : TotalAmount
```

* **Aggregate Root:** `Booking`
* **Entities:** `Booking`
* **Value Objects:** `RoomId`, `CustomerId`, `Money`

#### 2. Domain Events

* `BookingCreated(BookingId, CustomerId, RoomId, From, To)`
* `BookingCancelled(BookingId, Reason)`
* `BookingConfirmed(BookingId)`

#### 3. Use Case / Command Handler Beispiel

```csharp
public class CreateBookingCommand {
    public Guid CustomerId { get; }
    public Guid RoomId { get; }
    public DateTime From { get; }
    public DateTime To { get; }
}

public class CreateBookingHandler {
    private readonly IBookingRepository _repo;
    private readonly IMessageBus _bus;

    public CreateBookingHandler(IBookingRepository repo, IMessageBus bus) {
        _repo = repo;
        _bus = bus;
    }

    public async Task Handle(CreateBookingCommand cmd) {
        var booking = Booking.ReserveRoom(cmd.CustomerId, cmd.RoomId, cmd.From, cmd.To);
        await _repo.AddAsync(booking);
        await _bus.Publish(new BookingCreated(booking.Id, cmd.CustomerId, cmd.RoomId, cmd.From, cmd.To));
    }
}
```

#### 4. UML-Klassendiagramm

Das folgende UML-Diagramm zeigt die wichtigsten Klassen, ihre Beziehungen und Methoden rund um den Aggregatstamm `Booking`. Es illustriert, wie Buchungen modelliert werden und wie daraus Domain Events wie `BookingCreatedEvent` entstehen.

- **Booking** ist die zentrale Domänenklasse mit allen buchungsrelevanten Informationen.
- Sie enthält Methoden zur Erstellung und Stornierung einer Buchung.
- Über die Methode `ReserveRoom(...)` wird ein neues Booking erstellt.
- Nach erfolgreicher Erstellung wird ein `BookingCreatedEvent` generiert und an das Messaging-System übergeben.


```mermaid
classDiagram
    class Booking {
        +Booking Id
        +Customer CustomerId
        +Room RoomId
        +DateTime From
        +DateTime To
        +BookingStatus Status
        +static Booking ReserveRoom(CustomerId, RoomId, DateTime, DateTime)
        +void Cancel(string reason)
    }
    class BookingCreatedEvent {
        +Booking Id
        +Customer CustomerId
        +Room RoomId
        +DateTime From
        +DateTime To
    }
    Booking --> BookingCreatedEvent
```

---
