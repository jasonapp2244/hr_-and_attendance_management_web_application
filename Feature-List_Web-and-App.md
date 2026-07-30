# HR & Attendance Management — Master Feature List (Web + Mobile App)

**Project:** HR & Attendance Management Application
**Prepared for:** Alfonzo
**Stack:** Laravel 12 + MySQL · Blade/Bootstrap (SmartHR) web · Android/iOS app on the same Laravel API
**Date:** 2026-07-29
**Purpose:** the complete feature set for the grown product — what is already delivered, and what is planned — split across the **Web Dashboard**, the **Mobile App**, and the **Shared Backend/API**.

**Legend:** ✅ Built · 🟡 Partial · ⬜ Planned

---

# PART A — WEB DASHBOARD (Admin / HR)

## A1. Authentication & Access Control
| # | Feature | Status |
|---|---|---|
| A1.1 | Secure login / logout (Laravel session) | ✅ |
| A1.2 | Role-based access — Admin, HR, Employee (Spatie RBAC) | ✅ |
| A1.3 | Granular permissions per role (16 seeded) | ✅ |
| A1.4 | Roles & permissions editor UI | ✅ |
| A1.5 | Profile page + change password | ✅ |
| A1.6 | Password reset via email ("forgot password") | ⬜ |
| A1.7 | Two-factor authentication (2FA) for Admin/HR | ⬜ |
| A1.8 | Login activity & audit trail (who did what, when) | ⬜ |
| A1.9 | Session timeout + forced re-login policy | ⬜ |

## A2. Company & Organization Setup
| # | Feature | Status |
|---|---|---|
| A2.1 | Company profile (name, logo, address, timezone) | ✅ |
| A2.2 | Office / branch management | ✅ |
| A2.3 | Office GPS coordinates + geofence radius | 🟡 stored, not enforced |
| A2.4 | Departments — CRUD & assignment | ✅ |
| A2.5 | Designations / job titles | ✅ |
| A2.6 | General settings page | ✅ |
| A2.7 | Company holiday calendar | ✅ |
| A2.8 | Weekend / working-days configuration per office | 🟡 company-level setting; no UI |
| A2.9 | Attendance & leave policy rules engine | ⬜ |
| A2.10 | Multi-company (SaaS tenancy) support | ⬜ |

## A3. Employee Management
| # | Feature | Status |
|---|---|---|
| A3.1 | Employee CRUD + deactivate | ✅ |
| A3.2 | Assign department / office / designation | ✅ |
| A3.3 | Employment details (code, job title, hire date, status) | ✅ |
| A3.4 | Work mode — office / WFH / hybrid | ✅ |
| A3.5 | Login credential creation & reset | ✅ |
| A3.6 | CSV / Excel bulk import | ✅ |
| A3.7 | Employee profile photo upload | ⬜ |
| A3.8 | Document vault (contract, ID, certificates) with expiry alerts | ⬜ |
| A3.9 | Emergency contact & personal details | ⬜ |
| A3.10 | Org chart / reporting-manager hierarchy | 🟡 reporting line set + cycle-safe; no org chart view |
| A3.11 | Employee export (CSV / Excel) | ⬜ |
| A3.12 | Onboarding & offboarding checklists | ⬜ |

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
| A4.11 | Weekly rollup summaries | ⬜ |
| A4.12 | Manual attendance entry / correction by HR (with audit reason) | ⬜ |
| A4.13 | Attendance regularisation requests (employee raises, HR approves) | ⬜ |
| A4.14 | Overtime calculation & tracking | ⬜ |
| A4.15 | Break in / break out punches | ⬜ |
| A4.16 | Geofence enforcement (block punch outside office radius) | ⬜ optional |
| A4.17 | Auto-absent marking for missed days (scheduled job) | ⬜ |
| A4.18 | Missing-checkout auto-close policy | ⬜ |
| A4.19 | Live "who's in right now" board | ⬜ |

## A5. Shift & Schedule Management
| # | Feature | Status |
|---|---|---|
| A5.1 | Shift creation (start, end, grace period) | ✅ |
| A5.2 | Shift assignment per department | ✅ |
| A5.3 | Weekly roster view | ✅ leave, holidays and company weekend aware |
| A5.4 | Shift-driven attendance validation | ✅ |
| A5.5 | Per-employee shift override | ✅ |
| A5.6 | Rotating / night shift patterns | ✅ |
| A5.7 | Break rule configuration | 🟡 unpaid break deducted from paid hours; no break punches |
| A5.8 | Roster drag-and-drop planner + publish to staff | 🟡 grid planner + draft/publish; no drag-and-drop |
| A5.9 | Shift swap requests between employees | ⬜ |

