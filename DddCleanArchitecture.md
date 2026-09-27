# Table of contents

- [DDD and Clean Architecture](#ddd-and-clean-architecture)
  - [How to read this](#how-to-read-this)
  - [They are not the same thing](#they-are-not-the-same-thing)
  - [Strategic design](#strategic-design)
    - [Ubiquitous language](#ubiquitous-language)
    - [Bounded contexts](#bounded-contexts)
    - [Context map](#context-map)
    - [Shared kernel, published language, anti-corruption](#shared-kernel-published-language-anti-corruption)
    - [When this is the wrong tool](#when-this-is-the-wrong-tool)
  - [Domain model](#domain-model)
    - [Value objects](#value-objects)
    - [Entities and identity](#entities-and-identity)
    - [Aggregates](#aggregates)
    - [State transitions](#state-transitions)
    - [Reference by identity, one transaction](#reference-by-identity-one-transaction)
    - [Loading must not replay factories](#loading-must-not-replay-factories)
    - [Concurrency](#concurrency)
    - [Domain services and policies](#domain-services-and-policies)
    - [Domain events on the aggregate](#domain-events-on-the-aggregate)
  - [Where a rule lives](#where-a-rule-lives)
    - [The same rule in three layers](#the-same-rule-in-three-layers)
    - [Uniqueness is a constraint, not a method](#uniqueness-is-a-constraint-not-a-method)
    - [Authorization is not a domain rule](#authorization-is-not-a-domain-rule)
  - [The dependency rule](#the-dependency-rule)
    - [Allowed references](#allowed-references)
    - [The host is the composition root](#the-host-is-the-composition-root)
    - [Enforce the rule with a test](#enforce-the-rule-with-a-test)
    - [One project, four folders](#one-project-four-folders)
  - [Project layout](#project-layout)
    - [Feature folders inside a layer](#feature-folders-inside-a-layer)
    - [Project files](#project-files)
  - [Hexagonal architecture](#hexagonal-architecture)
    - [The inside and the adapters](#the-inside-and-the-adapters)
    - [Driving and driven](#driving-and-driven)
    - [What this sample already is](#what-this-sample-already-is)
    - [One use case, two driving adapters](#one-use-case-two-driving-adapters)
    - [One port, two driven adapters](#one-port-two-driven-adapters)
    - [Where the interface lives](#where-the-interface-lives)
    - [What people draw instead](#what-people-draw-instead)
  - [Application layer](#application-layer)
    - [Use-case shape](#use-case-shape)
    - [Ports](#ports)
    - [Edge validation](#edge-validation)
    - [Idempotency](#idempotency)
    - [Two aggregates in one context](#two-aggregates-in-one-context)
    - [Expected failure versus invariant failure](#expected-failure-versus-invariant-failure)
    - [A pipeline without a mediator](#a-pipeline-without-a-mediator)
    - [Registration](#registration)
  - [Infrastructure and EF Core](#infrastructure-and-ef-core)
    - [One DbContext per bounded context](#one-dbcontext-per-bounded-context)
    - [Mapping](#mapping)
    - [Value converters](#value-converters)
    - [Backing fields and owned lines](#backing-fields-and-owned-lines)
    - [Concurrency token](#concurrency-token)
    - [Global query filters](#global-query-filters)
    - [Repository](#repository)
    - [Do not leak IQueryable](#do-not-leak-iqueryable)
  - [The API is an adapter](#the-api-is-an-adapter)
    - [Contracts](#contracts)
    - [Problem details](#problem-details)
    - [Strongly typed ids on the route](#strongly-typed-ids-on-the-route)
  - [Reads do not go through the aggregate](#reads-do-not-go-through-the-aggregate)
    - [Query objects](#query-objects)
    - [Keyset pagination](#keyset-pagination)
    - [A separate read model](#a-separate-read-model)
  - [Events and the outbox](#events-and-the-outbox)
    - [Three event kinds](#three-event-kinds)
    - [Stable event names](#stable-event-names)
    - [Write the outbox in the same transaction](#write-the-outbox-in-the-same-transaction)
    - [An interceptor so a stray SaveChanges still records events](#an-interceptor-so-a-stray-savechanges-still-records-events)
    - [Publish without holding the row lock](#publish-without-holding-the-row-lock)
    - [Inbox](#inbox)
    - [Poison rows](#poison-rows)
  - [Anti-corruption layer](#anti-corruption-layer)
  - [A payment process](#a-payment-process)
  - [Testing](#testing)
    - [Aggregate tests](#aggregate-tests)
    - [Handler tests](#handler-tests)
    - [Architecture test](#architecture-test)
  - [Tenancy and time](#tenancy-and-time)
    - [Tenant comes from the host, not the body](#tenant-comes-from-the-host-not-the-body)
    - [Time and ids are inputs](#time-and-ids-are-inputs)
  - [Modular monolith](#modular-monolith)
    - [One process, many modules](#one-process-many-modules)
    - [What a module owns](#what-a-module-owns)
    - [How modules talk](#how-modules-talk)
    - [The host composes modules](#the-host-composes-modules)
    - [Enforce the module boundary](#enforce-the-module-boundary)
    - [When to split a module out](#when-to-split-a-module-out)
  - [Vertical slice](#vertical-slice)
    - [A slice is one change](#a-slice-is-one-change)
    - [Slice inside a module](#slice-inside-a-module)
    - [Registration stays with the slice](#registration-stays-with-the-slice)
    - [What slices share](#what-slices-share)
  - [Worked examples](#worked-examples)
    - [A place request, end to end](#a-place-request-end-to-end)
    - [Ship commits the sale](#ship-commits-the-sale)
    - [Mapping failures](#mapping-failures)
    - [Do not share a transaction across modules](#do-not-share-a-transaction-across-modules)
    - [Two writers, one row](#two-writers-one-row)
  - [What to skip](#what-to-skip)
  - [Checklist](#checklist)
  - [Related guides](#related-guides)

# DDD and Clean Architecture

Domain-Driven Design (DDD) is how you model one business area: the words people use, the rules that must always be true, and the cluster of objects you save in one transaction. Clean Architecture and hexagonal architecture (ports and adapters) are how you keep that model from taking a reference on ASP.NET Core, EF Core, or a payment vendor's SDK. A modular monolith is several of those models in one process. A vertical slice is one use case laid out in one folder inside a module.

A solution can use either idea alone. A CRUD settings screen does not need an aggregate. A rich `Order` type can live in the same project as its `DbContext` and still be DDD. The failure this guide is about is the common one: four projects named Domain, Application, Infrastructure, and Api, and the rule `a shipped order cannot be cancelled` still sitting in a controller as `order.Status = "Cancelled"`.

The running example is an ordering bounded context:

- place an order at the **catalog price**, not a price posted by the client
- keep a **snapshot** of the product name and unit price on the line (the catalog can change tomorrow)
- reserve stock in the same context
- refuse a second cancel, a ship-before-pay, and a line with no quantity
- publish `ordering.order_placed.v1` after the row commits
- let a payment message move the order to paid, or release the stock when payment fails

The sample targets .NET 9 (`Guid.CreateVersion7`, primary constructors, `TimeProvider`, EF Core 8+ `ComplexProperty`). Identity types are shown in full below; the shorter treatment and model binding notes also live in [DotnetPattern.md](DotnetPattern.md#strongly-typed-ids).

## How to read this

One ordering context runs from the domain model through payment. A second context sharing the process comes after that. The four projects in [Project layout](#project-layout) are one module. The tree in [Modular monolith](#modular-monolith) is several modules in one host. They are two shapes of the same dependency rule, not two samples you stack.

| If you are here to... | Start at |
|------------------------|----------|
| Move a rule out of a controller | [Domain model](#domain-model) and [Where a rule lives](#where-a-rule-lives) |
| Decide what a project may reference | [The dependency rule](#the-dependency-rule) and [Hexagonal architecture](#hexagonal-architecture) |
| Follow place, cancel, pay, and ship | [Application layer](#application-layer) through [A payment process](#a-payment-process) |
| Put Billing beside Ordering in one process | [Modular monolith](#modular-monolith) |
| Keep one use case in one folder | [Vertical slice](#vertical-slice) |
| See one request on a timeline | [Worked examples](#worked-examples) |

[What to skip](#what-to-skip) and the [checklist](#checklist) repeat the same rules in short form.

## They are not the same thing

| | DDD | Clean Architecture | Hexagonal |
|--|-----|--------------------|------------|
| Question it answers | What is an order, and what is allowed to change together? | Which project is allowed to reference EF Core? | How does a use case call PostgreSQL and Stripe without naming them? |
| Main ideas | Ubiquitous language, bounded context, entity, value object, aggregate | Concentric circles. Source dependencies point inward | A port for each conversation. An adapter per technology |
| Unit of design | One business capability | One direction of references | One port, swapped adapters |
| Failure mode | A `CustomerService` full of `if`s and entities with public setters | A `Domain` project that references `Microsoft.EntityFrameworkCore` | A handler whose constructor takes `DbContext` and a vendor client |
| You can stop early | A single rich type plus a unique index is often enough | Folders with the right references are enough until a second adapter exists | One interface for each thing you actually replace in a test or a deploy |

Tactical patterns (entity, value object, aggregate, repository, domain event) are the part most .NET samples copy. Strategic design (which boundary the word `Order` belongs to) is the part that decides whether those patterns help. Hexagonal names are the part that decides whether `PlaceOrderHandler` is allowed to see `Npgsql` or `StripeClient`.

## Strategic design

### Ubiquitous language

The names in the code are the names the business uses for this context. If support says "we cancel a placed order" and "we void a paid order", those are two transitions, not one `Status = "Cancelled"` flag with a comment.

Write the words into types and methods:

| They say | In code | Avoid |
|----------|---------|--------|
| Place an order | `Order.Place` | `OrderService.Create` |
| Reserve stock | `Stock.Reserve` | `inventory.qty -= n` in a controller |
| Capture payment | `Order.MarkPaid` | `order.IsPaid = true` |
| Ship | `Order.Ship` | `UPDATE orders SET status = 3` |

A glossary markdown file that the code does not use is not ubiquitous language. The compiler is the glossary. `OrderStatus.Paid` is a word. `"P"` in a `char` column is a private joke the next reader will not get. Map the short column in EF (`HasConversion<string>()`), and keep the enum name as the word.

Rename when the business corrects you. `Submit` that support calls `Place` will be searched for and not found during an incident.

### Bounded contexts

A bounded context is a boundary where a word means one thing and a model is consistent. `Order` in ordering is a checkout document: lines, a snapshotted price, a status, a reserve against stock. `Order` in billing, if they even use that word, is an invoice with tax lines and a due date. They are not two projections of one class.

They share a **customer id**, not a `Customer` entity, not a navigation, and not one `DbContext`.

```text
Ordering context                         Billing context
-------------------                      -------------------
Order                                    Invoice
  OrderId                                  InvoiceId
  CustomerId  ── same Guid value ──►       CustomerId
  lines, status, stock reserve             tax lines, due date
schema: ordering.*                       schema: billing.*
```

Inside one process (a modular monolith) this is still two models:

- `Ordering.Domain.Order` and `Billing.Domain.Invoice`
- `OrderingDbContext` and `BillingDbContext`
- a PostgreSQL schema per context (`ordering`, `billing`), or two databases when a team boundary is real

A shared `AppDbContext` with `DbSet<Order>` and `DbSet<Invoice>` and a navigation `Order.Invoice` couples the migrations. Billing cannot ship a column change without ordering's model snapshot. That is one context pretending to be two.

### Context map

A context map is the list of relationships between contexts. You do not need a diagram tool. You need an explicit choice for each neighbor:

| Relationship | Meaning for this codebase | Ordering example |
|--------------|---------------------------|------------------|
| Separate ways | No calls, no shared types | A reporting warehouse reads a replica you do not code against |
| Customer / supplier | Downstream conforms to upstream's published events | Billing consumes `ordering.order_placed.v1` and does not get a vote on the payload every week |
| Open host / published language | You publish a stable contract for many consumers | The integration event, versioned, documented |
| Anti-corruption layer | You translate their model into yours at the edge | A payment vendor's `"CAPTURED"` becomes `Order.MarkPaid` |
| Shared kernel | A tiny shared library both teams change together | A `CustomerId` struct, if and only if both sides agree to version it |
| Partnership | Two teams ship one change as one release | Rare. Treat it as a temporary shared kernel |
| Conformist | You accept their model as yours | Their DTO becomes your entity. This is how payment fields leak into `Order` |
| Big ball of mud | One project, every feature references every other | The thing the rest of this guide is here to avoid |

The map is a decision about **source references and message contracts**. If ordering's csproj references `PaymentProvider.Sdk`, the vendor's types will show up in `Order` within a quarter.

### Shared kernel, published language, anti-corruption

**Shared kernel.** A project both contexts reference. Keep it boring and stable: `CustomerId`, `Money` only if both contexts truly share the same currency rules. The moment ordering wants `Money` to reject negative amounts and billing wants a credit memo with a negative amount, `Money` splits. Duplicating a 15-line struct is cheaper than a meeting every time one side changes it.

**Published language.** The integration event schema, not the C# domain event. Consumers bind to `ordering.order_placed.v1` with plain `Guid` and `decimal` fields. They do not reference `Ordering.Domain`.

**Anti-corruption layer (ACL).** A translator at the edge. The vendor (or the other context) has a model. You map it into commands your aggregate already understands. The mapping lives in infrastructure or in the API adapter. `Order` never sees `CaptureResponse`.

```C#
public static class PaymentTranslation
{
    public static PaymentOutcome ToOutcome(string vendorStatus) => vendorStatus switch
    {
        "CAPTURED" => PaymentOutcome.Paid,
        "DECLINED" or "CANCELLED" => PaymentOutcome.Failed,
        _ => PaymentOutcome.Unknown
    };
}

public enum PaymentOutcome { Paid, Failed, Unknown }
```

`Unknown` is a value. A `default` arm that calls `MarkPaid` will mark orders paid when the vendor adds `"PENDING_CAPTURE"`.

### When this is the wrong tool

Skip the aggregate when all of these are true:

- the screen is create / edit / delete of fields the user owns
- there is no transition ("a shipped order cannot be cancelled")
- two users editing the row is solved by "last write wins" and that is acceptable
- no other context needs a reliable "this happened" message

A warehouse location admin page is a table and a form. Wrapping it in `Location.Rename` plus four projects adds files and no invariant. Use EF and a validated request DTO.

Use the aggregate when a bug in a status flag costs a shipment, a double charge, or a negative stock count. The patterns below exist to make those transitions hard to skip.

## Domain model

The domain project contains the rules and the words. It has no `Save`, no `HttpClient`, and no attribute from EF. EF learns the shape later, in infrastructure, by convention and `IEntityTypeConfiguration<T>`.

### Value objects

A value object has no identity. Two `Money` values with the same amount and currency are interchangeable. Validate in the constructor so an illegal value cannot be represented. Equality is by the components (a `readonly record struct` gives you that).

```C#
public readonly record struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentOutOfRangeException(nameof(amount), "Amount cannot be negative.");
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        if (currency.Length != 3)
            throw new ArgumentException("Currency is a 3-letter code.", nameof(currency));

        Amount = decimal.Round(amount, 2, MidpointRounding.ToEven);
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (!string.Equals(Currency, other.Currency, StringComparison.Ordinal))
            throw new DomainRuleException($"Cannot add {Currency} to {other.Currency}.");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Times(int quantity)
    {
        if (quantity < 0)
            throw new ArgumentOutOfRangeException(nameof(quantity));
        return new Money(Amount * quantity, Currency);
    }
}
```

Rounding lives at the boundary where money is created, so two paths do not store `10.005` and `10.01` for the same price. `ToEven` is banker's rounding. If finance wants half-up, that decision is this constructor, not `Math.Round` scattered in handlers.

`Amount + other.Amount` can be one cent off from a per-line round if you sum unrounded values. `Times` rounds again inside `new Money(...)`. Call `Times` per line, then `Add` the line totals. Do not sum raw `decimal`s and round once at the end unless finance asked for that and you have a test that pins it.

#### Address is copied onto the order

A customer address that the order navigates to will change when the customer moves. The shipment still had to go to the old door.

```C#
public readonly record struct Address
{
    public string Line1 { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string line1, string city, string postalCode, string country)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(line1);
        ArgumentException.ThrowIfNullOrWhiteSpace(city);
        ArgumentException.ThrowIfNullOrWhiteSpace(postalCode);
        ArgumentException.ThrowIfNullOrWhiteSpace(country);
        if (country.Trim().Length != 2)
            throw new ArgumentException("Country is a 2-letter code.", nameof(country));

        Line1 = line1.Trim();
        City = city.Trim();
        PostalCode = postalCode.Trim();
        Country = country.Trim().ToUpperInvariant();
    }
}
```

`Order` stores an `Address` value captured at `Place`. Later edits to the customer profile do not write through to old orders.

#### ❌ BAD — a value with identity, or an entity with no behavior

```C#
public sealed class Money
{
    public int Id { get; set; }          // a row per amount; 10.00 USD is not a new thing every time
    public decimal Amount { get; set; }  // negative is representable
    public string Currency { get; set; } = "";
}
```

An `Id` on money means two 10 USD amounts are different objects. You will write `money.Id` joins you do not need. Public setters mean the next mapper sets `Amount = -1` after the constructor ran.

#### Record struct, record class, and collections

Prefer `readonly record struct` for small values (`Money`, `Address`, `OrderId`): value equality, no heap allocation per id. Prefer a `sealed class` when EF owns a collection of them (`OrderLine`), because owned-entity collections are reference types with a private constructor EF can hit.

Do not put a mutable `List<T>` inside a value object and still call it a value. Equality will not look at the list the way you think if you hand-roll it, and callers can mutate the list after you "copy" the value. Expose `IReadOnlyList<T>` from the aggregate instead, and keep the `List<T>` private there.

`record class` for `Money` also works and is easier to spot in a debugger as a reference. It allocates. For money created in a loop over lines, the struct is the calmer default. Be consistent inside one context so EF configuration does not mix converter styles at random.

### Entities and identity

An entity is defined by its id. Two orders with the same lines and different ids are different orders. `OrderId` and `CustomerId` are different types so `GetAsync(customerId)` does not compile.

```C#
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.CreateVersion7());

    public static bool TryParse(string? text, IFormatProvider? provider, out OrderId id)
    {
        if (Guid.TryParse(text, out var value))
        {
            id = new OrderId(value);
            return true;
        }

        id = default;
        return false;
    }

    public override string ToString() => Value.ToString();
}

public readonly record struct CustomerId(Guid Value);
public readonly record struct ProductId(Guid Value);
public readonly record struct UserId(Guid Value);

public sealed class DomainRuleException(string message) : Exception(message);
```

`TryParse(string?, IFormatProvider?, out OrderId)` is the overload minimal APIs bind for route and query values. A JSON body still needs a `JsonConverter<OrderId>` (shown with the API). `Guid.CreateVersion7()` is .NET 9+. On .NET 8 use `Guid.NewGuid()`.

Generate the id in the domain factory, before the insert. The handler then knows `order.Id` and can write the idempotency row in the same commit. A database `IDENTITY` / `gen_random_uuid()` default hides the id until `SaveChanges` returns, which splits "I reserved this id" from "I inserted". Database-generated integer keys are fine for a CRUD table. They are awkward for an aggregate that emits `OrderPlaced(order.Id)` before commit: you want that id stable even if the commit fails and you must not reuse a half-published id. A UUIDv7 generated in the factory is stable and sorts by time. See [Database_Indexing.md](Database_Indexing.md) for why random UUIDv4 as a clustered key splits pages.

#### Child identity inside the aggregate

`OrderLine` is not an aggregate. It has no repository. It is saved only with its `Order`. It still needs a way to say "this line" when the user changes a quantity.

If the business rule is "one line per product", `ProductId` is the identity inside the order. `ChangeQuantity(ProductId, int)` finds the line. A separate `LineId` is extra surface.

If the business rule is "the same product can appear twice" (two shipments, two prices), `ProductId` is not unique inside the order and a `LineId` is the identity. Pick the rule the business actually has. The sample uses one line per product.

```C#
public sealed class OrderLine
{
    private OrderLine() { }

    public OrderLine(ProductId productId, string productName, int quantity, Money unitPrice)
    {
        if (quantity <= 0)
            throw new DomainRuleException("Quantity must be positive.");
        ArgumentException.ThrowIfNullOrWhiteSpace(productName);

        ProductId = productId;
        ProductName = productName.Trim();
        Quantity = quantity;
        UnitPrice = unitPrice;
    }

    public ProductId ProductId { get; private set; }
    public string ProductName { get; private set; } = "";
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }

    public Money LineTotal => UnitPrice.Times(Quantity);

    internal void ChangeQuantity(int quantity)
    {
        if (quantity <= 0)
            throw new DomainRuleException("Quantity must be positive.");
        Quantity = quantity;
    }
}
```

`ProductName` and `UnitPrice` are a snapshot. They are copied from the catalog at the moment of `Place`. The line does not hold a `Product` navigation. `internal ChangeQuantity` means only `Order` (same assembly) changes a line. A handler cannot do `line.Quantity = 0`.

The private parameterless constructor is for EF Core materialization. Application code uses the public constructor.

### Aggregates

An aggregate is the consistency boundary. It is the cluster you load and save together so a rule stays true without a lock on the rest of the database. `Order` is a root. Its lines are inside it. `Stock` is a different root: many orders reserve the same SKU, and locking the SKU row is a different contention point than locking one order.

Rules of thumb that hold up in this kind of system:

1. A transaction changes **one aggregate**, or a small set you have explicitly accepted (the sample's place-and-reserve is that exception, and it is discussed below).
2. Reach another aggregate by id (`CustomerId`, `ProductId`), not by a navigation you lazy-load.
3. Keep the cluster small enough that two clerks are not serializing on the same row for unrelated edits. An `Order` that also contains the customer's profile, the product catalog, and the year's invoices will deadlock.
4. Invariants that span roots ("this customer's lifetime spend") are eventually consistent or they are a different aggregate. They are not a join in `Order.Place`.

```C#
public enum OrderStatus
{
    Placed,
    Paid,
    Shipped,
    Cancelled,
    PaymentFailed
}

public sealed class Order : IHasDomainEvents
{
    private readonly List<OrderLine> _lines = [];
    private readonly List<IDomainEvent> _events = [];

    private Order() { }

    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShipTo { get; private set; }
    public string? IdempotencyKey { get; private set; }
    public DateTimeOffset PlacedAt { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events;

    public Money Total => _lines
        .Select(line => line.LineTotal)
        .Aggregate(new Money(0, Currency), static (sum, line) => sum.Add(line));

    public string Currency => _lines.Count == 0
        ? throw new DomainRuleException("An order has no currency until it has a line.")
        : _lines[0].UnitPrice.Currency;

    public static Order Place(
        OrderId id,
        CustomerId customerId,
        Address shipTo,
        IReadOnlyList<OrderLine> lines,
        string? idempotencyKey,
        DateTimeOffset now)
    {
        ArgumentNullException.ThrowIfNull(lines);
        if (lines.Count == 0)
            throw new DomainRuleException("An order needs at least one line.");

        var order = new Order
        {
            Id = id,
            CustomerId = customerId,
            ShipTo = shipTo,
            Status = OrderStatus.Placed,
            IdempotencyKey = idempotencyKey,
            PlacedAt = now
        };

        foreach (var line in lines)
            order.AddLine(line);

        order.Raise(new OrderPlaced(Guid.CreateVersion7(), id, now));
        return order;
    }

    public void AddLine(OrderLine line)
    {
        EnsureStatus(OrderStatus.Placed, "Lines can change only on a placed order.");
        if (_lines.Any(existing => existing.ProductId == line.ProductId))
            throw new DomainRuleException("A product appears on at most one line.");
        if (_lines.Count > 0 && _lines[0].UnitPrice.Currency != line.UnitPrice.Currency)
            throw new DomainRuleException("An order uses one currency.");

        _lines.Add(line);
    }

    public void ChangeQuantity(ProductId productId, int quantity, DateTimeOffset now)
    {
        EnsureStatus(OrderStatus.Placed, "Quantity can change only on a placed order.");
        var line = _lines.SingleOrDefault(l => l.ProductId == productId)
            ?? throw new DomainRuleException("That product is not on the order.");
        line.ChangeQuantity(quantity);
        Raise(new OrderQuantityChanged(Guid.CreateVersion7(), Id, productId, quantity, now));
    }

    public void MarkPaid(DateTimeOffset now)
    {
        EnsureStatus(OrderStatus.Placed, "Only a placed order can be paid.");
        Status = OrderStatus.Paid;
        Raise(new OrderPaid(Guid.CreateVersion7(), Id, now));
    }

    public void MarkPaymentFailed(DateTimeOffset now)
    {
        EnsureStatus(OrderStatus.Placed, "Only a placed order can fail payment.");
        Status = OrderStatus.PaymentFailed;
        Raise(new OrderPaymentFailed(Guid.CreateVersion7(), Id, now));
    }

    public void Ship(DateTimeOffset now)
    {
        EnsureStatus(OrderStatus.Paid, "Only a paid order can ship.");
        if (_lines.Count == 0)
            throw new DomainRuleException("Cannot ship an empty order.");
        Status = OrderStatus.Shipped;
        Raise(new OrderShipped(Guid.CreateVersion7(), Id, now));
    }

    public void Cancel(DateTimeOffset now)
    {
        EnsureStatus(OrderStatus.Placed, "Only a placed order can be cancelled.");
        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelled(Guid.CreateVersion7(), Id, now));
    }

    public void ClearEvents() => _events.Clear();

    private void EnsureStatus(OrderStatus required, string message)
    {
        if (Status != required)
            throw new DomainRuleException(message);
    }

    private void Raise(IDomainEvent domainEvent) => _events.Add(domainEvent);
}
```

`Place` calls `AddLine`, so the "one product, one currency" rules exist once. A later `AddLine` from a handler on a paid order fails `EnsureStatus`. There is no public `List` to `Add` into.

`Total` folds line totals that are already rounded. The seed `new Money(0, Currency)` uses the first line's currency, so an empty order throws a domain exception instead of inventing `"USD"`.

#### ❌ BAD — anemic model

The type is a row. Any caller can set any status, including a typo the database will store if the column is a string without a check constraint.

```C#
public sealed class Order
{
    public Guid Id { get; set; }
    public string Status { get; set; } = "Placed";
    public List<OrderLine> Lines { get; set; } = [];
}

public sealed class OrderService(OrderingDbContext db)
{
    public async Task CancelAsync(Guid id, CancellationToken cancellationToken)
    {
        var order = await db.Orders.SingleAsync(o => o.Id == id, cancellationToken);
        order.Status = "Cancelled";
        await db.SaveChangesAsync(cancellationToken);
    }
}
```

The service is also untestable without a database, and a second endpoint can assign `Status` and never call `CancelAsync`.

#### ✅ GOOD — the transition is the API

`Status` has a private setter. Tests call `Cancel` and assert the exception. EF is configured later to use the backing field and does not need a public setter.

### State transitions

Write the allowed moves down so a new method cannot invent `Shipped → Placed`.

```text
Placed ──MarkPaid──────────► Paid ──Ship──► Shipped
   │
   ├──Cancel────────────────► Cancelled
   └──MarkPaymentFailed─────► PaymentFailed
```

`Cancelled` and `Shipped` and `PaymentFailed` have no outgoing business transition in this slice. A refund is a new method with its own rule (`Paid` or `Shipped`, and not already refunded), not a reuse of `Cancel`.

#### ❌ BAD — a boolean pile

```C#
public bool IsCancelled { get; set; }
public bool IsPaid { get; set; }
public bool IsShipped { get; set; }
```

`IsCancelled && IsShipped` is representable. Every reader re-derives the state machine, differently. One enum and methods that set it are easier to test and easier to map to a check constraint:

```sql
ALTER TABLE ordering.orders
  ADD CONSTRAINT ck_orders_status
  CHECK (status IN ('Placed', 'Paid', 'Shipped', 'Cancelled', 'PaymentFailed'));
```

The check constraint is the backstop when someone runs SQL in a console. The enum is the backstop when someone writes C#. `ChangeQuantity` takes `DateTimeOffset now` the same way `Cancel` does. The aggregate does not call `TimeProvider` or `DateTime.UtcNow`. The handler reads the clock once per use case and passes that value in. Tests pass a fixed `DateTimeOffset`.

### Reference by identity, one transaction

`Order` holds `CustomerId`. It does not hold `Customer`. Loading a customer graph to place an order locks and copies data the order does not invariant-check. The handler checks "customer exists" through a port if it must, then passes the id in.

```C#
public interface ICustomerDirectory
{
    Task<bool> ExistsAsync(CustomerId id, CancellationToken cancellationToken);
}
```

That port is application-level. The directory's implementation reads the customer context (HTTP, or a replica, or a table this context is allowed to read). A missing customer is `NotFoundException` or a domain exception you chose for "we do not sell to unknown ids". It is not `order.Customer.Name`.

**One transaction, one root** is the default because the aggregate is the lock boundary. Two clerks paying two orders do not block each other.

**Place and reserve** in this sample touch `Order` and `Stock` in one commit on purpose. They sit in the same bounded context, the same database, and the business rule is "we do not create an order we could not reserve". A saga would leave a `Placed` order with no reserve and a worker to repair it. That machinery is justified when stock lives in another service. It is ceremony when both rows are in schema `ordering`.

The cost is a hot `stock` row for a popular SKU. Keep the transaction short: load stock, `Reserve`, commit. Do not call the payment vendor while that row lock is held. Payment happens after commit, from the outbox or from the next request.

```C#
public sealed class Stock : IHasDomainEvents
{
    private readonly List<IDomainEvent> _events = [];

    private Stock() { }

    public ProductId ProductId { get; private set; }
    public int OnHand { get; private set; }
    public int Reserved { get; private set; }
    public int Available => OnHand - Reserved;
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events;

    public static Stock Receive(ProductId productId, int onHand)
    {
        if (onHand < 0)
            throw new DomainRuleException("On-hand cannot be negative.");
        return new Stock { ProductId = productId, OnHand = onHand };
    }

    public void ClearEvents() => _events.Clear();

    public void Reserve(int quantity, OrderId orderId, DateTimeOffset now)
    {
        if (quantity <= 0)
            throw new DomainRuleException("Quantity must be positive.");
        if (quantity > Available)
            throw new DomainRuleException("Not enough stock.");

        Reserved += quantity;
        _events.Add(new StockReserved(Guid.CreateVersion7(), ProductId, orderId, quantity, now));
    }

    public void Release(int quantity, OrderId orderId, DateTimeOffset now)
    {
        if (quantity <= 0)
            throw new DomainRuleException("Quantity must be positive.");
        if (quantity > Reserved)
            throw new DomainRuleException("Cannot release more than reserved.");

        Reserved -= quantity;
        _events.Add(new StockReleased(Guid.CreateVersion7(), ProductId, orderId, quantity, now));
    }
}
```

`Stock` does not store every reservation as a child list in this slice. `Reserved` is a counter. That loses "which order reserved" inside the aggregate. `StockReserved` carries `OrderId` for the outbox, and the order lines are the record of what this order asked for. If you must cancel a single reserve without trusting the order lines, store a `Reservation(OrderId, Quantity)` collection inside `Stock` and make `Release` find it. That collection grows with every open order for the SKU. For a busy SKU, a counter plus the order as the source of "how many" is the smaller aggregate. Write the choice down next to `Reserve` so the next person does not add a collection of every historical reservation "for audit" and turn the hot row into a megabyte.

`Stock` implements `IHasDomainEvents` so the same interceptor that copies `Order` events can copy `StockReserved` once you give it a mapper. `ClearEvents` is for that interceptor after a successful save. Handlers do not call it.

Negative `Available` is the invariant. A raw `UPDATE stock SET reserved = reserved + @q` in a controller will go negative under two concurrent checkouts that both read `Available == 1`. The concurrency token later in this guide closes the lost update. `Reserve` closes the illegal state in memory. You need both: the method so a single caller cannot reserve a negative, the token so two callers cannot both pass the check on a stale `Available`.

### Loading must not replay factories

EF Core materializes `Order` by calling the private parameterless constructor and setting fields. It does **not** call `Place`. That is what you want. `Place` emits `OrderPlaced`. Re-loading an order on cancel must not emit `OrderPlaced` again.

Consequences:

- Invariants in `Place` are not re-checked on load. A row that was corrupted by SQL stays corrupted until a method runs. The check constraint and the private setters are why you do not offer a public `SetStatus`.
- Do not put the "raise OrderPlaced" logic in a property setter. EF sets properties on load and you will publish a ghost event every read that happens to track the entity.
- A factory that always increments a counter in static state will not see loads. Fine. A factory that assumes it is the only way to get an `Order` is wrong about EF.

```C#
private Order() { } // EF. Does not validate. Does not raise.

public static Order Place(...) { /* validates, raises once */ }
```

If you delete the private constructor and leave only `Place`'s object initializer path, EF still needs a constructor it can call. A single public constructor with parameters can be used by EF Core as a binding constructor. Arguments passed by EF bypass your `if` checks only if you put the checks after assignment in a way EF's constructor binding skips. Keep checks in the constructor body, and keep a private empty constructor so EF's path and the factory path stay obvious. Then configure the factory as the only method application code calls. Analyzers will not save you from `new Order()` if the empty constructor is public. Keep it private.

### Concurrency

Two requests load the same `Placed` order. Both call `Cancel`. Both call `SaveChanges`. Without a token, both updates write `Cancelled` and you might send two emails. With a token, the second update throws `DbUpdateConcurrencyException`. Map that to **409**.

The domain does not catch that exception. The domain has no idea a concurrent transaction exists. Infrastructure throws. The API maps it. The handler can also catch it when a retry policy lives in the application layer. Retrying `Cancel` is safe: the second attempt loads `Cancelled` and `Cancel` throws `DomainRuleException`, which you map to 400 ("only a placed order can be cancelled"). Retrying `Place` is not safe unless the idempotency key returns the original id. See the idempotency section.

Optimistic concurrency is the default for orders (conflicts are rare). A hot `Stock` row can use the same token; conflicts mean "retry the reserve". Pessimistic `SELECT ... FOR UPDATE` on stock is reasonable when retries thrash. It belongs in the repository implementation, held only for the length of the transaction, and never across an HTTP call to a payment API. A timed walkthrough of both races is in [Two writers, one row](#two-writers-one-row).

### Domain services and policies

A domain service is stateless business logic that does not naturally sit on one entity, and that does no I/O. If it needs `DbContext` or `HttpClient`, it is an application or infrastructure service with a confusing name.

Prefer a method on `Order` when only that aggregate's data is involved. `Total` is a method (a property here) because it reads lines. A **policy** is a domain service you expect to replace: staff discount, no discount, a voucher.

```C#
public interface IDiscountPolicy
{
    Money DiscountFor(Order order);
}

public sealed class NoDiscount : IDiscountPolicy
{
    public Money DiscountFor(Order order) => new Money(0, order.Currency);
}

public sealed class RateDiscount(decimal rate) : IDiscountPolicy
{
    public Money DiscountFor(Order order)
    {
        if (rate is < 0 or > 0.5m)
            throw new DomainRuleException("Discount rate is out of range.");
        return new Money(order.Total.Amount * rate, order.Currency);
    }
}
```

`IDiscountPolicy` lives in the domain project. Implementations that are pure math live there too. An implementation that reads a promotions HTTP API is **not** this interface. That is an application port that returns a `decimal rate`, and the handler then calls `new RateDiscount(rate)`. The domain stays synchronous and deterministic. Tests pass `new RateDiscount(0.1m)` and assert `DiscountFor`.

Do not name a class `OrderDomainService` and inject five repositories into it. That is the anemic model with a longer name.

### Domain events on the aggregate

A domain event is a fact that already happened inside the model: `OrderPlaced`, `OrderCancelled`. It is raised by the method that made it true. It is not raised by the handler after the fact (the handler will forget on one path) and it is not raised by an EF interceptor that compares `Status` strings (the interceptor does not know why the status changed).

```C#
public interface IDomainEvent
{
    Guid EventId { get; }
    DateTimeOffset OccurredAt { get; }
}

public interface IHasDomainEvents
{
    IReadOnlyCollection<IDomainEvent> DomainEvents { get; }
    void ClearEvents();
}

public sealed record OrderPlaced(Guid EventId, OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record OrderPaid(Guid EventId, OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record OrderPaymentFailed(Guid EventId, OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record OrderShipped(Guid EventId, OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record OrderCancelled(Guid EventId, OrderId OrderId, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record OrderQuantityChanged(
    Guid EventId,
    OrderId OrderId,
    ProductId ProductId,
    int Quantity,
    DateTimeOffset OccurredAt) : IDomainEvent;

public sealed record StockReserved(
    Guid EventId,
    ProductId ProductId,
    OrderId OrderId,
    int Quantity,
    DateTimeOffset OccurredAt) : IDomainEvent;

public sealed record StockReleased(
    Guid EventId,
    ProductId ProductId,
    OrderId OrderId,
    int Quantity,
    DateTimeOffset OccurredAt) : IDomainEvent;

public sealed record StockCommitted(
    Guid EventId,
    ProductId ProductId,
    OrderId OrderId,
    int Quantity,
    DateTimeOffset OccurredAt) : IDomainEvent;
```

`EventId` is generated when the fact happens, not when the outbox worker publishes. A retry of the worker resends the same id. Consumers dedupe on it.

`ClearEvents` is for infrastructure after the events have been copied onto the outbox. Application handlers should not call it. If that bothers you, make the method `internal` and add `InternalsVisibleTo` for the infrastructure assembly. Do not make the event list a public `List<T>`.

The aggregate does not dispatch. Dispatch needs handlers, and handlers need I/O, and I/O in `Cancel` sends mail for a transaction that later rolls back.

```C#
// ❌ BAD
public void Cancel(DateTimeOffset now, IEmailSender email)
{
    Status = OrderStatus.Cancelled;
    email.Send(CustomerId, "cancelled");
}
```

Collect the event. Commit the order and the outbox row together. Send mail only from a consumer that read a committed message.

## Where a rule lives

| Rule | Owner | Example |
|------|--------|---------|
| JSON shape, required fields, max length of a string the UI sent | API / edge validator | `Quantity` is an int and is present |
| "This caller may do this" | Application, using the user id the endpoint extracted | Only the buyer or a staff role cancels |
| Business invariant | Aggregate | Quantity is positive; a shipped order cannot be cancelled; one line per product |
| Cross-aggregate step in this context | Application handler | Place the order and reserve stock, then one commit |
| Uniqueness across rows | Database unique index | One idempotency key, one order |
| SQL, transactions, outbox insert | Infrastructure | EF mapping, `SaveChangesAsync` |
| Vendor words | Anti-corruption translator | `"CAPTURED"` → `MarkPaid` |

The aggregate is the authority for invariants. A check in the endpoint is a fast 400 so a bad form does not spend a transaction. If a message consumer calls the same handler, the endpoint check never runs. The handler and the aggregate still have to be right.

### The same rule in three layers

"Quantity must be positive" will try to live in three places. Put the authority in one.

```C#
// Edge: shape. Stops a missing field before you build a command.
public sealed record LineRequest(Guid ProductId, int Quantity);

// Aggregate: the rule that every caller hits, including the consumer and the test.
public OrderLine(ProductId productId, string productName, int quantity, Money unitPrice)
{
    if (quantity <= 0)
        throw new DomainRuleException("Quantity must be positive.");
    // ...
}
```

#### ❌ BAD — the validator is the only copy

```C#
public sealed class PlaceOrderValidator
{
    public void Validate(PlaceOrder command)
    {
        if (command.Lines.Any(l => l.Quantity <= 0))
            throw new ValidationException("Quantity must be positive.");
    }
}

public void AddLine(OrderLine line) => _lines.Add(line); // any in-process caller skips the validator
```

A test that calls `Order.Place` with quantity `0` should fail inside the domain. If it fails only when the test remembers to call `PlaceOrderValidator`, the rule is optional.

Keep a coarse edge check when you want a validation list ("line 2 quantity, line 4 currency") instead of the first exception. The aggregate still checks. Duplicate `if (quantity <= 0)` in both places is acceptable. Duplicate *and the aggregate does not check* is how the consumer corrupts a row.

### Uniqueness is a constraint, not a method

"An email is unique" and "an idempotency key is used once" cannot be enforced by an entity. The entity does not see the other rows. Two concurrent requests can both pass `AnyAsync(e => e.Email == email)` and both insert.

The authority is a unique index. The handler's pre-check exists to return a clean 409 instead of a database exception. On the race, catch the unique violation and translate it.

```C#
builder.HasIndex(o => o.IdempotencyKey).IsUnique();
```

```C#
try
{
    await unitOfWork.CommitAsync(cancellationToken);
}
catch (DbUpdateException ex) when (IsUniqueViolation(ex))
{
    var winner = await idempotency.FindAsync(command.Key, cancellationToken);
    if (winner is not null)
        return winner.OrderId;
    throw;
}
```

`IsUniqueViolation` for PostgreSQL checks SQLSTATE `23505`. For SQL Server it checks error number `2601` or `2627`. That function lives in infrastructure, beside the `DbContext`. The application port can expose `Task<OrderId?> FindExistingAsync(...)` and the repository catches the provider exception, so the handler stays free of `PostgresException`. Either place is fine. The handler referencing `PostgresException` is not.

Do not try to make this a domain service that takes `IOrderRepository`. You would be teaching the domain about query races it cannot close.

### Authorization is not a domain rule

`Order.Cancel` answers "is this order in a state that can be cancelled?". It does not answer "is this user allowed to cancel it?". Roles change without the business invariant changing. Staff can cancel a customer's placed order. The customer can cancel their own. A warehouse user cannot.

```C#
public sealed class CancelOrderHandler(
    IOrderRepository orders,
    IUnitOfWork unitOfWork,
    TimeProvider time)
{
    public async Task HandleAsync(CancelOrder command, CancellationToken cancellationToken)
    {
        if (command.Actor.Role is not (ActorRole.Buyer or ActorRole.Staff))
            throw new ForbiddenException();

        var order = await orders.GetAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException("Order was not found.");

        if (command.Actor.Role == ActorRole.Buyer && order.CustomerId != command.Actor.CustomerId)
            throw new ForbiddenException();

        order.Cancel(time.GetUtcNow());
        await unitOfWork.CommitAsync(cancellationToken);
    }
}
```

The endpoint's authorization policy can reject an anonymous caller earlier. The handler check is the one that also runs when the call arrives on a bus from an internal tool. Passing `Actor` in the command keeps `HttpContext` out of the application project.

`ForbiddenException` maps to 403. `NotFoundException` maps to 404. Returning 404 for someone else's order is a product choice (it leaks less). Pick one and use it in the handler, not a mix where the endpoint returns 404 and the consumer throws 403 for the same fact.

## The dependency rule

Source dependencies point toward the domain. At runtime the host composes everything. At compile time the domain compiles with nothing from this solution except the BCL. This is the [hexagon](#hexagonal-architecture) drawn as projects: adapters on the outside, the model and the use cases on the inside.

```text
Ordering.Api  -----------> Ordering.Application -------> Ordering.Domain
     |                            ^
     +-----> Ordering.Infrastructure
```

### Allowed references

| Project | May reference | Must not reference |
|---------|---------------|--------------------|
| `Ordering.Domain` | BCL | EF Core, ASP.NET Core, vendor SDKs, `Ordering.Infrastructure`, `Ordering.Application` |
| `Ordering.Application` | Domain | EF Core, ASP.NET Core (`HttpContext`, `Results`), infrastructure |
| `Ordering.Infrastructure` | Application, Domain, EF Core, vendor SDK | The HTTP pipeline as a place to put rules |
| `Ordering.Api` | Application, Infrastructure | Domain rules copied into endpoints |
| `Ordering.Domain.Tests` | Domain | EF Core, the web host |

`Ordering.Application` referencing `Microsoft.AspNetCore.Http` so a handler can read `HttpContext.User` is a layer leak. Pass `UserId` in the command. The endpoint reads the claims.

`Ordering.Domain` referencing `MediatR` so `Order.Cancel` can `Publish` is the same leak with a smaller package name.

Logging: the domain throws or returns a result. The handler or the exception middleware logs. An aggregate that takes `ILogger` cannot be new-ed in a unit test without a logger, and the log line becomes part of the business API.

### The host is the composition root

`Ordering.Api` referencing infrastructure is normal. Something has to call `AddDbContext` and bind `IOrderRepository` to `EfOrderRepository`. That something is `Program.cs` (or a worker's `Program.cs`). Endpoints ask for `PlaceOrderHandler`, not for `OrderingDbContext`.

A second host (a `BackgroundService` worker in another executable) is a second composition root. It references infrastructure too. It does not justify moving `AddDbContext` into the application project. If the application project calls `UseNpgsql`, every test and every future host learns the connection string shape.

```C#
// Ordering.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();

builder.Services.AddOrderingApplication();
builder.Services.AddOrderingInfrastructure(builder.Configuration);

var app = builder.Build();
app.UseExceptionHandler();
OrderEndpoints.Map(app);
app.Run();
```

`AddOrderingApplication` lives in the application project and registers handlers. `AddOrderingInfrastructure` lives in infrastructure and is the only method that mentions Npgsql. The extension methods are the seam. See [Add{Feature} extension methods](DotnetPattern.md#addfeature-extension-methods).

### Enforce the rule with a test

A comment in the csproj rots. A test fails in CI the day someone adds `PackageReference Include="Microsoft.EntityFrameworkCore"` to the domain project to "just map one attribute".

```C#
public sealed class DependencyRuleTests
{
    [Fact]
    public void Domain_has_no_framework_references()
    {
        var names = typeof(Order).Assembly
            .GetReferencedAssemblies()
            .Select(a => a.Name)
            .ToArray();

        Assert.DoesNotContain(names, static n => n!.StartsWith("Microsoft.EntityFrameworkCore", StringComparison.Ordinal));
        Assert.DoesNotContain(names, static n => n!.StartsWith("Microsoft.AspNetCore", StringComparison.Ordinal));
        Assert.DoesNotContain(names, static n => n!.StartsWith("Npgsql", StringComparison.Ordinal));
    }

    [Fact]
    public void Application_does_not_reference_infrastructure_or_ef()
    {
        var names = typeof(PlaceOrderHandler).Assembly
            .GetReferencedAssemblies()
            .Select(a => a.Name)
            .ToArray();

        Assert.DoesNotContain(names, static n => n!.Contains("Infrastructure", StringComparison.Ordinal));
        Assert.DoesNotContain(names, static n => n!.StartsWith("Microsoft.EntityFrameworkCore", StringComparison.Ordinal));
    }
}
```

`NetArchTest` and similar packages can also ban a namespace. The reflection test above has no extra dependency and catches the mistake that matters: a package reference. It does not catch `Infrastructure` types used through a transitive reference you did not intend. Keep the csproj references tight and this test is enough.

Run it in the domain test project and the application test project. A test project may reference both sides; production projects may not.

### One project, four folders

Four class libraries pay off when a second adapter exists (HTTP and a worker, or EF and an in-memory fake used by many tests) or when you want the compiler to stop a reference. Until then, folders enforce the same rule only by review:

```text
src/Ordering/
  Domain/
  Application/
  Infrastructure/
  Api/
```

The day `Infrastructure/OrderingDbContext.cs` is `using`-ed from `Domain/Order.cs`, nothing fails the build. Split into projects when that review starts missing things, not on the first afternoon.

## Project layout

```text
src/
  Ordering.Domain/
    Orders/Order.cs
    Orders/OrderLine.cs
    Orders/OrderStatus.cs
    Orders/Events.cs
    Stock/Stock.cs
    Money.cs
    Ids.cs
    DomainRuleException.cs
  Ordering.Application/
    Orders/Place/PlaceOrder.cs
    Orders/Place/PlaceOrderHandler.cs
    Orders/Cancel/CancelOrderHandler.cs
    Orders/Payment/MarkOrderPaidHandler.cs
    OrderingApplication.cs          # AddOrderingApplication
    Ports/IOrderRepository.cs
    Ports/IStockRepository.cs
    Ports/IPriceList.cs
    Ports/IUnitOfWork.cs
    Ports/IIdempotencyStore.cs
  Ordering.Infrastructure/
    Persistence/OrderingDbContext.cs
    Persistence/OrderConfiguration.cs
    Persistence/StockConfiguration.cs
    Persistence/EfOrderRepository.cs
    Persistence/OutboxInterceptor.cs
    Persistence/OutboxProcessor.cs
    Catalog/EfPriceList.cs
    Payments/PaymentTranslation.cs
    OrderingInfrastructure.cs       # AddOrderingInfrastructure
  Ordering.Api/
    Program.cs
    OrderEndpoints.cs
    DomainExceptionHandler.cs
  Ordering.Domain.Tests/
  Ordering.Application.Tests/
```

This tree is **one module, split into projects** so the compiler rejects an EF reference from the domain. [Modular monolith](#modular-monolith) is the other shape: one assembly per bounded context, plus a Contracts assembly, when Ordering and Billing share a process. Do not do both at once (four projects times every module). Start with the tree above for the first context. Switch a module to one assembly when a second context appears and the project count is the problem.

### Feature folders inside a layer

Group by feature under the layer, not by technical kind across the whole project.

#### ❌ BAD — one folder per stereotype

```text
Application/
  Commands/PlaceOrder.cs
  Commands/CancelOrder.cs
  Handlers/PlaceOrderHandler.cs
  Handlers/CancelOrderHandler.cs
  Validators/PlaceOrderValidator.cs
  Interfaces/IOrderRepository.cs
```

Placing an order touches four directories. A new hire greps `PlaceOrder` and gets a pile of similarly named files with no story.

#### ✅ GOOD — one folder per use case

```text
Application/Orders/Place/
  PlaceOrder.cs
  PlaceOrderHandler.cs
  PlaceOrderValidator.cs
Application/Orders/Cancel/
  CancelOrderHandler.cs
```

Ports used by several use cases (`IOrderRepository`, `IUnitOfWork`) stay in `Ports/`. A port used by one use case can sit next to that handler.

The domain project groups by aggregate (`Orders/`, `Stock/`), because the consistency boundary is the reason those files change together. The folder under `Application/Orders/Place/` is one [vertical slice](#vertical-slice). The aggregate stays shared by every slice in the Ordering module.

### Project files

```xml
<!-- Ordering.Domain.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

```xml
<!-- Ordering.Application.csproj -->
<ItemGroup>
  <ProjectReference Include="..\Ordering.Domain\Ordering.Domain.csproj" />
</ItemGroup>
```

```xml
<!-- Ordering.Infrastructure.csproj -->
<ItemGroup>
  <ProjectReference Include="..\Ordering.Application\Ordering.Application.csproj" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.0" />
</ItemGroup>
```

```xml
<!-- Ordering.Api.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\Ordering.Application\Ordering.Application.csproj" />
    <ProjectReference Include="..\Ordering.Infrastructure\Ordering.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

The domain csproj has no `FrameworkReference` for ASP.NET and no EF package. `ImplicitUsings` for `Microsoft.NET.Sdk` does not pull in ASP.NET. The web SDK on the API project does. That split is the point of a separate domain project.

## Hexagonal architecture

Hexagonal architecture is Alistair Cockburn's ports-and-adapters picture from 2005. The application is the inside. Everything technical is an adapter on the outside. The shape is a hexagon so that no side is "the bottom" the way a layered diagram has a database layer at the bottom that the domain is tempted to reference.

It is the same dependency rule as [Clean Architecture](#the-dependency-rule) and as Jeffrey Palermo's onion architecture. Clean Architecture draws four rings (entities, use cases, interface adapters, frameworks). Onion draws domain, domain services, application services, then UI and infrastructure outside. The hexagon draws one inside and many sides. In this solution the inside is `Ordering.Domain` plus `Ordering.Application`. The sides are HTTP, EF Core, the payment vendor, the outbox publisher, and the test fakes.

You do not need six sides, six projects, or six `ports` folders. Six was a drawing that left room for more than the usual two (UI and database). Add a side when a new technology has to talk to the use case.

### The inside and the adapters

```text
                         driving adapters                         driven adapters
                    (call the application)                   (the application calls them)

                    Ordering.Api  endpoints                         EfOrderRepository
                    CancelOrderConsumer (queue)                     EfPriceList / HttpPriceList
                    ExpireUnpaidOrders (timer)          --->  INSIDE  --->   StripePaymentGateway
                    Ordering.Application.Tests                      OutboxProcessor's publisher
                                                                    FakeOrders, FakePriceList

                    INSIDE = Ordering.Domain + Ordering.Application
                    ports live here: PlaceOrderHandler's public method,
                    IOrderRepository, IPriceList, IPaymentGateway
```

A **port** is a small interface (or a concrete use-case class) expressed in the application's words: `Place`, `GetAsync(OrderId)`, `ProductOffer`. An **adapter** is the code that translates between that and a technology: JSON, SQL, a vendor body, a test double.

The inside compiles without ASP.NET Core, EF Core, or the vendor SDK. Adapters reference the inside. The inside does not reference the adapters. That is the whole rule. [The dependency rule](#the-dependency-rule) is this picture flattened into projects.

### Driving and driven

Two directions. Mixing them up is how a "hexagonal" codebase still has `HttpContext` in the handler and `StripeClient` in the domain.

| | Driving (primary) | Driven (secondary) |
|--|-------------------|--------------------|
| Who starts the call | The outside | The use case |
| Examples here | HTTP endpoint, payment webhook, unpaid-order sweeper, a test | `EfOrderRepository`, `HttpPriceList`, payment gateway, integration publisher |
| What the port looks like | The handler's `HandleAsync` | `IOrderRepository`, `IPriceList`, `IPaymentGateway` |
| Who implements it | Nobody. The adapter *calls* it | The adapter class |
| Project in this sample | `Ordering.Api`, plus hosted workers | `Ordering.Infrastructure` |
| Test double | The test method itself | `FakeOrders`, `FakePriceList` |

A driving adapter's job ends when it has built a command and called the handler. Stock, prices, and status transitions are not in the endpoint and not in the consumer. If both the HTTP adapter and the queue adapter check "is there stock?" before calling the handler, the checks will drift and the handler's check is the only one that runs for every caller. See [where a rule lives](#where-a-rule-lives).

A driven adapter's job begins at the port. `IPriceList.FindAsync` returns `ProductOffer`. The HTTP adapter that talks to a catalog service parses JSON inside `HttpPriceList` and does not show that JSON type to `PlaceOrderHandler`. That parser is the [anti-corruption layer](#anti-corruption-layer) for this side of the hexagon. The same idea applies when the vendor calls you: the webhook is a driving adapter, and `PaymentTranslation` sits in that adapter, not on `Order`.

### What this sample already is

The projects in [project layout](#project-layout) are a hexagon whether or not the folders say `Adapters`.

| Hexagon word | In this guide |
|--------------|----------------|
| Inside, application core | `Ordering.Domain`, `Ordering.Application` |
| Primary port | `PlaceOrderHandler.HandleAsync`, `CancelOrderHandler.HandleAsync`, `MarkOrderPaidHandler.HandleAsync` |
| Driving adapter | [The API](#the-api-is-an-adapter), the payment webhook, `ExpireUnpaidOrders` |
| Secondary port | [Ports](#ports): `IOrderRepository`, `IStockRepository`, `IPriceList`, `IUnitOfWork`, `IIdempotencyStore` |
| Driven adapter | `EfOrderRepository`, `EfPriceList`, `EfUnitOfWork`, the outbox publisher |
| Composition root | `Program.cs` and `AddOrderingInfrastructure` |

`IOrderRepository` is a port that also happens to be the DDD repository for one aggregate. `IPriceList` is a port and is not a repository. Do not rename every port to `Repository` to sound like DDD, and do not rename `Order` to `Port` to sound like a hexagon. The aggregate stays the aggregate. The port is how the use case reaches something it does not implement.

### One use case, two driving adapters

`CancelOrderHandler` is the primary port. HTTP and a queue message are two adapters. Neither one contains `order.Cancel`.

```C#
// Driving adapter 1. Ordering.Api. Knows HTTP. Does not know EF.
app.MapPost("/orders/{id}/cancel", async (
    OrderId id,
    HttpContext http,
    CancelOrderHandler handler,
    CancellationToken cancellationToken) =>
{
    await handler.HandleAsync(new CancelOrder(id, http.ToActor()), cancellationToken);
    return Results.NoContent();
});
```

```C#
// Driving adapter 2. A consumer. Knows the message contract. Does not know HTTP.
public sealed class CancelOrderConsumer(CancelOrderHandler handler)
{
    public Task HandleAsync(CancelOrderV1 message, CancellationToken cancellationToken) =>
        handler.HandleAsync(
            new CancelOrder(
                new OrderId(message.OrderId),
                new Actor(new UserId(message.RequestedBy), ActorRole.System, customerId: null)),
            cancellationToken);
}

public sealed record CancelOrderV1(Guid OrderId, Guid RequestedBy);
```

The consumer lives next to the host that reads the queue, not in `Ordering.Domain`. It may live in `Ordering.Api` or in a worker project. It references the application project so it can construct `CancelOrder`. It does not reference `OrderingDbContext`.

Adding `ICancelOrderHandler` as a mirror of the one class does not create a port you did not already have. The public method is the port. Introduce the interface when a second *implementation* of the use case exists, which is rare (a decorator around the handler is that case; see [a pipeline without a mediator](#a-pipeline-without-a-mediator)).

#### ❌ BAD — the rule is copied into each adapter

```C#
app.MapPost("/orders/{id}/cancel", async (Guid id, OrderingDbContext db, CancellationToken ct) =>
{
    var order = await db.Orders.SingleAsync(o => o.Id == new OrderId(id), ct);
    if (order.Status != OrderStatus.Placed)
        return Results.BadRequest();
    order.Cancel(DateTimeOffset.UtcNow);
    await db.SaveChangesAsync(ct);
    return Results.NoContent();
});

public sealed class CancelOrderConsumer(OrderingDbContext db)
{
    public async Task HandleAsync(CancelOrderV1 message, CancellationToken ct)
    {
        var order = await db.Orders.SingleAsync(o => o.Id == new OrderId(message.OrderId), ct);
        order.Status = OrderStatus.Cancelled; // the HTTP path checks Placed; this path does not
        await db.SaveChangesAsync(ct);
    }
}
```

Two driving adapters, zero inside. The queue path skips the invariant. Both adapters also take `OrderingDbContext`, so there is no driven port left to fake.

#### ✅ GOOD — both adapters call the same handler

The HTTP delegate and `CancelOrderConsumer` only build a `CancelOrder`. `CancelOrderHandler` loads, checks the actor, calls `Order.Cancel`, releases stock, and commits. A test calls that same method with a `FakeOrders`. The test is a third driving adapter. See [handler tests](#handler-tests).

### One port, two driven adapters

`IPriceList` is a driven port. The handler asks for a `ProductOffer`. Which technology answers is a composition-root decision.

```C#
public interface IPriceList
{
    Task<ProductOffer?> FindAsync(ProductId productId, CancellationToken cancellationToken);
}
```

```C#
public sealed class HttpPriceList(HttpClient http) : IPriceList
{
    public async Task<ProductOffer?> FindAsync(ProductId productId, CancellationToken cancellationToken)
    {
        using var response = await http.GetAsync($"/products/{productId.Value}", cancellationToken);
        if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
            return null;
        response.EnsureSuccessStatusCode();

        var body = await response.Content.ReadFromJsonAsync<CatalogProductBody>(cancellationToken)
            ?? throw new InvalidOperationException("Catalog returned an empty body.");

        return new ProductOffer(productId, body.Name, new Money(body.Amount, body.Currency));
    }

    private sealed record CatalogProductBody(string Name, decimal Amount, string Currency);
}
```

`CatalogProductBody` is private to the adapter. `PlaceOrderHandler` never sees it. `Money`'s constructor still rejects a negative amount and a bad currency, so a catalog bug fails at the edge of the inside instead of being stored on a line. Register the client the way [HttpClientGuidance.md](HttpClientGuidance.md) describes. Do not `new HttpClient()` inside `FindAsync`.

```C#
public static IServiceCollection AddCatalogPriceList(
    this IServiceCollection services,
    IConfiguration configuration)
{
    if (configuration.GetValue("Catalog:Mode", "Database") == "Http")
    {
        services.AddHttpClient<HttpPriceList>(client =>
        {
            client.BaseAddress = new Uri(configuration["Catalog:BaseUrl"]
                ?? throw new InvalidOperationException("Catalog:BaseUrl is missing."));
        });
        services.AddScoped<IPriceList>(sp => sp.GetRequiredService<HttpPriceList>());
    }
    else
    {
        services.AddScoped<IPriceList, EfPriceList>();
    }

    return services;
}
```

`EfPriceList` is the other driven adapter for the same port. It reads a table or a view and returns the same `ProductOffer`. The handler does not gain an `if (useHttp)` branch. A test registers `FakePriceList` and neither adapter.

The payment gateway is the same shape, aimed outward:

```C#
public interface IPaymentGateway
{
    Task AuthorizeAsync(OrderId orderId, Money amount, CancellationToken cancellationToken);
}

public sealed class StripePaymentGateway(HttpClient http) : IPaymentGateway
{
    public async Task AuthorizeAsync(OrderId orderId, Money amount, CancellationToken cancellationToken)
    {
        var body = new
        {
            amount = ToMinorUnits(amount),
            currency = amount.Currency.ToLowerInvariant(),
            metadata = new { orderId = orderId.Value }
        };

        using var response = await http.PostAsJsonAsync("/v1/payment_intents", body, cancellationToken);
        response.EnsureSuccessStatusCode();
    }
}
```

`ToMinorUnits` stays in this adapter, as in the [anti-corruption](#anti-corruption-layer) section. `Order.MarkPaid` is not waiting on this HTTP call. Authorize runs after the reserve has committed, from a driving side (the request that starts payment) or from the outbox worker. The port method does not return a `PaymentIntent` type from the vendor SDK. If the application needs an outcome, define `AuthorizationResult` next to the port in the application project.

#### ❌ BAD — the driven technology is the port

```C#
public sealed class PlaceOrderHandler(OrderingDbContext db, StripeClient stripe)
{
    public async Task HandleAsync(PlaceOrder command, CancellationToken cancellationToken)
    {
        var price = await stripe.Prices.GetAsync(command.Lines[0].ProductId.ToString(), cancellationToken: cancellationToken);
        db.Orders.Add(new Order { Id = OrderId.New(), Status = OrderStatus.Placed });
        await db.SaveChangesAsync(cancellationToken);
    }
}
```

The constructor *is* a port, and this one says the use case only works with EF and Stripe. A unit test boots both, or it does not exist. Swapping the catalog from a table to HTTP means editing the handler.

### Where the interface lives

| Type | Lives in | Referenced by |
|------|----------|----------------|
| Driving adapter (`OrderEndpoints`, `CancelOrderConsumer`) | Host project | Application |
| Primary port (`CancelOrderHandler`) | Application | Driving adapters, tests |
| Driven port (`IPriceList`) | Application (or Domain, if you chose that for repositories) | Application, driven adapters, tests |
| Driven adapter (`HttpPriceList`, `EfOrderRepository`) | Infrastructure | The composition root only |
| Vendor DTO (`CatalogProductBody`) | Inside the driven adapter, `private` | Nothing else |

The host project is the only one that sees both an adapter and the registration. Endpoints ask for `CancelOrderHandler`, not for `EfOrderRepository`. A controller that injects `EfOrderRepository` has skipped the port and pinned one adapter. Inject the port or the handler.

Driven ports return domain types or small application records (`ProductOffer`, `OrderSummary` on a query port). They do not return `IQueryable`, `DbSet`, `PaymentIntent`, or `HttpResponseMessage`. [Do not leak IQueryable](#do-not-leak-iqueryable).

A driving adapter may use ASP.NET types (`HttpContext`, `Results`, `[FromBody]`). Those types stop at the adapter. The command record has none of them. A minimal API lambda that is 40 lines of pricing logic is a use case wearing an adapter's clothes. Move the lines into the handler.

Query ports and command ports are both driven or driving in the same way. `IOrderQueries` is a driven port used by a driving HTTP adapter for `GET`. The implementation can run SQL without loading `Order`. That does not make the read a second hexagon. It is another port on this one. See [reads](#reads-do-not-go-through-the-aggregate).

### What people draw instead

**A layer cake with the database at the bottom.** `Api → Application → Domain → Infrastructure`, and the domain project takes a reference on EF "because the bottom layer is down there". The hexagon's point is that the database is beside the UI, not under the model. Both are outside. The [allowed references](#allowed-references) table is that correction.

**Six projects because the drawing has six sides.** `Adapters.Http`, `Adapters.Grpc`, `Adapters.Persistence`, `Adapters.Bus`, `Adapters.Email`, `Adapters.Clock` for an application that has HTTP and PostgreSQL. Two adapter projects, or one infrastructure project with folders, is the same architecture. Split an adapter into its own project when it has a heavy SDK you do not want on the web host's compile, or when a second host should not reference it.

**A port per class.** `IOrder`, `IOrderLine`, `IMoney`. Those are the model, not ports. Ports sit at the boundary where a technology can be replaced. `Order.Cancel` is not replaceable by Stripe. `IPaymentGateway` is.

**Interfaces in the infrastructure project, implemented in the infrastructure project, injected as the concrete class anyway.** The handler references `EfOrderRepository`. The interface never constrained anything. Put `IOrderRepository` in the application project or delete it.

**The hexagon as a replacement for the aggregate.** Ports do not keep `Reserved` from going negative. `Stock.Reserve` does. A hexagonal `OrderService` full of public setters is still anemic. DDD says what the inside means. The hexagon says what the inside is allowed to call.

Onion, Clean, and hexagonal diagrams argue about ring count. The review comment that matters is: this type talks to SQL or HTTP, and it is in the domain project. Move it out, or move the rule back in.

## Application layer

A use case loads aggregates, calls methods, and commits. It does not set `Status`. It does not take `HttpContext`. It does not reference EF. The class with `HandleAsync` is the use case. A mediator is optional and is not this layer.

### Use-case shape

One handler, one transaction, a command object that is a `sealed record` of values the handler needs. The command is not the HTTP DTO. The endpoint maps the DTO to the command so a bus consumer can send the same command without JSON attributes on it.

```C#
public sealed record Actor(UserId UserId, ActorRole Role, CustomerId? CustomerId);

public enum ActorRole { Buyer, Staff, System }

public sealed record PlaceOrder(
    CustomerId CustomerId,
    Address ShipTo,
    IReadOnlyList<PlaceOrderLine> Lines,
    string IdempotencyKey,
    Actor Actor);

public sealed record PlaceOrderLine(ProductId ProductId, int Quantity);

public sealed record CancelOrder(OrderId OrderId, Actor Actor);
```

`PlaceOrderLine` carries `ProductId` and `Quantity`. It does not carry a price. The price comes from `IPriceList` inside the handler. A client-supplied price is how a checkout sells a $400 item for $0.01.

`Actor` is who is calling. `ActorRole.System` is the payment consumer. It is allowed to mark paid. It is not a `HttpContext`.

### Ports

These are the driven ports from [hexagonal architecture](#hexagonal-architecture): interfaces the handler needs and infrastructure implements. They return domain types or plain application DTOs. They do not return `IQueryable`, `DbSet`, or a vendor SDK type. The handler's own `HandleAsync` is the driving port. Callers (HTTP, a consumer, a test) use that method and do not need a second interface that only restates the class.

```C#
public interface IOrderRepository
{
    Task AddAsync(Order order, CancellationToken cancellationToken);
    Task<Order?> GetAsync(OrderId id, CancellationToken cancellationToken);
}

public interface IStockRepository
{
    Task<Stock?> GetAsync(ProductId productId, CancellationToken cancellationToken);
}

public interface IPriceList
{
    Task<ProductOffer?> FindAsync(ProductId productId, CancellationToken cancellationToken);
}

public sealed record ProductOffer(ProductId ProductId, string Name, Money UnitPrice);

public interface IUnitOfWork
{
    Task CommitAsync(CancellationToken cancellationToken);
}

public interface IIdempotencyStore
{
    Task<OrderId?> FindAsync(string key, CancellationToken cancellationToken);
}

public sealed class NotFoundException(string message) : Exception(message);
public sealed class ForbiddenException : Exception;
public sealed class ConflictException(string message) : Exception(message);
```

There is no `Update` on `IOrderRepository`. The scoped `DbContext` tracks the instance `GetAsync` returned. `Cancel` mutates it. `CommitAsync` calls `SaveChangesAsync`. An `Update` method that calls `db.Update(order)` marks the whole graph `Modified`, including unchanged lines, and fights the concurrency token.

`IUnitOfWork` exists so the handler does not reference EF, and so "commit" is one word even if later you have an outbox interceptor hanging off `SaveChanges`. One `DbContext` already is the unit of work. A second interface that only forwards `SaveChanges` and does nothing else is optional. Keep it here because the handler must not take `OrderingDbContext`. See [unit of work](DotnetPattern.md#unit-of-work--transaction-scope).

Repository per aggregate root. `IOrderRepository` does not load `Stock`. A generic `IRepository<T>` grows `Include` strings and specification objects until it is EF with extra steps.

Where the interface sits: this guide puts ports in the application project, because the use case is what loads and saves. Putting `IOrderRepository` in the domain project is also common and also fine. Do not put it in both. Do not put EF types on it in either place.

### Edge validation

Validate shape before you touch the database: missing key, empty line list, quantity that is not a number. The framework already does some of this when the JSON does not match `LineRequest`. What is left is rules about the command that are not invariants of a single `Order` (an empty batch, a key longer than the column).

```C#
public static class PlaceOrderRules
{
    public const int MaxLines = 50;

    public static void EnsureShape(PlaceOrder command)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(command.IdempotencyKey);
        if (command.IdempotencyKey.Length > 80)
            throw new DomainRuleException("Idempotency key is too long.");
        if (command.Lines.Count is 0 or > MaxLines)
            throw new DomainRuleException($"An order has 1 to {MaxLines} lines.");
    }
}
```

Throwing `DomainRuleException` for a bad key mixes "your JSON is nonsense" with "a shipped order cannot be cancelled". Both are 400s, so the sample stays small. When you want different titles in problem details, introduce `ValidationException` for shape and keep `DomainRuleException` for the aggregate. Map both to 400 with different `title` values.

FluentValidation and DataAnnotations are edge tools. They are not a reason to skip `if (quantity <= 0)` inside `OrderLine`. A consumer that builds a `PlaceOrder` in C# will not run your HTTP validator.

### Idempotency

The client retries `POST /orders` when the response is lost. Without a key you create two orders and reserve stock twice. The client sends `Idempotency-Key`. You store it on the order (unique index) and return the original `OrderId` on a replay.

```C#
public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    IStockRepository stockItems,
    IPriceList prices,
    IIdempotencyStore idempotency,
    IUnitOfWork unitOfWork,
    TimeProvider time)
{
    public async Task<OrderId> HandleAsync(PlaceOrder command, CancellationToken cancellationToken)
    {
        PlaceOrderRules.EnsureShape(command);

        var existing = await idempotency.FindAsync(command.IdempotencyKey, cancellationToken);
        if (existing is not null)
            return existing.Value;

        var now = time.GetUtcNow();
        var lines = new List<OrderLine>(command.Lines.Count);
        foreach (var requested in command.Lines)
        {
            var offer = await prices.FindAsync(requested.ProductId, cancellationToken)
                ?? throw new NotFoundException($"Unknown product {requested.ProductId}.");
            lines.Add(new OrderLine(offer.ProductId, offer.Name, requested.Quantity, offer.UnitPrice));
        }

        var order = Order.Place(
            OrderId.New(),
            command.CustomerId,
            command.ShipTo,
            lines,
            command.IdempotencyKey,
            now);

        await orders.AddAsync(order, cancellationToken);

        foreach (var line in order.Lines)
        {
            var stock = await stockItems.GetAsync(line.ProductId, cancellationToken)
                ?? throw new DomainRuleException($"No stock row for {line.ProductId}.");
            stock.Reserve(line.Quantity, order.Id, now);
        }

        try
        {
            await unitOfWork.CommitAsync(cancellationToken);
        }
        catch (UniqueConstraintViolationException)
        {
            var winner = await idempotency.FindAsync(command.IdempotencyKey, cancellationToken);
            if (winner is not null)
                return winner.Value;
            throw;
        }

        return order.Id;
    }
}
```

`UniqueConstraintViolationException` is an application exception. Infrastructure catches `DbUpdateException`, checks SQLSTATE `23505` (or SQL Server 2601/2627), and throws this. The handler does not reference Npgsql.

The race: two requests pass `FindAsync` before either commits. Both build an order. One insert wins the unique index. The loser catches, re-reads, and returns the winner's id. The loser's in-memory `Order` and stock reserve are on a `DbContext` that failed `SaveChanges`. Dispose the scope. Do not `Commit` again on that context: tracked `Added` entities are in a bad state. ASP.NET Core disposes the request scope when the exception leaves the handler if you rethrow; because we catch and return, the same context is still alive and still tracking the failed graph.

That is a real bug. After a unique violation, do not return the in-memory id, and do not keep using that `DbContext` to `Find`. Resolve the winner through a query that does not depend on the dirty tracker, or fail the request and let the client retry. The clean version opens the lookup on a context that did not track the loser. `IDbContextFactory` is one way. A simpler way that fits this guide: let the exception propagate as a 409 and tell the client to retry with the same key. The retry hits `FindAsync` and returns the winner. You lose the "transparent replay" on the racing request only.

```C#
catch (UniqueConstraintViolationException)
{
    throw new ConflictException("This idempotency key is already in use. Retry the request.");
}
```

Document the client contract: same key, same body, safe to retry. Same key and a **different** body is a conflict. Compare a hash of the body if you need to detect that, stored next to the key. Returning the old order id for a different cart is a quiet data bug.

### Two aggregates in one context

`PlaceOrderHandler` mutates `Order` and one or more `Stock` roots, then one `CommitAsync`. That is a deliberate widening of "one aggregate per transaction". It is still one `DbContext` and one database transaction. `SaveChanges` wraps the tracked changes.

Do not call `SaveChanges` after the order and again after each stock row unless you want an order that exists with a partial reserve. One commit.

When stock is another service, this handler must not pretend a local method call is a transaction. The flow becomes:

1. Commit the order in `Placed` (or in `Reserving`).
2. Publish `ordering.order_placed.v1`.
3. The stock service reserves and publishes `stock.reserved.v1` or `stock.rejected.v1`.
4. A handler in ordering calls `MarkPayment...` or cancels and records why.

That is a process. It has states for "placed but not yet reserved". The UI cannot assume stock was reserved because HTTP 201 returned. 201 means the order row committed.

Inside one database, prefer the single commit. Across services, prefer the process and the outbox. Mixing them (HTTP call to stock inside the database transaction) holds locks for the length of a network timeout.

`CancelOrderHandler` loads the order, calls `Cancel`, then `Release`s each line's stock, then commits. If `Cancel` throws because the order is `Shipped`, stock is untouched. Call `Cancel` before `Release` so a rejected cancel does not free units.

```C#
public sealed class CancelOrderHandler(
    IOrderRepository orders,
    IStockRepository stockItems,
    IUnitOfWork unitOfWork,
    TimeProvider time)
{
    public async Task HandleAsync(CancelOrder command, CancellationToken cancellationToken)
    {
        if (command.Actor.Role is not (ActorRole.Buyer or ActorRole.Staff))
            throw new ForbiddenException();

        var order = await orders.GetAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException("Order was not found.");

        if (command.Actor.Role == ActorRole.Buyer && order.CustomerId != command.Actor.CustomerId)
            throw new ForbiddenException();

        var now = time.GetUtcNow();
        order.Cancel(now);

        foreach (var line in order.Lines)
        {
            var stock = await stockItems.GetAsync(line.ProductId, cancellationToken)
                ?? throw new DomainRuleException($"No stock row for {line.ProductId}.");
            stock.Release(line.Quantity, order.Id, now);
        }

        await unitOfWork.CommitAsync(cancellationToken);
    }
}
```

`PaymentFailed` releases stock the same way. `Ship` does not release; a fuller `Stock` would `CommitSale` (decrement `OnHand` and `Reserved`). Add that method when you ship, and call it from `ShipOrderHandler` in the same commit as `order.Ship`. Do not decrement `OnHand` at reserve time or available inventory double-counts the penalty.

### Expected failure versus invariant failure

| Situation | Type | HTTP | Retry |
|-----------|------|------|-------|
| Quantity is zero, or cancel on a shipped order | `DomainRuleException` | 400 | No, the body or the state is wrong |
| Order id is unknown | `NotFoundException` | 404 | No |
| Caller is not allowed | `ForbiddenException` | 403 | No |
| Idempotency key race, or concurrency token | `ConflictException` / `DbUpdateConcurrencyException` | 409 | Yes, with the same key for place; for cancel, reload |
| Catalog port timed out | exception from the port | 503 or 500 | Yes |

A `Result<T>` type is the other shape: the method returns `Result.Fail("Not enough stock")` and the compiler forces the handler to branch. Use it when failure is a normal outcome you branch on in-process (a batch that continues after one line fails). Use exceptions when failure aborts the use case, which is this sample. Do not mix them in one handler (`Result` for stock, exceptions for cancel) or every caller guesses.

If you adopt `Result`, keep it in the application and domain projects as your own small type. A 2,000-line result library to avoid `throw` does not change the transaction boundary.

```C#
public readonly struct Result<T>
{
    private Result(T? value, string? error, bool ok)
    {
        Value = value;
        Error = error;
        IsOk = ok;
    }

    public T? Value { get; }
    public string? Error { get; }
    public bool IsOk { get; }

    public static Result<T> Ok(T value) => new(value, null, true);
    public static Result<T> Fail(string error) => new(default, error, false);
}
```

`Reserve` returning `Result` instead of throwing is reasonable for "not enough stock", because the caller might want to place a backorder. In this sample, not enough stock aborts the place, so throw. The unique-index race is the case that is awkward with exceptions because of the dirty `DbContext`, and it is still an exception thrown by infrastructure. Catching it at the handler boundary is the application policy.

### A pipeline without a mediator

Cross-cutting work (log the command name, time the handler, open a transaction) can be a decorator. You do not need `IMediator.Send`.

```C#
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

public sealed class LoggingHandler<TCommand, TResult>(
    ICommandHandler<TCommand, TResult> inner,
    ILogger<LoggingHandler<TCommand, TResult>> logger) : ICommandHandler<TCommand, TResult>
{
    public async Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken)
    {
        logger.LogInformation("Handling {Command}", typeof(TCommand).Name);
        return await inner.HandleAsync(command, cancellationToken);
    }
}
```

Register the inner handler and decorate it the way [DotnetPattern.md](DotnetPattern.md#decorator-around-an-existing-registration) shows, or skip the interface and log inside the one handler you actually have. Add the pipeline when a second concern appears on every use case. A pipeline of ten behaviors for three handlers is a framework you now have to debug at 2 a.m.

Do not inject `IMediator` into `Order` "so the domain can emit events the application handles before commit". Those handlers run before you know the commit succeeded.

### Registration

Handlers are scoped, same lifetime as `DbContext`. `TimeProvider.System` is a singleton.

```C#
public static class OrderingApplication
{
    public static IServiceCollection AddOrderingApplication(this IServiceCollection services)
    {
        services.AddScoped<PlaceOrderHandler>();
        services.AddScoped<CancelOrderHandler>();
        services.AddScoped<MarkOrderPaidHandler>();
        services.AddScoped<MarkOrderPaymentFailedHandler>();
        services.TryAddSingleton(TimeProvider.System);
        return services;
    }
}
```

`TryAddSingleton(TimeProvider.System)` lets a test host replace the clock with a `FakeTimeProvider` before calling `AddOrderingApplication`. See [TimeProvider](DotnetPattern.md#timeprovider).

`AddOrderingInfrastructure` (next section) registers the `DbContext` and the adapters. [Modular monolith](#modular-monolith) folds those two extension methods into one `AddOrdering` plus `MapOrdering` on the module, because the host should not list every handler of every module. The registrations are the same. Only the place you call them changes.

## Infrastructure and EF Core

EF maps the aggregate. It does not define it. Configuration, migrations, the outbox table, and vendor translators live here.

### One DbContext per bounded context

```C#
public sealed class OrderingDbContext(DbContextOptions<OrderingDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Stock> Stock => Set<Stock>();
    public DbSet<OutboxMessage> Outbox => Set<OutboxMessage>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("ordering");
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(OrderingDbContext).Assembly);
    }
}
```

`HasDefaultSchema("ordering")` keeps billing's tables out of this model. A second `DbContext` in another project owns schema `billing`. They do not share a migration history table. Point each context at its own history table if they share a database:

```C#
options.UseNpgsql(connectionString, npgsql =>
    npgsql.MigrationsHistoryTable("__ef_migrations", "ordering"));
```

`AddDbContext` is scoped. Do not register it singleton. A background worker creates a scope per batch. See [scoped work from a singleton](DotnetPattern.md#scoped-work-from-a-singleton-iservicescopefactory).

```C#
public static class OrderingInfrastructure
{
    public static IServiceCollection AddOrderingInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("Ordering")
            ?? throw new InvalidOperationException("Connection string 'Ordering' is missing.");

        services.AddDbContext<OrderingDbContext>((sp, options) =>
        {
            options.UseNpgsql(connectionString, npgsql =>
                npgsql.MigrationsHistoryTable("__ef_migrations", "ordering"));
            options.AddInterceptors(sp.GetRequiredService<OutboxInterceptor>());
        });

        services.AddScoped<OutboxInterceptor>();
        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IStockRepository, EfStockRepository>();
        services.AddScoped<IPriceList, EfPriceList>();
        services.AddScoped<IUnitOfWork, EfUnitOfWork>();
        services.AddScoped<IIdempotencyStore, EfIdempotencyStore>();
        services.AddScoped<IOrderQueries, EfOrderQueries>();
        return services;
    }
}
```

Register `OutboxInterceptor` as scoped and resolve it from the `AddDbContext` lambda. A singleton interceptor must not capture a scoped `DbContext`. This one does not capture one; it receives the context in `SavingChangesAsync`. Scoped is still the safer lifetime.

### Mapping

```C#
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => new OrderId(value));
        builder.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, value => new CustomerId(value));
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(32);
        builder.Property(o => o.IdempotencyKey)
            .HasMaxLength(80)
            .HasColumnName("idempotency_key");
        builder.HasIndex(o => o.IdempotencyKey)
            .IsUnique()
            .HasFilter("idempotency_key IS NOT NULL");

        builder.Ignore(o => o.DomainEvents);
        builder.Ignore(o => o.Total);
        builder.Ignore(o => o.Currency);

        builder.ComplexProperty(o => o.ShipTo, address =>
        {
            address.Property(a => a.Line1).HasMaxLength(200).HasColumnName("ship_line1");
            address.Property(a => a.City).HasMaxLength(100).HasColumnName("ship_city");
            address.Property(a => a.PostalCode).HasMaxLength(20).HasColumnName("ship_postal_code");
            address.Property(a => a.Country).HasMaxLength(2).HasColumnName("ship_country");
        });

        builder.OwnsMany(o => o.Lines, line =>
        {
            line.ToTable("order_lines");
            line.WithOwner().HasForeignKey("OrderId");
            line.HasKey("OrderId", nameof(OrderLine.ProductId));
            line.Property(l => l.ProductId)
                .HasConversion(id => id.Value, value => new ProductId(value));
            line.Property(l => l.ProductName).HasMaxLength(200);
            line.ComplexProperty(l => l.UnitPrice, money =>
            {
                money.Property(m => m.Amount).HasPrecision(18, 2).HasColumnName("unit_price_amount");
                money.Property(m => m.Currency).HasMaxLength(3).HasColumnName("unit_price_currency");
            });
        });

        builder.Navigation(o => o.Lines)
            .UsePropertyAccessMode(PropertyAccessMode.Field);

        builder.Property<uint>("xmin")
            .HasColumnType("xid")
            .ValueGeneratedOnAddOrUpdate()
            .IsConcurrencyToken();
    }
}
```

`Ignore(DomainEvents)` keeps the in-memory list out of the table. `Ignore(Total)` and `Ignore(Currency)` keep calculated values out of the table. If you store `total` as a column for reporting, it is a denormalized cache you must update in `AddLine` and `ChangeQuantity`, and a query should still not trust it over the lines without a test. The sample calculates it.

`HasFilter` is raw SQL. It must name the **column**, not the C# property. Without `HasColumnName("idempotency_key")`, EF creates `"IdempotencyKey"` and PostgreSQL rejects the filter. The filter allows many orders with a null key (staff tools that are not retried) while real checkouts stay unique. PostgreSQL unique indexes treat nulls as distinct unless you use `NULLS NOT DISTINCT` (PostgreSQL 15+).

`ComplexProperty` for `Address` stores columns on `orders`. `Address` is a `readonly record struct`, and `OwnsOne` only accepts a class. `OwnsMany` for lines stores `order_lines` with a composite key `(OrderId, ProductId)`, which matches "one product per order". `ComplexProperty` on the line stores `Money` the same way (EF Core 8). Neither value object gets its own id.

SQL Server has no `xmin`. Use a rowversion instead, and do not add both:

```C#
builder.Property<byte[]>("Version").IsRowVersion();
```

### Value converters

A `ValueConverter<OrderId, Guid>` registered once beats a copied lambda on every property. EF Core also lets you set a convention for the struct:

```C#
public sealed class OrderIdConverter : ValueConverter<OrderId, Guid>
{
    public OrderIdConverter() : base(id => id.Value, value => new OrderId(value)) { }
}
```

Apply it in `ConfigureConventions` if many entities use `OrderId`. A converter tells EF how to store the value. It does not run `Order.Place`. Do not put validation in the converter that throws on data already in the database: a migration you cannot read takes the site down. Validate in the domain constructor on the way in. On the way out, trust the column and the check constraint.

`HasConversion<string>()` on the enum stores `Placed`, not `0`. A reorder of the enum will not silently rewrite history. The cost is a wider column and an index on a string. That is the right trade for a status you will read in SQL during an incident.

### Backing fields and owned lines

`Lines` is `IReadOnlyList<OrderLine>` with no setter. EF cannot assign the property. `UsePropertyAccessMode(PropertyAccessMode.Field)` tells it to fill `_lines`. The field name `_lines` matches EF's convention for a property `Lines`. If you name the field `items`, configure it:

```C#
builder.Metadata.FindNavigation(nameof(Order.Lines))!
    .SetField("items");
```

`Include(o => o.Lines)` on a repository `Get` is explicit. Owned collections are not always loaded the way people remember from one EF version to the next. Include them in the aggregate repository so `Cancel` sees the lines it must release. A query DTO should project in SQL and not rely on this include.

Lazy-loading proxies are how a handler touches `order.Customer.Address` and runs SQL from a loop. Leave them off. If a use case needs another aggregate, the handler calls that aggregate's repository.

### Concurrency token

The `xmin` property is a shadow property. It is not on `Order`, so the domain stays free of persistence tokens. Npgsql updates it on write. The second `SaveChanges` sees a different `xmin` and throws `DbUpdateConcurrencyException`.

Map it at the edge:

```C#
catch (DbUpdateConcurrencyException)
{
    throw new ConflictException("The order was updated by someone else. Reload and retry.");
}
```

Catch this in `EfUnitOfWork.CommitAsync` so handlers stay free of EF types. Retry policy, if you add one, belongs in the handler for `Reserve` (reload stock, call `Reserve` again, commit). A blind retry of the same `DbContext` after a concurrency exception does not reload. You must `GetAsync` again (new context or `Entry.Reload`). For a web request, returning 409 and letting the client retry is the smaller design.

`Stock` needs the same token. Two checkouts reading `Available == 1` will both pass `Reserve` in memory. The token makes the second commit fail. Without the token, `Reserved` loses an increment.

```C#
public sealed class StockConfiguration : IEntityTypeConfiguration<Stock>
{
    public void Configure(EntityTypeBuilder<Stock> builder)
    {
        builder.ToTable("stock");
        builder.HasKey(s => s.ProductId);
        builder.Property(s => s.ProductId)
            .HasConversion(id => id.Value, value => new ProductId(value));
        builder.Ignore(s => s.Available);
        builder.Ignore(s => s.DomainEvents);
        builder.Property<uint>("xmin")
            .HasColumnType("xid")
            .ValueGeneratedOnAddOrUpdate()
            .IsConcurrencyToken();
    }
}
```

### Global query filters

A tenant filter and a soft-delete filter are infrastructure. They are also easy to get wrong. The `Order` in this guide has no `TenantId` until you add one; the filter below is the shape after that property exists.

```C#
builder.HasQueryFilter(o => o.TenantId == _tenantId);
```

`_tenantId` must come from a scoped tenant accessor evaluated per query, not from a value copied once when the model was built. EF Core supports this when the filter closes over a property on the `DbContext`:

```C#
public Guid CurrentTenantId => _tenant.TenantId;

// in Configure:
builder.HasQueryFilter(o => o.TenantId == CurrentTenantId);
```

`IgnoreQueryFilters()` on a repository method is a loaded gun. An admin report that calls it without a new tenant predicate will return every tenant's orders. Prefer a separate `OrderingAdminDbContext` with no filter, registered only in the admin host, over a boolean `includeAllTenants` on the application repository.

Soft delete is a filter of `DeletedAt == null`, which hides those rows from `GetAsync`. The handler then throws `NotFoundException` and the unique index on `IdempotencyKey` still sees the row. The client retries the key, the pre-check finds nothing, the insert fails the unique index. If you soft-delete orders, the unique index has to match the filter (`WHERE deleted_at IS NULL`) or you clear the key on delete. The aggregate method is `Cancel`, not `Delete`. Prefer a terminal status over a hidden row for orders you must still show on a receipt.

### Repository

```C#
public sealed class EfOrderRepository(OrderingDbContext db) : IOrderRepository
{
    public Task AddAsync(Order order, CancellationToken cancellationToken)
    {
        db.Orders.Add(order);
        return Task.CompletedTask;
    }

    public Task<Order?> GetAsync(OrderId id, CancellationToken cancellationToken) =>
        db.Orders
            .Include(o => o.Lines)
            .SingleOrDefaultAsync(o => o.Id == id, cancellationToken);
}

public sealed class EfIdempotencyStore(OrderingDbContext db) : IIdempotencyStore
{
    public async Task<OrderId?> FindAsync(string key, CancellationToken cancellationToken)
    {
        var id = await db.Orders.AsNoTracking()
            .Where(o => o.IdempotencyKey == key)
            .Select(o => (Guid?)o.Id.Value)
            .SingleOrDefaultAsync(cancellationToken);

        return id is null ? null : new OrderId(id.Value);
    }
}

public sealed class EfUnitOfWork(OrderingDbContext db) : IUnitOfWork
{
    public async Task CommitAsync(CancellationToken cancellationToken)
    {
        try
        {
            await db.SaveChangesAsync(cancellationToken);
        }
        catch (DbUpdateConcurrencyException)
        {
            throw new ConflictException("The row was updated by someone else. Reload and retry.");
        }
        catch (DbUpdateException ex) when (PostgresUniqueViolation.Is(ex))
        {
            throw new UniqueConstraintViolationException();
        }
    }
}
```

`AddAsync` does not call `SaveChanges`. `GetAsync` tracks. `FindAsync` for idempotency uses `AsNoTracking` so a lookup is not a second tracked `Order` fighting the one you are inserting.

`PostgresUniqueViolation.Is` reads the inner `PostgresException.SqlState == "23505"`. Keep that type in infrastructure. `UniqueConstraintViolationException` is the application type.

`IPriceList` reading a catalog table:

```C#
public sealed class EfPriceList(OrderingDbContext db) : IPriceList
{
    public async Task<ProductOffer?> FindAsync(ProductId productId, CancellationToken cancellationToken)
    {
        var row = await db.Set<CatalogPriceRow>().AsNoTracking()
            .SingleOrDefaultAsync(p => p.ProductId == productId.Value, cancellationToken);

        if (row is null)
            return null;

        return new ProductOffer(productId, row.Name, new Money(row.Amount, row.Currency));
    }
}
```

`CatalogPriceRow` is a persistence type in infrastructure, mapped to a table or view the catalog context owns and this context may only read. It is not a domain entity and it has no `Reserve` method. Do not add `DbSet<Product>` to `OrderingDbContext` and navigate from `OrderLine` to it. The moment you do, ordering migrations start altering the catalog table.

If the price list is HTTP, `HttpPriceList` implements the same port, translates the vendor JSON in this class, and times out with a `CancellationToken`. The handler does not change. See [HttpClientGuidance.md](HttpClientGuidance.md) for the client lifetime. Do not new up `HttpClient` inside the adapter.

### Do not leak IQueryable

```C#
// ❌ BAD — the application project now writes EF queries and references the provider
public interface IOrderRepository
{
    IQueryable<Order> Query();
}
```

Callers compose `Include`, `Where`, and provider-specific functions in the handler. The port cannot be faked without an in-memory query provider that does not match PostgreSQL. Transaction rules move out of the aggregate and into whatever query the latest caller wrote.

Commands go through `GetAsync` / `AddAsync`. Reads go through a separate query interface that returns DTOs already projected. Both implementations use EF. Only the implementations are allowed to.

Specifications that are `Expression<Func<Order, bool>>` are `IQueryable` with a costume. A specification that is `bool IsSatisfiedBy(Order order)` is an in-memory predicate. It is fine on an aggregate you already loaded, and it is usually a method you could have put on `Order`. Prefer the method until you have two predicates you actually compose.

## The API is an adapter

The endpoint translates HTTP into a command and a result into HTTP. It does not set `order.Status`. It does not catch domain exceptions one route at a time.

### Contracts

```C#
public sealed record PlaceOrderRequest(
    Guid CustomerId,
    AddressRequest ShipTo,
    IReadOnlyList<LineRequest> Lines);

public sealed record AddressRequest(string Line1, string City, string PostalCode, string Country);
public sealed record LineRequest(Guid ProductId, int Quantity);
public sealed record PlaceOrderResponse(Guid OrderId);

public static class OrderEndpoints
{
    public static void Map(WebApplication app)
    {
        app.MapPost("/orders", async (
            PlaceOrderRequest request,
            HttpContext http,
            PlaceOrderHandler handler,
            CancellationToken cancellationToken) =>
        {
            var key = http.Request.Headers["Idempotency-Key"].ToString();
            if (string.IsNullOrWhiteSpace(key))
                return Results.Problem(title: "Missing idempotency key", statusCode: StatusCodes.Status400BadRequest);

            var actor = http.ToActor();
            var command = new PlaceOrder(
                new CustomerId(request.CustomerId),
                new Address(request.ShipTo.Line1, request.ShipTo.City, request.ShipTo.PostalCode, request.ShipTo.Country),
                request.Lines.Select(l => new PlaceOrderLine(new ProductId(l.ProductId), l.Quantity)).ToArray(),
                key,
                actor);

            var id = await handler.HandleAsync(command, cancellationToken);
            return Results.Created($"/orders/{id.Value}", new PlaceOrderResponse(id.Value));
        });

        app.MapPost("/orders/{id}/cancel", async (
            OrderId id,
            HttpContext http,
            CancelOrderHandler handler,
            CancellationToken cancellationToken) =>
        {
            await handler.HandleAsync(new CancelOrder(id, http.ToActor()), cancellationToken);
            return Results.NoContent();
        });
    }
}
```

`ToActor()` reads claims and returns `Actor`. It lives in the API project. A missing claim is 401 from the auth middleware, not a domain exception.

Do not serialize `Order` as the response. The JSON shape becomes a public contract: private events, `ShipTo`, internal status names, whatever you add next month. `PlaceOrderResponse` is the contract. You can add a field to `Order` without breaking clients.

Do not accept `unitPrice` on `LineRequest`. The handler prices from `IPriceList`. If a client sends a price anyway, the extra JSON property is ignored. Ignoring is better than honoring it.

### Problem details

```C#
public sealed class DomainExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (status, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Not found"),
            ForbiddenException => (StatusCodes.Status403Forbidden, "Forbidden"),
            DomainRuleException => (StatusCodes.Status400BadRequest, "Rule violated"),
            ConflictException => (StatusCodes.Status409Conflict, "Conflict"),
            _ => (0, "")
        };

        if (status == 0)
            return false;

        await Results.Problem(
                title: title,
                detail: exception.Message,
                statusCode: status)
            .ExecuteAsync(httpContext);

        return true;
    }
}
```

`detail: exception.Message` is acceptable for `DomainRuleException` because you wrote those messages for a caller. Do not do it for unexpected exceptions. Returning `false` lets the generic handler produce a 500 without a stack trace in the body. Register `AddProblemDetails()` and `UseExceptionHandler()`.

`ForbiddenException` in the sample has no message. `detail` will be empty. That is fine. Do not put "you are not the buyer of order X" in a message a different user can read if you chose 404-style hiding. This sample uses 403 and a generic title.

### Strongly typed ids on the route

`MapPost("/orders/{id}/cancel", (OrderId id, ...) => ...)` works when `OrderId` has `TryParse(string?, IFormatProvider?, out OrderId)`. A failed parse is a 400 from the framework before your handler runs.

JSON bodies do not use `TryParse`. A request DTO in this sample uses `Guid` and the endpoint wraps `new CustomerId(...)`. If you want `OrderId` inside the JSON body, add a converter and register it on the serializer:

```C#
public sealed class OrderIdJsonConverter : JsonConverter<OrderId>
{
    public override OrderId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) =>
        new(reader.GetGuid());

    public override void Write(Utf8JsonWriter writer, OrderId value, JsonSerializerOptions options) =>
        writer.WriteStringValue(value.Value);
}
```

One converter per id type, or a factory converter. Do not `ToString()` an `OrderId` into a response and parse it back with `Guid.Parse` in the client of the same solution if you can pass the struct. At the HTTP boundary, strings and numbers are the contract. That is the adapter's job.

## Reads do not go through the aggregate

`GetAsync` loads an `Order` so a command can change it. A list screen does not need `Cancel`, change tracking, or the event list. Tracking every row in a page of 50 turns a read into a memory tax and invites a bug where a serializer walks `Lines` and a later `SaveChanges` writes accidental mutations.

### Query objects

```C#
public sealed record OrderSummary(
    Guid Id,
    string Status,
    string Currency,
    decimal Total,
    int LineCount,
    DateTimeOffset PlacedAt);

public interface IOrderQueries
{
    Task<OrderSummary?> GetSummaryAsync(OrderId id, CancellationToken cancellationToken);
    Task<IReadOnlyList<OrderSummary>> ListByCustomerAsync(
        CustomerId customerId,
        DateTimeOffset? placedBefore,
        Guid? idBefore,
        int limit,
        CancellationToken cancellationToken);
}

public sealed class EfOrderQueries(OrderingDbContext db) : IOrderQueries
{
    public Task<OrderSummary?> GetSummaryAsync(OrderId id, CancellationToken cancellationToken) =>
        db.Orders.AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderSummary(
                o.Id.Value,
                o.Status.ToString(),
                o.Lines.Select(l => l.UnitPrice.Currency).FirstOrDefault() ?? "",
                o.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity),
                o.Lines.Count,
                o.PlacedAt))
            .SingleOrDefaultAsync(cancellationToken);

    public async Task<IReadOnlyList<OrderSummary>> ListByCustomerAsync(
        CustomerId customerId,
        DateTimeOffset? placedBefore,
        Guid? idBefore,
        int limit,
        CancellationToken cancellationToken)
    {
        var query = db.Orders.AsNoTracking().Where(o => o.CustomerId == customerId);

        if (placedBefore is not null && idBefore is not null)
        {
            query = query.Where(o =>
                o.PlacedAt < placedBefore
                || (o.PlacedAt == placedBefore && o.Id.Value.CompareTo(idBefore.Value) < 0));
        }

        return await query
            .OrderByDescending(o => o.PlacedAt)
            .ThenByDescending(o => o.Id)
            .Take(limit)
            .Select(o => new OrderSummary(
                o.Id.Value,
                o.Status.ToString(),
                o.Lines.Select(l => l.UnitPrice.Currency).FirstOrDefault() ?? "",
                o.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity),
                o.Lines.Count,
                o.PlacedAt))
            .ToListAsync(cancellationToken);
    }
}
```

The projection runs in SQL. `AsNoTracking` keeps the identity map out of a read. The interface returns `OrderSummary`, not `Order`.

`Sum` in the projection duplicates `Order.Total`. That is the cost of not loading the aggregate. If the rounding rule changes, update `Money.Times` and this projection together, and pin both with a test. Storing `total_amount` on the order at write time removes the duplication and makes the list query an index-only-friendly column. Then the write model is responsible for keeping the column equal to the lines. Pick one. Do not calculate it in the UI from a payload that might be stale and also store a different total.

### Keyset pagination

`OFFSET 10000` walks 10000 rows it will throw away. A list sorted by `PlacedAt DESC, Id DESC` should page with the last row's sort key:

```http
GET /customers/{customerId}/orders?limit=20
GET /customers/{customerId}/orders?limit=20&placedBefore=2026-09-01T00:00:00Z&idBefore={lastId}
```

The `OR` predicate in the sample matches that order. Index `(customer_id, placed_at DESC, id DESC)`. The indexing guide covers why a leading range kills the rest of a composite index; here the customer id is equality and the timestamps are the range, which is the shape you want. See [Database_Indexing.md](Database_Indexing.md).

Cap `limit` (for example at 100) in the endpoint. A client that sends `limit=1000000` will get a capped query, not a bigger one.

### A separate read model

A summary you can project from `orders` in one query does not need a second database. Add a read model when the screen is a join across contexts you refuse to join at query time, or when the write shape (normalized lines) cannot serve the read shape (one row per order with a JSON blob the UI wants) without a heavy query.

Build it from integration events, not from "the API writes both tables and hopes":

```C#
public sealed class OrderSummaryProjection
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = "";
    public decimal Total { get; set; }
    public string Currency { get; set; } = "";
    public DateTimeOffset PlacedAt { get; set; }
}
```

The projector is a consumer of `ordering.order_placed.v1` and `ordering.order_cancelled.v1`. It upserts `order_summaries` in a schema the list endpoint reads. It is eventually consistent: the 201 response can return before the summary row exists. The order-details endpoint can still read the write model by id. The list can be a few hundred milliseconds behind. If the product cannot tolerate that, project in the same transaction as the outbox insert (same `DbContext`, same commit) and accept that the read table is coupled to the write transaction. That is still not "the controller updates both".

Do not point EF's `Order` aggregate at the summary table. Two models, two types. The summary type can be a public-setter DTO. It has no `Cancel` method. If someone calls `Cancel` on it, you put behavior in the wrong model.

## Events and the outbox

### Three event kinds

| Kind | Who raises it | Who sees it | Example |
|------|---------------|-------------|---------|
| Domain event | Aggregate method | In-process, after you copy it to the outbox. Not a public schema | `OrderPlaced` with `OrderId` struct |
| Integration event | Mapper in infrastructure, from a domain event | Other contexts and other services | `ordering.order_placed.v1` with `Guid` fields |
| Application notification | A handler, rarely | In-process UI cache bust, metrics | Prefer the integration event so a restart does not drop it |

Publish the integration event, not the domain type. The domain record will gain a field because a use case needed it, and every consumer will break. The domain record's JSON also needs converters for `OrderId`. Consumers should not need those converters.

```C#
public sealed record OrderPlacedV1(
    Guid EventId,
    Guid OrderId,
    Guid CustomerId,
    string Currency,
    decimal Total,
    DateTimeOffset OccurredAt);

public sealed record OrderCancelledV1(Guid EventId, Guid OrderId, DateTimeOffset OccurredAt);
public sealed record OrderPaidV1(Guid EventId, Guid OrderId, DateTimeOffset OccurredAt);
public sealed record OrderPaymentFailedV1(Guid EventId, Guid OrderId, DateTimeOffset OccurredAt);
public sealed record OrderShippedV1(Guid EventId, Guid OrderId, DateTimeOffset OccurredAt);
```

Map it in infrastructure when you write the outbox, or map it in the publisher. Mapping at write time freezes the contract into the row. Mapping at publish time lets you fix a mapper bug without a backfill, and it also lets you publish a different contract than you stored if you are not careful. This sample stores the **integration** payload, produced by a mapper next to the interceptor, so the row is what you will send.

The domain event still exists. The mapper reads it. `Order` does not reference `OrderPlacedV1`.

### Stable event names

`domainEvent.GetType().Name` is `OrderPlaced` today and `OrderSubmitted` the day someone renames the class. Old rows and new code disagree. Store a name you chose:

```C#
public readonly record struct IntegrationEvent(string Name, string Payload);

public static class IntegrationEvents
{
    public static IntegrationEvent? From(Order order, IDomainEvent domainEvent) => domainEvent switch
    {
        OrderPlaced placed => new IntegrationEvent("ordering.order_placed.v1", JsonSerializer.Serialize(new OrderPlacedV1(
            placed.EventId,
            placed.OrderId.Value,
            order.CustomerId.Value,
            order.Currency,
            order.Total.Amount,
            placed.OccurredAt))),
        OrderCancelled cancelled => new IntegrationEvent("ordering.order_cancelled.v1", JsonSerializer.Serialize(new OrderCancelledV1(
            cancelled.EventId,
            cancelled.OrderId.Value,
            cancelled.OccurredAt))),
        OrderPaid paid => new IntegrationEvent("ordering.order_paid.v1", JsonSerializer.Serialize(new OrderPaidV1(
            paid.EventId,
            paid.OrderId.Value,
            paid.OccurredAt))),
        OrderPaymentFailed failed => new IntegrationEvent("ordering.order_payment_failed.v1", JsonSerializer.Serialize(new OrderPaymentFailedV1(
            failed.EventId,
            failed.OrderId.Value,
            failed.OccurredAt))),
        OrderShipped shipped => new IntegrationEvent("ordering.order_shipped.v1", JsonSerializer.Serialize(new OrderShippedV1(
            shipped.EventId,
            shipped.OrderId.Value,
            shipped.OccurredAt))),
        OrderQuantityChanged => null,
        _ => throw new InvalidOperationException($"No integration mapping for {domainEvent.GetType().Name}.")
    };
}
```

The default arm throws. `MarkPaid`, `Ship`, and `MarkPaymentFailed` each raise an event, so those arms have to exist or the commit that follows the handler throws. `OrderQuantityChanged` returns null: it stays inside the module. A new domain event fails the commit until you either map it or add an explicit skip. Do not skip by accident of `GetType().Name`.

`v1` stays forever. A breaking change is `ordering.order_placed.v2`. Publish both during a migration window, or publish v2 and let consumers you still own move first. Do not edit the fields of v1 in place.

### Write the outbox in the same transaction

```C#
public sealed class OutboxMessage
{
    public Guid Id { get; private set; }
    public string Type { get; private set; } = "";
    public string Payload { get; private set; } = "";
    public DateTimeOffset OccurredAt { get; private set; }
    public DateTimeOffset? ProcessedAt { get; private set; }
    public DateTimeOffset? LockedUntil { get; private set; }
    public int Attempts { get; private set; }
    public string? LastError { get; private set; }

    public static OutboxMessage Create(Guid id, string type, string payload, DateTimeOffset occurredAt) => new()
    {
        Id = id,
        Type = type,
        Payload = payload,
        OccurredAt = occurredAt
    };

    public void MarkProcessed(DateTimeOffset now)
    {
        ProcessedAt = now;
        LockedUntil = null;
    }

    public void MarkFailed(string error, DateTimeOffset now)
    {
        LastError = error.Length > 500 ? error[..500] : error;
        LockedUntil = null;
        _ = now;
    }

    public void Claim(DateTimeOffset now, TimeSpan lease)
    {
        Attempts += 1;
        LockedUntil = now.Add(lease);
    }
}

public sealed class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> builder)
    {
        builder.ToTable("outbox_messages");
        builder.HasKey(m => m.Id);
        builder.Property(m => m.Id).HasColumnName("id");
        builder.Property(m => m.Type).HasMaxLength(80).HasColumnName("type");
        builder.Property(m => m.Payload).HasColumnType("jsonb").HasColumnName("payload");
        builder.Property(m => m.OccurredAt).HasColumnName("occurred_at");
        builder.Property(m => m.ProcessedAt).HasColumnName("processed_at");
        builder.Property(m => m.LockedUntil).HasColumnName("locked_until");
        builder.Property(m => m.Attempts).HasColumnName("attempts");
        builder.Property(m => m.LastError).HasMaxLength(500).HasColumnName("last_error");
    }
}
```

`Id` is the domain event's `EventId`, not a new guid. Republish is the same id. `MarkFailed` keeps `ProcessedAt` null so the worker retries. The `FromSql` later in this section selects `id`, `locked_until`, and the other snake_case names. Those strings match `HasColumnName`. `jsonb` is PostgreSQL; on SQL Server store `Payload` as `nvarchar(max)`.

The original guide copied events inside `EfUnitOfWork` before `SaveChanges`. That works until a caller calls `db.SaveChangesAsync` directly and the outbox stays empty. The interceptor runs for every save on this context.

### An interceptor so a stray SaveChanges still records events

```C#
public sealed class OutboxInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var db = (OrderingDbContext)eventData.Context!;
        Stage(db);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private static void Stage(OrderingDbContext db)
    {
        var orders = db.ChangeTracker.Entries<Order>()
            .Select(entry => entry.Entity)
            .Where(order => order.DomainEvents.Count > 0)
            .ToArray();

        foreach (var order in orders)
        {
            foreach (var domainEvent in order.DomainEvents)
            {
                if (db.Outbox.Local.Any(message => message.Id == domainEvent.EventId))
                    continue;

                var mapped = IntegrationEvents.From(order, domainEvent);
                if (mapped is null)
                    continue;

                db.Outbox.Add(OutboxMessage.Create(
                    domainEvent.EventId, mapped.Value.Name, mapped.Value.Payload, domainEvent.OccurredAt));
            }
        }
    }
}
```

Adding `Outbox` rows inside `SavingChanges` includes them in the same `SaveChanges` and the same database transaction. If the commit fails, the outbox rows roll back with the order.

`Outbox.Local` is the set already tracked. A retry of `SaveChanges` on the same context must not insert a second copy of the same `EventId`. The `Id` primary key would also stop the duplicate at the database, but the exception would fail a commit that should have succeeded.

Clear domain events only after a successful save. If you clear them in `SavingChanges` and the save throws, a retry has nothing to stage (the first staging already added outbox entities, so the `Local` check covers that path). If you clear them and also detach the outbox entities on failure, the events are gone. `SavedChangesAsync` is the place to clear:

```C#
public override ValueTask<int> SavedChangesAsync(
    SaveChangesCompletedEventData eventData,
    int result,
    CancellationToken cancellationToken = default)
{
    var db = (OrderingDbContext)eventData.Context!;
    foreach (var aggregate in db.ChangeTracker.Entries<IHasDomainEvents>().Select(e => e.Entity))
        aggregate.ClearEvents();

    return base.SavedChangesAsync(eventData, result, cancellationToken);
}
```

`Stock` raises `StockReserved`, `StockReleased`, and `StockCommitted` in the same database as the order. This sample does not publish them: a warehouse service would need its own mapper, and that mapper must not query `Order` during `SavingChanges`. Clear them with the order events so the next save in the same context does not see a stale list. If the mapper's default arm throws, a stock event with no arm fails the commit. Give stock its own switch with an explicit `null` for events that stay local.

Do not publish to a broker inside the interceptor. The broker call is slower than the transaction, and a failed HTTP call after a successful local insert is a different problem (the outbox worker's problem). The interceptor only inserts rows.

`EfUnitOfWork` stays a `SaveChangesAsync` wrapper that translates concurrency and unique violations. It does not also copy events. One mechanism, or you will double-publish.

### Publish without holding the row lock

A worker reads unpublished rows and sends them. Two workers must not send the same row at the same time and then both mark it processed. Holding `FOR UPDATE` while you call the broker keeps a row lock for the length of the HTTP call. Do not do that.

Claim, commit, publish, mark:

```C#
public sealed class OutboxProcessor(
    IServiceScopeFactory scopes,
    IIntegrationPublisher publisher,
    TimeProvider time,
    ILogger<OutboxProcessor> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var publishedAny = await PublishBatchAsync(stoppingToken);
            if (!publishedAny)
                await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken);
        }
    }

    private async Task<bool> PublishBatchAsync(CancellationToken cancellationToken)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderingDbContext>();
        var now = time.GetUtcNow();

        await using var tx = await db.Database.BeginTransactionAsync(cancellationToken);

        var batch = await db.Outbox
            .FromSqlInterpolated($"""
                SELECT id, type, payload, occurred_at, processed_at, locked_until, attempts, last_error
                FROM ordering.outbox_messages
                WHERE processed_at IS NULL
                  AND attempts < 10
                  AND (locked_until IS NULL OR locked_until < now())
                ORDER BY occurred_at
                LIMIT 20
                FOR UPDATE SKIP LOCKED
                """)
            .ToListAsync(cancellationToken);

        if (batch.Count == 0)
        {
            await tx.RollbackAsync(cancellationToken);
            return false;
        }

        foreach (var message in batch)
            message.Claim(now, TimeSpan.FromMinutes(2));

        await db.SaveChangesAsync(cancellationToken);
        await tx.CommitAsync(cancellationToken);

        foreach (var message in batch)
        {
            try
            {
                await publisher.PublishAsync(message.Type, message.Payload, message.Id, cancellationToken);
                message.MarkProcessed(time.GetUtcNow());
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogWarning(ex, "Outbox publish failed for {EventId}", message.Id);
                message.MarkFailed(ex.Message, time.GetUtcNow());
            }
        }

        await db.SaveChangesAsync(cancellationToken);
        return true;
    }
}
```

Read this carefully before copying it.

`FOR UPDATE SKIP LOCKED` is PostgreSQL. SQL Server uses `READPAST` with different syntax. The `FromSqlInterpolated` query must be the whole statement EF executes. Do not append `.Where` or `.OrderBy` after it. EF wraps composed SQL in a subquery, and PostgreSQL rejects `FOR UPDATE` in a subquery.

The column list must match the table mapping, including column names EF expects. If the table is `outbox_messages` and the properties are `ProcessedAt`, configure column names or the reader will not bind. This is the fragile part of `FromSql`. A dedicated `ClaimBatch` SQL function in the database is often clearer than a mapped entity query. The behavior you want is: lock 20 unpublished rows, skip rows another worker has locked, do not block.

`Claim` increments `Attempts` and sets `LockedUntil` two minutes out, inside the short transaction. A crash after that commit and before publish still counts as an attempt. `processed_at` stays null. The next claim does not see the row until the lease expires, so a second worker does not publish the same message while the first is still in `PublishAsync`. If the first worker dies, the lease expires and the message is retried. `MarkFailed` clears `LockedUntil` so a known failure can be retried on the next poll instead of waiting out the lease. Consumers must still be idempotent: a slow publish can finish after the lease expired and a second worker may already have sent the same event.

`IServiceScopeFactory` is required. `OutboxProcessor` is a singleton hosted service. `OrderingDbContext` is scoped. Do not inject `OrderingDbContext` into the hosted service constructor. See [AsyncGuidance.md](AsyncGuidance.md) for `BackgroundService` and [DotnetPattern.md](DotnetPattern.md#ihostedservice--backgroundservice).

`IIntegrationPublisher` is a port. The RabbitMQ or Service Bus adapter lives in infrastructure. Publishing is at-least-once. The inbox makes it effectively once for the consumer's side effects.

### Inbox

A consumer that sends email on `ordering.order_cancelled.v1` will send two emails if the publisher retries. Store the event id you already handled, in the same transaction as the side effect.

```C#
public sealed class InboxMessage
{
    public Guid EventId { get; private set; }
    public DateTimeOffset ProcessedAt { get; private set; }

    public static InboxMessage Create(Guid eventId, DateTimeOffset now) => new()
    {
        EventId = eventId,
        ProcessedAt = now
    };
}

public sealed class OrderCancelledConsumer(BillingDbContext db, TimeProvider time)
{
    public async Task HandleAsync(OrderCancelledV1 message, CancellationToken cancellationToken)
    {
        if (await db.Inbox.AnyAsync(i => i.EventId == message.EventId, cancellationToken))
            return;

        var invoice = await db.Invoices.SingleOrDefaultAsync(i => i.OrderId == message.OrderId, cancellationToken);
        invoice?.VoidUnpaid();

        db.Inbox.Add(InboxMessage.Create(message.EventId, time.GetUtcNow()));
        await db.SaveChangesAsync(cancellationToken);
    }
}
```

Unique primary key on `inbox.event_id`. The `AnyAsync` check plus the insert still races. The unique key is the authority. Catch `23505` and treat it as "already processed".

The consumer maps `OrderCancelledV1` into `Invoice.VoidUnpaid`. It does not new up `Ordering.Domain.Order`. Billing does not reference the ordering domain project. The message contract can live in a small `Ordering.Contracts` package that contains records and no behavior. Both sides may reference contracts. Neither side references the other's domain.

### Poison rows

`attempts < 10` stops a message that will never succeed (a payload you cannot deserialize) from spinning forever. Those rows need a person. Log them with `EventId` and `LastError`. Do not delete them on the tenth failure; you will want the payload when you fix the mapper.

A deserializer that throws is a failed attempt, not a process crash. Catch it inside the publish or consume loop. An uncaught exception in `BackgroundService.ExecuteAsync` stops the loop. The outbox then sits unpublished until the process restarts, and if you do not catch, it stops again. See the hosting notes in [AsyncGuidance.md](AsyncGuidance.md).

Order of events for one order is usually the order you inserted them (`occurred_at`, then `id`). Across orders, consumers must not assume global order. `order_paid` can be delivered to billing before billing has consumed `order_placed` if you use two queues. The consumer that cannot find the invoice yet should retry (not mark the inbox done). A retry with backoff beats a permanent failure for a message that arrived early.

## Anti-corruption layer

The payment vendor's API is a bounded context you do not control. Translate at the edge. `Order` understands `MarkPaid` and `MarkPaymentFailed`. It does not understand `"CAPTURED"`, `"REQUIRES_ACTION"`, or a vendor error struct.

```C#
public sealed record VendorPaymentNotification(string VendorPaymentId, string Status, string OrderReference);

public interface IPaymentNotifications
{
    Task HandleAsync(VendorPaymentNotification notification, CancellationToken cancellationToken);
}

public sealed class PaymentNotificationHandler(
    MarkOrderPaidHandler paid,
    MarkOrderPaymentFailedHandler failed,
    ILogger<PaymentNotificationHandler> logger) : IPaymentNotifications
{
    public async Task HandleAsync(VendorPaymentNotification notification, CancellationToken cancellationToken)
    {
        if (!Guid.TryParse(notification.OrderReference, out var orderGuid))
        {
            logger.LogWarning("Payment notification without an order id: {VendorPaymentId}", notification.VendorPaymentId);
            return;
        }

        var orderId = new OrderId(orderGuid);
        switch (PaymentTranslation.ToOutcome(notification.Status))
        {
            case PaymentOutcome.Paid:
                await paid.HandleAsync(new MarkOrderPaid(orderId), cancellationToken);
                return;
            case PaymentOutcome.Failed:
                await failed.HandleAsync(new MarkOrderPaymentFailed(orderId), cancellationToken);
                return;
            case PaymentOutcome.Unknown:
                logger.LogWarning(
                    "Unrecognized payment status {Status} for order {OrderId}",
                    notification.Status,
                    orderId.Value);
                return;
        }
    }
}
```

`Unknown` does not change the order. A webhook you do not understand should be visible in logs and safe to replay once you add a mapping. Treating unknown as failure will cancel paid orders the day the vendor adds a status.

The webhook endpoint in the API project authenticates the vendor (signature header), builds `VendorPaymentNotification`, and calls `IPaymentNotifications`. Signature checks are adapter code. They are not a method on `Order`.

Outbound calls are the same shape in reverse. `IPaymentGateway.Authorize(orderId, money)` is a port. `StripePaymentGateway` maps `Money` to the vendor's minor units (cents as `long`) and their currency string. Multiplying decimal dollars by 100 in the domain "because Stripe wants cents" puts a vendor rule in `Order`. Do it in the gateway adapter, in one place, with a test for `10.10m` → `1010`.

```C#
public static long ToMinorUnits(Money money) =>
    money.Currency switch
    {
        "USD" or "EUR" => (long)(money.Amount * 100m),
        "JPY" => (long)money.Amount,
        _ => throw new DomainRuleException($"No minor-unit scale for {money.Currency}.")
    };
```

Throw on a currency you have not classified. A default of "times 100" mis-charges yen by two digits.

## A payment process

Place-and-reserve commits locally. Payment is another context. The order stays `Placed` until a notification arrives. Stock stays reserved while the customer is on the card page. If you never hear back, a timeout worker releases the stock and marks payment failed.

```C#
public sealed record MarkOrderPaid(OrderId OrderId);
public sealed record MarkOrderPaymentFailed(OrderId OrderId);

public sealed class MarkOrderPaidHandler(
    IOrderRepository orders,
    IUnitOfWork unitOfWork,
    TimeProvider time)
{
    public async Task HandleAsync(MarkOrderPaid command, CancellationToken cancellationToken)
    {
        var order = await orders.GetAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException("Order was not found.");

        if (order.Status == OrderStatus.Paid)
            return;

        order.MarkPaid(time.GetUtcNow());
        await unitOfWork.CommitAsync(cancellationToken);
    }
}
```

The status check before `MarkPaid` makes the consumer idempotent at the domain level too: a second `CAPTURED` is a no-op, not a 400. `MarkPaid` itself still throws if the status is `Shipped` or `Cancelled`. A payment captured after cancel is an incident, not a silent success. The handler can branch:

```C#
if (order.Status == OrderStatus.Cancelled)
    throw new ConflictException("Payment captured for a cancelled order.");
```

Someone has to refund. Hiding it inside `MarkPaid` by ignoring the call loses money. `ConflictException` on the webhook returns 409 so the vendor retries, which is noisy, or you record a `PaymentCapturedAfterCancel` event and return 200 so the vendor stops, while an operator queue picks up the event. Returning 200 and recording the event is the better webhook contract. Pick it explicitly in the adapter, not by swallowing exceptions.

Timeout:

```C#
public sealed class ExpireUnpaidOrders(
    IServiceScopeFactory scopes,
    TimeProvider time) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ExpireBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task ExpireBatchAsync(CancellationToken cancellationToken)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderingDbContext>();
        var handler = scope.ServiceProvider.GetRequiredService<MarkOrderPaymentFailedHandler>();
        var cutoff = time.GetUtcNow().AddMinutes(-15);

        var ids = await db.Orders.AsNoTracking()
            .Where(o => o.Status == OrderStatus.Placed && o.PlacedAt < cutoff)
            .OrderBy(o => o.PlacedAt)
            .Take(50)
            .Select(o => o.Id)
            .ToListAsync(cancellationToken);

        foreach (var id in ids)
        {
            try
            {
                await handler.HandleAsync(new MarkOrderPaymentFailed(id), cancellationToken);
            }
            catch (DomainRuleException)
            {
                // Paid or cancelled between the query and the load.
            }
            catch (ConflictException)
            {
                // Concurrency token. The next pass will load a fresh row.
            }
        }
    }
}
```

`MarkOrderPaymentFailedHandler` calls `order.MarkPaymentFailed` and `stock.Release` for each line, then commits. The query uses `AsNoTracking` and only ids. Each id is its own transaction so one bad row does not roll back the batch. The domain exception means the row moved on; that is success for a sweeper.

Fifteen minutes is a product rule. It lives next to the sweeper, not inside `Order.Place`. The aggregate does not know the timeout. If you pass `payBy` into `Place` and store it, the aggregate can refuse `MarkPaid` after that instant. That is a stronger rule ("we do not take money after the reserve expired") and it belongs on `Order` if finance asked for it. The sweeper is what *causes* the failure in time. The method is what *allows* it.

Do not run the sweeper's query inside `Place`. Do not hold the stock row lock for fifteen minutes. Reserve, commit, release later.

## Testing

### Aggregate tests

Domain tests new up the aggregate. No EF, no web host, no mocks.

```C#
public sealed class OrderTests
{
    private static readonly DateTimeOffset Now = new(2026, 9, 27, 12, 0, 0, TimeSpan.Zero);
    private static readonly Address ShipTo = new("1 Main", "Hanoi", "10000", "VN");

    [Fact]
    public void Cancel_from_placed_sets_status_and_emits_one_event()
    {
        var order = PlacedOrder();

        order.Cancel(Now);

        Assert.Equal(OrderStatus.Cancelled, order.Status);
        var cancelled = Assert.Single(order.DomainEvents.OfType<OrderCancelled>());
        Assert.Equal(order.Id, cancelled.OrderId);
        Assert.Equal(Now, cancelled.OccurredAt);
    }

    [Fact]
    public void Cancel_from_shipped_throws()
    {
        var order = PlacedOrder();
        order.MarkPaid(Now);
        order.Ship(Now);
        order.ClearEvents();

        var ex = Assert.Throws<DomainRuleException>(() => order.Cancel(Now));
        Assert.Contains("placed", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    [Fact]
    public void Place_rejects_two_lines_for_the_same_product()
    {
        var line = new OrderLine(new ProductId(Guid.CreateVersion7()), "Tea", 1, new Money(10, "USD"));

        var ex = Assert.Throws<DomainRuleException>(() => Order.Place(
            OrderId.New(),
            new CustomerId(Guid.CreateVersion7()),
            ShipTo,
            [line, line],
            "key-1",
            Now));

        Assert.Contains("one line", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    [Fact]
    public void Total_rounds_each_line_before_adding()
    {
        var order = Order.Place(
            OrderId.New(),
            new CustomerId(Guid.CreateVersion7()),
            ShipTo,
            [
                new OrderLine(new ProductId(Guid.CreateVersion7()), "A", 2, new Money(0.10m, "USD")),
                new OrderLine(new ProductId(Guid.CreateVersion7()), "B", 1, new Money(0.20m, "USD"))
            ],
            "key-2",
            Now);

        Assert.Equal(new Money(0.40m, "USD"), order.Total);
    }

    private static Order PlacedOrder() => Order.Place(
        OrderId.New(),
        new CustomerId(Guid.CreateVersion7()),
        ShipTo,
        [new OrderLine(new ProductId(Guid.CreateVersion7()), "Tea", 2, new Money(10, "USD"))],
        "key",
        Now);
}
```

`ClearEvents` in the shipped-cancel test is fair: you are isolating the cancel assertion. Production code clears events in the interceptor after commit. A test that asserts `DomainEvents` after two transitions without clearing should expect both events. Do not assert event order unless a consumer depends on it.

`Stock` tests cover `Reserve` past `Available`, `Release` past `Reserved`, and a reserve of zero. Those three tests prevent the counter bugs. You do not need a test per property.

### Handler tests

Handler tests use fakes that implement the ports. They do not boot PostgreSQL. They answer "given these offers and this stock, does place reserve and commit once?".

```C#
public sealed class FakeOrders : IOrderRepository
{
    public List<Order> Added { get; } = [];
    public Dictionary<OrderId, Order> Store { get; } = [];

    public Task AddAsync(Order order, CancellationToken cancellationToken)
    {
        Added.Add(order);
        Store[order.Id] = order;
        return Task.CompletedTask;
    }

    public Task<Order?> GetAsync(OrderId id, CancellationToken cancellationToken) =>
        Task.FromResult(Store.GetValueOrDefault(id));
}

public sealed class FakeUnitOfWork : IUnitOfWork
{
    public int Commits { get; private set; }
    public Task CommitAsync(CancellationToken cancellationToken)
    {
        Commits++;
        return Task.CompletedTask;
    }
}

public sealed class PlaceOrderHandlerTests
{
    [Fact]
    public async Task Uses_catalog_price_and_reserves_stock()
    {
        var productId = new ProductId(Guid.CreateVersion7());
        var orders = new FakeOrders();
        var stock = new FakeStock();
        stock.Store[productId] = Stock.Receive(productId, 5);
        var prices = new FakePriceList();
        prices.Offers[productId] = new ProductOffer(productId, "Tea", new Money(12, "USD"));
        var uow = new FakeUnitOfWork();
        var handler = new PlaceOrderHandler(
            orders, stock, prices, new FakeIdempotency(), uow, new FakeTimeProvider(DateTimeOffset.UnixEpoch));

        var id = await handler.HandleAsync(new PlaceOrder(
            new CustomerId(Guid.CreateVersion7()),
            new Address("1 Main", "Hanoi", "10000", "VN"),
            [new PlaceOrderLine(productId, 2)],
            "idem-1",
            new Actor(new UserId(Guid.CreateVersion7()), ActorRole.Buyer, null)), default);

        var order = orders.Store[id];
        Assert.Equal(new Money(12, "USD"), order.Lines.Single().UnitPrice);
        Assert.Equal(2, stock.Store[productId].Reserved);
        Assert.Equal(1, uow.Commits);
    }
}
```

The fakes are trivial and live in the test project. A fake that is a second business implementation ("in-memory EF") will drift. Prefer a dictionary.

One integration test, not fifty, hits PostgreSQL: map an order, commit, reload, assert lines and that `Place` was not run again (event list empty on reload). That test catches a backing-field mistake the fakes cannot catch. Use the same `OrderingDbContext` and configuration as production. Testcontainers or a local database is an infrastructure concern. Keep it out of the domain test project so `dotnet test` on the domain project stays hermetic.

Assert the handler does not commit when `Reserve` throws. A fake stock that throws on the second line, and a `Commits` count of 0, locks the "one transaction" rule.

### Architecture test

The reference test earlier in the guide is the architecture test. Add one more if you want to stop handlers from taking the DbContext type by name:

```C#
[Fact]
public void Handlers_do_not_take_dbcontext_in_their_constructors()
{
    var handlerTypes = typeof(PlaceOrderHandler).Assembly.GetTypes()
        .Where(t => t.Name.EndsWith("Handler", StringComparison.Ordinal));

    foreach (var handler in handlerTypes)
    {
        var ctor = handler.GetConstructors().Single();
        Assert.DoesNotContain(
            ctor.GetParameters(),
            p => p.ParameterType.Name.Contains("DbContext", StringComparison.Ordinal));
    }
}
```

This fails closed when someone adds `OrderingDbContext` to `CancelOrderHandler` to "just this once" run a query. It is a string check on the type name. It is enough.

## Tenancy and time

### Tenant comes from the host, not the body

A client that may send `tenantId` in JSON will send someone else's. The API sets the tenant from the authenticated principal or the host name, and the handler receives it on the command if the aggregate must store it.

```C#
builder.Property<Guid>("TenantId");
builder.HasIndex("TenantId", nameof(Order.Id));
```

If every row is tenant-scoped, put `TenantId` on the aggregate as a real property set in `Place` from the command, and filter in the `DbContext` as shown earlier. A shadow property plus a filter works until a raw SQL report forgets the predicate. A column the domain knows about is harder to forget in a factory and easier to forget in SQL. The unique index for idempotency becomes `(tenant_id, idempotency_key)`. A key that is unique globally will collide when two tenants reuse a client-generated UUID. Scope the uniqueness to the tenant.

The domain rule is not "tenants exist". The domain rule is "this order belongs to the tenant that placed it and does not move". There is no `ChangeTenant` method.

### Time and ids are inputs

`Order` takes `DateTimeOffset now` and `OrderId id`. The handler gets the clock from `TimeProvider` and the id from `OrderId.New()`. Tests pass `DateTimeOffset` literals. Do not call `DateTime.Now` (local, ambiguous around DST) anywhere on this path. Do not call `DateTime.UtcNow` inside the aggregate; call it at most in the handler via `TimeProvider`, so a test can move the sweeper's cutoff without sleeping for fifteen minutes.

`FakeTimeProvider` in Microsoft.Extensions.TimeProvider.Testing, or a one-line subclass:

```C#
public sealed class FixedTimeProvider(DateTimeOffset now) : TimeProvider
{
    public override DateTimeOffset GetUtcNow() => now;
}
```

One clock read per use case (`var now = time.GetUtcNow()`), then pass `now` into `Place`, `Reserve`, and the outbox. Two reads in one handler can stamp the order at `T` and the stock event at `T+1ms` for no reason. Worse, a clock read inside `Order.Place` and another inside the handler disagree in tests.

## Modular monolith

A modular monolith is one deployable process and one database server, with a separate model per bounded context. Ordering and Billing ship in the same executable and can use the same PostgreSQL instance. They do not share an `Order` class, a `DbContext`, or a migration history.

It is the step before microservices. You get a boundary the compiler can see, without a network hop between place-order and create-invoice. Split a module into its own service later, when a real constraint shows up (scale, a team that must deploy alone, a datastore that cannot live in this instance). Splitting on day one because a diagram has boxes means you operate a distributed system whose modules still share types through a "common" project.

| | One codebase, one model | Modular monolith | Microservices |
|--|-------------------------|------------------|---------------|
| Deploy | One process | One process | One process per service |
| Domain model | One `Order` used by every feature | One model per module | One model per service |
| How modules talk | Method calls on each other's entities | Contracts and integration events, in process | HTTP or a broker, across the network |
| Data | One schema, foreign keys everywhere | One schema per module, no cross-schema foreign keys | One database per service |
| Failure you take on | Every change can break every feature | A wrong project reference | Deploys, timeouts, partial failure |

[Bounded contexts](#bounded-contexts) are the modules. [Do not share a transaction across modules](#do-not-share-a-transaction-across-modules) is the rule against enlisting both `DbContext`s in one `TransactionScope`.

### One process, many modules

```text
src/
  Host/                          # composition root. No Order, no Invoice
  Ordering/
    Domain/                      # internal to this assembly
    Features/Place/              # vertical slices, below
    Features/Cancel/
    Infrastructure/              # OrderingDbContext, EF, outbox
    OrderingModule.cs            # AddOrdering + MapOrdering
  Ordering.Contracts/            # public: OrderPlacedV1, OrderCancelledV1
  Billing/
    Domain/
    Features/
    Infrastructure/
    BillingModule.cs
  Billing.Contracts/
```

`Host` references `Ordering` and `Billing`. `Billing` references `Ordering.Contracts`. `Billing` does not reference `Ordering`. That missing reference is the boundary. A folder named `Modules/Billing` inside one project does not stop `using Ordering.Domain`.

Four projects per module (Domain, Application, Infrastructure, Api) times five modules is twenty projects before a feature exists. One assembly per module plus a thin Contracts assembly is the same [dependency rule](#the-dependency-rule), with `internal` on the aggregate instead of a separate csproj. Split the module into Domain and Infrastructure when people keep calling EF from a feature folder and review is not catching it. The host stays the only composition root.

### What a module owns

Each module owns the things that must change together and must not be edited by the neighbor:

| Owned by Ordering | Not owned by Ordering |
|-------------------|------------------------|
| `Order`, `Stock`, status transitions | `Invoice`, tax lines |
| `OrderingDbContext`, schema `ordering`, its `__ef_migrations` | Billing's schema and history table |
| Outbox rows for `ordering.order_placed.v1` | The consumer that creates the invoice |
| HTTP under `/orders` | HTTP under `/invoices` |
| The catalog price port and its adapter | The payment vendor's invoice object |

`CustomerId` may be a shared value type if both modules agree it is a `Guid` wrapper and nothing else. The moment one module wants behavior on it, copy the struct into that module. See [shared kernel](#shared-kernel-published-language-anti-corruption).

No foreign key from `billing.invoices.order_id` to `ordering.orders.id`. PostgreSQL will allow a cross-schema FK. The FK makes Ordering's delete and Billing's migration one operation. Store the `Guid` and treat a missing order as an event that has not arrived yet, or as a 404 on a query port, not as a database constraint the other team can trip over.

### How modules talk

Three legal ways. The first is the default.

**1. An integration event, after commit.** Ordering writes `ordering.order_placed.v1` in its own transaction. Billing's inbox consumer creates the invoice. Ordering does not call `Invoice.Create`. Billing being slow does not roll back the checkout. The payload is the contract: `OrderPlacedV1` with `Guid`s and decimals, in `Ordering.Contracts`. See [events and the outbox](#events-and-the-outbox).

**2. A query port in Contracts, for a read that cannot wait for an event.** Billing sometimes needs the total of an order that already exists and was not on the event it has. Put the interface in Contracts, the implementation in Ordering, and register it in the host. Billing depends on the interface only.

```C#
// Ordering.Contracts
public interface IOrderTotals
{
    Task<OrderTotal?> FindAsync(Guid orderId, CancellationToken cancellationToken);
}

public sealed record OrderTotal(Guid OrderId, decimal Amount, string Currency);
```

```C#
// Ordering.Infrastructure — Billing never sees this class
public sealed class EfOrderTotals(OrderingDbContext db) : IOrderTotals
{
    public async Task<OrderTotal?> FindAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var row = await db.Orders.AsNoTracking()
            .Where(o => o.Id == new OrderId(orderId))
            .Select(o => new
            {
                Amount = o.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity),
                Currency = o.Lines.Select(l => l.UnitPrice.Currency).FirstOrDefault()
            })
            .SingleOrDefaultAsync(cancellationToken);

        return row is null || row.Currency is null
            ? null
            : new OrderTotal(orderId, row.Amount, row.Currency);
    }
}
```

The port returns a contract record, not `Order`. If Ordering is later extracted to another process, `HttpOrderTotals` implements the same interface and Billing's handlers stay. That is the [driven port](#one-port-two-driven-adapters) at module scale.

**3. HTTP inside the process.** Only when a module is already destined to leave, and you want the adapter in place. `HttpClient` to `localhost` is still a network stack, timeouts, and no shared transaction. Prefer the in-process port until the split is real.

#### ❌ BAD — the neighbor is a project reference away

```C#
// Billing.csproj references Ordering.csproj
public sealed class CreateInvoiceHandler(OrderingDbContext ordering, BillingDbContext billing)
{
    public async Task HandleAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var order = await ordering.Orders.Include(o => o.Lines)
            .SingleAsync(o => o.Id == new OrderId(orderId), cancellationToken);
        billing.Invoices.Add(Invoice.From(order)); // Billing now depends on Order's shape
        await billing.SaveChangesAsync(cancellationToken);
    }
}
```

Billing's compile depends on Ordering's entity. A private setter change, a rename of `Lines`, or a new required field on `Order` breaks invoices. The handler also holds two contexts and can save one and throw on the other.

#### ✅ GOOD — Billing sees a contract

The invoice handler takes `OrderPlacedV1` from the inbox, or `IOrderTotals` when it must read. It constructs `Invoice` from those fields. `Order` stays `internal` in the Ordering assembly.

### The host composes modules

`Program.cs` in `Host` is the composition root. It does not contain `Order.Place`. Each module exposes `Add` and `Map`. The pattern is the same as [module registration](DotnetPattern.md#fluent-module--vertical-slice-registration); the host lists modules instead of scanning the world.

```C#
public static class OrderingModule
{
    public static IServiceCollection AddOrdering(this IServiceCollection services, IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("Ordering")
            ?? throw new InvalidOperationException("Connection string 'Ordering' is missing.");

        services.AddDbContext<OrderingDbContext>(options =>
            options.UseNpgsql(connectionString, npgsql =>
                npgsql.MigrationsHistoryTable("__ef_migrations", "ordering")));

        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IOrderTotals, EfOrderTotals>();
        services.AddScoped<PlaceOrderHandler>();
        services.AddScoped<CancelOrderHandler>();
        return services;
    }

    public static IEndpointRouteBuilder MapOrdering(this IEndpointRouteBuilder app)
    {
        var orders = app.MapGroup("/orders").WithTags("Orders");
        PlaceOrderEndpoint.Map(orders);
        CancelOrderEndpoint.Map(orders);
        return app;
    }
}
```

```C#
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOrdering(builder.Configuration);
builder.Services.AddBilling(builder.Configuration);

var app = builder.Build();
app.MapOrdering();
app.MapBilling();
app.Run();
```

`AddOrdering` is idempotent: a second call must not register a second `DbContext` pool by accident. Use `TryAdd` for services a module might share (`TimeProvider`). Do not `BuildServiceProvider()` inside `AddOrdering`.

One connection string can point both modules at the same instance. The schemas differ (`ordering`, `billing`). Two connection strings are how you later move Billing to another instance without editing handlers.

### Enforce the module boundary

`internal` on `Order` stops other assemblies from naming the type. It does not stop Billing from referencing the Ordering project and using every `public` type you forgot. Keep the implementation assembly's public surface to `OrderingModule`, the handlers the host maps, and the contract interfaces you meant to publish. Prefer those interfaces in `Ordering.Contracts`, so the implementation assembly has almost no public API.

A test locks the reference list:

```C#
[Fact]
public void Billing_does_not_reference_the_ordering_implementation()
{
    var names = typeof(CreateInvoiceHandler).Assembly
        .GetReferencedAssemblies()
        .Select(a => a.Name)
        .ToArray();

    Assert.DoesNotContain(names, static n => n == "Ordering");
    Assert.Contains(names, static n => n == "Ordering.Contracts");
}
```

`InternalsVisibleTo` is for the Ordering test project, so tests can construct `Order` while Billing cannot. Do not add `InternalsVisibleTo` for Billing.

### When to split a module out

Move Ordering to its own deployable when at least one of these is true:

- Checkout traffic needs more instances than invoicing, and scaling the whole host wastes the rest.
- A team must ship Ordering on a cadence Billing's release train blocks, and the contract between them is already stable.
- Ordering needs a different datastore or a hard isolation boundary (a PCI scope around payments is the usual one).

Until then, a second process adds failure modes the monolith does not have: the event bus is down, the invoice call times out, the order exists and the invoice does not, and you now need the outbox you should have had inside the process anyway. Build the outbox and the contract first. The process split is then a change of adapter, not a rewrite of `Order`.

## Vertical slice

A vertical slice is one use case, laid out so that the files for that use case sit together. Place order is a slice. Cancel order is a slice. The customer order list is a slice. You change one feature by staying in one folder.

A slice is not a bounded context and not a module. Ordering is the module. Place, Cancel, Ship, and the list are slices inside it. They share `Order` and `OrderingDbContext`. They do not each own a copy of the status machine.

This is the folder rule from [feature folders](#feature-folders-inside-a-layer), plus the rule about what is allowed to be shared. Jimmy Bogard's vertical-slice style is the same idea. A `IRequest` / mediator type is optional. The slice exists without one.

### A slice is one change

| | Layered folders | Vertical slice |
|--|-----------------|----------------|
| Place order touches | `Controllers/`, `Handlers/`, `Validators/`, `Dtos/` | `Features/Place/` |
| Question it answers | Where do all handlers live? | What files does this feature need? |
| Risk | A new rule lands in whichever layer the author had open | A new rule lands next to the only handler that should enforce it |
| Shared model | Easy to turn into a dumping ground of services | Still shared, but only the aggregate and the ports, not the DTOs |

#### ❌ BAD — technical folders

```text
Controllers/OrderController.cs      # Place, Cancel, Get, List
Application/Commands/               # every command in the module
Application/Handlers/
Application/Validators/
Infrastructure/Repositories/
```

Cancel is a method on a controller that also places orders, a handler in a different project, and a validator discovered by naming convention. A change to cancel opens four trees. A change to place risks editing the same controller method by mistake.

#### ✅ GOOD — one folder per use case

```text
Ordering/
  Domain/Order.cs
  Domain/Stock.cs
  Features/
    Place/
      PlaceOrderEndpoint.cs
      PlaceOrder.cs
      PlaceOrderHandler.cs
      PlaceOrderRules.cs
    Cancel/
      CancelOrderEndpoint.cs
      CancelOrder.cs
      CancelOrderHandler.cs
    ListByCustomer/
      ListOrdersEndpoint.cs
      ListOrdersQuery.cs
      EfListOrders.cs
  Infrastructure/
    OrderingDbContext.cs
    EfOrderRepository.cs
```

Open `Features/Cancel/` and the endpoint, the command, and the handler are there. The status rule is not in the folder. It is on `Order.Cancel`, because every slice that cancels must hit the same method. The slice is allowed to know the aggregate. The slice is not allowed to set `Status`.

### Slice inside a module

A write slice calls one handler and one commit. A read slice does not load the aggregate. `ListByCustomer` is a slice whose implementation is a query port, as in [reads](#reads-do-not-go-through-the-aggregate). Putting that query in `Features/ListByCustomer/` keeps the SQL next to the route that needs it, instead of growing `OrderRepository` with every screen.

```C#
public static class CancelOrderEndpoint
{
    public static void Map(IEndpointRouteBuilder orders)
    {
        orders.MapPost("/{id}/cancel", async (
            OrderId id,
            HttpContext http,
            CancelOrderHandler handler,
            CancellationToken cancellationToken) =>
        {
            await handler.HandleAsync(new CancelOrder(id, http.ToActor()), cancellationToken);
            return Results.NoContent();
        });
    }
}
```

The endpoint is the driving adapter for this slice. It stays in the feature folder, not in a `Controllers` folder that every module dumps into. `ToActor()` can live next to the host's auth setup; the slice calls it and does not parse claims itself in five endpoints.

Slices inside one module may share ports (`IOrderRepository`, `IUnitOfWork`, `TimeProvider`). A port used by only one slice can live in that slice's folder (`IListOrders` beside `EfListOrders`). When a second slice needs it, move the interface to `Ports/`. Moving it later is cheaper than a `Ports` folder of interfaces with one caller.

Slices do not call each other. `PlaceOrderHandler` does not call `CancelOrderHandler` to undo a failed line. It throws, and the transaction does not commit. A slice that orchestrates two other slices is a workflow. Put that workflow in its own feature folder (`Features/Checkout/`) that calls the domain methods, or publish an event and let the other slice's consumer run. Do not chain handlers so the call stack is Place → Reserve → Email → Invoice inside one HTTP request.

### Registration stays with the slice

The module's `MapOrdering` calls each slice's `Map`. The module's `AddOrdering` registers each slice's handler. A new slice is a folder plus two lines in `OrderingModule`. It is not an edit to a 2,000-line `Program.cs`.

```C#
public static IServiceCollection AddOrdering(this IServiceCollection services, IConfiguration configuration)
{
    services.AddOrderingPersistence(configuration);
    services.AddScoped<PlaceOrderHandler>();
    services.AddScoped<CancelOrderHandler>();
    services.AddScoped<IListOrders, EfListOrders>();
    return services;
}

public static IEndpointRouteBuilder MapOrdering(this IEndpointRouteBuilder app)
{
    var orders = app.MapGroup("/orders");
    PlaceOrderEndpoint.Map(orders);
    CancelOrderEndpoint.Map(orders);
    ListOrdersEndpoint.Map(orders);
    return app;
}
```

Discovery by reflection (`every class ending in Handler`) will register a test double, a base class, and the wrong lifetime. An explicit list is short when each module owns it. If you do scan, scan `IModule` only, cache the result, and keep feature services in `Add`. See [DotnetPattern.md](DotnetPattern.md#fluent-module--vertical-slice-registration).

### What slices share

| Shared inside the Ordering module | Private to the slice |
|-----------------------------------|----------------------|
| `Order`, `OrderLine`, `Stock` | `PlaceOrderRequest`, `CancelOrder` command |
| `OrderingDbContext` | The endpoint and its route |
| `IOrderRepository`, `IUnitOfWork` | A query used by one screen |
| `OrderPlaced` and the outbox mapping | Validator for that command's shape |
| `TimeProvider` | Log messages for that use case |

#### ❌ BAD — a slice that is a private domain

```C#
namespace Ordering.Features.Cancel;

public sealed class Order
{
    public Guid Id { get; set; }
    public string Status { get; set; } = "";
}

public sealed class CancelOrderHandler(OrderingDbContext db)
{
    public async Task HandleAsync(Guid id, CancellationToken cancellationToken)
    {
        var order = await db.Set<Order>().SingleAsync(o => o.Id == id, cancellationToken);
        order.Status = "Cancelled";
        await db.SaveChangesAsync(cancellationToken);
    }
}
```

Place has another `Order` in another namespace, or this handler bypasses `Order.Cancel`. The module no longer has one status machine. Vertical slice organized the files and deleted the model.

#### ✅ GOOD — the slice is thin, the aggregate is shared

`CancelOrderHandler` in `Features/Cancel/` loads the module's `Order` and calls `Cancel`. `PlaceOrderHandler` in `Features/Place/` calls `Place` on that same type. A test of the status machine still lives next to `Domain/Order.cs` and does not boot either endpoint.

Across modules, a slice is not shared. Billing does not reference `Features/Place`. It references `OrderPlacedV1`. The slice is an implementation detail of Ordering, the same way `EfOrderRepository` is.

## Worked examples

The sections above are the rules. These traces put the same rules on one timeline: one `POST /orders`, one ship, the EF mappings a unit test will not catch, a transaction that must not include Billing, and two writers on one row.

### A place request, end to end

One `POST /orders` that succeeds crosses every layer once. Reading it as a single story is how you check that a new feature did not skip a step.

```text
HTTP
  auth middleware            principal → Actor, tenant id
  OrderEndpoints             JSON → PlaceOrder (no price field)
Application
  PlaceOrderHandler
    shape check              key present, 1..50 lines
    idempotency lookup       return existing OrderId if the key won before
    IPriceList.FindAsync     catalog name + Money, per line
    Order.Place              invariants, OrderPlaced in memory
    Stock.Reserve            per line, StockReserved in memory
    IUnitOfWork.CommitAsync
Infrastructure
  OutboxInterceptor          domain events → ordering.order_placed.v1 rows
  SaveChanges                one PostgreSQL transaction
    INSERT orders, order_lines
    UPDATE stock
    INSERT outbox_messages
API
  201 Created                { "orderId": "..." }   the aggregate is not the body
later
  OutboxProcessor            claim, publish, mark processed
  billing inbox              dedupe on EventId, create invoice
```

If any step throws before `CommitAsync`, the database does not have the order, the reserve, or the outbox row. The in-memory objects die with the request scope. There is nothing to compensate.

If `CommitAsync` throws a unique violation, the transaction rolls back the same way. The client retries with the same `Idempotency-Key`. The lookup returns the winner.

If `CommitAsync` succeeds and the process dies before the HTTP response is flushed, the client retries. The lookup returns the same id. The outbox row is still unpublished and the worker sends it once (plus at-least-once retries the inbox collapses).

If the catalog port throws, stock is not reserved, because `Reserve` is after the price loop and `Commit` has not run. Do not reorder this so that `Reserve` happens before you know the product exists. You would release in a `catch`, and a crash between reserve and release leaks a hold. The order of calls in the handler is part of the design.

```C#
// prices first (no writes), then one graph of tracked changes, then one commit
foreach (var requested in command.Lines)
{
    var offer = await prices.FindAsync(...) ?? throw new NotFoundException(...);
    lines.Add(new OrderLine(...));
}
var order = Order.Place(...);
await orders.AddAsync(order, cancellationToken);
foreach (var line in order.Lines)
    (await stockItems.GetAsync(line.ProductId, cancellationToken))!.Reserve(...);
await unitOfWork.CommitAsync(cancellationToken);
```

A handler that calls `CommitAsync` inside the line loop will persist an order whose later line failed the price list. Delete that pattern in review.

### Ship commits the sale

`Reserve` increments `Reserved` and leaves `OnHand` alone. `Available` drops. The units are still in the building. `Ship` is the moment they leave, so the stock root has a third transition, called in the same commit as `order.Ship`.

```C#
public void CommitSale(int quantity, OrderId orderId, DateTimeOffset now)
{
    if (quantity <= 0)
        throw new DomainRuleException("Quantity must be positive.");
    if (quantity > Reserved)
        throw new DomainRuleException("Cannot ship more than reserved.");
    if (quantity > OnHand)
        throw new DomainRuleException("Cannot ship more than on hand.");

    Reserved -= quantity;
    OnHand -= quantity;
    _events.Add(new StockCommitted(Guid.CreateVersion7(), ProductId, orderId, quantity, now));
}
```

```C#
public sealed class ShipOrderHandler(
    IOrderRepository orders,
    IStockRepository stockItems,
    IUnitOfWork unitOfWork,
    TimeProvider time)
{
    public async Task HandleAsync(ShipOrder command, CancellationToken cancellationToken)
    {
        if (command.Actor.Role != ActorRole.Staff)
            throw new ForbiddenException();

        var order = await orders.GetAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException("Order was not found.");

        var now = time.GetUtcNow();
        order.Ship(now);

        foreach (var line in order.Lines)
        {
            var stock = await stockItems.GetAsync(line.ProductId, cancellationToken)
                ?? throw new DomainRuleException($"No stock row for {line.ProductId}.");
            stock.CommitSale(line.Quantity, order.Id, now);
        }

        await unitOfWork.CommitAsync(cancellationToken);
    }
}
```

Call `order.Ship` before `CommitSale`. A placed-but-unpaid order throws from `Ship` and the stock numbers do not change. The reverse order would decrement `OnHand` and then throw, and only a disciplined rollback saves you. The rollback does save you if both calls happen before `CommitAsync` and you do not catch the exception and commit anyway. Reviewers miss the catch. The call order makes the mistake visible in a unit test that never opens a database: fake stock starts at on-hand 5, reserved 2; `Ship` on a placed order throws; reserved is still 2.

`Cancel` calls `Release` (reserved down, on-hand unchanged). `Ship` calls `CommitSale` (both down). Using `Release` on ship puts the units back on the shelf in the counter while the truck leaves. That bug is a wrong method name, which is why the names are `Release` and `CommitSale` and not `UpdateQty(-1)`.

`StockCommitted` needs an integration mapping if the warehouse context is separate. If warehouse is a screen on this service, you can skip the bus and let the ship handler be the only writer. Do not also publish a message that a second handler uses to decrement stock again.

### Mapping failures

These are the EF mistakes that produce a green domain test and a broken reload.

| Symptom | Cause | Fix |
|---------|--------|-----|
| `OrderPlaced` inserted again on every cancel | `Raise` lives in a property setter or in a constructor EF calls | `Raise` only inside `Place`, `Cancel`, `Ship`, … Private empty constructor for EF |
| Lines empty after `GetAsync` | Collection not included, or EF wrote a shadow collection because it could not see `_lines` | `Include`, and `UsePropertyAccessMode(Field)` |
| Insert fails, `null` product name | Public constructor never ran; a mapper set properties EF then saved | Application code calls `new OrderLine(...)`. No public setters for the mapper |
| Two lines, one product, both saved | Composite key `(OrderId, ProductId)` missing, EF used a surrogate | `line.HasKey("OrderId", nameof(OrderLine.ProductId))` plus the domain check |
| Money stored as one string column you cannot sum | No `ComplexProperty` / owned columns | Amount `numeric(18,2)`, currency `char(3)` |
| Second cancel wins silently | No concurrency token | `xmin` or `rowversion`, map `DbUpdateConcurrencyException` to 409 |
| Idempotency key allows two rows | Unique index not created, or filter excludes the rows you insert | Migration contains the unique index; a test inserts twice |
| Query filter hides a row the unique index still sees | Soft-delete filter without a matching unique index predicate | Cancel is a status, or the index has the same `WHERE` |

Configure the model in `IEntityTypeConfiguration<T>`. A data annotation on the entity (`[Required]`, `[Table]`) is an EF reference in the domain project, which fails the dependency test the day you add the package, or it is a stringly convention nobody runs. Keep attributes out of `Ordering.Domain`.

Migrations:

```bash
dotnet ef migrations add PlaceOrder --project Ordering.Infrastructure --startup-project Ordering.Api --output-dir Persistence/Migrations
```

The startup project is the host because it has the connection string and `AddOrderingInfrastructure`. Review the generated `Up` method. EF will try to create a table for `DomainEvents` if you forgot `Ignore`. It will create a `Total` column if you forgot `Ignore` on the calculated property. Throw that migration away, fix the configuration, add it again. Hand-editing a migration you do not understand is how `xmin` gets created as a real `bigint` the application then fights over.

One migration history per context, in that context's schema. A single solution-level `Migrations` folder shared by ordering and billing will apply the wrong model to the wrong database the first time both teams ship.

### Do not share a transaction across modules

Ordering does not call `Invoice.Create` in `PlaceOrderHandler`. That call would compile only if ordering referenced billing's domain, and the place transaction would then include the invoice. Billing's database being down would refuse checkouts. The outbox row is the coupling: ordering commits without billing. Billing's consumer creates the invoice when it can. The project split for that is [modular monolith](#modular-monolith).

An in-process shortcut that is still honest: after commit, enqueue the integration event on a `Channel<T>` the billing dispatcher reads. The channel is not a transaction. A process crash drops the channel and the outbox worker is still the durable path. If you only have the channel, you do not have an outbox. Keep the table.

Do not share a transaction by enlisting both `DbContext` instances in one `TransactionScope` "so the invoice is never missing". You reintroduced the distributed decision you split the contexts to avoid, and `TransactionScope` plus async has its own history of flowing onto the wrong thread. One context, one `SaveChanges`, per use case.

Shared kernel, if any, is `CustomerId` in a third project both domains reference. Integration records live in contracts, not in the shared kernel. The kernel has no events, no EF, no handlers.

### Two writers, one row

Two cancel requests load the same placed order. `xmin` is `100` on both reads.

| Time | Request A | Request B | Row |
|------|-----------|-----------|-----|
| 1 | `GetAsync`, xmin 100 | | `Placed`, xmin 100 |
| 2 | | `GetAsync`, xmin 100 | `Placed`, xmin 100 |
| 3 | `Cancel`, `SaveChanges` | | `Cancelled`, xmin 101 |
| 4 | | `Cancel`, `SaveChanges` | update matches xmin 100, **0 rows**, `DbUpdateConcurrencyException` |

Request B's in-memory object is `Cancelled`. The database was already `Cancelled` by A. B did not overwrite a newer business state in this particular race (both wanted cancel). The token still fires, because B's write was based on a stale snapshot. Map it to 409. If B's client retries, the handler loads `Cancelled` and `Cancel` throws `DomainRuleException` (400). The user sees "already cancelled", which is the truth.

The stock race is the one that loses money if you skip the token:

| Time | Checkout A | Checkout B | `OnHand` | `Reserved` |
|------|------------|------------|----------|------------|
| 1 | read Available 1 | | 1 | 0 |
| 2 | | read Available 1 | 1 | 0 |
| 3 | `Reserve(1)`, save | | 1 | 1 |
| 4 | | `Reserve(1)`, save, no token | 1 | 1 (lost update) or 2 (if you wrote `reserved = reserved + 1` in SQL) |

An EF update without a token writes the **values** it loaded: B's entity has `Reserved == 1` and overwrites A's `Reserved == 1` with `1`. You sold two units and reserved one. `ExecuteUpdate` with `SET reserved = reserved + quantity` and a `WHERE on_hand - reserved >= quantity` is the SQL form of the same rule. The aggregate plus a concurrency token is the form that keeps the rule in `Reserve`. Use the SQL predicate in the repository only when the hot row's retry rate is too high for optimistic conflicts, and keep the `WHERE` so a lost precondition cannot update.

```C#
var updated = await db.Stock
    .Where(s => s.ProductId == productId && s.OnHand - s.Reserved >= quantity)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(s => s.Reserved, s => s.Reserved + quantity), cancellationToken);

if (updated == 0)
    throw new DomainRuleException("Not enough stock.");
```

That statement is infrastructure. It bypasses `Stock.Reserve` and the domain event. If you take this path for the hot SKU, the repository method is `TryReserve` on the port and it returns whether the update hit, and the handler raises or the SQL is paired with an outbox insert in the same transaction. Do not leave a second writer that "also updates stock" through `ExecuteUpdate` while `ShipOrderHandler` uses the aggregate. One writer per column.

## What to skip

- **Generic `IRepository<T>` returning `IQueryable<T>`.** Each root has its own load rules. The queryable port is EF in the application layer.
- **A `Domain` project of POCOs with public setters.** The rules are still in the controller. The folders are a rename.
- **One `DbContext` shared by ordering and billing.** One migration history, one word `Order` meaning two things, and a navigation the teams will use because it is there.
- **Four class libraries on the first CRUD screen.** Split when a second adapter or a real invariant shows up. Keep the dependency direction in folders until then.
- **A mediator as the architecture.** It hides the handler's constructor. Add a decorator when every use case shares a real pipeline.
- **`DbContext` or `IEmailSender` injected into `Order`.** The handler loads, the entity decides, the unit of work saves, the consumer sends mail after commit.
- **Domain events dispatched before commit.** In-process handlers that send mail or call HTTP will do that work for a row that rolls back.
- **Publishing the CLR type name on the bus.** Renames become breaking changes. Store `ordering.order_placed.v1`.
- **Client-supplied prices and tenant ids.** The price comes from `IPriceList`. The tenant comes from the principal.
- **`OFFSET` pagination on the order list.** Page with `(placed_at, id)`.
- **Lazy-loading proxies** so the model can reach into another aggregate by accident.
- **Soft-delete on orders that still have a unique idempotency key.** Cancel is a status. A hidden row still occupies the unique index.
- **Holding a stock row lock across a payment HTTP call.** Commit the reserve, then talk to the vendor, then mark paid from the webhook.
- **An in-memory "unit of work" interface plus a repository `Save` on every method.** You will commit partial graphs. One `CommitAsync` per handler.
- **AutoMapper profiles that write into aggregate setters** you made public for the mapper. Map DTOs at the edge. Call `Place` and `Cancel` for changes.
- **Sharing `Money` in a shared kernel the moment billing needs negative amounts.** Duplicate the struct.
- **Six projects because a hexagon has six sides.** HTTP and PostgreSQL are two adapters. A folder is enough until a SDK should not compile into the web host.
- **A port for `Order`, `Money`, and `OrderLine`.** Those are the model. A port is a boundary you replace: the price list, the gateway, the repository.
- **One `DbContext` for every module because there is one process.** The process is shared. The model, the schema, and the migration history are not.
- **A vertical slice with its own `Order` class.** Slices in one module share the aggregate. Copying the entity per feature puts the status rules back in each handler.
- **MediatR as the definition of a slice.** A folder, a handler, and `Map` on the endpoint are the slice. A mediator is optional. See [a pipeline without a mediator](#a-pipeline-without-a-mediator).

## Checklist

Before you add a project, a port, or an event, answer these:

1. What sentence does the business use, and is that the method name?
2. What must stay true after a crash at any line of the handler? That sentence is the transaction boundary.
3. Is this rule about one aggregate's fields (method), about two rows in this database (unique index or one commit), or about another service (outbox and a process)?
4. Can a second caller skip the rule by setting a property? If yes, the setter is public and the design is still anemic.
5. Does load-from-EF raise a domain event? If yes, a setter or a factory is on the materialization path. Move the `Raise` into the method only the handler calls.
6. What is the concurrency behavior when two requests load the same row? Token and 409, or a lost update.
7. What happens when the client retries? Idempotency key on place. Inbox on consume.
8. What is the HTTP contract? A response DTO and problem details, not the aggregate and not `exception.ToString()`.
9. Which project would fail to compile if someone used EF in the domain? If the answer is "none", the dependency rule is a comment.
10. Does the list screen load the aggregate? It should not.
11. If a second driver showed up tomorrow (a queue, a test, a CLI), could it call the same handler? If the handler's constructor names `DbContext` or a vendor SDK, that driver will drag the technology with it.
12. Which module owns this table? If Billing's csproj references Ordering's assembly to read `Order`, the modular boundary is already gone. Billing gets a contract or an event.

## Related guides

- [DotnetPattern.md](DotnetPattern.md) — unit of work, strongly-typed IDs, `TimeProvider`, decorators, scoped `DbContext`, hosted services
- [AspNetCoreGuidance.md](AspNetCoreGuidance.md) — `HttpContext`, exception handlers, request DI
- [AsyncGuidance.md](AsyncGuidance.md) — `async`/`await`, `CancellationToken`, `BackgroundService`, not blocking the outbox loop
- [HttpClientGuidance.md](HttpClientGuidance.md) — `HttpClient` lifetime for `HttpPriceList` and the payment gateway
- [Database_Indexing.md](Database_Indexing.md) — indexes for the keyset list, unique idempotency keys, and hot stock rows

