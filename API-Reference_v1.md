# Employment Management Portal — Mobile API Reference (v1)

**Base URL:** `https://<host>/api/v1`
**Auth:** Laravel Sanctum personal access tokens (`Authorization: Bearer <token>`)
**Format:** JSON in, JSON out. Always send `Accept: application/json`.

> Send `Accept: application/json` on every request. Without it Laravel may answer a
> failure with an HTML redirect instead of the error shape below.

This document is checked by `tests/Feature/Api/ApiDocsTest.php`: every route the
application registers under `api/v1` must appear here, so the reference cannot
quietly fall behind the code.

---

## 1. Conventions

### Versioning

The version is in the path, not a header. An app already installed on someone's
phone cannot be forced to upgrade, so `v2` has to be able to run alongside `v1`
rather than replacing it under the same URLs.

### Response shape

Every response carries `ok`, so a client branches on one field rather than
inferring from the status code.

**Success**

```json
{ "ok": true, "...": "endpoint-specific keys" }
```

**Failure** — the same three keys for every failure mode, whatever caused it:

```json
{ "ok": false, "error": "validation_failed", "message": "The given data was invalid." }
```

Validation failures — and only validation failures — add per-field detail:

```json
{
  "ok": false,
  "error": "validation_failed",
  "message": "The given data was invalid.",
  "errors": { "end_date": ["The end date field is required."] }
}
```

### Error codes

| `error` | HTTP | Meaning |
|---|---|---|
| `validation_failed` | 422 | Input rejected. See `errors` for the fields. |
| `unauthenticated` | 401 | Missing, malformed or revoked token. |
| `forbidden` | 403 | Authenticated, but not allowed to do this. |
| `not_found` | 404 | No such endpoint or record. |
| `too_many_requests` | 429 | Rate limited. See §2. |
| `invalid_credentials` | 401 | Login: wrong address or password. |
| `account_disabled` | 403 | Login: the account has been switched off. |
| `duplicate_scan` | 429 | Punch: within the cooldown of the last one. |
| `no_office` | 422 | Punch: the company has no office set up. |
| `wrong_password` | 422 | Password change: current password incorrect. |
| `invalid_range` | 422 | `from` is after `to`. |
| `range_too_large` | 422 | Date window exceeds the endpoint's maximum. |
| `server_error` | 500 | Unexpected. The detail is in the server log, not the response. |

A `403` always means *not allowed*, whether it came from a permission check or
an ownership check — the `message` says which.

### Dates and times

- **Dates** are `YYYY-MM-DD`.
- **Timestamps** are ISO 8601 **with the company's UTC offset**, e.g.
  `2026-07-30T16:57:07-04:00`. Render them in the company timezone
  (`user.company.timezone`), not the handset's — an employee travelling must
  still see their own office's clock.
- **Times of day** on a shift are `HH:MM:SS` in the company timezone.
- **Numbers** are JSON numbers: a whole day count arrives as `3`, a half day as
  `0.5`. Parse as a decimal, not an integer.

### Server-authoritative time

The device clock is never trusted. A punch is stamped by the server; sending a
timestamp has no effect.

---

## 2. Rate limits

Limits are per **user** where a token is present, falling back to IP for calls
made before anyone is identified. A whole office behind one address shares an
IP, so limiting on that would have one busy person throttle their colleagues.

| Limiter | Applies to | Limit |
|---|---|---|
| `api` | every endpoint | 120 / minute |
| `login` | `POST /auth/login` | 5 / minute, per address **and** IP |
| `punch` | `POST /attendance/check`, `POST /attendance/break` | 20 / minute |
| `write` | every endpoint that creates or changes a record | 30 / minute |

The stricter limiters stack on top of the ceiling. Every response carries
`X-RateLimit-Limit` and `X-RateLimit-Remaining`; a `429` adds `Retry-After`.

---

## 3. Service

### `GET /ping`

No token required. Use it to check reachability and confirm the API version
before showing a login screen.

```json
{ "ok": true, "service": "Employment Management Portal", "version": "v1", "time": "2026-07-30T19:13:39+00:00" }
```

---

## 4. Authentication

### `POST /auth/login`

No token required.

| Field | Type | Notes |
|---|---|---|
| `email` | string, required | |
| `password` | string, required | |
| `device_name` | string, required, ≤100 | Names the token so it can be recognised and revoked. Use something stable and human — "Ann's Pixel". |

