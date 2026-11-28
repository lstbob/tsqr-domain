# TownSquare (TSQR): the domain model, from a DDD point of view

> **Scope:** how the TownSquare platform is modelled using Domain-Driven Design. It covers subdomains, bounded contexts, the context map, aggregates, invariants, state machines, domain events, consistency and the ubiquitous language.
> It describes the model **as it exists in code on 2026-11-19**. Where the code and the intended model differ, this document says so in *Model tension* notes.
> For the business view see [`townsquare.md`](./townsquare.md). For defects see [`thegoodandthebad-v2.md`](./thegoodandthebad-v2.md).

---

## 1. Domain vision statement

> TownSquare lets a neighbourhood **pool what it owns and organise how it helps itself**.
>
> The heart of the domain is **fair, trusted circulation of shared physical goods**: who may borrow what, for how long, in which order, and what happens when things come back late, broken or not at all.
>
> Around that heart sit the **organisation of mutual-aid events** (community kitchens) and the **registry of the communities themselves**, all on a shared identity and permission backbone.

---

## 2. Subdomains

| Subdomain | Type | Why | Bounded context |
|---|---|---|---|
| **Shared-goods circulation** (lending, queues, policies, fines, repair, member eligibility) | **Core** | This is what sets TownSquare apart. Its rules (trust, fairness, scarcity) are what make neighbours willing to share. | Tool Library (`tsqr-tool-lib`) |
| **Mutual-aid event coordination** (meal events, guests, volunteers, donations) | **Supporting** | Specific to the mission, but structurally simple: plan, register, capacity. | Soup Kitchen (`tsqr-soup-kitchen`) |
| **Community and geography registry** | **Supporting** | Needed to anchor every community (tenant) to a place, but it holds no rich rules. | Communities (`tsqr-communities`) |
| **Authentication and credentials** | **Generic** | Could be bought (Keycloak, Entra ID and so on). | Autheo (`tsqr-autheo`) |
| **User profile and administration** (profiles, suspension, roles, permissions) | **Generic / supporting** | Standard IAM administration. | Identity (`tsqr-identity`) |
| **Customer support and service status** | **Generic** | Standard help-desk functionality. | Support (`tsqr-support`) |
| **Learning** (spaced retrieval, recall practice, calibration) | **Supporting** | Helps members learn and remember what they practise. The rules are small: cards, Leitner boxes and due dates. | Learning (`tsqr-learning`) |

Most of the modelling effort has gone, correctly, into the core subdomain.

---

## 3. Bounded contexts and context map

```
                          ┌──────────────────────────────────────┐
                          │       SHARED KERNEL: tsqr-common     │
                          │ Result/Error · IDomainEvent ·        │
                          │ DomainEventDispatcher · Outbox ·     │
                          │ Permissions/RolePermissions ·        │
                          │ WebApi defaults (JWT, problem        │
                          │ details, OTel)                       │
                          └──────────────────┬───────────────────┘
                     (all .NET contexts depend on it; the gateway does not)
                                             │
 ┌──────────────┐  OHS + Published Language: JWT (sub, role, permission[], community_id)
 │   AUTHEO     │──────────────────────────────────────────────────────────────┐
 │ (Generic)    │                                                              │
 │ User,        │◄─── Customer/Supplier ──── TOOL LIBRARY answers              │
 │ RefreshToken │     (sync HTTP, internal   "which community does this       │
 └──────▲───────┘      API key)              subject belong to?"               │
        │ internal change-email call                     │                     │
        │ (identity is the customer)                     │                     │
 ┌──────┴───────┐                     ┌──────────────────▼─────┐               │
 │  IDENTITY    │   shares `sub`      │   TOOL LIBRARY (Core)  │◄──────────────┤ conformist
 │ UserProfile, │   as its key        │ Tool · InventoryItem · │   to the JWT  │
 │ Role         │                     │ Loan · Reservation ·   │               │
 └──────────────┘                     │ Member · Policy        │               │
                                      │ (+ MemberAccounts ACL) │               │
                                      └────────────────────────┘               │
                                                                               │
 ┌──────────────┐                     ┌────────────────────────┐               │
 │ COMMUNITIES  │  CommunityId is an  │    SOUP KITCHEN        │◄──────────────┤
 │ Community,   │  opaque integer in  │ Event (+ Meal, Guest,  │               │
 │ Location     │  all other contexts │ Volunteer, Donation)   │               │
 └──────────────┘  (no live lookup)   └────────────────────────┘               │
                                                                               │
 ┌──────────────┐                                                              │
 │  SUPPORT     │◄─────────────────────────────────────────────────────────────┘
 │ Ticket,      │
 │ Incident     │
 └──────────────┘

 ┌──────────────┐
 │  LEARNING    │  Deck, Card, Review
 │ (supporting) │  conformist to Autheo's JWT
 └──────────────┘

 GATEWAY (YARP): edge proxy and BFF aggregator; no domain logic.
```

