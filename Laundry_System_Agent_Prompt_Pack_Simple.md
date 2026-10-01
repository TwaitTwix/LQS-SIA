# Laundry Queuing & Notification System: Simple Agent Prompt Pack

Build the system from your deck with an AI coding agent, one phase at a time, without the agent inventing requirements.

**What changed from the long version:** one spec file instead of two, plain numbered rules instead of SRC/D/ADD IDs, 11 phases instead of 16, a shorter master prompt and one recovery prompt. The guardrails are the same: trace everything, verify tools, prove it by running, stay in the phase.

---

## How to use

1. Fill in the 6 blanks in **Part A, section 8**.
2. Save Part A as `docs/SPEC.md` in your project.
3. Give the agent **Part B** as standing instructions (project rules or system prompt). If your tool can't keep them, paste Part B at the top of every phase.
4. Send the phases in **Part C** one at a time, starting with P0.
5. Each phase ends with a short report. Check it, then reply `APPROVED P#` and send the next phase. If something is wrong, say what, and don't advance.

**Quick report check (1 minute):** Is there real command output, not just "tests pass"? Does every item cite a spec number? Is "Deviations" empty or explained? Did it build anything from section 7 (not in v1)?

---

# Part A: Spec (save as `docs/SPEC.md`)

Items marked **(deck)** come from your slides. Items marked **(default)** are my proposals to fill gaps the deck leaves open. Edit any you disagree with before starting.

## 1. What it is (deck)
Replaces manual laundry tracking and customer calling with a **queue number**, **live status** and **automatic notification**.

Actors: Customer, Admin/Owner, Laundry Staff, SMS/Email service.

## 2. Statuses (deck)
`WAITING → WASHING → DRYING → READY_FOR_PICKUP → CLAIMED`

| Code | Label | Customer-facing meaning |
|---|---|---|
| WAITING | Waiting | Order received |
| WASHING | Washing | Laundry is being washed |
| DRYING | Drying | Laundry is being dried |
| READY_FOR_PICKUP | Ready for Pickup | (deck says "Customer is notified"; propose wording as an assumption) |
| CLAIMED | Claimed | Transaction completed |

**Notification is triggered when status = READY_FOR_PICKUP (deck).**

## 3. Data
**Deck ERD (use these exact names, snake_case):**
- `customer`: customer_id, name, contact_number, email
- `laundry_order`: order_id, customer_id, queue_number, status, total_amount
- `laundry_item`: item_id, order_id, service_id, weight, price
- `service`: service_id, service_name, price
- `status_history`: status_id, order_id, status, date_time
- `payment`: payment_id, order_id, amount, payment_status

**Additions the deck's own diagrams need (default):**
- `app_user`: user_id, name, username (unique), password_hash, role (ADMIN/STAFF), is_active
- `notification`: notification_id, order_id, channel (SMS/EMAIL), recipient, trigger (AUTO/MANUAL), status (PENDING/SENT/FAILED), attempts, last_error, next_attempt_at, sent_at
- `setting`: key, value (holds `shop_name`, `unclaimed_threshold_days`)
- Extra columns: `laundry_order.queue_date`, `laundry_order.created_by`, `status_history.changed_by`, `payment.paid_at`, `payment.recorded_by`, `service.is_active`, `customer.is_active`
- `payment_status`: UNPAID / PAID. Every table also gets `created_at` / `updated_at`.
- There is **no queue table**. The queue is a view of active orders.

