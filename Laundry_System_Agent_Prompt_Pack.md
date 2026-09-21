# Laundry Queuing & Customer Notification System — Agent Prompt Pack

A phase-by-phase prompt system for building the system in your deck (`Laundry_Queuing_and_Notification_System.pptx`) with an AI coding agent, one function at a time, with guardrails against invented requirements.

---

## 0. How this pack works

**The problem it solves.** If you give an agent the whole deck in one prompt, it fills every gap with guesses (tech stack, pricing, roles, SMS provider, queue rules, and so on) and reports success without running anything. This pack prevents that with four mechanisms:

1. **A Source of Truth (Section 1)** — facts copied from your deck, each with an ID (`SRC-xx`) the agent must cite.
2. **A Decision Register (Section 2)** — every rule the deck does *not* specify, resolved in advance (`D-xx`). Anything the agent still can't trace, it must ask about or log as an assumption. It may not invent it.
3. **A Master Prompt (Section 3)** — standing rules: trace everything, verify every library, prove every claim by running it, stay inside the phase.
4. **16 phase prompts (Section 4, P0 to P15)** — one function per prompt, each with scope, steps, acceptance criteria, and a stop gate.

**Where the content comes from — please read.**
- Section 1 is taken from your deck. It is the only place requirements come from.
- Section 2 contains decisions **I proposed** to fill gaps the deck leaves open. They are defaults, not facts from your deck. Edit any you disagree with *before* starting. Six fields are marked **USER FILL** because I can't decide them for you.
- Section 2.3 lists places where your deck's ERD is missing things its own diagrams require (users, notification log, settings). These need your approval.

### Setup (5 minutes)
1. Fill in the six **USER FILL** items in Section 2 (or leave blank and read the note on each).
2. In your project folder, save Section 1 as `docs/00_SOURCE_OF_TRUTH.md` and Section 2 as `docs/01_DECISIONS.md`. If your agent can't read files, paste both into the chat instead.
3. Give the agent the **Master Prompt (Section 3)** as its standing instructions (system prompt / project rules). If your tool doesn't keep standing instructions, paste it at the top of every phase.
4. Send the phase prompts **one at a time**, P0 first. The agent finishes with a *Phase Report* and stops.
5. Check the report (checklist below). If good, reply exactly `APPROVED P#`, then send the next phase. If not, reply with what failed. Don't advance.

### How to check a Phase Report (2 minutes)
- Does every item have a source ID? Anything without one is invented.
- Is there real command output, not just "tests pass"?
- Is "Deviations" empty or explained? Is "NOT RUN" honest?
- Did it add anything from the Out-of-Scope list (Section 2.5)?

Recovery prompts for when an agent drifts are in Section 5.

---

## 1. Source of Truth — save as `docs/00_SOURCE_OF_TRUTH.md`

> Extracted from the deck. Wording is kept close to the slides. Nothing here was added by the prompt author except where marked **[INTERPRETED]**.

### SRC-01 — System identity (slide 1)
- Name: **Laundry Queuing & Customer Notification System**
- Purpose: replace manual laundry tracking and customer calling with **real-time status, queue management, and notifications**.

### SRC-02 — Problems being solved (slide 2)
| ID | Problem | Deck text |
|---|---|---|
| SRC-02a | Manual recording | Orders and customer details are written or tracked manually. |
| SRC-02b | Manual queue | Staff must determine which laundry should be processed next. |
| SRC-02c | Status uncertainty | Customers have to ask whether their clothes are washing, drying, or finished. |
| SRC-02d | Manual calling | Staff must contact customers one by one when laundry is ready. |
| SRC-02e | Hard to monitor | Owner/staff have limited real-time visibility of active and unclaimed orders. |

### SRC-03 — Solution pillars and status meanings (slide 5)
Three pillars: **Queue Number**, **Live Status**, **Auto Notification**.

| Status label | Deck meaning |
|---|---|
| Waiting | Order received |
| Washing | Laundry is being washed |
| Drying | Laundry is being dried |
| Ready for Pickup | Customer is notified |
| Claimed | Transaction completed |

### SRC-04 — Proposed system context diagram, Level 0 (slide 6)
- System: **Laundry Queuing & Notification System**
- External entities (4): **Customer**, **Admin / Owner**, **Laundry Staff**, **SMS / Email Notification**
- Flow labels present on the slide: `orders / requests` (Customer side), `reports / settings` and `analytics / records` (Admin/Owner side), `status / queue` (Laundry Staff side).
- The arrow directions are **not stated** in the deck text. **[INTERPRETED]** (confirm in P1):
  - Customer → system: orders / requests
  - Admin/Owner → system: settings; system → Admin/Owner: reports, analytics/records
  - Staff → system: status updates; system → Staff: queue
  - System → SMS/Email entity: notification requests

### SRC-05 — Proposed Level-1 DFD (slide 7)
Processes (7): `1.0 Manage Customer`, `2.0 Create Order`, `3.0 Manage Queue`, `4.0 Update Status`, `5.0 Send Notification`, `6.0 Payment & Claim`, `7.0 Generate Reports`.
Data stores (5): `D1 Customers`, `D2 Orders`, `D3 Queue`, `D4 Status History`, `D5 Payments`.
External entities on the slide: `Customer`, `Staff/Admin`.
Process→store access is **not readable** from the deck text. **[INTERPRETED]** from names:
| Process | Reads / writes |
|---|---|
| 1.0 Manage Customer | D1 |
| 2.0 Create Order | D1 (read), D2 (write), D3 (queue position), D4 (initial status), D5 (payment record) |
| 3.0 Manage Queue | D2, D3 (read) |
| 4.0 Update Status | D2, D4 (write) |
| 5.0 Send Notification | D1, D2 (read); triggered by D4 change |
| 6.0 Payment & Claim | D5 (write), D2, D4 (write) |
| 7.0 Generate Reports | D2, D4, D5 (read) |

### SRC-06 — Status flow (slide 8)
`WAITING → WASHING → DRYING → READY FOR PICKUP → CLAIMED`
Rule stated on the slide: **"Automatic notification is triggered when status = READY FOR PICKUP."**

### SRC-07 — ERD, "Core database structure" (slide 9)
| Table | Columns (exact names from deck) |
|---|---|
| CUSTOMER | PK `customer_id`, `name`, `contact_number`, `email` |
| LAUNDRY_ORDER | PK `order_id`, FK `customer_id`, `queue_number`, `status`, `total_amount` |
| LAUNDRY_ITEM | PK `item_id`, FK `order_id`, FK `service_id`, `weight`, `price` |
| SERVICE | PK `service_id`, `service_name`, `price` |
| STATUS_HISTORY | PK `status_id`, FK `order_id`, `status`, `date_time` |
| PAYMENT | PK `payment_id`, FK `order_id`, `amount`, `payment_status` |
Relationship lines are drawn but **cardinalities are not labelled**. **[INTERPRETED]:** CUSTOMER 1–N LAUNDRY_ORDER; LAUNDRY_ORDER 1–N LAUNDRY_ITEM; SERVICE 1–N LAUNDRY_ITEM; LAUNDRY_ORDER 1–N STATUS_HISTORY; LAUNDRY_ORDER 1–1 PAYMENT (see D-08).

### SRC-08 — Intended benefits (slide 10) — used as non-functional intent
| ID | Benefit | Deck text |
|---|---|---|
| SRC-08a | Faster service | Orders are organized in a clear queue. |
| SRC-08b | Less manual calling | Customers receive a ready-for-pickup notification. |
| SRC-08c | Real-time visibility | Staff can see every active laundry order and its status. |
| SRC-08d | Better customer experience | Customers can check progress instead of repeatedly asking staff. |
| SRC-08e | Accurate records | Orders, payments, and status history are stored digitally. |
| SRC-08f | Useful reports | Owner can monitor sales, completed orders, and unclaimed laundry. |

### SRC-09 — Suggested core features (slide 11) — **this is the v1 scope**
Deck note: *"Keep the first version focused and realistic."*
| ID | Feature |
|---|---|
| SRC-09a | Customer registration / customer information |
| SRC-09b | Laundry order creation and queue number |
| SRC-09c | Order status: Waiting → Washing → Drying → Ready → Claimed |
| SRC-09d | Customer status tracking |
| SRC-09e | Automatic ready-for-pickup notification |
| SRC-09f | Payment and claim recording |
| SRC-09g | Admin dashboard and basic reports |
| SRC-09h | Unclaimed laundry monitoring |

### SRC-10 — Existing (as-is) manual system (slides 3–4) — **context only, do NOT build**
As-is DFD processes: 1.0 Receive Laundry, 2.0 Record Order, 3.0 Manage Queue, 4.0 Wash & Dry, 5.0 Check Completion, 6.0 Contact Customer, 7.0 Claim & Payment. Stores: D1 Customer Records, D2 Laundry Orders, D3 Payment Records.
Use only to understand what is being replaced.

