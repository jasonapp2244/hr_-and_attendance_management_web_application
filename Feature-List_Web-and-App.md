# Employment Management Portal — Master Feature List (Web + Mobile App)

**Project:** Employment Management Portal
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
| A1.2 | Role-based access — Admin, HR, Employee (Spatie RBAC) | ✅ |
| A1.3 | Granular permissions per role (18 seeded) | ✅ |
| A1.4 | Roles & permissions editor UI | ✅ |
| A1.5 | Profile page + change password | ✅ |
| A1.6 | Password reset via email ("forgot password") | ✅ request → emailed link → new password; revokes app tokens and push. Needs a real `MAIL_MAILER` to leave the box |
| A1.7 | Two-factor authentication (2FA) for Admin/HR | ✅ TOTP, any authenticator app; secret encrypted at rest, 8 single-use recovery codes, optional company-wide requirement on Admin/HR. Setup is by typed key — no QR image yet |
| A1.8 | Login activity & audit trail (who did what, when) | ✅ immutable log of sign-ins, failed attempts, lockouts, timeouts, password and settings changes; filterable by event, person, date and IP. Admin-only |
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
| A2.9 | Attendance & leave policy rules engine | 🟡 the policies themselves are configurable — working week, reminder and auto-close windows, geofence, 2FA requirement, idle timeout — but there is no conditional rule builder |
| A2.10 | Multi-company (SaaS tenancy) support | ⬜ |

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
| A4.15 | Break in / break out punches | ✅ |
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
| A5.7 | Break rule configuration | 🟡 unpaid break deducted from paid hours, and real break punches now override it (A4.15); no per-shift break policy builder |
| A5.8 | Roster drag-and-drop planner + publish to staff | 🟡 grid planner + draft/publish; no drag-and-drop |
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

---

# PART B — MOBILE APP (Android + iOS, Employee-facing)

> Every screen below consumes the shared Laravel API (Part C), which is complete.
> The app lives in `mobile/` — Flutter, nine screens, tested with 35 unit tests
> and 10 integration tests driven against a live server.

## B1. Onboarding & Auth
| # | Feature | Status |
|---|---|---|
| B1.1 | Splash + branded onboarding screens | 🟡 splash holds while the token is verified; no onboarding carousel |
| B1.2 | Login with email + password (Sanctum token) | ✅ |
| B1.3 | Biometric unlock (fingerprint / Face ID) | ⬜ |
| B1.4 | Forgot password flow | ✅ app requests the link; the link opens the web reset page, not a token screen in the app |
| B1.5 | Stay-logged-in / secure token refresh | ✅ token in the device keystore, re-verified against `/auth/me` at launch |
| B1.6 | Device registration & binding (one account ↔ trusted device) | ⬜ |
| B1.7 | Logout / remote session revoke | ✅ sign out, and sign out everywhere for a lost handset |

## B2. Attendance (app core)
| # | Feature | Status |
|---|---|---|
| B2.1 | Big one-tap Check In / Check Out button | ✅ double-tap reads as success, not as an error |
| B2.2 | Live status card (checked in at 09:02, hours so far) | ✅ ticks locally between refreshes |
| B2.3 | GPS capture at punch | ✅ `geolocator`, permission asked at the first punch; no fix, no permission or no signal sends the punch without coordinates |
| B2.4 | Offline punch queue → auto-sync when back online | ⬜ |
| B2.5 | Geofence-aware punch (warn or block outside office) | ⬜ optional |
| B2.6 | Break in / break out | ⬜ needs A4.15 first |
| B2.7 | Mock-location / rooted-device detection | ⬜ |
| B2.8 | Home-screen widget / quick action for fast punching | ⬜ |

## B3. Employee Self-Service
| # | Feature | Status |
|---|---|---|
| B3.1 | View own profile & employment details | ✅ |
| B3.2 | Edit permitted fields (phone, address, emergency contact) | ⬜ `PUT /profile` exists; the app never calls it |
| B3.3 | Change password | ✅ |
| B3.4 | Attendance history with monthly calendar view | 🟡 day rows over 7/30/92 days; no calendar grid |
| B3.5 | Personal attendance score / on-time streak | 🟡 present/late/leave/absent/worked totals; no score or streak |
| B3.6 | View assigned shift & upcoming roster | ✅ published roster only |
| B3.7 | Download own payslip / documents | ⬜ needs A3.8 |
| B3.8 | Company directory (colleagues, departments) | ⬜ no endpoint yet |

