# KEMP — Master Feature List (Web + Mobile App)

**Project:** KEMP — Klutch Employment Management Program (formerly the Employment Management Portal)
**Prepared for:** Alfonzo
**Stack:** Laravel 12 + MySQL · Blade/Bootstrap (SmartHR) web · Flutter app on the same Laravel API
**Date:** 2026-08-04
**Purpose:** the complete feature set for the grown product — what is already delivered, and what is planned — split across the **Web Dashboard**, the **Mobile App**, and the **Shared Backend/API**.

**Legend:** ✅ Built · 🟡 Partial · ⬜ Planned

---

# PART A — WEB DASHBOARD (Admin / HR)

## A1. Authentication & Access Control
| # | Feature | Status |
|---|---|---|
| A1.1 | Secure login / logout (Laravel session) | ✅ |
| A1.2 | Role-based access — Admin, HR, Manager, Employee (Spatie RBAC) | ✅ four roles, each landing on the only area it can reach. Manager is a first-class role with its own dashboard (A10), not a permission bolted onto the employee portal |
| A1.3 | Granular permissions per role (18 seeded) | ✅ |
| A1.4 | Roles & permissions editor UI | ✅ |
| A1.5 | Profile page + change password | ✅ |
| A1.6 | Password reset via email ("forgot password") | ✅ request → emailed link → new password; revokes app tokens and push. Needs a real `MAIL_MAILER` to leave the box |
| A1.7 | Two-factor authentication (2FA) for Admin/HR | ✅ TOTP, any authenticator app; secret encrypted at rest, 8 single-use recovery codes, optional company-wide requirement on Admin/HR. **Setup is by scan or by typed key** — the `otpauth://` URI is drawn as a QR beside the key, never instead of it: a camera that will not focus, a desktop authenticator and a password manager on the same machine all need the key, so replacing one with the other would have been a regression dressed as an improvement. The code is **inline SVG**, not an `<img>` pointing at a route — it encodes the TOTP secret, and a second request for it would put that secret in the web server's log and possibly in a proxy cache; inline, it inherits the protection the page already has. SVG rather than PNG so nothing depends on `ext-gd` being compiled into the host's PHP. The encoder is `bacon/bacon-qr-code` and lives in `App\Support\QrCode`, which `Totp` does not reference: the algorithm stays free of any vendor, and Reed-Solomon over a Galois field is not the forty lines of obvious arithmetic that HMAC is — hand-rolling it would have bought a class of bug that shows up as "my phone will not scan this" rather than as a failing test. A code that cannot be encoded returns null and the page draws the key alone rather than falling over |
| A1.8 | Login activity & audit trail (who did what, when) | ✅ immutable log of sign-ins, failed attempts, lockouts, timeouts, password and settings changes; filterable by event, person, date and IP. Admin-only. **The trail was silent about the whole mobile app until 2026-09-15, and silent about self-service password changes on both halves.** The design was right and the wiring was not: `AppServiceProvider::recordAuthenticationEvents` hangs the trail off the framework's auth events precisely so one listener covers every door, and its own comment named "the mobile API's token endpoint" as one of them — but token login checks the hash itself and never calls `Auth::attempt`, so `Login` and `Failed` were never dispatched and **not one sign-in from a handset had ever been recorded**. Separately `PASSWORD_CHANGED` had a label and a badge colour on the screen and was written by nothing at all, so a password changed by whoever currently holds an account left no trace — the one move an attacker makes on every account they take; only an HR-initiated reset was logged. Both are closed: the API now dispatches the framework events rather than logging separately, so there is still one writer; the disabled-account case is recorded directly because the framework has no event for "credentials were right but the account is switched off", which is the most interesting line on the screen; "sign out everywhere" — the lost-phone endpoint — records how far it reached; and every entry now names its door in words (**"from the mobile app"**, **"from the web dashboard"**) rather than the guard name, because an administrator reading this after an incident needs to know whether a phone was involved and `via sanctum` does not tell them. **A contact edit is deliberately not a security event** — only the sign-in address changing is recorded, because logging every new phone number would bury the one line that matters, and pointing the address at yourself is step one of taking an account over by password reset. **Being rate limited on the app is recorded too**, which it was not: the API answered 429 and said nothing, so somebody working through a password list produced a run of failed attempts that stopped dead at five with no line explaining why — reading like the attacker gave up rather than like the fence doing its job. Wiring that up surfaced a separate latent defect in `bootstrap/app.php`: the API's error renderer turned **any** `HttpResponseException` into a 500, so a response a caller had deliberately built to short-circuit with was discarded and replaced. Nothing had needed one until the limiter did |
| A1.9 | Session timeout + forced re-login policy | ✅ per-company idle timeout, off by default; resets on activity so long work is never interrupted, and a timeout is logged apart from a deliberate sign-out |
| A1.10 | Rate limiting on the password form | ✅ five wrong passwords per address per source per minute, then refused with the wait remaining. Counted on email **and** IP, so one person's mistakes cannot lock out a colleague behind the same office address. Raises Laravel's `Lockout` event rather than a bare 429, so every lockout lands in the audit trail and on the Security panel |

## A2. Company & Organization Setup
| # | Feature | Status |
|---|---|---|
| A2.1 | Company profile (name, logo, address, timezone) | ✅ |
| A2.2 | Office / branch management | ✅ |
| A2.3 | Office GPS coordinates + geofence radius | ✅ stored, and enforced when the company switches A4.16 on |
| A2.4 | Departments — CRUD & assignment | ✅ |
| A2.5 | Designations / job titles | ✅ |
| A2.6 | General settings page | ✅ |
| A2.7 | Company holiday calendar | ✅ |
| A2.8 | Weekend / working-days configuration per office | ✅ editable working week, company-level — the same definition leave charging, absence and the roster all read. A seven-day week is expressible; a zero-day one is refused |
| A2.9 | Attendance & leave policy rules engine | 🟡 the policies themselves are configurable — working week, reminder and auto-close windows, geofence, 2FA requirement, idle timeout, directory contact details — but there is no conditional rule builder |
| A2.10 | Multi-company (SaaS tenancy) support | ⬜ **not built — but its foundation is now tested rather than assumed.** `CLAUDE.md` records that "the schema is company-scoped throughout, so this is a routing and onboarding job rather than a data-model one". That claim is what the whole feature rests on, and it had never been verified. It holds, on both counts. **Schema**: 25 business tables carry `company_id`; the 17 that do not are framework tables (cache, jobs, sessions, migrations), `companies` itself, Spatie's role tables, or rows scoped through a user (`notifications`, `push_devices`, `personal_access_tokens`). **Queries**: `tests/Feature/CrossCompanyIsolationTest` stands up two whole companies and, as one company's administrator, attempts 36 real crossings — opening, editing and deleting the other company's employees, departments, designations, offices, shifts, holidays, leave types and announcements; publishing their announcement; deciding their leave; reading their employee's document vault and checklist; three API endpoints including the document download; and seven listings that must not merely refuse but must not *contain* the other company's rows. All refuse. **The suite was mutation-checked rather than trusted for passing first time**: removing the guard from `EmployeeDocumentController` and `DepartmentController` makes exactly the right three tests fail, one of them on a 200 for another company's document list. Three guard idioms are in use across the controllers — `authorizeCompany`, `authoriseCompany` and a bare `abort_unless` — plus ownership checks in self-service and team checks in the manager paths; reading each proves nothing about the next one somebody writes, which is why this is a test and not a review. **The row is mis-labelled and the remaining work is smaller than ⬜ implies** — see `Multi-Company_Tenancy-Assessment.md`, which is the full working. Two companies can already be created (`emp:install --force`, or `--company-id=N` to attach an admin to an existing one), administered separately, and cannot see each other. What is left is **onboarding, one correctness fix, and a product decision**: creating a company is a command-line operation and there is deliberately no sign-up route, because a public "create your company" form on the client's own server would let anybody on the internet create tenants on it. **The correctness fix that had to come first is done**: the `?? Office::value('company_id')` fallback, repeated at 23 call sites, silently handed a user with no company whichever company owns the first office row — harmless on one company, a cross-tenant read on two. `companyId()` now lives once on the base `Controller` and fails closed, the 19 duplicated copies are gone, and three tests cover it, verified by restoring the old behaviour and watching a company-less admin get 200 on the dashboard. Also open, and recorded so it is a decision rather than a discovery: Spatie's `roles`/`permissions` carry no `company_id`, so all companies share one set — defensible, since the four roles and 19 permissions mean the same thing everywhere |

## A3. Employee Management
| # | Feature | Status |
|---|---|---|
| A3.1 | Employee CRUD + deactivate | ✅ deleting anyone who has ever clocked in is refused — `attendance_logs` cascades, so it would take the hours a finished payroll was calculated from. They are set to Terminated instead; deletion stays for records typed in by mistake |
| A3.2 | Assign department / office / designation | ✅ |
| A3.3 | Employment details (code, job title, hire date, status) | ✅ |
| A3.4 | Work mode — office / WFH / hybrid | ✅ |
| A3.5 | Login credential creation & reset | ✅ **Sign-in Account** panel on the employee page: create the login, set or generate a password (shown once, never stored readable), reset it, change the role, disable and re-enable. An employee record and a login are separate rows, so adding somebody to the payroll deliberately does not give them one |
| A3.6 | CSV / Excel bulk import | ✅ 11 columns incl. office/department/designation/manager, matched by name; whole file validated before anything is written, every problem reported at once; department required because it carries the shift; template download |
| A3.7 | Employee profile photo upload | ✅ JPG/PNG/WebP up to 2 MB; replacing one deletes the old file, and saving without one keeps what is there |
| A3.8 | Document vault (contract, ID, certificates) with expiry alerts | ✅ seven document types, held on the private disk and streamed through the app — never a public URL. Anything with an expiry date is chased to HR 30 days out, once per document, and deleting an employee takes their files with them |
| A3.9 | Emergency contact & personal details | ✅ contact name, phone and relationship, plus personal email, address, national ID and blood group |
| A3.10 | Org chart / reporting-manager hierarchy | ✅ printable nested tree from one query; anybody whose manager has left shows at the top rather than vanishing |
| A3.11 | Employee export (CSV / Excel) | ✅ same columns as the bulk import, in the same order, so an export can be edited and fed back in. Honours the filters on screen |
| A3.12 | Onboarding & offboarding checklists | ✅ company-standard steps with an owner and a due offset; raising a list **copies** them onto the person, so editing a template never rewrites history and deleting one leaves finished checklists intact. Every tick records who and when |