### SRC-11 — Closing statement (slide 12)
"The proposed system connects the queue, laundry status, customer notification, payment, and records into one workflow."

---

## 2. Reconciliation & Decision Register — save as `docs/01_DECISIONS.md`

> **Status legend:** `DEFAULT` = my proposal, change it if you disagree. `USER FILL` = you must supply it. The agent treats every entry here as approved once you start P0.

### 2.1 Feature map (deck features → build units)
| Unit | Feature | Source | DFD process | Phase |
|---|---|---|---|---|
| E-01 | Authentication & roles (Staff/Admin) | derived from SRC-04, SRC-05 | (all) | P4 |
| E-02 | Service catalogue & settings | derived from SRC-07 SERVICE, SRC-04 "settings" | (support) | P5 |
| F-01 | Customer registration / information | SRC-09a | 1.0 | P6 |
| F-02 | Order creation & queue number | SRC-09b | 2.0 | P7 |
| E-03 | Queue view (manage queue) | SRC-05 (3.0), SRC-02b, SRC-08a, SRC-08c | 3.0 | P8 |
| F-03 | Order status flow & history | SRC-09c, SRC-06 | 4.0 | P9 |
| F-05 | Automatic ready-for-pickup notification | SRC-09e, SRC-06 | 5.0 | P10 |
| F-04 | Customer status tracking | SRC-09d, SRC-08d | (customer-facing) | P11 |
| F-06 | Payment and claim recording | SRC-09f | 6.0 | P12 |
| F-08 | Unclaimed laundry monitoring | SRC-09h, SRC-02e | (support of 7.0) | P13 |
| F-07 | Admin dashboard & basic reports | SRC-09g, SRC-08f | 7.0 | P14 |

Units starting with **E** are enabling units required by the deck's diagrams but not listed on slide 11.

### 2.2 Decisions

**USER FILL (6 items)**
| ID | Topic | Value |
|---|---|---|
| D-01 | Tech stack (language, framework, database) | `USER FILL: ______`. If blank, the agent must **stop in P2** and offer 2–3 options for you to choose. It may not pick one itself. |
| D-09a | SMS provider | `USER FILL: ______`. If blank: **Mock adapter only**. The agent must not integrate any real provider. |
| D-09b | Email delivery (SMTP host/service) | `USER FILL: ______`. If blank: **Mock adapter only**. |
| D-19a | Shop timezone (IANA name) | `USER FILL: ______` |
| D-19b | Currency (ISO 4217 code) | `USER FILL: ______` |
| D-20 | Shop name (shown in UI and messages) | `USER FILL: ______` |

**DEFAULTS (my proposals — edit freely)**

| ID | Topic | Decision |
|---|---|---|
| D-02 | Platform | Responsive **web app**: staff/admin interface + a public customer tracking page. No native mobile app. |
| D-03 | Roles & permissions | Two roles: **ADMIN** (= Owner) and **STAFF**. Customers have **no accounts**. Matrix below. |
| D-04 | Customer tracking lookup | Public page. Customer enters **queue number + contact number**; both must match. Shows only: queue number, status label + meaning text (SRC-03), time of last status change. Any mismatch → identical generic "not found" response. Shows the **latest** matching order (highest queue_date). |
| D-05 | Queue number | Integer that **resets to 1 each business day** (day boundary = shop timezone). Displayed zero-padded to 3 digits (e.g. `007`). Unique per (`queue_date`, `queue_number`). Assigned atomically inside the order-creation transaction. |
| D-06 | Pricing | Per-kg. `SERVICE.price` = price per kg (2 decimals). `LAUNDRY_ITEM.weight` = kg, up to 2 decimals, > 0. `LAUNDRY_ITEM.price` = **line amount** = weight × service price at order time, rounded **half-up to 2 decimals**, stored as a snapshot (later service price changes never alter existing items). `LAUNDRY_ORDER.total_amount` = sum of item prices. An order needs ≥ 1 item. Use **decimal arithmetic, never binary floats**. |
| D-07 | Status transitions | Exactly `WAITING → WASHING → DRYING → READY_FOR_PICKUP → CLAIMED`. **Forward only, one step at a time.** No skipping, no reverting, no cancelling in v1. Every transition, including the initial `WAITING` at order creation, writes a `STATUS_HISTORY` row **in the same transaction**. |
| D-08 | Payment | **Cash, recorded manually.** Exactly **one PAYMENT per order**, created `UNPAID` (amount = total_amount) inside the order-creation transaction. Staff marks it `PAID` at any time. **`CLAIMED` requires `PAID`.** No partial payments, refunds, or online gateway. |
| D-10 | Notification behavior | Sent **automatically once**, when an order enters `READY_FOR_PICKUP` (SRC-06). **No automatic notifications for any other status.** Runs **after** the status change commits; notification failure **never** blocks or rolls back a status change. Up to **3 attempts** total (delays: 1 min, then 5 min; configurable constants). Result recorded per channel in `NOTIFICATION`. Staff can **manually resend** (recorded as a new row with trigger `MANUAL`). Auto-send is **idempotent**: at most one AUTO notification per (order, channel). Templates are fixed in code in v1. |
| D-10m | Message text | SMS/Email body: `Hi {customer_name}, your laundry (Queue #{queue_number}) at {shop_name} is ready for pickup.` Email subject: `Your laundry is ready for pickup - Queue #{queue_number}`. |
| D-09c | Channels | SMS and Email, via provider-agnostic adapter interfaces. `contact_number` is required → SMS always attempted. `email` is optional → Email attempted only if present. |
| D-11 | Unclaimed laundry | "Unclaimed" = status `READY_FOR_PICKUP`. **Ready time** = timestamp of that order's `READY_FOR_PICKUP` row in `STATUS_HISTORY`. **Overdue** when (now − ready time) **>** `unclaimed_threshold_days` × 24 h. Threshold is a setting, default **3**, ADMIN-editable. |
| D-12 | Reports (ADMIN only) | (a) **Sales**: sum of `PAID` payment amounts per day over a date range, by `paid_at`. (b) **Completed orders**: orders whose `CLAIMED` history time falls in the range (count + list). (c) **Unclaimed laundry**: as D-11 with days waiting. Dashboard: active orders per status, today's sales, today's completed count, overdue-unclaimed count. **No CSV/PDF export** in v1. |
| D-13 | "Real-time" | **Polling**, default every 10 s, for staff queue view and public tracking page. No websockets/SSE in v1. |
| D-14 | Customer records | Created by staff (no self-registration). `contact_number` **unique** after normalization: strip spaces, dashes, parentheses; keep an optional leading `+`; 7–15 digits. `email` optional, format-validated if given. Search by partial name or contact number. Edit allowed. Deactivate/reactivate instead of delete. |
| D-15 | No hard deletes | Customers, services, users, orders, items, payments, history and notifications are **never hard-deleted**. Services and users use `is_active`. An inactive service cannot be added to new orders but stays on old items. |
| D-16 | Queue ordering | Strict FIFO by (`queue_date`, `queue_number`) across all active orders. No manual reordering or priority. **Active** = status ≠ `CLAIMED`. "Next to process" = oldest `WAITING` order. `D3 Queue` is a **derived view** of `LAUNDRY_ORDER`, not a separate table. |
| D-17 | Order editing | Not supported after creation in v1. |
| D-18 | Security baseline | Passwords hashed with a modern password hash (library verified per R3). Server-side validation of every input. Authorization enforced **server-side** per D-03. Secrets only in environment variables. No personal data in logs. Basic rate limiting on login and on the public tracking lookup. Default-deny on all non-public routes. |
| D-19c | Time storage | Store all timestamps in **UTC**; convert to shop timezone (D-19a) only for display and for day-boundary logic (queue_date, report days). |

**D-03 permission matrix**
| Capability | ADMIN | STAFF | Public |
|---|---|---|---|
| Manage users | ✔ | ✘ | ✘ |
| Manage services & settings | ✔ | ✘ | ✘ |
| Create / search / edit / deactivate customers | ✔ | ✔ | ✘ |
| Create orders | ✔ | ✔ | ✘ |
| View queue view & unclaimed list | ✔ | ✔ | ✘ |
| Advance status | ✔ | ✔ | ✘ |
| Record payment / claim | ✔ | ✔ | ✘ |
| Resend notification | ✔ | ✔ | ✘ |
| Dashboard & reports | ✔ | ✘ | ✘ |
| Tracking lookup (D-04) | – | – | ✔ |

### 2.3 ERD reconciliation

**Deck ERD** (SRC-07) is implemented **exactly**, with the same table/column names in snake_case (`customer`, `laundry_order`, `laundry_item`, `service`, `status_history`, `payment`).

