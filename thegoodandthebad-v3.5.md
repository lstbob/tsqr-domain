# TSQR (TownsSquare): Design Audit, Round 4.5 (v3.5)

**Audit date:** 2026-11-27. **Previous audits:** [`thegoodandthebad-v3.md`](./thegoodandthebad-v3.md) (round 4, 2026-11-26), [`thegoodandthebad-v2.md`](./thegoodandthebad-v2.md), [`thegoodandthebad.md`](./thegoodandthebad.md), [`post-analyi.txt`](./post-analyi.txt).
**Product overview:** [`townsquare.md`](./townsquare.md). **DDD model:** [`domain-driven-design.md`](./domain-driven-design.md).

**Method**

- We read `main` HEAD of every `tsqr-*` repo. No commit reached `main` of a code repo after the v3 audit. Only `tsqr-domain` (the v3 file) and `tsqr-sdlc` (runbook paths, cc94832) changed.
- Nine repos (autheo, common, communities, deploy, gateway, identity, soup-kitchen, support, tool-lib) have uncommitted work on branch `feat/service-audiences` for autheo#20 (per-service audiences, gateway token exchange). This audit does not count that work as shipped.
- This round did not re-run `dotnet build` or `dotnet test`. The v3 results (all green, NU1903 warning, 34 CS8618 warnings in tool-lib) still apply, because the code on `main` did not change.
- Four lenses: domain-driven design (D), C# and design patterns (T), infrastructure (I), security (S).
- The status of the v3 sets did not change: Round 2 = 11 fixed / 16 partial / 2 open. Round 3 = 1 / 6 / 16. V3-1 to V3-37 = 37 open, and 0 were filed as issues before this round.
- GitHub project 4 ("tsqr") before this round: 46 items: 41 Done (one is the test item tool-lib#39), 4 Backlog (identity#6, support#6, deploy#3, domain#1), 1 In progress (autheo#20, Sprint 11).
- New in v3.5: 38 findings (D-1 to D-10, T-1 to T-7, I-1 to I-10, S-1 to S-11). Some findings group open V3 or R3 items so that one issue closes them.
- Findings are numbered D-n, T-n, I-n, S-n. The column or line **Absorbs** names the open V3 or R3 items that the same fix closes.

**Legend:**
- Severity: Critical, High, Medium, Low.
- Paths are relative to the repo named in the finding.

---

## 1. Executive summary

**What changed since 2026-11-26:**
- No commit reached `main` of a code repo.
- `tsqr-domain` got the v3 file.
- `tsqr-sdlc` got the runbook path fixes (cc94832).
- Nine repos hold uncommitted work on `feat/service-audiences` for autheo#20. This audit does not count that work as shipped.

**Headline risks:**
1. **S-1:** The token signer and the token verifier both accept HS256 with a key that is committed to git.
2. **S-3:** Support shows ticket and incident data, including submitter emails, to anonymous callers.
3. **S-2:** The static role map in `tsqr-common` widens every token, so a permission removed in identity still passes.
4. **T-2:** The persistence code is copied six times, and the communities copy loses data (V3-2).
5. **I-1:** One PostgreSQL superuser serves all six databases, so one compromised service can read the others.

### Scorecard

| Set | Fixed | Partial | Open | Note |
|---|---|---|---|---|
| Round 2 (#1–#29) | 11 | 16 | 2 | unchanged since v3 |
| Round 3 (R3-1 – R3-23) | 1 | 6 | 16 | unchanged |
| V3 (V3-1 – V3-37) | 0 | 0 | 37 | first filed as issues in this round, through the v3.5 items |
| v3.5 (38 findings) | n/a | n/a | 38 | D 10, T 7, I 10, S 11 |

### Findings by lens and severity

| Lens | Critical | High | Medium | Low | Total |
|---|---|---|---|---|---|
| D (domain-driven design) | 0 | 3 | 4 | 3 | 10 |
| T (C# and design patterns) | 0 | 1 | 3 | 3 | 7 |
| I (infrastructure) | 0 | 4 | 3 | 3 | 10 |
| S (security) | 2 | 4 | 4 | 1 | 11 |
| **Total** | **2** | **12** | **14** | **10** | **38** |

---

## 2. Domain-driven design (lens 1)

<a id="d-1"></a>
### D-1. Four role models and none is the authority (High)

- **What happens:**
  - The autheo `RoleName` (`tsqr-autheo/src/TSQR.Autheo.Domain/Aggregates/UserAggregate/RoleName.cs`) goes into the token.
  - The identity `Role` and the persisted `user_roles` and role permissions (`tsqr-identity/src/TSQR.Identity.Domain/Entities/Role.cs`) never reach the token.
  - The static `RolePermissions` map in `tsqr-common/src/TSQR.Common/Authorization/RolePermissions.cs` decides in practice, because autheo calls `RolePermissions.ForRole(role)`.
  - The tool-lib `MembershipType` (Admin, LocationCoordinator ...) in `MembershipRecord` is used by `MemberVerificationService.CanVerifyMember`.
  - Roles are global. A LocationCoordinator is coordinator of every community, so V3-7 follows.
- **Fix:** Make identity the one authority for identity and access management (IAM). Define a role grant as (user `sub`, community, role). Put the permissions of the active community in the token. Write an Architecture Decision Record (ADR) first.
- **Absorbs:** R3-11. It is the root cause of V3-7.

<a id="d-2"></a>
### D-2. Community membership is owned by the wrong context (Medium)

- **What happens:**
  - autheo gets `community_id` from the Tool Library (the core subdomain) over synchronous HTTP (`ToolLibraryCommunityResolver.cs`, tool-lib `InternalMembersController.cs`). A tool-lib outage drops the claim.
  - Soup Kitchen `Volunteer.MemberId` is an `int` (`Volunteer.cs:6`) with no link to `sub`. A volunteer and a borrower are not the same neighbour.
  - A token has one `community_id` claim, so a user can belong to one community only.
- **Fix:** Let Communities (or a new Membership context) own "who belongs to which community". It publishes `MemberJoinedCommunity` and `MemberLeftCommunity` through the outbox. autheo keeps a local projection.
- **Absorbs:** This finding relates to DDD doc §12.7 and V3-18.

<a id="d-3"></a>
### D-3. Tool Library `Member` copies the person and has three overlapping status flags (Medium)

- **What happens:**
  - `Member.cs:5-55` stores FirstName, MiddleName, LastName, Age (int), Address, Email and PhoneNumber. The identity `UserProfile` owns the same data, and nothing keeps the two in sync. `Age` as an int goes stale.
  - `Status`, `IsVerified` and `AccessRequestStatus` together form an implicit state machine with invalid combinations. For example: Active, not verified, access Denied.
  - Primitive obsession: email, phone and address are plain strings. `CommunityId` is an `int`.
- **Fix:** Keep only the borrower facts and the `sub` link in `Member`. Add one `MembershipState` (Applicant → Verified → Suspended → Banned, with Banned as the terminal state). This closes R3-12. Add value objects for Email and Phone. Store a date of birth or an "age verified" fact instead of Age.

<a id="d-4"></a>
### D-4. Tenancy is not a model concept (High)

- **What happens:**
  - `CommunityId` is a raw `int` in every context (`Event.cs:11`, `Ticket.cs:26`, `Member.cs`). `tsqr-common` has no typed id for it.
  - Services read the claim in three different ways: soup-kitchen `Program.cs:27` and `CommunityControllerBase.cs:17`, support `CurrentUser.cs:45` (falls back to 1), and common `TsqrCorrelation.cs:89` (logging only). tool-lib and communities never read it.
  - Child entities take the tenant as a parameter and not from the root: `Event.AddGuest(name, contactInfo, guestCount, int communityId)` (`Event.cs:181`). A caller can pass a different tenant.
- **Fix:** Add a `CommunityId` value object and `ITenantContext` to `tsqr-common` (`TSQR.Common.WebApi`). Use one throw-site that returns 403 when the claim is missing. Let child entities copy the tenant from the root. Make repositories filter by tenant.
- **Absorbs:** R3-9 and V3-17 (tool-lib part). It supports R3-4.

<a id="d-5"></a>
### D-5. Soup Kitchen `Event` aggregate is too large (Medium)

- **What happens:**
  - `Event` owns Meals, Guests, Volunteers and Donations (`Event.cs:158-311`). Only the guest capacity is a true invariant across children.
  - Every donation or meal change loads the whole graph (N+1, `EventRepository.cs:23-63`) and bumps the Event `xmin`. Unrelated edits then conflict with 409.
- **Fix:** Keep Event and the guest registrations (capacity) together. Make `Donation` and `VolunteerShift` their own aggregates that reference `EventId`. Meal can stay or move. Decide in the issue.

<a id="d-6"></a>
### D-6. Dead or misplaced Tool Library model (Low)

- **What happens:**
  - `Location` and `Country` (tool-lib `Domain/Aggregates/LocationAggregate`) copy the Communities geography. They have no validation and are not aggregate roots. `LocationId` is still passed as null to Policy.
  - `MemberVerificationService` has 0 references. `INoSqlRepository` and `IDocumentRepository` have 0 implementations (speculative generality).
  - `FineService.CalculateFine` has an overload with a hard-coded 1.00 fee. `Loan.DefaultLateFeePerDay` (`Loan.cs:7`) is a second copy. Policy must be the only source of the fee.
- **Fix:** Delete the unused types. Reduce Location to a `PickupPointId` value or remove it. Take the fee only from Policy.

<a id="d-7"></a>
### D-7. Fines have no model (High)

- **What happens:**
  - Fines are `decimal` fields on Loan (`Loan.cs:37` `FineAccrued`). There is no Money value object, no currency, no payment and no waiver.
  - `DapperLoanRepository.cs:47-57` sums every `FineAccrued` forever. A member above 50 is blocked for life.
- **Fix:** Add a `FineAccount` (ledger) aggregate for each member, with the entries Accrued, Paid and Waived and a balance. Make eligibility read the balance. Add a `Money` value object.
- **Absorbs:** V3-15.

<a id="d-8"></a>
### D-8. Domain and integration events leak internals (Low)

- **What happens:**
  - `OutboxCrossDomainEventBus` stores `eventType.FullName` as the event type (`tsqr-common/src/TSQR.Common/Messaging/OutboxCrossDomainEventBus.cs`). A CLR namespace becomes the published language, and a rename breaks consumers. There is no version field.
  - Code raises events before the id exists (`TicketCreated` without TicketId, `ReservationCreated.ReservationId = 0`).
  - `DomainEvent.OccurredOn = DateTime.UtcNow` (`IDomainEvent.cs`) is not testable.
- **Fix:** Add explicit event names and a version to `IIntegrationEvent`. Raise "created" events after the id is assigned, or use client-generated ids. Take the time from a clock.
- **Absorbs:** V3-32.

<a id="d-9"></a>
### D-9. Domain rule 4 ("errors are values") is broken in two contexts (Low)

- **What happens:**
  - communities throws in 9 places (`CountryCode.cs:55`, `Community.cs:29`, `SlugGenerator.cs:18` and `:43`, `City.cs:14`, `Neighbourhood.cs:14`, `Location.cs:28`, `:33`, `:43`).
  - identity `PendingEmailChange` throws in 5 places (`PendingEmailChange.cs:29-84`).
  - `ConcurrencyConflictException` is copied into 5 Domain projects (autheo, communities, soup-kitchen, support, tool-lib).
- **Fix:** Make the factories return `Result<T>`. Keep one conflict exception in `tsqr-common`. Put the identity part into the existing identity#6.

<a id="d-10"></a>
### D-10. No process manager for time-based rules (Medium)

- **What happens:**
  - Nothing reads `Reservation.ExpiryDate`.
  - `ReturnReminderEvent` is never raised.
  - `NextInLine`, `PickupReminder` and `LoanOverdue` change state and notify nobody.
  - No scheduler exists.
- **Fix:** Add a scheduled background service in tool-lib (a process manager). It expires reservations, raises reminders and writes integration events to the outbox. This gives the outbox its first producer.
- **Absorbs:** R3-17 (expiry and reminders part) and R3-18 (first producer).

<a id="circulation"></a>
### Circulation loop (tool-lib, open since Round 3)

The circulation loop is open since Round 3 and we still did not file it. In DDD terms, the Loan life cycle and the Reservation queue are not closed inside aggregates. Handlers patch the state.

The open items are:

- R3-1
- V3-3
- V3-4
- V3-5
- V3-6
- V3-12
- V3-13
- V3-14
- V3-16
- V3-33

We file it as a tool-lib epic with 5 children.

---

## 3. C#, OOP and design patterns (lens 2)

<a id="t-1"></a>
### T-1. `Result` type is easy to misuse and every service maps errors by hand (Medium)

- **What happens:**
  - `Result` and `Result<T>` are structs. `default(Result<T>)` has `IsSuccess = false` and `Error = null`.
  - `Value` is readable on failure and returns `default!`.
  - `Error.None` is a `ValidationError` with empty strings.
  - There is no `Match`, `Map` or `Bind`. There is no `ConflictError`, `ForbiddenError` or `UnauthorizedError`.
  - There are 10 private error-to-HTTP mappers in 4 repos (autheo 1, identity 3, support 2, tool-lib 4). autheo `AuthController.ToError` maps everything except NotFound to 400.
- **Pattern and fix:** Make the Result a proper discriminated result (guard `Value`, add `Match`). Add the missing error types. Add one `ToActionResult()` or `ToProblem()` extension in `TSQR.Common.WebApi` with one mapping table.

<a id="t-2"></a>
### T-2. Persistence plumbing is copied six times and the copies drift (High)

- **What happens:**
  - `DapperUnitOfWork.cs` exists in autheo, communities, identity, soup-kitchen, support and tool-lib. `SqlRepository.cs` exists in communities, soup-kitchen and tool-lib. `DapperConnection.cs` exists in autheo, communities, identity, soup-kitchen and tool-lib.
  - The drift already caused data loss. The communities copy retries `Commit` on a new empty transaction (V3-2).
  - Synchronous Dapper calls run on the request thread: communities `SqlRepository.cs:42`, soup-kitchen `SqlRepository.cs:46` and `EventRepository.cs:103-119`, and tool-lib `SqlRepository.Update` (V3-35).
- **Pattern and fix:** Use Unit of Work and a generic Repository, published once as `TSQR.Common.Persistence`. Use async calls only, the `xmin` check, the affected-row check, no retry and one conflict exception. Adopt it per service in separate PRs.
- **Absorbs:** V3-2 (as a symptom) and V3-35 (extended).

<a id="t-3"></a>
### T-3. No clock abstraction (Low)

- **What happens:**
  - Domain projects call `DateTime.UtcNow` 37 times (autheo 14, tool-lib 9, identity 8, soup-kitchen 4, communities 1, support 1). Nothing uses `TimeProvider` (0 uses).
  - Tests cannot fix the time for due dates, expiry, reset tokens and fines.
- **Pattern and fix:** Inject `TimeProvider` (.NET 8 and later) in handlers. Pass `now` into aggregate methods. Prefer `DateTimeOffset` for new fields.

<a id="t-4"></a>
### T-4. Build settings do not enforce quality (Medium)

- **What happens:**
  - No repo has `Directory.Build.props` or `Directory.Packages.props`.
  - There is no `TreatWarningsAsErrors` and no analyzer level. NuGet audit only warns (NU1903 for `Microsoft.OpenApi` 2.0.0 passes CI).
  - tool-lib has 34 CS8618 warnings.
  - The Clean Architecture direction is a rule in `CLAUDE.md`, but no test checks it.
- **Pattern and fix:** Add `Directory.Build.props` (Nullable, `TreatWarningsAsErrors`, `AnalysisLevel` latest-recommended, `NuGetAuditLevel` moderate as error). Add central package versions. Add a reflection-based architecture test (no new library).

<a id="t-5"></a>
### T-5. Outbound HTTP has no resilience and the BFF hides failures (Medium)

- **What happens:**
  - No project uses `Microsoft.Extensions.Http.Resilience` or Polly. Only 3 clients set a timeout (autheo 3 s, identity 15 s, gateway WIP 5 s).
  - `BffController` (`tsqr-gateway/src/TSQR.Gateway.Api/Controllers/BffController.cs`) changes `DefaultRequestHeaders` and cuts the token with `Replace("Bearer ", "")`. It turns every downstream failure into zero stats, so an outage looks like an empty dashboard.
  - The gateway named clients fall back to `http://localhost:50xx` when the config is missing (`Program.cs`). A bad config then fails late.
- **Pattern and fix:** Use typed clients with `AddStandardResilienceHandler()`. Add a `DelegatingHandler` that forwards the token. Use options with `ValidateOnStart()`. Add a partial-response model that marks a failed section.

<a id="t-6"></a>
### T-6. Controllers hold application logic; cross-cutting checks are copied (Low)

- **What happens:**
  - identity `ProfileController.cs:88-128` writes files and calls `UserProfile.Create`, the repository and the unit of work directly.
  - The internal API key check exists twice: the tool-lib static `InternalApiKeyValidator` and the autheo private `IsInternalCaller` (`AuthController.cs:203-215`).
- **Pattern and fix:** Move the logic to Application command handlers (the existing `IInteractor`). Add one `InternalApiKey` authentication handler or policy in `TSQR.Common.WebApi`.

<a id="t-7"></a>
### T-7. Shared-kernel encapsulation (Low)

- **What happens:**
  - `Entity.SetRowVersion` and `Entity<TId>.SetAssignedId` are public (`Entity.cs`).
  - `ValueObject` compares `IEnumerable<object?>` (boxing) and offers a shallow `GetCopy()`.
  - Typed ids are classes in some contexts. A `readonly record struct` gives value equality for free.
- **Pattern and fix:** Move the two setters behind a persistence-only accessor. Allow `record`-based value objects. Remove `GetCopy`.
- **Absorbs:** V3-34.

---

## 4. Infrastructure, deployment and database (lens 3)

<a id="i-1"></a>
### I-1. One PostgreSQL superuser for all six databases (High)

- **What happens:**
  - Every `*_CONNECTION` in `.env.example` uses `Username=tsqr`, the `POSTGRES_USER` (a superuser). `01-create-dbs.sh` grants ALL on every database to that user.
  - A compromised soup-kitchen can read the autheo password hashes and refresh-token hashes. "Services never read another database" is a convention only.
- **Fix:** Create one login role for each service with CONNECT on its own database only. Add a separate owner role for schema changes. Use runtime roles without DDL.

<a id="i-2"></a>
### I-2. Containers run as root and are not hardened (High)

- **What happens:**
  - None of the 9 Dockerfiles sets `USER`. The .NET runtime images install `curl` with apt only for the health check.
  - compose sets `no-new-privileges` only. There is no `read_only`, no `cap_drop`, no memory or CPU limit and no `restart` policy.
- **Fix:** Use `USER $APP_UID` (the aspnet image has the user `app`), `USER node` and `nginxinc/nginx-unprivileged`. Set `read_only: true` with `tmpfs` for /tmp. Set `cap_drop: [ALL]`, limits and `restart: unless-stopped`. Add a health probe without curl.

<a id="i-3"></a>
### I-3. Ports open to the LAN and one flat network (Medium)

- **What happens:**
  - `5000:8080` (gateway), `3000:3000` (ui) and `4200:80` (support-ui) bind to all interfaces. Only Postgres binds to 127.0.0.1.
  - All 10 containers share `tsqr-net`. The UI containers can reach Postgres and every internal endpoint.
- **Fix:** Bind to 127.0.0.1 by default. Use the networks `edge` (gateway, ui, support-ui), `backend` (gateway and services) and `data` (services and postgres, `internal: true`).

<a id="i-4"></a>
### I-4. No TLS anywhere (Medium)

- **What happens:**
  - All traffic is plain HTTP. `RequireHttpsMetadata = false` says "TLS terminates at the ingress", but no ingress exists.
  - `UseHttpsRedirection()` runs with no HTTPS port. There is no HTTP Strict Transport Security (HSTS).
- **Fix:** Add a Transport Layer Security (TLS) terminating edge proxy (Caddy or Traefik) in compose. Set HSTS at the edge. Trust forwarded headers from the proxy.

<a id="i-5"></a>
### I-5. Supply chain is not pinned or scanned (High)

- **What happens:**
  - Base images float: `postgres:16`, `nginx:alpine`, `node:22-alpine`, `mcr.microsoft.com/dotnet/sdk:10.0` and `aspnet:10.0`.
  - Actions are pinned by tag (`@v4`). Reusable workflows are pinned by `@main`.
  - There is no Dependabot, no CodeQL, no image scan and no Software Bill of Materials (SBOM). Caller workflows give `packages: write` to pull-request runs.
- **Fix:** Pin images by digest and actions by SHA. Add Dependabot for docker, nuget, npm and github-actions. Add CodeQL for C# and JS and a Trivy image scan. Give `packages: write` only to the push job.
- **Absorbs:** V3-23 (workflow refs) and V3-27 (process part).

<a id="i-6"></a>
### I-6. Docker build context and image tags (Low)

- **What happens:**
  - Only tool-lib, tsqr-ui and tsqr-support-ui have `.dockerignore`. The .NET Dockerfiles copy `src/` with the host `bin/` and `obj/`. `COPY --from=common .` copies the whole `tsqr-common` checkout (with `.git`).
  - `tsqr-common` is packed from source inside each of 7 images. The images carry the mutable tag `:main`.
- **Fix:** Add `.dockerignore` everywhere. Use one shared common pack step (or restore from the GHCR feed at the pinned version). Use semver or SHA tags only.

<a id="i-7"></a>
### I-7. Database life cycle: no migrations, no backups, liveness depends on the database (High)

- **What happens:**
  - docker-compose.yml mounts 28 init scripts one by one. They run only on an empty volume. Every new script needs a compose edit and a manual retrofit.
  - There is no backup or restore. The only copy is the `pgdata` volume.
  - `/api/health` runs every check, including the database (`MapHealthChecks(options.HealthPath)` without a predicate). Compose uses it as the container health check. A short database outage marks every service unhealthy, and the gateway (`depends_on: service_healthy`) does not start.
- **Fix:** Do deploy#3 (migration runner). Add a backup job (scheduled `pg_dump` to a volume) and a tested restore runbook. Make liveness work without the database and readiness work with it.
- **Absorbs:** This finding links to deploy#3.

<a id="i-8"></a>
### I-8. Avatars live in the container layer (Low)

- **What happens:**
  - identity writes uploads to `Path.Combine(Directory.GetCurrentDirectory(), "avatars")` (`ProfileController.cs`). The identity service has no volume, so every rebuild deletes all avatars.
- **Fix:** Use a named volume now. Use object storage later.

<a id="i-9"></a>
### I-9. Telemetry has no receiver (Low)

- **What happens:**
  - Services export OTLP (the OpenTelemetry protocol) only if `OTEL_EXPORTER_OTLP_ENDPOINT` is set. compose sets it nowhere and runs no collector.
  - The gateway has its own OpenTelemetry and correlation code (`Program.cs:21-63`) and does not use the common defaults.
- **Fix:** Add an optional compose profile with an OpenTelemetry Collector and a trace and log backend. Make the gateway use the common telemetry setup.
- **Absorbs:** R3-19 (OTLP part) and the common#3 gap.

<a id="i-10"></a>
### I-10. Secrets and environment policy (Medium)

- **What happens:**
  - The RSA private key goes in as an environment variable (`Jwt__PrivateKeyBase64`). `docker inspect` shows it.
  - `ASPNETCORE_ENVIRONMENT` is `Production` for autheo, identity and support, `Docker` for the gateway, and the image default for the rest.
- **Fix:** Mount compose `secrets:` as files (`Jwt:PrivateKeyPath` and `Jwt:PublicKeyPath` already exist). Document one environment for each stack.
- **Absorbs:** This finding relates to V3-11.

---

## 5. Security (lens 4)

<a id="s-1"></a>
### S-1. HS256 fallback on both sides of the token (Critical)

- **What happens:**
  - Verifier: `TsqrJwtAuthenticationExtensions.cs:68-78` in `tsqr-common` accepts HS256 with `Jwt:Key` when no public key is set.
  - Signer (new in v3.5): autheo `JwtKeyMaterial.cs:26-43` signs HS256 with `Jwt:Key` when no private key is set.
  - identity ships a `Jwt:Key` in `src/TSQR.Identity.Api/appsettings.json:12` (on `main`, and in git history).
- **Fix:** Remove the symmetric path outside Development on both sides. Delete and rotate the identity key. Add a startup test that refuses to start without RSA keys.
- **Absorbs:** V3-1 and R3-5.

<a id="s-2"></a>
### S-2. The static role map widens every token (High)

- **What happens:**
  - `PermissionAuthorizationHandler` (`tsqr-common/src/TSQR.Common.WebApi/Authorization/PermissionAuthorizationHandler.cs`) succeeds if the `permission` claims contain the permission, or if the `role` claim maps to it in `RolePermissions`.
  - A permission removed from a role in identity still passes, because the role name alone grants it. Persisted permissions can add rights but never remove them.
- **Fix:** If a token carries any `permission` claim, do not use the role map. Remove the fallback after one refresh-token lifetime (7 days).

<a id="s-3"></a>
### S-3. Support exposes ticket and incident data (Critical)

- **What happens:**
  - Ticket list and detail are `[AllowAnonymous]` (support `Controllers.cs:130-143`). The gateway routes `support-tickets-read` and `support-tickets-get` with no policy. The DTOs carry `SubmitterEmail`.
  - With sign-in, any member reads every ticket in the community.
  - `CurrentUser.cs:46` falls back to community 1.
  - `/api/uptime/incidents` is anonymous and returns `Findings` and `AssignedTeam`.
- **Fix:** Require authentication for reads. Without `support.manage`, show own tickets only. Return 403 without the claim. Use a public incident DTO.
- **Absorbs:** R3-4, V3-9 and V3-21.

<a id="s-4"></a>
### S-4. Reset and confirmation tokens are logged in Production (High)

- **What happens:**
  - docker-compose.yml sets `Email__LogResetToken: "true"` and `Email__LogConfirmationToken: "true"` next to `ASPNETCORE_ENVIRONMENT: Production`. `LoggingEmailSender` writes the reset URL with the token.
- **Fix:** Refuse the flags outside Development at startup (autheo and identity). Remove them from compose.
- **Absorbs:** V3-11.

<a id="s-5"></a>
### S-5. Authorization holes inside a tenant (High)

- **What happens:**
  - soup-kitchen: plain `[Authorize]` on every controller. No soup-kitchen permission exists (V3-8).
  - communities: a LocationCoordinator can update any community (V3-7).
  - tool-lib: `AdminId` and `OwnerId` come from the body. Renew and return have no ownership check. `request-access` is anonymous (R3-10).
- **Fix:**
  - Add a `soupkitchen.manage` permission and require it on writes and guest reads. Bind volunteer sign-up to `sub`.
  - On a community update, require a platform admin or a `community_id` claim equal to the target id.
  - In tool-lib, take actor ids from the token, and guard renew and return with `AccessGuard`.
- **Absorbs:** V3-7, V3-8 and R3-10.

<a id="s-6"></a>
### S-6. Credential policy is below current guidance (Medium)

- **What happens:**
  - The hash is PBKDF2-HMAC-SHA256 with 100 000 iterations (`IPasswordHasher.cs:16`). The OWASP Password Storage Cheat Sheet (2023) asks for 600 000. The hash stores the iteration count, so a rehash on login is possible, but it is not built.
  - The minimum length is 6 (`RegisterHandler.cs:21`, `ChangePasswordHandler.cs:16`, `ResetPasswordHandler.cs:22`). NIST SP 800-63B asks for at least 8, and 15 for single-factor.
  - There is no per-account lockout. Reset and change do not revoke refresh tokens. The login timing shows which accounts exist.
- **Fix:** Use 600 000 iterations with a rehash on login. Set the minimum to 12 and add a block-list of common passwords. Add a per-account failure counter. Revoke all refresh tokens on reset and change. Use a constant-time login path.
- **Absorbs:** R3-13 (rest), V3-19 (account part) and V3-29.

<a id="s-7"></a>
### S-7. Avatar endpoint trusts its input (Medium)

- **What happens:**
  - `GET /api/profile/avatar/{userId}` is anonymous. It uses `userId` as a file search pattern (`Directory.GetFiles(avatarsDir, $"{userId}.*")`, `ProfileController.cs:131-152`). A pattern such as `*` returns the file of another user.
  - The upload checks only the file extension, not the content. The code never deletes an old avatar with another extension, so the wrong file can be served.
- **Fix:** Parse `userId` as a Guid. Check the magic bytes and re-encode the image. Keep one file for each user. Send `X-Content-Type-Options: nosniff`.
- **Absorbs:** This finding extends R3-20.

<a id="s-8"></a>
### S-8. No security response headers (Low)

- **What happens:**
  - The gateway, tsqr-ui (`next.config.ts` has no `headers()`) and the support-ui nginx config send no Content Security Policy (CSP), no `X-Content-Type-Options`, no `frame-ancestors`, no `Referrer-Policy` and no HSTS.
- **Fix:** Add one header middleware at the gateway. Set the same headers in nginx.

<a id="s-9"></a>
### S-9. One shared internal API key (Medium)

- **What happens:**
  - One `INTERNAL_API_KEY` protects identity→autheo, autheo→tool-lib and (in the WIP) the gateway→autheo token exchange. Any holder can call every internal endpoint. There is no rotation.
- **Fix:** Use per-caller credentials. Or use service tokens from autheo (client-credentials), now that per-service audiences exist.
- **Absorbs:** This finding extends V3-37.

<a id="s-10"></a>
### S-10. Personal data has no retention or erasure path (Medium)

- **What happens:**
  - The system stores Guest `ContactInfo`, Donation `DonorName`, Member `Identification`, address and phone, and ticket `SubmitterEmail` in plain text.
  - Soft delete keeps them forever. No context has an erasure command.
- **Fix:** Write a personal-data inventory for each context. Set a retention period. Add an erasure command. Add column encryption for identification numbers.

<a id="s-11"></a>
### S-11. The edge is allow-by-default and the rate limiter is global (High)

- **What happens:**
  - The gateway has no `FallbackPolicy` and no rate limiter (`Program.cs`). The route policies disagree with the services (V3-22).
  - The common limiter partitions by `RemoteIpAddress` (`TsqrApiDefaultsExtensions.cs:125`), and common never calls `UseForwardedHeaders`. All clients then share the gateway IP.
  - autheo clears `KnownProxies` and `KnownIPNetworks` (`Program.cs:28-29`), so it trusts `X-Forwarded-For` from any peer.
- **Fix:** Set the gateway FallbackPolicy to authenticated. Add a gateway rate limiter and a route-contract test. Add forwarded headers to `UseTsqrApiDefaults` and trust them from the gateway network only.
- **Absorbs:** R3-14, V3-10, V3-19 (proxy part) and V3-22.

---

## 6. Suggested order

1. Security P0: S-1, S-2, S-3 (and the gateway part of S-11), S-4. All are small.
2. Data integrity: the communities commit retry (V3-2, 1 SP), then the I-7 backups.
3. Circulation loop epic (tool-lib). Do the tenant context (D-4) first, so new code is tenant-safe.
4. Platform: I-1, I-2, I-3, I-5, the rest of S-11, and T-2.
5. IAM: write the D-1 ADR, then remove the S-2 fallback, then do D-2.
6. The remaining Medium and Low findings.

## 7. GitHub issues filed in this round

The issues are on GitHub project 4 (`tsqr`) with status Backlog, a priority, a size and an estimate in story points (SP). Every issue has the label `audit-v3.5`. Each child is 5 SP or less.

| Epic | Priority | SP | Children |
|---|---|---|---|
| [tsqr-domain#2](https://github.com/lstbob/tsqr-domain/issues/2) Security hardening | P0 | 44 | 16 |
| [tsqr-domain#3](https://github.com/lstbob/tsqr-domain/issues/3) Domain model (DDD) corrections | P1 | 68 | 12 |
| [tsqr-tool-lib#66](https://github.com/lstbob/tsqr-tool-lib/issues/66) Close the tool-lib circulation loop (child of the DDD epic) | P0 | 21 | 5 |
| [tsqr-domain#4](https://github.com/lstbob/tsqr-domain/issues/4) Infrastructure, deployment and database | P1 | 34 | 12 |
| [tsqr-domain#5](https://github.com/lstbob/tsqr-domain/issues/5) C# and design-pattern improvements | P2 | 26 | 9 |

The DDD epic also links the open identity#6. The infrastructure epic also links the open deploy#3.

| Issue | Finding | Priority | SP | Title |
|---|---|---|---|---|
| [tsqr-tool-lib#73](https://github.com/lstbob/tsqr-tool-lib/issues/73) | [Circulation](#circulation) | P0 | 5 | End the active loan and calculate the fine on return |
| [tsqr-tool-lib#74](https://github.com/lstbob/tsqr-tool-lib/issues/74) | [Circulation](#circulation) | P0 | 5 | Complete the reservation on loan and enforce the queue order |
| [tsqr-common#15](https://github.com/lstbob/tsqr-common/issues/15) | [D-4](#d-4) | P0 | 3 | Add a CommunityId value object and ITenantContext with 403 on a missing claim |
| [tsqr-gateway#12](https://github.com/lstbob/tsqr-gateway/issues/12) | [S-11](#s-11), [S-3](#s-3) | P0 | 5 | Deny by default at the gateway: FallbackPolicy, rate limiter and route-contract test |
| [tsqr-common#12](https://github.com/lstbob/tsqr-common/issues/12) | [S-1](#s-1) | P0 | 2 | Remove the HS256 fallback from shared JWT validation |
| [tsqr-identity#18](https://github.com/lstbob/tsqr-identity/issues/18) | [S-1](#s-1) | P0 | 1 | Delete and rotate the committed identity Jwt:Key |
| [tsqr-autheo#23](https://github.com/lstbob/tsqr-autheo/issues/23) | [S-1](#s-1) | P0 | 1 | Refuse to sign tokens with HS256 outside Development |
| [tsqr-common#13](https://github.com/lstbob/tsqr-common/issues/13) | [S-2](#s-2) | P0 | 2 | Stop the static role map from widening explicit token permissions |
| [tsqr-support#19](https://github.com/lstbob/tsqr-support/issues/19) | [S-3](#s-3) | P0 | 5 | Close ticket and incident data exposure and the community-1 fallback |
| [tsqr-deploy#29](https://github.com/lstbob/tsqr-deploy/issues/29) | [S-4](#s-4) | P0 | 2 | Stop logging reset and confirmation tokens in Production |
| [tsqr-communities#32](https://github.com/lstbob/tsqr-communities/issues/32) | [T-2](#t-2) | P0 | 1 | Remove the commit retry that commits an empty transaction |
| [tsqr-tool-lib#75](https://github.com/lstbob/tsqr-tool-lib/issues/75) | [Circulation](#circulation) | P1 | 5 | Guard repair and lost transitions and release the item when the queue is empty |
| [tsqr-tool-lib#76](https://github.com/lstbob/tsqr-tool-lib/issues/76) | [Circulation](#circulation) | P1 | 3 | Add a loan read model and return the loan id on create |
| [tsqr-tool-lib#77](https://github.com/lstbob/tsqr-tool-lib/issues/77) | [Circulation](#circulation) | P1 | 3 | Let regular members borrow with a tools.borrow permission |
| [tsqr-tool-lib#72](https://github.com/lstbob/tsqr-tool-lib/issues/72) | [D-10](#d-10) | P1 | 5 | Expire reservations and send reminders from a scheduled process manager |
| [tsqr-domain#7](https://github.com/lstbob/tsqr-domain/issues/7) | [D-1](#d-1) | P1 | 2 | ADR: one IAM authority and community-scoped role grants |
| [tsqr-autheo#26](https://github.com/lstbob/tsqr-autheo/issues/26) | [D-1](#d-1) | P1 | 5 | Issue token permissions from identity role grants of the active community |
| [tsqr-tool-lib#68](https://github.com/lstbob/tsqr-tool-lib/issues/68) | [D-4](#d-4) | P1 | 5 | Scope tool-lib reads and writes by tenant through ITenantContext |
| [tsqr-tool-lib#71](https://github.com/lstbob/tsqr-tool-lib/issues/71) | [D-7](#d-7) | P1 | 5 | Model fine payment and waiver as a fine ledger |
| [tsqr-deploy#30](https://github.com/lstbob/tsqr-deploy/issues/30) | [I-1](#i-1) | P1 | 3 | Use one least-privilege PostgreSQL role per service |
| [tsqr-deploy#40](https://github.com/lstbob/tsqr-deploy/issues/40) | [I-10](#i-10) | P1 | 3 | Mount secrets as files and use one environment policy |
| [tsqr-deploy#31](https://github.com/lstbob/tsqr-deploy/issues/31) | [I-2](#i-2) | P1 | 5 | Run every container as non-root with a hardened compose profile |
| [tsqr-deploy#32](https://github.com/lstbob/tsqr-deploy/issues/32) | [I-3](#i-3) | P1 | 2 | Bind published ports to localhost and split the compose network |
| [tsqr-deploy#34](https://github.com/lstbob/tsqr-deploy/issues/34) | [I-5](#i-5) | P1 | 5 | Pin images and actions, and add Dependabot, CodeQL and image scanning |
| [tsqr-deploy#37](https://github.com/lstbob/tsqr-deploy/issues/37) | [I-7](#i-7) | P1 | 3 | Add a database backup job and a tested restore runbook |
| [tsqr-common#14](https://github.com/lstbob/tsqr-common/issues/14) | [S-11](#s-11) | P1 | 3 | Trust forwarded headers only from the gateway network so the rate limiter is per client |
| [tsqr-soup-kitchen#33](https://github.com/lstbob/tsqr-soup-kitchen/issues/33) | [S-5](#s-5) | P1 | 3 | Add soup-kitchen permissions for writes and guest reads |
| [tsqr-communities#29](https://github.com/lstbob/tsqr-communities/issues/29) | [S-5](#s-5) | P1 | 2 | Restrict community updates to the coordinators of that community |
| [tsqr-tool-lib#67](https://github.com/lstbob/tsqr-tool-lib/issues/67) | [S-5](#s-5) | P1 | 3 | Take actor ids from the token and guard renew and return |
| [tsqr-autheo#24](https://github.com/lstbob/tsqr-autheo/issues/24) | [S-6](#s-6) | P1 | 5 | Raise the credential policy: PBKDF2 600 000, 12-character minimum, account lockout, revoke on reset |
| [tsqr-identity#19](https://github.com/lstbob/tsqr-identity/issues/19) | [S-7](#s-7) | P1 | 2 | Validate avatar ids and file content |
| [tsqr-common#18](https://github.com/lstbob/tsqr-common/issues/18) | [T-1](#t-1) | P1 | 3 | Harden Result<T> and add one Result-to-HTTP mapper |
| [tsqr-common#19](https://github.com/lstbob/tsqr-common/issues/19) | [T-2](#t-2) | P1 | 5 | Publish TSQR.Common.Persistence: async Dapper unit of work and xmin repository |
| [tsqr-deploy#35](https://github.com/lstbob/tsqr-deploy/issues/35) | [T-4](#t-4) | P1 | 2 | Fail CI on vulnerable packages and new warnings |
| [tsqr-communities#30](https://github.com/lstbob/tsqr-communities/issues/30) | [D-2](#d-2) | P2 | 5 | Make Communities the owner of community membership |
| [tsqr-tool-lib#69](https://github.com/lstbob/tsqr-tool-lib/issues/69) | [D-3](#d-3) | P2 | 5 | Reduce Member to borrower facts with one membership state machine |
| [tsqr-soup-kitchen#34](https://github.com/lstbob/tsqr-soup-kitchen/issues/34) | [D-5](#d-5) | P2 | 5 | Split Donation and volunteer shifts out of the Event aggregate |
| [tsqr-tool-lib#70](https://github.com/lstbob/tsqr-tool-lib/issues/70) | [D-6](#d-6) | P2 | 2 | Remove the dead tool-lib model: Location, Country, unused services and NoSQL contracts |
| [tsqr-common#16](https://github.com/lstbob/tsqr-common/issues/16) | [D-8](#d-8) | P2 | 3 | Give integration events stable names and versions |
| [tsqr-communities#31](https://github.com/lstbob/tsqr-communities/issues/31) | [D-9](#d-9) | P2 | 2 | Return Result instead of exceptions in the communities domain |
| [tsqr-deploy#33](https://github.com/lstbob/tsqr-deploy/issues/33) | [I-4](#i-4) | P2 | 3 | Add a TLS-terminating edge proxy with HSTS |
| [tsqr-deploy#36](https://github.com/lstbob/tsqr-deploy/issues/36) | [I-6](#i-6) | P2 | 2 | Add .dockerignore everywhere and build tsqr-common once per pipeline |
| [tsqr-common#17](https://github.com/lstbob/tsqr-common/issues/17) | [I-7](#i-7) | P2 | 2 | Split liveness and readiness health checks |
| [tsqr-deploy#38](https://github.com/lstbob/tsqr-deploy/issues/38) | [I-8](#i-8) | P2 | 1 | Persist identity avatars on a named volume |
| [tsqr-deploy#39](https://github.com/lstbob/tsqr-deploy/issues/39) | [I-9](#i-9) | P2 | 3 | Add an OpenTelemetry collector and a trace backend profile |
| [tsqr-domain#6](https://github.com/lstbob/tsqr-domain/issues/6) | [S-10](#s-10) | P2 | 3 | Write a personal-data inventory with retention and erasure rules |
| [tsqr-gateway#13](https://github.com/lstbob/tsqr-gateway/issues/13) | [S-8](#s-8) | P2 | 2 | Send security response headers at the edge |
| [tsqr-autheo#25](https://github.com/lstbob/tsqr-autheo/issues/25) | [S-9](#s-9) | P2 | 3 | Replace the shared internal API key with per-caller service credentials |
| [tsqr-tool-lib#78](https://github.com/lstbob/tsqr-tool-lib/issues/78) | [T-3](#t-3) | P2 | 3 | Inject TimeProvider and pass the time into aggregate methods |
| [tsqr-tool-lib#79](https://github.com/lstbob/tsqr-tool-lib/issues/79) | [T-4](#t-4) | P2 | 3 | Add Directory.Build.props and an architecture test, and fix the 34 CS8618 warnings |
| [tsqr-gateway#14](https://github.com/lstbob/tsqr-gateway/issues/14) | [T-5](#t-5) | P2 | 3 | Use typed resilient HTTP clients in the BFF and validate service URLs at startup |
| [tsqr-identity#20](https://github.com/lstbob/tsqr-identity/issues/20) | [T-6](#t-6) | P2 | 3 | Move avatar and profile logic from ProfileController into commands |
| [tsqr-common#20](https://github.com/lstbob/tsqr-common/issues/20) | [T-6](#t-6) | P2 | 2 | Provide one internal API key authentication handler |
| [tsqr-common#21](https://github.com/lstbob/tsqr-common/issues/21) | [T-7](#t-7) | P2 | 3 | Tighten shared-kernel encapsulation and allow record value objects |

*These findings come from reading the code on main. Review them before you act on them.*