## A6. Leave Management
| # | Feature | Status |
|---|---|---|
| A6.1 | Leave types (annual, sick, unpaid, casual…) | ✅ |
| A6.2 | Leave request submission | ✅ |
| A6.3 | Multi-step approval workflow (manager → HR) | ✅ |
| A6.4 | Leave balance tracking & accrual rules | 🟡 tracked, provisioned & HR-adjustable; no automatic accrual |
| A6.5 | Leave history & status management | ✅ |
| A6.6 | Company leave policy configuration | 🟡 types + holidays + weekend config; no rules engine |
| A6.7 | Team leave calendar / conflict detection | 🟡 conflicts flagged to the manager; no calendar view |
| A6.8 | Leave ↔ attendance integration (leave day ≠ absent) | ✅ |
| A6.9 | Carry-forward & year-end processing | ⬜ |

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
| A7.10 | Leave reports | ⬜ |
| A7.11 | Overtime reports | ⬜ |
| A7.12 | Scheduled report email delivery (daily/weekly/monthly) | ⬜ |
| A7.13 | Custom report builder (pick columns + filters) | ⬜ |
| A7.14 | Payroll-ready export (hours worked per employee per period) | ⬜ |

## A8. Dashboard
| # | Feature | Status |
|---|---|---|
| A8.1 | Live tiles — present, late, absent, headcount | ✅ |
| A8.2 | Recent attendance feed | ✅ |
| A8.3 | Charts / visualisation | ✅ |
| A8.4 | Role-specific dashboards (Admin vs HR view) | ⬜ |
| A8.5 | Configurable widgets | ⬜ |
| A8.6 | Trend comparison (this week vs last week) | ⬜ |

## A9. Notifications (Web)
| # | Feature | Status |
|---|---|---|
| A9.1 | In-app notification bell + centre | ⬜ |
| A9.2 | Email notifications | ⬜ |
| A9.3 | Late-arrival alert to HR | ⬜ |
| A9.4 | Leave request / approval / rejection alerts | ⬜ |
| A9.5 | Schedule update alerts | ⬜ |
| A9.6 | Missing-checkout reminder | ⬜ |

---

# PART B — MOBILE APP (Android + iOS, Employee-facing)

> Every screen below consumes the shared Laravel API (Part C). Nothing here can start until **C1 (Sanctum API)** exists.

## B1. Onboarding & Auth
| # | Feature | Status |
|---|---|---|
| B1.1 | Splash + branded onboarding screens | ⬜ |
| B1.2 | Login with email + password (Sanctum token) | ⬜ |
| B1.3 | Biometric unlock (fingerprint / Face ID) | ⬜ |
| B1.4 | Forgot password flow | ⬜ |
| B1.5 | Stay-logged-in / secure token refresh | ⬜ |
| B1.6 | Device registration & binding (one account ↔ trusted device) | ⬜ |
| B1.7 | Logout / remote session revoke | ⬜ |

## B2. Attendance (app core)
| # | Feature | Status |
|---|---|---|
| B2.1 | Big one-tap Check In / Check Out button | ⬜ |
| B2.2 | Live status card (checked in at 09:02, hours so far) | ⬜ |
| B2.3 | GPS capture at punch | ⬜ |
| B2.4 | Offline punch queue → auto-sync when back online | ⬜ |
| B2.5 | Geofence-aware punch (warn or block outside office) | ⬜ optional |
| B2.6 | Break in / break out | ⬜ |
| B2.7 | Mock-location / rooted-device detection | ⬜ |
| B2.8 | Home-screen widget / quick action for fast punching | ⬜ |

## B3. Employee Self-Service
| # | Feature | Status |
|---|---|---|
| B3.1 | View own profile & employment details | ⬜ |
| B3.2 | Edit permitted fields (phone, address, emergency contact) | ⬜ |
| B3.3 | Change password | ⬜ |
| B3.4 | Attendance history with monthly calendar view | ⬜ |
| B3.5 | Personal attendance score / on-time streak | ⬜ |
| B3.6 | View assigned shift & upcoming roster | ⬜ |
| B3.7 | Download own payslip / documents | ⬜ |
| B3.8 | Company directory (colleagues, departments) | ⬜ |