**Required additions** — the deck's own diagrams need these but the ERD omits them. Approved by default; say so in P0 if you object.
| ID | Addition | Why it's required |
|---|---|---|
| ADD-1 | Table `app_user` (`user_id` PK, `name`, `username` unique, `password_hash`, `role` enum ADMIN/STAFF, `is_active`) | Staff/Admin actors exist (SRC-04, SRC-05) and actions must be attributable. |
| ADD-2 | Table `notification` (`notification_id` PK, `order_id` FK, `channel` enum SMS/EMAIL, `recipient`, `trigger` enum AUTO/MANUAL, `status` enum PENDING/SENT/FAILED, `attempts`, `last_error` nullable, `next_attempt_at` nullable, `sent_at` nullable) | DFD 5.0 and the SMS/Email entity (SRC-04, SRC-05); SRC-08b needs proof a customer was notified. |
| ADD-3 | Table `setting` (`key` PK, `value`) holding `shop_name`, `unclaimed_threshold_days` | Admin "settings" flow (SRC-04); D-11, D-20. |
| ADD-4 | Extra columns: `laundry_order.queue_date`, `laundry_order.created_by`→app_user; `status_history.changed_by`→app_user; `payment.paid_at` nullable, `payment.recorded_by` nullable→app_user; `service.is_active`; `customer.is_active` | D-05, D-07, D-08, D-12, D-15. |
| ADD-5 | Enums: order/history `status` = the 5 values (code form: `WAITING, WASHING, DRYING, READY_FOR_PICKUP, CLAIMED`); `payment_status` = `UNPAID, PAID`. Standard `created_at`/`updated_at` on every table. | SRC-06, D-08. |

Cardinalities: as **[INTERPRETED]** under SRC-07. `D3 Queue` = derived view (D-16).

### 2.4 Micro-decision policy (for anything not covered above)
- **LOW-IMPACT** (label wording, field length limits, cosmetic UI): agent picks a sensible value, logs it in `docs/02_ASSUMPTIONS.md` as `A-xx (PROPOSED)`, keeps it easy to change, and continues.
- **HIGH-IMPACT** (data model, business rule, security, external service, anything affecting money/notification/status): agent **stops and asks**.

### 2.5 Out of scope for v1 (agent must NOT build these)
Washer/dryer machine assignment or capacity · inventory or supplies · online payment or payment gateways · partial payments, refunds, discounts · order cancellation, order editing, status reverting · customer accounts/login/self-registration · native mobile apps · delivery/pickup-service dispatch · loyalty programs · multi-branch · websockets/push notifications · CSV/PDF/report export · editable notification templates · password-recovery emails · printing/receipt-printer integration · any feature not in Section 2.1.

---

## 3. MASTER PROMPT — give this to the agent as standing instructions

```text
ROLE
You are a senior software engineer building the "Laundry Queuing & Customer
Notification System" for a small laundry shop. You work in strict phases. You do
not guess. You prove your work by running it.

AUTHORITY ORDER (highest first)
1. My messages in this conversation.
2. docs/01_DECISIONS.md  (Decision Register: D-xx, ADD-x).
3. docs/00_SOURCE_OF_TRUTH.md  (facts from the analysis deck: SRC-xx).
4. Nothing else. Your general knowledge is NOT a source of requirements.
If two sources conflict, stop and ask me. Do not choose silently.

ANTI-HALLUCINATION RULES
R1  TRACE EVERYTHING. Every requirement, table, column, endpoint, screen, message
    and business rule you create must cite a source ID: SRC-xx, D-xx, ADD-x, or an
    approved A-xx. If you cannot cite one, do not create it.
R2  NEVER FILL GAPS SILENTLY. If something is missing from the sources:
      LOW-IMPACT (label wording, field length limits, cosmetic UI): choose a
        sensible value, log it in docs/02_ASSUMPTIONS.md as "A-xx PROPOSED", keep
        it easy to change, continue.
      HIGH-IMPACT (data model, business rule, security, external services, money,
        notifications, status logic): STOP and ask me. Give a numbered question
        with your proposed default. Ask at most 5 questions at once.
R3  NO INVENTED FACTS ABOUT TOOLS. Before using any library, framework, CLI flag,
    API or provider, verify it exists by running a real command (package manager
    info/show/search, --help, reading the installed source or official docs).
    Never write version numbers, function signatures or API endpoints from memory.
    If you cannot verify something, mark it UNVERIFIED and put it behind an
    interface with a mock instead.
R4  NO UNPROVEN CLAIMS. You may say "done" or "works" only for things you ran.
    Show the exact command and real output (trimmed). If you did not run
    something, write NOT RUN and say why.
R5  SCOPE DISCIPLINE. Do only the current phase. Never build anything in the
    Out-of-Scope list (Decisions section 2.5). No "nice to have" additions. Do not
    refactor earlier phases except to fix a defect, and report that as a deviation.
R6  EXACT NAMES. Use the glossary names below exactly. No synonyms.
R7  ONE HOME PER RULE. Each business rule lives in exactly one domain module.
    API handlers and UI call it. Never re-implement a rule in a second place.
R8  MONEY AND TIME. Use decimal arithmetic for money, never binary floats. Store
    timestamps in UTC. Convert to shop timezone only for display and day
    boundaries.
R9  DO NOT EDIT docs/00 or docs/01. If you believe they need a change, propose it
    as a diff in your report and wait for my approval.
R10 TESTS COME FROM ACCEPTANCE CRITERIA. Write tests from the phase's acceptance
    criteria first (where feasible), then implement. Expected values in tests must
    be hand-computed literals, never produced by the code under test.

PHASE PROTOCOL (repeat for every phase)
 1. READ     List the files you read and the SRC/D/ADD IDs relevant to this phase.
 2. PLAN     Numbered plan: files to create/change, tests to write, open
             questions. If you have HIGH-IMPACT questions, stop here and ask.
 3. TESTS    Write tests from the acceptance criteria.
 4. BUILD    Implement the minimum that satisfies the criteria.
 5. VERIFY   Run tests, linter, type-check, migrations as applicable. Paste output.
 6. DRIFT CHECK  Scan everything you produced. List any item with no source ID.
             Remove it or flag it as a Proposed Addition.
 7. REPORT   Use the PHASE REPORT FORMAT below. Update docs/PROGRESS.md.
 8. STOP     Wait for me to reply "APPROVED P#". Do not start the next phase.

PHASE REPORT FORMAT (use these exact headings)
 1. Phase and goal
 2. Files created/changed (path - purpose)
 3. Traceability table: item | source ID | test name
 4. Verification evidence (command + real output)
 5. Assumptions added (A-xx) and open questions
 6. Deviations from sources (must be "None" or explained)
 7. NOT RUN / not done
 8. Ready for next phase? (Yes/No, with reasons)

GLOSSARY (use exactly)
 Order            = one customer drop-off; table laundry_order.
 Item             = one service line within an order; table laundry_item.
 Service          = washing/drying service with a price per kg; table service.
 Queue number     = the customer-facing daily number on an order.
 Status codes     = WAITING, WASHING, DRYING, READY_FOR_PICKUP, CLAIMED.
 Status labels    = Waiting, Washing, Drying, Ready for Pickup, Claimed.
 Roles            = ADMIN (the Owner) and STAFF. Customers have no accounts.
 Active order     = any order whose status is not CLAIMED.
 Unclaimed order  = an order whose status is READY_FOR_PICKUP.
 Overdue          = unclaimed longer than unclaimed_threshold_days (D-11).
 Ready time       = time of the order's READY_FOR_PICKUP status_history row.

PROGRESS FILE
Maintain docs/PROGRESS.md: a table with columns Phase | Status (NOT STARTED /
IN REVIEW / APPROVED) | Date | Notes. Update it at the end of every phase.

Acknowledge these rules in one sentence, then wait for the first phase prompt.
```

---

## 4. PHASE PROMPTS

Send one at a time. Each starts with the same line so the agent knows to follow the protocol.

---

### P0 — Ingest and read-back (no code)

*Purpose: prove the agent understood the sources before it builds anything.*