## 4. Rules (default unless marked deck)
1. **Platform.** Responsive web app: staff/admin screens plus one public tracking page. Customers have no accounts.
2. **Roles.** ADMIN (owner) and STAFF. See the permission table in section 5.
3. **Customers.** Created by staff. `contact_number` is required and unique after normalizing (strip spaces, dashes, parentheses; keep an optional leading `+`; 7 to 15 digits). `email` is optional but validated if given. Search by partial name or number. Edit and deactivate/reactivate are allowed.
4. **Pricing.** `service.price` is per kg (2 decimals). `weight` is in kg, more than 0, up to 2 decimals. Item `price` = weight × service price at order time, **rounded half-up to 2 decimals**, stored as a snapshot (later price changes never alter old items). `total_amount` = sum of item prices. At least 1 item per order. **Decimal arithmetic only, never floats.**
5. **Order creation.** One transaction creates: the order (status WAITING), its items, a `status_history` row (WAITING), and a `payment` row (UNPAID, amount = total). Any failure rolls back everything. No order editing or cancelling.
6. **Queue number.** Resets to 1 each shop-local day. Shown as 3 digits (`007`). Unique per (`queue_date`, `queue_number`). Assigned atomically inside the order transaction. Strict FIFO by (`queue_date`, `queue_number`). **Active order** = status is not CLAIMED. **Next to process** = oldest WAITING order. No manual reordering.
7. **Status changes.** Forward only, one step at a time. The caller never chooses the target status; the system computes the next one. No skipping, reverting or cancelling. Every change writes a `status_history` row in the same transaction. If two people advance at once, exactly one succeeds. The transition rules live in **one place** in the code.
8. **Payment.** Cash, recorded manually. One payment per order. Staff can mark it PAID at any time. **CLAIMED requires status READY_FOR_PICKUP and payment PAID.** No partial payments, refunds or online payment.
9. **Notification.** Sent automatically **once**, when an order becomes READY_FOR_PICKUP, and for no other status. Runs after the status change commits and never blocks or rolls it back. SMS is always attempted; email only if the customer has one. Up to 3 attempts (retry after 1 min, then 5 min). Each result is recorded. Staff can manually resend (new row, trigger MANUAL, only while the order is READY_FOR_PICKUP). At most one AUTO notification per (order, channel).
   - SMS/email body: `Hi {customer_name}, your laundry (Queue #{queue_number}) at {shop_name} is ready for pickup.`
   - Email subject: `Your laundry is ready for pickup - Queue #{queue_number}`
   - Use **mock adapters only** unless a real provider is filled in (section 8).
10. **Customer tracking page (public).** Customer enters queue number (accept `7`, `007`, `#007`) plus contact number. Both must match; the latest matching order is shown. It shows **only**: queue number, status label and meaning, time of last status change. Any mismatch gives the same generic "not found" response. Rate limited, not cached.
11. **Unclaimed laundry.** Unclaimed = status READY_FOR_PICKUP. **Ready time** = time of that order's READY_FOR_PICKUP history row. **Overdue** when (now − ready time) is **strictly greater than** `unclaimed_threshold_days` × 24 h. Default threshold 3, editable by ADMIN, applied immediately.
12. **Reports (ADMIN only).** Sales per day (sum of PAID amounts by `paid_at`, over a date range, zero-sales days shown), completed orders by CLAIMED time, unclaimed list. Dashboard: active orders per status, today's sales, today's completed count, overdue count. No CSV/PDF export.
13. **"Real-time".** Polling every 10 seconds (one named constant) on the queue view, unclaimed view and tracking page.
14. **Security.** Hashed passwords, server-side validation and authorization, default-deny on all non-public routes, secrets in environment variables only, no personal data in logs, rate limits on login and tracking lookup.
15. **No hard deletes** anywhere. Use `is_active`. Store all timestamps in **UTC**; use the shop timezone only for display and day boundaries.

## 5. Who can do what
| Capability | ADMIN | STAFF | Public |
|---|---|---|---|
| Manage users, services, settings | ✔ | ✘ | ✘ |
| Customers, orders, queue, advance status, payment/claim, resend notification, unclaimed list | ✔ | ✔ | ✘ |
| Dashboard and reports | ✔ | ✘ | ✘ |
| Tracking lookup | – | – | ✔ |

## 6. v1 feature list (deck)
Customer info · order creation with queue number · status flow · customer status tracking · automatic ready-for-pickup notification · payment and claim recording · admin dashboard and basic reports · unclaimed laundry monitoring.

## 7. NOT in v1 (do not build)
Machine assignment or capacity · inventory · online payment · partial payments, refunds, discounts · order cancel/edit/status revert · customer accounts or self-registration · native mobile apps · delivery · loyalty · multi-branch · websockets/push · report export · editable message templates · password-recovery email · receipt printing · anything not in section 6.

