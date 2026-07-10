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
   - [19. Schema Design: SELECT *, Missing Constraints, No Indexes](#19-schema-design-select--missing-constraints-no-indexes)
   - [20. Identity Schema Duplication and Type Mismatch](#20-identity-schema-duplication-and-type-mismatch)
   - [21. No Pagination on Any List Query](#21-no-pagination-on-any-list-query)
   - [22. String Concatenation for Dynamic WHERE Clauses](#22-string-concatenation-for-dynamic-where-clauses)
   - [23. No Optimistic Concurrency Control](#23-no-optimistic-concurrency-control)
   - [24. Ignored Affected Row Counts from Writes](#24-ignored-affected-row-counts-from-writes)
   - [25. Hard Deletes Without Audit Trail](#25-hard-deletes-without-audit-trail)
   - [26. Separate Connections Bypassing the Scoped Unit of Work](#26-separate-connections-bypassing-the-scoped-unit-of-work)
   - [27. Unnecessary TCP Connections for Read Queries](#27-unnecessary-tcp-connections-for-read-queries)
   - [28. Blocking Async Call in Autheo's UserRepository](#28-blocking-async-call-in-autheos-userrepository)
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

### 19. Schema Design: SELECT \*, Missing Constraints, No Indexes

Three schema-level issues appear in every microservice's database.

**SELECT \* in every GetById mapping.** Every `GetByIdAsync` in Communities, Soup Kitchen, and several in Tool Library uses `SELECT * FROM {Table} WHERE Id = @Id`:

```sql
-- CommunityMapping.cs, CountryMapping.cs, EventMapping.cs, GuestMapping.cs, etc.
SELECT * FROM Communities WHERE Id = @Id
SELECT * FROM Events WHERE Id = @Id
SELECT * FROM Guests WHERE Id = @Id
```

This is fragile — adding, removing, or reordering columns can silently break queries at runtime. It's also wasteful, fetching every column (including large TEXT/BLOB fields) when only a subset may be needed. The Tool Library mappings are the exception — they explicitly list columns.

**No CHECK constraints on any INTEGER-encoded enum column.** Status, Type, Category, State, and Role are stored as plain `INTEGER` with no `CHECK` constraint. A value of `99` can be inserted into `ItemStatus`, `EventStatus`, `LoanStatus`, `TicketCategory`, or `ServiceState` and the database will accept it silently. Once bad data lands, application code that does `Enum.Parse<ItemStatus>(99)` or a switch without a default arm will throw at runtime.

Affected columns across all services:

| Column | Table(s) | Legal Values (intended) |
|---|---|---|
| `Status` | Members, InventoryItems, Reservations, MaintenanceRecords, Loans, Events, Guests, Volunteers, Communities, Tickets, Incidents, Apps | 1-5 range depending on entity |
| `ToolType` | Tools | 0-6 |
| `AmortizationRate` | Tools | 0-3 |
| `Condition` | InventoryItems | 0-5 |
| `Category` | Tickets, Meals | 1-6 |
| `State` | ServiceStatuses | 1-3 |
| `MembershipType` | Members | 0-4 |
| `ScarcityLevel` | ToolScarcityByLocation | 0-4 |

**No indexes on critical query paths.** Across all 6 microservices, only 2 indexes are explicitly created (both in tsqr-identity): `IX_user_roles_RoleId` and `IX_pending_email_changes_UserId`/`TokenHash`. Every other table has only the implicit primary key index on `Id`. Columns queried regularly with no index:

- `Members.Email` — used for login lookup in Tool Library
- `Members.CommunityId` — scoping queries
- `InventoryItems.Status` — dashboard aggregations
- `InventoryItems.CurrentHolderId` — member's active loans
- `Reservations.ItemId` — reservation queue lookups
- `Loans.MemberId` — member's borrowing history
- `Loans.Status` — active vs. returned filtering
- `Events.CommunityId` — community scoping
- `Tickets.SubmitterEmail` / `SubmitterMemberId` — looking up user's tickets
- All FK columns (ManufacturerId, ToolId, CityId, NeighbourhoodId, EventId, etc.)

At production scale (thousands of members, tens of thousands of inventory items), every one of these queries becomes a sequential scan.

**Optimization suggestions:**
```sql
-- Replace CHECK-less INTEGER enums
ALTER TABLE InventoryItems ADD CONSTRAINT CK_InventoryItems_Status
    CHECK (Status BETWEEN 1 AND 5);

-- Add critical indexes
CREATE INDEX IX_Members_Email ON Members(Email);
CREATE INDEX IX_Members_CommunityId ON Members(CommunityId);
CREATE INDEX IX_InventoryItems_Status ON InventoryItems(Status);
CREATE INDEX IX_Reservations_ItemId ON Reservations(ItemId);
CREATE INDEX IX_Loans_MemberId ON Loans(MemberId);
CREATE INDEX IX_Loans_Status ON Loans(Status);
CREATE INDEX IX_Events_CommunityId ON Events(CommunityId);
```

### 20. Identity Schema Duplication and Type Mismatch

`tsqr-identity` has **two mutually exclusive database schemas** that define different tables for the same concepts:

| Source | Tables Created | Used By Code? |
|---|---|---|
| `src/.../Api/docker/init.sql` | `AspNetUsers`, `AspNetRoles`, `AspNetUserRoles`, `AspNetUserClaims`, `RefreshTokens` | **No** — mirrors ASP.NET Identity tables |
| `src/.../Infrastructure/Migrations/001_InitialSchema.sql` | Same as above | **No** — identical copy |
| `../../tsqr-deploy/infra/05-identity.sql` | `user_profiles`, `roles`, `user_roles`, `pending_email_changes` | **Yes** — all 3 repositories query these tables |

The code's `ProfileRepository`, `UserAdminRepository`, and `RoleRepository` all query `user_profiles` and `roles` tables. The `init.sql` and `001_InitialSchema.sql` are dead code — they define `AspNetUsers` which is never queried.

**Type mismatch across microservices.** `tsqr-identity` stores `user_profiles.Id` as `VARCHAR(128)`, while `tsqr-autheo` stores `Users.Id` as `UUID`. Both are supposed to hold the same GUID value (e.g., `11111111-1111-1111-1111-111111111111`). The type mismatch prevents:
- A `REFERENCES` foreign key if these tables were ever in the same database
- Direct JOIN queries across the two services
- Database-level type safety — a non-UUID string can be inserted into identity's `Id` column

**Optimization suggestions:**
- Remove the dead `init.sql` and `001_InitialSchema.sql` files
- Change `user_profiles.Id` from `VARCHAR(128)` to `UUID`
- Add a `REFERENCES` direction (identity's `user_profiles` references autheo's `Users`) if they ever share a database, or accept the application-level join if they remain separate

### 21. No Pagination on Any List Query

Every "get all" or "list" query across all microservices returns an **unbounded result set**:

| File | Query | Risk |
|---|---|---|
| `ToolRepository.cs:7` | `SELECT ... FROM Tools ORDER BY t.Model` | With 10K+ tools, loads everything into memory |
| `GeographyQueries.cs` | `SELECT Id FROM Countries ...` | Low risk (few countries) but inconsistent |
| `DashboardQueries.cs` (support) | `SELECT COUNT(*) ...` | Safe (aggregates are fine) |
| `TicketsRepository.cs` | `SELECT ... FROM Tickets` | Support tickets can grow without bound |
| `SupportQueries.cs:ListTicketsAsync` | Uses `LIMIT 50` | **The only query with a limit** — all others are unbounded |

The Tool Library's `ToolRepository.GetAllAsync()` is the most dangerous — it loads every tool with a JOIN to Manufacturers, then for each tool the `ToolMapping.GetByIdAsync` also loads scarcity rows. This is an N+1 pattern embedded in a "get all" query.

**Optimization suggestions:**
```csharp
// Add pagination to all list queries
public async Task<List<Tool>> GetAllAsync(int page = 1, int pageSize = 50, CancellationToken ct = default)
{
    var offset = (page - 1) * pageSize;
    var rows = await Database.QueryAsync<ToolRow>(
        @"SELECT t.Id, t.Model, t.Description, t.ToolType, t.AmortizationRate, t.Metadata,
                 m.Id AS ManufacturerId, m.Name AS ManufacturerName
          FROM Tools t
          INNER JOIN Manufacturers m ON m.Id = t.ManufacturerId
          ORDER BY t.Model
          LIMIT @Limit OFFSET @Offset",
        new { Limit = pageSize, Offset = offset }
    );
    // ...
}
```

### 22. String Concatenation for Dynamic WHERE Clauses

The Soup Kitchen WebApi query handlers build dynamic `WHERE` clauses using string concatenation:

```csharp
// GetMealsQuery.cs
var conditions = new List<string>();
if (request.EventId.HasValue) conditions.Add("m.EventId = @EventId");
if (request.Category.HasValue) conditions.Add("m.Category = @Category");
var where = conditions.Count > 0 ? "WHERE " + string.Join(" AND ", conditions) : "";

var countSql = $"SELECT COUNT(*) FROM Meals m {where}";
var sql = $@"
    SELECT m.Id, ...
    FROM Meals m
    {where}
    ORDER BY m.Name
    LIMIT @PageSize OFFSET @Offset";
```

The same pattern appears in `GetVolunteersQuery.cs`, `GetEventsQuery.cs`, `GetGuestsQuery.cs`, and `GetDonationsQuery.cs`. While the actual values are parameterized (using `@EventId`, `@Category` etc.), the WHERE clause itself is string concatenated. If any future contributor adds a condition using user input in the string part rather than the parameter, this becomes a SQL injection vector. More immediately, it's fragile: a missing space before `WHERE` or between conditions will produce invalid SQL.

**Optimization suggestion:** Build the WHERE clause once and reuse it for both count and data queries, using a consistent builder pattern or a dedicated query object:

```csharp
private static (string where, object parameters) BuildMealFilter(int? eventId, int? category)
{
    var conditions = new List<string>();
    var parameters = new Dictionary<string, object>();

    if (eventId.HasValue) { conditions.Add("m.EventId = @EventId"); parameters["EventId"] = eventId.Value; }
    if (category.HasValue) { conditions.Add("m.Category = @Category"); parameters["Category"] = category.Value; }

    var where = conditions.Count > 0 ? "WHERE " + string.Join(" AND ", conditions) : "";
    return (where, parameters);
}
```

Even better, use Dapper's `DynamicParameters` or a proper query specification pattern.

### 23. No Optimistic Concurrency Control

No table has a row version column (`xmin` in PostgreSQL, or an explicit `RowVersion`/`ConcurrencyToken`). No `UPDATE` or `DELETE` statement uses a `WHERE` clause that verifies the row hasn't changed since it was read:

```sql
-- Current pattern (in every UPDATE):
UPDATE Tools SET Model = @Model, Description = @Description ... WHERE Id = @Id

-- What would prevent lost updates:
UPDATE Tools SET Model = @Model, Description = @Description ...
WHERE Id = @Id AND ToolType = @OriginalToolType  -- optimistic lock
```

Without this, two concurrent requests can read the same aggregate, both modify it, and the second write silently overwrites the first. This is especially dangerous for:

- `InventoryItem.Loan()` / `InventoryItem.Return()` — two concurrent loans on the same item could both succeed (the second overwrites the first)
- `Member.Suspend()` / `Member.Reinstate()` — a concurrent suspend and reinstate could lose one operation
- `Reservation.Cancel()` / `Reservation.ConfirmPickup()` — race conditions on queue management

**Optimization suggestion:** Use PostgreSQL's built-in `xmin` system column as an implicit row version, or add an explicit `RowVersion` column:

```csharp
// Option A: Use xmin (PostgreSQL-specific, zero overhead)
var row = await db.QuerySingleOrDefaultAsync<ToolRow>(
    "SELECT *, xmin AS RowVersion FROM Tools WHERE Id = @Id", new { Id = id });

// Then UPDATE ... WHERE Id = @Id AND xmin = @RowVersion

// Option B: Explicit version column
// ALTER TABLE InventoryItems ADD COLUMN RowVersion INTEGER NOT NULL DEFAULT 0;
// UPDATE InventoryItems SET Status = @NewStatus, RowVersion = RowVersion + 1
// WHERE Id = @Id AND RowVersion = @ExpectedVersion
```

### 24. Ignored Affected Row Counts from Writes

No `Update` or `Delete` call checks the number of rows affected:

```csharp
// SqlRepository.cs (all 4 microservices)
public virtual void Update(TEntity entity)
{
    db.Execute(mapping.UpdateSql, mapping.ToUpdateParameters(entity));  // return value ignored
}

public virtual void Delete(TEntity entity)
{
    db.Execute(mapping.DeleteSql, new { Id = entity.Id });  // return value ignored
}
```

If the `UPDATE` or `DELETE` affects zero rows (because the row was already deleted by another request, or the `WHERE` clause didn't match), the application has no way to detect this. `DapperUnitOfWork.SaveChangesAsync()` will still succeed because it only commits the transaction — it doesn't verify that any mutation actually occurred.

**Optimization suggestions:**
```csharp
public virtual void Update(TEntity entity)
{
    var affected = db.Execute(mapping.UpdateSql, mapping.ToUpdateParameters(entity));
    if (affected == 0)
        throw new ConcurrencyException($"Expected to update 1 row but updated 0 for {typeof(TEntity).Name} Id={entity.Id}");
}
```

### 25. Hard Deletes Without Audit Trail

All 6 microservices use hard `DELETE` with no soft-delete, audit trail, or tombstone record:

```sql
DELETE FROM Tools WHERE Id = @Id
DELETE FROM Members WHERE Id = @Id
DELETE FROM Loans WHERE Id = @Id
```

Once a record is deleted, there is no way to:
- Recover accidentally deleted data
- Audit who deleted what and when
- Maintain referential integrity for historical records (e.g., a deleted member referenced by past loans)
- Query historical data for analytics

This is especially problematic for:
- **Members** — deleting a member breaks FK references from their past loans, inventory items they own, and maintenance records they reported
- **Loans** — financial history is lost
- **Tickets** — support history is destroyed

**Optimization suggestion:** Add `DeletedAt TIMESTAMP NULL` and `DeletedBy VARCHAR(200) NULL` columns to every table, and replace `DELETE` with `UPDATE ... SET DeletedAt = now()`:

```sql
ALTER TABLE Members ADD COLUMN DeletedAt TIMESTAMP NULL;
ALTER TABLE Members ADD COLUMN DeletedBy VARCHAR(200) NULL;

-- Replace:
--   DELETE FROM Members WHERE Id = @Id
-- With:
--   UPDATE Members SET DeletedAt = now(), DeletedBy = @DeletedBy WHERE Id = @Id
```

### 26. Separate Connections Bypassing the Scoped Unit of Work

Several query implementations create their **own independent `NpgsqlConnection`** rather than using the scoped `ISqlUnitOfWork` from DI:

```csharp
// GeographyQueries.cs (tsqr-communities)
public sealed class GeographyQueries(string connectionString) : IGeographyQueries
{
    public async Task<int?> FindCountryIdByNameAsync(string name, CancellationToken ct = default)
    {
        using var conn = new NpgsqlConnection(connectionString);  // NEW CONNECTION
        await conn.OpenAsync(ct);
        return await conn.ExecuteScalarAsync<int?>(...);
    }
}

// DashboardQueries.cs (tsqr-communities)
public sealed class DashboardQueries(string connectionString) : IDashboardQueries
{
    public async Task<DashboardStatsData> GetStatsAsync(CancellationToken ct = default)
    {
        using var conn = new NpgsqlConnection(connectionString);  // NEW CONNECTION
        await conn.OpenAsync(ct);
        // ...
    }
}
```

The same pattern appears in:
- `tsqr-communities`: `GeographyQueries` (3 methods), `DashboardQueries`
- `tsqr-soup-kitchen`: `DashboardQueries`, and all 6 WebApi query handlers (`GetEventsHandler`, `GetMealsHandler`, `GetVolunteersHandler`, `GetGuestsHandler`, `GetDonationsHandler`)
- `tsqr-autheo`: All repositories use the scoped `ISqlUnitOfWork` correctly

These bypass the ambient transaction. If a command handler runs a query through `GeographyQueries` mid-transaction, that query sees a different snapshot of the database (whatever is already committed, not the in-progress transaction). This breaks read-your-writes consistency.

**Optimization suggestion:** Inject `ISqlUnitOfWork` instead of `connectionString`:

```csharp
public sealed class GeographyQueries(ISqlUnitOfWork uow) : IGeographyQueries
{
    public async Task<int?> FindCountryIdByNameAsync(string name, CancellationToken ct = default)
    {
        return await uow.Connection.ExecuteScalarAsync<int?>(
            "SELECT Id FROM Countries WHERE LOWER(Name) = LOWER(@name) LIMIT 1",
            new { name });
    }
}
```

### 27. Unnecessary TCP Connections for Read Queries

Even when transactions aren't needed (read-only queries), opening a new `NpgsqlConnection` per request is wasteful. Each `new NpgsqlConnection(connectionString)` + `OpenAsync()` involves:

1. TCP 3-way handshake
2. SSL/TLS negotiation (if enabled)
3. PostgreSQL authentication handshake
4. Backend process spawn on the server

For a dashboard page that calls 5 separate `DashboardQueries`-style methods, this means **5 TCP connections**, each with the full handshake overhead. Npgsql's connection pooling mitigates this somewhat, but the connections still compete for pool slots and the multi-step setup is far slower than reusing an already-opened connection from the scoped `DapperUnitOfWork`.

The Soup Kitchen's WebApi query handlers are the worst offenders — each request to list events, meals, volunteers, etc. creates its own connection, even though `Program.cs` registers `ISqlUnitOfWork` as scoped (one per HTTP request) which is already available via DI.

**Optimization suggestion:** Inject the scoped `ISqlUnitOfWork` into all query handlers instead of `connectionString`. This reuses the single connection-per-request that already exists in DI registration.

### 28. Blocking Async Call in Autheo's UserRepository

`UserRepository.Update()` uses `.GetAwaiter().GetResult()` — a blocking call on an async method:

```csharp
// UserRepository.cs:44-66
public void Update(User user)
{
    uow.Connection.ExecuteAsync(
        @"UPDATE Users ... WHERE Id = @Id",
        new { ... }).GetAwaiter().GetResult();  // BLOCKS the current thread
}
```

This blocks the ASP.NET Core request thread while waiting for the database. Under load, this causes **thread pool starvation** — all available threads are blocked waiting for I/O, new requests queue up, and throughput collapses.

The `IRepository.Update()` interface is synchronous (`void Update`), so the repository is forced into this pattern. The other bounded contexts avoid this by having `Update` call `db.Execute()` (the synchronous Dapper method) instead of `ExecuteAsync()`.

**Optimization suggestions:**
```csharp
// Option A: Make IRepository.Update async (preferred, but affects all implementations)
Task UpdateAsync(TEntity entity, CancellationToken ct = default);

// Option B: Use synchronous Dapper method (current pattern in other contexts)
public void Update(User user)
{
    uow.Connection.Execute(  // synchronous
        @"UPDATE Users SET ... WHERE Id = @Id", new { ... });
}
```

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

The **28 issues identified** fall into four categories:
- **Architectural (8):** Aggregate boundary violations, missing event wiring, broken transactions, non-switchable persistence, missing NoSQL abstraction, no folder separation for persistence backends
- **Modeling (6):** Anemic domains, orphaned entities, duplicated value objects, naive slugs, wrong aggregate root markers
- **Code quality (4):** Debug method leaked, inconsistent error handling, contradictory state transitions, dead domain services
- **Persistence & SQL (10):** Schema design gaps (no CHECK constraints, no indexes), identity schema duplication, no pagination, SQL injection risk via concatenation, no concurrency control, ignored row counts, hard deletes, connections bypassing UoW, unnecessary TCP connections, blocking async calls

The foundation is solid but incomplete. The Tool Library bounded context shows what the architecture is capable of; the other contexts need to be brought up to the same standard.