Logging in again from the **same** `device_name` replaces that device's token
rather than issuing a second one, so a reinstall does not leave a valid
credential behind.

```json
{
  "ok": true,
  "token": "3|aFK5Blm...",
  "user": {
    "id": 3,
    "name": "James Smith",
    "email": "james.smith@acme.test",
    "roles": ["employee", "manager"],
    "permissions": ["view-attendance", "approve-leave", "view-team", "approve-swaps"],
    "company": { "id": 1, "name": "Acme", "timezone": "America/New_York", "currency": "EUR" },
    "employee": {
      "id": 1, "employee_code": "EMP-0001", "full_name": "James Smith",
      "department": "Engineering", "designation": "Software Engineer",
      "office": "Head Office", "work_mode": "office", "is_manager": true
    }
  }
}
```

Store the token in the platform keychain, never in plain preferences.

`employee` is `null` for an account with no employee record (an admin login, for
example). Such an account can sign in and read its profile, but every
employee-scoped endpoint answers `403`.

A wrong address and a wrong password give the **same** answer — saying which was
wrong would tell an attacker which addresses exist.

**Failures:** `invalid_credentials` (401) · `account_disabled` (403) ·
`validation_failed` (422) · `too_many_requests` (429)

### `POST /auth/forgot-password`

Starts a password reset. **No token required** — somebody who could authenticate
would not need this.

| Field | Type | Notes |
|---|---|---|
| `email` | string, required, email | The address the person signs in with. |

```json
{
  "ok": true,
  "message": "If that email address has an account, a reset link is on its way."
}
```

**The answer is the same whatever happened** — address unknown, account
deactivated, or a link already sent a moment ago all return this. Do not try to
branch on it: there is nothing to branch on, by design. For an HR system the
staff list is exactly what an attacker is after, and a form that says "no such
user" is a way to enumerate it.

The link in the email opens the **web** reset page. The app's part of the flow
ends with this call — do not build a token entry screen. A reset has to work
from a borrowed laptop, because the phone is often the thing the person has lost
access to.

Completing a reset **revokes every API token and push registration** for that
account, so a handset that was signed in will get `unauthenticated` on its next
call and must send the person back to the login screen.

**Failures:** `validation_failed` (422) · `too_many_requests` (429, the `login`
limiter — 5/minute)

### `GET /auth/me`

The payload above, for a client restoring a session on launch. Call it at
startup: roles and permissions change without the app knowing.

### `POST /auth/logout`

Signs out this device.

| Field | Type | Notes |
|---|---|---|
| `push_token` | string, optional | Send the token registered with `POST /devices`. Without it the handset keeps receiving this person's notifications after they have signed out. |

### `POST /auth/logout-all`

Signs out every device and removes every registered handset — for a phone that
has been lost.

```json
{ "ok": true, "message": "Signed out on all devices.", "tokens_revoked": 2, "devices_removed": 1 }
```

### `GET /auth/devices`

Sessions currently holding a valid token, for a "where am I signed in" screen.
`current` marks the one making the call.

```json
{ "ok": true, "devices": [
  { "id": 4, "name": "Ann Pixel", "last_used_at": "2026-07-30T18:02:11-04:00", "created_at": "...", "current": true }
] }
```

---

## 5. Attendance

### `POST /attendance/check`

One endpoint for both directions. **The server decides whether this is a clock
in or a clock out** from what is already on record — the app does not say, so a
stale screen cannot post the wrong one.

| Field | Type | Notes |
|---|---|---|
| `latitude` | numeric, optional, −90…90 | Recorded for HR. Never blocks a punch. |
| `longitude` | numeric, optional, −180…180 | As above. |

Location is a record, not a gate: office, remote and hybrid staff all clock in
from wherever they are. If the handset refuses permission, send the punch
without it.

```json
{
  "ok": true,
  "punch": {
    "id": 109, "type": "in", "status": "late",
    "scanned_at": "2026-07-30T16:57:07-04:00", "time": "04:57 PM",
    "office": "Head Office", "source": "mobile"
  },
  "next_action": "out",
  "message": "You clocked IN at 04:57 PM."
}
```

`type` is `in` or `out`. `status` is `ontime`, `late` or `early_leave`, measured
against the shift rostered for that day.

