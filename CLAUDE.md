# HR & Attendance Management — working notes

Laravel 12 + MySQL web dashboard in `hrms/`, Flutter client in `mobile/`, both on
the same API. This file is for whoever picks the project up next. It records the
things that are **not** obvious from reading the code, and the traps that have
already cost time.

---

## Running it

MySQL must be started **from the XAMPP Control Panel** — launching `mysqld.exe`
as a background task does not persist, it exits.

```bash
cd hrms
php artisan serve            # http://127.0.0.1:8000
php artisan test             # 874 tests, ~90s, SQLite in memory
```

`config('app.timezone')` is **deliberately UTC** and must stay that way. Per-company
time comes from `Company::tz()`. "Fixing" it to a local zone would shift what
"today" means for every other company.

### Signing in locally

The old `admin@hrms.test / password` **no longer exists**. Switch on the demo
panel instead — it puts one-click role buttons on the login page:

```
DEMO_QUICK_LOGIN=true
DEMO_QUICK_LOGIN_ACCOUNTS="test.admin@local.test:...,test.hr@local.test:..."
```

It is forced off when `APP_ENV=production`, and `hrms:preflight` fails a deploy
that still has it on.

### Browser testing

**The Claude-in-Chrome extension cannot reach `127.0.0.1:8000`** — it shows an
error page regardless of the URL, and it is a site permission only the user can
grant. Use the **chrome-devtools MCP** instead (`new_page`), which works against
localhost first time.

---

## Traps that have already bitten

These are not hypothetical. Each one shipped, and each was invisible until
something specific broke.

### 1. Date-cast columns and range queries

`work_date` and `shift_assignments.date` are `date` casts. MySQL holds a real
DATE; **every other engine stores `"2026-08-04 00:00:00"`**, and
`"2026-08-04 00:00:00" <= "2026-08-04"` is false as a string.

So `whereBetween` and `whereIn` on those columns **silently drop the last day of
every range**, and a single-day range returns nothing at all. Invisible in
production, and equally invisible in the tests, which run on SQLite.

**Always use the model scopes:**

```php
AttendanceLog::forDates($from, $to)      // never whereBetween('work_date', …)
ShiftAssignment::between($from, $to)     // never whereBetween('date', …)
LeaveRequest::overlapping($from, $to)
```

Four separate bugs came from this. Assume a fifth is waiting.

### 2. `validate()` drops absent nullable keys

`$request->validate(['x' => 'nullable|date'])` returns an array **without `x`**
when the caller never sent it. Reading `$data['x']` is then an undefined-index
500, not the fallback you intended. Bit twice (`employee_code`, `anchor_date`).

```php
$value = ($data['x'] ?? null) ?: $fallback;   // always
```

### 3. Blade directives glued to the preceding word

`clock@if($x)` is **not** a directive — Blade leaves it literal but still
compiles the `@endif`, so the view fails to parse at all. Took the live board
down entirely. Always leave a space before `@if`.

### 4. Undeclared `company_id` columns

MySQL carries them, migrations do not declare them, SQLite tests stay green
because the column does not exist there. Bit on `attendance_logs` and
`leave_balances`. All nine company-scoped tables were audited and are consistent;
if you add a tenth, declare it in a guarded `hasColumn` migration **and** fill it
in the model's `booted()`, not at call sites.

### 5. `actingAs` persists for the whole test

A "signed out" assertion after an `actingAs` call is still signed in and asserts
nothing. Log out explicitly, or build the fixture without authenticating.

---

## Conventions

- **Editing Blade files: use a `php <<'PHPEOF'` heredoc**, not the Edit tool and
  not inline `php -r`. The templates are tab-indented and the strings do not
  round-trip; nested quotes break in Git Bash.
- **Attendance is append-only.** Edit and delete throw; punches are voided
  instead, and every write records actor, source, IP and a full snapshot.
  `ActivityLog` and `AttendanceAuditEvent` refuse updates and deletes too.
- **Every new policy defaults to off.** `session_idle_timeout_minutes` (0),
  `enforce_geofence` (false), `require_two_factor_for_staff` (false). Each would
  otherwise change behaviour for a working installation on upgrade. They live in
  `Company::POLICY_DEFAULTS` and are edited at `/settings/policies`.
