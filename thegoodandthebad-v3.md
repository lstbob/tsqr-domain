# TSQR (TownsSquare): DDD Design Audit, Round 4 (v3)

**Audit date:** 2026-11-26. **Previous audits:** [`thegoodandthebad.md`](./thegoodandthebad.md) (round 1), [`thegoodandthebad-v2.md`](./thegoodandthebad-v2.md) (round 2, status as of 2026-11-19, and Round 3), [`post-analyi.txt`](./post-analyi.txt).
**Product overview:** [`townsquare.md`](./townsquare.md). **DDD model:** [`domain-driven-design.md`](./domain-driven-design.md).

**Method**

- All `tsqr-*` repositories were re-read on `main` HEAD, after `git fetch` showed every repo in sync with `origin`.
- The work was split into four parallel passes:
  - tool-lib;
  - autheo + identity;
  - communities + soup-kitchen + support;
  - cross-cutting: common, gateway, deploy, UIs and sdlc.
- Every Round 2 and Round 3 finding was re-checked against the code.
- New findings are numbered **V3-n**. Those marked **(v)** were also confirmed by reading the cited code directly when this document was written.
- `dotnet build` / `dotnet test` was run for every .NET repo (dotnet 10.0.401).
- None of the V3 findings have been filed as GitHub issues.

**Legend:**
- ✅ fixed and confirmed in code.
- 🟡 shipped, but a gap is still in code.
- ❌ open.
- Paths are relative to the repo named in the row.

---

## 1. Executive summary

**What changed since 2026-11-19 (about 35 commits):**
- **autheo:** RS256 signing, JWKS and key-ring rotation, rate limits on the auth endpoints, forwarded headers, and the `community_id` claim derived from tool-library membership.
- **tsqr-common:** shared domain kernel (`Entity`, `ValueObject`, `IAggregateRoot`, `IIntegerId`) and multi-key RS256 validation. Five services adopted the kernel.
- **soup-kitchen:** reads and writes scoped by community, state-machine guards, transition endpoints, and the guest-capacity limit.
- **communities:** lifecycle guards and race-safe quick-register.
- **gateway:** shared RS256 validation, the JWKS proxy, and the public soup-kitchen read routes removed.
- **deploy:** RS256 keys and unique geography keys.

**What did not change:**
- **tool-lib.** Apart from package bumps and the kernel refactor, it is untouched. All six of its Round 3 core-loop bugs are still open, including **R3-1: returning a tool never ends the loan.**
- **identity.** No commits at all, so R3-5, R3-6, R3-11 and R3-20 are unchanged.
- **support.** R3-4, the anonymous read of submitter emails, is still open and is also routed publicly by the gateway.

**Headline new risks (V3):**
1. **Identity can be made to accept HS256 tokens signed with a key that is committed to git (V3-1).** This happens when the RS256 env vars are missing, because common silently falls back to `Jwt:Key`.
2. **Communities can silently lose data (V3-2).** After a transient error, the commit retry commits an *empty* transaction and reports success.
3. **The tool-lib circulation loop is broken in more places than Round 3 found (V3-3 to V3-6, V3-12 to V3-16).**
   - Reservations are never completed on loan, so items get blocked.
   - Queue jumping is possible.
   - Regular members cannot borrow.
   - Repair on a loaned item corrupts its state.
   - Fines can never be paid down.
4. **Authorization within a tenant is coarse (V3-7 to V3-9).**
   - A community coordinator can edit *any* community.
   - Soup-kitchen has no permission checks at all.
   - Every member can read every support ticket in their community, including submitters' emails.
5. **The per-IP rate limiter in common is effectively global (V3-10),** because no service except autheo uses forwarded headers.

**Build and tests:** all green (section 6). None of the risky paths above has a test, and no repo has Postgres integration tests.

### Scorecard