## B4. Leave (app)
| # | Feature | Status |
|---|---|---|
| B4.1 | Apply for leave (type, dates, reason, attachment) | 🟡 type, dates and reason; no attachment |
| B4.2 | View leave balances | ✅ |
| B4.3 | Track request status & history | ✅ |
| B4.4 | Cancel / withdraw a pending request | ✅ |
| B4.5 | Manager approval inbox (approve/reject in-app) | ✅ tab appears only with `approve-leave` |
| B4.6 | Team leave calendar | ⬜ |

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
> iOS needs, are the whole of what is left — see `Push-Notifications_Setup.md`.

| # | Feature | Status |
|---|---|---|
| B5.1 | Clock-in reminder at shift start | ⬜ no server job either |
| B5.2 | Clock-out reminder at shift end | ✅ end to end; needs credentials to leave the box |
| B5.3 | Leave approved / rejected | ✅ end to end, both stages of the approval chain |
| B5.4 | Schedule / roster updated | ⬜ needs A9.5 |
| B5.5 | HR announcements & broadcasts | ⬜ |
| B5.6 | In-app notification centre | 🟡 a snack bar for a message arriving with the app open; no history |

## B6. App Experience
| # | Feature | Status |
|---|---|---|
| B6.1 | Dark mode | ✅ light and dark themes, follows the system |
| B6.2 | Multi-language support | ⬜ |
| B6.3 | Offline-first cache of profile, history, roster | ⬜ |
| B6.4 | Accessibility (font scaling, contrast) | ⬜ not audited |
| B6.5 | Crash & analytics reporting | ⬜ |
| B6.6 | Force-update / maintenance-mode gate | ⬜ |

## B7. Manager Mode (optional in-app role)
| # | Feature | Status |
|---|---|---|
| B7.1 | Team attendance today | ✅ present vs in-now reported separately |
| B7.2 | Approve leave / regularisation from phone | ✅ leave only — regularisation needs A4.13 |
| B7.3 | Team roster view | ✅ `GET /team/roster` plus a Roster tab in the app — a week per direct report, published days only, with leave outranking a rostered shift |

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

*Not built: biometrics, and anything offline.*

---

# PART C — SHARED BACKEND & API

