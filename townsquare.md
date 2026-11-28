# TownSquare (TSQR): what it is, the problem it solves, and why it matters

> **What this document is:** the starting point for anyone who wants to understand TownSquare as a product: what it does, who it is for, how it works and how mature it is.
> It is based on a scan of all 15 `tsqr-*` repositories as of **2026-11-19**.
>
> **Companion documents in this folder:**
> - [`domain-driven-design.md`](./domain-driven-design.md): how the system is modelled, from a DDD point of view.
> - [`thegoodandthebad.md`](./thegoodandthebad.md) and [`thegoodandthebad-v2.md`](./thegoodandthebad-v2.md): design audits, with a status update and Round 3 findings.
> - [`post-analyi.txt`](./post-analyi.txt): short list of what still needs fixing.
> - [`analysis.md`](./analysis.md): the original domain analysis from 2026-10-10 (historical).

---

## 1. TownSquare in short

**TownSquare is a digital toolkit for neighbourhoods.** It lets a local community share physical resources and organise mutual aid without each group having to build its own tools.

The platform has four parts:

| Part | What it does for a community |
|---|---|
| **Tool Library** ("Library of Things") | Neighbours put tools into a shared pool. Verified members reserve, borrow, renew and return them. Broken tools go to repair. Fair-use rules keep sharing fair. |
| **Soup Kitchen** | Volunteer-run community kitchens plan meal events, list the meals needed, register guests, schedule volunteers and record donations. |
| **Community Hub** | Every community gets a place on a real-world map (Country → City → Neighbourhood → Community), so it can be found, registered and confirmed. |
| **Learning** (`tsqr-learning`) | Members practise what they learn. Due cards are mixed across decks and reviewed on a spaced schedule. Each answer records a confidence rating, so members can see how well their confidence matches their results. |

Two supporting parts sit under these: a single **sign-in and identity** service shared by all features, and a **Support Portal** where users report problems and check service status.

**Mission.** In the words of the app: *"access to the right tool should never be a barrier"*. The broader idea is that neighbours who share what they own, and who feed and help each other, are cheaper, greener and more resilient than households that each fend for themselves.

### About the name

The name appears in two forms in the code:

- **"TownsSquare" / TownSquare.** Used in the deployment, process and analysis documents, and in the default admin account `admin@townssquare.local`. This is the name of the initiative: a digital town square where a neighbourhood meets.
- **"TSQR = Tool Sharing & Quick Reservation".** Used in the user interface (`tsqr-ui/app/layout.tsx`) and on the Support Portal's About Us page. This expansion comes from the first feature built, the tool library.

Both are correct. The tool library came first and is the most mature part. The soup kitchen and community hub widen the product into a general platform for neighbourhood mutual aid.

---

## 2. The problem it is trying to solve

### 2.1 For residents
- **Owned things sit idle.** Most households own drills, ladders, hedge trimmers and tile cutters that they use a few times a year. Buying them is expensive and storing them takes space. Throwing them away creates waste.
- **Borrowing from a neighbour depends on luck.** You have to know who owns what, feel comfortable asking, and trust that it comes back. Without structure, sharing stays within small circles of friends.
- **Help is hard to find.** People who need a meal, and people who want to cook, serve or donate, often cannot find each other.

### 2.2 For community organisers
Volunteer groups usually run on spreadsheets, paper sign-up sheets and group chats. That breaks down quickly. Organisers need to know:

- who has the tool now, and when it is due back;
- who is next in the queue;
- whether a borrower has a history of not returning things;
- how many guests are expected at Thursday's soup kitchen, how many portions are ready, and who is driving the donation run.

Every group that starts a tool library or community kitchen ends up rebuilding the same tools. TownSquare's answer is to **build them once, properly, and let any neighbourhood use them.** Each community's data stays separate from every other community's.

### 2.3 For the wider community
- **Sustainability:** fewer duplicate purchases, less e-waste, and longer tool life through tracked maintenance.
- **Affordability:** access instead of ownership. This matters most for low-income households.
- **Social fabric:** every loan, volunteer shift and shared meal is a contact between neighbours. Over time those contacts build trust, which is what lets communities cope with hard times.
- **Food security:** organised, reliable community kitchens.

---

## 3. What the app actually does

### 3.1 Tool Library (`tsqr-tool-lib`): the core

