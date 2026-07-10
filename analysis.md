# TSQR (TownsSquare) — Domain-Driven Analysis

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Bounded Context Map](#2-bounded-context-map)
3. [Authentication Context (tsqr-autheo)](#3-authentication-context-tsqr-autheo)
4. [Identity & Profile Context (tsqr-identity)](#4-identity--profile-context-tsqr-identity)
5. [Community Context (tsqr-communities)](#5-community-context-tsqr-communities)
6. [Tool Library Context (tsqr-tool-lib)](#6-tool-library-context-tsqr-tool-lib)
7. [Soup Kitchen Context (tsqr-soup-kitchen)](#7-soup-kitchen-context-tsqr-soup-kitchen)
8. [Support Context (tsqr-support)](#8-support-context-tsqr-support)
9. [Presentation Layer (tsqr-ui / tsqr-support-ui)](#9-presentation-layer-tsqr-ui--tsqr-support-ui)
10. [Ubiquitous Language Glossary](#10-ubiquitous-language-glossary)
11. [Cross-Cutting Invariants](#11-cross-cutting-invariants)
12. [Summary: TownsSquare as a Whole](#12-summary-townssquare-as-a-whole)

---

## 1. Executive Summary

**TownsSquare** is a community-based digital platform for sustainability, collaboration, and community growth. It operates under the tagline *"Community-powered tool sharing for a more sustainable tomorrow."*

The platform's mission is threefold:

1. **Tool Library (Library of Things):** Enable communities to pool and share physical tools — from hand tools to power equipment — reducing individual consumption, lowering costs, and fostering neighbourly interdependence.
2. **Soup Kitchen Management:** Provide a digital backbone for community kitchens — event planning, meal coordination, volunteer scheduling, donation tracking, and guest registration — so that feeding programs run smoothly.
3. **Community Hub:** Organise the real-world geography into Countries → Cities → Neighbourhoods → Communities, giving each community a digital presence, a membership roster, and a suite of shared resources.

The architecture follows **Domain-Driven Design** with **Clean Architecture** across multiple C# .NET 10 microservices, each owning a distinct bounded context, sharing a common JWT-based authentication boundary, and communicating through a YARP reverse proxy gateway.

---

## 2. Bounded Context Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                  AUTHENTICATION BOUNDARY                      │   │
│   │                                                               │   │
│   │   ┌──────────────┐       ┌──────────────┐                    │   │
│   │   │  tsqr-autheo  │◄──────►  tsqr-identity │                  │   │
│   │   │  (Auth tokens │       │  (Profiles,   │                  │   │
│   │   │   login/reg)  │       │   roles)      │                  │   │
│   │   └──────┬───────┘       └──────┬───────┘                    │   │
│   └──────────┼──────────────────────┼────────────────────────────┘   │
│              │ JWT (sub, email,     │ JWT (role)                      │
│              │ name, role, mbr)     │                                  │
│              ▼                      ▼                                  │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                   DOMAIN BOUNDED CONTEXTS                    │   │
│   │                                                               │   │
│   │  ┌──────────────────┐  ┌──────────────────┐                  │   │
│   │  │ tsqr-communities  │  │  tsqr-tool-lib   │                  │   │
│   │  │ (Geography &      │  │ (Library of       │                  │   │
│   │  │  Communities)     │  │  Things)           │                  │   │
│   │  └────────┬─────────┘  └────────┬─────────┘                  │   │
│   │           │                      │                            │   │
│   │  ┌────────▼─────────┐  ┌────────▼─────────┐                  │   │
│   │  │ tsqr-soup-kitchen │  │  tsqr-support    │                  │   │
│   │  │ (Meals, Events,   │  │  (Tickets,       │                  │   │
│   │  │  Volunteers)      │  │   Uptime)         │                  │   │
│   │  └──────────────────┘  └──────────────────┘                  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                  PRESENTATION LAYER                          │   │
│   │                                                               │   │
│   │  ┌──────────────────┐  ┌──────────────────┐                  │   │
│   │  │ tsqr-ui           │  │ tsqr-support-ui  │                  │   │
│   │  │ (Next.js/React)   │  │ (Angular)        │                  │   │
│   │  └──────────────────┘  └──────────────────┘                  │   │
│   │                                                               │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                  CROSS-CUTTING: tsqr-deploy                  │   │
│   │                  tsqr-gateway (YARP proxy)                   │   │
│   │                  tsqr-common (shared primitives)             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Context Relationships

| Upstream Context | Downstream Context | Shared Artifact |
|---|---|---|
| tsqr-autheo | tsqr-identity | JWT token (sub claim for user ID) |
| tsqr-autheo | tsqr-tool-lib | JWT token (role, mbr claims for authorization) |
| tsqr-autheo | tsqr-communities | JWT token (auth gate) |
| tsqr-autheo | tsqr-support | JWT token (auth gate) |
| tsqr-identity | tsqr-tool-lib | MemberId ↔ UserId mapping |
| tsqr-communities | tsqr-tool-lib | CommunityId (scoping tool library to a community) |
| tsqr-communities | tsqr-soup-kitchen | CommunityId (scoping soup kitchen to a community) |

All domain contexts share:
- `tsqr-common` for `Result<T>`, `Error` types, `IInteractor<TCommand, TResult>` pattern
- `tsqr-gateway` for routing and BFF aggregation
- PostgreSQL for persistence (separate databases per context)

---

## 3. Authentication Context (tsqr-autheo)

### 3.1 Domain Intent

Authentication is the **gateway to the entire platform**. The problem it solves: *How does a person prove who they are, obtain a secure credential, and maintain their session across multiple microservices without re-authenticating for every request?*

This is a classic **Identity and Access Management** domain, but scoped specifically to a community platform context. The system must support:
- Self-registration (anyone can join as a Regular member)
- Admin-seeded privileged accounts
- Long-lived sessions via refresh token rotation
- Password recovery for non-admin users
- Role-based authorization claims embedded in the token

### 3.2 Aggregates

#### User (Aggregate Root)
- **Identity:** `UserId` (Guid) — a value object
- **Core Attributes:** `Email` (unique), `PasswordHash`, `FullName`, `Role`, `IsActive`, `IsEmailConfirmed`
- **Lifecycle:**
  - Created via `User.Create(...)` factory method with validation (non-empty email, non-empty hash, valid role)
  - Can be `Activate()`d / `Deactivate()`d (soft disable without data loss)
  - `ConfirmEmail()` marks the email as verified
  - `ChangePassword(newHash)` updates the password hash
  - `RecordLogin(at)` timestamps the last login
- **Domain Invariants:**
  - Email must be unique across the system (enforced by DB unique constraint + check in `RegisterHandler`)
  - A deactivated user must NOT be able to log in (`LoginHandler` checks `IsActive`)
  - Password hash must be non-empty and in the correct format (`iterations.salt.hash`)
  - Role must be a valid `RoleName` enum value

#### RefreshToken (Aggregate Root)
- **Identity:** `RefreshTokenId` (Guid)
- **Core Attributes:** `UserId`, `TokenHash` (SHA256 of the raw token), `ExpiresAt`, `CreatedAt`, `CreatedByIp`, `RevokedAt`, `RevokedByIp`, `ReplacedByTokenHash`, `RevokedReasonIsReuse`
- **Lifecycle:**
  - Created alongside a JWT access token during login
  - `Rotate()`: revokes the current token, creates a new token with `ReplacedByTokenHash` pointing to the new one
  - `Revoke(...)`: marks the token as revoked with reason and IP
  - `IsActive` (computed): not revoked AND not expired
- **Domain Invariants (critical security):**
  - The raw refresh token is given to the client exactly once; only the SHA256 hash is stored
  - **Family Revocation (theft detection):** If a revoked token is presented at `/refresh`, the entire descendant chain is walked via `ReplacedByTokenHash` and all descendants are revoked with `RevokedReasonIsReuse = true`. This prevents token theft — if an attacker steals a token and uses it after the legitimate owner has rotated, the attacker's use is the *old* (already revoked) token, and the entire family is burned.
  - Refresh tokens expire after a configurable period (default 7 days)
  - A single user can have multiple active refresh tokens (multiple devices/sessions)

#### PasswordResetToken (Entity)
- **Identity:** Internal Guid
- **Core Attributes:** `UserId`, `TokenHash`, `ExpiresAt`, `CreatedAt`, `IsUsed`, `UsedAt`
- **Lifecycle:**
  - Created on `/forgot-password` with a 1-hour expiry
  - `MarkAsUsed()` on successful reset
- **Domain Invariants:**
  - `IsValid` = not used AND not expired
  - Token hash prevents token prediction
  - `ForgotPasswordHandler` always returns success (even if email doesn't exist) to prevent user enumeration attacks
  - `RevokeAllForUserAsync` invalidates all existing reset tokens for a user before creating a new one

### 3.3 Domain Services

| Service | Role |
|---|---|
| `Pbkdf2PasswordHasher` | Hashes passwords with PBKDF2 (HMACSHA256, 100K iterations, 16-byte salt, 32-byte hash). Format: `iterations.salt-base64.hash-base64`. Uses `CryptographicOperations.FixedTimeEquals` for timing-safe verification. |
| `JwtTokenService` | Issues JWT access tokens (15 min expiry) with claims: `sub` (userId), `email`, `name`, `role`, `mbr` (memberId), `jti`. Generates 32-byte random refresh tokens. |

### 3.4 Domain Events

*None defined yet.* The domain layer defines `IAggregateRoot`, `IUnitOfWork`, and base `Entity<TId>`, but no domain events are raised. Events like `UserRegistered`, `UserLoggedIn`, `PasswordChanged` would be natural additions.

---

## 4. Identity & Profile Context (tsqr-identity)

### 4.1 Domain Intent

While `tsqr-autheo` handles *authentication* (verifying credentials and issuing tokens), `tsqr-identity` handles *identity* — the richer profile information and administrative management of users and roles. The problem it solves: *Once a user is authenticated, what additional profile data is needed? How are users administered? How are roles assigned?*

This bounded context is deliberately thinner — it assumes authentication has already happened and focuses on CRUD operations over profile data and role assignments.

### 4.2 Entities

#### UserProfile
- **Identity:** `UserId` (string, matching the `sub` claim from autheo's JWT)
- **Core Attributes:** `FirstName`, `LastName`, `AvatarUrl`, `Bio`, `CreatedAt`, `UpdatedAt`, `LockoutEnd`
- **Lifecycle:**
  - Created implicitly when a user first updates their profile (upsert pattern in `UpdateProfileHandler`)
  - Updated via `PUT /api/profile`
  - Can be locked/unlocked by admins
- **Domain Invariants:**
  - UserId must match the authenticated user's JWT `sub` claim (enforced by the API layer reading `ClaimTypes.NameIdentifier`)
  - `LockoutEnd` in the future means the user is locked; null or past means unlocked

#### Role
- **Identity:** `Role.Id` (string)
- **Core Attributes:** `Name`, `NormalizedName`
- **Lifecycle:** Created, listed, deleted by admins
- **Invariants:** Role names are case-insensitive (normalized to uppercase)

### 4.3 Domain Relationships (Cross-Context)

This context has an important **shared identity** relationship with `tsqr-autheo`:
- The `UserId` in tsqr-identity is the same GUID as the `sub` claim issued by tsqr-autheo
- Roles in tsqr-identity mirror the `RoleName` enum in tsqr-autheo
- When tsqr-autheo issues a JWT, the `role` claim is set; tsqr-identity allows admin-level assignment/removal of roles

### 4.4 Domain Invariants

- A user cannot be deleted if they have active role assignments (enforced by FK constraints)
- Admin operations (`/api/users/*`, `/api/roles/*`) require the caller to have the `Admin` role in their JWT
- `LockUser` with `LockoutEnd: null` is semantically an **unlock** operation
- Profile updates only modify the fields explicitly provided (partial update semantics)

---

## 5. Community Context (tsqr-communities)

### 5.1 Domain Intent

Before a tool library or soup kitchen can function, there must be a **community**. The Communities bounded context solves: *How do we model the real-world geographic hierarchy so that communities can be found, registered, and organised?*

The key insight is that every resource in the platform (tools, meals, events) is scoped to a **Community**. A Community is the organizing unit — it represents a real neighbourhood-level group of people who share resources.

### 5.2 The Geographic Hierarchy

The domain models a strict hierarchy:

```
Country (e.g., "Canada")
  └── City (e.g., "Toronto")
       └── Neighbourhood (e.g., "Leslieville")
            └── Community (e.g., "Leslieville Tool Share")
```

Each level is an **independent Aggregate Root** — they are not children of a single aggregate. This allows:
- Countries, cities, and neighbourhoods to be created independently
- De-duplication by name lookup during quick registration
- Multiple communities in the same neighbourhood

### 5.3 Aggregates

#### Country (Aggregate Root)
- **Identity:** `CountryId` (int)
- **Core Attributes:** `Name`, `Code` (auto-uppercased, e.g., "CAN")
- **Behaviors:** `Update(name, code)`
- **Invariants:** Code is always uppercase; no duplicate country names

#### City (Aggregate Root)
- **Identity:** `CityId` (int)
- **Core Attributes:** `Name`, `CountryId`
- **Behaviors:** `Update(name)`
- **Invariants:** A city belongs to exactly one country; city names are unique within a country (application-level)

#### Neighbourhood (Aggregate Root)
- **Identity:** `NeighbourhoodId` (int)
- **Core Attributes:** `Name`, `CityId`
- **Behaviors:** `Update(name)`
- **Invariants:** A neighbourhood belongs to exactly one city

#### Community (Aggregate Root — the Core Aggregate)
- **Identity:** `CommunityId` (int)
- **Core Attributes:** `NeighbourhoodId`, `Name`, `Slug` (auto-generated URL-friendly name), `Description`, `ContactEmail?`, `ContactPhone?`, `Status` (enum), `CreatedAt`
- **Behaviors:**
  - `Create(...)` — creates a community in `PendingConfirmation` status
  - `Confirm()` — transitions to `Active` (raises a conceptual domain event: "community is now live")
  - `Suspend()` — transitions to `Suspended` (administrative hold)
  - `Archive()` — transitions to `Archived` (soft deletion)
  - `UpdateDetails(...)` — updates name, slug, description, contact info
- **Status Lifecycle:**

```
PendingConfirmation ──► Active ──► Suspended
                      │              │
                      └──────◄───────┘
                      │
                      └──► Archived (terminal)
```

- **Domain Invariants:**
  - Slug is auto-generated from name and must be unique (not yet enforced at repository level, but should be)
  - At creation, the community is `PendingConfirmation` — it must be explicitly confirmed by an admin before it becomes `Active`
  - A `Suspended` community cannot create new tool library items, soup kitchen events, etc. (enforced by downstream contexts at the application level)
  - An `Archived` community is read-only (historical data preserved, no new operations)

### 5.4 Domain Services / Queries

| Service | Role |
|---|---|
| `IGeographyQueries` | Read-side lookups for de-duplication: given a country/city/neighbourhood name, find the existing record or return null so the caller can decide to create. |
| `QuickRegisterCommunityCommand` | Orchestrates find-or-create across all 4 levels of the hierarchy, then creates the community. This is the primary entry point for community registration. |

### 5.5 Cross-Context Sharing

Every downstream context references `CommunityId`:
- `tsqr-tool-lib`: Every `Tool`, `InventoryItem`, `Member`, `Loan`, `Reservation`, `MaintenanceRecord` has a `CommunityId`
- `tsqr-soup-kitchen`: Every `Event`, `Meal`, `Guest`, `Volunteer`, `Donation` has a `CommunityId`

This is a **shared identity** relationship — the CommunityId is a foreign key without a formal FK constraint across separate databases. Integrity is maintained at the application layer.

---

## 6. Tool Library Context (tsqr-tool-lib)

### 6.1 Domain Intent

This is the **richest and most mature bounded context** in the TSQR platform. It solves: *How does a community share physical tools among its members — from cataloging and inventory management to borrowing, returning, reserving, repairing, and enforcing accountability?*

This is a "Library of Things" domain — the library domain pattern applied to physical objects rather than books. It must handle:
- Tool cataloging (what tools exist abstractly)
- Physical inventory (multiple copies of the same tool, each with its own condition and status)
- Member management (who can borrow)
- Borrowing and returning (with due dates and fines)
- Reservations (queuing for popular tools)
- Maintenance tracking (repair lifecycle)
- Location-based scarcity (tool availability varies by neighbourhood)
- Policy-driven lending rules (late fees, max loan duration, renewals)

### 6.2 Aggregates

The domain is modelled as 7 aggregates, each with a clear transactional boundary:

#### Tool (Aggregate Root)
- **Identity:** `ToolId` (int)
- **Core Attributes:** `Model`, `Description`, `Manufacturer`, `Type` (enum), `AmortizationRate` (enum for wear), `ScarcityByLocation` (dictionary of LocationId → ScarcityLevel), `CommunityId`
- **Behaviors:**
  - `Register(...)` — creates a tool in the catalog, raises `ToolRegisteredEvent`
  - `UpdateToolDetails(...)` — changes model, description, type, amortization
  - `SetScarcityLevel(location, level)` — marks a tool as scarce in a given location
  - `RemoveScarcityLevel(location)` — clears scarcity for a location
- **Invariants:**
  - A Tool is a **catalog definition** — it describes what a tool *is*, not a physical copy
  - Scarcity levels: `NotSet`, `Low`, `Medium`, `High`, `Critical` — influence reservation queuing
  - ToolType categories: `HandTool`, `PowerTool`, `GardeningTool`, `ConstructionTool`, `SpecialtyTool`, `Other`

#### Manufacturer (Entity within Tool Aggregate)
- **Identity:** `ManufacturerId` (int)
- **Core Attributes:** `Name`, `CommunityId`
- **Role:** A reference entity for tool branding; scoped to a community

#### Policy (Entity within Tool Aggregate)
- **Identity:** `PolicyId` (int)
- **Core Attributes:** `ToolType`, `LocationId`, `LateFeePerDay`, `MaxLoanDurationDays`, `MaxRenewalCount`, `MaxLoanReservationDays`
- **Role:** Defines lending rules for a given tool type + location combination.
- **Current Status:** Defined in the domain model but **not yet enforced** by any application handler. This is a known gap.

#### Member (Aggregate Root)
- **Identity:** `MemberId` (int)
- **Core Attributes:** `FirstName`, `MiddleName`, `LastName`, `Age`, `Address`, `Email`, `PhoneNumber`, `Status` (Active/Suspended/Banned), `IsVerified`, `VerifiedByAdminId`, `VerificationDate`, `Identification` (value object), `AccessRequestStatus` (Pending/Approved/Denied), `MembershipRecord` (value object with start/end dates and type), `CommunityId`
- **Behaviors:**
  - `RequestAccess()` — initial request to join the community tool library
  - `ApproveAccess()` / `DenyAccess()` — admin response to access request
  - `Verify(...)` — identity verification, raises `MemberVerifiedEvent`
  - `Suspend()` / `Ban()` / `Reinstate()` — status transitions
  - `IsEligibleToBorrow()` — checks if member is Active, Verified, and not currently at fault
- **Status Lifecycle:**

```
┌─► Active (verified, can borrow)
│
RequestAccess ──► Pending ──► Approved ──► Verified ──► Active
                  │               │                        │
                  └──► Denied      └──► Suspended ──► Active (reinstated)
                                         │
                                         └──► Banned (terminal)
```

- **Invariants:**
  - A member must be `Approved` before they can be `Verified`
  - A member must be `Verified` to borrow tools
  - `Banned` is a terminal state (no reinstate from banned in current model)
  - A `Suspended` member cannot borrow but their existing loans remain active
  - `MembershipRecord` tracks start/end dates and type (`Regular`, `Repairman`, `LocationCoordinator`, `Admin`)
  - `Identification` requires both a type (`VideoIdentification`, `PictureIdentification`) and a reference string

#### InventoryItem (Aggregate Root)
- **Identity:** `InventoryItemId` (int)
- **Core Attributes:** `ToolId`, `OwnerId` (the member who contributed it), `SerialNumber`, `Status` (Available/Reserved/Loaned/UnderMaintenance/Lost), `Condition` (New/Good/Fair/Repaired/Poor), `CurrentHolderId`, `LoanCount`, `TotalUsageTime`, `IsUnderRepair`, `CommunityId`
- **Behaviors:**
  - `Loan(memberId, dueDate)` — marks item as `Loaned`, raises `ItemLoanedDomainEvent`
  - `Return(...)` — marks item as `Available` (or next in reservation queue), raises `ToolReturnedEvent`
  - `Reserve(memberId)` — marks item as `Reserved`
  - `ClearReservation()` — reverts to `Available`
  - `MarkAsLost()` — terminal status
  - `MarkForRepair(...)` — marks as `UnderMaintenance`, raises `ToolMarkedForRepairEvent`
  - `CompleteRepair(condition)` — back to `Available` with updated condition
- **Invariants:**
  - An InventoryItem is a **physical copy** of a Tool catalog definition
  - A tool cannot be loaned if it is `UnderMaintenance`, `Lost`, or `Loaned`
  - A tool cannot be reserved if it is already reserved
  - Status transitions are constrained: `Available → Reserved/Loaned`, `Loaned → Available/Lost`, `UnderMaintenance → Available`, `Lost` is terminal

#### Loan (Aggregate Root)
- **Identity:** `LoanId` (int)
- **Core Attributes:** `MemberId`, `ItemId`, `CheckoutDate`, `DueDate`, `Status` (Active/Returned/Overdue/Canceled), `ReturnedDate`, `FineAccrued`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — starts a loan
  - `EndLoan(returnedDate)` — computes overdue duration, accrues fine (hardcoded $1/day currently), raises `LoanOverdueDomainEvent` if overdue, transitions to `Returned`
- **Invariants:**
  - A loan is always for a single inventory item and a single member
  - `FineAccrued` is computed at return time based on days overdue × late fee per day
  - A loan cannot be created for a member who is not eligible to borrow
  - A loan cannot be created for an item that is not `Available`

#### Reservation (Aggregate Root)
- **Identity:** `ReservationId` (int)
- **Core Attributes:** `ItemId`, `MemberId`, `ReservationDate`, `ExpiryDate`, `Status` (Pending/Confirmed/Active/Cancelled/Completed), `QueuePosition`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — creates a pending reservation with a queue position
  - `ConfirmPickup()` — member confirms they will pick up, raises `ReservationConfirmedEvent`
  - `Activate()` — reservation is active (item is held for the member)
  - `Cancel()` — cancels, raises `ReservationCancelledEvent`, triggers `MoveDownInQueue` for subsequent reservations
  - `Complete()` — reservation fulfilled
  - `NotifyNextInLine()` — raises `NextInLineNotificationEvent`
- **Invariants:**
  - Reservations are queue-ordered per inventory item (`QueuePosition`)
  - A reservation expires after a configurable period (28 days max advance, enforced in `ReserveToolCommand`)
  - Only one reservation can be active per item at a time
  - When a reservation is cancelled or expires, the next in line is notified
  - `QueuePosition` is recalculated when a reservation is removed from the queue

#### MaintenanceRecord (Aggregate Root)
- **Identity:** `MaintenanceRecordId` (int)
- **Core Attributes:** `ItemId`, `ReportedById`, `ReportedDate`, `Description`, `Status` (Reported/InProgress/Completed), `CompletedById`, `CompletedDate`, `ResultingCondition`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — reports a need for repair
  - `StartWork()` — transitions to `InProgress`
  - `Complete(completedBy, resultingCondition)` — transitions to `Completed`, raises `ToolRepairedEvent`, updates the inventory item's condition
- **Invariants:**
  - A maintenance record is linked to exactly one inventory item
  - `ResultingCondition` is set only when the record is completed
  - An item cannot have two concurrent active (Reported or InProgress) maintenance records

### 6.3 Domain Services

| Service | Role |
|---|---|
| `FineService` | Calculates overdue fines: `daysOverdue * lateFeePerDay`. Referenced by `Loan.EndLoan`. |
| `MemberVerificationService` | Encapsulates eligibility checks: "Is this member eligible to borrow?" Checks `IsActive`, `IsVerified`, status not Suspended/Banned. Also validates admin authorization for verification. |
| `ReservationQueueService` | Manages the queue: calculates `QueuePosition` for new reservations, selects the next-in-line, shifts queue positions after a cancellation. |

### 6.4 Domain Events (14 total)

| Event | Raised By | Purpose |
|---|---|---|
| `ToolRegisteredEvent` | `Tool.Register()` | A new tool type has been added to the community catalog |
| `ItemLoanedDomainEvent` | `InventoryItem.Loan()` | A physical item has been checked out to a member |
| `ToolReturnedEvent` | `InventoryItem.Return()` | An item has been returned; notify next in reservation queue |
| `ToolMarkedForRepairEvent` | `InventoryItem.MarkForRepair()` | An item has been flagged for maintenance |
| `ToolRepairedEvent` | `MaintenanceRecord.Complete()` | A repair has been completed; item is available again |
| `MemberVerifiedEvent` | `Member.Verify()` | A member has completed identity verification |
| `LoanOverdueDomainEvent` | `Loan.EndLoan()` | A loan was returned past its due date |
| `ReservationConfirmedEvent` | `Reservation.ConfirmPickup()` | Member confirmed they will pick up the reserved item |
| `ReservationCancelledEvent` | `Reservation.Cancel()` | A reservation was cancelled; queue must shift |
| `ToolReservedEvent` | *(defined but not yet raised)* | An item has been reserved |
| `NextInLineNotificationEvent` | `Reservation.NotifyNextInLine()` | The next member in the queue should be notified |
| `PickupReminderEvent` | *(defined but not yet raised)* | Remind a member to pick up their reserved item |
| `ReturnReminderEvent` | *(defined but not yet raised)* | Remind a member to return a borrowed item |

### 6.5 Known Gaps (from domain analysis)

1. **Policy enforcement:** The `Policy` entity defines lending rules (fees, durations, renewals) but no handler currently enforces them. Fines are hardcoded to $1/day.
2. **Loan renewal:** `Policy.MaxRenewalCount` exists but no `Loan.Renew()` method is implemented.
3. **Some events are stubs:** `PickupReminderEvent`, `ReturnReminderEvent`, `ToolReservedEvent` are defined but never raised.
4. **Queue automation:** The `ReservationQueueService` shifts queue positions on cancellation, but the cascade to automatically notify the next-in-line is not fully wired through infrastructure (e.g., email/notification service).

---

## 7. Soup Kitchen Context (tsqr-soup-kitchen)

### 7.1 Domain Intent

The Soup Kitchen bounded context solves: *How does a community organise a feeding program — from planning an event, to preparing meals, to registering guests and volunteers, to tracking donations?*

This is a **community event management** domain specialised for food service. It is inherently event-centric: every activity revolves around a specific `Event` (the meal service happening on a given date at a given location).

### 7.2 Aggregates

#### Event (Aggregate Root)
- **Identity:** `EventId` (int)
- **Core Attributes:** `Name`, `Description`, `EventDate`, `Location`, `Status` (Planned/Active/Completed/Cancelled), `MaxGuests?`, `CommunityId`, `CreatedAt`
- **Behaviors:**
  - `Create(...)` — plans a new soup kitchen event
  - `UpdateDetails(...)` — changes name, description, date, location, max guests
  - `Activate()` — opens the event for volunteer/guest registration
  - `Complete()` — marks the event as finished
  - `Cancel()` — cancels the event
- **Lifecycle:**

```
Planned ──► Active ──► Completed
  │                      │
  └──► Cancelled         └── (terminal)
```

- **Invariants:**
  - An event must have at least a name and date
  - `MaxGuests` is an optional capacity limit (not yet enforced at the aggregate level — the `Guest` aggregate does not check against this)
  - Status transitions are one-way: `Completed` and `Cancelled` are terminal

#### Meal (Aggregate Root)
- **Identity:** `MealId` (int)
- **Core Attributes:** `EventId`, `Name`, `Description`, `Category` (Soup/MainDish/SideDish/Dessert/Beverage/Bread), `QuantityNeeded`, `QuantityPrepared`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — defines a meal item for an event
  - `UpdatePreparation(quantityPrepared)` — updates how much was actually prepared
- **Invariants:**
  - A meal belongs to exactly one event
  - `QuantityPrepared` should logically never exceed `QuantityNeeded` (not enforced at aggregate level — a future invariant)
  - Meal categories help with menu planning and dietary reporting

#### Guest (Aggregate Root)
- **Identity:** `GuestId` (int)
- **Core Attributes:** `EventId`, `Name`, `ContactInfo?`, `GuestCount`, `Status` (Pending/Confirmed/Cancelled), `RegisteredAt`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — registers a guest for an event
  - `Confirm()` — confirms attendance
  - `Cancel()` — cancels registration
- **Invariants:**
  - `GuestCount` represents how many people the guest is bringing (e.g., a family)
  - Total confirmed guests should not exceed `Event.MaxGuests` (not yet enforced)
  - A guest can be registered for multiple events but each registration is independent

#### Volunteer (Aggregate Root)
- **Identity:** `VolunteerId` (int)
- **Core Attributes:** `EventId`, `MemberId`, `Role` (Cook/Server/Cleaner/Organizer/Driver/Other), `Status` (Pending/Confirmed/Cancelled), `HoursScheduled?`, `Notes?`, `CommunityId`
- **Behaviors:**
  - `Create(...)` — signs up a volunteer
  - `Confirm()` — confirms their participation
  - `Cancel()` — withdrawal
- **Invariants:**
  - A volunteer is linked to a `MemberId` (the platform member), not just a name — this ties back to the Identity/Authentication context
  - `HoursScheduled` tracks expected volunteer hours (useful for coordination)
  - Each volunteer role is at the per-event level (a volunteer can serve in different roles for different events)

#### Donation (Aggregate Root)
- **Identity:** `DonationId` (int)
- **Core Attributes:** `EventId`, `DonorName`, `Item`, `Quantity`, `DonationDate`, `Notes?`, `CommunityId`
- **Lifecycle:** Simple — created once, no state changes
- **Invariants:**
  - A donation is always for a specific event
  - `Quantity` is a positive integer
  - Donor is tracked by name (not necessarily a platform member)

### 7.3 Domain Invariants (Cross-Aggregate)

| Invariant | Enforcement |
|---|---|
| A meal, guest, volunteer, or donation must reference an existing EventId | Application-level (no FK across aggregates) |
| All entities are scoped to a CommunityId | Every aggregate carries CommunityId |
| An event cannot accept new guests/volunteers if its status is not Active | Application handler checks |
| A confirmed guest count should not exceed Event.MaxGuests | Not yet enforced (known gap) |

### 7.4 Domain Events

*None defined.* The Soup Kitchen context has no domain events yet — it's primarily CRUD at this stage. Natural candidates would be: `EventScheduled`, `GuestRegistered`, `VolunteerSignedUp`, `DonationReceived`.

---

## 8. Support Context (tsqr-support)

### 8.1 Domain Intent

The Support Context solves: *How do community members get help, report issues, and stay informed about platform health?* This is a classic **customer support** domain — ticketing, service status, and contact forms — but scoped to the platform itself rather than to an individual community.

Unlike the other domains which are community-facing, this one is **platform-facing**: it supports the users of the TSQR platform.

### 8.2 Domain Model

This bounded context takes a simpler approach to domain modelling compared to tsqr-tool-lib. Rather than rich aggregates with behaviour, it uses:
- **Enums** for domain concepts (statuses, categories, states)
- **Repository and Query interfaces** for persistence
- **Read models** (records) for all query output
- **Commands** with handlers for mutation

#### Core Domain Concepts

**Ticket** — a support request submitted by a user
- Categories: `BugReport`, `FeatureRequest`, `AccessIssue`, `GeneralInquiry`
- Statuses: `Open` → `InProgress` → `Resolved`
- Fields: Subject, Description, SubmitterEmail, SubmitterMemberId, AssignedTo, ReportedAt, ResolvedAt, Summary
- **Invariants:**
  - A ticket in `Open` or `InProgress` status is considered "recent" (active)
  - `ResolvedAt` is set automatically when status transitions to `Resolved`
  - Tickets can be submitted without authentication (anonymous bug reports or inquiries)

**ContactMessage** — a simpler contact form submission
- Fields: Name, Email, Message, ReceivedAt
- Linked to a Ticket (auto-creates a `GeneralInquiry` ticket)
- **Invariant:** Every contact message generates a corresponding ticket

**App** — a catalog of TSQR platform applications
- Statuses: `Active`, `ComingSoon`, `Retired`
- Fields: Name, Description

**ServiceStatus** — uptime monitoring for platform services
- States: `Up`, `Degraded`, `Down`
- Fields: ServiceName, UptimePercent30d, Summary
- Services tracked: Tool Library API, Member Service, Reservation Service, Inventory Service, Notification Service, Postgres Database

**Incident** — historical record of service disruptions
- Statuses: `Investigating` → `Ongoing` → `Resolved`
- Fields: OccurredOn, ServiceName, DurationMinutes, Status

**TeamMember** — "About Us" team listing
- Fields: Initials, Name, RoleName

**AboutContent** — key-value CMS for informational pages

### 8.3 Domain Invariants

- Ticket categories and statuses are constrained by enum validation (range 1-4 for categories, 1-3 for statuses)
- Service states are constrained to enum values (1-3)
- Incident status flows are linear: `Investigating → Ongoing → Resolved`
- Contact form submissions always create a linked `GeneralInquiry` ticket (this is an application invariant enforced by `SubmitContactHandler`)

---

## 9. Presentation Layer (tsqr-ui / tsqr-support-ui)

### 9.1 tsqr-ui (Next.js / React)

The main frontend is a **Backend-for-Frontend (BFF)** architecture using Next.js App Router:

- **Server Components** fetch data directly from the backend gateway for initial page loads (detail pages, layouts)
- **Client Components** handle interactivity (forms, filters, pagination, actions) via Next.js API routes that proxy to the gateway
- **Authentication** is managed via httpOnly cookies (`tsqr_at` for access token, `tsqr_rt` for refresh token) with transparent refresh on 401 responses
- **Route organisation** mirrors the bounded contexts:
  - `/portal/*` — Dashboard, Members, Reservations, Loans, Inventory
  - `/tools/*` — Tool catalog, registration, actions
  - `/soup-kitchen/*` — Events, Meals, Volunteers, Donations, Guests
  - `/communities/*` — Browse, Quick Register, Community detail/edit

### 9.2 tsqr-support-ui (Angular)

A standalone Angular SPA for the Support Portal:
- Tickets management (list, create, update status)
- Service uptime dashboard
- Contact form
- Apps catalog
- About Us / Team page

This is deployed separately and accessed at `localhost:4200` in development, proxied through an nginx container.

---

## 10. Ubiquitous Language Glossary

| Term | Definition | Context |
|---|---|---|
| **AmortizationRate** | A rating of how quickly a tool wears out (Low, Medium, High) | Tool Library |
| **Community** | A neighbourhood-level group that shares resources | Communities, all domains |
| **CommunityId** | The identity that scopes all resources to a community | All domains |
| **Condition** | The physical state of an inventory item (New, Good, Fair, Repaired, Poor) | Tool Library |
| **Family Revocation** | Security mechanism that revokes an entire chain of refresh tokens if token reuse is detected | Authentication |
| **InventoryItem** | A physical copy of a tool catalog definition | Tool Library |
| **JWT** | JSON Web Token carrying authentication and authorization claims | Authentication (gateway to all) |
| **Library of Things** | The domain of borrowing physical objects (tools) rather than books | Tool Library |
| **Loan** | The record of a member borrowing an inventory item | Tool Library |
| **MaintenanceRecord** | A repair ticket for an inventory item | Tool Library |
| **Member** | A person who has joined a community tool library | Tool Library |
| **Reservation** | A queue position to borrow an inventory item in the future | Tool Library |
| **ScarcityLevel** | How hard it is to find a given tool in a given location (Low to Critical) | Tool Library |
| **Slug** | A URL-friendly identifier generated from a community name | Communities |
| **Tool** | The catalog definition of a type of tool (not a physical copy) | Tool Library |
| **ToolType** | The category of a tool (HandTool, PowerTool, etc.) | Tool Library |

---

## 11. Cross-Cutting Invariants

### 11.1 Community Scope Invariant

**Every domain entity across Tool Library, Soup Kitchen, and Communities is scoped to exactly one `CommunityId`.** This is the fundamental tenet of the platform: all resources belong to a community. There are no global/shared resources.

### 11.2 Authentication Boundary

- All domain microservices validate JWT tokens issued by `tsqr-autheo`
- The shared key (`Jwt:Key`) must be identical across all services
- The `sub` claim identifies the user; the `role` claim authorizes actions; the `mbr` claim links to the Tool Library Member identity
- Default authorization policy: all endpoints require authentication unless explicitly marked `[AllowAnonymous]`

### 11.3 Identity Mapping

There are two distinct identity systems that must be kept in sync:

| System | Identity | Surface |
|---|---|---|
| tsqr-autheo | `UserId` (Guid) | The platform user account (login credential) |
| tsqr-identity | `UserId` (string) | Same GUID, stored as string for profile and roles |
| tsqr-tool-lib | `MemberId` (int) | The community membership, linked via autheo's `mbr` JWT claim |
| tsqr-soup-kitchen | `MemberId` (int) | Same as tool-lib, used for Volunteer registration |

### 11.4 Repository & Unit of Work Pattern

All backend microservices follow a consistent pattern:
- `IRepository<TAggregateRoot, TId>` with `GetByIdAsync`, `AddAsync`, `Update`, `Delete`
- `IUnitOfWork.SaveAsync()` for transactional commit
- Dapper implementation with PostgreSQL, retry logic (3 attempts) for transient failures
- Generic `SqlRepository<TEntity, TId>` with `ISqlEntityMapping<TEntity>` per aggregate

### 11.5 Command/Query Pattern (CQRS)

All services use `IInteractor<TCommand, TResult>` for commands and separate query handlers for reads. The read side uses direct SQL via Dapper (bypassing the aggregate repositories), while the write side uses repositories + unit of work for transactional consistency.

---

## 12. Summary: TownsSquare as a Whole

TownsSquare (TSQR) is a **community-powered platform for sustainable living**. It transforms the concept of a "library of things" from a single physical location into a digital community resource-sharing ecosystem.

### The Problem

In modern consumer society, every household owns tools and equipment that are used rarely — a pressure washer used twice a year, a tent used once, a power drill used for a weekend project. This is economically wasteful and environmentally unsustainable. Meanwhile, community kitchens and food programs struggle with coordination — spreadsheets, phone trees, and paper sign-up sheets.

### The Solution

TownsSquare provides a **full-stack digital platform** organised around real-world neighbourhoods (Communities) that enables:

1. **Resource Pooling (Tool Library):** Members contribute tools to a shared inventory. Other members borrow them through a structured system with reservations, due dates, fines, and maintenance tracking. Scarcity levels help communities understand which tools are most needed.

2. **Community Feeding (Soup Kitchen):** Event-based coordination for meal services — planning menus, registering guests, scheduling volunteers, and tracking donations. Everything is scoped to a community so that the soup kitchen serves the same neighbourhood the event is in.

3. **Community Organisation (Communities):** A geographic hierarchy (Country → City → Neighbourhood → Community) that makes it easy to discover and join local groups. Each community has its own lifecycle (pending confirmation → active → suspended/archived).

4. **Platform Support (Support):** A support portal with tickets, service status monitoring, and contact forms — because the platform itself needs to be supported.

5. **Centralised Identity (Autheo + Identity):** One account, one login, JWT-based single sign-on across all services. Role-based access control (Regular, Repairman, LocationCoordinator, Admin) ensures the right people have the right permissions.

### Architectural Philosophy

The system is built on **Domain-Driven Design** principles:
- Each bounded context is an independent microservice with its own database
- Aggregates enforce their own invariants
- Communication between contexts is through shared identities (CommunityId, UserId, MemberId) rather than direct service calls
- The YARP gateway provides a single entry point and BFF aggregation
- The `Result<T>` pattern (from tsqr-common) ensures explicit error handling rather than exceptions

### Current Maturity

- **tsqr-tool-lib** is the most mature — full DDD model with 7 aggregates, 14 domain events, 3 domain services. Some gaps remain (policy enforcement, event wiring).
- **tsqr-autheo** is production-ready with robust security (PBKDF2, family revocation, timing-safe comparison).
- **tsqr-communities**, **tsqr-soup-kitchen**, **tsqr-support**, **tsqr-identity** are functional but have simpler domain models with fewer domain events and invariants.
- **tsqr-ui** (Next.js) covers the full surface area of all domains with a BFF architecture.
- **tsqr-support-ui** (Angular) provides the support portal.
- **tsqr-deploy** orchestrates everything via Docker Compose with a shared PostgreSQL instance.

The platform is well-positioned as a **real-world community sustainability tool** — the domain model reflects genuine operational needs of community tool libraries and soup kitchens, with room to grow into notification services, payment processing, and cross-community resource sharing.