### Relationships

| Upstream → downstream | DDD pattern | Mechanism |
|---|---|---|
| Autheo → all contexts | **Open Host Service + Published Language** | Signed JWT. The contract is in `tsqr-autheo/docs/JWT-CONTRACT.md` and covered by a contract test. |
| All contexts ← Autheo | **Conformist** | Downstream contexts take `sub`, `role` and `permission` claims as given, and authorise against the shared permission catalogue. |
| Autheo → Learning | **Conformist** | Learning takes the `sub` and `community_id` claims as given. Learning has no other dependency on another context. |
| Tool Library → Autheo | **Customer/Supplier** (Tool Library is the supplier) | `GET /internal/members/community?subject=` with an `X-Internal-Api-Key` header. Autheo uses it to put `community_id` into the token. Tool Library is the **source of truth for community membership**. |
| Autheo ← Identity | **Customer/Supplier** (Autheo is the supplier) | Identity runs the email-change confirmation, then calls Autheo's internal `change-email`. Autheo owns credentials and email. |
| Communities → others | **Separate Ways (implicit reference)** | `CommunityId` appears in every tenant-scoped aggregate as an opaque `int`, but nothing checks it against Communities at runtime. |
| Tool Library internal: auth subject ↔ Member | **Anti-Corruption Layer** | The `MemberAccounts` table maps the external `sub` (UUID) to the local `MemberId` (int), through `CurrentMemberResolver`. |
| All contexts | **Shared Kernel** | `tsqr-common` (NuGet `TSQR.Common` 1.3.0 and `TSQR.Common.WebApi` 1.2.0). |

### Identity across contexts: the most important integration rule

| Concept | Context | Type | Meaning |
|---|---|---|---|
| `User.Id` = `sub` | Autheo (owner), Identity (key) | UUID | **Canonical cross-service identity** |
| `Member.Id` | Tool Library | int | The local identity of a borrower. It is **not** `sub`. |
| `MemberAccount(AuthSubject → MemberId)` | Tool Library | mapping | ACL that translates between the two |
| `Volunteer.MemberId` | Soup Kitchen | int | Local reference. *Model tension: it is not linked to `sub` or checked against the caller.* |
| `community_id` claim | All tenant contexts | int | Active tenant. It **must** come from the signed token, and services must **never** fall back to community 1 when it is missing. |
| `mbr` claim | — | — | **Retired** in Nov 2026. It was never consumed and had the wrong type. |

**Rule: users are not members.** A *User* is someone who can log in. A *Member* is someone a community trusts to borrow. Becoming a member is a domain process (request, verification, approval), not something that comes with an account.

---

## 4. Tool Library context (core)

### 4.1 Aggregates

