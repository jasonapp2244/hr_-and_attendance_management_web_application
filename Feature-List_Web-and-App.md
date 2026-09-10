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
| A1.2 | Role-based access — Admin, HR, Manager, Employee (Spatie RBAC) | ✅ four roles, each landing on the only area it can reach. Manager is a first-class role with its own dashboard (A10), not a permission bolted onto the employee portal |
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
| A2.9 | Attendance & leave policy rules engine | 🟡 the policies themselves are configurable — working week, reminder and auto-close windows, geofence, 2FA requirement, idle timeout, directory contact details — but there is no conditional rule builder |
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
| B1.6 | Device registration & binding (one account ↔ trusted device) | ⬜ |
| B1.7 | Logout / remote session revoke | ✅ sign out, and sign out everywhere for a lost handset |

## B2. Attendance (app core)
| # | Feature | Status |
|---|---|---|
| B2.1 | Big one-tap Check In / Check Out button | ✅ double-tap reads as success, not as an error |
| B2.2 | Live status card (checked in at 09:02, hours so far) | ✅ ticks locally between refreshes |
| B2.3 | GPS capture at punch | ✅ `geolocator`, permission asked at the first punch; no fix, no permission or no signal sends the punch without coordinates |
| B2.4 | Offline punch queue → auto-sync when back online | ✅ A failed punch is kept on the handset at **the moment it was tapped** and delivered by `POST /attendance/sync` when there is something to deliver it over. **This is the one place the device clock is trusted**, by decision: stamping a queued punch on arrival would file a 09:00 check-in as 17:00 and hand payroll a wrong number. It is bounded — future times and anything over 48h are refused rather than clamped — and labelled `source: mobile_offline` with the delivery delay in `notes`. No connectivity library: those answer "is there an interface", which is not the question on hotel wifi behind a captive portal, so a punch is always attempted first and queued only when the attempt actually fails. Held in a JSON file, so it survives a force-quit; cleared on sign-out, since undelivered punches belong to whoever made them |
| B2.5 | Geofence-aware punch (warn or block outside office) | ⬜ optional |
| B2.6 | Break in / break out | ✅ `POST /attendance/break` wraps the same `recordBreak` the web portal's button has called since A4.15; the Clock screen gains a Start/End break button, shown only on the clock, and the status card reads "On a break" as a third state rather than a fourth word for clocked out. `can_break` defaults to **false** when absent, so a build talking to an older server shows no button instead of one that 404s. Punch rows now label all four types — a ternary on `isIn` rendered `break_start` as "Checked out", the same mistake the server made. (The row read "needs A4.15 first" long after A4.15 shipped) |
| B2.7 | Mock-location / rooted-device detection | ⬜ |
| B2.8 | Home-screen widget / quick action for fast punching | ⬜ |