## A4. Attendance Management *(core module)*
| # | Feature | Status |
|---|---|---|
| A4.1 | One-tap Check In / Check Out (button-based) | ✅ |
| A4.2 | Server-authoritative timestamps (device clock never trusted) | ✅ |
| A4.3 | GPS + IP captured per punch (record-only, non-blocking) | ✅ |
| A4.4 | Duplicate-punch cooldown | ✅ |
| A4.5 | Auto in/out detection based on last punch | ✅ |
| A4.6 | Attendance log table (employee, office, type, time, status) | ✅ |
| A4.7 | Filterable, paginated attendance history | ✅ |
| A4.8 | Late / early-leave status vs assigned shift | ✅ |
| A4.9 | Daily summary (present / late / on leave / absent / headcount) | ✅ |
| A4.10 | Monthly attendance scoring (on-time %, late count) | ✅ |
| A4.11 | Weekly rollup summaries | ✅ one row per week — present, leave, absent, late, on-time % and attendance %. Weeks are clipped to the window so a short first week is reported as short. Schedulable by email |
| A4.12 | Manual attendance entry / correction by HR (with audit reason) | ✅ |
| A4.13 | Attendance regularisation requests (employee raises, HR approves) | ✅ |
| A4.14 | Overtime calculation & tracking | ✅ |
| A4.15 | Break in / break out punches | ✅ button on the employee portal, and the note under it now states **this shift's** break policy rather than a fixed sentence. It used to read "breaks are not counted as worked time" for everybody, which A5.7 made false for a paid break — see the note in that row |
| A4.16 | Geofence enforcement (block punch outside office radius) | ✅ off by default; exempts WFH/hybrid staff, offices with no coordinates, and punches that arrive without a location. Refusal names the distance |
| A4.17 | Auto-absent marking for missed days (scheduled job) | 🟡 absence stays derived; nightly job refreshes the scores that count it |
| A4.18 | Missing-checkout auto-close policy | ✅ closes at the scheduled shift end, marked `source: auto` |
| A4.19 | Live "who's in right now" board | ✅ four buckets that partition the roster — on the clock (breaks flagged), been and gone, on approved leave, unaccounted for. Refreshes each minute, pauses when the tab is hidden |

## A5. Shift & Schedule Management
| # | Feature | Status |
|---|---|---|
| A5.1 | Shift creation (start, end, grace period) | ✅ |
| A5.2 | Shift assignment per department | ✅ |
| A5.3 | Weekly roster view | ✅ leave, holidays and company weekend aware |
| A5.4 | Shift-driven attendance validation | ✅ |
| A5.5 | Per-employee shift override | ✅ |
| A5.6 | Rotating / night shift patterns | ✅ |
| A5.7 | Break rule configuration | ✅ **each shift says what its break *means*, in two checkboxes.** `shifts.break_minutes` had always been a number with one hardcoded reading — unpaid, and deducted only as long as the break somebody actually punched — and both halves of that are company policy rather than arithmetic. **Break is paid**: nothing comes off, live or settled, and the shift's scheduled hours stop subtracting it too, because taking it out of one side and not the other would hand every employee on that shift a break's worth of overtime a day. **Deduct at least this much**: the shift's break is the *floor*, however short the punched one — which is what "an unpaid 30-minute lunch" normally means, and it closes the half of the incentive problem the old code left open. A day with no break punches already lost the nominal break, so before this a ten-minute lunch earned 20 minutes of overtime while the colleague who never touched the button earned 30; `overtimeFor` already named that incentive as the wrong one to build into a payroll figure, and this is the other end of it. **Both default false, which is exactly what every existing shift does today** — nothing about a live company's payroll moves until somebody ticks a box, and there is a test that says so. The policy is read in **one place** (`Shift::settledBreakDeduction`), which also removed a split brain: `workedMinutes` deducted the punched break and `overtimeFor` separately deducted the nominal one, so the rule lived in two functions that had to agree. `workedMinutes` now reports present time less only what the shift says comes off it, with the accounting split into `presentMinutes` and `punchedBreakMinutes` so the policy has somewhere to apply. **A live counter honours only the paid flag** — the nominal break has not been taken yet at five past nine, and a minimum cannot be judged until the break is over. **The policy reaches the person taking the break, on both halves — which it did not at first.** On the app, `/attendance/today` carried the shift's name, window and grace period but nothing about its break, so it computed hours correctly and could not say why; the three fields now travel with the shift and the Clock screen states the consequence under the button (see B2.6). On the **web portal the omission was worse than silence**: the page asserted flatly that *"breaks are not counted as worked time"*, which stopped being true the day a shift could mark its break paid — a false statement, on the screen where somebody decides whether to take one, addressed to the only person it mattered to. It now reads the shift's actual policy, and a paid break in progress says the clock is still running rather than *"worked time is paused"*, which would have been wrong in the most alarming direction. **Not built**: multiple named break windows, or a break the shift insists is taken between set hours. Nobody has asked, nothing in the schema wants it, and it would be a builder invented rather than needed |
| A5.8 | Roster drag-and-drop planner + publish to staff | ✅ **a palette of shift chips above the grid; drag one onto a day, or drag a day onto another to move it.** The design decision that makes it safe: **drag-and-drop writes into the selects that were already there and posts through the form that was already there** — no new endpoint, no new validation surface, and no second idea of what the roster says, because the select is the value and the chip is only a picture of it. Turn the script off and the planner is exactly what it was. That matters more than usual here: **HTML5 drag events do not fire on a touch screen at all**, so a planner that had replaced the selects would have quietly excluded anybody holding a tablet, and left no keyboard path either. The palette is `hidden` in the markup and unhidden by the script, because an instruction to drag something is worse than no instruction at all on a browser that cannot honour it. A day dragged onto another **moves** — copying instead would silently double a shift every time somebody fixed a mistake, which is the more expensive way to be wrong. Cells changed since load carry an amber edge, distinct from the Draft badge, which means something else entirely (planned but not published); leaving with unsaved changes warns, because the Draft badges on screen otherwise make it look as though something was kept. **Verified by driving a real browser**, not only by asserting markup: palette→day, day→day move, the Clear chip, and a drop onto the day it came from were each dispatched as real drag events, then the form was posted and the row read back out of MySQL. That pass found a genuine defect — the drop repaints the source cell and so destroys the element the drag started from, and `dragend` on a detached node never reaches the listener on `document`, leaving the cell somebody had just moved a shift *out of* stuck at 45% opacity, looking disabled, for the rest of the session. **Not built**: multi-select, drag across a whole row, and a modifier key to copy rather than move |
| A5.9 | Shift swap requests between employees | ✅ |

## A6. Leave Management
| # | Feature | Status |
|---|---|---|
| A6.1 | Leave types (annual, sick, unpaid, casual…) | ✅ |
| A6.2 | Leave request submission | ✅ |
| A6.3 | Multi-step approval workflow (manager → HR) | ✅ |
| A6.4 | Leave balance tracking & accrual rules | ✅ per-type: all at once, or a twelfth a month pro-rated from the hire date. The nightly job only ever raises a balance, so an HR adjustment is never undone |
| A6.5 | Leave history & status management | ✅ |
| A6.6 | Company leave policy configuration | 🟡 types + holidays + weekend config; no rules engine |
| A6.7 | Team leave calendar / conflict detection | ✅ month grid, weekend- and holiday-aware, filterable by department. Pending is drawn alongside approved so cover is not granted twice onto one day |
| A6.8 | Leave ↔ attendance integration (leave day ≠ absent) | ✅ |
| A6.9 | Carry-forward & year-end processing | ✅ capped by the type (null uncapped, 0 off); an overdrawn balance starts at zero rather than in debt, and the roll is safe to run twice |

*Built so far: leave types, the employee self-service screen (balances, apply, withdraw),
weekend- and holiday-aware day counting, balance enforcement, the company-wide leave register
for Admin/HR, and the two-step approval chain — line manager, then HR. An employee with no
manager set goes straight to HR. Days are deducted only on final approval.*

*Approved leave now feeds attendance: it is reported as leave rather than absence on the
dashboard, the attendance overview and the department report, and the monthly score measures
absence against working days only. Weekends and company holidays count as neither.*

## A7. Reporting & Analytics
| # | Feature | Status |
|---|---|---|
| A7.1 | Attendance report (date range, office, department filters) | ✅ |
| A7.2 | PDF export | ✅ |
| A7.3 | Excel export | ✅ |
| A7.4 | Late employee report | ✅ |
| A7.5 | Attendance outlier report | ✅ |
| A7.6 | Department report | ✅ |
| A7.7 | Historical attendance analysis | ✅ |
| A7.8 | Company adherence overview | ✅ |
| A7.9 | Location & IP columns in exports | ✅ |
| A7.10 | Leave reports | ✅ days taken per employee split by the company's own leave types, pending days separately, and the year's unspent entitlement |
| A7.11 | Overtime reports | ✅ |
| A7.12 | Scheduled report email delivery (daily/weekly/monthly) | ✅ standing orders with a PDF or Excel attachment, sent at 07:00 in the company's timezone; recipients need no login; "Send Now" to test one. Needs a real `MAIL_MAILER` to leave the box |
| A7.13 | Custom report builder (pick columns + filters) | ✅ 18 columns over three groups, filtered by office, department, work mode and period; exports to PDF and Excel like the fixed reports |
| A7.14 | Payroll-ready export (hours worked per employee per period) | ✅ |

> **The report window is also the export's filename, and it used to be
> unvalidated.** `from` and `to` came straight off the query string and were
> concatenated into the name handed to `Excel::download()` and to dompdf — in
> `ReportController::handle` and again in `AttendanceController::reportData`. A
> filename is a path, and `maatwebsite/excel` wrote outside the configured disk
> when given a caller-controlled one through **3.1.69 (CVE-2026-84374, high)**,
> which is the version this project was pinned to. Both ends are now validated
> as `Y-m-d` at both call sites, and the dependency is on 3.1.70;
> `league/commonmark` went 2.8.3 → 2.10.1 in the same pass, clearing two
> medium advisories, and `composer audit` is clean. The validation is the half
> that stays true after the next upgrade — a controller that hands unfiltered
> request input to a file writer is one dependency bump away from the same hole.

## A8. Dashboard
| # | Feature | Status |
|---|---|---|
| A8.1 | Live tiles — present, late, absent, headcount | ✅ |
| A8.2 | Recent attendance feed | ✅ |
| A8.3 | Charts / visualisation | ✅ |
| A8.4 | Role-specific dashboards (Admin vs HR view) | ✅ admin opens on security and the trail; HR on approvals and expiring documents |
| A8.5 | Configurable widgets | ✅ eight panels, chosen per user rather than per role; a panel the viewer lacks permission for is never shown, never offered and cannot be saved. A hidden panel costs no queries |
| A8.6 | Trend comparison (this week vs last week) | ✅ like for like — both windows run Monday to the same weekday, so a Tuesday is compared with a Tuesday rather than a finished week |

## A9. Notifications (Web)
| # | Feature | Status |
|---|---|---|
| A9.1 | In-app notification bell + centre | ✅ one inbox shared by the dashboard and the portal |
| A9.2 | Email notifications | 🟡 leave emails queued and rendering; MAIL_MAILER still `log` |
| A9.3 | Late-arrival alert to HR | ✅ one digest a day naming everybody and how late they were — not an alert per person. Silent on a day with no lateness |
| A9.4 | Leave request / approval / rejection alerts | ✅ routed by NotificationService, both stages |
| A9.5 | Schedule update alerts | ✅ publishing a roster tells each affected employee once, covering the whole range rather than one message per day. In-app, email and push |
| A9.6 | Missing-checkout reminder | ✅ sent once, a configurable grace after the shift ends |

## A10. Manager Workspace *(team leads)*