**Failures:** `duplicate_scan` (429, within the cooldown of the last punch) ·
`outside_geofence` (422) · `no_office` (422) · `forbidden` (403, no employee
record) · `too_many_requests` (429, the `punch` limiter)

Treat `duplicate_scan` as success from the user's point of view — the punch they
wanted is already recorded.

`outside_geofence` only ever appears for a company that has switched enforcement
on (A4.16, off by default). Its `message` names the distance, so show it rather
than a generic failure — "move closer" is the one thing the person can act on.
A punch that arrives with no coordinates is never fenced, so refusing location
permission does not lock anybody out.

### `POST /attendance/break`

Start or end a break. **The server decides which**, from the day's punches —
same reasoning as `check`, and for the same reason.

| Field | Type | Notes |
|---|---|---|
| `latitude` | numeric, optional, −90…90 | Recorded, never a gate. |
| `longitude` | numeric, optional, −180…180 | As above. |

```json
{
  "ok": true,
  "punch": {
    "id": 114, "type": "break_start", "status": "ontime",
    "scanned_at": "2026-07-30T13:02:44-04:00", "time": "01:02 PM",
    "office": "Head Office", "source": "mobile"
  },
  "on_break": true,
  "next_break_action": "end",
  "message": "Break started at 01:02 PM. Your worked time pauses until you return."
}
```

`type` is `break_start` or `break_end`. `status` is always `ontime`: a break is
neither early nor late, and reusing the punch statuses here would hang a
meaningless "late" badge on somebody's lunch.

A break is only available **on the clock**. Starting one while checked out, or
ending one nobody started, is refused rather than guessed — an unpaired break
marker has to be discarded by the hours calculation, so the button would appear
to work while the total silently did not move. Read `can_break` from
`/attendance/today` and grey the button instead of letting the tap fail.

**Failures:** `break_not_available` (422, not clocked in) · `duplicate_scan`
(429) · `no_office` (422) · `forbidden` (403) · `too_many_requests` (429, the
`punch` limiter)

### `GET /attendance/today`

Everything a home screen needs.

```json
{
  "ok": true,
  "date": "2026-07-30",
  "server_time": "2026-07-30T16:59:17-04:00",
  "timezone": "America/New_York",
  "next_action": "out",
  "can_check": true,
  "on_break": false,
  "break_started_at": null,
  "can_break": true,
  "next_break_action": "start",
  "is_clocked_in": true,
  "worked_minutes": 2,
  "punches": [ { "id": 109, "type": "in", "status": "late", "scanned_at": "...", "time": "04:57 PM", "office": "Head Office", "source": "mobile" } ],
  "shift": { "id": 1, "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00", "late_grace_minutes": 15, "crosses_midnight": false },
  "is_day_off": false,
  "holiday": null,
  "leave": null
}
```

- `date` is the day a punch made **now** would count against. On a shift that
  crosses midnight this is still yesterday — a night worker opening the app at
  02:00 sees the day their shift started, matching how the punch is filed.
- `can_check` is `false` only while the duplicate cooldown is running. Grey the
  button rather than letting a tap fail.
- `is_clocked_in` stays `true` **through a break** — the person has not gone
  home. Do not derive it from the last entry in `punches`: `break_end` is
  neither `in` nor `out`, and reading it as the end of the day would offer
  "Check In" to somebody already on the clock and open a second stretch.
  `next_action`, `is_clocked_in` and `can_break` are the server's answer to
  exactly that question; use them.
- `can_break` is `true` only on the clock and outside the cooldown.
  `next_break_action` (`start` / `end`) labels the break button, and is kept
  apart from `next_action` because one screen carries both and they move
  independently.
- `break_started_at` is set only while `on_break` is `true`.
- `worked_minutes` counts closed in/out pairs, plus the open stretch up to now
  when `is_clocked_in` is true, **less any completed break**.
- `leave` being set does **not** disable the button. Somebody who books a day
  off and comes in anyway worked, and the record has to say so.

### `GET /attendance/history`

One row per **day**, newest first — the question is "did I make it in, and
when", which is a day-shaped answer.

| Query | Default | Notes |
|---|---|---|
| `from` | `to` − 29 days | `YYYY-MM-DD` |
| `to` | today | Clamped to today. A day that has not happened cannot be an absence. |

Maximum window: **92 days**.