| # | Feature | Status |
|---|---|---|
| C1.1 | **Laravel Sanctum token auth + `routes/api.php`** | ✅ |
| C1.2 | `/auth/login`, `/auth/logout`, `/auth/me` (+ `logout-all`, `devices`) | ✅ |
| C1.3 | `/attendance/check`, `/attendance/history`, `/attendance/today` | ✅ same AttendanceService as the web button — one set of punch rules |
| C1.4 | `/leave/*` endpoints | ✅ balances, apply, list, withdraw + the manager inbox — all via LeaveService |
| C1.5 | `/schedule`, `/profile` endpoints | ✅ published roster only, leave/holiday/weekend aware; profile read + contact edit + password |
| C1.6 | Device token registration for push | ✅ register/list/unregister; cleared on sign-out. Delivery is Phase 5 |
| C1.7 | API rate limiting + throttling | ✅ per-user limiters — 120/min ceiling, login 5, punch 20, writes 30 |
| C1.8 | Consistent JSON error format + API versioning (`/api/v1`) | ✅ |
| C1.9 | Queue worker + scheduler (reminders, auto-absent, reports) | 🟡 three scheduled jobs; queued notifications survive a deleted record and retry a bad send. The cron line and the worker unit are written (`deploy/`) but not yet installed on a server |
| C1.10 | Immutable audit log for attendance records | ✅ punches are append-only (edit/delete refused); every write records actor, source, IP and a full snapshot |
| C1.11 | Automated test suite (feature + unit) | ✅ 908 tests covering attendance, leave, roster, swaps, the API, the audit trail, password reset, push, backups, install, employee import and preflight |
| C1.12 | API documentation (Scribe / OpenAPI) | ✅ `API-Reference_v1.md`, kept honest by a test that walks the route table |
| C1.13 | Database backup & restore strategy | ✅ `db:backup --verify` nightly — dumps, restores into a scratch database to prove it reads back, then rotates |
| C1.14 | Production deployment (HTTPS, env hardening) | 🟡 written, not run — `deploy/` scripts, nginx + systemd + cron, `.env.production.example`, `emp:preflight` and `Deployment-Guide_Production.md`. No server exists yet |
| C1.15 | Push delivery to handsets (FCM v1) | ✅ channel alongside database and mail; silent until a service-account key is configured; deletes handsets FCM reports UNREGISTERED, keeps ones that merely 503'd |
| C1.16 | Public privacy policy + account-deletion pages | ✅ no login required — both stores demand it before an app with accounts is listed |
| C1.17 | Real-install setup, no demo data | ✅ `emp:install` creates the company and first admin, or attaches an admin to an existing company (`--company-id`); validated timezone, roles seeded, one transaction. `db:seed` now makes roles only. `emp:purge-demo --dry-run` clears a seeded database and names the real rows the cascade would take with it |

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
| **Stage 5** | B1–B3 — Mobile app v1 (login, punch, self-service) | 🟡 **In progress — the screens work and punches now carry GPS; biometrics and offline are missing** |
| **Stage 6** | B4–B5 — Leave + push in app | ✅ Leave and push both done. Push is silent until the Firebase project exists — console work, not code |
| **Stage 7** | A4.12–A4.15, A7.10–A7.14 | ✅ Attendance depth + reporting — correction, regularisation, overtime, break punches, payroll export, leave reports, scheduled delivery, report builder |
| **Stage 8** | A1.7–A1.9, A2.3, A2.8, A4.16 | ✅ 2FA, the security trail, the idle timeout, the working-week editor and geofence enforcement |
| **Stage 9** | A3.7–A3.11, A6.4/A6.7/A6.9, A4.19, A9.3 | ✅ Photos, the document vault, emergency contacts, the org chart, the roster export, leave accrual and carry-forward, the leave calendar, the live board and the late-arrival digest |
| **Stage 10** | A3.12, A4.11, A8.4–A8.6, A9.5 | ✅ Dashboards per role and per person, week-on-week trends, weekly rollups, schedule alerts and on/offboarding checklists |
| **Stage 11** | B7 — Manager mode in the app | ✅ Team roster added; approvals and team attendance already shipped |
| **Stage 12** | D1 — AI HR Assistant | ⬜ Out of scope for now, by decision. Needs mature data across attendance + leave |

### The one thing gating the rest

**C1.14 — deploy to a real domain.** A handset cannot resolve `127.0.0.1`, FCM will
not call back a laptop, and neither store accepts a privacy-policy URL pointing at
localhost. Push (B5), store submission and real-device testing all sit behind it,
and the scripts to do it are already written. See `Deployment-Guide_Production.md`.

---

## Counts

| Area | Built | Partial | Planned | Total |
|---|---|---|---|---|
| Web Dashboard (A) | 82 | 9 | 3 | 94 |
| Mobile App (B) | 20 | 5 | 19 | 44 |
| Backend / API (C) | 15 | 2 | 0 | 17 |
| AI Assistant (D) | 0 | 0 | 7 | 7 |
| **Total** | **117** | **16** | **29** | **162** |

**The web dashboard is complete, AI excluded.** Stages 8, 9, 10 and 11 are all
delivered. Three planned rows and nine partial ones remain across Part A, and none
of them blocks a production deployment. The AI assistant (Part D) is deliberately
out of scope.

**Still open, and worth being explicit about:** multi-company tenancy (A2.10), a
conditional rules engine (A2.9, A6.6), a drag-and-drop roster planner (A5.8), a
per-shift break policy builder (A5.7), a team leave calendar in the app (B4.6),
biometrics and offline punching (B1.3, B2.4, B6.3), and a QR image on the 2FA
setup screen — the key can be typed in, which every authenticator supports.

**Two things are code-complete but inert until configured**, and neither is a
code change: `MAIL_MAILER` is still `log`, so no email leaves the box; and push
stays silent until a Firebase project exists.

---

*Updated 2026-08-07 from the live codebase — `emp/` and `mobile/` both read directly
rather than from the previous edition of this file. Supersedes the stale build-status
section of `Phase-1_Admin-Dashboard_Attendance_SOW.md`.*