> A parallel area at `/manager/*`, gated `role:manager` **and**
> `permission:view-team`. Not part of the admin app: that group is wrapped in
> `role:admin|hr`, which refuses a manager whatever permission they hold, so the
> role can only be reached by a group of its own.
>
> Scope is `employees.manager_id` — **direct reports only**, resolved by
> `App\Services\ManagerScope`, which every query goes through. No new tables:
> the reporting line already existed. Leave approval is a single hop (manager →
> HR), so the scope deliberately does not recurse down the tree.
>
> Managers are read-only outside approvals. Correcting attendance is
> `manage-attendance`, planning the roster is `manage-shifts`, exporting is
> `export-reports`, and HR-grade PII sits behind `manage-employees` — none of
> which this role holds, and none of which this area asks for.

| # | Feature | Status |
|---|---|---|
| A10.1 | Manager dashboard — team, today, and what needs deciding | ✅ tiles (team, on the clock, present, late, on leave, unaccounted), leave and swaps waiting on *this* manager, missing clock-outs from finished days, 7-day published coverage with unplanned working days flagged, today's team list and recent punches. Every figure counted from real rows; nothing is placeholder |
| A10.2 | Team list + search | ✅ direct reports with today's status, shift and worked hours; searchable by name, code or department |
| A10.3 | Team member detail | ✅ employment facts, leave balances, upcoming published shifts, 30 days of attendance and leave history. Personal PII deliberately excluded |
| A10.4 | Team attendance board (any past date) | ✅ same status vocabulary as the employee's own history and the app — one `dayStatus`, so nobody reads two different words for one day |
| A10.5 | Team punch log | ✅ filterable by person, date, type and status. An employee id outside the team narrows to nothing rather than reaching past the scope |
| A10.6 | Team schedule (published roster) | ✅ fortnight grid, leave outranking a rostered shift, drafts invisible exactly as they are to staff |
| A10.7 | Team approvals inbox | ✅ the existing controller in the manager shell; leave and shift swaps, with clashes flagged. Writes stay on the one portal endpoint |
| A10.8 | Team-scoped reports | ✅ Late Arrivals, Overtime and Weekly Rollup through the same `ReportService` and the same results partial the HR reports use, handed the team as an id list. Print only — no PDF/Excel, which is `export-reports` |
| A10.9 | Manager notifications | ✅ already routed by `NotificationService::approversFor` — a request awaiting a manager goes to that manager. No new events, deliberately: a notification per punch would be spam |
| A10.10 | Manager role on mobile | ✅ no change needed — the app derives manager mode from `permissions`, which `/auth/me` already returns, and `/team/attendance` and `/team/roster` are unchanged |
| A10.11 | Roster editing / attendance correction by managers | ⬜ by decision — see the note above |

---

# PART B — MOBILE APP (Android + iOS, Employee-facing)

> Every screen below consumes the shared Laravel API (Part C), which is complete.
> The app lives in `mobile/` — Flutter, tested with 176 unit and widget tests
> plus 10 integration tests driven against a live server.

## B1. Onboarding & Auth
| # | Feature | Status |
|---|---|---|
| B1.1 | Splash + branded onboarding screens | ✅ the splash holds while the token is verified, and a four-card introduction sits in front of the **first** sign-in on a handset. Each card answers a question somebody actually asks on their first day — is the recorded time mine or theirs, what happens on a site with no signal, where do I book a day off, will it tell me anything — rather than saying "Welcome!" over a stock illustration, which trains people to skip past the one useful card. **Skipping settles it as firmly as finishing does**: skipping is a decision about the app, not a request to be asked again next launch. The flag describes the *handset*, so unlike the punch queue, the offline cache and the biometric preference it is **not** cleared with the token — somebody signing back in at the end of a shift on a shared phone is not introduced to the app again. It is also never shown to a session that restored, whatever the keystore says. Every card scrolls and nothing has a fixed height, so it passes the 2× font-scaling check with the rest (B6.4) |
| B1.2 | Login with email + password (Sanctum token) | ✅ |
| B1.3 | Biometric unlock (fingerprint / Face ID) | ✅ off until somebody switches it on in Profile, and the switch appears only on a handset with a biometric actually **enrolled** — on a phone with nothing but a PIN it would prompt for that PIN and call it a fingerprint. **Turning it on requires a check that passes first**, because saving the preference and then discovering the sensor refuses everybody locks its owner out of their own app, and the only way back is a reinstall — which also discards any punch still queued for a signal. For the same reason the lock screen always offers **Sign out instead**: fingerprints get removed and face data gets reset between one launch and the next, and a handset that can no longer say yes must not be the only way in. It engages before the splash lifts, not after the home screen has been drawn, and again when the app has been away for longer than a minute — long enough that the OS's own dialogs, the biometric sheet included, do not lock the app behind themselves. The device passcode is allowed as a fallback: it is already what protects the keystore the token lives in. The preference is on the handset, not the account, and is cleared with the token |
| B1.4 | Forgot password flow | ✅ app requests the link; the link opens the web reset page, not a token screen in the app |
| B1.5 | Stay-logged-in / secure token refresh | ✅ token in the device keystore, re-verified against `/auth/me` at launch |
| B1.6 | Device registration & binding (one account ↔ trusted device) | ✅ **Closes the oldest gap in attendance — lending a colleague your password so they can clock you in.** Neither of the other controls touches it: the geofence is satisfied, because the colleague really is at the office, and the B2.7 flags are silent, because nothing is being spoofed. The only thing that separates the two is which phone signed in. Off by default like every policy that can refuse somebody, and this one can refuse a *sign-in*, which is the sharpest thing on the list. **Switching it on locks nobody out that day**: each account claims its own handset at its next sign-in and the control bites from the second — a rollout that turned the whole workforce away on Monday morning would be switched back off by Tuesday. A build that sends no device id is let through for the same reason; that is an older app, not an impostor, and it simply gets none of the protection. The check runs **after** the password, so somebody guessing passwords is never told that an account exists *and* is bound — two facts more than they had — and a refusal issues **no token**, which is the assertion guarding against a future refactor sliding the check below `createToken`. `device_id` is a **UUID the app generates once and keeps in the keystore, never a hardware identifier**: `ANDROID_ID` and `identifierForVendor` are persistent cross-app handles on a person, collected for an HR system with no need of one and dragging Play Store data-safety declarations behind them, and they would buy nothing — the threat is a *second* handset, which has a different id either way. **It deliberately survives sign-out**, unlike the token, queue, cache and lock, all of which belong to the person: clear it and a borrowed login could sign out, sign in and be trusted as a new phone, with nothing on screen looking wrong. A **Trusted phones** screen under `manage-employees` is what makes the policy safe to switch on — binding creates exactly one new way to be locked out (a lost, broken or wiped phone) and a control with no release is a control somebody disables permanently. Releasing stamps the row rather than deleting it, because who was trusted and when that stopped is precisely what somebody asks after a dispute. Own table, not a column on `push_devices`: a push token rotates on the OS's schedule, and a security control hanging off a rotating value fails open at a moment nobody chose. Mutation-checked in both halves — removing the refusal, and clearing the id at sign-out |
| B1.7 | Logout / remote session revoke | ✅ sign out, and sign out everywhere for a lost handset |

## B2. Attendance (app core)
| # | Feature | Status |
|---|---|---|
| B2.1 | Big one-tap Check In / Check Out button | ✅ double-tap reads as success, not as an error |
| B2.2 | Live status card (checked in at 09:02, hours so far) | ✅ ticks locally between refreshes |
| B2.3 | GPS capture at punch | ✅ `geolocator`, permission asked at the first punch; no fix, no permission or no signal sends the punch without coordinates |
| B2.4 | Offline punch queue → auto-sync when back online | ✅ A failed punch is kept on the handset at **the moment it was tapped** and delivered by `POST /attendance/sync` when there is something to deliver it over. **This is the one place the device clock is trusted**, by decision: stamping a queued punch on arrival would file a 09:00 check-in as 17:00 and hand payroll a wrong number. It is bounded — future times and anything over 48h are refused rather than clamped — and labelled `source: mobile_offline` with the delivery delay in `notes`. No connectivity library: those answer "is there an interface", which is not the question on hotel wifi behind a captive portal, so a punch is always attempted first and queued only when the attempt actually fails. Held in a JSON file, so it survives a force-quit; cleared on sign-out, since undelivered punches belong to whoever made them |
| B2.5 | Geofence-aware punch (warn or block outside office) | ✅ **The blocking half has existed since A4.16; what was missing was the warning.** Somebody walked to the car park, held the button through a GPS fix and a round trip, and was then told they were two kilometres away. Now `GET /attendance/today` carries the fence — office, centre, radius — and the Clock screen does two things with it. It **states the rule before anybody taps** ("Clock in within 100 m of Head Office"), which costs no fix, no permission and no battery, because a radius is policy rather than position. And on the tap, with a fix in hand, it **answers locally instead of spending a round trip**, naming the office and the gap, which is the only part a person can act on. **One definition of the rule, not two.** Whether a fence applies depends on the company policy, the employee's work mode and whether the office has coordinates at all — the app never works that out, it is told. `AttendanceService::geofenceFor` answers it once and both the enforcement and the description read it, because two copies would drift the first time an exemption changed and the app would confidently refuse a punch the server takes. Null means say nothing, and null covers four different cases the client must not try to distinguish — including a build talking to an older server. **Same haversine, same radius, same exemptions**: a more accurate distance that disagreed with the enforcement would be the worse choice, and the local check deliberately stays quiet when the punch carries no coordinates, because the server exempts those and inventing a stricter rule would lock out anybody whose phone cannot see the sky. A home or hybrid worker sees nothing at all — telling them every morning how far they are from an office they were told not to attend is the same bug the enforcement already avoids, with a wider audience. Mutation-checked: removing the work-mode exemption fails three tests, one of them the app-facing one |
| B2.6 | Break in / break out | ✅ `POST /attendance/break` wraps the same `recordBreak` the web portal's button has called since A4.15; the Clock screen gains a Start/End break button, shown only on the clock, and the status card reads "On a break" as a third state rather than a fourth word for clocked out. `can_break` defaults to **false** when absent, so a build talking to an older server shows no button instead of one that 404s. Punch rows now label all four types — a ternary on `isIn` rendered `break_start` as "Checked out", the same mistake the server made. (The row read "needs A4.15 first" long after A4.15 shipped). **The button now says what pressing it costs** (A5.7): one line under it — *"A 30-minute break comes off your hours"*, or *"Your 30-minute break is paid — it stays on the clock"*, or, where the shift treats its break as a floor, *"30 minutes comes off your hours, even if you take less"*. The policy had been computed correctly on the server since it shipped and told nobody, so the one screen with a break button could not answer the only question somebody has before pressing it. The line states the **consequence, not the setting** — "unpaid" is a payroll word; "comes off your hours" is what somebody standing in a corridor deciding about lunch needs — and the minimum case is worded separately because under it cutting a break short buys nothing, which changes what people do. A shift with no break configured says nothing at all, rather than "0 minutes, unpaid" |
| B2.7 | Mock-location / rooted-device detection | ✅ **Recorded, never enforced** — the same rule this system already applies to location, and the reason it is safe to ship: office, remote and hybrid staff punch from wherever they are, and a false positive that stops somebody being paid is a worse failure than a true positive nobody acted on for a day. Three signals travel with every punch, break and queued sync: `location_mocked` from **the operating system's own report of that fix** (`Position.isMocked` — Android since API 18, iOS 15 for a simulated position, so it is not a guess the app is making), plus `device_rooted` and `device_emulator` from `safe_device`. Stored on `attendance_logs`, flagged in the register beside the map pin — because a mocked fix is a map link that means nothing — and findable through a **Flagged** filter, which is the whole point: a flag nobody can search for is a flag nobody reads. **All three columns are nullable, and null is not false.** Silence is what a web-portal punch, a kiosk punch and every build older than this feature carry; treating it as a denial would fill the filter with the company's entire history on its first use, and treating it as clean would write a bill of health nobody issued. Written **at creation only** — `attendance_logs` refuses to be edited, which is exactly right for a claim a handset made at one moment. Per punch rather than per device or per sync: a queue can hold one fix taken with a spoofer running and the next taken without, and the queued flags are the ones captured **at the tap**, not re-read when the queue finally drains. Nothing may cost a punch — a check that throws, hangs or has no platform channel resolves to unknown and the clock-in goes anyway. **Honest limits**: a root check runs on the device it is judging, so the root it finds can patch it out, and a custom ROM trips it with no fraud anywhere near — which is why the mock-location flag, the one the OS vouches for, is the strongest of the three and why none of them is treated as proof |
| B2.8 | Home-screen widget / quick action for fast punching | ✅ **A quick action, not a widget, and the choice is the feature.** Long-press the KEMP icon and the launcher offers one item — *Check in* or *Check out* — and tapping it opens the app on the Clock tab and makes the punch. Two presses from a locked home screen. **One item, never two**: the server decides the direction of a punch from the ones before it, so a menu offering both would let somebody pick the one that is not true and *still succeed* — the tap would work and the label would have lied about what it did. So the shortcut is republished from the same `next_action` the big button reads, every time the day settles, and the two cannot disagree. **The tap is held, not dropped, while the day loads.** Launching *from* the shortcut is the whole point, so the tap always lands while the first `/attendance/today` is still in flight; a handler that cleared it there would leave the feature working only when the app was already open, which is when nobody needs it. It waits, `_load` asks again when it settles, and it is spent either way once it has — a tap that survived into the next refresh would punch somebody in twice, minutes apart, for one press they had forgotten. **It goes through the same `_punch()` as the button**, so the geofence warning (B2.5), the GPS fix, the mock-location flags (B2.7), the duplicate cooldown and the offline queue (B2.4) all apply without a second copy of any of them — `_canPunchNow` is now the one rule the button and the shortcut both read. **The title is translated and the type is not** (trap 18): the OS keeps the `type` string on the launcher *between runs* and hands it straight back, so this is the only string in the app that outlives the process that wrote it — a Spanish handset publishes `Fichar entrada` over `clock_in`. **It comes off the launcher on sign-out**, in `_clearToken` beside the queue and the cache, because on a shared work handset a *Check out* left behind by the last person is one tap from clocking out the next one, and it sits there whether or not the app is running. No icon ships with it: a shortcut icon is a native drawable and an xcasset, neither of which a Dart test can see, and the launcher falls back to the KEMP mark, which is already the right picture. 18 tests, and the two guards that carry it were mutation-checked — removing the wait-while-loading makes exactly the cold-launch test fail, and removing the cooldown check makes exactly the cooldown test fail. `quick_actions` (flutter.dev) adds **no permission** to the merged manifest, checked per trap 32 |