```
Tool (AR) ──────────────── catalogue record: "what this tool is"
  • Model, Description, ToolType, AmortizationRate, Metadata
  • Manufacturer (VO: Id, Name, CommunityId)
  • ScarcityByLocation : Location → ScarcityLevel
  • CommunityId

InventoryItem (AR) ─────── one physical copy: "which drill"
  • SerialNumber, Status, Condition, CurrentHolderId, OriginalOwner
  • LoanCount, TotalUsageTime
  └─ MaintenanceRecord (child entity, at most one open)

Loan (AR) ─────────────── one borrowing of one copy by one member
  • MemberId, ItemId, CheckoutDate, DueDate, Status
  • LateFeePerDay (copied from Policy at checkout), RenewalCount, Fine

Reservation (AR) ──────── one place in an item's queue
  • MemberId, ItemId, QueuePosition, ReservationDate, ExpiryDate, Status

Member (AR) ───────────── a borrower in one community
  • Status, MembershipType, AccessRequestStatus, Identification (VO)
  • IsVerified, outstanding fines, CommunityId
  └─ MembershipRecord (plain class)

Policy (AR) ───────────── lending rules for (ToolType, Location?)
  • LateFeePerDay, MaxLoanDurationDays, MaxRenewalCount, MaxLoanReservationDays
```

**Key modelling decision: the record/copy split.** `Tool` and `InventoryItem` are separate aggregates. This is the same split public library systems such as Koha use between bibliographic record and item. Queues and scarcity relate to the *kind* of thing, while loans and repairs happen to a *specific copy*.

**Why Loan and Reservation are separate aggregates and not children of InventoryItem.** Each has its own lifecycle and identity. Both are queried independently by member. Each changes on its own schedule: renewals, queue shifts, expiry. Keeping them out of InventoryItem keeps that aggregate small and limits concurrency conflicts.

### 4.2 Value objects
- Typed IDs for every aggregate (int-wrapping): `ToolId`, `InventoryItemId`, `LoanId`, `ReservationId`, `MemberId`, `PolicyId`, and others.
- `Manufacturer` (it was once an entity; it was demoted to a value object in Nov 2026).
- `Identification` (type and reference), used for proof of identity when requesting access.
- `Address` (defined but not used; `Member.Address` is a string).

### 4.3 State machines (enforced inside aggregate methods, which return `DomainError`)

**InventoryItem.Status**
```
              Reserve()                Loan()
 Available ───────────► Reserved ───────────► Loaned
    ▲  │                                        │
    │  │ Loan()                                 │ Return(condition)
    │  └────────────────────────────────────────┤
    │◄──────────────────────────────────────────┘
    │
    │ CompleteRepair()        MarkForRepair()
    └──────────── UnderMaintenance ◄──────────── (any state with no open repair)

 any state except Lost ── MarkAsLost() ──► Lost   (terminal; the holder is cleared)
```

**MaintenanceRecord:** Reported → InProgress (`StartWork`) → Completed (`Complete`). `CompleteRepair` on the item starts the work automatically if needed.

**Loan**
```
 Active ── Renew() [RenewalCount < policy.MaxRenewalCount] ──► Active (due date extended)
 Active ── EndLoan(date ≤ DueDate) ──► Returned
 Active ── EndLoan(date > DueDate) ──► Overdue  + Fine = ceil(days late) × LateFeePerDay
                                                + LoanOverdueDomainEvent
```

**Reservation**
```
 Pending ─ConfirmPickup()─► Confirmed ─Activate()─► Active ─Complete()─► Completed
    └───────────────┬──────────────┬─────────────────┘
                    └── Cancel() ──► Cancelled        (from any non-terminal state)
 MoveUpInQueue(): QueuePosition-- (never below 1)
```

**Member**
```
 AccessRequest:  NotSet/Denied ─RequestAccess(identification)─► Pending
                 Pending ─ApproveAccess()─► Approved (also calls Verify → Active)
                 Pending ─DenyAccess()──► Denied

 Status:         Active ⇄ Suspended (Suspend / Reinstate: only from Suspended)
                 any ─Ban()─► Banned   (intended to be terminal)
```
*Model tension:* `Suspend()` does not guard against `Banned`. Banned → Suspended → Active is therefore possible, and the "Banned is terminal" invariant leaks.