This is the most mature part. It follows the way real tool libraries and public lending libraries work, and the design notes cite Koha, Lend Engine, myTurn and the Berkeley and Oakland tool lending libraries as references.

**Catalogue and copies.** A **Tool** is the catalogue entry, for example "Bosch PSB 500 hammer drill". An **InventoryItem** is one physical copy of it, with a serial number, condition, current holder, loan count and usage time. A tool also records its **type**, which can be Hand, Power, Gardening, Construction, Specialty or Other.

Each tool also records two more things:
- its **amortisation rate** (how worn it is);
- **scarcity per location**, which shows where the neighbourhood is short of something.

**Membership.** To borrow, you must be a **Member**. The process is:
1. You **request access** and give an identification reference, so that the library knows who is borrowing.
2. A coordinator **approves** or **denies** the request.
3. Approved members are **verified** and can borrow.
4. Members can be **suspended**, **reinstated** or **banned**. A ban is meant to be permanent.

**Borrowing a tool:**

```
 Request access ─► Approved & verified member
                          │
     ┌────────────────────┴──────────────────┐
     ▼                                       ▼
 Tool available? ── yes ──► Loan (checkout) ──► Renew (if policy allows)
     │ no                                    │
     ▼                                       ▼
 Reserve ─► join the queue ─► next in line ─► Return (condition recorded)
             (position 1, 2, 3…)             │
                                             ├─► next reserver is notified
                                             └─► or mark for repair / mark lost
```

- **Reservations** form a **queue** for each item. When a tool is returned, or a reservation ahead of you is cancelled, the queue moves up and the next person is told it is their turn. Pickups are confirmed, activated and completed.
- **Loans** have a due date. They can be **renewed** a limited number of times. Late returns become **overdue** and add a **fine per day**.
- **Repairs.** A broken tool is **marked for repair**. This creates a maintenance record and cancels any reservations waiting for it. When the repair is finished the tool returns to circulation. Lost tools are marked as **lost**.
- **Eligibility.** You cannot borrow if you are not verified and active, if you owe more than **50** (currency units; the model does not set a currency) in fines, or if you have an overdue loan.

**Lending rules (policies)** are set per tool type. These are the seeded defaults:

| Tool type | Late fee / day | Loan period | Renewals | Reserve window |
|---|---|---|---|---|
| Hand | 0.50 | 7 days | 2 | 14 days |
| Power | 1.50 | 5 days | 1 | 10 days |
| Gardening | 0.75 | 7 days | 2 | 14 days |
| Construction | 0.75 | 7 days | 2 | 14 days |
| Specialty | 3.00 | 3 days | 0 | 7 days |
| Other | 0.50 | 7 days | 2 | 14 days |

Expensive or specialist equipment circulates faster and costs more when late. Simple hand tools are more relaxed. *(Known bug: the Specialty policy cannot currently be loaded. See §8.)*

### 3.2 Soup Kitchen (`tsqr-soup-kitchen`)

This helps a community run food-aid events:
- **Events:** a dated meal service at a location with a maximum number of guests. Status goes Planned → Active → Completed, or Cancelled.
- **Meals:** what needs to be cooked, by category (soup, main, side, dessert, beverage, bread), with quantity needed and quantity prepared.
- **Guests:** people registering to be fed, with a party size and an optional contact.
- **Volunteers:** members signing up as cook, server, cleaner, organiser, driver or other, with scheduled hours. A member cannot sign up twice for the same event.
- **Donations:** donor, item, quantity and date.
- **Dashboard:** totals for the community.

Guest and donor data is sensitive, because it shows who in the neighbourhood needs food aid. Since 2026-11-19 every soup-kitchen read requires a logged-in user, and data is scoped to the user's community.

### 3.3 Community Hub (`tsqr-communities`)

- **Geography:** a reference tree of Countries (with ISO-style two-letter codes), Cities and Neighbourhoods, maintained by platform admins.
- **Communities:** a named group in a neighbourhood with a URL-friendly **slug**, a description and contact details.
- **Quick register:** one step creates a community and any missing country, city or neighbourhood it needs.
- **Lifecycle:** a quick-registered community starts as *Pending confirmation*, and a platform admin **confirms** it. *Suspended* and *Archived* states exist in the model but cannot be reached through the API yet.
- **Discovery:** anyone can browse countries, cities and neighbourhoods and search communities.

