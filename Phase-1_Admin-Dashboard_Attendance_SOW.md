# Phase 1 — Admin Web Dashboard (Attendance Focus)
### Scope of Work & Technical Plan

**Project:** HR & Attendance Management Application
**Prepared For:** Alfonzo
**Phase:** 1 of N — *Admin Web Dashboard*
**Primary Module:** Attendance Management
**Stack:** Laravel 11 (PHP) + MySQL + Blade/Bootstrap
**Date:** 2026-07-16

---

## ✅ BUILD STATUS — Completed 2026-07-17
Phase 1 admin dashboard **built and verified** in `hrms/` (Laravel 12 + MySQL + SmartHR template).
- Auth (admin-only gate), dashboard, full **Attendance module** (rotating-QR kiosk + PWA scanner +
  validation + logs + reports), Employees (CRUD + CSV import), Departments, Designations, Company,
  Offices, Roles & Permissions (Spatie), Settings, Profile — **all pages return HTTP 200**.
- QR flow tested end-to-end: token rotation, clock in/out recording, and rejection of forged/expired codes.
- Run: start MySQL (XAMPP), `cd hrms && php artisan serve`, open http://127.0.0.1:8000, login `admin@hrms.test` / `password`.

## 1. Purpose of This Phase

This document defines the scope for **Phase 1** of the larger HR & Attendance project
described in the master Scope of Work. Rather than building the Android/iOS apps first,
Phase 1 delivers a **web-based Admin Dashboard** with the **Attendance module** as the
core focus.

This approach front-loads the parts every later phase depends on:

- A **database schema** for companies, employees, and attendance.
- A **backend API** (Laravel) that the future mobile apps will reuse.
- A working **admin panel** to configure the system and view/verify attendance data.

> **Answer to "is it possible?" — Yes.** A Laravel + MySQL web dashboard that fetches and
> displays attendance data is a standard, well-supported build. Because Laravel ships with
> an API layer (Sanctum), the same backend later serves the mobile apps with no rewrite.

---

## 2. Phase 1 Goals

1. Secure **admin login** with role-based access (Admin / HR).
2. **Company & Department** setup.
3. **Employee management** (CRUD + bulk import).
4. **Attendance module** — the deep focus: QR-based clock in/out, logs, history, scoring, summaries.
5. **Attendance reporting** with PDF + Excel export.
6. A clean **dashboard** that fetches and visualizes attendance data in real time.

**Out of Phase 1 (future phases):** Android app, iOS app, AI Assistant, push notifications,
leave management UI, shift/schedule engine. *(Data model may reserve space for these, but no UI/logic is built now.)*

### ✅ Confirmed Decisions (2026-07-16)
- **Roles:** **Admin-only** *login* for Phase 1. HR is deferred but its role/permission structure is built now (see below).
- **Build start:** On hold until the **client shares a frontend template**. The dashboard UI will be
  built on top of that template rather than a plain Bootstrap scaffold.
- **First milestone:** M1 (Laravel install + auth + base layout) begins *after* the template is received.

### ✅ Role & Permission Structure (RBAC) — Confirmed 2026-07-17
- **Package:** **Spatie `laravel-permission`** — full roles + permissions RBAC (industry standard).
- **Roles seeded now:** `admin`, `hr`, `employee`.
- **Granular permissions** (e.g. `manage-employees`, `view-attendance`, `export-reports`,
  `manage-departments`, `configure-qr`) assigned to each role via a seeder.
- **Admin** = all permissions. **HR** = HR subset (seeded but login-disabled in Phase 1).
  **Employee** = reserved for the mobile phase.
- **Login gate:** Admin-only for Phase 1. Enabling HR later = flip the gate + confirm the HR permission
  set — **no schema change, no refactor.**
- Permission-based checks used throughout: `@can(...)` in Blade, `authorize(...)` in controllers,
  `permission:` middleware on routes.

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────┐
│              Admin Web Dashboard             │
│         (Laravel Blade + Bootstrap)          │
└───────────────────┬─────────────────────────┘
                    │  HTTP (web routes)
┌───────────────────▼─────────────────────────┐
│              Laravel Application             │
│  Controllers · Services · Eloquent Models    │
│  Auth (session) + API (Sanctum, future)      │
└───────────────────┬─────────────────────────┘
                    │
┌───────────────────▼─────────────────────────┐
│                   MySQL DB                    │
│   companies · departments · employees ·       │
│   attendance_logs · qr_tokens · users         │
└──────────────────────────────────────────────┘
                    ▲
                    │  REST API (Sanctum) — reserved
        ┌───────────┴───────────┐
        │  Future: Android / iOS │   ← Phase 2+
        └────────────────────────┘