```json
{
  "ok": true, "from": "2026-07-27", "to": "2026-07-30",
  "days": [
    { "date": "2026-07-30", "weekday": "Thu", "status": "present", "late": true,
      "first_in": "2026-07-30T16:57:07-04:00", "last_out": null,
      "worked_minutes": 0, "punches": 1, "holiday": null }
  ],
  "totals": { "present_days": 1, "late_days": 1, "leave_days": 0, "absent_days": 3, "worked_minutes": 0 }
}
```

`status` is one of:

| Value | Meaning |
|---|---|
| `present` | Punched in. Wins over every reason not to be there. |
| `leave` | Approved leave, no punch. |
| `holiday` | Company holiday, no punch. |
| `day_off` | Rostered off, no punch. |
| `weekend` | Not a working day for this company. |
| `absent` | A working day nobody planned off, not booked, not shown up for. |

A day never clocked out of reports `worked_minutes: 0` — there is no honest
number for a stretch that was never closed.

**Failures:** `invalid_range` (422) · `range_too_large` (422) · `validation_failed` (422)

### Regularisation requests

Asking for the record to be corrected (A4.13) — a punch that reads wrong, or one
that should be there and isn't.

**Raising only.** There is no approve or reject endpoint and there will not be
one: approving voids a punch and writes a replacement, which is
`manage-attendance` and lives on the web. **A manager has no step here.** Leave
approval is manager-then-HR; a correction is HR's alone, so an approve button in
the app's manager tab would advertise a stage that does not exist.

Attendance is append-only. Nothing on this endpoint changes a punch — a request
is inert until HR decides it, and the correction that follows goes through the
same void-and-re-enter path HR uses by hand, so it carries the same audit trail.

#### `GET /attendance/regularisations`

| Query | Notes |
|---|---|
| `status` | `pending`\|`approved`\|`rejected`\|`cancelled` |
| `page` | 15 per page |

```json
{
  "ok": true,
  "requests": [
    { "id": 7, "type": "out", "work_date": "2026-08-03",
      "requested_at": "2026-08-03T18:00:00-04:00",
      "reason": "Left at 6pm but forgot to press check out",
      "status": "pending", "challenges_a_punch": false,
      "attendance_log_id": null, "can_cancel": true,
      "submitted_at": "2026-08-04T09:12:00-04:00",
      "decision_note": null, "decided_by": null, "decided_at": null }
  ],
  "meta": { "current_page": 1, "last_page": 1, "per_page": 15, "total": 1 },
  "recent_punches": [
    { "id": 109, "type": "in", "status": "late", "work_date": "2026-08-03",
      "scanned_at": "2026-08-03T09:40:00-04:00", "time": "09:40 AM", "office": "Head Office" }
  ]
}
```

`recent_punches` is the last 30, newest first, and is why this list ships them:
`/attendance/history` answers in day-shaped rows and carries no punch ids, so
without it the app cannot name the reading it is disputing. Voided punches are
never included — there is nothing to dispute about a reading already struck out.

`decided_by` is the name recorded **on the row** at the moment of the decision,
so it still reads correctly after that account is deleted.

#### `POST /attendance/regularisations`

| Field | Type | Notes |
|---|---|---|
| `attendance_log_id` | integer, optional | The punch being disputed. Omit to report one that is missing. |
| `type` | `in`\|`out`, required | What the corrected punch should be. |
| `requested_at` | date-time, required | The time it should read. |
| `reason` | string, required, 5–500 | |

Three rules beyond the shape, all shared with the portal form:

- **No future times.** A correction to a moment that has not happened is refused
  on `requested_at`.
- **Only your own punches.** An `attendance_log_id` from somebody else's record
  is refused on `attendance_log_id` — it is checked, never trusted.
- **One open request per problem** — per punch, or per date-and-type where there
  is no punch. Refused on `reason`. Without it a double submit produces two
  approvals and two corrections for one problem.

Returns **201** with the created request.

#### `POST /attendance/regularisations/{id}/cancel`

Withdraw one that is still pending. A decided request has already moved
attendance; "cancelling" it afterwards would leave the correction standing with
nothing on record explaining it, so it is refused on `status`.