## 8. Fill these in (6 blanks)
| Item | Value | If left blank |
|---|---|---|
| Tech stack (language, framework, database) | `______` | Agent must stop in P1 and offer 2 to 3 options. It may not choose. |
| SMS provider | `______` | Mock adapter only |
| Email provider | `______` | Mock adapter only |
| Shop timezone (IANA name, e.g. `Asia/Manila`) | `______` | Agent must ask |
| Currency (ISO code) | `______` | Agent must ask |
| Shop name | `______` | Agent must ask |

---

# Part B: Master prompt (standing instructions)

```text
ROLE
You are a senior engineer building the Laundry Queuing & Notification System for
a small laundry shop. You work one phase at a time. You do not guess. You prove
your work by running it.

SOURCES
Only two: my messages, and docs/SPEC.md. Your general knowledge is not a source
of requirements. If they conflict, stop and ask.

RULES
1. TRACE EVERYTHING. Every table, column, endpoint, screen, message and rule you
   create must cite a SPEC section/rule number or a logged assumption. No
   citation, no creation.
2. DON'T FILL GAPS SILENTLY.
   - Low-impact (wording, field lengths, cosmetic UI): pick a sensible value, log
     it in docs/ASSUMPTIONS.md, keep it easy to change, continue.
   - High-impact (data model, money, status logic, notifications, security,
     external services): STOP and ask. Give numbered questions (max 5) with a
     proposed default for each.
3. VERIFY TOOLS. Before using any library, CLI flag or API, confirm it exists by
   running a real command (package manager info, --help) or reading the official
   docs. Never write version numbers or API details from memory. If you can't
   verify something, mark it UNVERIFIED and put it behind an interface with a mock.
4. PROVE IT. Say "done" or "works" only for things you ran. Show the command and
   real output (trimmed). Otherwise write NOT RUN.
5. STAY IN THE PHASE. Build only what the current phase asks. Never build
   anything in SPEC section 7. Don't refactor earlier phases except to fix a
   defect, and report that.
6. ONE HOME PER RULE. Each business rule lives in exactly one place. APIs and UI
   call it; they never re-implement it.
7. MONEY AND TIME. Decimal arithmetic for money, never floats. Timestamps in UTC.
8. TESTS FIRST. Write tests from the phase's acceptance criteria, then implement.
   Expected values in tests are hand-computed literals, never produced by the
   code under test.
9. Don't edit docs/SPEC.md. If it needs a change, propose a diff and wait.

PHASE LOOP
Plan (list files, tests, open questions; ask if high-impact) -> write tests ->
build -> run tests/linter/type-check and paste output -> scan your own work for
anything with no spec number (remove it or list it as a proposed addition) ->
report -> update docs/PROGRESS.md -> STOP and wait for "APPROVED P#".

REPORT FORMAT
1. What was built (file - purpose)
2. Spec items covered and the test for each
3. Evidence (command + real output)
4. Assumptions added and open questions
5. Deviations from the spec (none, or explained)
6. NOT RUN / not done

Reply with one sentence confirming you understand, then wait for the first phase.
```

---

# Part C: Phase prompts

Send one at a time. Each ends with the report and a stop.

### P0: Read-back (no code)
```text
PHASE P0: READ-BACK. Follow the phase loop. NO CODE.

Read docs/SPEC.md fully. Then write back, in your own words:
a. The system's purpose in 3 sentences or less.
b. The five statuses in order, and the single rule that triggers notification.
c. Roles and what each may do.
d. Every table and column: the deck ERD first, then the additions, clearly separated.
e. Which of the 6 blanks in section 8 are empty, and what you'll do about each.
f. Every contradiction, ambiguity or gap you found, numbered, each with a
   proposed fix marked low- or high-impact.
g. Anything you would have to invent that the spec doesn't cover.

Create docs/PROGRESS.md (phases P0 to P10, table: Phase | Status | Notes) and
docs/ASSUMPTIONS.md (empty table: ID | Assumption | Phase | Status).

ACCEPTANCE
- Items a to d match the spec exactly. Items f and g are specific, not generic.
- No code written, nothing installed.
```