```text
PHASE P0 - INGEST AND READ-BACK. Follow the Master Prompt phase protocol.
NO CODE IN THIS PHASE.

READ: docs/00_SOURCE_OF_TRUTH.md and docs/01_DECISIONS.md completely.

TASKS
1. Produce a READ-BACK containing, in this order:
   a. The system purpose in at most 3 sentences (cite SRC ids).
   b. The actors/roles and what each may do (from D-03).
   c. The five status codes in order, and the one rule that triggers notification.
   d. The seven Level-1 processes (1.0 to 7.0) and five data stores (D1 to D5).
   e. Every table and column: the deck ERD (SRC-07) as-is, then the additions
      ADD-1 to ADD-5, clearly separated.
   f. The feature units E-01..E-03 and F-01..F-08 with their phase numbers.
   g. Which USER FILL items in Decisions are still blank, and what you will do
      about each (per the rules in the register).
   h. Every contradiction, ambiguity or missing detail you find between the two
      documents. Number them. For each, propose a resolution and mark it
      LOW-IMPACT or HIGH-IMPACT.
   i. A list of things you would have to invent to build this system that are
      NOT covered by either document. (If the list is empty, say so and justify.)
2. Create these files: docs/PROGRESS.md (phase table P0 to P15, all NOT STARTED
   except P0 IN REVIEW), docs/02_ASSUMPTIONS.md (empty table: ID | Assumption |
   Impact | Phase | Status), docs/03_GLOSSARY.md (copy of the Master Prompt glossary).

ACCEPTANCE CRITERIA
- AC-P0-1: Read-back items a to f match the two documents exactly (I will compare).
- AC-P0-2: Section h and i are present and specific, not generic.
- AC-P0-3: No code, no dependencies installed, no project scaffold created.

Finish with the PHASE REPORT and STOP.
```

---

### P1 — Requirements catalogue and traceability

```text
PHASE P1 - REQUIREMENTS CATALOGUE AND TRACEABILITY. Follow the Master Prompt
phase protocol. NO CODE IN THIS PHASE.

READ: docs/00, docs/01, docs/02, docs/03, docs/PROGRESS.md.
SOURCES: all SRC ids; D-01..D-20; ADD-1..ADD-5.

TASKS
1. Create docs/10_REQUIREMENTS.md. For each unit E-01..E-03 and F-01..F-08 write
   atomic functional requirements named FR-<unit>-<n> (example: FR-F02-3).
   Each FR must:
   - be a single testable statement using "shall";
   - cite at least one SRC/D/ADD id;
   - have acceptance criteria in Given/When/Then form, with at least one
     negative case where an input can be invalid.
2. Add a permission matrix section reproducing D-03 exactly.
3. Add non-functional requirements, named NFR-<n>, ONLY where sourced from
   SRC-08, D-13, D-18, D-19c. Do not add performance numbers, uptime targets or
   scale figures that are not in the sources.
4. Confirm the [INTERPRETED] items in SRC-04, SRC-05 and SRC-07. List each one
   and say "confirmed as written" or propose a change (do not apply it).
5. Create docs/11_TRACEABILITY.md: a matrix with rows for EVERY item below and
   columns: Source id | FR ids | Phase | Test (leave "TBD"):
   - each SRC-09a..h;
   - each Level-1 process 1.0..7.0;
   - each table and column in SRC-07 and in ADD-1..ADD-5;
   - each SRC-08a..f;
   - each decision D-xx.
6. Add a section "Proposed additions" for any feature you believe is missing.
   Put nothing there into the FR list.

ACCEPTANCE CRITERIA
- AC-P1-1: Every FR cites a source id; every FR has Given/When/Then.
- AC-P1-2: Every row listed in task 5 exists in the traceability matrix; no row is
  empty in "FR ids" unless marked "structural (no FR)" with a reason.
- AC-P1-3: No FR relates to anything in the Out-of-Scope list (Decisions 2.5).
- AC-P1-4: Report includes counts: number of FRs per unit, number of NFRs.

Finish with the PHASE REPORT and STOP.
```

---

### P2 — Architecture and project scaffold

```text
PHASE P2 - ARCHITECTURE AND SCAFFOLD. Follow the Master Prompt phase protocol.

READ: docs/00, docs/01, docs/10_REQUIREMENTS.md, docs/PROGRESS.md.
SOURCES: D-01, D-02, D-13, D-18, D-19c, SRC-05.

STEP 0 (GATE). Check D-01 in docs/01_DECISIONS.md.
If D-01 is blank: STOP. Present 2 or 3 stack options. For each give: language,
web framework, database, why it fits this project (a small shop, staff on
desktop/tablet, public phone-friendly tracking page, transactional integrity for
queue numbers and payments), and the main trade-off. Ask me to choose. Do
nothing else until I answer.

TASKS (after D-01 is set)
1. Verify tooling per rule R3. For every runtime, package manager, framework and
   library you plan to use, run a real command to show the tool exists and what
   version is installed or available. Paste the output. No versions from memory.
2. Create docs/20_ARCHITECTURE.md containing:
   - an ADR (decision record) for the stack, citing D-01;
   - layers: domain (business rules), services, API, UI, persistence, adapters;
   - a module list where each module maps to one DFD process 1.0..7.0
     (SRC-05) or to E-01/E-02/E-03;
   - folder structure; naming, error-handling, validation and logging
     conventions; configuration through environment variables only;
   - the rule that all money uses decimals and all timestamps are UTC.
3. Scaffold the project: health-check endpoint, test runner, linter/formatter,
   type-check if the language has one, database migration tool, .env.example with
   variable NAMES only (no secrets, no real values), README with exact commands
   to install, migrate, test, and run.
4. Commit the lock file. Add only dependencies you actually use.
5. NO business logic and NO tables beyond what the migration tool itself needs.

ACCEPTANCE CRITERIA
- AC-P2-1: From a clean checkout, the documented commands install, run tests
  (at least one trivial passing test) and start the app. You must actually run
  this and paste the output.
- AC-P2-2: The health-check endpoint returns success; show the request/response.
- AC-P2-3: Every dependency in the lock/manifest has verification evidence in
  the report or docs/20_ARCHITECTURE.md; no unused dependencies.
- AC-P2-4: .env.example contains no secrets.
- AC-P2-5: docs/20 has a module-to-DFD-process mapping covering 1.0..7.0.

Finish with the PHASE REPORT and STOP.
```

---

### P3 — Database schema and migrations

```text
PHASE P3 - DATABASE SCHEMA. Follow the Master Prompt phase protocol.

READ: docs/00, docs/01, docs/20_ARCHITECTURE.md, docs/10_REQUIREMENTS.md.
SOURCES: SRC-07, ADD-1..ADD-5, D-05, D-06, D-07, D-08, D-14, D-15, D-16, D-19c.

TASKS
1. Write migrations for exactly these tables: customer, laundry_order,
   laundry_item, service, status_history, payment (SRC-07, using the deck's
   column names) plus app_user, notification, setting (ADD-1..ADD-3) and the
   extra columns in ADD-4 and enums in ADD-5. Do not add any other table,
   column or enum value. There is NO queue table (D-16).
2. Constraints (each must be enforced by the database, not only by app code):
   - primary keys and foreign keys; all foreign keys ON DELETE RESTRICT (D-15);
   - NOT NULL on every column except: customer.email, notification.last_error,
     notification.next_attempt_at, notification.sent_at, payment.paid_at,
     payment.recorded_by;
   - CHECK: laundry_item.weight > 0; laundry_item.price >= 0; service.price >= 0;
     payment.amount >= 0; laundry_order.total_amount >= 0; notification.attempts >= 0;
   - UNIQUE (queue_date, queue_number) on laundry_order (D-05);
   - UNIQUE customer.contact_number (D-14); UNIQUE app_user.username;
   - UNIQUE payment.order_id (one payment per order, D-08);
   - UNIQUE (order_id, channel) on notification WHERE trigger = AUTO (D-10), or the
     nearest equivalent your database supports (verify per R3; explain if
     you use a different mechanism);
   - money columns use a fixed-point decimal type with 2 decimals; weight has
     2 decimals.
3. Indexes, each justified in one line: laundry_order(status);
   laundry_order(queue_date, queue_number); status_history(order_id, date_time);
   notification(order_id); notification(status, next_attempt_at);
   customer(contact_number). No other indexes.
4. Migrations must be reversible (down works) and idempotent to run from empty.
5. Dev-only seed: one ADMIN user whose username and password come from
   environment variables (never hard-coded), plus setting rows shop_name (from
   D-20) and unclaimed_threshold_days = 3. No sample customers or orders in the
   seed. Sample services are allowed only in a separate dev fixture file clearly
   named SAMPLE and excluded from production.
6. Create docs/30_DATA_DICTIONARY.md: for every table and column: type,
   nullable, default, and the source id (SRC-07 or ADD-n) that justifies it.

TESTS (database-level; each violation must be REJECTED by the database)
- one test per CHECK, per UNIQUE, and per FK constraint above;
- migrate up from empty, then down, then up again succeeds;
- deleting a customer that has an order is rejected.

ACCEPTANCE CRITERIA
- AC-P3-1: Every column named in SRC-07 exists with the same name.
- AC-P3-2: The set of columns in the migrations equals the set in the data
  dictionary (show a comparison; no extras either way).
- AC-P3-3: All constraint tests pass; paste the real output.
- AC-P3-4: No column or table exists without a source id in the dictionary.

Finish with the PHASE REPORT and STOP.
```

---

### P4 — E-01 Authentication and roles