- **Reports return a uniform shape** — `title, subtitle, tiles, headings, rows` —
  so the screen, the PDF and the Excel export are all generic. Row keys must
  match `headings` exactly or the exports throw.
- **The API-docs test walks the route table.** A new endpoint fails the suite
  until it is written up in `API-Reference_v1.md`. That is deliberate.

---

## The four roles

Admin, HR, manager, employee — 18 permissions, all seeded by
`RolePermissionSeeder`.

| | Admin | HR | Manager | Employee |
|---|---|---|---|---|
| Lands on | `/dashboard` | `/dashboard` | `/employee/dashboard` | `/employee/dashboard` |
| Roles, policies, activity log, settings | ✅ | ❌ | ❌ | ❌ |
| Employees, reports, leave register | ✅ | ✅ | ❌ | ❌ |
| Team approvals tab | — | — | ✅ | ❌ |

**`manager` is a role *and* a relationship, and both must line up.** The role
grants the gate; `employees.manager_id` decides the scope. Role but no reports →
empty team, not an error. Reports but no role → 403.

The admin app is wrapped in `role:admin|hr`, which runs **before** any
`permission:` middleware on the route inside it. Adding `|approve-leave` to a
route in that group advertises manager access that can never be reached — the
manager holds the permission but not the role.

**Every mobile API call needs an employee record.** `ApiController::employee()`
aborts 403 "No employee record is linked to this account". A hand-created admin
has no employee row, so it signs in and then 403s on nearly everything.

---

## Where things stand

The web dashboard is **complete except for four deliberate omissions**. The
mobile app and the API are done. `Feature-List_Web-and-App.md` is the live status
board — read it first — and `hrms/config/roadmap.php` drives the phase panel on
the Settings screen.

**Not built, by decision:**

- **AI assistant** (Part D) — out of scope, parked.
- **Multi-company tenancy** (A2.10) — the schema is company-scoped throughout, so
  this is a routing and onboarding job rather than a data-model one.
- **Conditional rules engine** (A2.9, A6.6) — the policies are configurable, but
  there is no if-this-then-that builder.
- **Drag-and-drop roster planner** (A5.8) — the grid planner works; the dragging
  does not exist.
- **QR image on the 2FA setup screen** — composer cannot currently resolve a new
  dependency (an unrelated `league/commonmark` advisory blocks the resolver), so
  `App\Support\Totp` is hand-rolled and verified against the RFC 6238 vectors.
  Setup is by typed key, which every authenticator supports. When composer is
  unblocked, rendering the existing `otpauth://` URI as a QR is the only change.

**Inert until configured — neither is a code change:**

- `MAIL_MAILER` is still `log`. Password resets, leave decisions, scheduled
  reports and document-expiry warnings are all built and tested, and all go
  nowhere until real SMTP is set.
- Push is silent until a Firebase project exists. See `Push-Notifications_Setup.md`.

---

## Deploying

**No production server exists yet.** The tooling is written and tested:

- `Deployment-Guide_Production.md` — the runbook.
- `deploy/` — nginx config, the systemd worker unit, the cron line, `deploy.sh`.
- `hrms/.env.production.example` — the env template.
- `php artisan hrms:preflight` — gates a deploy. Fails on debug-on,
  `MAIL_MAILER=log`, a localhost or http `APP_URL`, the sync queue, no recent
  backup, a bad company timezone, the demo panel left on, and seeded passwords.

**The database dump is not the whole backup.** Contracts, ID scans and employee
photos live on disk (`storage/app/employee-documents/`, `storage/app/public/avatars/`).
A restored database with no files behind it is a list of documents that all 404.

**`public/storage` must exist.** Without the symlink every employee photo 404s
with nothing in the log and no error on screen. `deploy.sh` runs `storage:link`
and preflight checks for it.

---

## Git

`origin` is `github.com/jasonapp2244/hr_-and_attendance_management_web_application`.

`git push` can hang on a hidden credential-manager dialog —
**`GIT_TERMINAL_PROMPT=0 git push`** completes instantly. `gh` is not installed,
so PRs must be opened through a browser link.