## B3. Employee Self-Service
| # | Feature | Status |
|---|---|---|
| B3.1 | View own profile & employment details | ✅ |
| B3.2 | Edit permitted fields (phone, address, emergency contact) | ✅ **two sheets, because they are two records.** Name and phone are the *account*, through `PUT /profile`, which had existed unused since the API shipped. Home address, city, country, personal email and the emergency contact are the *employee record*, through `PUT /profile/details` — new, because the fields had no endpoint at all and somebody could not correct their own emergency contact from a phone, which is the one field in an HR record that is only ever read on the worst day. **The sign-in address is shown but not editable, deliberately** — changing it is account takeover in two steps (set it to your own, then "forgot password"), and it would need nothing but an unlocked phone; changing the *password* already demands the current one for that reason. `personal_email` is editable because it is plainly "how to reach me" and is read by nothing in authentication. Date of birth, national id, blood group, hire date, department, manager, shift and status are **not** editable and are not in the payload: those are HR's to set, and an app that let people change them would be a hole in the personnel record. **Only the fields sent are written** — an omitted key is left alone, so a client that knows about six of the seven cannot wipe the seventh by never having heard of it — and an **empty string clears** one, because "this is no longer the right person to call" has to be sayable. Values are trimmed, and a field holding only spaces is stored as null rather than as an empty field wearing a disguise. **Nothing here is cached**: the offline copy of `/auth/me` already carries the PII the profile screen shows, but a home address and an emergency contact on a handset that may be shared or lost is a different order of disclosure, and none of it is any use without signal — so the sheet reads live and leaves nothing behind. The contact has no read-only view of its own for the same reason: opening the sheet **is** how it is read, and the form says who can see these fields before the first one rather than after the last |
| B3.3 | Change password | ✅ |
| B3.4 | Attendance history with monthly calendar view | ✅ **two shapes of the same rows, because they answer two questions.** The list answers *what happened recently* over 7, 30 or 92 days; the grid answers *which days did I miss*, which is month-shaped, and is the one somebody opens before disputing a day. The toggle sits in the app bar and the range menu disappears with it — in the grid a month **is** the range, and two controls arguing over one window is a bug waiting to be filed. **Every date still comes off the server.** The month is counted from the day the server named, not from the handset, and the grid is the only caller in the app that sends `to` — which is the same rule rather than an exception to it, since both ends are days of a month the server handed over. That `to` also uncovered a trap the list never could: the endpoint echoes the window it used, so paging back to March replies `to: 2025-03-31` — true, and *not today* — and taking it as today killed the forward arrow and then handed the list a window ending three weeks early. The anchor is now read only from a reply to a request that named no `to`, with two independent tests on it, both mutation-checked. **Colour is never the only carrier**: each cell has a glyph as well as a tint (✓ present, ⏱ late, 🏖 leave, ✕ absent, ⚑ holiday) and a legend under the grid, because this screen is read by whoever cannot separate the green from the red as well. A day the server sent no row for is drawn blank and reads *No record* — in the month that is running that is every day still to come, and marking tomorrow as an absence is a charge nobody can answer. The forward arrow stops at the server's own month for the same reason. Tapping a day spells it out underneath using **the same row the list draws**, so there is one rendering of a day rather than two to keep in agreement, and offline the whole grid refuses honestly rather than guessing a month from the phone's clock |
| B3.5 | Personal attendance score / on-time streak | ✅ a score card above the totals on the History tab, and **two numbers with deliberately different shapes**. The score answers one sentence — *of the days you were meant to be here, how many did you make on time?* — and is computed **from the very rows printed under it** rather than from a second pass over the calendar, because a number that disagreed with the list it sits on would be worse than no number. Approved leave, holidays, weekends and rostered days off leave the denominator: a booked day off is not a day you failed to attend. **Null, never zero, when nobody was expected in** — a fortnight of leave has no score, and showing 0 would read as a failure to somebody on their first day back. The streak **ignores the range on screen**, because "eleven days" has to mean eleven days and not eleven of the last thirty, and **today is only ever counted, never held against them**: asking at nine in the morning before anybody has clocked in returns yesterday's streak, where treating an unfinished day as an absence would show every employee a zero every morning. `attendance_scores.score` was a copy of `ontime_pct` under a different name and read by nothing; it now means what the app shows, through the same function |
| B3.6 | View assigned shift & upcoming roster | ✅ published roster only |
| B3.7 | Download own payslip / documents | ✅ `GET /documents` lists the caller's own shelf (expiring first, same four `expiry_state` values as the web badge) and `GET /documents/{id}` streams the file; **My documents** opens from the Profile tab, downloads through the token and hands the file to whatever the phone opens that type with. Saved to the *temporary* directory, not Documents — these are passport scans, and leaving copies where every other app can browse them would undo the point of an authenticated endpoint. `notes` and the uploader are withheld server-side. **No payslip** — there is no payroll module, only the hours export (A7.14) |
| B3.8 | Company directory (colleagues, departments) | ✅ **Colleagues** opens from the Profile tab — searchable, debounced so a five-letter name is one request rather than five. Active staff of the caller's own company only. Contact details sit behind the `directory_show_contact_details` policy, **off by default**: there is one phone column on an employee record, and where staff have no desk line it holds a personal mobile. Call and email buttons are drawn only when the company shares details **and** that person has some — the app reads `shows_contact_details` rather than inferring from a missing phone, so it never draws a button that cannot do anything. Everything HR-grade — DOB, address, national ID, personal email, emergency contact, the reporting line — is never returned at any setting |
| B3.9 | Raise an attendance correction | ✅ **Corrections** opens from History — next to the record it disputes, since asking for one is rare and makes sense nowhere else. Pick a recent punch to dispute or leave it on "one is missing"; the server tells the two apart from whether `attendance_log_id` is present, so the form never sends a mode that could disagree with itself. Break punches are filtered out of the picker — only `in` and `out` are correctable. The date picker will not go past today, because the server refuses a future correction and offering one would be a trap. Raising and withdrawing only: no decide button for anybody, manager included. Added as a row because Part B only ever tracked the *manager* half (B7.2), which left the employee half invisible |

## B4. Leave (app)
| # | Feature | Status |
|---|---|---|
| B4.1 | Apply for leave (type, dates, reason, attachment) | ✅ **`leave_requests.attachment` had existed since the table was created and nothing had ever written to it** — there was no way to attach a sick note from the app *or* the web, which is what left this amber. Now: optional on both, `pdf jpg jpeg png webp doc docx` up to 10 MB, the same ceiling the document vault uses. The app posts the whole request as **multipart** whether or not a file is attached, rather than keeping two code paths that have to stay in step. **The file is stored before the request is submitted and deleted again if the submission is refused** — the other order leaves a medical document on disk belonging to a request that was never created, and submissions are refused often (overlapping dates, an exhausted balance are both ordinary). `LeaveService::submitWithAttachment` owns that rollback so it is not implemented twice. **The uploaded name never reaches the filesystem**: the path is hashed, and `attachment_name` is display text the download is *named* from — a path built from user-supplied text is a traversal waiting to happen, and two people attaching `scan.pdf` on the same day must not collide. **Who can read one is deliberately narrow** — the employee and their own line manager on the API, HR and administrators on the web through `manage-leave`. Not a colleague and not another team's manager: holding `approve-leave` gets a manager through the door and grants nothing on its own, so a manager who cannot decide on the request has no business reading the doctor's note attached to it. Three routes, because the three audiences sit behind three different gates and one route serving all of them would mean one route with three authorisation rules. `has_attachment` is computed **from the disk** rather than from the column, so a request whose file has been lost reports no attachment while still carrying its name — a paperclip pointing at nothing is worse than no paperclip. The file is deleted with the row, in the model rather than a controller, because leave can go through a cascade. Verified end to end on a handset: attached, uploaded, listed on the employee's own card, opened by the manager from their approvals inbox |
| B4.2 | View leave balances | ✅ |
| B4.3 | Track request status & history | ✅ |
| B4.4 | Cancel / withdraw a pending request | ✅ |
| B4.5 | Manager approval inbox (approve/reject in-app) | ✅ tab appears only with `approve-leave` |
| B4.6 | Team leave calendar | ✅ **an "Off" tab in the manager's Team area, and `GET /team/leave-calendar` behind it** — the month grid the web has had since A6.7, for the manager who is holding a phone rather than sitting at the desk. **It is in the manager's tab and not on the employee's Leave screen, deliberately**: the directory is explicit that another person's leave is not a colleague-grade fact, whereas a manager already reads every one of these in the approval inbox and `clashesFor` already names who else is off over the dates of the request in front of them — so this is that same disclosure arranged by day rather than by request, and nothing new is revealed to anybody. **Pending is drawn alongside approved**, which is the point rather than a detail: a month showing only what is already granted is a month a manager can approve a second person onto, and each entry carries its own status so the two never look alike. Date-major, the opposite of the roster tab beside it — that one is read down a person to see their week; this answers "can I let a second person go that week", which is a question about a day. Every day of the month is sent, weekends and holidays included, because a client drawing a grid should not have to reconstruct the holes. A stretch is expanded into every day it covers and clipped to the month, but `start_date` and `end_date` still name the real request, so a fortnight that began in July reads correctly on 1 August. Direct reports only and the manager's own leave is not on it — the team is exactly what `ManagerScope` says it is, the same definition `/team/attendance` and `/team/roster` use. **The month is never built from `DateTime.now()`**: the first load asks for no month at all and reads the answer out of the reply, because on the 1st or the 31st the handset and the company disagree about which month it is. Unlike the attendance board, a future month is not refused — leave is booked ahead, so next month is the most useful month it answers for |