### P1: Setup and database
```text
PHASE P1: SETUP AND DATABASE. Follow the phase loop.

GATE: if the tech stack in SPEC section 8 is blank, STOP and offer 2 to 3 stack
options (language, framework, database, why it fits a small shop with desktop
staff and a phone-friendly tracking page, main trade-off). Wait for my choice.

TASKS
1. Verify every tool and library you plan to use (rule 3). Add only dependencies
   you actually use. Commit the lock file.
2. Scaffold: health-check endpoint, test runner, linter, type-check (if the
   language has one), migration tool, .env.example (variable NAMES only), README
   with exact install/migrate/test/run commands.
3. Migrations for exactly the tables and columns in SPEC section 3. No queue
   table. Enforce in the DATABASE (not just app code):
   - primary keys; foreign keys with ON DELETE RESTRICT
   - CHECK: weight > 0; all money columns >= 0; attempts >= 0
   - UNIQUE: (queue_date, queue_number); customer.contact_number;
     app_user.username; payment.order_id; and one AUTO notification per
     (order_id, channel) (use the nearest equivalent your database supports
     and explain it)
   - money as fixed-point decimal with 2 decimals
   Migrations must run up, down, and up again from empty.
4. Dev seed: one ADMIN (username and password from environment variables, never
   hard-coded) plus settings shop_name and unclaimed_threshold_days = 3. No
   sample customers or orders.
5. docs/DATA_DICTIONARY.md: every table and column, type, nullable, and the spec
   section that justifies it.

ACCEPTANCE
- From a clean checkout, the README commands install, run tests and start the
  app. Paste the real output, including the health-check response.
- One test per constraint proves the database rejects the violation.
- Deleting a customer that has an order is rejected.
- Migrated columns exactly equal the data dictionary (show the comparison).
```

### P2: Login, roles, services and settings
```text
PHASE P2: LOGIN, ROLES, SERVICES, SETTINGS. Follow the phase loop.

ASK FIRST: session cookies or tokens? Recommend the idiomatic option for the
chosen stack with reasons, and wait for my answer.

BUILD (each with tests)
1. Password hashing with a verified library. Passwords are never stored, logged
   or returned.
2. Login and logout. A wrong username and a wrong password give the same generic
   error. Inactive users cannot log in, and their existing sessions stop working.
3. Current-user endpoint (user_id, name, role only).
4. Authorization guard: default-deny; role checks on the server per SPEC section 5.
   Public routes are an explicit allow-list (health check now, tracking later).
5. ADMIN user management: create, list, change role, deactivate/reactivate, reset
   password. No delete. An ADMIN cannot deactivate themselves.
6. Login rate limiting.
7. Services (ADMIN writes): create, edit, deactivate/reactivate. STAFF list shows
   only active services; ADMIN list shows all. A price change affects future
   orders only.
8. Settings (ADMIN writes): shop_name (non-empty), unclaimed_threshold_days (whole
   number >= 1; log your upper bound as an assumption).
9. Basic screens for users, services and settings.

ACCEPTANCE
- Unauthenticated -> 401 on every protected route.
- A table-driven test runs every registered route as ADMIN, STAFF and
  unauthenticated and checks the results against SPEC section 5.
- No response or log line contains a password or hash.
- Login rate limit triggers (show the test).
- Test "10.005" as a service price and log whether you reject or round it.
- Duplicate service_name behavior is decided, logged and tested.
```