```text
PHASE P4 - E-01 AUTHENTICATION AND ROLES. Follow the Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (E-01 section), docs/20_ARCHITECTURE.md,
docs/30_DATA_DICTIONARY.md.
SOURCES: SRC-04, SRC-05, D-03 (matrix), D-15, D-18, ADD-1.

HIGH-IMPACT QUESTION TO ASK FIRST (R2): session cookies vs tokens. Propose the
option that is idiomatic for the chosen stack, with reasons, and wait for my
answer before implementing session handling.

FUNCTIONS TO BUILD (one at a time, each with tests)
1. Password hashing/verification with a verified library (R3). Passwords are
   never stored, logged or returned.
2. Login: username + password. Success creates a session/token. Failure gives a
   generic error that does not reveal whether the username exists. Inactive users
   cannot log in.
3. Logout: invalidates the session/token.
4. Current-user endpoint: returns user_id, name, role only.
5. Authorization guard: default-deny on every non-public route; role check for
   ADMIN and STAFF enforced on the server. Public routes are an explicit
   allow-list (currently only health check and the tracking lookup, built later).
6. ADMIN user management: create user (name, username, initial password, role),
   list users, change role, deactivate/reactivate, reset password. No delete
   (D-15). An ADMIN cannot deactivate their own account.
7. Basic rate limiting on login (D-18).
NOT IN THIS PHASE: self-registration, password-recovery email, social login,
two-factor authentication (all out of scope).

ACCEPTANCE CRITERIA
- AC-P4-1: Unauthenticated request to any protected route -> 401.
- AC-P4-2: STAFF calling any ADMIN-only route -> 403. Provide a table-driven test
  that iterates every route registered so far against ADMIN, STAFF and
  unauthenticated, checked against the D-03 matrix.
- AC-P4-3: Wrong username and wrong password produce the same response body.
- AC-P4-4: An inactive user cannot log in and existing sessions stop working.
- AC-P4-5: No response, log line or error contains a password or password hash.
- AC-P4-6: Login rate limit triggers after repeated failures (show the test).
- AC-P4-7: An ADMIN cannot deactivate themselves.

Finish with the PHASE REPORT and STOP.
```

---

### P5 — E-02 Service catalogue and settings

```text
PHASE P5 - E-02 SERVICE CATALOGUE AND SETTINGS. Follow the Master Prompt phase
protocol.

READ: docs/10_REQUIREMENTS.md (E-02), docs/30_DATA_DICTIONARY.md.
SOURCES: SRC-04 ("settings"), SRC-07 (SERVICE), D-03, D-06, D-11, D-15, D-20, ADD-3, ADD-4.

FUNCTIONS TO BUILD
1. Create service (ADMIN): service_name, price per kg (decimal, 2 dp, >= 0).
2. Edit service name and price (ADMIN). A price change affects only FUTURE
   orders; existing laundry_item rows are never touched (D-06).
3. Deactivate / reactivate service (ADMIN). No delete (D-15).
4. List services: ADMIN sees all (with active flag); STAFF sees ACTIVE only
   (needed by order creation in P7).
5. Settings: read and update shop_name and unclaimed_threshold_days (ADMIN only
   to write). Validate: shop_name non-empty; threshold is a whole number >= 1
   (log the upper bound you choose as a LOW-IMPACT assumption).
6. UI screens: services list + form, settings form (ADMIN only).

ACCEPTANCE CRITERIA
- AC-P5-1: STAFF cannot create/edit/deactivate services or change settings (403).
- AC-P5-2: STAFF service list contains only active services; ADMIN list contains
  all.
- AC-P5-3: Price is stored with exactly 2 decimals; negative or non-numeric
  price rejected; test with "10.005" and state the exact behaviour (reject or
  round) as a LOW-IMPACT assumption A-xx.
- AC-P5-4: Editing a service's price does not alter any existing laundry_item.price
  (test with a fixture item).
- AC-P5-5: Settings persist and are read back correctly after restart.
- AC-P5-6: Duplicate service_name behaviour is decided, logged as an assumption,
  and tested.

Finish with the PHASE REPORT and STOP.
```

---

### P6 — F-01 Customer registration / information

```text
PHASE P6 - F-01 CUSTOMER MANAGEMENT (DFD 1.0, store D1). Follow the Master Prompt
phase protocol.

READ: docs/10_REQUIREMENTS.md (F-01), docs/30_DATA_DICTIONARY.md.
SOURCES: SRC-09a, SRC-05 (1.0), SRC-02a, D-03, D-14, D-15.

FUNCTIONS TO BUILD
1. Contact-number normalization: pure function. Strip spaces, dashes,
   parentheses; keep one optional leading "+"; result must be 7 to 15 digits.
   Invalid input -> validation error. Table-driven tests including:
   "0917 123 4567" (spaces), "(02) 8123-4567", "+63 917 123 4567", "12345"
   (too short), "abcdefghij" (non-numeric), "" (empty).
2. Create customer: name (required, trimmed), contact_number (required,
   normalized), email (optional, format-validated). A duplicate contact_number is
   rejected with an error that includes the existing customer's id and name so
   staff can use the existing record.
3. Search customers: partial name (case-insensitive) OR partial contact number.
   Returns active customers by default; option to include inactive.
4. View customer details: profile + list of that customer's orders
   (queue_date, queue number, status, total_amount). Read-only; there are no
   orders until P7, so test with fixtures.
5. Edit customer (name, contact_number, email) with the same validation; the
   duplicate check excludes the customer being edited.
6. Deactivate / reactivate customer. No delete. Inactive customers cannot be
   selected for new orders (enforced in P7).
7. UI: search/list screen, create/edit form, detail screen. Walk-in flow: the
   search box is on the create screen so staff check for an existing customer
   first. Both roles may use these screens.

ACCEPTANCE CRITERIA
- AC-P6-1: All normalization vectors in task 1 behave as specified.
- AC-P6-2: Creating two customers whose numbers normalize to the same value
  (e.g. "0917 123 4567" and "09171234567") is rejected.
- AC-P6-3: A customer with orders cannot be hard-deleted; no delete route exists.
- AC-P6-4: Invalid email rejected; missing email accepted.
- AC-P6-5: Search returns expected matches for partial name and partial number
  (hand-written expected lists in tests).
- AC-P6-6: Unauthenticated -> 401 on all customer routes.

Finish with the PHASE REPORT and STOP.
```

---

### P7 — F-02 Order creation and queue number

```text
PHASE P7 - F-02 ORDER CREATION AND QUEUE NUMBER (DFD 2.0, stores D2, D3, D4, D5).
Follow the Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-02), docs/30_DATA_DICTIONARY.md, docs/20_ARCHITECTURE.md.
SOURCES: SRC-09b, SRC-05 (2.0), SRC-03, D-05, D-06, D-07, D-08, D-15, D-17, D-19a, D-19c.

This phase has 6 functions. Build and test them in order.

FUNCTION 1 - Line amount calculation (pure, decimal arithmetic, no floats)
  line_amount = weight x service_price, rounded half-up to 2 decimals.
  Required test vectors (hand-computed):
    weight 2.50, price 30.00 -> 75.00
    weight 1.33, price 25.00 -> 33.25
    weight 0.75, price 19.99 -> 14.99   (exact product 14.9925)
    weight 3.10, price 12.35 -> 38.29   (exact product 38.285; half-up)
    weight 0.01, price 0.50  -> 0.01    (exact product 0.005; half-up)
  Also: total_amount = sum of line amounts (test with 3 items).

FUNCTION 2 - Queue number assignment
  Business date (queue_date) = current date in the shop timezone D-19a. Number =
  1 + the highest number already used for that queue_date, starting at 1.
  Assigned inside the order-creation transaction using a mechanism that is safe
  under concurrency for the chosen database (verify per R3 and state the
  mechanism in the report). Display format is zero-padded to 3 digits
  (1 -> "001"); numbers above 999 display in full ("1000").
  Tests: first order of a day = 1; next = 2; a new queue_date restarts at 1
  (use a controllable clock); 20 orders created in parallel -> 20 distinct
  numbers, no duplicates; midnight boundary in the shop timezone (23:59 local vs
  00:01 local) lands on different queue_dates even when the UTC dates are equal.

FUNCTION 3 - Create order (single transaction)
  Input: customer_id, items[] of {service_id, weight}. Actor = logged-in user.
  Validate: customer exists and is active; items not empty; every service exists
  and is active; every weight > 0 with at most 2 decimals.
  In ONE database transaction create:
    laundry_order (status WAITING, queue_date, queue_number, total_amount,
      created_by),
    laundry_item rows (price = line amount, snapshot per D-06),
    status_history row (WAITING, changed_by = actor, date_time = now UTC),
    payment row (payment_status UNPAID, amount = total_amount) per D-08.
  Any failure rolls back everything. Test with a forced failure after the order
  row is inserted and prove NO partial rows remain.

FUNCTION 4 - Order confirmation screen/response
  Shows the queue number prominently (formatted), customer name, items,
  total_amount and status label. (Printing is out of scope.)

FUNCTION 5 - Order detail read
  Returns the order with customer, items, payment summary, status, queue number.
  (Status history and notification log are added in P9/P10, do not build them now.)

FUNCTION 6 - UI: new-order screen
  Select customer (search from P6), add item rows (service dropdown of ACTIVE
  services from P5, weight input), live total preview using the same rule as
  Function 1 (call the shared rule, do not duplicate it, R7), submit.

ACCEPTANCE CRITERIA
- AC-P7-1: All Function 1 vectors pass, and a test proves float arithmetic is not
  used (the 38.29 vector is the guard).
- AC-P7-2: Empty items, zero/negative weight, weight with 3 decimals, inactive
  customer, inactive service, unknown service -> rejected with clear errors.
- AC-P7-3: A successful order creates exactly 1 laundry_order, N laundry_item,
  1 status_history (WAITING), 1 payment (UNPAID); status is WAITING.
- AC-P7-4: Rollback test passes (no partial data).
- AC-P7-5: Parallel-creation and midnight-boundary tests pass.
- AC-P7-6: Editing a service price afterwards does not change existing item prices.
- AC-P7-7: No order-edit or order-cancel route exists (D-17, out of scope).

Finish with the PHASE REPORT and STOP.
```