## B4. Leave (app)
| # | Feature | Status |
|---|---|---|
| B4.1 | Apply for leave (type, dates, reason, attachment) | ⬜ |
| B4.2 | View leave balances | ⬜ |
| B4.3 | Track request status & history | ⬜ |
| B4.4 | Cancel / withdraw a pending request | ⬜ |
| B4.5 | Manager approval inbox (approve/reject in-app) | ⬜ |
| B4.6 | Team leave calendar | ⬜ |

## B5. Push Notifications (FCM / APNs)
| # | Feature | Status |
|---|---|---|
| B5.1 | Clock-in reminder at shift start | ⬜ |
| B5.2 | Clock-out reminder at shift end | ⬜ |
| B5.3 | Leave approved / rejected | ⬜ |
| B5.4 | Schedule / roster updated | ⬜ |
| B5.5 | HR announcements & broadcasts | ⬜ |
| B5.6 | In-app notification centre | ⬜ |

## B6. App Experience
| # | Feature | Status |
|---|---|---|
| B6.1 | Dark mode | ⬜ |
| B6.2 | Multi-language support | ⬜ |
| B6.3 | Offline-first cache of profile, history, roster | ⬜ |
| B6.4 | Accessibility (font scaling, contrast) | ⬜ |
| B6.5 | Crash & analytics reporting | ⬜ |
| B6.6 | Force-update / maintenance-mode gate | ⬜ |

## B7. Manager Mode (optional in-app role)
| # | Feature | Status |
|---|---|---|
| B7.1 | Team attendance today | ⬜ |
| B7.2 | Approve leave / regularisation from phone | ⬜ |
| B7.3 | Team roster view | ⬜ |

---

# PART C — SHARED BACKEND & API

| # | Feature | Status |
|---|---|---|
| C1.1 | **Laravel Sanctum token auth + `routes/api.php`** ⛔ *blocker for all of Part B* | ⬜ |
| C1.2 | `/auth/login`, `/auth/logout`, `/auth/me` | ⬜ |
| C1.3 | `/attendance/check`, `/attendance/history`, `/attendance/today` | ⬜ |
| C1.4 | `/leave/*` endpoints | ⬜ |
| C1.5 | `/schedule`, `/profile` endpoints | ⬜ |
| C1.6 | Device token registration for push | ⬜ |
| C1.7 | API rate limiting + throttling | ⬜ |
| C1.8 | Consistent JSON error format + API versioning (`/api/v1`) | ⬜ |
| C1.9 | Queue worker + scheduler (reminders, auto-absent, reports) | ⬜ |
| C1.10 | Immutable audit log for attendance records | ⬜ |
| C1.11 | Automated test suite (feature + unit) | ⬜ |
| C1.12 | API documentation (Scribe / OpenAPI) | ⬜ |
| C1.13 | Database backup & restore strategy | ⬜ |
| C1.14 | Production deployment (HTTPS, env hardening) | ⬜ |

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

| Stage | Contents | Why this order |
|---|---|---|
| **✅ Done** | A1–A5, A7, A8 (minus gaps) | Phase 1 web dashboard is live and verified |
| **Stage 1** | C1 — Sanctum API layer | Hard blocker; every mobile feature depends on it |
| **Stage 2** | A6 — Leave Management (web) | Biggest missing HR module; must exist before the app can expose it |
| **Stage 3** | A9 + C1.9 — Notifications + scheduler | Powers reminders, approvals, auto-absent |
| **Stage 4** | A5.5–A5.9 — finish Shift & Schedule | Roster must be complete before staff see it on the phone |
| **Stage 5** | B1–B3 — Mobile app v1 (login, punch, self-service) | First app release |
| **Stage 6** | B4–B5 — Leave + push in app | App v2 |
| **Stage 7** | A4.12–A4.19, A7.10–A7.14 | Attendance depth + payroll-ready reporting |
| **Stage 8** | D1 — AI HR Assistant | Needs mature data across attendance + leave |

---

## Counts

| Area | Built | Planned |
|---|---|---|
| Web Dashboard (A) | 40 | 51 |
| Mobile App (B) | 0 | 40 |
| Backend / API (C) | 0 | 14 |
| AI Assistant (D) | 0 | 7 |
| **Total** | **40** | **112** |

---

*Generated 2026-07-29 from the live codebase + `Requirements.html` SRS v1.0. Supersedes the stale build-status section of `Phase-1_Admin-Dashboard_Attendance_SOW.md`.*