### 4.4 Invariants

| Invariant | Where |
|---|---|
| A copy can only be loaned when it is not loaned, under maintenance or lost | `InventoryItem.Loan` |
| A copy has at most one open repair | `InventoryItem.MarkForRepair` |
| A borrower must be verified, Active, owe ≤ 50 in fines, and have no overdue active loan | `Member.EnsureEligibleToBorrow` |
| A member cannot hold two active loans on the same copy, and cannot take a copy held by someone else's confirmed or active reservation | `LoanToolCommandHandler` (*model tension: application-layer rule*) |
| Checkout date is not in the future, due date is not in the past, fee ≥ 0 | `Loan.Create` |
| Renewals are capped by the policy's `MaxRenewalCount` | `Loan.Renew` |
| Lending terms are **copied into the loan at checkout** (fee per day, period) | `Loan.Create(…, policy values)` |
| Policy ranges: loan 1–7 days, renewals 1–2, reservation 1–14 days, fee ≥ 0 | `Policy.ValidatePolicyFields` |
| Queue position > 0; next = count + 1; the next in line is the lowest Pending/Confirmed position | `Reservation`, `ReservationQueueService` |
| A subject can be linked to only one member | `LinkMemberAccountCommand` |
| Callers may only act on their own reservations and loans unless they hold a staff permission | `AccessGuard.EnsureOwns` / `EnsureActingFor` |

### 4.5 Domain services
- **`FineService`** calculates the overdue fine. It is called from `Loan`.
- **`ReservationQueueService`** handles queue arithmetic: next position, next in line, shifting after a cancellation.
- **`MemberVerificationService`** exists but is not used.

### 4.6 Domain events and how they connect aggregates

| Event | Raised by | Reaction (handler) |
|---|---|---|
| `InventoryItemRequiredEvent` | Register-tool use case | Creates the first InventoryItem for a newly registered Tool |
| `LoanCreatedDomainEvent` | `Loan.Create` | `InventoryItem.Loan(member)` |
| `ReservationCreatedDomainEvent` | `Reservation.Create` | Reserves the item if this is position 1 and the item is Available |
| `ReservationCancelledEvent` | `Reservation.Cancel` | Shifts the queue and notifies the next in line |
| `NextInLineNotificationEvent` | `Reservation.NotifyNextInLine` | Reserves the item for that member if it is Available |
| `PickupReminderEvent` | `Reservation.ConfirmPickup` | Reserves the item if it is Available |
| `ToolReturnedEvent` | `InventoryItem.Return` | Notifies the next in line |
| `ToolMarkedForRepairEvent` | `InventoryItem.MarkForRepair` | Cancels pending and confirmed reservations, which in turn triggers queue shifts |
| `ToolRegisteredEvent`, `ItemLoanedDomainEvent`, `LoanOverdueDomainEvent`, `ReservationConfirmedEvent`, `ToolRepairedEvent`, `MemberVerifiedEvent` | the respective aggregates | Audit log only |
| `ReturnReminderEvent` | *never raised* | No-op |

*Model tension: "notification" events change state but do not notify anyone.* No notification context or transport exists yet. A future **Notifications** context would subscribe to these events.

*Model tension: Return does not end the Loan.* `InventoryItem.Return` raises `ToolReturnedEvent`, but nothing closes the matching `Loan` or calculates its fine. This is the most important missing link in the core model (see Round 3, R3-1).

### 4.7 Consistency model
- **One aggregate per command, with side effects through events.** Each use case loads and saves one root and raises events. `DomainEventOrchestrator` dispatches handlers **before commit and inside the same transaction**, in passes up to depth 16, collecting any aggregates the handlers modify. If any handler fails, the whole transaction rolls back. This is the eShopOnContainers "Option A" pattern. It is still *atomic across several aggregates*, which is a deliberate choice for a single-database context.
- **Post-commit handlers** run after the commit on a best-effort basis.
- **Integration events** go through a transactional **outbox** (`OutboxMessages`, polled every 5 seconds). The transport is a logging placeholder and no producer is wired up yet. Cross-context communication is therefore synchronous HTTP only.
- **Optimistic concurrency** uses PostgreSQL `xmin` as `RowVersion` on every update and delete, and returns HTTP 409 on conflict.

