# HR & Attendance Management Web Application

A web-based HR & Attendance management system built with **Laravel 12** and **PHP 8.2**.
Phase 1 delivers the Admin dashboard and a secure QR-code attendance engine.

## Features (Phase 1)

- **Employee management** — full CRUD + bulk import
- **Org structure** — companies, offices/branches, departments, designations
- **QR attendance** — rotating one-time QR kiosk + scanner (clock in/out)
- **Auto status** — on-time / late / early-leave from office work hours + grace period
- **Live dashboard** — present/late/absent stats, 7-day trend, activity feed
- **Reports** — attendance, late arrivals, outliers, department, summary (PDF & Excel export)
- **Role-based access control** — Admin & HR roles (Spatie permissions)

## Tech stack

| Layer | Technology |
|-------|-----------|
| Framework | Laravel 12 |
| Language | PHP 8.2 |
| Database | MySQL |
| Auth / RBAC | spatie/laravel-permission |
| QR | endroid/qr-code |
| Exports | barryvdh/laravel-dompdf, maatwebsite/excel |
| UI | Bootstrap 5 (SmartHR template) |

## Getting started

```bash
cd hrms
composer install
cp .env.example .env
php artisan key:generate
# configure DB credentials in .env
php artisan migrate --seed
php artisan serve
```

Then open http://127.0.0.1:8000

### Demo accounts

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@hrms.test | password |
| HR | hr@hrms.test | password |

> Change these credentials before deploying to production.

## Roadmap

- Leave management
- Shift & schedule
- AI assistant
- Native mobile apps (employee self-service)
