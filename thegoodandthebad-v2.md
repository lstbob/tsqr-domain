# TSQR (TownsSquare) — DDD Design Audit, Round 2: New Issues

This audit is a **second-round** analysis commissioned after issues `thegoodandthebad.md` #1–#28 (plus four infra items) were filed and the first three tool-lib refactors (#33, #34, #35) were merged to `main`. It catalogues **NEW issues** verified by reading the actual code on `main`, including three regressions introduced by the recent refactors. Issues already covered in v1 are NOT repeated.

> **Companion:** [`thegoodandthebad.md`](./thegoodandthebad.md) (round 1).
> **Method:** Each bounded context inspected in parallel by an `explore` agent; cross-cutting concerns (tsqr-common, tsqr-deploy, tsqr-gateway, integration) inspected separately. File paths and line numbers reference the state of `main` on the audit date.

---

## Table of Contents

### Tool Library (regressions + new gaps opened by #33/#34/#35)

1. [ReservationQueueService Direction Bug + ReservationCreatedEvent Breakage](#1-reservationqueueservice-direction-bug--reservationcreatedevent-breakage)
2. [Authorization: No Ownership Checks on Reservation/Loan/Repair Commands](#2-authorization-no-ownership-checks-on-reservationloanrepair-commands)
3. [Six Orphaned Domain Event Handlers](#3-six-orphaned-domain-event-handlers)
4. [RenewLoanCommand + Endpoint Deferred](#4-renewloancommand--endpoint-deferred)
5. [MarkToolForRepairCommandHandler Still Violates One-Aggregate-Per-Tx](#5-marktoolforrepaircommandhandler-still-violates-one-aggregate-per-tx)
6. [LoanToolCommand Lost Loanability Pre-check + Member.IsEligibleToBorrow Ignores Fines](#6-loantoolcommand-lost-loanability-pre-check--memberiseligibletoborrow-ignores-fines)

### Communities

7. [Authorization Wide Open; QuickRegister Is Anonymous](#7-authorization-wide-open-quickregister-is-anonymous)
8. [State-Transition Guards + UNIQUE Constraints + Suspend/Archive Commands](#8-state-transition-guards--unique-constraints--suspendarchive-commands)
9. [Read Query Layer Is in the Wrong Place (WebApi + Domain)](#9-read-query-layer-is-in-the-wrong-place-webapi--domain)
10. [Slug Immutability + VO-Typed FKs + CountryCode Value Object](#10-slug-immutability--vo-typed-fks--countrycode-value-object)

### Soup Kitchen

11. [Multi-Tenancy Declared but Not Enforced; Reads Expose Cross-Tenant PII](#11-multi-tenancy-declared-but-not-enforced-reads-expose-cross-tenant-pii)
12. [Cross-Aggregate Invariants: Event Existence/Capacity Checks Missing in Child Commands](#12-cross-aggregate-invariants-event-existencecapacity-checks-missing-in-child-commands)
13. [Unreachable Behavior: State Machines, Donation Aggregate, and Command Handlers](#13-unreachable-behavior-state-machines-donation-aggregate-and-command-handlers)
14. [Unique Constraints + Domain Events + Faithful UnitOfWork](#14-unique-constraints--domain-events--faithful-unitofwork)

### Identity

15. [Email-Change Feature Is Non-Functional + Token Returned in HTTP Body](#15-email-change-feature-is-non-functional--token-returned-in-http-body)
16. [IAggregateRoot Defined but Never Implemented; No Entity Base + No Value Objects](#16-iaggregateroot-defined-but-never-implemented-no-entity-base--no-value-objects)
17. [LockUserCommand Has No Invariant Validation + No Audit of Who Locked](#17-lockusercommand-has-no-invariant-validation--no-audit-of-who-locked)
18. [CancellationToken Silently Dropped + Upsert Race on UpdateProfile](#18-cancellation-silently-dropped--upsert-race-on-updateprofile)
19. [Role Has No Permission Model + String-Role Authorization](#19-role-has-no-permission-model--string-role-authorization)

### Autheo

20. [/register Self-Admin + Broken ForgotPassword/ResetPassword + RevokedReasonIsReuse Not Persisted](#20-register-self-admin--broken-forgotpasswordresetpassword--revokedreasonisreuse-not-persisted)
21. [Concurrent Refresh Token Rotation Has No Optimistic Lock + Revoke Overwrites Audit](#21-concurrent-refresh-token-rotation-has-no-optimistic-lock--revoke-overwrites-audit)
22. [No Rate Limiting + Single Static JWT Signing Key + Single Audience Across 7 Services](#22-no-rate-limiting--single-static-jwt-signing-key--single-audience-across-7-services)

### Support

23. [Anonymous Ticket Submission + No Multi-Tenancy + Critical Field-Mismatch Bug](#23-anonymous-ticket-submission--no-multi-tenancy--critical-field-mismatch-bug)
24. [Ticket State Machine Unenforced + Dead Incident Aggregate + No Comments/Attachments + Hardcoded LIMIT 50](#24-ticket-state-machine-unenforced--dead-incident-aggregate--no-commentsattachments--hardcoded-limit-50)

### Cross-Cutting (Shared Kernel, Deployment, Integration)

25. [Shared Kernel (tsqr-common) Is Anemic; DDD Base Classes Copy-Pasted in 4 Contexts](#25-shared-kernel-tsqr-common-is-anemic-ddd-base-classes-copy-pasted-in-4-contexts)
26. [Bespoke Error/Error MW/Hosting Defaults in Every Service — Need `TSQR.Common.WebApi`](#26-bespoke-errorerrormwhosting-defaults-in-every-service--need-tsqrcommonwebapi)
27. [Deploy SQL Drifted from Local init.sql — `Loans.LateFeePerDay`/`RenewalCount`/`Policies` Table Missing](#27-deploy-sql-drifted-from-local-initsql--loanslatefeeperdayrenewalcountpolicies-table-missing)
28. [No DB Migration Framework — `ALTER TABLE` Without `IF NOT EXISTS`, No Version Table](#28-no-db-migration-framework--alter-table-without-if-not-exists-no-version-table)
29. [`mbr` JWT Claim Undocumented, Unconsumed, Type-Mismatched Across Contexts](#29-mbr-jwt-claim-undocumented-unconsumed-type-mismatched-across-contexts)

---

## Tool Library — Regressions and New Gaps Opened by #33/#34/#35

### 1. ReservationQueueService Direction Bug + ReservationCreatedEvent Breakage

> **GitHub Issues:** [tsqr-tool-lib#42](https://github.com/lstbob/tsqr-tool-lib/issues/42) (P0, 5 SP)

**Severity:** Critical — **regression from #34** that broke the reservation queue entirely.

**Evidence:**

`Reservation.cs:172-175`:
```csharp
public void MoveDownInQueue()
{
    QueuePosition++;          // increments — moves AWAY from the front
}
```

`ReservationQueueService.cs:19-30`:
```csharp
public IReadOnlyCollection<Reservation> ShiftQueueAfterCancellation(
    IReadOnlyCollection<Reservation> activeReservations,
    Reservation cancelledReservation)
{
    foreach (var reservation in activeReservations
        .Where(r => r.QueuePosition > cancelledReservation.QueuePosition))
    {
        reservation.MoveDownInQueue();   // called on everyone BEHIND the cancelled slot
    }
    return activeReservations;
}
```

The handler commentary (`ReservationCancelledEventHandler.cs:9-11`) promises "closes gaps like 1, 3, 4 → 1, 2, 3" — but the code increments positions. Cancelling slot #2 in `[1,2,3,4]` produces `[1,(cancelled),4,5]` instead of `[1,(cancelled),2,3]`. Each subsequent cancellation compounds the drift. `MoveDownInQueue` is also a misnomer: when something ahead of you cancels, you move **up** the queue (decrement). **Zero unit tests** on `ReservationQueueService` — the regression sailed through.

**Compounding bug in `ReservationCreatedDomainEventHandler.cs:12-16`:**
```csharp
var reserveResult = item.Reserve();
if (reserveResult.IsFailure)
    throw new InvalidOperationException($"InventoryItem.Reserve failed: {reserveResult.Error}");
```

`InventoryItem.Reserve()` rejects any item not in `Available` status (`InventoryItem.cs:166-173`). The whole point of a reservation queue is to let members queue up for items currently **loaned to someone else**. After #34, every reservation creation tries to set `Available → Reserved` and throws on loaned/reserved items. Items with one `Pending` reservation can never have a second reservation queued (`InvalidOperationException` → 500; transaction rolls back). The `QueuePosition` feature is functionally dead.

**Affected Files:**
- `Reservation.cs:172-175` (rename + decrement)
- `ReservationQueueService.cs:19-30` (logic inverse)
- `ReservationCreatedDomainEventHandler.cs:7-17` (only the first reservation on an item should call `item.Reserve()`; later ones simply join the queue)
- New unit tests for `ReservationQueueService` covering 1/2/3-slot input cases

**Suggested fix:** Rename to `MoveUpInQueue()` and `QueuePosition--`. In `ReservationCreatedDomainEventHandler`: only invoke `item.Reserve()` when `allReservations.Any(r => r.Status is Pending or Confirmed or Active)` is false (i.e., no queue exists yet). Add unit tests for both behaviors.

---

### 2. Authorization: No Ownership Checks on Reservation/Loan/Repair Commands

> **GitHub Issue:** [tsqr-tool-lib#43](https://github.com/lstbob/tsqr-tool-lib/issues/43) (P1, 5 SP)

**Severity:** High.

**Evidence:** `ReservationsCommandController.cs` (whole controller), `CancelReservationCommandHandler`:

```csharp
[HttpPatch("{id}/cancel")]
[Authorize(Roles = "Admin,LocationCoordinator,Repairman,Regular")]   // Regular included
public async Task<ActionResult> Cancel(int id, ...) { var command = new CancelReservationCommand(new ReservationId(id)); ... }

// Handler accepts only ReservationId — never the caller's MemberId claim
public async Task<Result> ExecuteAsync(CancelReservationCommand command, ...) {
    var reservation = await reservationRepository.GetByIdAsync(command.ReservationId, ...);
    var cancelResult = reservation.Cancel();    // any reservation, any caller
}
```

`Cancel`, `Activate`, `ConfirmPickup`, `Complete` only take a `ReservationId`; the controller permits `Regular`. Any member can enumerate integer IDs and cancel/activate anyone else's reservation. `LoanToolCommand` similarly lets the caller pass any `MemberId` in the body — the handler does not confirm the caller == that member. There is no `[Authorize]` policy asserting "caller's `MemberId` claim owns the mutated entity."

**Affected Files:** `ReservationsCommandController.cs`, `ToolsCommandController.cs`, `MembersCommandController.cs`, all four reservation command handlers + `LoanToolCommandHandler`.

**Suggested fix:** Pass the caller's `MemberId` / role claim into the command; verify ownership (or admin/coordinator role) inside each handler before mutating. Restrict `Cancel` and `ConfirmPickup` to "self or Admin/LocationCoordinator".

---

### 3. Six Orphaned Domain Event Handlers

> **GitHub Issue:** [tsqr-tool-lib#44](https://github.com/lstbob/tsqr-tool-lib/issues/44) (P1, 3 SP)

**Severity:** High.

**Evidence:** `DependencyInjection.cs:65-73` registers handlers for only `ReservationCancelledEvent`, `ToolReturnedEvent`, `InventoryItemRequiredEvent`, `LoanCreatedDomainEvent`, `ReservationCreatedDomainEvent`, `ToolMarkedForRepairEvent`, `NextInLineNotificationEvent`, `PickupReminderEvent`, `ReturnReminderEvent`. Aggregates raise these but **no handler is registered**:

| Event | Raised by | DI handler? | Intended side effect |
|---|---|---|---|
| `ToolRegisteredEvent` | `Tool.Register()` | none | Downstream notification / catalog indexing |
| `ItemLoanedDomainEvent` | `InventoryItem.Loan()` | none | Member ledger / eligibility refresh |
| `LoanOverdueDomainEvent` | `Loan.EndLoan()` | none | Overdue escalation / member suspension |
| `ReservationConfirmedEvent` | `Reservation.ConfirmPickup()` | none | Pickup-window tracking |
| `ToolRepairedEvent` | `MaintenanceRecord.Complete()` | none | Release item from `UnderMaintenance` |
| `MemberVerifiedEvent` | `Member.Verify()` | none | Authorization refresh / audit |

`DomainEventDispatcher` uses `serviceProvider.GetServices(handlerType)` (`DomainEventDispatcher.cs:15`), which silently returns empty when no handler is registered — events are dispatched to nobody.

**Why it's a bug:** v1 issue #2 called out "events defined but never raised." #34/#35 made some events raise, but introduced several new orphaned ones. The original "Defined but Dead" pattern reborn.

**Affected Files:** `DependencyInjection.cs:65-73`, plus new handler files in `Application/Events/`.

**Suggested fix:** Either delete the orphaned events (if no current consumer needs them) or add concrete handlers. Either way, add a unit test asserting every raised event has at least one registered handler.

---

### 4. RenewLoanCommand + Endpoint Deferred

> **GitHub Issue:** [tsqr-tool-lib#45](https://github.com/lstbob/tsqr-tool-lib/issues/45) (P1, 3 SP)

**Severity:** High — half-wiring closes only half of audit §6 ("No loan renewal").

**Evidence:**
- `Loan.cs:196-215` — `Loan.Renew(maxLoanDurationDays, maxRenewalCount)` exists and has 3 passing tests.
- `DependencyInjection.cs:45-50` — only `LoanToolCommand` and `MarkLoanAsNotReturnedCommand` are registered.
- `LoansCommandController.cs` — no `/renew` endpoint.

`Policy.MaxRenewalCount` is loaded in `LoanToolCommand` and `ReserveToolCommand` but never consulted by either handler. The domain method is reachable only from unit tests, not from any HTTP endpoint or application handler. Half-wiring is the danger: the contract looks complete but the feature is dead in production.

**Affected Files:** `DependencyInjection.cs` (register a new `RenewLoanCommandHandler`), new `Loan/Commands/RenewLoanCommand.cs`, `LoansCommandController.cs` (new `POST /api/loans/{id}/renew`).

**Suggested fix:** Add `RenewLoanCommand(LoanId)` + handler that loads the loan → resolves the `Tool` via `InventoryItem.ToolId` → loads the policy by `(ToolType, null)` → calls `loan.Renew(policy.MaxLoanDurationDays, policy.MaxRenewalCount)`.

---

### 5. MarkToolForRepairCommandHandler Still Violates One-Aggregate-Per-Tx

> **GitHub Issue:** [tsqr-tool-lib#46](https://github.com/lstbob/tsqr-tool-lib/issues/46) (P1, 5 SP)

**Severity:** High — audit §1 listed this handler among the boundary violators; #33 closed 3 of 4, this one was left.

**Evidence:** `Inventory/Commands/MarkToolForRepairCommand.cs:11-32`:

```csharp
var markResult = item.MarkForRepair(...);                              // mutate InventoryItem
var recordResult = MaintenanceRecord.Create(...);                     // create MaintenanceRecord
await maintenanceRepository.AddAsync(record, ct);
inventoryRepository.Update(item);
await orchestrator.SaveEntitiesAsync([item, record], ct);             // both aggregates in one tx
```

`item.MarkForRepair()` already raises `ToolMarkedForRepairEvent` — the side effect (`MaintenanceRecord` creation) belongs in a `ToolMarkedForRepairNotificationHandler`, parallel to how `InventoryItemRequiredEvent` works for tool registration.

**Affected Files:** `MarkToolForRepairCommandHandler.cs`, `ToolMarkedForRepairNotificationHandler.cs` (add `MaintenanceRecord` creation), `DependencyInjection.cs`.

**Suggested fix:** Remove the inline `MaintenanceRecord.Create`/`AddAsync` from the command; have `ToolMarkedForRepairNotificationHandler` create the `MaintenanceRecord` (it already cancels reservations — extend it rather than split). The originating command then mutates only `InventoryItem` per tx.

---

### 6. LoanToolCommand Lost Loanability Pre-check + Member.IsEligibleToBorrow Ignores Fines

> **GitHub Issue:** [tsqr-tool-lib#47](https://github.com/lstbob/tsqr-tool-lib/issues/47) (P1, 5 SP)

**Severity:** High.

**Evidence (two related concerns):**

**(a)** `LoanToolCommandHandler.cs` (post-#33) never loads the `InventoryItem`; it goes straight to `Loan.Create(...)` + `orchestrator.SaveEntitiesAsync`. The `InventoryItem.Loan()` side effect was moved into `LoanCreatedDomainEventHandler.cs:7-17`, where it has two escape paths: succeed, or throw `InvalidOperationException`. A user loaning an already-loaned / `Lost` / non-existent item now receives `500 Internal Server Error` via `ExceptionHandlingMiddleware` instead of a domain-shaped `Result` 400. Pre-#33 the handler returned `DomainError` synchronously.

**(b)** `Member.IsEligibleToBorrow()` (`Member.cs:222-225`):
```csharp
public bool IsEligibleToBorrow()
{
    return IsVerified && Status == MemberStatus.Active;     // nothing about fines or overdue loans
}
```

With #35 making fines snapshot-driven and actually accruable, this is the natural gate. A member who returned a tool 6 months late and owes the library $200 can immediately borrow another. There is also no invariant that a member can't hold **two active loans on the same item** — currently enforced only incidentally via `InventoryItem.Loan`'s `Status == Loaned` check, which depends on item state, not on the loan aggregate.

**Affected Files:** `LoanToolCommand.cs`, `Member.cs`, `LoanCreatedDomainEventHandler.cs`.

**Suggested fix:**
1. Load the `InventoryItem` in the command; pre-check loanability; only then call `Loan.Create` so handlers don't throw on known-bad state.
2. Add an `ILoanRepository.GetActiveLoansByMemberAsync(MemberId)` lookup in `LoanToolCommandHandler`; reject if outstanding fine balance > threshold, or if the member already has an active loan on the same item.
3. Either enhance `Member.IsEligibleToBorrow(outstandingFines, activeOverdueLoans)` or split into a `MemberBorrowingEligibilityService` domain service.

---

## Communities — NEW Issues

### 7. Authorization Wide Open; QuickRegister Is Anonymous

> **GitHub Issue:** [tsqr-communities#6](https://github.com/lstbob/tsqr-communities/issues/6) (P0, 3 SP)

**Severity:** Critical.

**Evidence:** `CommunitiesController.cs:68-105`:

```csharp
[ApiController]
[Route("api/communities")]
[Authorize]                                    // no role / no policy
public sealed class CommunitiesCommandController(...) : ControllerBase
{
    [HttpPatch("{id}/confirm")]
    public async Task<IActionResult> Confirm(int id) { ... }   // any authenticated user can confirm ANY community

    [HttpPost("quick-register")]
    [AllowAnonymous]                            // internet-wide open
    public async Task<IActionResult> QuickRegister(...) { ... }
}
```

There is no domain concept of "Community Moderator", "Platform Admin", or "Community Owner". Anyone with any valid JWT can confirm / rename / update ANY community. Worse, `QuickRegister` is fully anonymous and persists real geography rows + a `PendingConfirmation` community from unauthenticated traffic.

**Affected Files:** `CommunitiesController.cs`, `Program.cs` (authorization policies), all write controllers in `Communities.WebApi`.

**Suggested fix:** Introduce `PlatformAdmin` and `CommunityModerator` authorization policies; remove `[AllowAnonymous]` from `QuickRegister` (or rate-limit + verify email before persisting).

---

### 8. State-Transition Guards + UNIQUE Constraints + Suspend/Archive Commands

> **GitHub Issue:** [tsqr-communities#7](https://github.com/lstbob/tsqr-communities/issues/7) (P1, 8 SP)

**Severity:** High — three issues in one cohesive refactor.

**Evidence (a) — unconditional state-setting:** `Community.cs:29-31`:
```csharp
public void Confirm()  { Status = CommunityStatus.Active; }
public void Suspend()  { Status = CommunityStatus.Suspended; }
public void Archive()  { Status = CommunityStatus.Archived; }
```
`Confirm()` re-activates communities already `Active`/`Archived`. `Archive()` accepts `PendingConfirmation`. Mirrors audit §13 but on a different aggregate.

**Evidence (b) — unreachable behavior:** `Suspend()` and `Archive()` defined on the aggregate but no matching command, handler, or endpoint. The "Suspended" state is named in `CommunityStatus` and seeded into `DashboardQueries.cs:14` but no production path ever produces it. Dedicated dashboard COUNT(*) tracking is forever zero.

**Evidence (c) — schema missing UNIQUE:**
- `tsqr-deploy/infra/07-communities.sql:22-32` — `Communities.Slug` has no UNIQUE.
- `07-communities.sql:4-20` — no UNIQUE on `Countries.Name`, `Cities(CountryId, Name)`, `Neighbourhoods(CityId, Name)`. The find-or-create pattern in `QuickRegisterCommunityCommand.cs:26-53` then silently duplicates geography rows on concurrent calls.

**Affected Files:** `Community.cs` (`Result`-returning guarded methods), `City.cs` / `Neighbourhood.cs` (same), new `SuspendCommunityCommand` / `ArchiveCommunityCommand`, `DependencyInjection.cs`, `07-communities.sql`.

**Suggested fix:** Each transition returns `Result`; introduce the two missing commands; add a `CREATE UNIQUE INDEX IX_Communities_Slug` on `Communities(Slug)` and `UNIQUE (CountryId, Name)` on `Cities` / `UNIQUE (CityId, Name)` on `Neighbourhoods`.

---

### 9. Read Query Layer Is in the Wrong Place (WebApi + Domain)

> **GitHub Issue:** [tsqr-communities#8](https://github.com/lstbob/tsqr-communities/issues/8) (P1, 5 SP)

**Severity:** High — violates audit §3 ("Domain layer has zero infrastructure dependencies").

**Evidence (a) — raw SQL/Dapper embedded in WebApi controllers:** `CommunitiesReadController.cs:11-65`, `CitiesReadController.cs:11-22`, `CountriesReadController.cs:11-20`, `NeighbourhoodsReadController.cs:11-22`:
```csharp
public sealed class CommunitiesReadController(ISqlUnitOfWork uow) : ControllerBase
{
    [HttpGet]
    public async Task<PagedResult<CommunityListItem>> GetAll(...)
    {
        const string filter = @"FROM Communities c JOIN Neighbourhoods n ON ... WHERE ...";
        var totalCount = await uow.Connection.ExecuteScalarAsync<int>($"SELECT COUNT(*) {filter}", args);
        ...
    }
}
```

**Evidence (b) — query contracts live in Domain:** `Communities.Domain/IDashboardQueries.cs:3`, `IGeographyQueries.cs:5-10`:
```csharp
public interface IDashboardQueries { Task<DashboardStatsData> GetStatsAsync(CancellationToken ct = default); }
public record DashboardStatsData(int TotalCommunities, ...);
```

The Domain layer is no longer technology-neutral — it explicitly knows it offers queries. The audit praises "CQRS Done Right" elsewhere; communities reads have no such discipline.

**Affected Files:** All `*ReadController.cs`, `IDashboardQueries.cs`, `IGeographyQueries.cs`, plus new `Application/Queries/ICommunityQueries.cs` and `Infrastructure/Dapper/Queries/CommunityQueries.cs`.

**Suggested fix:** Move `IDashboardQueries`, `IGeographyQueries`, and `DashboardStatsData` into the Application layer; add `ICommunityQueries` for the controller-embedded raw SQL; controllers depend only on the query interface, never on `ISqlUnitOfWork`.

---

### 10. Slug Immutability + VO-Typed FKs + CountryCode Value Object

> **GitHub Issue:** [tsqr-communities#9](https://github.com/lstbob/tsqr-communities/issues/9) (P2, 5 SP)

**Severity:** Medium — three related modelling refinements.

**Evidence (a):** `Community.cs:32-39`:
```csharp
public void UpdateDetails(string name, ...) {
    Name = name;
    Slug = GenerateSlug(name);          // destroys the existing slug on every PUT
}
```
Audit §10 covers **generation**; the more insidious problem is **immutability of slug-as-identity**. Renaming silently rewrites URLs others have bookmarked.

**Evidence (b) — primitive obsession:** `City.cs:10` (`int countryId`), `Neighbourhood.cs:10` (`int cityId`), `Community.cs:16` (`int neighbourhoodId`) — value-object ID types *exist* (`CountryId`, `CityId`, `NeighbourhoodId`) but are unused on the cross-aggregate references. `IGeographyQueries` returns bare `int?` and the handler passes raw ints back into constructors.

**Evidence (c):** `Country.cs:10-14` accepts any `code`:
```csharp
public Country(string name, string code) {
    Name = name;
    Code = code.ToUpperInvariant();      // empty string, "ABCDEFG", null → all accepted
}
```
ISO 3166-1 alpha-2 country code has a well-defined format (2 uppercase letters); nothing enforces this anywhere (domain, application, or schema). Schema has no `CHECK` / `UNIQUE` on `Countries.Code`.

**Affected Files:** `Community.cs`, `City.cs`, `Neighbourhood.cs`, `Country.cs`, `IGeographyQueries.cs`, `07-communities.sql`.

**Suggested fix:**
1. Treat `Slug` as immutable once assigned, or expose `ChangeSlug(newSlug)` with a uniqueness check.
2. Change public constructors to take `CountryId` / `CityId` / `NeighbourhoodId`; reject `default` in the constructor. Update `IGeographyQueries` to return the VO types.
3. Introduce a `CountryCode` value object (2-uppercase-letter validation); add `UNIQUE` on `Countries.Code` and `Countries.Name`.

---

## Soup Kitchen — NEW Issues

### 11. Multi-Tenancy Declared but Not Enforced; Reads Expose Cross-Tenant PII

> **GitHub Issue:** [tsqr-soup-kitchen#6](https://github.com/lstbob/tsqr-soup-kitchen/issues/6) (P0, 8 SP)

**Severity:** Critical — privacy breach.

**Evidence (a) — no `CommunityId` filter on any query:** `WebApi/Queries/GetEventsQuery.cs:13-37`, `GetMealsQuery.cs:11-37`, `GetGuestsQuery.cs:11-32`, `GetVolunteersQuery.cs:11-41`, `GetDonationsQuery.cs:11-31` — `grep CommunityId` in `WebApi/Queries` returns zero matches.

**Evidence (b) — caller controls `CommunityId`:** `EventsController.cs:48`, `:58`:
```csharp
new CreateEventCommand(request.Name, request.Description, request.EventDate,
    request.Location, request.MaxGuests,
    (request.CommunityId <= 0 ? 1 : request.CommunityId));
```
Identical pattern in `GuestsController.cs:38`, `MealsController.cs:39`, `VolunteersController.cs:39`, `DonationsController.cs:38`. `UpdateEventCommand.cs:24-31` lets a caller re-tenant an existing event.

**Evidence (c) — all reads `[AllowAnonymous]`:** `EventsController.cs:10-12`, `GuestsController.cs:10-12`, `DonationsController.cs:10-12`, `MealsController.cs:10-12`, `VolunteersController.cs:10-12`, `DashboardController.cs:8-10`. Each overrides the global `RequireAuthenticatedUser()` fallback. Combined with (a) and (b): the public internet can enumerate `?eventId=1,2,3,...` across all communities and harvest guest emails / personal contact info / donor names.

**Affected Files:** All `*Controller.cs` read controllers (remove `[AllowAnonymous]`); all `WebApi/Queries/*.cs` (add `WHERE CommunityId = @CommunityId`); all `*Command.cs` handlers (derive `CommunityId` from JWT, never the body).

**Suggested fix:**
1. Remove `[AllowAnonymous]` from all read controllers.
2. Derive `CommunityId` from the authenticated member's JWT claim (the `cid` / `mbr` claim — see cross-cutting #29).
3. Add `WHERE CommunityId = @CommunityId` to every list, detail, count, and dashboard query.
4. Reject `UpdateEventCommand`s whose `CommunityId` differs from the loaded aggregate's current `CommunityId`.

---

### 12. Cross-Aggregate Invariants: Event Existence/Capacity Checks Missing in Child Commands

> **GitHub Issue:** [tsqr-soup-kitchen#7](https://github.com/lstbob/tsqr-soup-kitchen/issues/7) (P1, 8 SP)

**Severity:** High.

**Evidence:** `GuestCommands/Commands/CreateGuestCommand.cs:13-19`:
```csharp
public sealed class CreateGuestCommandHandler(
    IRepository<Guest, GuestId> guestRepository)              // no Event repo
{
    public async Task<Result<GuestId>> ExecuteAsync(...) {
        var guest = new Guest(request.EventId, request.Name, request.ContactInfo,
                              request.GuestCount, request.CommunityId);
        await guestRepository.AddAsync(guest, ct);            // EventId never validated
    }
}
```
Same shape in `CreateMealCommand.cs:15-26`, `CreateVolunteerCommand.cs:14-25`, `CreateDonationCommand.cs:14-25`. An attacker POSTing `{ "EventId": 999999 }` receives a 500 (Postgres FK violation through `ExceptionHandlingMiddleware`), never a typed `NotFoundError`. Capacity invariants (`SUM(GuestCount) <= Event.MaxGuests`) — the entire reason `MaxGuests` exists on `Event` — are never checked anywhere.

**Affected Files:** `Create*CommandHandler.cs` (all four), optionally extend the `Event` aggregate to expose `HasCapacity(int count)`.

**Suggested fix:** Either (a) collapse `Guest`/`Meal`/`Volunteer`/`Donation` into the `Event` aggregate (audit §5) so `Event.RegisterGuest(...)` enforces invariants atomically; or (b) keep them separate but load `Event` in each handler, validate existence + capacity + community-match, and use a domain service for capacity arithmetic.

---

### 13. Unreachable Behavior: State Machines, Donation Aggregate, and Command Handlers

> **GitHub Issue:** [tsqr-soup-kitchen#8](https://github.com/lstbob/tsqr-soup-kitchen/issues/8) (P1, 8 SP)

**Severity:** High — three related "domain lies" (behavior advertised but unreachable).

**Evidence (a) — state machines not enforced:** `Event.cs:45-58`:
```csharp
public void Cancel()   { Status = EventStatus.Cancelled; }
public void Complete() { Status = EventStatus.Completed; }
public void Activate() { Status = EventStatus.Active;   }
// no guard on previous Status
```
A `Completed` event can be re-`Activate()`d; a `Cancelled` event can be `Complete()`d. Same at `Guest.cs:27-35` and `Volunteer.cs:27-35`. `UpdateEventCommand.cs:29` raw-casts `(EventStatus)request.Status` with no range or transition check; `Event.UpdateDetails` accepts any value (e.g., `99`).

**Evidence (b) — `Donation` has literally zero behavior:** `Donation.cs:1-26` is just public fields and a constructor. No `Cancel`/`Refund`/`CorrectQuantity`/`MarkReceived`. Even simple domain operations (donations do get returned/corrected in real soup kitchens) are absent. `DonationMapping.cs` has `UpdateSql` / `ToUpdateParameters` but no handler ever calls `donationRepository.Update`.

**Evidence (c) — domain methods unreachable from API:** `Guest.Confirm/Cancel`, `Volunteer.Confirm/Cancel`, `Event.Cancel/Complete/Activate` exist; no command handlers exist (verified by `ls Application/*/Commands/`). The only way to "confirm" a guest today is raw SQL — the named API the domain advertises is fictional.

**Affected Files:** `Event.cs` / `Guest.cs` / `Volunteer.cs` (add guard clauses returning `Result`), `Donation.cs` (add behavioral methods), new `ConfirmGuestCommandHandler`, `CancelVolunteerCommandHandler`, `CancelEventCommandHandler`, `DependencyInjection.cs`.

**Suggested fix:** Add state-machine guards (e.g., `if (Status == Completed) return Result.Failure(...)`); add `Donation.Cancel(reason)` / `CorrectQuantity(int)` / `MarkReceived()` methods; add the missing command handlers + DI registrations + controller endpoints.

---

### 14. Unique Constraints + Domain Events + Faithful UnitOfWork

> **GitHub Issue:** [tsqr-soup-kitchen#9](https://github.com/lstbob/tsqr-soup-kitchen/issues/9) (P2, 5 SP)

**Severity:** Medium — three scaffold issues.

**Evidence (a) — missing UNIQUE on natural keys:** `06-soup-kitchen.sql:25-33` for `Volunteers`:
```sql
CREATE TABLE IF NOT EXISTS Volunteers (
    Id SERIAL PRIMARY KEY,
    EventId INTEGER NOT NULL REFERENCES Events(Id),
    MemberId INTEGER NOT NULL,
    Role INTEGER NOT NULL DEFAULT 6,
    -- UNIQUE (EventId, MemberId) absent
);
```
No domain-level `Volunteer.HasAlreadySignedUp` check exists either — a member can volunteer for the same event 5 times.

**Evidence (b) — domain event dispatcher built but never wired:** `Application/Events/IDomainEventDispatcher.cs` and `DomainEventDispatcher.cs` exist; `Program.cs` never registers `IDomainEventDispatcher` in DI; `grep AddDomainEvent` returns only the `Entity.cs:9` definition with zero callers; no `IDomainEvent` implementation types exist (no `EventCreated`, `GuestRegistered`, `DonationReceived`, etc.). Entire framework is orphan code.

**Evidence (c) — UnitOfWork contract lies:** `DapperUnitOfWork.cs:30-58` — `SaveChangesAsync` returns a hardcoded `int 0` despite the `Task<int> SaveChangesAsync(...)` contract. The retry loop also re-issues `CommitAsync` after a rollback but never re-runs the Dapper commands, silently committing an empty transaction.

**Affected Files:** `06-soup-kitchen.sql` (add `UNIQUE (EventId, MemberId)` to Volunteers, similar to Guests); `Program.cs` (register `IDomainEventDispatcher`); new event records + aggregate raises; `IUnitOfWork.cs` (change to `Task`) OR `DapperUnitOfWork.cs` (aggregate affected rows).

**Suggested fix:**
1. Add `UNIQUE (EventId, MemberId)` to Volunteers and a domain-level guard for in-process validation.
2. Either delete the unused `Application/Events/` dispatcher files, or complete the pipeline (register in DI, define records, raise from aggregates).
3. Change `IUnitOfWork.SaveChangesAsync` to `Task` (honest no-op), OR track and aggregate affected rows.

---

## Identity — NEW Issues

### 15. Email-Change Feature Is Non-Functional + Token Returned in HTTP Body

> **GitHub Issue:** [tsqr-identity#5](https://github.com/lstbob/tsqr-identity/issues/5) (P0, 5 SP)

**Severity:** Critical — non-functional write-path + security token leak.

**Evidence:** `Commands/Profile/ChangeEmailCommand.cs:5-11`:
```csharp
public class ChangeEmailCommandHandler : IInteractor<ChangeEmailCommand, Result>
{
    public Task<Result> ExecuteAsync(ChangeEmailCommand command, CancellationToken ct = default)
    {
        return Task.FromResult(Result.Success());   // does literally nothing
    }
}
```

`Commands/Profile/RequestEmailChangeCommand.cs:10-14`:
```csharp
var token = Convert.ToBase64String(RandomNumberGenerator.GetBytes(32));
return Result<string>.Success(token);   // never persisted anywhere
```

`tsqr-deploy/infra/05-identity.sql:27-35` defines `pending_email_changes` (with 3 indexes) — `grep pending_email_changes` across all `.cs` files returns **zero matches**. The controller returns the security token directly to the caller in `ProfileController.cs:63-68`, defeating the entire purpose of email verification.

**Affected Files:** `ChangeEmailCommand.cs`, `RequestEmailChangeCommand.cs`, `ProfileController.cs`, new `PendingEmailChange` aggregate or repository, `05-identity.sql` (verify schema is what handler persists against).

**Suggested fix:**
1. Introduce a `PendingEmailChange` aggregate (or at minimum a repository over `pending_email_changes`) whose lifecycle is `Requested → Confirmed/Expired`.
2. `RequestEmailChange` should persist `(userId, newEmail, tokenHash, expiresAt)`, dispatch the raw token via an out-of-band email-sending domain event / handler, and return `Success` *without* the token.
3. `ChangeEmail` should load by token hash, verify expiry, then update `Users.Email` in autheo (or raise an `EmailChanged` integration event — see cross-cutting #29 for integration events story).

---

### 16. IAggregateRoot Defined but Never Implemented; No Entity Base + No Value Objects

> **GitHub Issue:** [tsqr-identity#6](https://github.com/lstbob/tsqr-identity/issues/6) (P1, 8 SP)

**Severity:** High — corrects v1 §8 and documents the consequences.

**Evidence:** `Domain/Contracts.cs:3`:
```csharp
public interface IAggregateRoot;
```
`Domain/Entities/UserProfile.cs:3` and `Entities/Role.cs:3`:
```csharp
public sealed class UserProfile   // no base class, no IAggregateRoot
public sealed class Role           // no base class, no IAggregateRoot
```

The scaffolding (`IAggregateRoot`, `IUnitOfWork`) is decorative. `UserProfile` and `Role` are plain `sealed class` with no base, no `IAggregateRoot`, no `DomainEvents` collection, no identity equality, no `Id` wrapper. v1 §8 claimed identity "*inherits the `Entity` base class which has a `DomainEvents` collection*"; identity has neither — that part of the audit was inaccurate.

**Affected Files:** `Domain/Contracts.cs`, `Entities/UserProfile.cs`, `Entities/Role.cs`, new `Domain/Common/Entity.cs`, new value objects (`UserId`, `RoleId`, `Email`, `AvatarUrl`, `PersonName`, `Bio`).

**Suggested fix:** Pick one of:
- **(a)** Delete the unused `IAggregateRoot` marker (be honest about the anemic model).
- **(b)** Apply it: introduce an `Entity<TId>` base with identity equality + a `DomainEvents` collection, mark `UserProfile : Entity<UserId>, IAggregateRoot`, route all mutation through named methods, and add value objects for ID, Email, AvatarUrl, PersonName, Bio.

This single refactor resolves items #16, #17, #18, #19 simultaneously.

---

### 17. LockUserCommand Has No Invariant Validation + No Audit of Who Locked

> **GitHub Issue:** [tsqr-identity#7](https://github.com/lstbob/tsqr-identity/issues/7) (P1, 5 SP)

**Severity:** High.

**Evidence:** `Commands/Users/LockUserCommand.cs:5,16-17`:
```csharp
public record LockUserCommand(string UserId, DateTime? LockoutEnd);
...
user.LockoutEnd = command.LockoutEnd;   // no validation
user.UpdatedAt = DateTime.UtcNow;       // no record of WHO locked them
```

`UsersController.cs:87-90`:
```csharp
public async Task<ActionResult> Unlock(string id, CancellationToken ct)
{
    var command = new LockUserCommand(id, null);   // "unlock" = lock-with-null hack
}
```

A past-dated `LockoutEnd` (e.g., `DateTime.MinValue`) is silently meaningless. A year-9999 future date is unbounded. There's no "lock reason," no "locked by." `Unlock` isn't a domain concept — it's a controller hack reusing the same command. Because `UserProfile.LockoutEnd` is a public setter (audit §8), the invariant lives nowhere.

`grep -rn 'CreatedBy|UpdatedBy|DeletedBy'` across the repo returns **zero**. Every handler sets `UpdatedAt = DateTime.UtcNow` but never records who. `user_roles` join has no `AssignedAt`/`AssignedBy`. For an identity/admin service this is a compliance gap: there's no way to answer "who suspended user X?" or "who granted user Y the Admin role?".

**Affected Files:** `LockUserCommand.cs` (split into `SuspendUserCommand` + `ReinstateUserCommand`), `UserProfile.cs` (add `Suspend(DateTime, string reason, string lockedBy)` + `Reinstate()` methods with guards), `user_profiles` and `user_roles` schema (`AssignedAt`, `AssignedBy`, `LockedBy`), every handler populating audit fields from the JWT `sub` claim.

**Suggested fix:** Add `UserProfile.Suspend(DateTime until, string reason, string lockedBy)` and `UserProfile.Reinstate()` methods with guards (`until > UtcNow`, `until < UtcNow + MaxLockout`); raise `UserLockedOut`/`UserReinstated` events; expose separate `SuspendUserCommand` / `ReinstateUserCommand` rather than overloading Lock with null. Add `CreatedBy`/`UpdatedBy` columns to `user_profiles` and `AssignedAt`/`AssignedBy` to `user_roles`; populate from the JWT `sub` claim.

---

### 18. CancellationToken Silently Dropped + Upsert Race on UpdateProfile

> **GitHub Issue:** [tsqr-identity#8](https://github.com/lstbob/tsqr-identity/issues/8) (P1, 3 SP)

**Severity:** High.

**Evidence (a):** `Infrastructure/Repositories/ProfileRepository.cs:9-13` (and every other repo method):
```csharp
public async Task<UserProfile?> GetByIdAsync(string userId, CancellationToken ct = default)
{
    return await uow.Connection.QuerySingleOrDefaultAsync<UserProfile>(
        "SELECT * FROM user_profiles WHERE Id = @Id", new { Id = userId });
    // ct is never referenced; DapperConnection.QuerySingleOrDefaultAsync overloads don't even accept CancellationToken
}
```

Every method in `ProfileRepository`, `RoleRepository`, `UserAdminRepository` accepts `ct` and ignores it. A client disconnect on `GET /api/users` (unbounded — audit §21) cannot cancel the in-flight query; the worker thread + DB connection stay pinned. This is a new variant of §28 (which covered autheo's `.GetAwaiter().GetResult()`); identity just silently swallows the token.

**Evidence (b) — "create-on-first-update" upsert:** `Commands/Profile/UpdateProfileCommand.cs:12-33`:
```csharp
var profile = await profiles.GetByIdAsync(command.UserId, ct);
if (profile is null)
{
    profile = new UserProfile
    {
        Id = command.UserId,
        FirstName = command.FirstName,
        ...
    };
    await profiles.CreateAsync(profile, ct);   // no ON CONFLICT
}
```

Same pattern in `ProfileController.UploadAvatar` with a *different* field set (only `Id` + `AvatarUrl`). Three problems compound:
- **No factory**: profile created via avatar upload has null `FirstName`/`LastName`/`Bio`; one created via profile update has null `AvatarUrl`.
- **Race condition**: two concurrent `PUT /api/profile` calls both see `null`, both INSERT; second throws a PK violation (no `ON CONFLICT`).
- **No `UserProfileCreated` event**: downstream consumers (welcome email, community default-role assignment) have no signal. `CreatedAt` becomes "first time the user happened to update their profile", not actual account-creation time.

**Affected Files:** `ISqlConnection` methods (add `CancellationToken`); all `*Repository.cs` (forward `ct` to Dapper's `QueryAsync<T>(..., cancellationToken)` overloads); `UpdateProfileCommand.cs` and `ProfileController.cs` (replace create-if-missing with a dedicated `CreateUserProfileCommand` or `UserRegistered` integration event consumer); `ProfileRepository.CreateAsync` (`INSERT ... ON CONFLICT (Id) DO NOTHING`).

---

### 19. Role Has No Permission Model + String-Role Authorization

> **GitHub Issue:** [tsqr-identity#9](https://github.com/lstbob/tsqr-identity/issues/9) (P2, 5 SP)

**Severity:** Medium.

**Evidence:** `Entities/Role.cs:3-7`:
```csharp
public sealed class Role
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string NormalizedName { get; set; } = string.Empty;
}
```

Authorization is enforced entirely by string literals at the controller boundary:
```csharp
[Authorize(Roles = "Admin")]   // tool-lib, communities, identity
[Authorize(Roles = "Admin,LocationCoordinator,Repairman,Regular")]   // ~20 sites
```

`05-identity.sql:42-46` seeds four roles but they carry no permissions. `Role` is a label, not a capability. There's no `Permission` value object, no `Role.Permissions` collection, no `IRoleAuthorizationService`. Adding a new permission (e.g., "can lock users") requires editing the `[Authorize(Roles = "...")]` string literal in every controller — nothing keeps them in sync. The role names are duplicated between the SQL seed, the JWT claims issued by autheo, and the controller attributes.

Also: `RoleRepository.DeleteAsync` (`Repositories/RoleRepository.cs:28-31`) is dead code — no `DeleteRoleCommand` ever calls it. If anyone wired it naively, `05-identity.sql:21-25` declares `user_roles.RoleId REFERENCES roles(Id)` with no `ON DELETE CASCADE`, so it would throw an FK violation rather than a domain-shaped `RoleInUseError`.

**Affected Files:** `Role.cs` (add `Permissions`); new `Permission` value object; new `roles_permissions` table; replace `[Authorize(Roles="...")]` with `[Authorize(Policy="...")] across all six microservices`; `IRoleRepository` (remove or guard `DeleteAsync`).

**Suggested fix:** Model `Permission` as a value object (`Permission.CanLockUsers`, etc.); give `Role` a `permissions` table or JSON column; replace `[Authorize(Roles = "Admin")]` with policy-based `[Authorize(Policy = "CanLockUsers")]` that checks the user's resolved permission set. Soft-delete roles rather than hard-delete per v1 §25.

---

## Autheo — NEW Issues

### 20. /register Self-Admin + Broken ForgotPassword/ResetPassword + RevokedReasonIsReuse Not Persisted

> **GitHub Issue:** [tsqr-autheo#3](https://github.com/lstbob/tsqr-autheo/issues/3) (P0, 5 SP)

**Severity:** Critical — three Critical security/functional bugs.

**Evidence (a) — Anonymous /register lets callers self-promote to Admin:** `AuthController.cs:28-46`, `RegisterHandler.cs:29`:
```csharp
[HttpPost("register"), AllowAnonymous]
public async Task<IActionResult> Register(RegisterRequest req, ...) {
    RoleName? role = null;
    if (!string.IsNullOrEmpty(req.Role) && Enum.TryParse<RoleName>(req.Role, true, out var r))
        role = r;
    var result = await register.ExecuteAsync(new RegisterCommand(req.Email, req.Password, ..., role, ...), ...);
}

// RegisterHandler
var role = cmd.Role ?? RoleName.Regular;
var user = User.Create(..., role, cmd.MemberId);
```

Any unauthenticated caller can POST `{"role":"Admin"}` and become Admin. `RoleName.Admin` maps directly to the JWT `role=Admin` claim downstream tool-lib `[Authorize(Roles="Admin")]` honors. `User.Create` blindly accepts the role.

**Evidence (b) — ForgotPassword/ResetPassword never commit + missing table:** `ForgotPasswordHandler.cs:11-29`, `ResetPasswordHandler.cs:13-39`:
```csharp
public sealed class ForgotPasswordHandler(IUserRepository userRepo, IPasswordResetTokenRepository tokenRepo) // no IUnitOfWork
{
    public async Task<Result> ExecuteAsync(...) {
        await tokenRepo.AddAsync(resetToken, ct);
        return Result.Success();   // never committed
    }
}
```

`infra/init.sql:1-44` declares only `Users` and `RefreshTokens` — no `PasswordResetTokens` table. Repository queries against `PasswordResetTokens` will throw `PostgresException: relation "passwordresettokens" does not exist`. The endpoint hard-crashes (caught by `ExceptionHandlingMiddleware` → 500). The "user enumeration prevention" praised in audit Good #7 returns 200 on unknown email, masking that the feature is dead.

**Evidence (c) — RevokedReasonIsReuse computed but not persisted:** `RefreshToken.cs:17,65-70`:
```csharp
public void Revoke(string? byIp = null, bool reuseDetected = false) {
    RevokedAt = DateTime.UtcNow;
    RevokedByIp = byIp ?? "unknown";
    RevokedReasonIsReuse = reuseDetected ? true : null;
}
```

`RefreshTokenRepository.cs:42-57` UPDATE omits `RevokedReasonIsReuse`:
```csharp
uow.Connection.ExecuteAsync(
    @"UPDATE RefreshTokens SET RevokedAt = @RevokedAt,
                                 RevokedByIp = @RevokedByIp,
                                 ReplacedByTokenHash = @ReplacedByTokenHash
        WHERE Id = @Id", ...).GetAwaiter().GetResult();
```

`infra/init.sql` doesn't declare the `RevokedReasonIsReuse` column. The audit praiseworthy "family-revocation pattern" cannot record *why* a revocation happened — the security-anomaly audit trail is fictional.

**Affected Files:** `AuthController.cs` (drop `Role` from public `RegisterRequest`); `RegisterHandler.cs` (force `RoleName.Regular`); `ForgotPasswordHandler.cs` / `ResetPasswordHandler.cs` (inject `IUnitOfWork` + `SaveChangesAsync`); `infra/init.sql` (add `PasswordResetTokens` table + `RevokedReasonIsReuse BOOLEAN NULL` column); `RefreshTokenRepository.cs` (include `RevokedReasonIsReuse` in SELECT/INSERT/UPDATE); `RefreshToken.Rehydrate` (hydrate the column).

---

### 21. Concurrent Refresh Token Rotation Has No Optimistic Lock + Revoke Overwrites Audit

> **GitHub Issue:** [tsqr-autheo#4](https://github.com/lstbob/tsqr-autheo/issues/4) (P0, 5 SP)

**Severity:** Critical — defeats the security feature praised in audit Good #7.

**Evidence (a):** `RefreshHandler.cs:17-61`, `RefreshTokenRepository.cs:42-57`:
```csharp
var existing = await refreshRepo.GetByTokenHashAsync(tokenHash, ct);
if (existing.RevokedAt is not null) { ... }
if (!existing.IsActive) { ... }
// No lock between the check and Rotate below
var replacement = existing.Rotate(...);
await refreshRepo.AddAsync(replacement, ct);
refreshRepo.Update(existing);   // UPDATE RefreshTokens SET ReplacedByTokenHash=... WHERE Id=@Id
await uow.SaveChangesAsync(ct);
```

The UPDATE has no `xmin` / `RowVersion` / `ReplacedByTokenHash IS NULL` predicate; `ReplacedByTokenHash` is not declared `UNIQUE` in `init.sql`. Two concurrent refresh calls with the same parent both pass the `IsActive` check, both `Rotate`, both insert a different replacement, both update the parent — last write wins, leaving one replacement orphaned and the family chain forked. From then on, reuse detection walks a single lineage and silently ignores the orphan branch.

**Evidence (b) — `Revoke` overwrites prior audit trail:** `RefreshToken.cs:65-70`, `RefreshHandler.cs:30-35`:
```csharp
public void Revoke(string? byIp = null, bool reuseDetected = false) {
    RevokedAt = DateTime.UtcNow;          // always overwrites
    RevokedByIp = byIp ?? "unknown";      // always overwrites
}
// RefreshHandler reuse branch:
if (existing.RevokedAt is not null) {
    existing.Revoke(cmd.ClientIp, reuseDetected: true);  // re-revokes an already-revoked token
}
```

When reuse is detected, `Revoke` is called on the original (already-rotated, already-revoked) token. Its earlier `RevokedAt` (when the legitimate rotation happened) and `RevokedByIp` (the legitimate client IP) are silently overwritten with the attacker's IP and the current timestamp. The exact data needed for incident forensics is destroyed.

**Affected Files:** `RefreshTokenRepository.cs` (add `WHERE Id = @Id AND ReplacedByTokenHash IS NULL`; treat 0-rows-affected as concurrent rotation → trigger reuse detection); `RefreshToken.cs` (guard against double-revocation); `init.sql` (declare `UNIQUE` on `ReplacedByTokenHash`).

**Suggested fix:**
1. Add the optimistic lock predicate; read affected rows; on 0-affected, revoke the family for reuse.
2. Guard `Revoke`: if `RevokedAt is not null`, only update `RevokedReasonIsReuse = true` (do NOT overwrite `RevokedAt`/`RevokedByIp`).
3. Add `UNIQUE` on `RefreshTokens.ReplacedByTokenHash` (partial, where not null).

---

### 22. No Rate Limiting + Single Static JWT Signing Key + Single Audience Across 7 Services

> **GitHub Issue:** [tsqr-autheo#5](https://github.com/lstbob/tsqr-autheo/issues/5) (P1, 8 SP)

**Severity:** High — three related hardening gaps.

**Evidence (a):** `Autheo.WebApi/Program.cs:15-87` — no `AddRateLimiter`/`UseRateLimiter`; no `[EnableRateLimiting]`. Authentication endpoints are textbook brute-force / credential-stuffing targets. The praised "user enumeration prevention" returns 200 on unknown email — but without rate limiting, an attacker mass-enumerates unthrottled. `/login` is bounded only by PBKDF2 cost (~50-200 ms/core).

**Evidence (b):** `IJwtTokenService.cs:30-57`, `Program.cs:20`, all six other services use the same `JWT_KEY`:
- `tsqr-deploy/.env.example:32-35` — one `JWT_KEY` variable
- All consumers use `Issuer="tsqr-autheo"`, `Audience="tsqr-services"`
- Symmetric signing means **every verifier** holds the secret. Compromise *any* one of the seven containers and an attacker can forge tokens for every service.
- No `kid` header, no key rotation, no vault integration (Vault/AWS Secrets), no per-service audience (`aud=tsqr-tool-lib`, `aud=tsqr-identity`, etc.).

Symmetric-key tokens issued for autheo are equivalently valid for tool-lib, identity, and every other context.

**Affected Files:** `Program.cs` (add `AddRateLimiter` with per-IP fixed-window policies on `/login`, `/forgot-password`, `/register`); `IJwtTokenService.cs` (issue `kid` header, support key ring, asymmetric `RS256`); `.env.example` and `appsettings.json` across all services (per-service `Audience` and per-service signing key containers).

**Suggested fix:**
1. Add `AddRateLimiter` with per-IP fixed-window policies (`/login` 5/min/IP, `/forgot-password` 3/min/IP, `/register` 5/hour/IP).
2. Move to asymmetric `RS256`: autheo holds the private key; every verifier holds only the public key. Add per-service audience and validate accordingly.
3. Store the JWT signing private key as a mounted Kubernetes Secret / Docker Swarm secret, not in an env var.

---

## Support — NEW Issues

### 23. Anonymous Ticket Submission + No Multi-Tenancy + Critical Field-Mismatch Bug

> **GitHub Issue:** [tsqr-support#5](https://github.com/lstbob/tsqr-support/issues/5) (P0, 5 SP)

**Severity:** Critical — three Critical bugs.

**Evidence (a) — Anonymous ticket submission; Update has no role check:** `Controllers/Controllers.cs:61-82`:
```csharp
[HttpPost]
[AllowAnonymous]                                                          // anyone can submit
public async Task<IActionResult> Submit(SubmitTicketRequest req, ...) { ... }

[HttpPatch("{id}")]
[Authorize]                                                                // any authenticated user can resolve ANY ticket
public async Task<IActionResult> Update(int id, UpdateTicketStatusRequest req, ...) { ... }
```

Same shape as Communities #7 + Soup-Kitchen #11. The only claim read is `sub`. Authorization has no domain model ("who can resolve a ticket?", "who can assign?").

**Evidence (b) — Not multi-tenant at all:** `infra/init.sql:30-42`:
```sql
CREATE TABLE IF NOT EXISTS Tickets (
    Id                SERIAL PRIMARY KEY,
    Subject           VARCHAR(500) NOT NULL,
    Category          INTEGER NOT NULL,
    SubmitterMemberId UUID NULL,                  -- no CommunityId; UUID instead of int MemberId
    ...
);
```

Every other context scopes by `CommunityId`; support is the exception. `GET /api/tickets` returns all communities' tickets. Type mismatch: support uses UUID `SubmitterMemberId` while tool-lib/soup-kitchen use `int MemberId` carrying `CommunityId`-bound identity — breaks any future cross-context join.

**Evidence (c) — Critical field-mismatch bug:** `Commands/Commands.cs:5,19-20`, `Controllers/Controllers.cs:69`, `ITicketsRepository.cs:5`:
```csharp
public record SubmitTicketCommand(string Subject, int Category, string Description,
                                  string? SubmitDate, string? SubmitterMemberId);
// handler:
var id = await tickets.CreateTicketAsync(cmd.Subject.Trim(), cmd.Category, cmd.Description.Trim(),
    cmd.SubmitDate,    // <-- positional argument passes SubmitDate as submitterEmail
    cmd.SubmitterMemberId, null);
// ITicketsRepository:
Task<int> CreateTicketAsync(string subject, int category, string description,
    string? submitterEmail, string? submitterMemberId, string? assignedTo);   // <-- positional mismatch
```

The 4th positional argument of `SubmitTicketCommand` is named `SubmitDate` but is typed `string?` and is passed into the repository's `submitterEmail` parameter. Every authenticated-user ticket has `SubmitterEmail = NULL`; the controller never populates it anyway. This is the textbook symptom of primitive obsession and a missing aggregate: with a `Ticket` aggregate the compiler would refuse the mismatch.

**Affected Files:** `Controllers.cs` (remove `[AllowAnonymous]` from `Submit`, add `[Authorize(Policy="SupportStaff")]` to `Update`); `init.sql` (add `CommunityId INTEGER NOT NULL`); all read queries (add `WHERE CommunityId = @CommunityId`); `SubmitTicketCommand` + `ITicketsRepository.CreateTicketAsync` (rename `SubmitDate` → `SubmitterEmail`, have the controller populate it from the JWT claim); long-term: introduce a `Ticket` aggregate replacing positional-primitive repo methods.

**Suggested fix:**
1. Authentication: deny anonymous; enforce staff policy on resolve/assign.
2. Add `CommunityId` to `Tickets` and `Incidents`; scope all queries.
3. Rename the command field `SubmitDate` → `SubmitterEmail`; controller populates from JWT.
4. Long-term: model a `Ticket` aggregate to make the field-mismatch impossible (same fix shape as Communities #10 for VOs).

---

### 24. Ticket State Machine Unenforced + Dead Incident Aggregate + No Comments/Attachments + Hardcoded LIMIT 50

> **GitHub Issue:** [tsqr-support#6](https://github.com/lstbob/tsqr-support/issues/6) (P1, 8 SP)

**Severity:** High — multiple cohesive gaps in one refactor.

**Evidence (a) — Ticket state machine unenforced:** `TicketsRepository.cs:27-37`:
```csharp
@"UPDATE Tickets
     SET Status = @Status,
         AssignedTo = COALESCE(@AssignedTo, AssignedTo),
         ResolvedAt = CASE WHEN @Status = 3 THEN now() ELSE ResolvedAt END
   WHERE Id = @Id"
```

`Resolved → Open` is silently allowed. The SQL `CASE` only sets `ResolvedAt` when transitioning *to* Resolved; **when reopening a Resolved ticket it leaves the old `ResolvedAt` in place**, producing inconsistent state (`Status=Open` + non-null `ResolvedAt`). The audit's §14 stated a state machine of `New / Assigned / Resolved / Closed / Reopened`; the actual enum has only `Open / InProgress / Resolved` (no `Closed`, no `Reopened`).

`COALESCE(@AssignedTo, AssignedTo)` cannot distinguish "preserve assignment" from "clear assignment" — an agent cannot unassign a ticket (primitive-obsession bug).

**Evidence (b) — Incident aggregate effectively dead:** `ISupportQueries.cs:7` has only `ListIncidentsAsync` (read-only); no command handler, no repo write path. `Enums.cs:17-22` defines `IncidentStatus { Investigating=1, Ongoing=2, Resolved=3 }` but **nothing ever instantiates Investigating or Ongoing** — seed data inserts everything as Resolved. There is no way for an operator to declare "Notification Service is Down — investigating," so the public status page is permanently stale.

**Evidence (c) — No Comments / Attachments aggregation:** The only `ContactMessages` flow (`Commands.cs:60-61`) creates the ticket *and* the initial message in the same tx; there is no endpoint to add follow-up messages to an existing ticket, no `Comments` table, no attachment table.

**Evidence (d) — Hardcoded `LIMIT 50`:** `SupportQueries.cs:35-37`:
```csharp
if (status is > 0) sql += " WHERE Status = @Status";
else if (recent)   sql += " WHERE Status IN (1, 2)";
sql += " ORDER BY ReportedAt DESC LIMIT 50";    // page 2+ impossible
```
Only `ListTicketsAsync` is bounded, but with no `OFFSET`. All other support list queries (`ListIncidentsAsync`, `ListAppsAsync`, `ListServiceStatusesAsync`) are unbounded. Older tickets are unreachable via the API.

**Evidence (e) — No audit columns:** `Tickets` table has no `ResolvedBy`, `ClosedBy`, `LastModifiedAt`, `LastModifiedBy`. Support tickets are compliance-sensitive (who handled a member's access-issue ticket?); the operation log is gone at the moment of action, not just on delete.

**Affected Files:** New `Ticket` aggregate with `Resolve(actor)`, `Reopen(actor)`, `AssignTo(agent)`, `Unassign()`; `TicketsRepository` (clear `ResolvedAt` on reopen, distinct params for assign vs. unassign); new `Incident` aggregate with `Investigate()`, `MarkOngoing()`, `Resolve()` and event raises; new `Comment` / `Attachment` child entities; `ISupportQueries` (add `page`/`pageSize`); `init.sql` (add `ResolvedBy`, `ClosedBy`, `LastModifiedAt/By`, `CommunityId`, soft-delete columns).

**Suggested fix:**
1. Encapsulate state transitions on a `Ticket` aggregate; clear `ResolvedAt` on reopen; reject `Resolved → Open` via `Reopen` only.
2. Add a real `Incident` aggregate with `Investigate`/`MarkOngoing`/`Resolve` + handlers + DI registrations.
3. Add `Comment`/`Attachment` child entities inside `Ticket` with size/MIME invariants.
4. Add `page`/`pageSize` to `ListTicketsQuery` / `ListIncidentsQuery`; default page 1, size 50.
5. Add `ResolvedBy` / `ClosedBy` / `LastModifiedAt` / `LastModifiedBy` audit columns; soft-delete per v1 §25.

---

## Cross-Cutting (Shared Kernel, Deployment, Integration) — NEW Issues

### 25. Shared Kernel (tsqr-common) Is Anemic; DDD Base Classes Copy-Pasted in 4 Contexts

> **GitHub Issue:** [tsqr-common#2](https://github.com/lstbob/tsqr-common/issues/2) (P0, 13 SP)

**Severity:** Critical (DDD foundation).

**Evidence:** `find /mnt/data/dev/tsqr-common/src/TSQR.Common/` — only `Errors/`, `Results/`, `Interactors/IInteractor.cs`, `Extensions/`. No `Entity`, `ValueObject`, `IAggregateRoot`, `IDomainEvent`, `IId<T>`.

Each bounded context forks its own copies:

| Repo | File | Lines | Notes |
|---|---|---|---|
| tool-lib | `Domain/{Entity,ValueObject,IAggregateRoot}.cs` | 115+82+5 | Includes the `TestAsnc()` debug leak (audit §4) |
| communities | `Domain/{Entity,ValueObject,IAggregateRoot}.cs` | 22 | `Equals` omits `GetType` check |
| soup-kitchen | `Domain/{Entity,ValueObject,IAggregateRoot}.cs` | ~30 | Different `Equals` shape |
| autheo | `Domain/Entity.cs` | 11 | **No `DomainEvents` collection at all** — `Entity<TId>.Id` has a public setter, no `ValueObject` constraint on `TId`, no `Equals`/`GetHashCode` override |

```csharp
// autheo's Entity<TId> (11 lines, completely divergent):
public abstract class Entity<TId> where TId : notnull
{
    public TId Id { get; protected set; } = default!;
}
```

**Affected Files:** New `TSQR.Common.DomainKernel` namespace / project in tsqr-common; delete four homegrown `Entity.cs` / `ValueObject.cs` / `IAggregateRoot.cs` copies; reconcile `Equals` implementations.

**Suggested fix:** Move `Entity`, `Entity<TId>`, `ValueObject`, `IAggregateRoot`, `IDomainEvent` (and a typed `IId<T>` wrapper) into `TSQR.Common.DomainKernel`. Each microservice's `Domain` project references it. Delete the four homegrown copies. Add contract tests that compile against the shared types.

---

### 26. Bespoke Error/Error MW/Hosting Defaults in Every Service — Need `TSQR.Common.WebApi`

> **GitHub Issue:** [tsqr-common#3](https://github.com/lstbob/tsqr-common/issues/3) (P1, 5 SP)

**Severity:** High — collapses several cross-cutting findings (logging/tracing, error format, hosting defaults) into one cohesive infrastructure issue.

**Evidence — divergence in identity's `Program.cs:53-59`:**
- No `FallbackPolicy` (other 5 services have `RequireAuthenticatedUser()` fallback)
- No `UseHttpsRedirection`
- No `UseMiddleware<ExceptionHandlingMiddleware>`
- No `AddOpenApi`

Every other service carries the same custom `InvalidModelStateResponseFactory` returning a bespoke `ErrorResponse(code, message)` JSON shape — six copies of custom plumbing, no RFC 7807 ProblemDetails, no `traceId` correlation.

`grep -rln "OpenTelemetry|CorrelationId|ActivitySource|UseW3C|Serilog" /mnt/data/dev/tsqr-*` → zero matches. A single browser request to the gateway triggers four downstream calls (autheo, tool-lib, soup-kitchen, communities); when something fails 1-in-100 requests, the operator has no way to correlate the four log streams.

`RequireHttpsMetadata = builder.Environment.IsDevelopment() ? false : false` in `gateway/Program.cs:18-20` — dead-code ternary lying about configuration.

**Affected Files:** All six `Program.cs` files; all six `WebApi/Middleware/ExceptionHandlingMiddleware.cs` files; gateway healthcheck routing.

**Suggested fix:** Publish `TSQR.Common.WebApi` package exposing `builder.AddTsqrApiDefaults()`:
- JWT bearer with anonymous→deny fallback
- `ConfigureApiBehaviorOptions` for the `ErrorResponse` factory (or built-in `AddProblemDetails`)
- `UseHttpsRedirection`, `UseMiddleware<ExceptionHandlingMiddleware>`, OpenAPI in Development
- `AddOpenTelemetry().WithTracing(t => t.AddAspNetCore().AddHttpClient().AddSource("TSQR.*"))` + OTLP exporter
- `AddHealthChecks().AddNpgSql(...)` + `MapHealthChecks("/api/health")` and `/api/health/ready` filtered to "ready" tag

Each `Program.cs` becomes ~10 lines. Six divergent configurations collapse into one.

---

### 27. Deploy SQL Drifted from Local init.sql — `Loans.LateFeePerDay`/`RenewalCount`/`Policies` Table Missing

> **GitHub Issue:** [tsqr-deploy#2](https://github.com/lstbob/tsqr-deploy/issues/2) (P0, 2 SP)

**Severity:** Critical — the canonical `docker compose up` deployment of tool-lib breaks on first loan.

**Evidence:** `tsqr-deploy/infra/02-tool-library.sql:89-99`:
```sql
CREATE TABLE IF NOT EXISTS Loans (
   Id SERIAL PRIMARY KEY, MemberId INTEGER NOT NULL REFERENCES Members(Id),
   CheckoutDate TIMESTAMP NOT NULL, DueDate TIMESTAMP NOT NULL,
   ItemId INTEGER NOT NULL REFERENCES InventoryItems(Id), Status INTEGER NOT NULL,
   ReturnedDate, FineAccrued NUMERIC(10,2) NOT NULL DEFAULT 0
);
```
Missing: `LateFeePerDay`, `RenewalCount`, `CommunityId` columns. No `Policies` table either — yet `LoanMapping.cs` INSERTs include those columns and `PolicyMapping.cs` reads/writes the `Policies` table. First `Loans` INSERT will throw `Npgsql.PostgresException: 42703: column "LateFeePerDay" of relation "Loans" does not exist`.

Local `tool-lib/src/.../docker/init.sql` matches the code; the shared `infra/02-tool-library.sql` is stale. The two have drifted silently.

`08-tool-library-community.sql` ADDs `CommunityId` but there's no `10-tool-library-loans-latefee.sql` to retrofit `LateFeePerDay`/`RenewalCount`, and no `11-tool-library-policies.sql` to create the `Policies` table.

**Affected Files:** `tsqr-deploy/infra/02-tool-library.sql` (reconcile with `tool-lib/src/.../docker/init.sql`); or add new ALTER scripts `10-…`, `11-…`.

**Suggested fix:** As an immediate patch, reconcile `02-tool-library.sql` against `tool-lib/src/.../docker/init.sql`. Long-term: this gap is the immediate symptom of #28 (no migration framework).

---

### 28. No DB Migration Framework — `ALTER TABLE` Without `IF NOT EXISTS`, No Version Table

> **GitHub Issue:** [tsqr-deploy#3](https://github.com/lstbob/tsqr-deploy/issues/3) (P1, 8 SP)

**Severity:** High — #27 is the immediate visible symptom; the root cause is no migration story.

**Evidence:** `docker-compose.yml:13-21` mounts `infra/*.sql` via `docker-entrypoint-initdb.d/`, which only runs when the data directory is empty (first `up -v`).

`08-tool-library-community.sql:6-11` and `09-soup-kitchen-community.sql:4-8` use bare `ALTER TABLE … ADD COLUMN` (no `IF NOT EXISTS`), so re-running fails — schema is "set once, never modified". There's no ordered runner, no idempotency, no rollback, no `_SchemaVersions` table.

The only way to add a column to a production DB is `docker compose down -v && up --build` — wiping user data.

**Affected Files:** New `Migrations/` directory per service (or shared `tsqr-deploy/migrations/`); add DbUp or FluentMigrator; each service's `Program.cs` runs `DbUp.Builder.Build().Upgrade()` against its connection string on startup; the init SQL becomes seed-only on first deploy.

**Suggested fix:** Adopt DbUp (or FluentMigrator). Each service references a `Migrations/` directory with `YYYYMMDDHHmm_*.sql` files ordered by name. On startup, the service runs `Upgrade()` against its connection string, recording applied scripts in a `_SchemaVersions` table.

---

### 29. `mbr` JWT Claim Undocumented, Unconsumed, Type-Mismatched Across Contexts

> **GitHub Issue:** [tsqr-autheo#6](https://github.com/lstbob/tsqr-autheo/issues/6) (P1, 5 SP)

**Severity:** High — distributed-system landmine.

**Evidence:** `IJwtTokenService.cs:45`:
```csharp
new Claim("mbr", user.MemberId?.ToString() ?? ""),
```

`Autheo.Domain/Aggregates/UserAggregate/User.cs:9`:
```csharp
public Guid? MemberId { get; private set; }   // so mbr is empty string OR a Guid string
```

`tsqr-deploy/infra/03-autheo.sql:10` — `MemberId UUID NULL`.

But `tool-lib/Domain/.../MemberId.cs` — `MemberId` wraps **`int`**, not `Guid`. `soup-kitchen/Domain/.../VolunteerAggregate/Volunteer.cs:6` — `public int MemberId { get; private set; }`.

`grep -rn '"mbr"\|FindFirstValue'` across all TSQR `src/` — **no consumer reads the `mbr` claim at all** (only the issuer writes it). Identity, tool-lib, soup-kitchen, communities, support all read `sub` and `ClaimTypes.NameIdentifier` only.

The audit's "Good #6" claim that the `mbr` claim links `User` to `Member` is fictional in production:
- No consumer reads the claim → enforced link = 0
- Value types are incompatible (`Guid?` vs `int`)
- The contract is nowhere documented
- The empty-string fallback (`?? ""`) lets unauthenticated-looking claims slip through

A future contributor will assume `mbr` is the member identifier and consume garbage.

**Affected Files:** `IJwtTokenService.cs` (drop `?? ""`; conditionally add the claim only when `MemberId.HasValue`); new `TSQR.Common.Identity` with a typed `MemberId` shared value object (resolves type mismatch with tool-lib/soup-kitchen); autheo migration `ALTER TABLE Users ALTER COLUMN MemberId TYPE INTEGER USING NULL` (or — preferred — adopt `MemberId == User.Id == Sub` single identity and eliminate the claim); gateway `Program.cs` (add `RequireClaim("mbr")` policy for member-scoped routes); tsqr-domain docs (document the contract).

**Suggested fix:** Either (a) decide `MemberId == User.Id == Sub` (single identity, eliminate the claim), or (b) keep the `User/Member` split but standardize on a typed `MemberId` in `TSQR.Common`, document the claim contract (`mbr` is a string-encoded `MemberId`, required when role ∈ {Regular, Repairman, Coordinator, Admin}), and enforce its presence with an authorization policy at the gateway AND in service-level `[Authorize(Policy="HasMember")]` middleware. Add a contract test in a new `tsqr-contracts` repo.

---

## Summary: Verdict (Round 2)

The first round audit identified 28 + 4 issues; #33, #34, #35 closed nine entries and introduced three regressions (queue direction (#1), reservation breakage (#1 compounding), audit-trail fiction (#20/#21)). The remainder of the original list is still pending.

**Tool Library** remains the most rigorously-modeled bounded context, but the #34 cascade-introduced regressions (Critical #1) and the #35 deferred surface (RenewLoanCommand, Member fines gating) need attention before more domain events are added.

**Communities, Soup Kitchen, Identity, and Support** share three structural gaps that the v1 audit only partially exposed:
- **Multi-tenancy is decorative**: `CommunityId` is declared on every entity but enforced nowhere in queries; soup-kitchen reads return cross-tenant PII (#11); support isn't multi-tenant at all (#23).
- **Authorization is binary**: `[AllowAnonymous]` on the open internet for sensitive endpoints (communities QuickRegister, soup-kitchen reads, support ticket submission, identity avatar fetch). Two-tier (auth + role/ownership) is missing across all four contexts.
- **Domain behavior is fictional**: methods exist on aggregates but no command/handler/endpoint ever invokes them (communities Suspend/Archive, soup-kitchen Guest/Volunteer/Event transitions + Donation behavior, identity Lock invariant). Audit §9 and §2 catalogued the pattern; round 2 enumerates ~10 new instances.

**Autheo** has the most security-critical Critical bugs:
- `/register` self-admin exploit (#20)
- Non-functional password reset (#20)
- Refresh-token reuse-detection fictional both via non-persistence (#20) AND concurrent rotation fork (#21)
- Single shared symmetric key across 7 services with no per-service audience (#22)

**Cross-cutting** issues are where the highest-leverage fixes live. Resolving #25 (shared DDD base classes), #26 (`TSQR.Common.WebApi` shared hosting), #27/#28 (migration framework + the immediate SQL drift fix), and #29 (JWT claim contract) collapses dozens of per-context findings in one cohesive refactor.

### Recommended Sprint order

**Critical (must-do before any production usage):**
1. autheo#3 (#20) — self-Admin + password reset + RevokedReasonIsReuse
2. autheo#4 (#21) — refresh concurrency + audit preservation
3. tool-lib#42 (#1) — queue regression + reservation breakage
4. soup-kitchen#6 (#11) — multi-tenancy + PII exposure
5. tsqr-deploy#2 (#27) — immediate SQL drift fix
6. communities#6 (#7) — anonymous QuickRegister
7. support#5 (#23) — anonymous submission + field-mismatch bug
8. identity#5 (#15) — dead email-change feature

**High (next 2-3 sprints):**
9. tsqr-common#2 (#25) — shared kernel base classes
10. autheo#5 (#22) — rate limiting + asymmetric JWT
11. tool-lib#43-46 + soup-kitchen#7-8 + identity#6-8 (High bucket)

**Medium (background):**
- tsqr-common#3, tsqr-deploy#3, autheo#6, communities#9, soup-kitchen#9, identity#9, tool-lib#44/45, support#6

### Issue index

| # | Issue | Repo | Priority | Estimate (SP) | Sprint |
|---|-------|------|----------|---------------|--------|
| 1 | [tsqr-tool-lib#42](https://github.com/lstbob/tsqr-tool-lib/issues/42) | tool-lib | P0 | 5 | Sprint 20 |
| 2 | [tsqr-tool-lib#43](https://github.com/lstbob/tsqr-tool-lib/issues/43) | tool-lib | P1 | 5 | Sprint 22 |
| 3 | [tsqr-tool-lib#44](https://github.com/lstbob/tsqr-tool-lib/issues/44) | tool-lib | P1 | 3 | Sprint 22 |
| 4 | [tsqr-tool-lib#45](https://github.com/lstbob/tsqr-tool-lib/issues/45) | tool-lib | P1 | 3 | Sprint 23 |
| 5 | [tsqr-tool-lib#46](https://github.com/lstbob/tsqr-tool-lib/issues/46) | tool-lib | P1 | 5 | Sprint 23 |
| 6 | [tsqr-tool-lib#47](https://github.com/lstbob/tsqr-tool-lib/issues/47) | tool-lib | P1 | 5 | Sprint 24 |
| 7 | [tsqr-communities#6](https://github.com/lstbob/tsqr-communities/issues/6) | communities | P0 | 3 | Sprint 21 |
| 8 | [tsqr-communities#7](https://github.com/lstbob/tsqr-communities/issues/7) | communities | P1 | 8 | Sprint 22 |
| 9 | [tsqr-communities#8](https://github.com/lstbob/tsqr-communities/issues/8) | communities | P1 | 5 | Sprint 23 |
| 10 | [tsqr-communities#9](https://github.com/lstbob/tsqr-communities/issues/9) | communities | P2 | 5 | Sprint 24 |
| 11 | [tsqr-soup-kitchen#6](https://github.com/lstbob/tsqr-soup-kitchen/issues/6) | soup-kitchen | P0 | 8 | Sprint 21 |
| 12 | [tsqr-soup-kitchen#7](https://github.com/lstbob/tsqr-soup-kitchen/issues/7) | soup-kitchen | P1 | 8 | Sprint 23 |
| 13 | [tsqr-soup-kitchen#8](https://github.com/lstbob/tsqr-soup-kitchen/issues/8) | soup-kitchen | P1 | 8 | Sprint 24 |
| 14 | [tsqr-soup-kitchen#9](https://github.com/lstbob/tsqr-soup-kitchen/issues/9) | soup-kitchen | P2 | 5 | Sprint 25 |
| 15 | [tsqr-identity#5](https://github.com/lstbob/tsqr-identity/issues/5) | identity | P0 | 5 | Sprint 21 |
| 16 | [tsqr-identity#6](https://github.com/lstbob/tsqr-identity/issues/6) | identity | P1 | 8 | Sprint 23 |
| 17 | [tsqr-identity#7](https://github.com/lstbob/tsqr-identity/issues/7) | identity | P1 | 5 | Sprint 24 |
| 18 | [tsqr-identity#8](https://github.com/lstbob/tsqr-identity/issues/8) | identity | P1 | 3 | Sprint 22 |
| 19 | [tsqr-identity#9](https://github.com/lstbob/tsqr-identity/issues/9) | identity | P2 | 5 | Sprint 25 |
| 20 | [tsqr-autheo#3](https://github.com/lstbob/tsqr-autheo/issues/3) | autheo | P0 | 5 | Sprint 20 |
| 21 | [tsqr-autheo#4](https://github.com/lstbob/tsqr-autheo/issues/4) | autheo | P0 | 5 | Sprint 20 |
| 22 | [tsqr-autheo#5](https://github.com/lstbob/tsqr-autheo/issues/5) | autheo | P1 | 8 | Sprint 22 |
| 23 | [tsqr-support#5](https://github.com/lstbob/tsqr-support/issues/5) | support | P0 | 5 | Sprint 21 |
| 24 | [tsqr-support#6](https://github.com/lstbob/tsqr-support/issues/6) | support | P1 | 8 | Sprint 24 |
| 25 | [tsqr-common#2](https://github.com/lstbob/tsqr-common/issues/2) | common | P0 | 13 | Sprint 22 |
| 26 | [tsqr-common#3](https://github.com/lstbob/tsqr-common/issues/3) | common | P1 | 5 | Sprint 23 |
| 27 | [tsqr-deploy#2](https://github.com/lstbob/tsqr-deploy/issues/2) | deploy | P0 | 2 | Sprint 21 |
| 28 | [tsqr-deploy#3](https://github.com/lstbob/tsqr-deploy/issues/3) | deploy | P1 | 8 | Sprint 23 |
| 29 | [tsqr-autheo#6](https://github.com/lstbob/tsqr-autheo/issues/6) | autheo | P1 | 5 | Sprint 24 |

**Total:** 29 issues across 8 repositories; 154 story points. P0 = 53 SP across 10 issues; P1 = 86 SP across 14 issues; P2 = 15 SP across 5 issues.