### 3.4 Accounts, identity and permissions (`tsqr-autheo`, `tsqr-identity`)

- **One login for everything.** `tsqr-autheo` handles registration, login, refresh tokens, password reset and email change. It issues a signed token that every service trusts.
- **Profiles and administration.** `tsqr-identity` stores names, avatars and bios. It also handles suspension (with reason, who did it and until when), soft deletion, and roles with permissions.
- **The token tells each service:**
  - who you are (`sub`, a stable user ID);
  - your role;
  - your permissions;
  - your active community (`community_id`), looked up from your tool-library membership.

**Roles and what they can do:**

| Role | Typical person | Can… |
|---|---|---|
| **Regular** | Any neighbour | Reserve tools and manage their own reservations |
| **Repairman** | Handy volunteer | Everything a Regular can do, plus confirm pickups and operate tools (loan, return, repair, lost) |
| **LocationCoordinator** | Local organiser | Register and manage members' borrowing, manage the tool catalogue, complete reservations, manage communities, handle support |
| **Admin** | Platform operator | Everything, including users, roles, geography and loan administration |

### 3.5 Support Portal (`tsqr-support`, `tsqr-support-ui`)

- **Tickets:** bug reports, feature requests, access problems and general questions. Status goes Open → In progress → Resolved → Closed, and a resolved ticket can be reopened.
- **Incidents:** service outages, which go Investigating → Ongoing → Resolved, with findings.
- **Service status:** an uptime page and a contact form.
- Support staff work tickets and incidents. Users submit tickets and check whether something is down.

### 3.6 Web front-ends and plumbing
- **`tsqr-ui`** (Next.js): dashboard, tool catalogue with search and filters, and a tool detail page with scarcity by location.
- **`tsqr-support-ui`** (Angular): the Support Portal. At the moment it is a static mock-up that does not call the API.
- **`tsqr-gateway`** (YARP): the single entry point. It routes `/api/...` to the right service, checks sign-in, and offers a combined "BFF" dashboard endpoint.
- **`tsqr-deploy`**: Docker Compose for the whole stack, with one PostgreSQL database per service and shared CI workflows.
- **`tsqr-common`**: shared building blocks used by all services: results and errors, domain events, outbox messaging, the permission catalogue, and API hosting defaults (problem details, rate limiting, health checks, OpenTelemetry, correlation IDs).
- **`tsqr-sdlc`**: backlog, sprint process and issue definitions. The project is run by one developer in 2-week sprints with a cap of 5 story points.
- **`tsqr-tempalert`** is **not part of TownSquare**. It is an unrelated embedded-C learning exercise (a simulated temperature sensor).

---

## 4. Who it is for

| Persona | What they want | What TownSquare gives them |
|---|---|---|
| **Resident / borrower** | Use a tool without buying it | Browse the catalogue, reserve, borrow, renew, return |
| **Tool donor / owner** | Put their idle tools to use and know they will come back | Register tools; loans, holders and condition are tracked |
| **Repair volunteer** | Keep the shared stock working | Repair queue; tools go out of circulation while broken |
| **Community coordinator** | Run the library or kitchen without spreadsheets | Member approval, policies, queues, event planning, volunteer rosters, donation log |
| **Person in need** | A warm meal, with discretion | Guest registration for soup-kitchen events, with data kept private to the community |
| **Platform operator** | Onboard new neighbourhoods safely | Geography, community confirmation, users and roles, support and incidents |

---

## 5. Why it matters for local communities

1. **It turns private surplus into a common resource.** A drill used twice a year by one household can be used every week by twenty. The money stays in the neighbourhood, and fewer things end up in landfill.
2. **It builds trust on purpose.** Verification, clear lending rules, fines and repair tracking replace "do I trust this person?" with "the community's rules protect both of us". People can then lend to neighbours they have never met.
3. **It treats mutual aid as infrastructure.** Food aid and tool sharing are run with the same reliability as any other service: queues, capacity, rosters and audit trails. Volunteers stop burning out on admin.
4. **It respects dignity and privacy.** Guests, donors and members are sensitive groups. Every community is kept separate, and sensitive reads require a login.
5. **It can be repeated.** One platform, many communities: a new neighbourhood can register and be running without writing any software.
6. **It is a base for more.** Planned extensions include an impact dashboard (CO₂ and money saved), trust and reputation scores, notifications, payments and multi-location transfers. These turn local sharing into evidence that funders and municipalities can act on.

