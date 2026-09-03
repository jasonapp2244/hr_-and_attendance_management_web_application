# Employment Management Portal Web Application

A web-based Employment Management Portal management system built with **Laravel 12** and **PHP 8.2**,
with a Flutter app on the same API.

## Features

- **Employee management** — full CRUD + bulk import
- **Org structure** — companies, offices/branches, departments, designations
- **Attendance** — one-tap check in / check out from any browser, phone or PC.
  Timestamps are the server's, never the device's; GPS and IP travel with the
  punch as a record, never as a gate; the log is append-only
- **Auto status** — on-time / late / early-leave from the assigned shift + grace period
- **Shifts & roster** — per-department shifts, per-employee overrides, night and
  rotating patterns, a weekly planner with draft/publish, and shift swaps
- **Leave** — types, balances, and a two-step approval chain (manager → HR),
  weekend- and holiday-aware, integrated with attendance so a leave day is not an absence
- **Live dashboard** — present/late/absent stats, 7-day trend, activity feed
- **Reports** — attendance, late arrivals, outliers, department, summary (PDF & Excel export)
- **Notifications** — in-app bell, queued email, and push to the mobile app
- **Role-based access control** — Admin, HR, Employee and Manager (Spatie permissions)

## Tech stack

| Layer | Technology |
|-------|-----------|
| Framework | Laravel 12 |
| Language | PHP 8.2 |
| Database | MySQL |
| Auth / RBAC | spatie/laravel-permission |
| API auth | laravel/sanctum |
| Exports | barryvdh/laravel-dompdf, maatwebsite/excel |
| UI | Bootstrap 5 (SmartHR template) |
| Mobile | Flutter (`mobile/`) |

## Getting started

```bash
cd emp
composer install
cp .env.example .env
php artisan key:generate
# configure DB credentials in .env
php artisan migrate
php artisan emp:install      # asks for the company and the first administrator
php artisan serve
```

Then open http://127.0.0.1:8000 and sign in with the account you just made.

`emp:install` seeds the roles and permissions, creates the company, and makes
the administrator. Run on a database that already has a company, it lists them
and asks which one the administrator is for rather than quietly adding another —
staff on different companies cannot see each other, so an admin attached to the
wrong one gets a dashboard that loads and is empty. To skip the question:

```bash
php artisan emp:install --company-id=1   # join an existing company
php artisan emp:install --force          # genuinely create a second company
```

### Demo data (local only)

`php artisan db:seed` creates roles and permissions and nothing else. The
fictional company, staff and attendance are still available, but have to be
asked for by name so nothing invents people by accident:

```bash
php artisan db:seed --class=Database\\Seeders\\DemoDataSeeder
php artisan db:seed --class=Database\\Seeders\\AttendanceSeeder
```

That gives you `admin@emp.test` / `password`. Never run either on a real
install — fake staff and fake punches are indistinguishable from real ones a
week later, and `emp:preflight` fails a deploy that still has them. To clear
them out again:

```bash
php artisan emp:purge-demo --dry-run   # read this before running it for real
php artisan emp:purge-demo
```

## Documentation

| Document | What it covers |
|---|---|
| `User-Guide_Web-Dashboard.md` | What the client supplies, and how to use every screen |
| `Deployment-Guide_Production.md` | Server setup, TLS, scheduler, queue worker, `emp:preflight` |
| `Push-Notifications_Setup.md` | The Firebase work needed to switch push on |
| `API-Reference_v1.md` | The v1 API the mobile app consumes |
| `Feature-List_Web-and-App.md` | Every feature, built and planned, with status |
| `Store-Submission_Checklist.md` | What Play and the App Store still want |

## Roadmap

Leave, shift & schedule and the mobile app are built — see
`Feature-List_Web-and-App.md` for what is and is not done in each. Still ahead:

- Attendance depth — manual correction, regularisation requests, overtime, break punches
- Leave, overtime and payroll-ready reporting
- AI assistant