---

### P8 — E-03 Queue view (manage queue)

```text
PHASE P8 - E-03 QUEUE VIEW (DFD 3.0, D3 as a derived view). Follow the Master
Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (E-03), docs/30_DATA_DICTIONARY.md.
SOURCES: SRC-05 (3.0), SRC-02b, SRC-08a, SRC-08c, D-13, D-16.

This phase is READ-ONLY: it changes no data.

FUNCTIONS TO BUILD
1. Active-queue query: all orders with status != CLAIMED, ordered by
   (queue_date ASC, queue_number ASC). Each row: order_id, formatted queue number,
   queue_date, customer name, status code + label, payment_status, and
   time-in-current-status (now minus the latest status_history.date_time for
   that order).
2. "Next to process": the oldest order with status WAITING, flagged in the
   response and highlighted in the UI. If none, no highlight.
3. Filter by status (single status or all).
4. UI: queue screen grouped by status (Waiting / Washing / Drying / Ready for
   Pickup), with the next-to-process order highlighted; empty state message;
   auto-refresh by polling every 10 seconds (D-13; interval as one named constant).
   No advance-status buttons yet (added in P9).
5. Access: ADMIN and STAFF.

ACCEPTANCE CRITERIA
- AC-P8-1: With fixtures spanning two queue_dates, an unfinished order from
  yesterday appears BEFORE today's orders.
- AC-P8-2: CLAIMED orders never appear.
- AC-P8-3: "Next to process" is the oldest WAITING order, and is not chosen from
  any other status (test with WASHING orders older than the oldest WAITING one).
- AC-P8-4: Time-in-status is computed from the latest history row, verified with a
  controlled clock and hand-computed expected durations.
- AC-P8-5: No manual reorder/priority feature exists.
- AC-P8-6: Unauthenticated -> 401.

Finish with the PHASE REPORT and STOP.
```

---

### P9 — F-03 Order status flow and history

```text
PHASE P9 - F-03 STATUS WORKFLOW AND HISTORY (DFD 4.0, stores D2, D4). Follow the
Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-03), docs/30_DATA_DICTIONARY.md, docs/20_ARCHITECTURE.md.
SOURCES: SRC-09c, SRC-06, SRC-03, SRC-05 (4.0), D-03, D-07, D-16.

SCOPE NOTE. In this phase you implement ONLY these three transitions:
  WAITING -> WASHING, WASHING -> DRYING, DRYING -> READY_FOR_PICKUP.
The transition READY_FOR_PICKUP -> CLAIMED is built in P12 because it requires
payment (D-08). In this phase, attempting to advance an order that is
READY_FOR_PICKUP must be rejected with a clear error stating that claiming is a
separate action.

FUNCTIONS TO BUILD (in order)
1. Transition table (single source of truth, R7): a data structure listing the
   valid next status for each status. Everything else (API, UI, notification
   trigger) reads it. Do not encode transitions anywhere else.
2. Advance command: "advance order X to its next status". The caller does NOT
   choose the target status; the domain layer computes it from the table. This
   removes any possibility of skipping.
3. Atomicity and concurrency: in ONE transaction, update laundry_order.status and
   insert a status_history row (order_id, new status, date_time = now UTC,
   changed_by = actor). Use an expected-current-status check so that if two staff
   press "advance" at the same moment, exactly one succeeds and the other gets a
   conflict error. Test with two simultaneous calls.
4. Domain event: after the transaction COMMITS, publish an in-process event
   OrderStatusChanged {order_id, from_status, to_status, at}. Provide a simple
   subscriber registry. A subscriber that throws must NOT affect the committed
   status change (test this). No subscriber exists yet (P10 adds one).
5. History read: endpoint returning an order's status timeline (status code,
   label, time in UTC, changed_by name), oldest first. Add it to the order detail
   screen from P7.
6. UI: on the queue screen from P8 add one button per order whose label depends on
   the current status: Waiting -> "Start washing"; Washing -> "Move to drying";
   Drying -> "Mark ready for pickup". Label wording is a LOW-IMPACT assumption; log it.
7. Access: ADMIN and STAFF.

ACCEPTANCE CRITERIA
- AC-P9-1: A table-driven test covers ALL 25 (from,to) pairs of the 5 statuses
  for the domain function. Exactly the 3 pairs above succeed via advance; the
  other 22 are rejected. Paste the test list.
- AC-P9-2: Advancing 3 times from WAITING yields exactly 4 status_history rows
  (WAITING, WASHING, DRYING, READY_FOR_PICKUP) in that order (the WAITING row
  came from P7).
- AC-P9-3: No API accepts a target status from the client; skipping is impossible
  through the API (test by sending a body with a "status" field and proving it is
  ignored or rejected).
- AC-P9-4: Concurrent advance test: 2 parallel calls -> 1 success, 1 conflict,
  exactly 1 new history row.
- AC-P9-5: A failing subscriber does not roll back or block the status change.
- AC-P9-6: Advancing a READY_FOR_PICKUP or CLAIMED order is rejected.
- AC-P9-7: No revert/cancel route exists.
- AC-P9-8: Unauthenticated -> 401.

Finish with the PHASE REPORT and STOP.
```

---

### P10 — F-05 Automatic ready-for-pickup notification

```text
PHASE P10 - F-05 AUTOMATIC NOTIFICATION (DFD 5.0, external entity "SMS / Email
Notification"). Follow the Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-05), docs/30_DATA_DICTIONARY.md,
docs/20_ARCHITECTURE.md, the OrderStatusChanged event code from P9.
SOURCES: SRC-09e, SRC-06, SRC-04 (SMS/Email entity), SRC-05 (5.0), SRC-02d,
SRC-08b, D-09a, D-09b, D-09c, D-10, D-10m, D-18, ADD-2.

ABSOLUTE RULE FOR THIS PHASE. Do NOT write any provider-specific HTTP call or SDK
usage from memory. Real provider code is allowed ONLY if D-09a / D-09b are filled,
and ONLY after you fetch the provider's official documentation, cite its URL in
docs/20_ARCHITECTURE.md, and verify every field name against it (R3). If D-09a or
D-09b are blank, build ONLY the interface and the Mock adapter.

HIGH-IMPACT QUESTION TO ASK FIRST (R2): how retries are scheduled. My preferred
default is a database-polled worker (select notifications that are PENDING with
next_attempt_at <= now) so no extra infrastructure is needed. Confirm with me or
propose an alternative before building the dispatcher.

FUNCTIONS TO BUILD (in order)
1. Channel interface: send(recipient, subject_or_null, body) returning either
   success or a failure with a message. Two channels: SMS and EMAIL.
2. Mock adapters for both channels that record what would have been sent and can
   be told to fail. All automated tests use mocks. Tests never call a real
   provider.
3. Message composer (pure function): builds the SMS body and the email
   subject/body from D-10m using customer_name, queue_number (formatted 3-digit)
   and shop_name (from the setting table). Test with exact expected strings,
   including a customer name containing an apostrophe and non-ASCII characters.
4. Event subscriber: on OrderStatusChanged where to_status = READY_FOR_PICKUP
   ONLY (D-10):
   - always create an SMS notification row (recipient = contact_number);
   - create an EMAIL row only if the customer has an email;
   - status PENDING, trigger AUTO, attempts 0, next_attempt_at = now;
   - idempotent: a repeated event for the same order must not create a second
     AUTO row for the same channel (rely on the DB constraint from P3).
5. Dispatcher: picks due PENDING rows, calls the channel, on success sets SENT and
   sent_at; on failure increments attempts, stores last_error, and reschedules
   next_attempt_at (+1 minute after the 1st failure, +5 minutes after the 2nd);
   after the 3rd failed attempt sets FAILED. Delays are named constants. The
   dispatcher never touches laundry_order.status.
6. Manual resend: STAFF/ADMIN action on an order in READY_FOR_PICKUP that creates
   new notification rows with trigger MANUAL and sends them through the same
   dispatcher. Resend is refused if the order is not READY_FOR_PICKUP.
7. Order detail shows a notification log: channel, recipient, trigger, status,
   attempts, last_error, sent_at. (Visible to authenticated staff only; D-03.)
8. Configuration through environment variables only. Real adapter, if any, is
   behind the same interface.

ACCEPTANCE CRITERIA
- AC-P10-1: Advancing an order to READY_FOR_PICKUP creates exactly 1 SMS row, plus
  1 EMAIL row only when the customer has an email.
- AC-P10-2: Advancing to WASHING, DRYING (and later CLAIMED) creates 0 rows.
- AC-P10-3: If the adapter fails permanently, the order STAYS in READY_FOR_PICKUP,
  the notification ends FAILED with attempts = 3 and a non-empty last_error.
- AC-P10-4: Delivering the same event twice creates no duplicate AUTO row.
- AC-P10-5: A failure on the first attempt followed by success on the second ends
  SENT with attempts = 2, using a controlled clock to verify the +1 minute delay.
- AC-P10-6: Message text matches the D-10m template exactly.
- AC-P10-7: Manual resend on a non-READY order is refused; on a READY order it
  creates MANUAL rows and leaves AUTO rows unchanged.
- AC-P10-8: No secrets or full phone numbers appear in logs.
- AC-P10-9: The report lists every external call the code can make, and for each
  one either the documentation URL used to verify it, or "MOCK ONLY".

Finish with the PHASE REPORT and STOP.
```