### P3: Customers
```text
PHASE P3: CUSTOMERS. Follow the phase loop.

BUILD (each with tests)
1. Contact-number normalization (pure function). Table-driven tests including:
   "0917 123 4567", "(02) 8123-4567", "+63 917 123 4567" (valid),
   "12345" (too short), "abcdefghij" (non-numeric), "" (empty).
2. Create customer: name (trimmed, required), contact_number (normalized,
   required), email (optional, validated). A duplicate number is rejected with an
   error showing the existing customer's id and name.
3. Search by partial name (case-insensitive) or partial number. Active only by
   default, with an option to include inactive.
4. Customer detail with their orders (read-only; use fixtures since orders come
   in P4).
5. Edit (same validation; duplicate check excludes the customer being edited).
6. Deactivate/reactivate. No delete route.
7. Screens: search/list, create/edit form, detail. Put the search box on the
   create screen so staff check for an existing customer first.

ACCEPTANCE
- All normalization vectors behave as specified.
- "0917 123 4567" and "09171234567" are treated as the same customer and the
  second is rejected.
- Invalid email rejected; missing email accepted.
- Search returns hand-written expected lists.
- No delete route exists. Unauthenticated -> 401.
```

### P4: Orders, queue number and queue view
```text
PHASE P4: ORDERS, QUEUE NUMBER, QUEUE VIEW. Follow the phase loop.

BUILD IN THIS ORDER (each with tests)
1. Line amount (pure, decimal, no floats): weight × service price, rounded
   half-up to 2 decimals. Required vectors:
     2.50 × 30.00 -> 75.00
     1.33 × 25.00 -> 33.25
     0.75 × 19.99 -> 14.99   (exact 14.9925)
     3.10 × 12.35 -> 38.29   (exact 38.285, so half-up matters)
     0.01 × 0.50  -> 0.01    (exact 0.005)
   Also: total = sum of line amounts (test with 3 items).
2. Queue number: queue_date = today in the shop timezone. Number = highest used
   that day + 1, starting at 1, assigned inside the order transaction using a
   concurrency-safe mechanism for your database (verify it, and name it in the
   report). Display as 3 digits; 1000+ displays in full.
3. Create order: input customer_id and items [{service_id, weight}]. Validate that
   the customer is active, items are not empty, services exist and are active,
   weights are > 0 with at most 2 decimals. One transaction creates the order
   (WAITING), items (price snapshot), history row (WAITING) and payment (UNPAID).
   Test a forced failure mid-way and prove no partial rows remain.
4. Order confirmation (queue number shown prominently, customer, items, total,
   status) and order detail read (customer, items, payment summary, status).
5. Queue view (read-only): active orders ordered by (queue_date, queue_number),
   each with formatted queue number, customer name, status label, payment status
   and time in current status (from the latest history row). Flag the oldest
   WAITING order as "next to process". Filter by status. Screen grouped by status,
   polls every 10 s, empty state, no advance buttons yet.
6. New-order screen: pick customer (P3 search), add item rows (active services),
   live total using the same shared rule as step 1 (do not duplicate it).

ACCEPTANCE
- All five line-amount vectors pass.
- Empty items, zero/negative weight, 3-decimal weight, inactive customer,
  inactive service and unknown service are all rejected.
- A good order makes exactly 1 order, N items, 1 history row (WAITING), 1 UNPAID
  payment. The rollback test passes.
- First order of the day is 1, the next is 2, a new day restarts at 1, and 20
  parallel creations give 20 distinct numbers.
- A midnight boundary in the shop timezone (23:59 vs 00:01 local) lands on
  different queue dates even when the UTC dates are equal.
- A later service price change doesn't alter existing item prices.
- An unfinished order from yesterday appears before today's orders; CLAIMED
  never appears; "next to process" is the oldest WAITING order even if older
  WASHING orders exist.
- No order edit/cancel route and no manual reorder exists.
```