| Set | ✅ | 🟡 | ❌ | Change since 2026-11-19 |
|---|---|---|---|---|
| Round 2 (#1–#29) | 11 | 16 | 2 | was 9 / 12 / 8 |
| Round 3 (R3-1 – R3-23) | 1 | 6 | 16 | first status check |
| New in v3 | n/a | n/a | 37 | 10 High, 17 Medium, 10 Low |

Open GitHub issues: autheo#20, deploy#3, identity#6, support#6. sdlc#6 is a sprint alert.

---

## 2. Status of Round 2 issues (#1–#29)

| # | Issue | Status | What is true in code today |
|---|---|---|---|
| 1 | tool-lib#43 Queue direction | ✅ | Fixed. `ReservationCreated` still carries `ReservationId = 0` (see V3-32). |
| 2 | tool-lib#44 Ownership checks | 🟡 | Reserve, activate, cancel, confirm, loan, repair and lost are guarded. **Renew** (`LoansCommandController.cs:333-345`) and **Return** (`ToolsCommandController.cs:186-198`) have no ownership check. |
| 3 | tool-lib#45 Orphaned handlers | 🟡 | All handlers are registered, but 6 are audit-only. `ReturnReminderHandler` is a stub and its event is never raised. |
| 4 | tool-lib#46 RenewLoan | 🟡 | It ignores waiting reservations and has no permission or ownership check (`RenewLoanCommand.cs:30-40`). In practice it cannot be reached (V3-14). |
| 5 | tool-lib#47 MarkForRepair one-aggregate | ✅ | |
| 6 | tool-lib#48 Eligibility / fines | ✅ | Works, but combined with R3-1 and V3-15 it locks members out. |
| 7 | communities#6 Authorization / QuickRegister | 🟡 | The service requires `communities.manage`. The gateway still routes quick-register as public (`tsqr-gateway .../appsettings.json:185-188`). The anonymous list and detail endpoints return ContactEmail and ContactPhone (V3-20). |
| 8 | communities#7 Lifecycle guards | ✅ | `Community.cs:55-80`, with tests. *(was ❌)* |
| 9 | communities#8 Read query layer | ✅ | |
| 10 | communities#9 Slug / typed FKs / CountryCode | 🟡 | `CountryCode.FromName` still uses the first two letters (`CountryCode.cs:119-133`). A collision now returns a clean `CountryCodeConflict`, but "Australia" cannot be added once "Austria" exists. |
| 11 | soup-kitchen#6 Multi-tenancy / PII | 🟡 | Every read filters by CommunityId, every write compares the tenant, and a missing claim gets 403. *(was ❌)* Within a tenant, any member can read guest PII (V3-8). |
| 12 | soup-kitchen#7 Cross-aggregate invariants | 🟡 | Capacity is enforced (`Event.cs:193-209`). Event status is not checked on adds (`Event.cs:158-261`). *(was ❌)* |
| 13 | soup-kitchen#8 State machines / Donation | ✅ | There is a transition table (`Event.cs:139-156`), and confirm/cancel endpoints for events, guests and volunteers. *(was ❌)* |
| 14 | soup-kitchen#9 Events / UoW | 🟡 | Events are dispatched after commit; there are still **no `IDomainEventHandler` implementations**. |
| 15 | identity#5 Email change | ✅ | Still not atomic: autheo is called first, then the local confirm is saved (`ChangeEmailCommand.cs:51-57`). |
| 16 | identity#6 Entity base / IAggregateRoot | ❌ | Identity has its own `IAggregateRoot` (`Domain/Contracts.cs:3`), no Entity base, and has not adopted the shared kernel. |
| 17 | identity#7 Suspend / Reinstate | ✅ | Domain only. **Not enforced at login** (R3-6). |
| 18 | identity#8 CancellationToken / upsert | 🟡 | The COALESCE upsert is still in place, so fields cannot be cleared (`ProfileRepository.cs:53-63`). |
| 19 | identity#9 Permission model | 🟡 | Token permissions still come from the static `RolePermissions.ForRole(role)` (`autheo .../IJwtTokenService.cs:95`). |
| 20 | autheo#3 | ✅ | |
| 21 | autheo#4 Rotation lock | ✅ | |
| 22 | autheo#5 Rate limit / RS256 / audiences | 🟡 | RS256, JWKS and the auth rate limits shipped. There is still a single audience, `tsqr-services` (autheo#20 is open). |
| 23 | support#5 Submission auth / tenancy | 🟡 | **Ticket list and detail are still `[AllowAnonymous]` (v)** (`Controllers.cs:130-131,142-143`). `CurrentUser.cs:46` falls back to community 1 (v). See R3-4. |
| 24 | support#6 Comments / attachments | 🟡 | No comment or attachment code exists. The issue is still open. |
| 25 | common#2 Shared DDD kernel | 🟡 | Shipped and adopted by 5 of 6 services. **Identity is not migrated.** *(was ❌)* |
| 26 | common#3 `TSQR.Common.WebApi` | 🟡 | All 6 services call `AddTsqrApiDefaults`. The gateway still has its own OTel and correlation code (`Program.cs:21-63`, `Observability/*`). |
| 27 | deploy#2 SQL drift | 🟡 | The deploy SQL is correct. tool-lib's local `WebApi/docker/init.sql` has still drifted (R3-19). |
| 28 | deploy#3 Migration runner | ❌ | No DbUp or FluentMigrator. `infra/09-soup-kitchen-community.sql:4-8` uses bare `ADD COLUMN`. |
| 29 | autheo#6 `mbr` contract | ✅ | |

---

## 3. Status of Round 3 findings (R3-1 – R3-23)

| # | Finding | Status | What is true in code today |
|---|---|---|---|
| R3-1 | Return never ends the Loan | ❌ **Critical** | `ReturnToolCommand.cs:95` and `ToolReturnedEventHandler` never call `EndLoan`. The only caller is `MarkLoanAsNotReturnedCommand.cs:18` (v). It is **worse than reported**: `LoanToolCommand.cs:54` also blocks the same member from borrowing the same item again, even before the due date. |
| R3-2 | Soup-kitchen UpdateEvent takeover | 🟡 | The tenant check on load is fixed (`UpdateEventCommand.cs:20-21`) and CommunityId cannot change (`Event.cs:90-95`). `UpdateDetails` still skips name, location and past-date validation and still accepts a raw status (`Event.cs:81-116`). |
| R3-3 | Soup-kitchen WIP does not compile | ✅ | Committed as 828b0f5. It builds, and the dashboard query is tenant-scoped. |
| R3-4 | Support anonymous PII / default tenant | ❌ **Critical** | The gateway routes `support-tickets-read` and `-get` with no policy (v) (`tsqr-gateway .../appsettings.json:73-80`). The service is `[AllowAnonymous]`, the DTOs carry `SubmitterEmail` and `SubmitterMemberId` (`Queries.cs:43-44`), and a missing claim falls back to tenant 1. The sdlc README (line 41) says this hardening "shipped". |
| R3-5 | JWT key committed in identity | ❌ **Critical** | It is still in `tsqr-identity/src/TSQR.Identity.Api/appsettings.json:12` (v) and in git history. It is now also a **live fallback**: see V3-1. |
| R3-6 | Suspension / deletion do not stop logins | ❌ | `LoginHandler.cs:26` and `RefreshHandler.cs:44` check only `IsActive`. Identity never notifies autheo. |
| R3-7 | Tool-lib access requests not persisted | ❌ | There are 0 references to `AccessRequestStatus` or `Identification` in Infrastructure, init.sql or the deploy SQL. |
| R3-8 | Seeded Specialty policy invalid | ❌ | `Policy.cs:141` rejects values ≤ 0, and the error message ("cannot be negative") is also wrong. The seed still uses 0 (`init.sql:270`, deploy `02:182`, `11:35`). |
| R3-9 | Tool-lib tenancy only a column | ❌ | The WebApi never reads `community_id`. Commands default to `CommunityId = 1`. No query filters by tenant. |
| R3-10 | Tool-lib authorization gaps | ❌ | `AdminId` and `OwnerId` come from the request body (`MembersCommandController.cs:95,133`, `ToolsCommandController.cs:68`). `request-access` is `[AllowAnonymous]` (`:61`). Renew has no check. Member and loan reads are open to any authenticated user. |
| R3-11 | Two disconnected IAM models | ❌ | Identity's `user_roles` never reaches the token. `RoleRepository` still has Create only. |
| R3-12 | Banned is not terminal | ❌ | `Member.cs:202-208`: `Suspend()` only guards `Suspended`. |
| R3-13 | Autheo hygiene | 🟡 | Fixed: forwarded headers (`Program.cs:23-30,70`) and an auth limiter of 5 per minute per IP. Still open: reset and change-password do not revoke refresh tokens (`ResetPasswordHandler.cs:39-48`, `ChangePasswordHandler.cs:26-31`); 7 days is hard-coded (`LoginHandler.cs:42`, `RefreshHandler.cs:50`); the password minimum is 6. |
| R3-14 | Gateway allows by default / route drift | 🟡 | The soup-kitchen public routes were removed, with tests. Still missing: a FallbackPolicy (`Program.cs:72-75`), a rate limiter and forwarded headers. Quick-register is still public. See V3-22. |
| R3-15 | Communities | 🟡 | Lifecycle is fixed. Still open: CountryCode (#10), the commit retry (**now confirmed as data loss**, V3-2), rehydration through the public constructor (`CommunityMapping.cs:52`), and `CreateCommunity` confirming automatically (`CreateCommunityCommand.cs:66`). |
| R3-16 | Soup-kitchen | 🟡 | Capacity is fixed, and concurrent adds are serialised by `xmin`, so they return 409, not an overbooking (untested). Still open: adds to closed events (`Event.cs:158-261`), no event handlers, N+1 loading (`EventRepository.cs:23-63`), and `Volunteer.MemberId` taken from the body (`VolunteersController.cs:67`). That field is also an `int`, while `sub` is a Guid, so it cannot be bound to the caller at all. |
| R3-17 | Tool-lib circulation rules | ❌ | Nothing reads `ExpiryDate` and there is no scheduler. Reserve does not check eligibility, duplicates, a per-member limit, or whether the item is lost. |
| R3-18 | Messaging | ❌ | Only tool-lib registers `LoggingIntegrationEventTransport` (`Program.cs:76`). There is no producer, inbox or broker. |
| R3-19 | Schema drift / migrations / OTLP | ❌ | tool-lib `init.sql` lacks `MemberAccounts` and `CommunityId` on 7 tables, so local compose fails on the first insert. There is no OTLP collector or `OTEL_*` env. |
| R3-20 | Identity profile / avatar | ❌ | COALESCE upsert (#18). `Directory.GetFiles(avatarsDir, $"{userId}.*")` sits on an anonymous route (`ProfileController.cs:133-137`). tsqr-common is still a ProjectReference. |
| R3-21 | Tool-lib read side | ❌ | `LoansReadController.cs:21-38` reads InventoryItems. `GetToolsQuery.cs:40` pages in memory. The dashboard uses magic numbers. |
| R3-22 | Product surface / stale docs | ❌ | `tsqr-ui` `app/page.tsx` has 5 `href="#"` links. `tsqr-support-ui` makes no HTTP calls. tool-lib docs are stale (`current-work.md:174`). |
| R3-23 | Housekeeping | 🟡 | The stray `Test` class is gone. SHA-256 `HashToken`-style code is still copied in several autheo handlers and in identity `EmailChangeToken.cs`. The empty `[tool-lib` and `[tool-lib]` files are still in tsqr-deploy. tsqr-tempalert is still unrelated. |

---

## 4. New findings (V3)

### High

#### V3-1. Identity falls back to the committed HS256 key (v): common + identity + deploy
- **What happens:**
  - `TsqrJwtAuthenticationExtensions.cs:68-78` (tsqr-common) uses `Jwt:Key` (HMAC) whenever neither `Jwt:PublicKeys` nor `Jwt:PublicKeyBase64` is set.
  - Identity ships a `Jwt:Key` in `appsettings.json:12`, and deploy runs it as `Production`.
  - So if `JWT_PUBLIC_KEY_BASE64` / `JWT_PUBLIC_KEYS` is empty, identity starts normally and accepts HS256 tokens signed with a key anyone with repo access holds. Those tokens can carry `users.manage` and `roles.manage`.
  - Autheo, by contrast, refuses to start without its keys.
- **Fix:**
  - Remove the symmetric fallback from common, or allow it only in Development.
  - Delete `Jwt:Key` from identity and rotate it.
  - Add a startup test.

#### V3-2. Communities commit retry silently loses data (v)
- **What happens:**
  - On a transient error, `DapperUnitOfWork.SaveChangesAsync` (`DapperUnitOfWork.cs:35-56`) rolls back, disposes the transaction and calls `EnsureTransaction()`, which opens a **new, empty** transaction. The loop then commits it and returns success.
  - So create and quick-register can return an Id for a row that was never stored.
- **Fix:** remove the retry and let the error surface, as soup-kitchen and support already do.

#### V3-3. Tool-lib: reservations are never completed when the item is loaned
- **What happens:**
  - `LoanToolCommand.cs:63-66` only reads reservations, and no `ItemLoanedDomainEvent` handler completes them. The holder's Confirmed or Active reservation therefore survives the loan.
  - After the return, either every other member is rejected with "reserved by another member", or `ToolReturnedEventHandler` re-holds the item for the person who just returned it.
  - Repair-triggered cancellation also skips Active reservations (`ToolMarkedForRepairNotificationHandler.cs:156-158`).
- **Fix:** complete the matching reservation inside the loan transaction.

#### V3-4. Tool-lib: queue jumping
- **What happens:**
  - `Reservation.ConfirmPickup()` (`Reservation.cs:128-144`) ignores `QueuePosition`.
  - `LoanToolCommand.cs:64` takes the first Confirmed or Active reservation in no particular order.
  - So position 5 can confirm and borrow ahead of position 1.
- **Fix:** allow confirm only for the next in line, and order the reservations by position.

#### V3-5. Tool-lib: loan permissions make self-service borrowing impossible
- **What happens:**
  - The loan endpoint requires `tools.operate`, which only Admin, LocationCoordinator and Repairman have. Regular members do not have it, so they cannot borrow at all.
  - Lending on behalf of another member requires `members.manage`, which only Admin has (`ToolsCommandController.cs:171`, tsqr-common `RolePermissions.cs`). So a coordinator who can operate tools cannot check one out to a member.
- **Fix:**
  - Add a `tools.borrow` permission for self-service.
  - Base `CanLoanForOthers` on `tools.operate`.

#### V3-6. Tool-lib: marking a loaned tool for repair corrupts its state (v)
- **What happens:**
  - `InventoryItem.MarkForRepair` (`InventoryItem.cs:204-225`) checks only whether a repair is already open, not the item's status.
  - A Loaned item becomes UnderMaintenance but keeps `CurrentHolderId` and its active loan.
  - After that, Return fails (`:153`), and `CompleteRepair` sets the item to Available with the borrower still recorded as holder (`:246-247`).
- **Fix:** reject repair on Loaned items, or require the return first.

#### V3-7. Communities: a LocationCoordinator can edit any community
- **What happens:**
  - LocationCoordinator has `communities.manage` (tsqr-common `RolePermissions.cs:68`), and the service never reads `community_id`.
  - So a coordinator of community A can `PUT /api/communities` to rewrite community B (`UpdateCommunityCommand.cs:158-161`).
- **Fix:** on update, require platform admin, or `community_id` equal to the target Id.

#### V3-8. Soup-kitchen: no permission checks on any endpoint
- **What happens:**
  - Every controller uses plain `[Authorize]` (e.g. `EventsController.cs:41`, `GuestsController.cs:12`), and `Permissions.All` has no soup-kitchen permission.
  - So any Regular member of the community can create, update and cancel events, and can read guests' `ContactInfo` (`GetGuestsQuery.cs:32`).
- **Fix:**
  - Add a `soupkitchen.manage` permission and use it for writes and guest reads.
  - Let volunteers sign themselves up via `sub`.

#### V3-9. Support: any member can read every ticket in the community
- **What happens:**
  - Even with authentication, `ListTickets` returns the email and description of every ticket in the community (`SupportQueries.cs:69-76`).
  - This is the second layer of R3-4. Fixing the anonymous access alone is not enough.
- **Fix:** callers without `support.manage` should see only tickets where `SubmitterMemberId = sub`.

#### V3-10. The shared rate limiter is effectively global (v)
- **What happens:**
  - The `TSQR.Common.WebApi` limiter partitions by `Connection.RemoteIpAddress` (`TsqrApiDefaultsExtensions.cs:125`).
  - Common never calls `UseForwardedHeaders`, so in 5 services every client shares the gateway's IP and one bucket of 600 requests per minute (`TsqrApiDefaultsOptions.cs:32`).
  - One noisy client can therefore throttle everyone.
- **Fix:** move autheo's forwarded-headers setup into `UseTsqrApiDefaults`, trusting only the gateway network (see also V3-19).

### Medium

| # | Context | Finding | Fix |
|---|---|---|---|
| V3-11 | deploy / autheo / identity | `Email__LogResetToken: "true"` and `Email__LogConfirmationToken: "true"` are set next to `ASPNETCORE_ENVIRONMENT: Production` (v) (`tsqr-deploy/docker-compose.yml:101-102,133-134`). `LoggingEmailSender.cs:21-31` writes the reset URL, including the token, to the logs, so anyone who can read the logs can take over accounts. | Refuse these flags outside Development. |
| V3-12 | tool-lib | `MarkLoanAsNotReturned` before the due date: `EndLoan(UtcNow)` takes the else-branch (`Loan.cs:464-468`), sets the loan to `Returned` with no fine, and leaves the item `Loaned`. The overdue branch never sets `ReturnedDate`. | Add a dedicated `NotReturned`/`Lost` transition on Loan. |
| V3-13 | tool-lib | `MarkAsLost` (`InventoryItem.cs:194-202`) raises no event, does not end the active loan (so the borrower is later locked out as overdue), does not cancel the queue, and leaves any open repair. The uncommitted working-tree diff only drops an unused parameter; `ReporterId` is still collected and then thrown away. | Raise an `ItemLost` event, add handlers for the loan, reservations and repair, and record the reporter. |
| V3-14 | tool-lib | No read endpoint returns a `LoanId`. `GET /api/loans` returns InventoryItems, and the loan POST returns `Ok()` with no Id. So renew and mark-not-returned cannot be reached from a client. | Add a read model over the Loans table, and return the Id on create. |
| V3-15 | tool-lib | Fines can never be paid down. `DapperLoanRepository.cs:47-57` sums every `FineAccrued` forever, so a member who passes 50 is blocked permanently. | Model fine payment and waiver. |
| V3-16 | tool-lib | An item stays `Reserved` after the last reservation is cancelled. `ReservationCancelledEventHandler.cs:272-274` returns when nobody is next in line. | Release the item to Available when the queue is empty. |
| V3-17 | tool-lib | `MaintenanceRecordMapping` insert and update omit `CommunityId`, so the DB default of 1 applies. `LoanToolCommand` never compares the item's community with the member's. | Persist CommunityId and reject loans across communities. |
| V3-18 | autheo | (a) The community lookup runs inside an open DB transaction: `LoginHandler.cs:32-35` calls `UpdateAsync`, which locks the Users row, before an HTTP call to tool-lib with a 3 s timeout. A slow tool-lib therefore exhausts the connection pool. (b) A tool-lib outage silently drops `community_id`: `ToolLibraryCommunityResolver.cs:63-83` returns null with no log or cache. Combined with support's fallback to tenant 1, users can land in the wrong tenant. | Resolve the community before the first write, cache it with a short TTL, log failures, and make every service return 403 when the claim is missing. |
| V3-19 | autheo | The login limiter is per IP only, with no per-account counter or lockout (`AuthRateLimitPolicies.cs:33`). `KnownProxies` and `KnownIPNetworks` are cleared (`Program.cs:28-29`), so `X-Forwarded-For` is trusted from any peer. | Add a per-email failure counter, and restrict known networks to the gateway subnet. |
| V3-20 | communities | The anonymous list returns Pending, Suspended and Archived communities with contact email and phone unless a status filter is passed (`CommunityQueries.cs:125,141`). | Default to Active only, and use a public DTO without contact details. |
| V3-21 | support | `/api/uptime/incidents` is anonymous and returns the internal `Findings` and `AssignedTeam` (`Controllers.cs:30,43-50`, `Queries.cs:31-42`). | Use a public DTO. |
| V3-22 | gateway ↔ services | Route and policy contracts disagree: (1) `POST /api/tickets` is public at the gateway (`:81-84`) but has no `[AllowAnonymous]` in the service; (2) tool-lib `request-access` is `[AllowAnonymous]` in the service but blocked at the gateway (`:127-131`); (3) quick-register is public at the gateway but needs a moderator in the service. The SPA catch-all is public. Gateway tests cover only soup-kitchen. | Add a contract test that derives the expected route policies from controller attributes, and set the gateway FallbackPolicy to authenticated. |
| V3-23 | CI / packaging | The `Version="1.4.0"` pins are cosmetic. CI (`dotnet-ci.yml:32-57`) and every Dockerfile pack `tsqr-common` from main HEAD or the local checkout, so an unversioned common change silently reaches every service. Common CI runs only on push to main, so common PRs are untested. Reusable workflows are referenced `@main`. | Check out the common tag matching the pin, add a `pull_request` trigger, and pin workflow refs. |
| V3-24 | CI | Deploy CI only runs `docker compose config`, so the SQL is never executed. The `tsqr-support-ui` spec exists but CI never runs it (`test: true` is not set). `tsqr-ui` has no tests, and no repo has Postgres integration tests; that is why drift, `ON CONFLICT` and `xmin` behaviour go unnoticed. | Add a Postgres job that applies `infra/*.sql` twice, and run Testcontainers-based repository tests. |
| V3-25 | deploy | A known default admin password is documented (`tsqr-deploy/README.md:87,96`) and seeded (`infra/03-autheo.sql:56`), with no forced change. | Generate it on first run, or force a reset at first login. |
| V3-26 | sdlc | `tsqr-sdlc/README.md` disagrees with the code: line 41 claims kernel adoption in six repos (identity has not adopted it) and that support hardening shipped (R3-4 is open), while line 119 still lists support#6 as open. The backlog has no items for gateway default-allow, the OTLP collector, R3-4 or the outbox transport. | Reconcile the README with GitHub and file the missing items. |
| V3-27 | all .NET repos | NU1903: `Microsoft.OpenApi` 2.0.0 has a high-severity advisory (GHSA-v5pm-xwqc-g5wc). It comes in transitively through `TSQR.Common.WebApi` / `Microsoft.AspNetCore.OpenApi` 10.0.0. | Bump it in common and republish. |

### Low

| # | Context | Finding |
|---|---|---|
| V3-28 | autheo / common | JWKS is published and proxied, but no verifier consumes it. Common reads only static `Jwt:PublicKeys`, so a key rotation means redeploying 7 services in order. `JwksController.cs:14` sets no Cache-Control. |
| V3-29 | autheo | Login timing reveals which accounts exist. `LoginHandler.cs:26-27` returns before PBKDF2 for unknown users, and `ForgotPassword` does writes and sends email inline only for known users. |
| V3-30 | autheo / identity | An email change neither revokes refresh tokens nor refreshes the `email` claim in tokens already issued. |
| V3-31 | communities | A concurrent duplicate slug returns 500, because Postgres 23505 is not mapped to 409. `Community.UpdateDetails` (`Community.cs:82-88`) has no validation or status guard, so archived communities can still be edited. |
| V3-32 | support / tool-lib | Domain events are raised before the Id exists: `TicketCreated` has no TicketId (`Ticket.cs:138`), and `ReservationCreated.ReservationId` is 0. |
| V3-33 | tool-lib | Duplicate queue positions: the next position is the active count + 1, and `Complete()` does not shift the queue. `ReservationCancelledEventHandler.cs:265-266` rewrites terminal reservations too. |
| V3-34 | common | The shared kernel made `SetAssignedId` and `SetRowVersion` public; they were internal in the per-service copies. This weakens aggregate encapsulation. |
| V3-35 | tool-lib | Synchronous `ExecuteScalar` runs on the request thread in `SqlRepository.Update` and `InventoryItemRepository.Update` (`:109-134`). |
| V3-36 | soup-kitchen / deploy | There is no index on `Events(CommunityId)`, although every query filters on it (`infra/06`, `23-soup-kitchen-schema-hardening.sql`). |
| V3-37 | misc | `reset-password` and `refresh` have no endpoint rate limit (`AuthController.cs:63,149`). `JwtTokenService.ValidateAccessToken` is dead code. One `INTERNAL_API_KEY` is shared by identity→autheo and autheo→tool-lib. Dev connection-string passwords and HMAC keys are in `appsettings.Development.json` for communities and soup-kitchen, and a connection password is in identity's `appsettings.json:9`. No PEM private key is tracked anywhere. |

---

## 5. Test gaps worth closing alongside the fixes

- **tool-lib:**
  - nothing covers return → loan → return, `ToolReturnedEventHandler`, `ReservationCancelledEventHandler`, MarkForRepair, CompleteRepair, MarkAsLost, ConfirmPickup ordering or mark-not-returned;
  - there are only Domain and Infrastructure test projects, and none for Application or WebApi.
- **autheo / identity:**
  - forwarded headers combined with the limiter, end to end;
  - suspended or deleted users at login;
  - startup without RS256 keys (V3-1).
- **soup-kitchen:** concurrent guest adds returning 409 (the behaviour is correct but unproven).
- **gateway:** a route-contract test for every service (V3-22).
- **All services:** Postgres integration tests (V3-24).

---

## 6. Build and test results (2026-11-26, dotnet 10.0.401)

| Repo | Build | Tests | Notes |
|---|---|---|---|
| tsqr-autheo | ✅ | 63 / 63 | NU1903 |
| tsqr-identity | ✅ | 74 / 74 | Built against the local `../tsqr-common` source (ProjectReference) |
| tsqr-communities | ✅ | 89 / 89 | NU1903 |
| tsqr-soup-kitchen | ✅ | 99 / 99 | NU1903 |
| tsqr-support | ✅ | 101 / 101 | NU1903 |
| tsqr-tool-lib (HEAD) | ✅ (34 CS8618 warnings) | 88 / 88 | The working tree with the uncommitted diff also builds and passes |
| tsqr-common | ✅ | 43 / 43 | 10 DomainKernel + 33 WebApi (13 JWT) |
| tsqr-gateway | ✅ | 29 / 29 | NU1903 |

### Shared package usage

| Service | TSQR.Common | TSQR.Common.WebApi | How it is actually built |
|---|---|---|---|
| autheo, communities, soup-kitchen, support, tool-lib | PackageReference 1.4.0 | PackageReference 1.4.0 | Packed from common **source** in Docker and CI (V3-23) |
| identity | **ProjectReference** | **ProjectReference** | Common source HEAD |
| gateway | via WebApi | PackageReference 1.4.0 | Does not call `AddTsqrApiDefaults` |

---

## 7. Suggested order

1. **Close the security holes (small and local):**
   - V3-1: remove the HS256 fallback, then rotate and delete the identity key (closes R3-5);
   - R3-4 + V3-9 + V3-21: support reads;
   - V3-11: token logging;
   - V3-8: soup-kitchen permissions;
   - V3-7: community ownership.
2. **Fix the tool-lib circulation loop as one piece of work:**
   - R3-1, V3-3, V3-4, V3-6, V3-12, V3-13 and V3-16, then R3-7, R3-8 and R3-12;
   - add V3-14 (loan read model) so that the flows can be reached and tested end to end;
   - add V3-5 so that regular members can actually borrow.
3. **Data integrity:**
   - V3-2: remove the communities commit retry;
   - R3-19 / V3-24: tool-lib `init.sql` plus a Postgres CI job;
   - deploy#3: migration runner.
4. **Platform hardening:**
   - V3-10 / V3-19: forwarded headers and the limiter in common;
   - R3-14 / V3-22: gateway FallbackPolicy and a route-contract test;
   - V3-23: pin the common version in CI;
   - V3-27: bump `Microsoft.OpenApi`.
5. **IAM consolidation:**
   - R3-6 and R3-11: one source of truth for roles and suspension, ideally through the outbox, which would give R3-18 its first producer;
   - autheo#20: per-service audiences;
   - identity#6: adopt the shared kernel.
6. **Tenancy in tool-lib:** R3-9 and V3-17, using the same rule as soup-kitchen: the tenant comes from the claim, otherwise 403.
7. **Then the medium and low remainder, and reconcile the sdlc backlog (V3-26).**

*These findings come from reading and building the code. Please review them before filing issues or acting on them.*