---

### P11 — F-04 Customer status tracking (public page)

```text
PHASE P11 - F-04 CUSTOMER STATUS TRACKING. Follow the Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-04), docs/30_DATA_DICTIONARY.md, the phone
normalization function from P6, the status transition/label code from P9.
SOURCES: SRC-09d, SRC-02c, SRC-08d, SRC-03, D-04, D-13, D-14, D-18.

FUNCTIONS TO BUILD (in order)
1. Lookup service: input queue number (accept "7", "007", "#007") and contact
   number. Normalize the contact number with the P6 function (R7: reuse, do not
   copy). Find the LATEST order (highest queue_date) whose queue number AND
   customer contact number both match.
2. Public response contract: an explicit allow-list of fields and nothing else:
   queue number (formatted), status code, status label, status meaning text from
   SRC-03 (Waiting: "Order received"; Washing: "Laundry is being washed";
   Drying: "Laundry is being dried"; Claimed: "Transaction completed"), and
   time of last status change. For READY_FOR_PICKUP the customer-facing text is
   not defined in SRC-03 (it says "Customer is notified"): log your proposed
   wording as a LOW-IMPACT assumption.
   MUST NOT be exposed: customer name, contact number, email, amounts, item
   details, order_id, user names.
3. Not-found behavior: wrong queue number, wrong contact number, or no order all
   produce the SAME response body and status code (no information leak).
4. Rate limiting on the lookup endpoint (D-18) and response header preventing
   caching (no-store).
5. UI: mobile-first public page: a form (queue number + contact number), then a
   progress display of the five statuses with the current one highlighted, plus
   the meaning text and last-updated time. Polls every 10 seconds (D-13) while
   open. No login.
6. This page is the only public route besides the health check.

ACCEPTANCE CRITERIA
- AC-P11-1: Correct queue + contact returns the order's current status.
- AC-P11-2: Wrong contact, wrong queue number and non-existent order return
  byte-identical responses.
- AC-P11-3: A test asserts the response contains ONLY the allow-listed keys
  (fail if any extra key appears).
- AC-P11-4: Formats "7", "007", "#007" and contact "0917 123 4567" vs "09171234567"
  resolve the same order.
- AC-P11-5: When two orders exist with the same queue number on different days for
  the same customer, the latest is returned.
- AC-P11-6: After a staff advance, a subsequent poll returns the new status.
- AC-P11-7: Rate limit triggers after repeated lookups (show the test).
- AC-P11-8: CLAIMED orders return status Claimed (not "not found").

Finish with the PHASE REPORT and STOP.
```

---

### P12 — F-06 Payment and claim recording

```text
PHASE P12 - F-06 PAYMENT AND CLAIM (DFD 6.0, stores D5, D2, D4). Follow the Master
Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-06), docs/30_DATA_DICTIONARY.md, the transition
table and advance command from P9.
SOURCES: SRC-09f, SRC-05 (6.0), SRC-03, D-07, D-08, D-12, D-03, ADD-4.

FUNCTIONS TO BUILD (in order)
1. Record payment: marks the order's existing payment row PAID, sets paid_at =
   now UTC and recorded_by = actor. amount is NOT changed (it already equals
   total_amount). Allowed at ANY order status except when already PAID. Marking
   an already-PAID payment returns a conflict and changes nothing.
2. Extend the P9 transition table with READY_FOR_PICKUP -> CLAIMED. Do this in the
   same single table (R7), guarded: the transition is allowed only if the
   order's payment_status is PAID.
3. Claim command: allowed only when status = READY_FOR_PICKUP AND payment is PAID.
   In one transaction: set status CLAIMED and insert a status_history row.
   Reject with distinct, clear errors: NOT_READY (status is not READY_FOR_PICKUP)
   and PAYMENT_REQUIRED (unpaid).
4. Combined action "Record payment and claim" for the counter: performs 1 then 3
   in ONE transaction. If the claim is rejected (e.g. not ready), the payment is
   NOT recorded (rollback). Test both outcomes.
5. After CLAIMED the order is immutable: no further status change, no payment
   change.
6. Publish OrderStatusChanged for the claim (the P10 subscriber must ignore it,
   D-10: no notification on CLAIMED).
7. UI: on the order detail screen show payment status/amount and the buttons
   "Record payment", "Claim", and "Record payment and claim" with only valid actions
   enabled.
8. Access: ADMIN and STAFF.

ACCEPTANCE CRITERIA
- AC-P12-1: Claim on an unpaid, READY order -> PAYMENT_REQUIRED, status unchanged.
- AC-P12-2: Claim on a paid order that is not READY (WAITING/WASHING/DRYING) ->
  NOT_READY.
- AC-P12-3: Paid + READY -> claim succeeds; history gains a CLAIMED row; order
  leaves the P8 active queue.
- AC-P12-4: Combined action on a READY unpaid order -> both recorded, atomically.
  Combined action on a non-READY order -> nothing recorded (payment stays UNPAID).
- AC-P12-5: Recording payment twice -> conflict, paid_at unchanged.
- AC-P12-6: The 25-pair transition test from P9 is updated and passes: exactly 4
  valid pairs now exist in the table.
- AC-P12-7: Claiming does not create any notification row.
- AC-P12-8: There is no route for partial payment, refund, discount, or
  un-claiming.

Finish with the PHASE REPORT and STOP.
```

---

### P13 — F-08 Unclaimed laundry monitoring

```text
PHASE P13 - F-08 UNCLAIMED LAUNDRY MONITORING. Follow the Master Prompt phase
protocol.

READ: docs/10_REQUIREMENTS.md (F-08), docs/30_DATA_DICTIONARY.md, the setting
code from P5, the notification resend code from P10.
SOURCES: SRC-09h, SRC-02e, SRC-08f, D-03, D-11, D-13.

FUNCTIONS TO BUILD (in order)
1. Ready-time derivation: for an order, ready time = date_time of its
   READY_FOR_PICKUP status_history row. If an order in READY_FOR_PICKUP has no
   such row, that is a data-integrity error: surface it, do not guess.
2. Unclaimed list: all orders with status READY_FOR_PICKUP, sorted by longest
   waiting first. Each row: formatted queue number, customer name, contact number,
   ready time, elapsed time since ready (days and hours), payment status,
   overdue flag, last notification status (from P10).
3. Overdue rule (single function, R7): overdue if (now - ready time) is STRICTLY
   GREATER than unclaimed_threshold_days x 24 hours, reading the threshold from
   the setting table at request time (a change applies immediately).
4. Count of unclaimed and count of overdue (exposed for the dashboard in P14).
5. UI: unclaimed screen with overdue rows visually marked and a "Resend
   notification" button per row that reuses the P10 action (do not duplicate it).
   Polls every 10 seconds (D-13).
6. Access: ADMIN and STAFF (D-03).

ACCEPTANCE CRITERIA
- AC-P13-1: With threshold 3: an order ready exactly 72h 00m ago is NOT overdue;
  ready 72h 01m ago IS overdue (controlled clock).
- AC-P13-2: Changing the threshold from 3 to 1 immediately flips a 30-hour-old
  unclaimed order to overdue.
- AC-P13-3: Only READY_FOR_PICKUP orders appear; WAITING/WASHING/DRYING/CLAIMED
  never appear.
- AC-P13-4: Sorted by longest waiting first (hand-written expected order).
- AC-P13-5: Resend from this screen creates MANUAL notification rows via the P10
  function (test proves the same code path).
- AC-P13-6: Elapsed time uses the READY_FOR_PICKUP history row, not the order
  creation time (test with an order created long before it became ready).

Finish with the PHASE REPORT and STOP.
```