### P5: Status flow and history
```text
PHASE P5: STATUS FLOW AND HISTORY. Follow the phase loop.

Only these 3 transitions in this phase: WAITING -> WASHING, WASHING -> DRYING,
DRYING -> READY_FOR_PICKUP. Claiming is built in P8 because it needs payment.
Advancing a READY_FOR_PICKUP order must be rejected with a message saying
claiming is a separate action.

BUILD (each with tests)
1. Transition table: one data structure that is the single source of truth.
   API, UI and notifications read it.
2. Advance command: "advance order X to its next status". The caller never
   supplies the target status.
3. In ONE transaction: update the order status and insert a history row (new
   status, UTC time, changed_by). Use an expected-current-status check so two
   simultaneous advances give one success and one conflict.
4. After the transaction COMMITS, publish an in-process event OrderStatusChanged
   {order_id, from_status, to_status, at} with a simple subscriber registry. A
   subscriber that throws must not affect the committed change. No subscriber yet.
5. History endpoint (oldest first: status, label, UTC time, changed_by name),
   shown on the order detail screen.
6. Queue screen: one button per order that moves it forward ("Start washing",
   "Move to drying", "Mark ready for pickup"; log the wording as an assumption).

ACCEPTANCE
- A test covers all 25 (from, to) status pairs: exactly the 3 above succeed and
  the other 22 are rejected. Paste the test list.
- Three advances from a new order leave exactly 4 history rows in order.
- Sending a "status" field in a request body is ignored or rejected.
- 2 parallel advances: 1 success, 1 conflict, exactly 1 new history row.
- A throwing subscriber does not block or roll back the change.
- No revert/cancel route exists. Unauthenticated -> 401.
```

### P6: Automatic notification
```text
PHASE P6: AUTOMATIC NOTIFICATION. Follow the phase loop.

ABSOLUTE RULE: do not write provider-specific HTTP calls or SDK usage from
memory. Real provider code is allowed ONLY if a provider is filled in SPEC
section 8, and ONLY after you read the provider's official docs, cite the URL in
the docs, and check every field name. If blank, build the interface and mock
adapters only.

ASK FIRST: how retries are scheduled. My default is a database-polled worker
(pick PENDING rows where next_attempt_at <= now). Confirm or propose an
alternative.

BUILD (each with tests)
1. Channel interface: send(recipient, subject_or_null, body) returns success or a
   failure message. Channels: SMS, EMAIL.
2. Mock adapters that record what they'd send and can be told to fail. All
   automated tests use mocks and never call a real provider.
3. Message composer (pure): exact strings from SPEC rule 9, using the 3-digit
   queue number and shop_name from settings. Test an apostrophe and non-ASCII
   characters in the customer name.
4. Subscriber for OrderStatusChanged where to_status = READY_FOR_PICKUP only:
   always create an SMS row; create an EMAIL row only if the customer has an
   email. Rows start PENDING, trigger AUTO, attempts 0. A repeated event must not
   create a duplicate AUTO row (rely on the DB constraint).
5. Dispatcher: send due PENDING rows. Success -> SENT with sent_at. Failure ->
   increment attempts, store last_error, retry after 1 min, then 5 min. After the
   3rd failure -> FAILED. Delays are named constants. It never touches the order
   status.
6. Manual resend (ADMIN/STAFF): only while the order is READY_FOR_PICKUP; creates
   MANUAL rows sent through the same dispatcher.
7. Notification log on the order detail screen (channel, recipient, trigger,
   status, attempts, last_error, sent_at).

ACCEPTANCE
- Reaching READY_FOR_PICKUP creates 1 SMS row, plus 1 EMAIL row only if the
  customer has an email. WASHING and DRYING create 0 rows.
- Permanent adapter failure: the order stays READY_FOR_PICKUP and the
  notification ends FAILED with attempts = 3 and a non-empty last_error.
- Fail once then succeed: SENT with attempts = 2, with the 1-minute delay
  verified using a controlled clock.
- The same event twice creates no duplicate AUTO row.
- Message text matches SPEC rule 9 exactly.
- Resend on a non-READY order is refused; on a READY order it adds MANUAL rows
  and leaves AUTO rows unchanged.
- No secrets or full phone numbers in logs.
- The report lists every external call the code can make, each with a docs URL
  or "MOCK ONLY".
```

