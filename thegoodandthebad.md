# TSQR (TownsSquare) — DDD Design Audit: The Good & The Bad

---

## Table of Contents

1. [The Bad: Design Issues](#the-bad-design-issues)
   - [1. Aggregate Boundary Violations](#1-aggregate-boundary-violations)
   - [2. Domain Events: Defined but Dead](#2-domain-events-defined-but-dead)
   - [3. Inconsistent Error Handling Strategy](#3-inconsistent-error-handling-strategy)
   - [4. Debug Leak in Production Code](#4-debug-leak-in-production-code)
   - [5. The Not-Aggregate Roots](#5-the-not-aggregate-roots)
   - [6. Communities: Broken Transactional Boundary in QuickRegister](#6-communities-broken-transactional-boundary-in-quickregister)
   - [7. Soup Kitchen: Public Constructors, Zero Validation](#7-soup-kitchen-public-constructors-zero-validation)
   - [8. Identity: Completely Anemic Domain](#8-identity-completely-anemic-domain)
   - [9. Reservation Queue Ignores Its Own Domain Service](#9-reservation-queue-ignores-its-own-domain-service)
   - [10. Slug Generation is Naive](#10-slug-generation-is-naive)
   - [11. Hardcoded Fine Rate Despite Having a Policy Aggregate](#11-hardcoded-fine-rate-despite-having-a-policy-aggregate)
   - [12. Policy Entity Is Orphaned](#12-policy-entity-is-orphaned)
   - [13. Supported but Not Banned: Reinstate Allows Banned → Active](#13-supported-but-not-banned-reinstate-allows-banned--active)
   - [14. Support Context Has No Domain Model](#14-support-context-has-no-domain-model)
   - [15. Value Object Duplication Across Contexts](#15-value-object-duplication-across-contexts)
   - [16. No `INoSqlRepository` Contract for Non-Relational Stores](#16-no-inosqlrepository-contract-for-non-relational-stores)
   - [17. No Folder Structure Separation for Persistence Technologies](#17-no-folder-structure-separation-for-persistence-technologies)
   - [18. No Pure Abstraction for Persistence](#18-no-pure-abstraction-for-persistence)
2. [The Good: What Has Been Done Right](#the-good-what-has-been-done-right)
   - [1. Tool Library Domain Model is Genuinely Strong](#1-tool-library-domain-model-is-genuinely-strong)
   - [2. In-Transaction Event Dispatch (eShop Pattern)](#2-in-transaction-event-dispatch-eshop-pattern)
   - [3. Clean Architecture Discipline](#3-clean-architecture-discipline)
   - [4. Generic Persistence Infrastructure](#4-generic-persistence-infrastructure)
   - [5. CQRS Done Right](#5-cqrs-done-right)
   - [6. Ubiquitous Language is Consistent](#6-ubiquitous-language-is-consistent)
   - [7. Autheo Security Design](#7-autheo-security-design)
   - [8. Scoped Multi-Tenancy via CommunityId](#8-scoped-multi-tenancy-via-communityid)
3. [Summary: Verdict](#summary-verdict)

---

## The Bad: Design Issues

### 1. Aggregate Boundary Violations

**Cross-aggregate transactions in Tool Library.** Three command handlers update multiple aggregates in a single database transaction:

| Handler | Aggregates Mutated | Problem |
|---|---|---|
| `RegisterToolCommandHandler` | `Tool` + `InventoryItem` | Creating a catalog entry and a physical copy are conceptually distinct operations with separate lifecycles. If inventory creation fails, the tool record is orphaned. |
| `LoanToolCommandHandler` | `Loan` + `InventoryItem` | `InventoryItem.Loan()` changes item status AND `Loan.Create()` persists a new loan record in the same transaction. A loan is a separate concern from inventory status. |
| `ReserveToolCommandHandler` | `Reservation` + `InventoryItem` | The handler mutates `InventoryItem.Reserve()` (which sets `ReservationDate` / `ReservationMemberId` on the item) AND creates a `Reservation` aggregate. |

**Reservation state is duplicated** across two aggregates (`InventoryItem.cs:46-47` has `ReservationDate` and `ReservationMemberId` properties that shadow the `Reservation` aggregate). This creates a dual-write risk — if one write succeeds and the other fails, the system has inconsistent reservation state.

In proper DDD, a transaction should span exactly one aggregate. The side effect on `InventoryItem` should happen through a domain event handler, not in the same command handler.

### 2. Domain Events: Defined but Dead

**Tool Library** defines 14 domain events but the wiring is incomplete:
- `PickupReminderEvent`, `ReturnReminderEvent`, `ToolReservedEvent` — defined in the domain layer as record types, **never raised** by any aggregate behavior.
- Four event handlers are empty stubs that return `Task.CompletedTask`:
  - `ToolMarkedForRepairNotificationHandler`
  - `NextInLineNotificationHandler`
  - `PickupReminderHandler`
  - `ReturnReminderHandler`

**Every other bounded context has zero domain events.** Communities, Soup Kitchen, Identity, and Support all inherit the `Entity` base class which has a `DomainEvents` collection, but no aggregate in those contexts ever calls `AddDomainEvent()`. The infrastructure for dispatching events doesn't exist outside of tool-lib.

### 3. Inconsistent Error Handling Strategy

Three different error patterns coexist across the codebase:

| Bounded Context | Error Pattern | Example |
|---|---|---|
| Tool Library | `Result<T>` / `Result` consistently | `Tool.Create()` returns `Result<Tool>`. Validation errors flow through the pipeline explicitly. |
| Autheo | **Exceptions** in domain layer, `Result<T>` in application layer | `User.Create()` throws `ArgumentException` when email is empty. `RefreshToken.Rotate()` throws `InvalidOperationException` if token is inactive. The application handlers use `Result<T>` so these exceptions crash the process. |
| Communities / Soup Kitchen | **No validation at all** | `new Community(...)` accepts empty `Name`. `new Event(...)` accepts `DateTime.MinValue` as `EventDate`. |
| Identity | **Anemic public setters** | `UserProfile.Name { get; set; }` — no encapsulation, no validation, no invariants. |

Autheo's mixed approach is the most dangerous: domain methods throw exceptions while application handlers return `Result<T>`. A validation failure in `User.Create()` will crash the request instead of returning a `ValidationError` to the client.

### 4. Debug Leak in Production Code

In `Entity.cs:56-59` (tool-lib domain base class, inherited by every aggregate):

```csharp
public Task<int> TestAsnc()
{
    return Task.FromResult(3);
}
```

A misspelled test method (`TestAsnc` instead of `TestAsync`) that returns the integer `3` is leaking into production. Every entity in the system inherits this method, making it available on every aggregate root — it serves no purpose and pollutes the domain API.

### 5. The Not-Aggregate Roots

Several classes marked `IAggregateRoot` should not have that status:

- **`Country`, `City`, `Neighbourhood`** (tsqr-communities) — these are reference/lookup data. They are created once and rarely mutated. Making each an independent aggregate root forces the `QuickRegisterCommunityCommand` to commit after each creation (see issue #6). They should be value objects or a single `Location` aggregate.

- **`Manufacturer`** (tsqr-tool-lib) — embedded within the `Tool` aggregate (referenced by `Tool.Manufacturer`) but given its own repository `IRepository<Manufacturer, ManufacturerId>`. It has no independent lifecycle outside of being referenced by a `Tool`. Should be a value object or a child entity within the `Tool` aggregate.

- **`Meal`, `Guest`, `Volunteer`, `Donation`** (tsqr-soup-kitchen) — these reference an `Event` by `EventId` but are independent aggregate roots. A `Meal` has no existence without an `Event`, so it should arguably be a child of `Event`. The current design makes it possible to create meals that belong to non-existent events.

### 6. Communities: Broken Transactional Boundary in QuickRegister

`QuickRegisterCommunityCommand.cs` calls `SaveChangesAsync()` after each individual entity creation:

```csharp
await countryRepo.UnitOfWork.SaveChangesAsync(ct);        // committed NOW
await cityRepo.UnitOfWork.SaveChangesAsync(ct);            // committed NOW
await neighbourhoodRepo.UnitOfWork.SaveChangesAsync(ct);   // committed NOW
await communityRepo.UnitOfWork.SaveChangesAsync(ct);       // committed NOW
```

Because all four repositories share the same scoped `DapperUnitOfWork`, the first `SaveChangesAsync()` commits the transaction. Subsequent operations run in a new transaction. If the community creation fails after the country has been saved, the country record is **orphaned** — no cleanup occurs.

The entire operation should be wrapped in a single transaction with a single commit at the end.

### 7. Soup Kitchen: Public Constructors, Zero Validation

All Soup Kitchen aggregates use **public constructors** with no validation:

```csharp
public Event(string name, string description, DateTime eventDate, string location,
    int? maxGuests, int communityId) : base(new EventId(0))
{
    Name = name;            // can be null or empty — not checked
    Description = description; // can be null or empty — not checked
    EventDate = eventDate;  // can be DateTime.MinValue or in the past — not checked
    Location = location;    // can be null or empty — not checked
    Status = EventStatus.Planned;
    MaxGuests = maxGuests;  // can be 0 or negative — not checked
    ...
}
```

`Guest`, `Meal`, `Volunteer`, and `Donation` have the same issue — public constructors, no validation, no null checks, no date-range checks. This is an anemic domain model where the domain layer provides zero protection against invalid state.

### 8. Identity: Completely Anemic Domain

`UserProfile` is a plain data class with public get/set properties:

```csharp
public sealed class UserProfile
{
    public string Id { get; set; } = string.Empty;      // anyone can change the ID
    public string? FirstName { get; set; }              // can be set to anything
    public string? LastName { get; set; }
    public string? AvatarUrl { get; set; }
    public string? Bio { get; set; }
    public DateTime CreatedAt { get; set; }              // mutable — can be rewritten
    public DateTime UpdatedAt { get; set; }
    public DateTime? LockoutEnd { get; set; }
}
```

No encapsulation, no domain behavior, no invariants. The domain layer (`TSQR.Identity.Domain`) contains no aggregate root marker, no entity base class, no domain events, no value objects. It is effectively a DTO namespace.

`Role` has the same anemic pattern.

### 9. Reservation Queue Ignores Its Own Domain Service

`ReservationQueueService.ShiftQueueAfterCancellation()` exists in the domain layer but is **never called** by any application handler. When a reservation is cancelled via `ReservationCancelledEventHandler`, the next-in-line is notified but the `QueuePosition` values are never renumbered — leaving gaps (1, 3, 4 instead of 1, 2, 3).

The domain service was created to handle this, but the wiring was never completed.

### 10. Slug Generation is Naive

`Community.cs:41-42`:

```csharp
private static string GenerateSlug(string name) =>
    name.ToLowerInvariant().Replace(' ', '-').Replace("--", "-");
```

This implementation fails on:
- Special characters (no stripping of `!@#$%^&*()`)
- Consecutive hyphens beyond two (e.g., `"a---b"` → `"a---b"` not `"a-b"`)
- Leading or trailing hyphens
- Non-ASCII / Unicode characters
- **No uniqueness enforcement** — two communities can have the same slug, which will break URL routing

### 11. Hardcoded Fine Rate Despite Having a Policy Aggregate

`Loan.cs:121-125`:

```csharp
private decimal CalculateFine(TimeSpan overduePeriod)
{
    var daysOverdue = (int)Math.Ceiling(overduePeriod.TotalDays);
    return daysOverdue * 1.00m;
}
```

The `FineService` exists specifically to provide configurable fine rates (`FineService.CalculateFine(Loan, DateTime, decimal lateFeePerDay)`). The `Policy` aggregate stores per-tool-type, per-location fine rates. But `Loan.EndLoan()` ignores both and hardcodes `$1/day`.

### 12. Policy Entity Is Orphaned

`Policy` (in the `ToolAggregate` namespace) defines lending rules:
- `LateFeePerDay` — not used by `Loan.EndLoan()` (see issue #11)
- `MaxLoanDurationDays` — not checked by `LoanToolCommandHandler`
- `MaxLoanReservationDays` — not checked by `ReserveToolCommandHandler`
- `MaxRenewalCount` — no `Loan.Renew()` method exists

The `Policy` entity and its repository `IManufacturerRepository` (note: misnamed — should be `IPolicyRepository`) exist in the domain model but no application handler ever loads or enforces a policy.

### 13. Supported but Not Banned: Reinstate Allows Banned → Active

`Member.Reinstate()` in `Member.cs:213-219`:

```csharp
public Result Reinstate()
{
    if (Status != MemberStatus.Suspended && Status != MemberStatus.Banned)
        return new DomainError(...);
    Status = MemberStatus.Active;
    return Result.Success();
}
```

`Banned` is semantically a terminal state in most community moderation systems — banned members should not be reinstatable without explicit overrides. The code allows `Banned → Active` directly, contradicting the original domain analysis which explicitly states *"Banned is a terminal state (no reinstate from banned in current model)"*.

### 14. Support Context Has No Domain Model

The Support context lacks any domain aggregates:
- `ITicketsRepository` takes primitive parameters: `CreateTicketAsync(string subject, int category, string description, ...)`
- `ISupportQueries` returns flat DTO records (`TicketListItemRow`, `TicketDetailRow`, etc.)
- The closest thing to a domain model is `Enums.cs` (5 enums with label extensions)
- No entity base class, no aggregate root marker, no value objects

There is no `Ticket` aggregate, no `Incident` aggregate. Business rules like *"a resolved ticket cannot be reopened"* or *"an incident must transition from Investigating → Ongoing → Resolved"* are not enforced anywhere.

### 15. Value Object Duplication Across Contexts

`LocationId` and `CountryId` are defined in **two separate bounded contexts** (`tsqr-tool-lib` and `tsqr-communities`):

| Context | Type | Path |
|---|---|---|
| tsqr-tool-lib | `LocationId`, `CountryId` | `.../LocationAggregate/LocationId.cs` |
| tsqr-communities | `CountryId` | `.../CountryAggregate/CountryId.cs` |

These represent the same real-world concepts but are separate .NET types in separate namespaces. When the Tool Library references a `LocationId` and Communities references a `CountryId`, there is no type-level guarantee they refer to the same thing. If the databases drift out of sync, no compilation error would catch it.

### 16. No `INoSqlRepository` Contract for Non-Relational Stores

The domain layer defines `IRepository<TAggregateRoot, TId>` as the sole persistence abstraction. There is no parallel contract for non-relational stores (e.g., `IDocumentRepository<T>` or `INoSqlRepository<TAggregateRoot>`). Every repository implementation — even a hypothetical MongoDB one — must conform to the same interface designed with SQL assumptions.

A MongoDB-backed `ReservationRepository` would need to:
- Ignore `ISqlEntityMapping<T>` entirely (it's SQL-specific)
- Ignore `IUnitOfWork` (MongoDB uses session callbacks, not transactions)
- Build its own mapping infrastructure from scratch with no help from the abstraction
- Either implement `IRepository<Reservation, ReservationId>` directly or extend `SqlRepository` (which leaks Dapper internals)

The domain layer should define a family of persistence contracts — for example `IRelationalRepository<T, TId>` and `IDocumentRepository<T>` — so the choice of storage technology is explicit at the interface level, not hidden behind a single abstraction that happens to have only a Dapper implementation today.

### 17. No Folder Structure Separation for Persistence Technologies

All persistence code lives under a flat `Infrastructure/Dapper/` directory:

```
Infrastructure/Dapper/
├── DapperConnection.cs
├── DapperUnitOfWork.cs
├── ISqlConnection.cs
├── ISqlEntityMapping.cs
├── ISqlUnitOfWork.cs
├── Mappings/
│   ├── ToolMapping.cs
│   ├── MemberMapping.cs
│   ├── ...
├── Repositories/
│   ├── DapperReservationRepository.cs
│   ├── ToolRepository.cs
│   ├── ...
└── SqlRepository.cs
```

There is no `Infrastructure/Persistence/` parent to group different storage backends. Adding a MongoDB implementation would force an awkward choice:
- Clutter `Dapper/` with non-Dapper files
- Create an ad-hoc `MongoDb/` folder at the same level as `Dapper/` with no established convention
- Mix concerns in `Repositories/` where Dapper and document repos would live side by side

A clean structure would be:

```
Infrastructure/Persistence/
├── Dapper/
│   ├── DapperConnection.cs
│   ├── DapperUnitOfWork.cs
│   ├── Mappings/
│   └── Repositories/
├── MongoDb/
│   ├── MongoUnitOfWork.cs
│   ├── Mappings/
│   └── Repositories/
└── SqlRepository.cs          (generic relational base, or moved into Dapper/)
```

This makes the technology choice explicit at the directory level and provides a clear pattern for adding new persistence backends without guesswork.

### 18. No Pure Abstraction for Persistence

The repository abstraction is **leaky** — tightly coupled to Dapper's relational model:

```csharp
// SqlRepository.cs (tool-lib infrastructure)
public virtual async Task<TEntity?> GetByIdAsync(TId id, ...)
{
    return await mapping.GetByIdAsync(uow.Connection, (int)((dynamic)id!).Value);
}
```

The use of `dynamic` dispatch to extract `.Value` from ID value objects assumes every ID wraps an `int`. The `ISqlEntityMapping<TEntity>` interface assumes SQL (`InsertSql`, `UpdateSql`, `DeleteSql`). The `DapperUnitOfWork` assumes a relational transaction.

**Switching to a document database (MongoDB, RavenDB) or a NoSQL store would require:**
1. Replacing every `ISqlEntityMapping<T>` with a new NoSQL mapping contract
2. Removing the `dynamic` cast that assumes `int` IDs
3. Replacing `DapperUnitOfWork` with a document-store unit of work
4. Rewriting the `TypeHandlers` for Dapper value objects

The `IRepository<TAggregateRoot, TId>` interface itself is technology-agnostic, but everything below it is Dapper-specific. A true persistence-agnostic design would use a pure repository interface with technology-specific implementations that never leak SQL assumptions into the abstraction.

---

## The Good: What Has Been Done Right

### 1. Tool Library Domain Model is Genuinely Strong

The tool-lib aggregate model is the highlight of the codebase:

- **Clear aggregate boundaries with distinct identities:** `Tool` (catalog definition) vs `InventoryItem` (physical copy) vs `Loan` (borrowing record) vs `Reservation` (queue position) vs `MaintenanceRecord` (repair ticket). Each has a distinct lifecycle, identity, and set of invariants.

- **Full encapsulation:** All setters are `private set`. Mutation happens only through named behavioral methods with pre-condition guards. Nothing can call `tool.Status = ItemStatus.Available` from outside — you must call `item.Return(condition)`.

- **`Result<T>` pattern for validation:** Invalid states cannot be constructed. `Tool.Create()` returns `Result<Tool>` — the caller must check `IsFailure` before proceeding. Validation errors are explicit typed objects (`ValidationError`, `DomainError`, `NotFoundError`), not exceptions or magic strings.

- **Domain events raised from within aggregate methods:** When `Tool.Register()` succeeds, it raises `ToolRegisteredEvent`. When `InventoryItem.Return()` completes, it raises `ToolReturnedEvent`. Events are a natural byproduct of domain behavior, not an afterthought in the application layer.

### 2. In-Transaction Event Dispatch (eShop Pattern)

`DomainEventOrchestrator.cs` implements the eShop "Option A" pattern:

```
1. Collect domain events from tracked aggregates
2. DISPATCH events to handlers
3. COMMIT database transaction
4. On success: clear domain events
```

Events are dispatched **before** the commit but **within** the same transaction. If a handler throws, the commit never happens — rollback is atomic. If the commit fails after dispatch, the events remain on the aggregates (not cleared), allowing retry.

This is the correct pattern for maintaining consistency across aggregate boundaries within a single bounded context. The handlers (e.g., `ToolReturnedEventHandler`) can safely mutate other aggregates because their changes are part of the same transaction.

### 3. Clean Architecture Discipline

All .NET microservices follow a strict 4-layer Clean Architecture with correct dependency direction:

```
   ┌──────────────────────────────────────────────────────┐
   │                     WebApi                           │
   │   Controllers, DTOs, Middleware, Program.cs          │
   └──────────────────┬───────────────────────────────────┘
                      │ depends on ▼
   ┌──────────────────▼───────────────────────────────────┐
   │                  Infrastructure                      │
   │   Dapper Repos, SQL Mappings, UoW, TypeHandlers      │
   └──────────────────┬───────────────────────────────────┘
                      │ depends on ▼
   ┌──────────────────▼───────────────────────────────────┐
   │                  Application                         │
   │   Commands, Handlers, Event Dispatch, DI Config      │
   └──────────────────┬───────────────────────────────────┘
                      │ depends on ▼
   ┌──────────────────▼───────────────────────────────────┐
   │                   Domain                             │
   │   Aggregates, Value Objects, Interfaces              │
   └──────────────────────────────────────────────────────┘
               ▲ depends on (all layers)
   ┌──────────────────────────────────────────────────────┐
   │                  TSQR.Common                         │
   │   Result<T>, Error types, IInteractor<,>             │
   └──────────────────────────────────────────────────────┘
```

No layer depends on a layer above it. The Domain layer has zero infrastructure dependencies. The Application layer only depends on Domain and `Microsoft.Extensions.DependencyInjection.Abstractions`.

### 4. Generic Persistence Infrastructure

`SqlRepository<TEntity, TId>` paired with `ISqlEntityMapping<TEntity>` provides a clean, type-safe generic persistence layer:

```csharp
// One mapping class per aggregate defines all SQL
public sealed class ToolMapping : ISqlEntityMapping<Tool>
{
    public string TableName => "Tools";
    public string InsertSql => "INSERT INTO Tools (...) VALUES (...) RETURNING Id";
    public string UpdateSql => "UPDATE Tools SET ... WHERE Id = @Id";
    public string DeleteSql => "DELETE FROM Tools WHERE Id = @Id";
    public async Task<Tool?> GetByIdAsync(ISqlConnection db, int id) { ... }
    public object ToInsertParameters(Tool entity) => new ToolInsertDto(...);
    public object ToUpdateParameters(Tool entity) => new ToolUpdateDto(...);
}
```

The `DapperUnitOfWork` with transient-failure retry (3 attempts, 100ms backoff, exponential delay) handles PostgreSQL transient errors gracefully — a production-grade concern often overlooked.

Value object type handlers (`TypeHandlers.cs`) allow Dapper to map `ToolId`, `MemberId`, etc. to/from database integers automatically.

### 5. CQRS Done Right

Reads and writes are properly separated:

- **Writes** (commands): `IInteractor<TCommand, TResult>` → domain aggregate mutation → `Repository.AddAsync/Update` → `UnitOfWork.SaveChangesAsync`
- **Reads** (queries): Dedicated query interfaces (`IDashboardQueries`, `IGeographyQueries`, `ISupportQueries`) → raw Dapper SQL → DTO records

No aggregate is ever loaded for a query. No lazy loading occurs. No ORM overhead. The query side bypasses the domain model entirely, which is the correct CQRS approach.

### 6. Ubiquitous Language is Consistent

The core terms are used consistently across all bounded contexts:

| Term | Meaning | Where Used |
|---|---|---|
| `Tool` | Catalog definition — what a tool *is* | Tool Library |
| `InventoryItem` | Physical copy — which specific instance | Tool Library |
| `Member` | Community participant (int ID) | Tool Library, Soup Kitchen |
| `User` | Platform login account (GUID ID) | Autheo, Identity |
| `Community` | Neighbourhood-level group | Communities, scopes all others |
| `Loan` | Borrowing record | Tool Library |
| `Reservation` | Queue position for future borrowing | Tool Library |
| `Event` | A soup kitchen service event | Soup Kitchen |

The distinction between `User` (authentication identity) and `Member` (community participant) is particularly well-handled — they are separate concepts with separate identities (GUID vs int), linked by the `mbr` JWT claim.

### 7. Autheo Security Design

The authentication context has several security-conscious design decisions:

- **PBKDF2** with 100,000 iterations, HMACSHA256, 16-byte salt, 32-byte derived key — well above minimum standards
- **`CryptographicOperations.FixedTimeEquals`** for password verification — timing-attack resistant
- **Refresh token family revocation** — if a revoked token is presented, the entire descendant chain is walked via `ReplacedByTokenHash` and revoked with `RevokedReasonIsReuse = true`. This detects and contains token theft.
- **User enumeration prevention** — `ForgotPasswordHandler` always returns `200 OK`, even if the email doesn't exist
- **Short-lived access tokens** (15 minutes) with 7-day refresh tokens

### 8. Scoped Multi-Tenancy via CommunityId

Every domain entity across Tool Library, Soup Kitchen, and Communities carries a `CommunityId`:

```csharp
public class Tool : Entity<ToolId>, IAggregateRoot {
    public int CommunityId { get; private set; }
    // ...
}
```

This creates a clean multi-tenant boundary within a single database deployment. No tool, loan, member, event, or reservation exists outside of a community. Queries are naturally scoped (or should be — this is enforced at the application level rather than by the domain model).

---

## Summary: Verdict

The codebase reflects a **team that understands DDD principles but applied them unevenly**.

**Tool Library** is the standout — it demonstrates proper aggregate design, encapsulation, domain events, and the `Result<T>` pattern. Most of the issues there are wiring gaps (events not raised, handlers that are stubs, policy not enforced) rather than fundamental modeling problems.

**Autheo** has strong security patterns but inconsistent error handling (exceptions in domain, `Result<T>` in application).

**Communities, Soup Kitchen, Identity, and Support** are functionally complete but domain-model-light. They implement the Clean Architecture structure and use the same infrastructure stack, but their domain layers lack the rigor of tool-lib — no validation, no domain events, anemic models, public constructors.

The **18 issues identified** fall into three categories:
- **Architectural (8):** Aggregate boundary violations, missing event wiring, broken transactions, non-switchable persistence, missing NoSQL abstraction, no folder separation for persistence backends
- **Modeling (6):** Anemic domains, orphaned entities, duplicated value objects, naive slugs, wrong aggregate root markers
- **Code quality (4):** Debug method leaked, inconsistent error handling, contradictory state transitions, dead domain services

The foundation is solid but incomplete. The Tool Library bounded context shows what the architecture is capable of; the other contexts need to be brought up to the same standard.