---

## 5. Soup Kitchen context (supporting)

### 5.1 Aggregate
```
Event (AR) ── Name, Description, EventDate, Location (free text), Status, MaxGuests, CommunityId
  ├─ Meal        Name, Category, QuantityNeeded (>0), QuantityPrepared (≥0)
  ├─ Guest       Name, ContactInfo?, GuestCount (>0), Status
  ├─ Volunteer   MemberId (>0), Role, HoursScheduled (≥0), Status
  └─ Donation    DonorName, Item, Quantity (>0), DonationDate
```
In Nov 2026 Meal, Guest, Volunteer and Donation were changed from aggregate roots into **child entities of Event**. Their consistency rules (same community, unique volunteers, and later capacity) all belong to the event.

### 5.2 Invariants enforced today
- `Event.Create`: required fields, the date is not in the past, `MaxGuests > 0`, a positive community ID.
- `Add*` children: the event must already be persisted, and the child's community must equal the event's community.
- A volunteer cannot be signed up twice for the same event (unless cancelled). This is also enforced by a partial unique index in the database.

### 5.3 State machines (declared, **not yet guarded**)
- `EventStatus`: Planned → Active → Completed, or Cancelled.
- `GuestStatus` and `VolunteerStatus`: Pending → Confirmed, or Cancelled.

### 5.4 Model tensions (open issues soup-kitchen #6, #7, #8 and #22)
- **Capacity:** the sum of `GuestCount` is never compared with `MaxGuests`.
- **Closed events:** you can add guests and other children to Completed or Cancelled events.
- **Anemic update:** `UpdateDetails` sets any status (bypassing the transitions), does no validation, and **overwrites `CommunityId`**, which lets one tenant take over another tenant's event.
- **Unused events:** the domain events (`EventCreated`, `GuestRegistered`, `VolunteerSignedUp`, `DonationReceived`, `MealAdded`) are dispatched after commit but have no handlers.
- **Location:** `Event.Location` is free text and is not linked to the Communities `Location` aggregate.

---

## 6. Communities context (supporting)