### P7: Customer tracking page (public)
```text
PHASE P7: CUSTOMER TRACKING PAGE. Follow the phase loop.

BUILD (each with tests)
1. Lookup: accept queue number as "7", "007" or "#007", plus a contact number.
   Normalize the contact number with the P3 function (reuse, don't copy). Return
   the LATEST order where queue number and the customer's contact number both
   match.
2. Response: an allow-list of only: formatted queue number, status code, status
   label, meaning text, time of last status change. Never expose name, contact,
   email, amounts, items, order_id or user names. Log your proposed customer text
   for READY_FOR_PICKUP as an assumption.
3. Wrong queue number, wrong contact and no order must return the identical body
   and status code.
4. Rate limit the endpoint and send a no-store cache header.
5. Mobile-first page: form, then a progress display of the five statuses with the
   current one highlighted, meaning text and last-updated time. Polls every 10 s.
   No login. This is the only public route besides the health check.

ACCEPTANCE
- Correct queue + contact returns the current status.
- The three not-found cases return byte-identical responses.
- A test fails if the response contains any key outside the allow-list.
- "7", "007", "#007" and "0917 123 4567" vs "09171234567" resolve the same order.
- Same queue number on different days: the latest order is returned.
- After a staff advance, the next poll shows the new status.
- Rate limit triggers (show the test). CLAIMED orders show "Claimed", not "not found".
```

### P8: Payment, claim and unclaimed monitoring
```text
PHASE P8: PAYMENT, CLAIM, UNCLAIMED. Follow the phase loop.

BUILD (each with tests)
1. Record payment: set the order's payment to PAID with paid_at (UTC) and
   recorded_by. Amount doesn't change. Allowed at any order status. Already PAID
   -> conflict, nothing changes.
2. Add READY_FOR_PICKUP -> CLAIMED to the SAME transition table from P5, allowed
   only if payment is PAID.
3. Claim command: only when status = READY_FOR_PICKUP and payment is PAID. One
   transaction sets CLAIMED and writes a history row. Distinct errors:
   NOT_READY and PAYMENT_REQUIRED.
4. "Record payment and claim" (counter action): one transaction. If the claim is
   rejected, the payment is NOT recorded.
5. After CLAIMED the order is immutable. Publish OrderStatusChanged for the claim
   (the notification subscriber must ignore it).
6. Order detail: show payment status and amount, with buttons "Record payment",
   "Claim", "Record payment and claim"; enable only valid actions.
7. Unclaimed list: READY_FOR_PICKUP orders, longest waiting first. Columns:
   queue number, customer name, contact number, ready time (from the history
   row; if missing, surface a data-integrity error, don't guess), elapsed days
   and hours, payment status, overdue flag, last notification status.
8. Overdue rule in ONE function: strictly greater than threshold × 24 h, with the
   threshold read from settings at request time. Expose unclaimed and overdue
   counts for the dashboard.
9. Unclaimed screen: overdue rows marked, a "Resend notification" button that
   reuses the P6 action, polls every 10 s.

ACCEPTANCE
- Claim on an unpaid READY order -> PAYMENT_REQUIRED, status unchanged.
- Claim on a paid order that is WAITING/WASHING/DRYING -> NOT_READY.
- Paid + READY -> claim succeeds, history gets a CLAIMED row, order leaves the queue.
- Combined action on a READY unpaid order records both. On a non-READY order it
  records nothing (payment stays UNPAID).
- Paying twice -> conflict, paid_at unchanged. Claiming creates no notification.
- The P5 25-pair test is updated and now shows exactly 4 valid transitions.
- Threshold 3: ready 72h 00m ago is NOT overdue; 72h 01m ago IS (controlled clock).
- Changing the threshold from 3 to 1 immediately flips a 30-hour-old order to overdue.
- Elapsed time uses the READY_FOR_PICKUP row, not the order's creation time.
- No route exists for partial payment, refund, discount or un-claiming.
```