---

### P14 — F-07 Admin dashboard and basic reports

```text
PHASE P14 - F-07 DASHBOARD AND REPORTS (DFD 7.0, reads D2, D4, D5). Follow the
Master Prompt phase protocol.

READ: docs/10_REQUIREMENTS.md (F-07), docs/30_DATA_DICTIONARY.md, the P13 unclaimed
functions.
SOURCES: SRC-09g, SRC-04 ("reports", "analytics / records"), SRC-08f, D-03, D-11,
D-12, D-19a, D-19c.

STEP 1 - DEFINITIONS FIRST. Before any code, create docs/50_REPORT_DEFINITIONS.md.
For each metric write: name, exact definition in words, the source columns, how
date boundaries work (day boundaries are in the SHOP timezone, converted to UTC
ranges), and 2 worked examples with numbers computed by hand. Metrics:
  - Sales (per day, in a date range): sum of payment.amount where payment_status
    = PAID and paid_at falls in that shop-local day.
  - Completed orders (range): orders whose CLAIMED status_history row falls in the
    range (count + list).
  - Unclaimed laundry: as D-11 / P13.
  - Dashboard: active orders per status; today's sales; today's completed count;
    overdue-unclaimed count.
Stop and show me this file before building if any definition is ambiguous.

STEP 2 - BUILD
1. Dashboard endpoint and screen (ADMIN only) with the four dashboard items.
2. Sales report: date range (from, to inclusive, shop-local dates), one row per
   day including days with zero sales, and a range total.
3. Completed-orders report: date range; count and a list (queue number,
   queue_date, customer name, total_amount, claimed time).
4. Unclaimed report: reuse the P13 function (R7).
5. Validation: from <= to; both dates required; reject invalid dates. Choose a
   maximum range as a LOW-IMPACT assumption and log it.
6. STAFF get 403 on all of the above. No export of any kind (out of scope).

ACCEPTANCE CRITERIA
- AC-P14-1: Fixture with hand-computed expected numbers (written as literals in the
  test): e.g. 3 paid orders on day A (100.00, 250.50, 49.50) and 1 on day B
  (80.00) -> day A = 400.00, day B = 80.00, total = 480.00.
- AC-P14-2: UNPAID orders contribute 0 to sales.
- AC-P14-3: Timezone boundary test: a payment at 23:30 shop-local time lands on
  that local day even if the UTC date is different; a payment at 00:30 local lands
  on the next local day.
- AC-P14-4: Days with no sales appear with 0.00.
- AC-P14-5: Completed orders use the CLAIMED history time, not order creation time.
- AC-P14-6: Dashboard counts equal the queue view (P8) and unclaimed (P13) counts
  on the same fixture (consistency test).
- AC-P14-7: STAFF -> 403; unauthenticated -> 401.
- AC-P14-8: docs/50_REPORT_DEFINITIONS.md exists and every implemented metric
  matches its definition.

Finish with the PHASE REPORT and STOP.
```

---

### P15 — Integration, audit and release readiness

```text
PHASE P15 - INTEGRATION, TRACEABILITY AND HALLUCINATION AUDIT. Follow the Master
Prompt phase protocol. Do NOT add features in this phase; only tests, fixes for
defects (reported as deviations), and documentation.

READ: everything in docs/, all code and tests.
SOURCES: all.

TASKS
1. END-TO-END HAPPY PATH (one automated test using mock notification adapters):
   ADMIN creates a service -> STAFF logs in -> creates a customer (with email) ->
   creates an order -> queue view shows it as next to process -> advance to
   WASHING, DRYING, READY_FOR_PICKUP -> exactly 1 SMS and 1 EMAIL notification
   recorded -> public tracking shows "Ready for Pickup" -> unclaimed list shows
   it -> "Record payment and claim" -> tracking shows "Claimed" -> ADMIN
   dashboard and sales report reflect the payment. Paste the output.
2. NEGATIVE PATHS (automated): skip a status; claim unpaid; claim before ready;
   STAFF opens reports; unauthenticated access to every protected route;
   notification adapter failure leaves status READY_FOR_PICKUP; tracking with
   wrong contact number.
3. CONCURRENCY (automated): 20 parallel order creations (distinct queue numbers);
   2 parallel advances (one wins).
4. TRACEABILITY AUDIT: produce docs/60_FINAL_TRACEABILITY.md. Rows for every
   SRC-09a..h, every DFD process 1.0..7.0, every SRC-07 table/column, every
   ADD-x, every D-xx, every FR/NFR. Each row must have: code location(s) and
   test name(s). Any row without both is a FAILURE to be fixed or reported.
5. ORPHAN SCAN: list every route, screen, table, column, environment variable,
   background job and setting that has no source id. Remove each or list it as a
   Proposed Addition for my approval.
6. OUT-OF-SCOPE SCAN: search the code for each item in Decisions 2.5 and confirm
   NONE is implemented. Show how you searched.
7. HALLUCINATION AUDIT: (a) table of every dependency with the evidence you used
   to verify it (command + output or doc URL); (b) list of every external call
   (SMS/email) with its verification source or "MOCK ONLY"; (c) grep for TODO,
   FIXME, lorem, placeholder, hard-coded secrets, hard-coded passwords, and
   commented-out code; show the commands and results; (d) list of everything
   marked UNVERIFIED or NOT RUN across all phases.
8. DOCUMENTATION: final README (install, configure, migrate, test, run);
   .env.example with every variable and a one-line purpose; a one-page STAFF guide
   and a one-page ADMIN guide (steps only for features that exist); a "Known
   limitations and future work" section that mirrors Decisions 2.5.
9. Run the FULL test suite, linter and type-check. Paste the summary.

ACCEPTANCE CRITERIA
- AC-P15-1: Happy-path and negative-path tests pass.
- AC-P15-2: docs/60_FINAL_TRACEABILITY.md has no row lacking code and test
  references.
- AC-P15-3: The orphan scan and out-of-scope scan report zero unresolved items.
- AC-P15-4: The hallucination audit lists every dependency and external call with
  verification evidence; UNVERIFIED list is empty or explained.
- AC-P15-5: A person following only the README can install and run the system
  (execute the README steps yourself in a clean directory and paste the output).

Finish with the PHASE REPORT (also add a final table: all phases, status,
approval) and STOP.
```

---

## 5. Recovery prompts (use only when needed)

**Drift — the agent added things not in the sources**
```text
STOP. List every item in your last output (files, tables, columns, routes, screens,
rules, dependencies) that has no source ID (SRC/D/ADD/approved A). For each: either
remove it now, or move it to a "Proposed Additions" list for my approval. Then redo
the DRIFT CHECK and show me the result. Do not continue the phase until I reply.
```

**Unproven claims — the agent says it works but shows no evidence**
```text
You said this works. Show the exact command you ran, its raw output, and the names
of the tests that cover it. If you did not run it, write NOT RUN and run it now.
Do not summarize; paste the real output.
```

**Unverified tools — the agent used a library or API from memory**
```text
For every library, CLI flag, function and external API you used in this phase,
show the verification (package-manager info/show output, --help output, or the
official documentation URL and the exact page section). Anything you cannot
verify: mark UNVERIFIED, replace it with an interface plus mock, and tell me.
```

**Context loss — new chat, agent forgot everything**
```text
New session. Read, in this order: docs/00_SOURCE_OF_TRUTH.md, docs/01_DECISIONS.md,
docs/02_ASSUMPTIONS.md, docs/03_GLOSSARY.md, docs/PROGRESS.md. Then tell me: the
last APPROVED phase, the phase now in progress, open assumptions and questions.
Write no code until I confirm your summary is right.
```

**Ambiguity — the agent is guessing on a business rule**
```text
Stop guessing. This is a HIGH-IMPACT rule. Ask me now: numbered question(s), your
proposed default for each, and the consequence of each option. Wait for my answer.
```

**Contradiction — the agent changed a decision on its own**
```text
Your change conflicts with docs/01_DECISIONS.md (cite the D-id). Revert it. If you
believe the decision is wrong, propose the change as a diff with reasons, and wait
for my approval. Do not edit docs/00 or docs/01 yourself.
```
