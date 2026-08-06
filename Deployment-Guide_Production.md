# Production deployment

**Closes:** C1.14 — the last open item in Part C.
**Unblocks:** the Flutter mobile app, which cannot be built against a laptop.

Everything here is done once, on the server. Afterwards a new revision ships
with `./deploy/deploy.sh` and nothing else.

---

## Why this comes before the app

The API, push delivery and the privacy pages are all built and tested, and all
three are unreachable from a phone as things stand. A handset cannot resolve
`127.0.0.1`; FCM will not deliver to a server it cannot call back; and neither
app store will accept a listing whose privacy-policy URL points at localhost.

Work done against a local server has to be redone against a real one, so the
hosting comes first and then the app is built once.

---

## What you need before starting

| | |
|---|---|
| A Linux server | 2 vCPU / 4 GB is comfortable for a few hundred employees. Ubuntu 22.04 or 24.04 assumed below. |
| A domain | e.g. `hr.yourcompany.com`, with an A record already pointing at the server's IP. DNS must resolve **before** you request a certificate. |
| An SMTP provider | Postmark, SES, Mailgun or Resend. Not the host's own sendmail — a new server's IP has no sending reputation and its mail lands in spam. |
| SSH root access | For the one-time setup only. |

The Firebase project is **not** needed yet. Push stays off until the app exists;
`Push-Notifications_Setup.md` covers it when you get there.

---

## 1. Server packages

```bash
sudo apt update
sudo apt install -y nginx mysql-server certbot python3-certbot-nginx \
  php8.3-fpm php8.3-mysql php8.3-mbstring php8.3-xml php8.3-curl \
  php8.3-zip php8.3-gd php8.3-bcmath php8.3-intl \
  git unzip
```

`php8.3-gd` and `php8.3-zip` are not optional here — they are what the Excel and
PDF exports need, and their absence shows up as a report that fails only when
somebody tries to run one.

Composer:

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

---

## 2. Database

```bash
sudo mysql
```

```sql
CREATE DATABASE hrms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'hrms'@'localhost' IDENTIFIED BY 'a-long-random-password';
GRANT ALL PRIVILEGES ON hrms.* TO 'hrms'@'localhost';

-- db:backup --verify restores each dump into a scratch database named
-- vrfy_hrms_<time> to prove it reads back, then drops it. The same user needs
-- to be able to create and drop those. The backslash escapes the underscore,
-- which is a single-character wildcard in a GRANT pattern.
GRANT ALL PRIVILEGES ON `vrfy\_%`.* TO 'hrms'@'localhost';

FLUSH PRIVILEGES;
```

Not `root`. The application account should be able to reach its own schema and
nothing else on the server.

---

## 3. The code

```bash
sudo mkdir -p /var/www
sudo chown -R www-data:www-data /var/www
sudo -u www-data git clone https://github.com/jasonapp2244/hr_-and_attendance_management_web_application.git /var/www/hrms-repo

# The Laravel app is the hrms/ subfolder of the repository.
sudo ln -s /var/www/hrms-repo/hrms /var/www/hrms

cd /var/www/hrms
sudo -u www-data composer install --no-dev --optimize-autoloader
```

Permissions — only these two trees need to be writable, and nothing else should
be:

```bash
sudo chown -R www-data:www-data /var/www/hrms/storage /var/www/hrms/bootstrap/cache
sudo chmod -R 775 /var/www/hrms/storage /var/www/hrms/bootstrap/cache
```

Backups live outside the application directory, so that losing the deploy does
not take the backups with it:

```bash
sudo mkdir -p /var/backups/hrms
sudo chown www-data:www-data /var/backups/hrms
sudo chmod 750 /var/backups/hrms
```

---

## 4. Environment

```bash
sudo -u www-data cp /var/www/hrms/.env.production.example /var/www/hrms/.env
sudo -u www-data php artisan key:generate
sudo -u www-data nano /var/www/hrms/.env
```

Every line marked `CHANGE ME` in that file has to be filled in. The four that
break the install silently rather than loudly:

- **`APP_DEBUG=false`** — debug pages print the database password, the mail
  password and `APP_KEY` to anyone who can trigger an error.