### P9: Dashboard and reports
```text
PHASE P9: DASHBOARD AND REPORTS (ADMIN only). Follow the phase loop.

STEP 1: write docs/REPORT_DEFINITIONS.md before any code. For each metric: exact
definition, source columns, how shop-local day boundaries convert to UTC ranges,
and 2 hand-computed worked examples. Metrics: sales per day, completed orders,
unclaimed laundry, and the four dashboard figures. If anything is ambiguous,
stop and show me the file.

STEP 2: build
1. Dashboard: active orders per status, today's sales, today's completed count,
   overdue-unclaimed count.
2. Sales report: from/to dates (inclusive, shop-local), one row per day including
   zero days, plus a range total.
3. Completed orders: date range; count and list (queue number, customer, total,
   claimed time).
4. Unclaimed report: reuse the P8 function.
5. Validation: both dates required, from <= to, valid dates. Pick a maximum range
   and log it as an assumption.
6. STAFF get 403. No export of any kind.

ACCEPTANCE
- Literal fixture: day A paid orders 100.00, 250.50, 49.50 and day B 80.00 ->
  day A = 400.00, day B = 80.00, total = 480.00.
- UNPAID orders add 0 to sales. Days with no sales show 0.00.
- A payment at 23:30 shop-local lands on that local day even if the UTC date
  differs; a payment at 00:30 local lands on the next local day.
- Completed orders use the CLAIMED history time, not creation time.
- Dashboard counts match the queue view and unclaimed list on the same fixture.
- STAFF -> 403, unauthenticated -> 401.
```

### P10: Final check
```text
PHASE P10: FINAL CHECK. Follow the phase loop. NO NEW FEATURES. Only tests, fixes
for defects (reported as deviations), and documentation.

1. HAPPY PATH (one automated test, mock adapters): ADMIN creates a service ->
   STAFF logs in -> creates a customer with an email -> creates an order -> queue
   shows it as next -> advance to READY_FOR_PICKUP -> exactly 1 SMS and 1 EMAIL
   recorded -> tracking shows "Ready for Pickup" -> unclaimed list shows it ->
   "Record payment and claim" -> tracking shows "Claimed" -> dashboard and sales
   report reflect the payment. Paste the output.
2. NEGATIVE PATHS (automated): skip a status; claim unpaid; claim before ready;
   STAFF opens reports; unauthenticated access to every protected route;
   notification failure leaves the status READY_FOR_PICKUP; tracking with the
   wrong contact number.
3. CONCURRENCY (automated): 20 parallel order creations; 2 parallel advances.
4. SPEC COVERAGE: docs/COVERAGE.md with one row per spec rule, table and column,
   giving the code location and test name. Any row missing either is a failure.
5. ORPHAN SCAN: list every route, screen, table, column, environment variable and
   setting that has no spec number. Remove it or list it as a proposed addition.
6. OUT-OF-SCOPE SCAN: search the code for each item in SPEC section 7 and show
   that none exists (show how you searched).
7. TOOL AUDIT: table of every dependency with its verification evidence; every
   external call with its docs URL or "MOCK ONLY"; grep results for TODO, FIXME,
   placeholder, hard-coded secrets/passwords and commented-out code; everything
   marked UNVERIFIED or NOT RUN in earlier phases.
8. DOCS: final README, .env.example with a one-line purpose per variable, a
   one-page STAFF guide and a one-page ADMIN guide (only features that exist),
   and a "Known limitations" section mirroring SPEC section 7.
9. Run the full test suite, linter and type-check. Paste the summary. Then follow
   the README in a clean directory and paste the output.

ACCEPTANCE: all tests pass; no coverage row lacks code and a test; orphan and
out-of-scope scans show zero unresolved items; every dependency and external call
has evidence.
End with a table of all phases, status and approval.
```

---

# Part D: Recovery prompt (when the agent drifts)

Use this one prompt for most problems:

```text
STOP. Do not continue the phase. Do these now:
1. List everything in your last output (files, tables, columns, routes, screens,
   rules, dependencies) that has no spec number or logged assumption. Remove each
   item, or move it to "Proposed additions" for my approval.
2. For every claim that something works, show the exact command and raw output.
   If you didn't run it, write NOT RUN and run it now.
3. For every library, flag or API you used, show how you verified it. Anything you
   can't verify: mark UNVERIFIED and replace it with an interface plus a mock.
4. If you changed a rule from docs/SPEC.md, revert it and propose the change as a
   diff with reasons.
Then wait for my reply.
```

**New chat, agent forgot everything:**
```text
New session. Read docs/SPEC.md, docs/ASSUMPTIONS.md, docs/PROGRESS.md. Tell me the
last APPROVED phase, the phase in progress, and any open assumptions or questions.
Write no code until I confirm your summary.
```