---

## 6. How it fits together (one picture)

```
                      Browser (residents, coordinators, support staff)
                         │                         │
                   tsqr-ui (Next.js)       tsqr-support-ui (Angular)
                         └──────────┬──────────────┘
                                    ▼
                         tsqr-gateway (YARP, :5000)
          ┌─────────┬──────────┬────┴─────┬───────────┬──────────────┐
          ▼         ▼          ▼          ▼           ▼              ▼
     tsqr-autheo tsqr-identity tsqr-tool-lib tsqr-soup-kitchen tsqr-communities tsqr-support
     (login,    (profiles,    (library of  (meal events,    (geography,   (tickets,
      tokens)    roles)        things)      volunteers)      communities)  incidents)
          │                       ▲
          └── asks tool-lib "which community is this user in?" (internal API)

     Every service has its own PostgreSQL database and shares tsqr-common.
     Everything is .NET 10 + Dapper, using Clean Architecture and Domain-Driven Design.
```

---

## 7. How mature it is (status on 2026-11-19)

**The back end is well engineered. The product surface is thin.**

- **Back end.** Between 3 and 19 November 2026 around 40 audit issues were shipped. They covered:
  - concurrency control;
  - schema hardening and soft deletes;
  - pagination;
  - permission-based authorisation;
  - tests and CI;
  - structured logging and tracing;
  - proper aggregates for support, identity and the soup kitchen.

  The tool library is a strong domain model.
- **Front end.** Only the tool catalogue (list and detail) and a dashboard are built. The Members, Reservations, Inventory, Maintenance and Loans screens are placeholders, and the Support Portal is a static mock-up.
- **Notifications.** Events such as "you're next in line" and "your tool is due tomorrow" are raised inside the system, but nothing sends them to people yet. There is no email, SMS or push transport.
- **Messaging between services.** An outbox pipeline exists, but no events flow through it yet.
- **Open work.** About 18 backlog items remain, including:
  - shared DDD base classes;
  - a database migration runner;
  - RS256 token signing and per-service audiences;
  - full multi-tenancy for the soup kitchen;
  - state-machine guards in communities and the soup kitchen;
  - support comments and attachments.

## 8. Known problems worth knowing before a demo

These were found in this review. They are detailed in [`thegoodandthebad-v2.md` § Round 3](./thegoodandthebad-v2.md#round-3--new-findings-2026-11-19):

- **Returning a tool does not close the loan.** The loan stays active, no fine is calculated, and once the due date passes the borrower is blocked as "overdue".
- **Access requests are not saved**, so approving or denying a membership request after a reload fails.
- **The Specialty tool policy cannot be loaded.** It allows 0 renewals, but the model requires 1–2. Specialty tools therefore cannot be loaned or reserved.
- **Soup kitchen:**
  - Editing an event can move it into another community.
  - The uncommitted community-scoping work in the working tree does not compile.
- **Support:** the public ticket list shows submitter email addresses, and users without a community claim are treated as community 1.
- **Identity:**
  - Suspending a user in identity does not stop them logging in.
  - A JWT signing key is committed in identity's `appsettings.json`.

---

## 9. Glossary (business terms)

| Term | Meaning |
|---|---|
| **Community** | A group of neighbours in one neighbourhood. Its data is kept separate from other communities. |
| **Member** | A verified person in a community's tool library. |
| **Tool / Item** | Catalogue entry / physical copy. |
| **Loan, due date, renewal, overdue, fine** | Borrowing lifecycle. |
| **Reservation, queue position, next in line, pickup** | Waiting-list lifecycle. |
| **Policy** | Lending rules per tool type: fee, loan period, renewals, reservation window. |
| **Repair / maintenance record** | Takes a broken copy out of circulation until it is fixed. |
| **Scarcity** | How short a location is of a type of tool. |
| **Event, meal, guest, volunteer, donation** | Soup-kitchen vocabulary. |
| **Quick register, confirm** | Creating a community, and an admin approving it. |
| **Role, permission** | What a person is allowed to do. |
| **Ticket, incident** | A user's problem report, and a service outage. |