- **`MAIL_MAILER`** — left as `log`, leave approvals are written to a file and
  the employee is never told. Nothing errors.
- **`TRUSTED_PROXIES=127.0.0.1`** — without it, nginx is the only address PHP
  sees, so every punch records the proxy's IP instead of the employee's and the
  per-punch IP capture becomes a column of `127.0.0.1`.
- **`APP_URL`** — decides the scheme of every generated link, and is the URL
  both app stores will check for the privacy policy.

Keep `APP_KEY` somewhere other than the server. It encrypts sessions and every
encrypted column; regenerating it later logs everyone out and makes existing
encrypted values unreadable.

Then migrate:

```bash
sudo -u www-data php artisan migrate --force
```

Then create the company and the first administrator:

```bash
sudo -u www-data php artisan hrms:install
```

It asks for the company name, timezone and currency, then the administrator's
name, email and password, shows you the lot, and creates it in one transaction —
so a failure halfway cannot leave you with a company nobody can sign in to.
It also seeds the roles and permissions, which is the step most often forgotten
when this is done by hand; without it the administrator exists but every page
refuses them, and nothing says why.

For an unattended deploy, pass everything instead:

```bash
sudo -u www-data php artisan hrms:install --no-interaction \
    --company="Your Company" \
    --timezone=America/New_York \
    --currency=USD \
    --name="Your Name" \
    --email=you@yourcompany.com \
    --password='a-real-password'
```

The company timezone is the one to get right. Attendance is judged against shift
times in it, so a wrong value marks the entire workforce late every morning —
and produces plausible data rather than an error. The command validates it
against the real IANA list, and `hrms:preflight` checks it again later.

### Adding an administrator to a company that already exists

If the install already has staff but no administrator — the account was lost, or
a purge removed the seeded one — attach the new administrator to the existing
company rather than making another:

```bash
sudo -u www-data php artisan hrms:install --company-id=1
```

Run it without arguments and it lists the companies with their employee counts
and asks. Getting this wrong is unusually hard to spot: the sign-in works, the
dashboard loads, and it is empty — because every employee is on the other
company and companies cannot see each other. `--force` is what creates a second
company, and it exists only for installs that genuinely run two.

**Do not run `db:seed --class=DemoDataSeeder` on production.** It creates
`admin@hrms.test` and `hr@hrms.test`, both with the password `password`, plus a
set of fictional employees and a fortnight of attendance they never worked. A
plain `php artisan db:seed` is safe — it now creates roles and permissions only
— but if a demo seeder was ever run here by mistake, clear it out:

```bash
sudo -u www-data php artisan hrms:purge-demo --dry-run   # read this first
sudo -u www-data php artisan hrms:purge-demo --force
```

Read the dry run properly. Attendance and leave cascade from `employees`, so
deleting a demo employee takes any real rows attached to them as well — on a
server that was tested before go-live, the real punches are usually against a
demo account. The dry run lists those rows individually; `--keep-employees=CODE`
spares the employee they belong to.

---

## 5. nginx and TLS

```bash
sudo cp /var/www/hrms-repo/deploy/nginx.conf.example /etc/nginx/sites-available/hrms
sudo nano /etc/nginx/sites-available/hrms      # replace hr.example.com and the php-fpm socket
sudo ln -s /etc/nginx/sites-available/hrms /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx

sudo certbot --nginx -d hr.yourcompany.com
```

The web root is `/var/www/hrms/public`, never `/var/www/hrms`. Pointing nginx at
the project directory serves `.env`, `storage/` and `vendor/` to anyone who
guesses a path — which is the database password and every backup.

Certbot installs its own renewal timer; confirm with `systemctl list-timers |
grep certbot`.

Leave the `Strict-Transport-Security` header commented out until the site is
confirmed working over https. Browsers remember HSTS per domain for its full
duration and it cannot be withdrawn quickly, so setting it early turns a
misconfiguration into a year-long one.

---

## 6. The scheduler

```bash
sudo cp /var/www/hrms-repo/deploy/hrms-scheduler.cron /etc/cron.d/hrms
sudo nano /etc/cron.d/hrms        # set MAILTO to a real inbox
sudo chmod 644 /etc/cron.d/hrms
```