### 6.1 Aggregates
```
Location (AR, keyed by NeighbourhoodId) ── a consistent geographic path
  ├─ Country       Name, CountryCode (VO: 2 ASCII letters, upper case)
  ├─ City          Name, CountryId
  └─ Neighbourhood Name, CityId
  invariant: city ∈ country; neighbourhood ∈ city

Community (AR) ── NeighbourhoodId, Name, Slug, Description, ContactEmail, ContactPhone, Status
  invariant: it references an existing neighbourhood; the slug is generated once and never changes
```
- **Slug:** Unicode-aware (accents stripped), unique among active communities (`UX_Communities_Slug_Active`).
- **`LocationRepository.EnsureAsync`** finds or creates the whole country, city and neighbourhood path, which makes it a factory for `Location`. Quick register uses it and commits the community and the location together in one unit of work.
- **Status:** PendingConfirmation → Active (`Confirm`); Suspended and Archived also exist. *Model tension:* the transitions are not guarded (open issue communities #7), and only `Confirm` can be reached through the API.
- There is **no membership model** in this context. Membership is owned by the Tool Library (`Member.CommunityId`). This is a notable allocation of responsibility: the Communities context defines what a community is, while the Tool Library defines who belongs to it.

---

## 7. Autheo context (generic: authentication)

- **User (AR):** Email, PasswordHash (PBKDF2-SHA256, 100k iterations), FullName, `RoleName` (Regular, Repairman, LocationCoordinator or Admin), IsActive, email confirmation, and `xmin` RowVersion. It has a validating `Create`. `ChangeEmail` is the only way to change the email address.
- **RefreshToken (AR):** stores only the SHA-256 hash.
  - `Rotate()` links each token to its replacement (`ReplacedByTokenHash`).
  - `Revoke(reuseDetected)` is idempotent and keeps the original audit fields.
  - **Reuse detection:** presenting a revoked token revokes every descendant in the chain.
  - Rotation is a conditional update, so two concurrent refreshes cannot fork the chain.
- **PasswordResetToken:** hashed, valid for 1 hour, single use. Earlier tokens are revoked when a new one is requested. The flow resists account enumeration.
- **Registration** always assigns the `Regular` role.
- **Published Language:** the JWT contract (see §3).

## 8. Identity context (generic: profile and administration)

- **UserProfile (AR, key = `sub`):**
  - names, avatar and bio;
  - audit fields (Created/Updated At/By);
  - **suspension** (`Suspend(until ≤ 365 days, reason, by)` and `Reinstate(by)`);
  - soft delete.
- **Role (AR):** a name plus a set of permission strings, stored in `roles_permissions`. It is assigned through `user_roles`.
- **PendingEmailChange:** a hashed token with a 24-hour lifetime. On confirmation, Identity calls Autheo.
- *Model tensions:*
  - Two IAM models exist side by side and do not feed each other:
    - Identity's roles and suspensions never reach Autheo's token issuing;
    - the token's permissions come from the static `RolePermissions` map in `tsqr-common`.
  - Profile updates create a new object and upsert it, so the aggregate is not reloaded.

## 9. Support context (generic)

- **Ticket (AR):** Subject, Category (Bug, Feature, Access, General), Priority (Low to Urgent), Description, SubmitterEmail and SubmitterMemberId (both optional), AssignedTo, CommunityId.
  ```
  Open ─Assign─► InProgress ─Resolve─► Resolved ─Close─► Closed (terminal)
                    ▲  (Assign also allowed from InProgress)   │
                    └──────────────── Reopen ──────────────────┘
  ```
  Events: `TicketCreated`, `TicketAssigned`, `TicketResolved`, `TicketReopened`, `TicketClosed`. Resolution time cannot be earlier than the report time.
- **Incident (AR):** Investigating → Ongoing (`Escalate`) → Resolved, with `UpdateFindings` allowed until resolved. Its events are handled after commit and only logged.
- **ServiceStatus** and **ContactMessage** are read and write records, not aggregates. A contact message also creates a General ticket.

---

## 10. Cross-cutting domain concepts

### 10.1 Multi-tenancy: *Community* as the tenant
Every tenant-scoped aggregate (Tool, InventoryItem, Loan, Reservation, Member, Event, Ticket, Incident) carries a `CommunityId`. The rules are:

1. the active tenant comes **only** from the signed `community_id` claim;
2. a missing claim results in **403**, never a default;
3. every read and write is filtered by tenant;
4. the tenant on an existing aggregate never changes.

How far each context follows these rules today:

| Context | State |
|---|---|
| Support | Filters by tenant, but a missing claim falls back to community 1 (breaks rule 2) |
| Soup Kitchen | Reads are scoped in the uncommitted work. The update path breaks rules 3 and 4. |
| Tool Library | `CommunityId` is stored, but commands default it to 1 and nothing filters by it |

### 10.2 Authorisation as a domain concept
Permissions are expressed in the ubiquitous language rather than as role names:

```
users.manage        roles.manage          members.register    members.manage
reservations.self-service   reservations.confirm-pickup   reservations.complete
tools.manage        tools.operate         loans.manage
communities.manage  geography.manage      support.manage
```

Roles map to these permissions (see `townsquare.md` §3.4). Services check **permissions**, not roles. The Tool Library adds a second layer, **ownership**: a caller may act only on their own reservations and loans unless they hold the staff permission (`AccessGuard`).

### 10.3 Tactical building blocks used everywhere
- **`Result` / `Result<T>`** with typed `ValidationError`, `NotFoundError` and `DomainError`. Invariant failures are values, not exceptions.
- **Factories:** a validating `Create(...)` that returns `Result<T>`, plus a separate `Rehydrate(...)` for loading from the database without validation.
- **Typed IDs** (int- or Guid-wrapping value objects).
- **Repositories per aggregate root** on a generic `SqlRepository<TEntity, TId>` with `ISqlEntityMapping<T>` (Dapper), and a **Unit of Work** per request.
- **CQRS read side:** queries skip the domain model and return DTOs directly from SQL, paginated (default 20–50, maximum 100–200).
- **Soft delete** (`DeletedAt` / `DeletedBy`) in Communities, Soup Kitchen, Support and Identity. The Tool Library still uses hard deletes, but exposes no delete endpoint.
- *Model tension:* `Entity<TId>`, `IAggregateRoot` and the unit of work are still copied into each service, because the shared kernel does not provide them yet (open issue common #2).

---

## 11. Ubiquitous language (by context)

| Context | Terms |
|---|---|
| **Tool Library** | Tool (catalogue record) · Inventory item / copy · Serial number · Original owner · Holder · Tool type · Amortisation rate · Scarcity (per location) · Manufacturer · **Member** · Access request · Identification · Verify · Approve / deny · Suspend / reinstate / ban · Member account (auth subject link) · **Loan** · Checkout · Due date · Renewal · Overdue · Fine / late fee per day · Outstanding fine balance · **Policy** (lending rules) · **Reservation** · Queue position · Next in line · Confirm pickup · Activate · Complete · **Repair** / maintenance record · Condition (New, Good, Fair, Repaired, Poor) · Lost |
| **Soup Kitchen** | Event (Planned, Active, Completed, Cancelled) · Max guests · Meal (category, quantity needed / prepared) · Guest (party size, contact) · Volunteer (role, hours scheduled, sign-up) · Donation (donor, item, quantity) |
| **Communities** | Country · Country code · City · Neighbourhood · Location (path) · Community · Slug · Quick register · Confirm · Pending confirmation · Active · Suspended · Archived |
| **Autheo** | User · Subject (`sub`) · Credential · Access token · Refresh token · Rotation · Reuse detection · Revoke · Password reset token · Community claim |
| **Identity** | User profile · Suspension (lockout end, reason, locked by) · Reinstate · Role · Permission · Pending email change |
| **Support** | Ticket · Category · Priority · Assign · Resolve · Reopen · Close · Incident · Escalate · Findings · Service status |
| **Learning** | Deck · Card · Cue · Expected answer · Box · Due · Review · Lapse · Confidence · Practice queue · Calibration |
| **Shared** | Community (tenant) · Permission · Domain event · Integration event · Outbox · Correlation ID |

**Words that mean different things in different contexts:**

| Word | Meanings |
|---|---|
| *Member* | In Tool Library: a verified borrower. In Soup Kitchen: a volunteer reference. In the JWT contract: "the user". |
| *Event* | In Soup Kitchen: a meal service. Everywhere else: a domain event. |
| *Location* | In Communities: a geography path. In Tool Library: a scarcity key. In Soup Kitchen: a free-text venue. |
| *Suspend* | In Tool Library: a member status. In Identity: an account lockout. In Communities: a community status. |

---

## 12. Where the model should go next (DDD view only)

1. **Close the circulation loop.** `ToolReturnedEvent` should end the active `Loan` and calculate its fine. Make `Banned` truly terminal. Move the loanability checks from `LoanToolCommandHandler` into the domain (for example a `CheckoutService` that takes Member, InventoryItem and Policy).
2. **Make Policy a real circulation rule.** Key it on *(location, member category, tool type)*. Use it for reservation expiry, renewal holds (no renewal when someone is queued) and per-member item limits.
3. **Add a Notifications context** that consumes `NextInLine`, `PickupReminder`, `ReturnReminder` and `LoanOverdue` through the outbox. This gives the outbox its first real producer and consumer.
4. **Guard the supporting state machines** (Community, Event, Guest, Volunteer), and enforce event capacity inside the `Event` aggregate.
5. **Make tenancy part of the model.** The tenant should be an immutable property of an aggregate, set by its factory from the token and never changed by an update. Add a repository-level tenant filter.
6. **Unify IAM.** Either Autheo issues claims from Identity's roles, permissions and suspension state (fed by integration events), or Identity becomes the role authority and Autheo queries it. Today there are two sources of truth.
7. **Decide who owns membership.** Communities defines the tenant while the Tool Library owns "who belongs". As the soup kitchen and other mutual-aid features grow, consider a **Membership** context, or move membership into Communities, so that soup-kitchen volunteers and tool-library members are the same verified neighbours.
8. **Finish the shared kernel** (`Entity`, `AggregateRoot`, `ValueObject` and the UoW contract in `tsqr-common`) so the building blocks above stop being copied between services.

---

## 13. Learning context (supporting)

The Learning context helps a learner remember what they study. It applies three keystone strategies from *Make It Stick*: retrieval practice, spaced retrieval and interleaving. It also records a confidence rating for calibration.

### 13.1 Aggregates

An aggregate root (AR) is the entity that one command loads and saves.

```
Deck (AR) ──────── a named set of cards, owned by one learner in one community
  • Name, owner (`sub`), community (`community_id`)

Card (AR) ──────── one prompt and its Leitner state
  • Cue, ExpectedAnswer
  • Box (LeitnerBox, 1 to 5), DueAt, ReviewCount, LapseCount
  └─ Review (insert-only row, one for each review of the card)
```

**Why Card is a separate aggregate root.** Each review changes the schedule and the counts of one card. Keeping the card separate from its deck keeps each command small. The deck does not change when a card is reviewed.

### 13.2 Value objects

- `LeitnerBox`: a box number from 1 to 5. A correct answer moves it up one box. A wrong answer moves it to box 1. The Leitner system is a flashcard system for spaced retrieval that the book describes.
- `Confidence`: one of Low, Medium or High. The learner gives it with each review.

### 13.3 Invariants

1. A correct answer moves the card up one box. The maximum is box 5.
2. A wrong answer moves the card to box 1 and adds one lapse.
3. The next review is due 1, 2, 4, 8 or 16 days later, for box 1 to 5. This is a design decision. The book does not give these intervals.
4. A card cannot be reviewed before `DueAt`.
5. A review needs a non-empty typed answer. The learner types the answer before the reveal.
6. The practice queue never returns `ExpectedAnswer`. The reveal returns it only after a typed attempt.
7. Every read and write filters by `community_id` and `sub`. A learner sees only their own decks and cards.
8. Each review adds one Review row. A Review row never changes.

### 13.4 Domain events

- `DeckCreatedDomainEvent`: a deck is created.
- `CardCreatedDomainEvent`: a card is created.
- `CardReviewedDomainEvent`: a card is reviewed.

No integration event is published in this version.

### 13.5 Not modelled yet

- Reflection, elaboration prompts, mnemonics and pretests.
- A separate Scheduling context. Card holds the prompt and its schedule together. Splitting them is a follow-up.

### 13.6 Sources

- Brown, P. C., Roediger, H. L. III and McDaniel, M. A. (2014). *Make It Stick: The Science of Successful Learning*. Belknap Press of Harvard University Press.
- Chapter titles and page numbers: https://www.degruyterbrill.com/document/doi/10.4159/9780674419377-toc/html
- Keystone strategies, pages 201 to 207 (chapter 8): https://cirl.etoncollege.com/?p=1136
- Calibration: chapter 5, "Avoid Illusions of Knowing".
- Not from the book: the Leitner interval numbers (1, 2, 4, 8 and 16 days) and the domain-driven design in this section.