## B5. Push Notifications (FCM / APNs)

> **Both halves are now built.** The app asks for permission at sign-in,
> registers with `POST /devices`, re-registers when the OS reissues the token,
> creates the `hrms_default` channel Android drops notifications without, sends
> the token back on sign-out, and routes a tap — including one that launched the
> app from cold — to the tab named by the payload's `route` key.
>
> It is silent until somebody creates the Firebase project: no
> `google-services.json` means no push, and the app builds and runs exactly as
> before rather than failing. That console work, and the one Xcode capability
> iOS needs, are the whole of what is left for B5.2 and B5.3 —
> see `Push-Notifications_Setup.md`.
>
> **`route` is a contract between two lists, and it drifted once.** The server
> sent `clock`, `leave`, `approvals` *and* `schedule`; `PushRoute` in the app
> parsed the first three, so every roster notification arrived and then landed
> nowhere in particular. An unknown key opens the app normally rather than
> crashing it, which is why the gap was silent for months — see B5.4.
>
> Both halves are now down to one list each. On the server every route comes
> from `App\Support\AppRoute`, which the push payload and the notification
> history (B5.6) both read; in the app every route — pushed or listed — is
> parsed by the same `PushRoute` enum. Adding a route still means adding its
> enum row, but there is now exactly one place on each side to look.

| # | Feature | Status |
|---|---|---|
| B5.1 | Clock-in reminder at shift start | ✅ **the mirror of B5.2, and not one line of the app changed** — `attendance.*` already routed to the Clock tab and the notification centre publishes whatever the server writes. `attendance:remind-checkin` sends inside a window that *closes*: `[start − lead, start)`, never after, because a reminder to clock in that lands once the shift has begun is an accusation and the late-arrival digest (A9.3) already covers that ground. **The five-minute scheduler interval is load-bearing** — with a quarter-hourly one a ten-minute window falls between two runs and nobody is ever reminded, silently, so `PolicyController` refuses a lead between 1 and 4 minutes. Almost all the logic is reasons *not* to send: a weekend, a company holiday, approved leave, a rostered day off, somebody already clocked in, somebody already told. **The only notification here with no mail leg**, deliberately: it is useful for ten minutes and an apology afterwards, and it is the only one that arrives *before* the working day on a personal phone — which is why the lead can be set to 0 and mean off |
| B5.2 | Clock-out reminder at shift end | ✅ end to end; needs credentials to leave the box |
| B5.3 | Leave approved / rejected | ✅ end to end, both stages of the approval chain |
| B5.4 | Schedule / roster updated | ✅ end to end. A9.5 had been sending `route: schedule` over FCM since it shipped; the app's `PushRoute` enum knew only `clock`, `leave` and `approvals`, so every roster notification arrived and then landed nowhere in particular. The enum row is added and the test now asserts one route per notification class on the server — the old one checked only the routes the app already knew, which is why it could never have caught this |
| B5.5 | HR announcements & broadcasts | ✅ **a composer and a register on the web, and one icon's worth of change on the phone** — an announcement is delivered as an ordinary notification, so it lands in the bell, the notification centre and FCM through the same path everything else uses. HR writes it, picks everyone / one department / one office, and sends. **Writing is a two-step gesture on purpose**: save saves a draft, and a second, separate button sends — one button beside a textarea is one slip from a company-wide message with half a sentence in it. **A published announcement can never be edited or deleted**, enforced on the model and not only in the controller: sending has already written a row into every recipient's history and woken every registered handset, and a register that disagrees with what two hundred people read is worse than a typo you cannot fix. `recipients_count` is frozen at send time for the same reason — staff join and move, and asking the audience again next month would report a number that was never true. **The text is not translated**: it goes out as typed, like leave type names and office names, because HR knows who reads what. No mail leg while `MAIL_MAILER` is still `log` (A9.2) — the first act of a new server would be two hundred identical messages from an IP with no sending reputation. The push carries the first 180 characters and the bell keeps the whole thing |
| B5.6 | In-app notification centre | ✅ a bell on the Clock tab with an unread count, and the history behind it. Everything shown has been in the server's `notifications` table since A9 — this is `GET /notifications` and the app catching up with the web dashboard, not a new store. Until now a push that arrived while the phone was in a locker was simply gone: the OS banner is swiped away and the app kept nothing. **Reading and going somewhere are separate gestures here**, unlike the web screen where a click does both — on a phone the list *is* the destination for most of these, because the body is the whole message. Only the four keys every notification class agrees on are published, plus a route, so a new notification type on the server needs no app change to appear. **That route now comes from one mapping** (`App\Support\AppRoute`) used by both the push payload and the history: they were written out separately before, which is how `schedule` came to be sent for months to an app whose enum had never heard of it. A notification with nowhere to go — a document-expiry warning is addressed to HR, who work at a desk — simply offers no button rather than inventing a screen to point at. Marking one read is optimistic and a read for an id that is no longer there is not an error, because the app may be delivering a tap made offline |