## B3. Employee Self-Service
| # | Feature | Status |
|---|---|---|
| B3.1 | View own profile & employment details | ✅ |
| B3.2 | Edit permitted fields (phone, address, emergency contact) | 🟡 name and phone edit from the Profile tab through `PUT /profile`, which had existed unused since the API shipped. **The sign-in address is shown but not editable, deliberately** — changing it is account takeover in two steps (set it to your own, then "forgot password"), and it would need nothing but an unlocked phone; changing the *password* already demands the current one for that reason. Address and emergency contact live on `employees` and have no endpoint at all — a server change, not an app one |
| B3.3 | Change password | ✅ |
| B3.4 | Attendance history with monthly calendar view | 🟡 day rows over 7/30/92 days; no calendar grid |
| B3.5 | Personal attendance score / on-time streak | 🟡 present/late/leave/absent/worked totals; no score or streak |
| B3.6 | View assigned shift & upcoming roster | ✅ published roster only |
| B3.7 | Download own payslip / documents | ✅ `GET /documents` lists the caller's own shelf (expiring first, same four `expiry_state` values as the web badge) and `GET /documents/{id}` streams the file; **My documents** opens from the Profile tab, downloads through the token and hands the file to whatever the phone opens that type with. Saved to the *temporary* directory, not Documents — these are passport scans, and leaving copies where every other app can browse them would undo the point of an authenticated endpoint. `notes` and the uploader are withheld server-side. **No payslip** — there is no payroll module, only the hours export (A7.14) |
| B3.8 | Company directory (colleagues, departments) | ✅ **Colleagues** opens from the Profile tab — searchable, debounced so a five-letter name is one request rather than five. Active staff of the caller's own company only. Contact details sit behind the `directory_show_contact_details` policy, **off by default**: there is one phone column on an employee record, and where staff have no desk line it holds a personal mobile. Call and email buttons are drawn only when the company shares details **and** that person has some — the app reads `shows_contact_details` rather than inferring from a missing phone, so it never draws a button that cannot do anything. Everything HR-grade — DOB, address, national ID, personal email, emergency contact, the reporting line — is never returned at any setting |
| B3.9 | Raise an attendance correction | ✅ **Corrections** opens from History — next to the record it disputes, since asking for one is rare and makes sense nowhere else. Pick a recent punch to dispute or leave it on "one is missing"; the server tells the two apart from whether `attendance_log_id` is present, so the form never sends a mode that could disagree with itself. Break punches are filtered out of the picker — only `in` and `out` are correctable. The date picker will not go past today, because the server refuses a future correction and offering one would be a trap. Raising and withdrawing only: no decide button for anybody, manager included. Added as a row because Part B only ever tracked the *manager* half (B7.2), which left the employee half invisible |

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
| B7.1 | Team attendance today | ✅ present vs in-now reported separately |
| B7.2 | Approve leave / regularisation from phone | ✅ **leave only, and settled** — a correction is HR's alone, not a manager's. Leave approval is manager-then-HR; regularisation has no manager step, so an approve button in the manager tab would advertise a stage that does not exist. The employee half is `POST /attendance/regularisations` (A4.13 on the API); deciding stays on the web behind `manage-attendance` |
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
| C1.5 | `/schedule`, `/profile`, `/documents` endpoints | ✅ published roster only, leave/holiday/weekend aware; profile read + contact edit + password; own documents listed and streamed, read-only |
| C1.6 | Device token registration for push | ✅ register/list/unregister; cleared on sign-out. Delivery is Phase 5 |
| C1.7 | API rate limiting + throttling | ✅ per-user limiters — 120/min ceiling, login 5, punch 20, writes 30 |
| C1.8 | Consistent JSON error format + API versioning (`/api/v1`) | ✅ |
| C1.9 | Queue worker + scheduler (reminders, auto-absent, reports) | 🟡 three scheduled jobs; queued notifications survive a deleted record and retry a bad send. The cron line and the worker unit are written (`deploy/`) but not yet installed on a server |
| C1.10 | Immutable audit log for attendance records | ✅ punches are append-only (edit/delete refused); every write records actor, source, IP and a full snapshot |
| C1.11 | Automated test suite (feature + unit) | ✅ 1212 server tests covering attendance, leave, roster, swaps, the API, the audit trail, password reset, push, backups, install, employee import, preflight and the manager role, plus 193 in the app — models, formatting, offline behaviour, the biometric lock, the gate, crash reporting, accessibility and the translations |
| C1.12 | API documentation (Scribe / OpenAPI) | ✅ `API-Reference_v1.md`, kept honest by a test that walks the route table |
| C1.13 | Database backup & restore strategy | ✅ `db:backup --verify` nightly — dumps, restores into a scratch database to prove it reads back, then rotates |
| C1.14 | Production deployment (HTTPS, env hardening) | 🟡 written, not run — `deploy/` scripts, nginx + systemd + cron, `.env.production.example`, `emp:preflight` and `Deployment-Guide_Production.md`. No server exists yet |
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
| **Stage 5** | B1–B3 — Mobile app v1 (login, punch, self-service) | ✅ The screens work, punches carry GPS, the app is usable offline, and the handset can be held behind its own biometric check. What is still open across B1–B3 is optional or is a server job, not app v1: device binding (B1.6), the on-phone geofence and mock-location checks (B2.5, B2.7), a home-screen widget (B2.8), a calendar grid and a personal score (B3.4, B3.5), and the address and emergency-contact fields, which have no endpoint to write to |
| **Stage 6** | B4–B5 — Leave + push in app | ✅ Leave and push both done. Push is silent until the Firebase project exists — console work, not code. One code exception: B5.4's `route: schedule` is sent but not parsed by the app |
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

### Roles on the phone (Stage 16)

The web separates the four roles properly. The app was written employee-first
and grew a manager tab, and the seams show in three places.

| | What the app gives them | Right? |
|---|---|---|
| **Employee** | Clock, History, Leave, Schedule, Profile, My documents | ✅ |
| **Manager** | The above plus Team — approvals, team attendance, published roster | ✅ |
| **HR** | The employee screens only — **desk-only by decision**, see below | ✅ |
| **Admin** | Signs in, then 403s on everything — no employee record | ⚠️ by design, but poorly explained |

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

### The one thing gating the rest

**C1.14 — deploy to a real domain.** A handset cannot resolve `127.0.0.1`, FCM will
not call back a laptop, and neither store accepts a privacy-policy URL pointing at
localhost. Push (B5), store submission and real-device testing all sit behind it,
and the scripts to do it are already written. See `Deployment-Guide_Production.md`.

---

## Counts

| Area | Built | Partial | Planned | Total |
|---|---|---|---|---|
| Web Dashboard (A) | 92 | 9 | 4 | 105 |
| Mobile App (B) | 36 | 5 | 4 | 45 |
| Backend / API (C) | 16 | 2 | 0 | 18 |
| AI Assistant (D) | 0 | 0 | 7 | 7 |
| **Total** | **144** | **16** | **15** | **175** |

**The web dashboard is complete, AI excluded.** Stages 8 through 12 are all
delivered. Four planned rows and nine partial ones remain across Part A, and none
of them blocks a production deployment. The AI assistant (Part D) is deliberately
out of scope.

**Still open, and worth being explicit about:** multi-company tenancy (A2.10), a
conditional rules engine (A2.9, A6.6), a drag-and-drop roster planner (A5.8), a
per-shift break policy builder (A5.7), roster editing and attendance correction
by managers (A10.11 — held with `manage-shifts` and `manage-attendance` on
purpose), a team leave calendar in the app (B4.6), and a QR image on the 2FA
setup screen — the key can be typed in, which every authenticator supports.

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
API never exposed it. B5.4 was the exception and is now 🟡 rather than ⬜ — the
server does send that push; only the app's `PushRoute` enum has not heard of it.
A row that names an *internal* dependency goes stale the day that dependency
ships, and nothing fails when it does, so re-read these against the code rather
than trusting the note.

---

*Updated 2026-09-10 from the live codebase — `hrms/` and `mobile/` both read directly
rather than from the previous edition of this file. Supersedes the stale build-status
section of `Phase-1_Admin-Dashboard_Attendance_SOW.md`.*