```

**Key idea:** the mobile apps in the master SOW will hit the **same Laravel backend** via
API routes. Phase 1 builds the web (session) side; the API side is scaffolded but expanded later.

---

## 4. Attendance Module — Detailed Scope (Primary Focus)

This is the heart of Phase 1. Feature breakdown from the master SOW §3.3:

| Feature | Phase 1 Implementation |
|---|---|
| **QR Clock In / Out** | Employee scans an office QR; server records timestamp + resolves who/where. |
| **Dynamic rotating QR** | Office QR encodes a short-lived token that rotates every N seconds — prevents screenshot/replay abuse. |
| **QR validation** | Server checks token freshness, office match, and that the employee is assigned there. |
| **Office-specific QR** | Each office/branch generates its own QR stream. |
| **Timestamp recording** | Server-side time (not device time) stored for every event. |
| **Attendance logs** | Full event table: employee, office, type (in/out), timestamp, status. |
| **Attendance scoring** | Derived score per employee (on-time %, late count, absences). |
| **Attendance summaries** | Daily / weekly / monthly rollups per employee & department. |
| **Attendance history** | Filterable, paginated history view in the dashboard. |

### ✅ QR Attendance Model — Confirmed 2026-07-17
- **Model A: Office displays rotating QR → employee scans.** (Matches the master SOW exactly.)
- **Phase 1 delivery without native app:**
  - **Web Kiosk** — a browser screen (tablet/monitor at the office entrance) shows the
    **dynamic rotating QR**, refreshing every ~15–30s. Built into the admin dashboard.
  - **PWA Scanner** — employees scan via a mobile web page (camera in the browser, no install).
    Same validation endpoint the future native apps will call — zero rework in Phase 2.
- **Server authority:** token freshness + office match + employee assignment validated server-side;
  **server timestamp** recorded (device clock never trusted).

### 4.1 Dynamic Rotating QR — How It Works
1. Each office has a secret. The server generates a token = `HMAC(office_secret, current_time_window)`.
2. The office display (or admin screen) shows a QR encoding `{office_id, token}` and refreshes every ~15–30s.
3. Employee scans → app/browser sends `{employee_id, office_id, token}` to the API.
4. Server recomputes the valid token for the current + previous time window and compares.
5. Valid → record clock event. Invalid/expired → reject.

*(Phase 1 builds the generation + validation on the server and an admin QR-display screen. The employee-side scanner is minimal/web for testing; the polished mobile scanner is Phase 2.)*

---

## 5. Supporting Modules (Core Admin)

| Module | Phase 1 Features |
|---|---|
| **Auth & Roles** | Login, logout, password change, role-based access (Admin, HR). |
| **Company** | Create/edit company profile; manage offices/branches. |
| **Departments** | Create / edit / assign departments; department-based filtering in reports. |
| **Employees** | CRUD, assign to department + office, credential creation, **CSV/Excel bulk import**. |
| **Reporting** | Attendance reports, late reports, outlier reports, department reports; **PDF + Excel export**. |
| **Dashboard** | Live tiles: present today, late today, absent, headcount; recent attendance feed; charts. |

---

## 6. Proposed Database Schema (Phase 1)

```
users            — id, name, email, password, role (admin/hr), company_id
companies        — id, name, logo, address, settings
offices          — id, company_id, name, address, qr_secret
departments      — id, company_id, name
employees        — id, company_id, department_id, office_id, code, name,
                   email, phone, job_title, status, hire_date
qr_tokens        — id, office_id, token, window_start, expires_at   (or computed on the fly)
attendance_logs  — id, employee_id, office_id, type (in/out),
                   scanned_at, source, status (ontime/late/invalid)
attendance_scores— id, employee_id, period, ontime_pct, late_count, absent_count
```

*Reserved for later phases (not built now): `leave_requests`, `shifts`, `schedules`, `notifications`.*

---

## 7. Deliverables (Phase 1)

- Laravel 11 project configured for XAMPP (Apache + MySQL).
- MySQL migrations + seeders for the schema above.
- Admin dashboard UI (Blade + Bootstrap): auth, company, departments, employees, attendance, reports.
- Attendance engine: QR generation, rotation, validation, logging, scoring, summaries.
- CSV/Excel employee import.
- PDF + Excel report export.
- Reserved API scaffolding (Sanctum) for future mobile apps.
- Setup/README documentation.

---

## 8. Suggested Build Order (Milestones)

1. **M1 — Foundation:** Laravel install on XAMPP, DB config, auth, base layout.
2. **M2 — Company & Employees:** Company/office/department setup, employee CRUD + import.
3. **M3 — Attendance Core:** Schema, QR generation + rotation, validation, clock in/out logging.
4. **M4 — Dashboard & History:** Live dashboard tiles, attendance history/filtering.
5. **M5 — Scoring & Reports:** Scoring logic, summaries, PDF/Excel export.
6. **M6 — Polish & Handoff:** Testing, seed demo data, documentation.

---

## 9. What I Need From You

- Confirmation of the **Phase 1 scope** above (add/remove anything).
- Any **branding** (company name, logo, colors) — optional, can use placeholders.
- Whether the **QR display** should live on the admin dashboard, a per-office kiosk screen, or both.
- ~~Confirm **roles** for Phase 1~~ → **Resolved: Admin-only for Phase 1.**
- **➡️ Pending: share the frontend template** so the dashboard is built on top of it.

---

## 10. Open Questions / Recommendations

- **Rotating-QR window:** recommend a 15–30s rotation with ±1 window tolerance for scan lag.
- **Server time authority:** all timestamps server-side to prevent device-clock spoofing.
- **Late/absent rules:** these depend on shifts/schedules (a later module). For Phase 1, I suggest a
  simple configurable "work start time" per office so scoring can function before the full schedule engine exists.

---

*End of Phase 1 Scope — awaiting your confirmation to proceed to build.*