## B6. App Experience
| # | Feature | Status |
|---|---|---|
| B6.1 | Dark mode | ✅ light and dark themes, follows the system |
| B6.2 | Multi-language support | ✅ **English and Spanish, and the app follows the phone unless somebody says otherwise.** Flutter's own `gen_l10n` — no third-party package, which matters here because the privacy policy, the Apple manifest and both store data forms all say this app contacts exactly one host and carries no SDK that phones home. Roughly 360 strings, every one of them in `lib/l10n/*.arb`; `flutter analyze` cannot see a missing translation, so `test/locale_test.dart` reads gen_l10n's own untranslated report and fails when it is not empty, and separately catches a row left as the English text pasted across. **Dates are a message, not a concatenation** — Spanish writes "4 de agosto de 2026" — and month names come from the ARB rather than from `intl`'s `DateFormat`, which needs a locale-data load that throws at the moment a date is drawn if it was ever forgotten. **The language is a handset setting and survives sign-out**, unlike the token, the punch queue, the cache and the biometric preference: clearing it would put the login form back into a language the person standing there cannot read, on the one screen they cannot get past to fix it. Two identity bugs came out of this — `HomeShell` matched a tapped notification's route against the *label under the icon*, and `PushRoute.tabLabel` supplied it, so every notification would have opened nothing at all on a Spanish handset; both now match on a stable id. `Accept-Language` travels with every request, and the API answers in the same language now (C1.18) |
| B6.3 | Offline-first cache of profile, history, roster | ✅ the last good answer from `/auth/me`, `/schedule`, `/attendance/history` and `/attendance/today` is kept, and served when — and **only** when — the request never arrived. A refusal is an answer: a 403 for an account that lost its employee record is shown, not papered over with yesterday's success. Every saved copy is labelled on screen with when it was taken, because a roster that is quietly three days old is worse than no roster. Today's clock screen additionally expires on its own — yesterday's copy is refused rather than telling somebody they are already at work — and the screen falls back to a punch button that queues, which is what makes B2.4 reachable at all: before this, the first refresh with no signal replaced that button with a "try again". The cache is cleared with the token on sign-out, and it never holds one. Nothing that takes a decision is cached — leave balances and approval inboxes would talk somebody into booking days they no longer have |
| B6.4 | Accessibility (font scaling, contrast) | ✅ audited against WCAG 2.1 AA, and the audit is `mobile/test/accessibility_test.dart` rather than a document — it measures, so it cannot go stale. **Contrast:** the status palette was chosen against a white card and reused unchanged in dark mode, where `present`, `absent` and `leave` landed between 2.8:1 and 3.4:1 — under AA for the 11–13px text they are mostly used for, and under everything on the raised surfaces. There is no single value that satisfies both themes (readable on white needs a luminance below ~0.17, readable on #1E262E needs one above ~0.26), so there are now two palettes resolved by brightness, and every colour clears 4.5:1 on the worst surface it meets *including its own 10–15% tint*, which is the background these are usually drawn on. Separately, `primary` was the bright brand orange with white on it: 3.15:1, failing on every 16px button in the app — it is now the deeper orange in light mode and near-black-on-orange in dark, and the bright orange is reserved for the 21px Check-in label, the splash mark and the focus ring, where 3:1 is the bar. **Font scaling:** nothing clamps the OS text size and nothing should, but the punch button and the break button were laid out in fixed-height boxes and clipped at the larger settings — on the one control the app exists for — and two rows on the clock screen overflowed sideways. Five screens now render at 2× in both themes as a test; Flutter reports an overflow as an exception, so that is a real check rather than a screenshot somebody has to look at. **Screen readers:** every icon-only control has a name; one was missing a tooltip and announced only "button" |
| B6.5 | Crash & analytics reporting | ✅ crash reporting. **Analytics is deliberately absent and should stay absent** — there is no advertising SDK and no analytics SDK, the app contacts exactly one host, and the privacy policy, the Apple privacy manifest and both store data forms all say so. **Crashes go to the employer's own server, not to Crashlytics or any third party**, for that same reason: a stack trace routinely carries fragments of whatever the app was holding, and shipping those to Google would falsify all three documents at once. A crash is written to the handset at the moment it happens — a reporter that posts from inside a dying process loses precisely the crash that killed it — and delivered on the next launch by `POST /app/crashes`, after the session restore so a signed-in handset's report says whose it was. The endpoint is unauthenticated on purpose: the crash worth having is the one that stops the app opening, and an authenticated one would collect every crash except that one. Reports are grouped server-side by a fingerprint of the exception plus the top few frames, so a hundred handsets on one bug read as one row, and are visible under **Administration → App Crash Reports** to whoever holds `manage-settings`. Nothing in the reporter is allowed to throw: an error handler that fails turns one crash into a loop |
| B6.6 | Force-update / maintenance-mode gate | ✅ `GET /app/status`, asked at launch and again after a spell in the background. **The server decides and the app obeys** — the comparison lives on the server because the app is the half that cannot be fixed: a handset with a broken comparator has already shipped, and the answer it is given is the only thing left that can change its behaviour. **It fails open at every level**: an unreachable server, an unreadable answer, a verdict invented after the build shipped, a missing version, a platform with no store link — all of them carry on. That is the point rather than caution, because the app is deliberately usable with no signal, and a gate that blocked whenever it could not reach the server would take the offline cache and the punch queue away in exactly the conditions they were built for. Maintenance is a flag of its own rather than `php artisan down`, which returns 503 to everything and is indistinguishable from an outage — the app would fall back to its cache and queue punches into a server being migrated underneath it. Both settings are config, not database, because the moment they matter most is the moment the database is unavailable, and both are empty/off by default; `emp:preflight` fails a deploy that leaves maintenance on, or sets a minimum version with no store link to send anybody to |

## B7. Manager Mode (optional in-app role)
| # | Feature | Status |
|---|---|---|
| B7.1 | Team attendance, today **and any past day** | ✅ present vs in-now reported separately, and the board now carries the day it is answering for. `GET /team/attendance` had taken a `date` since it shipped and the app only ever asked for today, so a manager holding a handset could not answer *"was she in yesterday?"* — the question that actually gets asked when somebody is missing this morning — while the web manager area had the same board with a date on it all along (A10.4). The forward arrow is **disabled on today rather than left to fail**: the endpoint refuses a future date, and a control that reliably produces an error is a trap, which is the same reason the corrections picker stops there. Today is sent explicitly rather than left to the server's default, so a handset left open across midnight cannot refresh into a day its own header does not name |
| B7.2 | Approve leave / regularisation from phone | ✅ **leave only, and settled** — a correction is HR's alone, not a manager's. Leave approval is manager-then-HR; regularisation has no manager step, so an approve button in the manager tab would advertise a stage that does not exist. The employee half is `POST /attendance/regularisations` (A4.13 on the API); deciding stays on the web behind `manage-attendance` |
| B7.3 | Team roster view | ✅ `GET /team/roster` plus a Roster tab in the app — a week per direct report, published days only, with leave outranking a rostered shift. **The week is no longer counted from the handset's clock**, which was the quietest instance of a mistake this codebase has now made four times: unlike the attendance board's `date`, this endpoint accepts whatever `from` it is handed rather than refusing a wrong one, so a phone a day ahead of the office was shown Tuesday-to-Monday under a heading that said "This week" and nothing anywhere said so. This week now sends no `from` at all and learns the company's today from the reply's echo; every other week counts from that anchor, and because this week always re-asks, a session left open across midnight corrects itself instead of drifting a day further out with each press |

*Built so far: sign-in with the token held in the device keystore, the clock screen
with its live worked-hours card, attendance history with totals, the published
roster, profile and password, the full leave round-trip — balances, apply, track,
withdraw — and the manager's tab: an approval inbox and who on the team is in today.
The manager tab is drawn from the signed-in user's permissions, not hardcoded.*

*GPS now travels with a punch. It is a record and never a gate: the app asks for
"while in use" at the first punch, and services off, a refusal, a sensor that
returns nonsense or no fix inside eight seconds all send the punch without
coordinates rather than failing it. Nothing runs in the background, which is what
the privacy policy already promised.*

*Forgotten passwords are the app's own now rather than a note telling people to
ask HR — which never helped the one account HR cannot reset, the administrator's.
The app asks for the link; the link opens the web page, because a reset has to
work from a borrowed laptop when the handset is the thing you are locked out of.*

*Push now works on the handset. Registration is a consequence of signing in and
is withdrawn on signing out, which is what stops a shared work phone showing the
previous person's leave decisions; a tap opens the tab the notification is about
rather than just the app. It stays silent until a Firebase project exists, and
the app is entirely usable in that state.*

*The app opens with no signal at all. The last verified `/auth/me` is kept, so a
handset that cannot reach the server restores the session from it rather than
landing on a login screen that also needs the network — which is what had made
the offline punch queue nearly unreachable in the case it exists for. That trust
is bounded to a week: roles, permissions and whether somebody still works here
are re-read on every launch that does reach the server, and past the grace the
app asks for a sign-in instead of going on trusting what it last knew.*

*And the handset can be held behind its own fingerprint or face check (B1.3),
off by default, cleared with the token, and never the only way in.*

---

# PART C — SHARED BACKEND & API

| # | Feature | Status |
|---|---|---|
| C1.1 | **Laravel Sanctum token auth + `routes/api.php`** | ✅ |
| C1.2 | `/auth/login`, `/auth/logout`, `/auth/me` (+ `logout-all`, `devices`) | ✅ |
| C1.3 | `/attendance/check`, `/attendance/break`, `/attendance/sync`, `/attendance/history`, `/attendance/today`, `/attendance/regularisations` | ✅ same AttendanceService as the web button — one set of punch rules. `today` reads clocked-in state from `breakState`, not from the last punch: `break_end` is neither `in` nor `out`, so the old reading would have offered "Check In" to somebody who never left |
| C1.4 | `/leave/*` endpoints | ✅ balances, apply, list, withdraw + the manager inbox — all via LeaveService |
| C1.5 | `/schedule`, `/profile`, `/documents` endpoints | ✅ published roster only, leave/holiday/weekend aware; profile read + contact edit + password, plus `PUT /profile/details` for the employee record's own address and emergency contact (B3.2); own documents listed and streamed, read-only |
| C1.6 | Device token registration for push | ✅ register/list/unregister; cleared on sign-out. Delivery is Phase 5 |
| C1.7 | API rate limiting + throttling | ✅ per-user limiters — 120/min ceiling, login 5, punch 20, writes 30 |
| C1.8 | Consistent JSON error format + API versioning (`/api/v1`) | ✅ one shape for every failure, derived from the status rather than from which exception class happened to be thrown, so one condition never arrives under two names. **One hole was closed on 2026-09-15**: the renderer turned any `HttpResponseException` into a generic 500, discarding a response the caller had deliberately built to short-circuit with. Nothing in the codebase had needed one, so it went unnoticed until the login rate limiter's own `response()` callback did — and the effect would have been a 500 on every future use of that pattern. Such a response is now returned as raised, and the throttled 429 that goes through it is pinned to the standard shape by a test |
| C1.9 | Queue worker + scheduler (reminders, auto-absent, reports) | ✅ **installed and running** on `hrams.devonlinetestserver.com` — `emp:preflight` reports the scheduler last ran seconds ago and the queue empty with no failed jobs. Cron-driven rather than systemd (`HOSTING_MODE=managed`, `deploy/emp-webspace.cron`), because the host is managed webspace with no systemd; that setting also widens the preflight tolerance from five minutes to fifteen, which is normal for cron and alarming for a daemon. Queued notifications survive a deleted record and retry a bad send. Nothing they send leaves the box while `MAIL_MAILER=log` |
| C1.10 | Immutable audit log for attendance records | ✅ punches are append-only (edit/delete refused); every write records actor, source, IP and a full snapshot |
| C1.11 | Automated test suite (feature + unit) | ✅ 1366 server tests covering attendance, leave, roster, swaps, the API, the audit trail, password reset, push, backups, install, employee import, preflight and the manager role, plus 232 in the app — models, formatting, offline behaviour, the biometric lock, the gate, crash reporting, accessibility, the translations, and the four screens that must not build a date from the handset's clock |
| C1.12 | API documentation (Scribe / OpenAPI) | ✅ `API-Reference_v1.md`, kept honest by a test that walks the route table |
| C1.13 | Database backup & restore strategy | ✅ `db:backup --verify` nightly — dumps, restores into a scratch database to prove it reads back, then rotates |
| C1.14 | Production deployment (HTTPS, env hardening) | ✅ **live at `https://hrams.devonlinetestserver.com`** — managed webspace, cron-driven queue and scheduler (`HOSTING_MODE=managed`), TLS clean, `/api/v1/ping` answering `{"ok":true,"service":"KEMP"}`, and `/privacy` and `/account-deletion` both reachable logged out, which is what the two stores fetch. Updates ship with `ALLOW_NON_PRODUCTION=1 bash deploy/deploy.sh` — the flag because the box's `.env` says `staging` and the script refuses to guess which database to migrate. **`emp:preflight` is not green on it and should not be read as if it were**: the demo quick-login panel and the seeded `password` accounts are live on a public URL, `MAIL_MAILER` is still `log`, and `TRUSTED_PROXIES` is unset behind Varnish so every punch records the proxy's IP rather than the employee's. **Preflight now also gates on dependency advisories** — `composer audit` runs inside the command, a critical or high advisory fails the deploy and medium or low warns, so no release ships past a known hole without somebody deciding to. It never fails because it *could not* look: the advisory database is fetched over the network, and no composer, no network or a timeout all warn and say which rather than turning "I could not check" into "you may not deploy". This closes the dependency register's finding 13.6, which had asked for exactly this and sat open long enough for the `maatwebsite/excel` CVE to arrive unnoticed. **The escape hatch is no longer all-or-nothing, and the next deploy to this box will stop.** `ALLOW_NON_PRODUCTION=1` used to run preflight as `|| echo "(advisory)"`, which downgraded every check in the command — so the seeded `password` accounts, the unset `TRUSTED_PROXIES` and an empty `APP_KEY` all printed red and the deploy said "Done" anyway. It now passes `--non-production`, which downgrades only the failures that can be a deliberate choice on a demo box (mail to the log, the quick-login panel) and never the five that cannot: `APP_KEY`, `Database`, `Demo credentials`, `Dependency advisories`, `TRUSTED_PROXIES`. Two of those are currently failing on this box, so the next run will refuse until the `.env` is fixed — which is the point, and is the change to be ready for rather than surprised by. |
| C1.15 | Push delivery to handsets (FCM v1) | ✅ channel alongside database and mail; silent until a service-account key is configured; deletes handsets FCM reports UNREGISTERED, keeps ones that merely 503'd |
| C1.16 | Public privacy policy + account-deletion pages | ✅ no login required — both stores demand it before an app with accounts is listed |
| C1.17 | Real-install setup, no demo data | ✅ `emp:install` creates the company and first admin, or attaches an admin to an existing company (`--company-id`); validated timezone, roles seeded, one transaction. `db:seed` now makes roles only. `emp:purge-demo --dry-run` clears a seeded database and names the real rows the cascade would take with it |
| C1.18 | API messages in the caller's language | ✅ **English and Spanish, and not one line of the app changed** — `ApiErrorText.text` already showed the server's words as they arrived, which is what the header shipped in B6.2 was for. `SetApiLocale` reads `Accept-Language` on the API group only and **fails open at every level**: an unknown language, a malformed header, a `q=0`, none at all, all answer in English rather than refusing, because a preference is not a credential. Region is dropped — `es-MX` and `es-419` are `es`. Every message, every validation error (Laravel's own `validation.php`, translated whole and checked against the framework's so an upgrade that adds a rule fails the suite), the leave stage, the geofence refusal, and the meridiem on a pre-formatted punch time. **Notifications are the half a request cannot decide**, and the reason `users.locale` exists: a worker renders them with no request and no header, usually because of somebody else's action — HR approving leave in English decides what an employee reads in Spanish — so the language has to be a fact about the recipient, which `User::preferredLocale()` supplies to push, the notification centre and the email alike. Leave types, office names and the maintenance message stay as typed: they are data, not vocabulary |

---

# PART D — AI HR ASSISTANT *(later phase)*

| # | Feature | Status |
|---|---|---|
| D1.1 | Natural-language HR queries ("who was late last week?") | ⬜ |
| D1.2 | Attendance history search | ⬜ |
| D1.3 | Leave history search | ⬜ |
| D1.4 | Auto-generated attendance & department summaries | ⬜ |
| D1.5 | Context-aware follow-up questions | ⬜ |
| D1.6 | **Permission-aware answers** (never leaks data above the asker's role) | ⬜ |
| D1.7 | Available in both web dashboard and mobile app | ⬜ |

---

# Delivery Roadmap

| Stage | Contents | Status |
|---|---|---|
| **Stage 0** | A1–A5, A7, A8 (minus gaps) | ✅ Phase 1 web dashboard, live and verified |
| **Stage 1** | C1 — Sanctum API layer | ✅ Done — the hard blocker is gone |
| **Stage 2** | A6 — Leave Management (web) | ✅ Done — no accrual engine, no calendar view |
| **Stage 3** | A9 + C1.9 — Notifications + scheduler | ✅ Built — `MAIL_MAILER` is still `log`, so no mail leaves the box |
| **Stage 4** | A5.5–A5.9 — finish Shift & Schedule | ✅ Done — planner is a grid, not drag-and-drop |
| **Stage 5** | B1–B3 — Mobile app v1 (login, punch, self-service) | ✅ **Closed out.** The screens work, punches carry GPS, the app is usable offline, and the handset can be held behind its own biometric check. Every row this note used to list as open has since shipped — device binding (B1.6), the on-phone geofence warning (B2.5), mock-location detection (B2.7) and now the launcher quick action (B2.8) — so there is nothing left across B1–B3. The calendar grid and the personal score (B3.4, B3.5) shipped earlier, and so did the address and emergency-contact fields, which had no endpoint to write to until `PUT /profile/details` |
| **Stage 6** | B4–B5 — Leave + push in app | ✅ Leave and push both done. Push is silent until the Firebase project exists — console work, not code. **The one code exception is closed**: B5.4's `route: schedule` is now parsed by the app (`PushRoute.schedule`), so a roster notification lands on the roster rather than nowhere in particular |
| **Stage 7** | A4.12–A4.15, A7.10–A7.14 | ✅ Attendance depth + reporting — correction, regularisation, overtime, break punches, payroll export, leave reports, scheduled delivery, report builder |
| **Stage 8** | A1.7–A1.9, A2.3, A2.8, A4.16 | ✅ 2FA, the security trail, the idle timeout, the working-week editor and geofence enforcement |
| **Stage 9** | A3.7–A3.11, A6.4/A6.7/A6.9, A4.19, A9.3 | ✅ Photos, the document vault, emergency contacts, the org chart, the roster export, leave accrual and carry-forward, the leave calendar, the live board and the late-arrival digest |
| **Stage 10** | A3.12, A4.11, A8.4–A8.6, A9.5 | ✅ Dashboards per role and per person, week-on-week trends, weekly rollups, schedule alerts and on/offboarding checklists |
| **Stage 11** | B7 — Manager mode in the app | ✅ Team roster added; approvals and team attendance already shipped |
| **Stage 12** | A10 — Manager workspace on the web | ✅ The manager promoted from a tab on the employee portal to a first-class role with its own area, dashboard, team screens, scoped reports and approvals inbox. One scope service, no new tables, no mobile change |
| **Stage 13** | D1 — AI HR Assistant | ⬜ Out of scope for now, by decision. Needs mature data across attendance + leave |
| **Stage 14** | The API catches up with the web | ✅ `/attendance/break`, `/documents`, `/attendance/regularisations`, `/directory`. Four features the web had shipped and the API had never exposed — every one of them had a status note naming an internal dependency that had long since been met |
| **Stage 15** | The app catches up with the API | ✅ `PushRoute.schedule`, profile editing, the break button, My documents, Corrections and Colleagues. Every endpoint the API offers now has a screen behind it |
| **Stage 16** | Roles behave the same on both halves | ✅ The Team-tab gate now needs the permission *and* a team, so HR and report-less managers no longer get an empty area. HR is desk-only by decision — see below |
| **Stage 17** | Reliability — offline and biometrics | ✅ **B2.4 offline punch queue, B6.3 offline cache and B1.3 biometric unlock.** The app opens, reads and clocks with no signal, and a shared handset can be held behind its own fingerprint or face check without the phone ever becoming the only way in |
| **Stage 18** | Store readiness | ✅ **All six rows done** — B6.6 the force-update and maintenance gate, B6.5 crash reporting, B6.4 the accessibility audit, B5.6 the notification centre, B1.1 the onboarding carousel and B6.2 multi-language. Five of them carry a server half or a test suite behind them rather than a document that goes stale: a preflight check, an administrator's crash screen, a history endpoint, an accessibility file that measures contrast and pumps six screens at 2× text, and a locale file that reads gen_l10n's own report of what is still untranslated. The library decision for B6.2 was Flutter's own `gen_l10n` and nothing else, for the same reason there is no crash SDK: four documents say this app carries nothing that talks to a third party |
| **Stage 19** | The submission bundle itself | ✅ **Five defects that no test could see, because none of them is code that runs.** `PrivacyInfo.xcprivacy` was a file in a folder and not in the iOS target, so every build shipped without it and Apple auto-rejects on that alone. `ACCESS_FINE_LOCATION` implies a **required** GPS `<uses-feature>`, which had been quietly filtering the Play listing off every device without the hardware — an app nobody could find rather than one that failed. The 512×512 Play icon was a crop of one corner of the mark. There was no 1024×500 feature graphic, which Play requires on every listing. And the adaptive icon had no `<monochrome>` layer, so a themed Android 13+ home screen left this app the one orange tile in a recoloured grid. All five verified in a real `bundleRelease` and against the merged manifest. Also written down for the first time: the reviewer sign-in account both consoles demand for a login-only app, and the console forms — content rating, app access, EU trader status, Play's 12-tester closed test — that block a release while the code sits finished. The app is also now declared **iPhone-only** — `TARGETED_DEVICE_FAMILY = 1` in all three configurations and no `~ipad` orientation key — which drops the 13" iPad screenshot set and the reviewer opening it on hardware nobody has laid it out for. See `Store-Submission_Checklist.md` |
| **Stage 20** | KEMP brand mark | ✅ The client's icon set applied across both halves. The supplied artwork is a **pre-rounded plate on transparency**, which is the wrong shape for a launcher and illegal for Apple — an alpha channel on an App Store icon is a rejection — so two masters are derived from it rather than handing it over as-is: an opaque full-bleed square for iOS and legacy Android, and the mark keyed off its navy for the Android adaptive foreground, sitting on `#052C6E` so any navy missed at an anti-aliased edge is invisible against the layer beneath it. `flutter_launcher_icons` regenerates every density from those two, then **silently rewrites `ic_launcher.xml` and drops the `<monochrome>` block** — it has no themed-icon support — so that layer is restored and verified inside the built `.aab`, not just in the source tree. The three store images are cut from the same master by hand, since that command does not touch them. On the web the sidebar, header and collapsed marks and the favicon all move from the Klutch Cleaning company logo to the KEMP lockup, with a white-wordmark variant for the dark sidebar — recoloured from the divider rightwards only, because whitening the whole lockup fills the plate solid and swallows the K inside it. Eight hardcoded `alt="Klutch Cleaning"` strings now read `config('app.name')`. The app itself is renamed **KEMP** — the Android label, both iOS bundle-name keys and `appTitle` in both ARB files, where Spanish carries the identical string because a brand is not translated; `locale_test.dart` already exempts rows under 20 letters for exactly that reason, so the suite stays honest rather than being loosened for it. The bundle id `com.hrms.attendance` stays as it is: not user-visible, and unchangeable after a first upload. **The app's interior stays brand orange** by decision: that palette is what `accessibility_test.dart` measures at 4.5:1 on every surface in both themes, and rethemeing means redoing that audit, not swapping a constant |

### Roles on the phone (Stage 16)

The web separates the four roles properly. The app was written employee-first
and grew a manager tab, and the seams showed in three places. All three are now
closed — the last of them, the administrator empty state, on 2026-09-18.

| | What the app gives them | Right? |
|---|---|---|
| **Employee** | Clock, History, Leave, Schedule, Profile, My documents | ✅ |
| **Manager** | The above plus Team — approvals, team attendance, published roster, who is off this month | ✅ |
| **HR** | The employee screens only — **desk-only by decision**, see below | ✅ the behaviour was always right; **the demo data was not, until 2026-09-16.** `hr@emp.test` had a user and a role and no employee record, so on a handset it landed on the admin empty state on all four employee screens — this row said one thing and the seeded account did another, and nothing failed to say so. HR now carries EMP-0006, reporting to nobody. `tests/Feature/Api/DemoRoleAccessTest` pins all four roles |
| **Admin** | Signs in, then is told why on every employee screen — no employee record | ✅ **was ⚠️ "by design, but poorly explained", and the explaining is now done.** Each of the seven screens already carried its own sentence and dropped its retry button, which is the half that was right. The half that was not: `AsyncView` drew `Icons.cloud_off` above all of them, so an administrator opening the app was told the network was down, seven times, directly above a sentence saying it was not — the picture and the words disagreed and the picture is what gets read first. `AsyncView` now takes `permanent`, which owns the icon **and** the retry suppression, so the rule lives in one place instead of being restated as `onRetry: _fatal ? null : _load` at seven call sites. Underneath it, the API stopped answering this with the generic `forbidden`: that code also means "that leave request is not yours" and "that correction is not yours", so the app's classifier was relabelling two ordinary, recoverable refusals as a permanent account defect and stripping the retry that would have cleared them. `App\Exceptions\NoEmployeeRecord` gives it `no_employee_record` of its own, and `test/no_employee_record_test.dart` now pins all four claims — message, no retry, right icon, and `forbidden` **not** treated as this — across all seven screens rather than three |

**Fixed:** the Team tab hung off the `approve-leave` permission alone. HR holds
that permission — it is the second step of the approval chain — and almost never
has direct reports, while every endpoint behind the tab is scoped to direct
reports. So HR got a Team tab whose every screen was empty, permanently, with
nothing explaining why; so did any manager with nobody reporting to them. The
web never had this, because `/manager/*` is gated `role:manager` **as well** and
refuses HR at the door. The tab now needs the permission **and** `is_manager` —
a field `/auth/me` had always returned and the app had always parsed and never
read. "manager is a role *and* a relationship, and both must line up."

**Decided: HR is desk-only, and that is the intended behaviour, not a gap.** HR
signs in and gets the employee screens — they clock in and book their own leave
like anybody else — and nothing more. Everything HR actually does is employee
records, the leave register, reports and the document vault, none of which is a
phone job; the company-wide stage-two approval queue stays on the dashboard.

So there is deliberately **no HR approvals inbox in the app**, and a future
reader should not restore one thinking it was forgotten. If the client asks for
phone approvals later it is an additive change — a company-wide endpoint gated
on the `hr` **role** rather than the `approve-leave` permission, which managers
share — and not a rework of anything here.

### The thing that used to gate the rest, and what replaced it

**C1.14 is done.** The app is live at `https://hrams.devonlinetestserver.com`.
A handset could not resolve `127.0.0.1`, FCM would not call back a laptop, and
neither store accepts a privacy-policy URL pointing at localhost — all three are
now answered by a real domain with a clean certificate. Store submission and
real-device testing are unblocked.

**What gates the rest now is three settings on that box, not code.** Every one
of them is a line in `.env` and a `config:cache`:

1. **The demo quick-login panel is ON at a public URL, and `admin@hrms.test`
   still has the seeded password.** One click on the login page is full admin —
   employee records, the document vault with passport scans, every punch. This
   is the one to fix today; the rest can wait.
2. **`MAIL_MAILER=log`.** Password resets, leave decisions, scheduled reports
   and document-expiry warnings are all built, tested and queued, and all go
   nowhere.
3. **`TRUSTED_PROXIES` is unset behind Varnish**, so `attendance_logs` is
   recording the proxy's address on every punch. The audit trail (C1.10) and the
   IP column in exports (A7.9) are both quietly wrong. **It no longer goes
   unsaid**: `DetectUntrustedProxy` notices a forwarded request arriving while
   nothing is trusted, and `emp:preflight` turns that sighting into a FAIL that
   names the header, the address the proxy claimed and the one actually stored.
   The setting is still the fix, and it is still an `.env` line on the box — but
   a deploy now refuses rather than passing quietly.

Push (B5) additionally needs the Firebase project, which is console work rather
than a setting. See `Deployment-Guide_Production.md` and
`Store-Submission_Checklist.md`.

---

## Counts

| Area | Built | Partial | Planned | Total |
|---|---|---|---|---|
| Web Dashboard (A) | 100 | 4 | 2 | 106 |
| Mobile App (B) | 45 | 0 | 0 | 45 |
| Backend / API (C) | 18 | 0 | 0 | 18 |
| AI Assistant (D) | 0 | 0 | 7 | 7 |
| **Total** | **163** | **4** | **9** | **176** |

**The web dashboard is complete, AI excluded.** Stages 8 through 12 are all
delivered. Two planned rows and four partial ones remain across Part A, and none
of them blocks a production deployment. Part B (mobile) has no open row
left. The AI assistant (Part D) is deliberately out of scope.

**Still open, and worth being explicit about:** multi-company tenancy (A2.10), a
conditional rules engine (A2.9, A6.6), and roster editing and attendance
correction by managers (A10.11 — held with `manage-shifts` and
`manage-attendance` on purpose).

**The 2FA QR (A1.7) is done, and the note that said it was blocked was wrong.**
It had been recorded as waiting on composer, which supposedly could not resolve
a new dependency because of a `league/commonmark` advisory. Composer resolved
`bacon/bacon-qr-code` without complaint on the first attempt. A blocker nobody
re-tested outlived the thing blocking it — worth re-reading any note of that
shape against the tool rather than trusting it.

**The break policy (A5.7) is done**, but read the row rather than the tick:
what shipped is *what the shift's break means for paid time* — paid or unpaid,
and whether the shift's figure is a floor — not a builder of multiple named
break windows. That last was left out deliberately: nobody has asked for it,
nothing in the schema wants it, and it would be a feature invented rather than
needed.

**The team leave calendar (B4.6) is done**, and it was the fifth instance of
the same pattern: the web had shipped the feature (A6.7) and the API had never
exposed it. `GET /team/leave-calendar` and an "Off" tab in the manager's Team
area. It landed behind the manager's gate rather than on the employee's Leave
screen — see the row for why, which is a disclosure decision and not a
placement one.

**The web dashboard is English only, and deliberately so.** It is HR's and the
administrator's screen; the workforce that needed Spanish is the one holding the
phone. Every message the two halves share is a `__()` call now, so the strings
are already there — what a Blade pass would still cost is the templates
themselves, and nobody has asked for it.

**Two smaller things a Spanish reader still meets in English**, both by
decision: `/privacy` and `/account-deletion`, which are web pages the app links
out to; and the leave types, office names and designations a company typed in,
which are renamed rather than translated.

**Two things are code-complete but inert until configured**, and neither is a
code change: `MAIL_MAILER` is still `log`, so no email leaves the box; and push
stays silent until a Firebase project exists.

**Four app rows were marked blocked on web work that has since shipped**, and
the notes have been corrected in place — B2.6, B3.7, B5.4 and B7.2. In every
case the real gap turned out to be the same one: the web has the feature and the
API never exposed it. B5.4 was the exception — the server sent that push and the
app's `PushRoute` enum had never heard of it — and it is now closed too, so all
four read ✅. A row that names an *internal* dependency goes stale the day that
dependency ships, and nothing fails when it does, so re-read these against the
code rather than trusting the note. **This paragraph is its own example**: it
described B5.4 as still open for a while after the enum row landed.

---

*Updated 2026-09-14 from the live codebase — `hrms/` and `mobile/` both read directly
rather than from the previous edition of this file. Supersedes the stale build-status
section of `Phase-1_Admin-Dashboard_Attendance_SOW.md`.*

*Last verified 2026-09-17: `php artisan test` **1420 passed (3498 assertions)**,
`flutter analyze` clean, `flutter test` 290 passed. The nine new tests are the
page-size pair — see the note at the end of this file. The app count is
unchanged because no Dart was touched.*

*2026-09-15: 1411 passed (3472 assertions). B4.6, B3.2, A5.7, A5.8 and
the 2FA QR were finished across that stretch; the last two screens that still
built a date from the handset's clock — attendance history and the team roster —
were fixed; and **the security trail was found to be silent about the entire
mobile app and about every self-service password change on both halves**, which
is now closed. See A1.8.
The A5.7 migration was run against the real `emp` MySQL database as well as the
in-memory sqlite the suite uses, and A5.8's drag-and-drop was exercised in a
real browser against that database rather than only asserted in markup.
The same pass closed **B1.6 device binding, B2.5's geofence warning and B2.7
mock-location recording**, and stood up `CrossCompanyIsolationTest`.
**B2.8, the launcher quick action, went in after that** and took the last
buildable ⬜ off this list with it — everything still marked ⬜ is now parked by
decision rather than outstanding. Its 18 tests are why the app count is 290
rather than the 272 the line above used to read, and `flutter build apk --debug`
was run to check the merged manifest per trap 32: `quick_actions` adds no
permission.*

*Also 2026-09-15, and not a feature: **`TRUSTED_PROXIES` had never been read.**
It was configured in `bootstrap/app.php`, whose `withMiddleware` closure runs
before Laravel loads .env, so the guard around it was false on every box in
every environment since the file was written. Behind the proxy that means every
punch recorded the proxy's address rather than the employee's, the IP column in
exports was one value repeated, and the login rate limiter — which keys on the
address — turned A1.10's "one person's mistakes cannot lock out a colleague"
into the opposite. Moved to `config/trustedproxy.php`, where the framework's own
middleware reads it and where `env()` survives `config:cache`; `emp:preflight`
now reads the same key rather than `env()`, so it cannot report "unset" on a box
that had just set it. Five tests, and honestly labelled: four of them would have
passed against the broken build, and only the fifth covers the fix. See trap 36.*

*2026-09-16, and the same shape of problem in the demo data rather than the
code: **`hr@emp.test` had no employee record**, so two of the four roles could
not be shown on a handset — HR landed on the administrator's empty state on all
four employee screens, contradicting the row above and the brief written for the
UI designer, with nothing failing to say so. HR now carries EMP-0006 and reports
to nobody; admin still deliberately has none, because that refusal is a designed
screen. `tests/Feature/Api/DemoRoleAccessTest` pins what each of the four roles
gets on the phone, and removing the new record fails exactly the three HR tests.*

*`composer audit` is clean as of that date. Three advisories were open against
this project and are now closed: `maatwebsite/excel` 3.1.69 → 3.1.70 (high —
exports written outside the configured disk on a caller-controlled filename,
and this project was handing it one) and `league/commonmark` 2.8.3 → 2.10.1
(two medium). The unvalidated report window that fed that filename is fixed
in the application too — see the note under A7.14.*

**One mistake, four times, and worth naming so it is not made a fifth.** The
team board, attendance history, the team roster and now the leave calendar all
built a date from `DateTime.now()`. Attendance is judged in the company's
timezone and the phone is wherever its owner is, so for part of every day the
two disagree — and only one of the four endpoints (`/team/attendance`) refuses a
wrong date loudly. The others clamp it, or simply answer for the window they
were handed, so the screen shows the wrong days under a heading that reads
correctly. **The rule: the app never names a date the server has not named
first.** Ask for the default, read the day out of the reply, and count from
that. Every one of these tests passed against the broken code until the mock
server's clock was deliberately skewed from the test device's — a shared clock
makes the whole class of bug invisible.

*Verified 2026-09-12 and not re-run since: `flutter build appbundle` produces a
44.1 MB release bundle whose merged manifest reads minSdk 24 / targetSdk 36 and
whose compiled `ic_launcher.xml` still carries the `<monochrome>` layer. Stages
19 and 20 were found and done in that pass.*

*2026-09-17, from an audit for hardcoded values rather than from the roadmap:
**almost nothing was.** Company identity, timezone, currency, weekend days, the
eight attendance policies, overtime rules, leave types, report columns, the
app's API base URL, every app string and the app's tab list already came from
the database, config or the signed-in user. No TODOs, no dummy data, no static
chart arrays, and not one untranslated `Text()` in the app. Two things did not,
and both are now closed.

**Page sizes.** Twenty-four literals across twenty-one controllers, five
different values, no rule anybody could state for which list got which — drift,
not design. They live in `config/pagination.php` now, reached through
`perPage()` on the base `Controller` beside `companyId()`. The existing numbers
were kept rather than flattened: a dense audit table and an employee's own leave
list do not want the same count, and collapsing them would have been a visual
change made under cover of a refactor. A grep for `paginate([0-9]` found
twenty-three; the twenty-fourth was a `const PER_PAGE` and surfaced only because
`API-Reference_v1.md` documented a `per_page` for an endpoint the change had not
touched. The documentation caught what the search could not.

**`per_page` was advertised and never accepted.** `pageMeta()` has returned it
since the API was written, telling every client there was a page size worth
knowing about while no endpoint read one from the request. It is a real
parameter now, **clamped rather than validated** — a list that 422s because
somebody asked for one row too many fails a person reading their own leave in
order to protect a server that could have answered. Every endpoint's default is
the size it served before, so an app that sends nothing sees no change.

Nine tests, mutation-checked: dropping the `min()` fails the two clamp tests and
nothing else. **Verified on the live box after deploying `33fe439`**, which is
the part worth recording — `?per_page=5` answered 5, `?per_page=100000` answered
100 rather than an error, and no parameter answered 30, the pre-existing
default. Not left as a local test result.*