**Failures for all three:** `validation_failed` (422, including the three rules
above) · `forbidden` (403, somebody else's request, or no employee record) ·
`not_found` (404) · `too_many_requests` (429, the `write` limiter)

---

## 6. Leave

### `GET /leave/balances`

Every type still open for booking, with this employee's balance. Render the
apply form from this list — inactive types keep their history but are not
offered.

| Query | Default |
|---|---|
| `year` | current year |

```json
{
  "ok": true, "year": 2026,
  "balances": [
    { "leave_type_id": 4, "name": "Annual Leave", "code": "AL", "color": "#4f46e5",
      "is_paid": true, "allow_half_day": true, "requires_approval": true,
      "entitled_days": 20, "carried_forward": 0, "used_days": 0,
      "available_days": 20, "is_capped": true }
  ]
}
```

**`is_capped` matters.** A type granting zero days is *uncapped*, not exhausted —
that is how unpaid leave is set up. Do not grey it out or show "0 days left".

### `POST /leave/requests`

| Field | Type | Notes |
|---|---|---|
| `leave_type_id` | int, required | Must be an active type belonging to the employee's company. |
| `start_date` | `YYYY-MM-DD`, required | |
| `end_date` | `YYYY-MM-DD`, required | On or after `start_date`; at most two years ahead. |
| `is_half_day` | bool, optional | Only for a type allowing it, and only when start and end are the same date. |
| `half_day_period` | `first_half`\|`second_half`, optional | |
| `reason` | string, optional, ≤1000 | |

**The server counts the days.** Weekends and company holidays inside the range
are free, so Friday-to-Monday over a two-day weekend costs **2**, not 4. Do not
compute and send a day count — the response tells you what it actually cost.

`201` on success:

```json
{
  "ok": true,
  "request": { "id": 8, "leave_type": "Annual Leave", "start_date": "2026-09-04",
               "end_date": "2026-09-07", "days": 2, "is_half_day": false,
               "status": "pending", "stage": "Awaiting HR", "can_cancel": true,
               "submitted_at": "2026-07-30T21:09:42+00:00" },
  "message": "Leave request submitted. You will be notified once it is reviewed."
}
```

A type that needs no approval comes back `approved` immediately.

Business-rule refusals arrive as `validation_failed` against the field that
caused them, so they can be shown on the form:

- `start_date` — overlaps leave you already have, or the range is entirely
  weekend and holiday so there is nothing to book
- `leave_type_id` — not available, or more days than the balance allows
- `end_date` — before the start, or more than two years ahead
- `is_half_day` — not allowed for this type, or spanning two dates

### `GET /leave/requests`

The caller's own requests, newest first, 15 per page.

| Query | Notes |
|---|---|
| `status` | `pending`, `approved`, `rejected`, `cancelled` |
| `year` | Filters on the start date |
| `page` | |

```json
{ "ok": true, "requests": [ … ], "meta": { "current_page": 1, "last_page": 1, "per_page": 15, "total": 2 } }
```

`stage` is what to show a person chasing a decision: `Awaiting Manager`,
`Awaiting HR`, or the final status. "Pending" alone does not say who to ask. An
employee with no manager set skips the manager step entirely.

### `GET /leave/requests/{id}`

The same fields plus `half_day_period`, `reason`, `manager_note`,
`manager_approved_by`, `manager_approved_at`, `decision_note`, `decided_by`,
`decided_at`.

`403` for anybody else's request.

### `POST /leave/requests/{id}/cancel`

Withdraws it. Approved leave gives its days back; pending leave never spent any.

Only possible while `can_cancel` is true — pending, or approved and not yet
started. Leave already under way is HR's to unwind.

**Failures:** `validation_failed` (422, with `errors.status`) · `forbidden` (403)

---

## 7. Manager approvals

Requires the `approve-leave` permission **and** the request must belong to one
of the caller's own direct reports. The permission opens the door; it is not
access to anyone else's team. Show this section only when
`user.permissions` contains `approve-leave`.

### `GET /leave/approvals`

What this manager still has to act on, soonest first. A request already passed
up to HR leaves the inbox.

```json
{
  "ok": true, "pending_count": 1,
  "pending": [
    { "id": 6, "employee": "Michael Brown", "employee_id": 3,
      "leave_type": "Sick Leave", "start_date": "2026-09-21", "end_date": "2026-09-22",
      "days": 2, "is_half_day": false, "reason": null,
      "submitted_at": "2026-07-30T17:08:41+00:00",
      "clashes": [ { "employee": "Sam Fox", "start_date": "2026-09-21", "end_date": "2026-09-23" } ] }
  ]
}
```

`clashes` is who else on the team is already off over the same dates. Show it
before the approve button, not after.

### `POST /leave/approvals/{id}/approve`

Passes the request to HR for the final decision. **Nothing is deducted here** —
the manager step commits no days.

| Field | Type |
|---|---|
| `manager_note` | string, optional, ≤1000 |

### `POST /leave/approvals/{id}/reject`

| Field | Type | Notes |
|---|---|---|
| `decision_note` | string, **required**, ≤1000 | The employee sees this. |

**Failures for both:** `forbidden` (403, outside the team or the manager's own
request) · `validation_failed` (422, already decided or already passed up)

---

## 7b. Team

### `GET /team/attendance`

Who on **your own team** is in today. Requires `approve-leave`, and answers only
for your direct reports — a team lead is not HR, and the web dashboard is where
company-wide attendance lives.

| Query | Default | Notes |
|---|---|---|
| `date` | today | `YYYY-MM-DD`. A future date is refused: nobody can be absent for a day that has not happened. |

```json
{
  "ok": true, "date": "2026-08-01", "timezone": "America/New_York",
  "summary": { "total": 5, "present": 3, "in_now": 2, "late": 1,
               "on_leave": 1, "absent": 1, "off": 0 },
  "team": [
    { "employee_id": 2, "name": "Emily Johnson", "employee_code": "EMP-0002",
      "status": "present", "late": true,
      "first_in": "09:14 AM", "last_out": null,
      "is_clocked_in": true, "worked_minutes": 214,
      "shift": { "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00" } }
  ]
}
```

`status` uses the same vocabulary as `/attendance/history` — `present`, `leave`,
`holiday`, `day_off`, `weekend`, `absent` — and is computed by the same code, so
a manager and the person they manage never see two different words for one day.

**`in_now` is not `present`.** Somebody who worked this morning and went home is
present for the day but not on the floor. A manager asking "who is here" wants
the first number; a manager asking "who turned up" wants the second.

A manager with nobody reporting to them gets `team: []` and a zeroed summary,
not an error.

**Failures:** `invalid_range` (422, a future date) · `forbidden` (403, no
`approve-leave` permission) · `validation_failed` (422)

---

### `GET /team/roster`

Your team's **published** roster over a stretch of days. Same gate and same team
as `/team/attendance`.

| Query | Default | Notes |
|---|---|---|
| `from` | today | `YYYY-MM-DD`, the first day of the window. |
| `days` | 7 | 1–31. |

```json
{
  "ok": true, "from": "2026-08-03", "to": "2026-08-09",
  "timezone": "America/New_York",
  "team": [
    { "employee_id": 2, "name": "Emily Johnson", "employee_code": "EMP-0002",
      "schedule": [
        { "date": "2026-08-03", "status": "working", "holiday": null,
          "shift": { "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00" },
          "is_rostered": true },
        { "date": "2026-08-04", "status": "leave", "holiday": null,
          "shift": null, "is_rostered": false }
      ] }
  ]
}
```

Returned **employee-major**: a manager reads down a person to see their week.
The across-a-day view is what `/team/attendance` already answers.

`status` is one of `working`, `leave`, `holiday`, `day_off`, `weekend`.

**Published only**, exactly like the employee's own `/schedule`. A manager
seeing draft shifts their team cannot see would tell somebody to come in on a
day still being planned — which is the whole reason the roster has a draft and
a published state.

**Leave outranks the roster.** Somebody rostered on a day they later booked off
comes back as `leave` with no shift, because showing the shift would have a
manager expecting them.

`is_rostered` distinguishes a day explicitly planned on the roster from one
falling back to the person's standing shift. Both are `working`; only the first
was deliberately placed.

A manager with nobody reporting to them gets `team: []`, not an error.

**Failures:** `forbidden` (403, no `approve-leave` permission) ·
`validation_failed` (422)

---

## 8. Schedule

### `GET /schedule`

| Query | Default |
|---|---|
| `from` | today |
| `to` | `from` + 13 days |

Maximum window: **92 days**. Oldest first — a schedule is read forwards.

```json
{
  "ok": true, "from": "2026-09-04", "to": "2026-09-08",
  "standing_shift": { "id": 1, "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00" },
  "days": [
    { "date": "2026-09-04", "weekday": "Fri",
      "shift": { "id": 1, "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00", "color": "#22c55e" },
      "is_day_off": false, "is_rostered": false, "is_working_day": true,
      "holiday": null, "leave": null },
    { "date": "2026-09-05", "weekday": "Sat", "shift": null,
      "is_day_off": false, "is_rostered": false, "is_working_day": false,
      "holiday": null, "leave": null }
  ]
}
```

- **Only published roster days are visible.** A roster still being planned falls
  back to the standing shift as though it did not exist — staff watching draft
  days move around is the problem publishing exists to prevent.
- `is_rostered` distinguishes a planned day from the standing shift filling in.
- `is_day_off` is a *planned* day with no hours, which is not the same as a day
  nobody planned.
- `shift` is `null` on weekends and holidays unless somebody was explicitly
  rostered on — the standing shift does not leak onto days the company does not
  work.
- `leave` names the type on days covered by **approved** leave only. A pending
  request is not time off yet.

---

## 9. Profile

### `GET /profile`

```json
{
  "ok": true,
  "account": { "id": 3, "name": "James Smith", "email": "…", "phone": null, "avatar": null, "roles": ["employee", "manager"] },
  "employee": { "id": 1, "employee_code": "EMP-0001", "full_name": "James Smith",
                "email": "…", "phone": null, "date_of_birth": null, "gender": "male",
                "hire_date": "2023-09-17", "status": "active", "work_mode": "office",
                "department": "Engineering", "designation": "Software Engineer",
                "office": "Head Office", "manager": null, "is_manager": true },
  "shift": { "id": 1, "name": "Morning Shift", "start_time": "09:00:00", "end_time": "17:00:00", "working_hours": "7h" },
  "company": { "id": 1, "name": "Acme", "timezone": "America/New_York", "currency": "EUR" }
}
```

`shift` here is the **standing** shift — "my usual hours". What applies on a
particular day comes from `/schedule` or `/attendance/today`.

### `PUT /profile`

Contact details only.

| Field | Type |
|---|---|
| `name` | string, required, ≤150 |
| `email` | string, required, unique |
| `phone` | string, optional, ≤30 |

Department, manager, shift, hire date and employee code are HR's to set. Anything
else posted here is ignored, not obeyed.

### `PUT /profile/password`

| Field | Type |
|---|---|
| `current_password` | string, required |
| `password` | string, required, ≥8, confirmed |
| `password_confirmation` | string, required |

The current password is required **even though the caller holds a valid token** —
a phone left unlocked for a minute should not be enough to take the account over.

On success every **other** device is signed out; the one making the change stays
in. Warn the user before they submit.

```json
{ "ok": true, "message": "Password changed.", "other_devices_signed_out": 1 }
```

**Failures:** `wrong_password` (422) · `validation_failed` (422)

---

## 10. Documents

The caller's own shelf of the document vault (A3.8) — contracts, ID scans,
right-to-work papers, certificates. **Read-only.** Filing is HR's job and sits
behind `manage-employees`; there is no upload, edit or delete here.

Neither route takes an employee id. There is nothing to tamper with and nothing
to forget in a `where`: the scope is the token's own employee record.

### `GET /documents`

```json
{
  "ok": true,
  "documents": [
    { "id": 12, "type": "right_to_work", "type_label": "Right to Work / Visa",
      "title": "Work visa", "original_name": "visa-2026.pdf",
      "mime_type": "application/pdf", "size_bytes": 184320, "size_label": "180 KB",
      "issued_on": "2023-04-01", "expires_on": "2026-10-01", "expiry_state": "soon" }
  ],
  "expiring_soon": 1,
  "expired": 0
}
```

Ordered **soonest to expire first**, undated last — the list is read to find what
needs renewing. `expiring_soon` and `expired` are counted server-side so a tab
badge cannot disagree with the list it opens.

| `expiry_state` | Meaning |
|---|---|
| `none` | No expiry date. A contract, usually. |
| `valid` | Expires, but not within 30 days. |
| `soon` | Expires within 30 days. HR is being chased about it too. |
| `expired` | The date has passed. |

`notes` and the uploader are **not** returned. Notes is where HR records why a
document is being chased; the employee is its subject, not its audience.

### `GET /documents/{id}`

The file itself — **the one endpoint that does not answer with `ok`.** On success
the body is the document, with `Content-Disposition: attachment` and the name it
was uploaded under. Failures are JSON as everywhere else, so parse only when the
status is not 2xx.

Streamed through the API, never a public URL: a guessable link that hands out a
passport scan without a token is the one mistake in this feature that would
matter.

**Failures:** `not_found` (404, no such document **or** somebody else's — the two
are deliberately indistinguishable) · `file_missing` (404, the row outlived its
file, which means a database was restored without
`storage/app/employee-documents/`) · `forbidden` (403, no employee record)

### Not here: payslips

B3.7 is written as "payslip / documents". There is no payslip, because there is
no payroll module — A7.14 exports hours for whatever runs payroll elsewhere. If
payslips are ever filed into the vault they arrive through this endpoint with no
change; nothing here needs to know they are special.

---

## 11. Directory

### `GET /directory`

Who else works here (B3.8) — **the only endpoint that answers about other
people**, and for that reason the one that carries the least.

| Query | Notes |
|---|---|
| `q` | Matches first name, last name or employee code |
| `department_id` | Narrows the same company-wide list |
| `office_id` | As above |
| `page` | 30 per page |

Active staff of the caller's own company, ordered by name. Leavers are not
listed: their record survives for the audit trail, not for finding somebody who
still works here.

```json
{
  "ok": true,
  "people": [
    { "id": 4, "employee_code": "E2", "full_name": "Bo Ray",
      "designation": "Cleaner", "department": "Ops", "office": "Head Office",
      "work_mode": "office", "photo_url": null }
  ],
  "meta": { "current_page": 1, "last_page": 1, "per_page": 30, "total": 1 },
  "shows_contact_details": false
}
```

**Contact details are a company policy, off by default.** With
`directory_show_contact_details` switched on at `/settings/policies`, each person
also carries `email` and `phone`:

```json
{ "…": "…", "email": "bo@acme.test", "phone": "+1 555 0134" }
```

It defaults off because there is a single phone column on an employee record,
and for a workforce with no desk lines it holds personal mobiles — turning that
on by default would publish every one of them to every colleague on an app
update, which is not a disclosure that can be withdrawn afterwards.

Read `shows_contact_details` rather than inferring from absent keys: the app has
to tell "this company does not share contact details" from "this person has none
on file", so it can hide a call button instead of showing a dead one.

**Never returned, at any policy setting:** date of birth, home address, national
id, blood group, personal email, emergency contact, hire date, employment
status, documents, attendance, leave — and the reporting line. Those are behind
`manage-employees`, and a *manager* does not see them for their own team; a
colleague cannot see more than a manager.

---

## 12. Push devices

Registration only. Nothing is delivered yet — notifications are Phase 5. The app
can register from its first release so it does not need an update when they land.

### `POST /devices`

Call on launch and again whenever the OS issues a new token. Safe to repeat: the
same token re-registers rather than duplicating.

| Field | Type | Notes |
|---|---|---|
| `token` | string, required, ≤255 | The FCM/APNs registration token. |
| `platform` | `android`\|`ios`\|`web`, required | |
| `device_name` | string, optional, ≤100 | |
| `app_version` | string, optional, ≤30 | |

A token that already exists under **another** account is moved to the caller. The
token belongs to the installation, not the person — a handed-on phone must stop
receiving the previous owner's approvals.

### `GET /devices`

Handsets registered to the caller. The push token itself is **never** returned:
it is a credential for sending to that handset.

### `DELETE /devices`

| Field | Type |
|---|---|
| `token` | string, required |

Removing an unknown token is not an error — `removed` is simply `0`. Scoped to
the caller, so a token cannot be used to silence somebody else's phone.

---

## 13. Client checklist

1. `GET /ping` before showing login, to distinguish "server down" from "wrong
   password".
2. Store the token in the keychain. Send it as `Authorization: Bearer <token>`.
3. On **any** `401`, discard the token and return to login — it has been revoked
   or the password changed.
4. Register for push after login; unregister on logout by sending `push_token`.
5. Render every timestamp in `user.company.timezone`, not the handset's.
6. Never compute leave day counts or punch direction locally. Ask the server.
7. Show `stage`, not `status`, on a pending leave request.
8. Respect `can_check` and treat `duplicate_scan` as success.
9. On `429`, honour `Retry-After` and back off rather than retrying immediately.