One line, every minute; Laravel decides internally what is due. Without it:

| Job | Cadence | What silently stops |
|---|---|---|
| `attendance:remind-checkout` | 15 min | Nobody is reminded they are still clocked in |
| `attendance:close-day` | hourly | Open punches are never closed; recorded hours stay wrong |
| `db:backup --verify` | 02:10 daily | There are no backups at all |

None of the three announces its absence. `hrms:preflight` infers the scheduler's
health from whether a backup actually appeared, which is the only end-to-end
evidence available.

---

## 7. The queue worker

```bash
sudo cp /var/www/hrms-repo/deploy/hrms-worker.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now hrms-worker
sudo systemctl status hrms-worker
```

Email and push are queued, so nothing is delivered without this. What makes it
easy to miss: the in-app notification bell writes directly and keeps working
perfectly, so the dashboard looks correct while no employee is being told
anything.

---

## 8. Verify

```bash
cd /var/www/hrms
sudo -u www-data php artisan config:cache
sudo -u www-data php artisan route:cache
sudo -u www-data php artisan view:cache
sudo -u www-data php artisan hrms:preflight
```

`hrms:preflight` is the actual gate. It checks debug mode, the URL and TLS
settings, proxy trust, database and pending migrations, writable paths, mail,
whether anything is draining the queue, whether the scheduler has produced a
recent backup, the backup path and binaries, push coherence, company timezones,
and whether any admin or HR account still uses the seeded password `password`.

It exits non-zero on a failure, which is why `deploy.sh` runs it last.

Then by hand:

- `https://hr.yourcompany.com` loads over TLS and the padlock is clean
- Log in as the administrator you created
- Check in and out once, and confirm the punch shows **your** IP rather than
  `127.0.0.1` — this is the proxy setting working
- Take a real backup: `sudo -u www-data php artisan db:backup --verify`
- Send a real email: submit a leave request and confirm it arrives
- `https://hr.yourcompany.com/privacy` and `/data-deletion` load while logged out

That last one matters for the next phase — both stores require a publicly
reachable privacy policy, checked while signed out.

---

## 9. Shipping a new revision

```bash
cd /var/www/hrms-repo
sudo -u www-data ./deploy/deploy.sh
```

It takes a verified backup before migrating, puts the site in maintenance mode,
pulls, installs, migrates, rebuilds caches, restarts the worker and runs
preflight — and brings the site back up even if a step fails.

---

## Ongoing

**Backups leave the server.** `db:backup --verify` proves the dump restores, but
it writes to the same machine. A server that dies takes its backups with it, so
copy `/var/backups/hrms` off-box nightly — object storage, or `rsync` to another
host.

**Watch these:**

```bash
tail -f /var/www/hrms/storage/logs/laravel-$(date +%F).log
sudo journalctl -u hrms-worker -f
php artisan queue:failed
```

**An uptime check on `/up`** — it needs no login and answers from the app rather
than nginx, so it fails if PHP or the database is down rather than only when the
whole box is.

---

## Then: the mobile app

With this done, Part C is complete and Stage 5 can start. The Flutter app has
what it needs:

- `https://hr.yourcompany.com/api/v1` — 25+ endpoints, documented in
  `API-Reference_v1.md`
- Sanctum token auth, per-user rate limits, one JSON error shape
- Push delivery server-side, waiting only on a Firebase project
- A public privacy policy and data-deletion page for the store listings

The app ships as `com.hrms.attendance` on both platforms — Android
`applicationId` and iOS `PRODUCT_BUNDLE_IDENTIFIER`. Firebase must be registered
against that exact string. The Android notification channel must be
`hrms_default` (`config/fcm.php`) — Android 8+ silently drops any notification
whose channel does not exist.

The app itself is built: login, the punch button and status card, history,
leave, roster and a manager tab, verified on an emulator against a live server.
What it still needs from a deployed host is a real `API_BASE`, since a release
build refuses to start pointed at a development address. See
`Store-Submission_Checklist.md` for what the two stores still want.

---

*Written 2026-08-03 against the live codebase. Companion files live in `deploy/`.*
