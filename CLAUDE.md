# KEMP — Klutch Employment Management Program

*Known internally as the Employment Management Portal (EMP) until the rebrand,
which is why `emp` is still the database name, the `emp:` artisan prefix and the
deployment subdomain. Those are identifiers, not the product name, and renaming
them would buy nothing and break every runbook.*

Working notes. Laravel 12 + MySQL web dashboard in `hrms/`, Flutter client in
`mobile/`, both on the same API. This file is for whoever picks the project up
next. It records the things that are **not** obvious from reading the code, and
the traps that have already cost time.

---

## Running it

MySQL must be started **from the XAMPP Control Panel** — launching `mysqld.exe`
as a background task does not persist, it exits.

```bash
cd hrms
php artisan serve            # http://127.0.0.1:8000
php artisan test             # 1398 tests, ~270s, SQLite in memory

cd ../mobile
flutter analyze
flutter test                 # 290 tests
```

The app's strings are generated from `mobile/lib/l10n/*.arb` on `flutter pub
get` and on every build, into a git-ignored `lib/l10n/generated/`. Nothing extra
to run after a clone; `flutter gen-l10n` regenerates them by hand.

`config('app.timezone')` is **deliberately UTC** and must stay that way. Per-company
time comes from `Company::tz()`. "Fixing" it to a local zone would shift what
"today" means for every other company.

### Signing in locally

**A plain `php artisan db:seed` creates no users at all** — `DatabaseSeeder`
calls `RolePermissionSeeder` and nothing else, deliberately, so that seeding a
real installation cannot conjure an account with a known password. Accounts come
from one of two places instead.

**`php artisan emp:install`** — one administrator, on an email and password you
are prompted for. No employee record, which is intended: see below.

**`php artisan db:seed --class=Database\Seeders\DemoDataSeeder`** — the demo
company, and seven accounts, **all on the password `password`**:

| Email | Roles | Employee | On the phone |
|---|---|---|---|
| `james.smith@acme.test` | employee + manager | EMP-0001 | Everything, **including the Team tab** |
| `emily.johnson@acme.test` | employee | EMP-0002 | The five employee tabs |
| `michael.brown@acme.test` | employee | EMP-0003 | The five employee tabs |
| `jessica.davis@acme.test` | employee | EMP-0004 | The five employee tabs |
| `david.wilson@acme.test` | employee | EMP-0005 | The five employee tabs |
| `hr@emp.test` | hr | EMP-0006 | The five employee tabs, **no Team tab** |
| `admin@emp.test` | admin | *none* | Signs in, then the no-employee-record state |

Four reporting lines run to EMP-0001, which is what makes `is_manager` true for
that account and nobody else — the app needs the `approve-leave` permission
**and** a direct report before it draws the Team tab, and these accounts are the
pair that tells those two apart. **HR holds the permission and leads nobody**,
on purpose.

**HR got its employee record on 2026-09-16, and had none before that.** The user
and the role had always been there, so nothing failed — HR simply landed on the
admin empty state on all four employee screens, and two of the four roles could
not be shown on a handset at all. `tests/Feature/Api/DemoRoleAccessTest` now
pins what each role gets, because the only previous way to find this out was to
sign in and look.

**Admin deliberately has no employee record and must not be given one.** An
administrator operates the system rather than working for the company. The
refusal it produces is a designed, tested screen with no retry button on it, and
a seeder that "fixed" this would hide the one state four screens are built
around.

The demo panel puts one-click role buttons on the login page:

```
DEMO_QUICK_LOGIN=true
DEMO_QUICK_LOGIN_ACCOUNTS="james.smith@acme.test:password,hr@emp.test:password"
```

**The panel does not create anybody** — it is a list of credentials to fill the
form with, so every address in it has to exist already. The local `.env`
currently names `test.admin@local.test` and two siblings that no seeder or
command in this repository creates; they were made by hand, and on a fresh
database those buttons fail.

It is forced off when `APP_ENV=production`, and `emp:preflight` fails a deploy
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

### 6. "Last punch" is not "clocked in"

There are **four** punch types, not two. `break_start` and `break_end` are
neither `in` nor `out`, so anything that reads the day's last row to decide
whether somebody is on the clock treats a returning employee as one who went
home — and the next press opens a second attendance stretch, losing the
morning's pairing.

`AttendanceService::record` gets this right by filtering to `whereIn('type',
['in','out'])`. The API's `/attendance/today` did not: it shipped reading
`$logs->last()`, so an employee who took a break on the web portal and then
opened the app was offered "Check In" while still on the clock. Nothing failed;
the screen was just wrong.

**`breakState()` is the one definition** — it is what the live board uses, and
what `today` uses now. Never re-derive this from a punch list.

### 7. The punch cooldown measures `created_at`, not `scanned_at`

`recentlyScanned()` compares `created_at` against `now()`. In a test that means
**`travelTo` before writing the fixture punch, not after**: a row written at the
real clock and then compared against a travelled `now()` is a negative diff,
which reads as "within the cooldown", and every POST in the test returns 429
`duplicate_scan`. Seven tests failed this way at once and the message points at
the endpoint rather than the fixture.

```php
$this->travelTo(Carbon::parse('2026-08-03 09:00:00'));   // first
$this->punch('in', '2026-08-03 09:00:00');
$this->travelTo(Carbon::parse('2026-08-03 13:00:00'));   // then move on
```

---

### 8. Real file I/O inside `testWidgets` hangs, it does not fail

`testWidgets` runs in a fake-async zone. It advances timers when you `pump`, and
it **never delivers a real file completion at all** — so a widget whose build or
`initState` reaches `OfflineCache`, `PunchQueue` or any other `dart:io` call
sits there for ever. No assertion fires, no timeout fires, the whole
`flutter test` run just stops with the test name on screen and nothing after it.
That looks exactly like an infinite `pumpAndSettle`, which is what you will
waste the time looking for.

Prime the stores first, inside `tester.runAsync`, and pump a screen that only
reads memory:

```dart
await tester.runAsync(() async {
  final store = OfflineCache(directory: dir);
  await store.write(OfflineCache.keyProfile, {'user': ...});  // loads the file
  final queue = PunchQueue(directory: dir);
  await queue.load();
  session = Session(api: ..., cache: store, queue: queue);
  await session.restore();
});

await tester.pumpWidget(app(session));   // memory only from here
```

Both stores read their file once and hold it, so this is also what a real
handset does after the first launch.

Where a screen has to load *through* the cache rather than be handed a primed
one, `test/support/settle.dart` is the only way to let it finish: `runAsync`
gives the disk real time and `pump` drains the continuations that finish because
of it, alternating, because a load is a chain of them. Those rounds are real
wall-clock time, so **the budget is a bet on how busy the machine is** — and
`flutter test` runs the files concurrently. Three files each carried their own
copy of the loop at six rounds, which was comfortable alone and intermittently
short with sixteen files running; the failure reads as a screen that ignored its
own response, not as a test that did not wait. One copy, twelve rounds.

### 9. The biometric lock can lock its own owner out

`AppLock.enable()` runs the check **before** it writes the preference, and that
order is the whole feature. A switch that saves first and asks afterwards puts
the app behind a sensor that has just refused somebody, on a phone they cannot
sign out of either — the way back is a reinstall, which also discards every
punch still queued for a signal.

Three more rules go with it, and each is there because the phone is not a
reliable partner:

- **The lock screen always offers *Sign out instead*.** Fingerprints get
  removed, face data gets reset, a sensor breaks. `BiometricOutcome.unavailable`
  is the one outcome that must never be retried into a dead end.
- **The switch is drawn only where `isAvailable()` is true**, which means a
  biometric actually *enrolled* — not merely "the device supports
  authentication". With a PIN and no fingerprint, `authenticate()` still
  succeeds by prompting for that PIN, so the row would offer fingerprint unlock
  on a phone that has none and then ask for something else.
- **`backgroundGrace` must stay well above zero.** The OS backgrounds the app
  for its own dialogs — the location prompt at the first punch, a document
  opening elsewhere, *and the biometric sheet itself*. Locking on every resume
  puts the lock behind the sheet it just opened. `_prompting` guards the sheet;
  the minute covers the rest.

The preference lives in the keystore beside the token and is cleared with it:
it says "this phone is shared", which is a statement about the person who set
it, not about the handset.

**Android needs three native changes, and two of them fail only at runtime.**
`MainActivity` extends `FlutterFragmentActivity` — androidx.biometric's prompt
is a Fragment and there is no FragmentManager under a plain `FlutterActivity`,
so the first press of Unlock throws. `LaunchTheme` and `NormalTheme` descend
from `Theme.AppCompat` for the same reason, and without it the prompt crashes on
Android 8 and below only. `USE_BIOMETRIC` is declared; there is deliberately no
`<uses-feature>`, which would hide the app in the Play Store from every device
without a sensor. On iOS, `NSFaceIDUsageDescription` is the same trap as the
location key — iOS kills the app rather than refusing, and only on a Face ID
handset.

**A Kotlin plugin builds noisily from a different drive.** With the pub cache on
`C:` and the project on `F:`, `local_auth_android`'s incremental compile logs a
stack of `IllegalArgumentException: this and base files have different roots`.
The build succeeds — `flutter build apk` finishes and the APK is written. It is
the first plugin in this app with Kotlin sources, so this noise is new, and it
is not a failure.

### 10. The app gate is the one switch that can stop everybody

`GET /app/status` decides whether a build may carry on (B6.6), and it is driven
by two values typed into an env file. Get either wrong and every handset in the
company stops at a screen — and the people it stops are the ones who clock in
with it. Nothing on the server side goes wrong when that happens, so there is
no alarm to notice.

Everything about it is therefore built to **fail open**, at every level:

- `AppVersion::compare()` returns **null**, not `-1`, for anything it cannot
  read. A caller treating that as "older" would refuse a whole fleet over a
  typo. `isOlderThan()` is the only thing that turns it into a bool, and it
  answers false whenever it does not know.
- The endpoint does **not** validate its query. A 422 is the one shape the app
  cannot act on — it asks this before it knows anything, so a refusal leaves it
  with no verdict at all. Junk in either parameter falls through to `ok`.
- A minimum version with no store link for that platform answers `ok`. An
  update screen with a dead button cannot be dismissed *or* acted on.
- `AppGate` in the app treats an unreachable server, an unparseable body and an
  action invented after the build shipped as `ok`. **This one matters most**:
  the app is deliberately usable with no signal, and a gate that blocked on a
  failed request would take the offline cache and the punch queue away in
  exactly the conditions they exist for.

**The comparison lives on the server, not in the app.** The app is the half
that cannot be fixed — a handset with a broken comparator has already shipped,
and the answer it is given is the only thing left that can change what it does.

**Maintenance is a flag of its own, not `php artisan down`.** `down` returns
503 to everything, which the app cannot tell apart from an outage: it would
fall back to its cache and let somebody queue punches into a server being
migrated underneath them. Both settings live in `config/mobile.php` rather than
in the database, because the moment they matter most is the moment the database
is unavailable. `emp:preflight` fails a deploy that leaves maintenance on.

### 11. The store declarations drift silently, and only Apple notices

Four documents describe what the app collects and a reviewer compares them: the
data-safety table in `Store-Submission_Checklist.md`, `/privacy` on the server,
`mobile/ios/Runner/PrivacyInfo.xcprivacy`, and the forms on both consoles.

**Two of them said the app does not collect location for some time after B2.3
shipped** — the checklist row read "not collected" and the Apple manifest had no
location entry at all, both with comments promising to be updated "when B2.3
ships". Nothing failed. An app that collects location without declaring it is
the most common cause of an enforcement removal, and the removal arrives after
review, not at upload.

Re-read all four against the code before any submission. The same trap is now
armed for push: an FCM token is a device identifier, and the day
`google-services.json` lands in a build, the data forms change with it.

### 12. A guard claimed after an `await` is not a guard

`PunchQueue.flush()` shipped with this, and `CrashReporter.flush()` was written
with it before it was caught:

```dart
await load();
if (_pending.isEmpty || _flushing) return;   // wrong
_flushing = true;
```

`await load()` yields to the microtask queue **even when `load()` has nothing to
read and returns immediately** — an `async` function's caller always suspends.
So two callers arriving together both get past the check, both set the flag, and
both send the same batch. The pair that does it in practice is a resume and a
manual retry landing in the same turn, which is exactly the case the guard was
written for.

Nothing looked broken: the server recognises the second delivery of a punch as
duplicates, so no attendance was written twice. The cost was on the app side —
a `SyncOutcome` reporting punches as *duplicate* that had in fact just been
accepted by its own first call.

**Claim the flag before the first `await`, and release it in a `finally`.**

### 13. Crash reports go to this server, and nowhere else

B6.5 is a table on the employer's own server, not Crashlytics, not Sentry. That
is a decision, not an oversight: a stack trace routinely carries fragments of
whatever the app was holding, and four documents — `/privacy`, the Apple
privacy manifest, and both store data forms — say the app shares nothing with
any third party and contacts exactly one host. A crash SDK falsifies all four at
once. If one is ever added, they all change with it, and so does the answer to
the tracking question.

The rest of the design follows from what a crash reporter has to survive:

- **Written to disk at the moment of the crash, delivered on the next launch.**
  A reporter that posts from inside a dying process loses the crash that killed
  it, which is the only kind worth having.
- **`POST /app/crashes` is unauthenticated.** The crash worth having most is the
  one that stops the app opening; an endpoint behind `auth:sanctum` would
  collect every crash except that one. The controller reads a token if one is
  present, so a report from a signed-in handset is attributed anyway. Being a
  public write, it has its own tight limiter and a hard cap on every field.
- **Nothing in `CrashReporter` may throw.** An error handler that fails turns
  one crash into a loop, so every path swallows its own failures.
- A report the server *refuses* is dropped rather than kept — holding it would
  retry one rejection at every launch for ever. A report that never *arrived* is
  kept. `ApiException.isNetworkFailure` is the difference, the same distinction
  the offline cache turns on.

### 14. A status colour cannot be one value

`AppTheme.present` and its four siblings used to be `static const Color`, chosen
against a white card and then drawn unchanged on a #161C22 one. Three of the
five came out between 2.8:1 and 3.4:1 in dark mode — under AA for the 11–13px
text they are almost always used for, and under *everything* on the raised
surfaces.

There is no fixing that by picking a better value. Body text on white needs a
relative luminance at or below about 0.17; body text on #1E262E needs one at or
above about 0.26. Nothing is both. So the colours live on **`AppColors`**, which
resolves by brightness:

```dart
final colors = AppColors.of(context);   // in build(), or before the first await
```

Every value clears **4.5:1 on the worst surface it meets, including its own
10–15% tint** — the house pattern for a banner or a chip puts the colour on a
wash of itself, which is the tightest pairing in the app and the one most easily
got wrong. `test/accessibility_test.dart` measures all of that; it does not
consult this comment.

**`AppTheme.brand` is identity, not text.** #F26522 is 3.15:1 on white: enough
for the 21px Check-in label (large text, 3:1), the splash mark and the focus
ring, and nowhere near enough for a caption or a 16px button label. It was
`primary` with white on it, which failed on every ordinary button in the app;
`primary` is `brandDeep` now in light mode and near-black-on-orange in dark.
Putting the bright orange back on `primary` fails a test.

### 15. Fixed heights clip at the OS's larger font sizes

Nothing in the app clamps `textScaler`, which is right — an employee who has
turned the system font up has done so deliberately. The cost is that a
`SizedBox(height: …)` wrapped round text is a clipping bug waiting for the first
person who uses that setting, and it had claimed the punch button, the break
button and two rows on the clock screen.

Use `ConstrainedBox(minHeight:)` and a matching `minimumSize` on the button
style — the theme's own `Size.fromHeight` will otherwise pull it back down — and
give a `Row` carrying text an `Expanded`, or make it a `Wrap`.

`test/accessibility_test.dart` pumps five screens at 2× in both themes. Flutter
raises an overflow as a rendering exception, which `testWidgets` fails on, so
that is a real check and not a screenshot somebody has to look at.

### 16. One list per side for the notification route

`route` decides which tab a notification opens, and it used to be written out
by hand in every `toPush()` on the server and listed again in `PushRoute` on
the app. The two drifted: `schedule` was sent for months to a build whose enum
had never heard of it, and because an unknown route opens the app normally
rather than crashing, nothing ever said so.

There is now one list on each side. On the server, **`App\Support\AppRoute`**
maps a notification's `type` to its route, and both the push payload and the
notification history (B5.6) read it. In the app, `PushRoute.parse` handles
both a pushed route and a listed one, so `AppNotification` cannot invent a
second answer.

It is keyed on `type` rather than on anything only a push carries, because
`toDatabase()` has never recorded a route — so every row already in the
`notifications` table has to get its answer from the type alone.

**Null is an ordinary answer.** `document_expiring` and `late_arrivals` are
addressed to HR, who work at a desk; the app has no screen for either, and a
notification with nowhere to go simply offers no button.

### 17. Not everything in the keystore belongs to the account

Four things on the handset are cleared when the token is: the punch queue, the
offline cache, the biometric preference and the unread badge. Each belongs to
the person who was signed in, and the next one on a shared phone must not
inherit it.

**The onboarding flag is the exception** (B1.1). It describes the *handset* —
whether this phone has ever been introduced to the app — and signing out at the
end of a shift is not a request to be walked through the carousel again in the
morning. `Session._clearToken` deliberately does not touch
`hrms_onboarding_seen`, and there is a test that says so.

**The language is the second exception, and for the same shape of reason**
(B6.2). `hrms_locale` describes the handset, and clearing it at sign-out would
put the *login form* back into a language the person standing there cannot read
— on the one screen they cannot get past in order to change it. `_clearToken`
deliberately does not touch it either, and `test/locale_test.dart` says so.

Two more rules go with it. `needsOnboarding` is read **once, inside
`restore()`**, because `_Root` builds synchronously and an answer arriving a
frame later has already flashed the login form at the person it was meant to
introduce. And it is never true for a session that restored: somebody signed in
on this handset has used it before, whatever the keystore says.


### 18. A translated label cannot also be an identifier

The app is drawn in English or Spanish (B6.2), and two things in it were
matching on **the word under an icon** rather than on a key.

`HomeShell` keyed its per-tab visibility map on the tab's label, and `PushRoute`
carried a `tabLabel` that a tapped notification was matched against. Translate
the labels and both stop finding anything — a notification tap would have opened
the app on whatever tab it happened to be on, on every Spanish handset, and
nothing would have thrown. `_Tab` has an `id` now, `PushRoute` has `tabId`, and
one function — `tabLabel(t, id)` in `home_shell.dart` — turns an id into the
word, so the shell and the notification row cannot disagree about what a tab is
called.

The rule generalises: **anything that has to *find* something matches on a key,
and only the last step turns a key into words.** The same shape applies to
`AppColors.statusStyle` and `punchTypeLabel`, both of which take a server key
and hand back a translated label.

### 19. `context.t` in `initState` is an assertion, not a warning

`AppLocalizations.of` is `dependOnInheritedWidgetOfExactType`, and Flutter
refuses that before the element has finished its first build. Every data screen
here calls `_load()` from `initState`, so a `final t = context.t;` at the top of
`_load` — the obvious place, because the `catch` is what needs it — takes the
screen down with *"dependOnInheritedWidgetOfExactType() was called before
initState() completed"*.

Nothing fails at compile time and `flutter analyze` says nothing. It shows up as
a widget test that finds none of the text it was looking for.

**Read the strings inside the `catch`, after the `mounted` check.** That is
already the house pattern for a `BuildContext` on the far side of an `await`,
and the strings are only ever needed there:

```dart
} on ApiException catch (e) {
  if (!mounted) return;
  final t = context.t;          // here, never above the `try`
```

A handler invoked by a button is fine — the constraint is `initState` alone.

### 20. Spanish does not build a date the way English does

"4 August 2026" is "4 **de** agosto **de** 2026". A date assembled in Dart as
`'$day $month $year'` cannot express that, so `dateShort` and `dateLong` are
**messages with placeholders** and the ordering belongs to whoever writes the
translation. The same goes for anything that reads as a sentence:
`Regularisation.summary` is four separate messages rather than "disputing a "
plus a punch name, because lower-casing an assembled English sentence is not a
translation strategy.

**The month names live in the ARB files, not in `intl`'s `DateFormat`.** That is
deliberate. `DateFormat('d MMM', 'es')` needs `initializeDateFormatting` to have
been called first and throws `LocaleDataException` when it has not — at the
moment a date is drawn, which is every screen in the app. A launch-time step
that a future translation change could quietly come to depend on is a worse
trade than twenty-four extra rows.

### 21. A notification is not written in the language of the request that caused it

HR approves leave in English; the employee reads Spanish. The message is
rendered by a **worker**, in a process with no request and no `Accept-Language`
header at all — so the language cannot come from the request, and
`app()->getLocale()` at send time is whoever pressed the button (C1.18).

The framework already has the answer, and it is the only one that covers push,
the notification centre and the email in one go: `User` implements
`HasLocalePreference`, and `Illuminate\Notifications\NotificationSender` wraps
every send in `withLocale($notifiable->preferredLocale())`. Nothing in a
notification class knows about locales; they just call `__()`.

`users.locale` is what fills it in, and **nobody types it**. `SetApiLocale`
writes the header there in `terminate()`, so the column is a record of what
somebody is actually being shown rather than a second preference to maintain. A
null — every account that has only ever used the web dashboard — resolves to the
default.

**The notification *history* keeps the words it was written with.** A row
written before somebody switched language stays in the old one. Translating on
read instead would mean storing keys and parameters in `notifications.data`, a
schema change that would also leave every existing row unreadable, for a payoff
nobody has asked for.

### 22. `*/` inside a docblock ends the docblock

`` `lang/*/leave.php` `` in a comment closes the block four words early and the
file stops parsing, with the error pointing at whatever line follows. Obvious in
hindsight, ten minutes in practice. Write it as "the `leave.status`
translations", or any other way that does not contain the sequence.

### 23. The exception's own message outranked the translation

`bootstrap/app.php` built the error payload as
`$e->getMessage() ?: $message`, so that an `abort(403, 'That leave request is
not yours.')` reached the client with its own wording rather than a generic
"forbidden". Reasonable, and it quietly undid half of C1.18: the exceptions the
framework raises **carry an English message of their own**.
`AuthenticationException` is "Unauthenticated.", `AuthorizationException` is
"This action is unauthorized.", `ThrottleRequestsException` is "Too Many
Attempts." — so the two refusals a handset meets most often came back in English
while everything around them was Spanish.

Each arm of the `match` already decides what to say, including the one that
prefers an abort's own message, so the payload takes `$message` and nothing
else.

**Nothing in the suite noticed**, and nothing was going to: 1154 tests and not
one of them read the `message` on a refusal — they assert the status and the
`error` code, which is exactly what the client is supposed to branch on. It took
a `curl` against a running server. Two tests cover it now, and the general
lesson is worth more than either: **a test suite that only asserts the fields a
client acts on cannot see anything about the fields a person reads.**

### 24. A roster time is a wall clock, and it was being read as UTC

A shift stores `end_time = '17:00:00'` and means five o'clock **where the
company is**. `shiftEndFor()` did `Carbon::parse($workDate.' '.$shift->end_time)`
with no zone, which is five o'clock UTC — and both its callers compare the
result against `now($company->tz())`, which is an **instant**, not a wall clock.
Carbon compares instants, so the two silently disagreed by the company's offset.

For a company four hours behind, "has the shift ended?" answered yes four hours
early: `attendance:remind-checkout` nudged people at lunchtime, and
`attendance:close-day` wrote an automatic clock-out — with the *scheduled*
hours — while they were still working. Every test passes because every test
company is on `UTC`, where the bug does not exist. The seeded install is UTC
too, so it would have surfaced on the first non-UTC client and nowhere before.

Both helpers now parse in `tzFor($employee)`. Note the one that must **not**:
`scheduledMinutesFor()` measures a duration, and a zone applied to one end and
not the other turns an eight-hour shift into a four-hour one — both sides there
are parsed the same way, deliberately.

`config('app.timezone')` being UTC is correct and must stay that way (see
"Running it"). That is exactly why anything holding a *company's* wall clock has
to say so at the point it is parsed.

### 25. A scheduled window can fall between two runs

B5.1's clock-in reminder fires inside `[start − lead, start)` — a window that
closes, unlike every other job here, which fires once a moment has passed and
can afford to be late. With the quarter-hourly cadence the other attendance jobs
use and the default ten-minute lead, an 09:00 shift's window is 08:50–09:00: the
08:45 run is too early and the 09:00 run is too late. **Nobody is ever
reminded** — no error, no log, no failing test, for every employee, forever.

`attendance:remind-checkin` is scheduled `everyFiveMinutes()` and
`PolicyController` refuses a lead between 1 and 4 minutes so the two ends cannot
drift apart. If the cadence is ever slowed, the validation has to move with it.
The general shape: **a job whose window closes needs an interval shorter than
the shortest window it can be asked for**, and the coupling has to be written
down at both ends because nothing enforces it at runtime.
### 26. A new permission cannot arrive by re-running the seeder

`RolePermissionSeeder` ends in `syncPermissions()`, which is right for a fresh
database and wrong for a live one: it does not add, it **replaces**. A client who
had taken `export-reports` away from HR through the Roles & Permissions editor
(A1.4) would find it handed back on the next deploy.

Which is why `deploy/deploy.sh` runs `migrate --force` and no seeder at all — and
why a permission added to the seeder alone reaches a fresh install and every
test, and **never reaches a running server**. The menu simply would not appear,
with nothing in any log to say why.

So a new permission is declared in two places, on purpose: the seeder for a new
database, and a data migration using `givePermissionTo` — additive, idempotent,
leaving every other grant as the administrator left it — for the ones already
out there. See `2026_09_11_000002_add_manage_announcements_permission.php`.

The path that matters in production is also the one `RefreshDatabase` cannot
reach, since migrations run before the seeder and the roles do not exist yet.
`AnnouncementTest` calls the migration's `up()` by hand for that reason.

### 27. A broadcast has no undo, so the model has to say so

Publishing an announcement (B5.5) writes a row into every recipient's
`notifications` table and pushes to every registered handset. Neither can be
recalled, so the register's own copy is refused an edit **on the model**, not
just in the controller — `booted()` throws on `updating` and `deleting` once
`published_at` is set, the same rule the attendance and activity trails keep.
The controller catches it and turns the exception into the sentence explaining
why; a console command or a future endpoint gets the exception.

`publish()` is the one caller allowed past the guard, and it goes through
`forceFill(...)->saveQuietly()`.
### 28. A score of zero and no score at all are different answers

B3.5's attendance score is `ontime / obliged`, and `obliged` is legitimately
zero — a window of weekends, a fortnight of booked leave, somebody's first week
before they started. Returning 0 for that is arithmetically defensible and
completely wrong in front of a person: it reads as a failure, and the person
most likely to see it is somebody just back from leave.

`scorePercent()` returns **null**, the API sends `null`, and the card draws
"No score yet" with a line saying why. The same rule applies to anything else
here that divides by a count of days.

The other half is the streak, and the trap there is the opposite: an unfinished
day is not an absence. Counting today against somebody before the day is over
would show every employee in the company a zero every morning — the feature
working perfectly and being useless. `onTimeStreak()` skips today when there is
no punch yet, and counts it the moment there is one.

### 29. The 2x accessibility pass only covers the screens pointed at it

`test/accessibility_test.dart` pumps screens at `TextScaler.linear(2.0)` in both
themes, and an overflow is a rendering exception, so it fails rather than
producing a screenshot nobody looks at. That only works for screens that are in
the file — and a screen needing an API was not, which is why B3.5's score card
shipped its first draft with a `Row` that overflowed by 180 pixels at the
largest text size.

A screen that loads from the server is pumped by **seeding the offline cache
and letting the mock client throw** — `offlineSession(tester, history: {...})` —
the same trick the clock screen already used. There is no reason left for a
screen to be missing from that file.

A `Wrap` is not enough on its own: it wraps its own children, not the contents
of a `Row` inside one. Text beside an icon needs `Flexible`.

### 30. The end of a window is only today when nobody asked for less

The app never names a date the server has not named first: it asks for the
default window, reads `to` out of the reply, and counts from that. Everything
dated on the History and Roster screens is built that way, because the phone is
wherever its owner is and attendance is judged in the company's zone.

`to` is the window's end. It is today **only because nothing earlier was
asked for**. The month grid (B3.4) is the first caller to send both ends, and
paging back to March gets `to: 2025-03-31` — a true statement about that
window, and not a statement about today. Anchoring on it moved the app's idea of
today to the end of whichever month was being read: the forward arrow went dead
one month back, and switching to the list then asked for thirty days ending
three weeks before today. Both screens looked internally consistent. Nothing
errored.

**So the anchor is taken only from a reply to a request that named no `to`.**
Widening the rule — "read the echo, it comes from the server" — is what broke
it; the echo is only today's date when the request left `to` alone. A reply to a
bounded window can be drawn, but it cannot be used to tell the time.

The same applies to `from`: it may never be later than the anchor, because a
window starting after today comes back empty and reads as a month nobody
attended. `test/history_calendar_test.dart` runs its mock server in **April
2025**, nowhere near the machine, so a date built from the handset fails on the
first expectation instead of passing for eleven months of the year.

**A screen that *offers* a date is covered by this rule too, and two were not.**
`showDateRangePicker` and `showDatePicker` take `currentDate` — the day they
draw a ring around — and Material defaults it to `DateTime.now()`. Caught on the
handset: the phone was on 15 September, the company (America/New_York) on the
14th, and the Clock, History and Schedule tabs all said the 14th while the leave
picker ringed the 15th. Somebody booking "from today" books the wrong day, and
the app visibly disagrees with itself.

On the corrections form it was worse than cosmetic: `lastDate` is a **rule** —
the server refuses a correction to a time that has not happened — so the picker
offered a date the app itself would then be refused for. `/leave/balances` and
`/attendance/regularisations` now carry `today` in the company's zone, and both
pickers take `currentDate`, `firstDate` and `lastDate` from it. `date('Y')` went
with them: PHP's `date()` reads the machine clock and ignores
`Carbon::setTestNow` entirely, so that line could not be made to fail in a test
no matter what timezone the company was in — an unfreezable clock is its own
reason not to ask one the time.

### 31. `flutter build apk` can ship a Dart kernel that is weeks old

Two consecutive `flutter build apk --debug` runs produced a **byte-identical**
APK that did not contain the source in front of them. The build reported
success in nine seconds, `adb install` reported success, the app launched, and
the feature simply was not there — which reads as the feature being broken, not
the build being stale. Deleting `.dart_tool/flutter_build` did **not** clear it.
Only `flutter clean` did.

So when a change does not appear on the device, **check the artifact before
debugging the code**:

```bash
unzip -p build/app/outputs/flutter-apk/app-debug.apk \
  assets/flutter_assets/kernel_blob.bin | grep -ac "some new string"
```

Zero means the APK is stale; `flutter clean` and rebuild. It costs about a
minute, and it is cheaper than the hour spent looking for a bug in code that was
never running. `uiautomator dump` is the other half of the same check — Flutter
paints to a canvas, so a screenshot cannot be grepped, but every `tooltip` and
`Semantics(label:)` lands in the accessibility tree and can be:

```bash
adb shell uiautomator dump /data/local/tmp/ui.xml
adb shell cat /data/local/tmp/ui.xml | tr '<' '\n<' | grep -oE 'content-desc="[^"]+"'
```

### 32. A Flutter package can add permissions you never asked for

`flutter pub add file_picker` (B4.1) put four permissions into the merged
manifest: `READ_EXTERNAL_STORAGE` and all three `READ_MEDIA_*`. The package
declares them for the modes that browse media; this app uses
`FileType.custom`, which is `ACTION_OPEN_DOCUMENT` — the system picker, which
grants access to the one file the user chose and **needs no permission at all**.

Nothing fails at runtime, which is why this is only ever found at review.
`READ_MEDIA_IMAGES` and `READ_MEDIA_VIDEO` put an app on Google Play's Photo and
Video Permissions policy path, wanting a declaration and a justification for
access the app never exercises — on top of asking every user for more than the
feature needs.

They are removed explicitly, with `tools:node="remove"` in
`android/app/src/main/AndroidManifest.xml`, so a future reader sees a decision
rather than an absence. **After adding any plugin, read what it merged in:**

```bash
flutter build apk --debug
grep -oE 'android:name="android\.permission\.[A-Z_]+"' \
  build/app/intermediates/merged_manifest/debug/processDebugMainManifest/AndroidManifest.xml | sort -u
```

The list should be the eight in the main manifest and nothing else.

### 33. `useSafeArea: true` does not cover the bottom of a bottom sheet

`showModalBottomSheet(useSafeArea: true)` wraps the sheet in
`SafeArea(bottom: false)` — deliberately, because a sheet usually has a keyboard
under it and the caller is expected to handle the bottom with `viewInsets`.
Every sheet in this app did exactly that and no more, so with the keyboard
**down** the end of the sheet was drawn underneath the navigation bar.

The end of a sheet is where the submit button lives. On the handset the **Save**
on *Home & emergency contact* was half-covered by 48dp of opaque black and the
sheet had nothing left to scroll: the only way to press it was to aim at the top
half of a button you could not fully see. The leave sheet lost the last line of
its footnote the same way.

`lib/widgets/sheet_padding.dart` is now the one expression, and the term that
matters is `padding.bottom` rather than `viewPadding.bottom`: `padding` is
already the system inset **minus** whatever `viewInsets` covers, so it is the
navigation bar when the keyboard is down and zero when the keyboard is up over
it. Adding it to `viewInsets.bottom` is correct in both states and
double-counts in neither.

### 34. A retry button is a claim that retrying can work

HR and administrator accounts are not employees — `emp:install` creates an
administrator with no employee row, and HR's work is on the web. Signing one
into the app is ordinary, and every employee-facing endpoint answers it with
`403 forbidden`.

All four data tabs rendered that as the ordinary error card, each offering **Try
again** for a condition that will still be true on the hundredth press. The
Profile tab already said the true thing and pointed at the web dashboard; the
rest implied the server was having a moment.

`AsyncView` already hides its button when `onRetry` is null, so the fix was to
know which failure this is: `ApiErrorText.isMissingEmployeeRecord`, and
`onRetry: _fatal ? null : _load` on the seven screens that can hit it. The flag
is cleared at the top of every `_load`, because a screen that dropped its retry
for good after one refusal would strand a user who later has no signal — which
`test/no_employee_record_test.dart` asserts in both directions.

### 35. A launcher shortcut outlives the app that wrote it

Everything else this app writes dies with the process or lives in a store it
controls. A quick action (B2.8) does neither: `setShortcutItems` hands the OS a
`type` string, the **launcher keeps it**, and it is handed back verbatim on a
tap — days later, after a reinstall's worth of upgrades, from whichever build
last published it.

Three consequences, and none of them shows up in a test that only runs one
build:

- **The type can never be a translated label.** Publishing `t.punchCheckIn` as
  the type puts `Fichar entrada` on a Spanish launcher, and `QuickAction.parse`
  has never heard of it. `wireValue` is the identifier and `title` is the
  words — trap 18 again, in the one place where the two live in different
  processes.
- **`parse` must tolerate a string this build does not know.** A downgrade, or
  a renamed enum row, hands back a type from the future or the past. Null, and
  the app opens normally.
- **It must be withdrawn on sign-out.** It is in `_clearToken` beside the punch
  queue and the cache, for the same reason they are: on a shared work handset a
  *Check out* left on the menu by the last person is one tap from clocking out
  the next one, and unlike anything on screen it is reachable **without opening
  the app at all**.

The tap is also held rather than dropped while `/attendance/today` is in
flight. Launching *from* the shortcut is the normal case, so the tap always
arrives before there is a day to punch against; clearing it there leaves the
feature working only when the app was already open. `_load`'s `finally` asks
again once the day has settled, and the tap is spent either way at that point —
one that survived into the next refresh would punch twice for one press.

The punch itself goes through the screen's own `_punch()`, so the fence, the
fix, the cooldown and the offline queue are not reimplemented behind it.

### 36. `env()` in `bootstrap/app.php` is read before .env exists

`TRUSTED_PROXIES` was configured in the `withMiddleware` closure:

```php
if ($proxies = env('TRUSTED_PROXIES')) { $middleware->trustProxies(at: …); }
```

**That guard was never once true**, on any box, in any environment, for the
whole life of the file. `withMiddleware` registers its callback on
`afterResolving(HttpKernel::class)`, and `Application::handleRequest` resolves
the kernel on the line *before* it calls `$kernel->handle()` — and `handle()` is
what runs `LoadEnvironmentVariables`. So the closure runs before .env has been
read and `env()` answers null. Verified rather than reasoned about: an
`afterResolving` hook registered alongside it reports `env('DB_DATABASE')` as
null too, on a build with no cached config at all.

This is **not** the familiar "don't call `env()` outside config files once you
run `config:cache`" rule. That one at least works until the first cache. This
fails always, and silently, because the value is optional by construction: a
null proxy list is indistinguishable from "no proxy in front of this box".

The consequences were all quiet. Behind Varnish, `$request->ip()` is the
proxy — so every punch filed the proxy's address (`Api\AttendanceController`),
the IP column in exports was a column of one repeated value, and the **login
rate limiter keyed on that address**, which turns A1.10's "one person's mistakes
cannot lock out a colleague" into its exact opposite: five wrong passwords
anywhere in the company, and the whole company is locked out.

It now lives in `config/trustedproxy.php`, read by the framework's own
`TrustProxies` at request time through its documented
`config('trustedproxy.proxies')` fallback — which means config is loaded by the
time it is read, and a config file is the only place `env()` survives
`config:cache`. `emp:preflight` reads the same config key rather than `env()`,
for the same reason: after a deploy has cached the config, an `env()` check
would report "unset" on the box that had just set it correctly.

`tests/Feature/TrustedProxyTest` drives the real punch endpoint through the real
global middleware stack. **Four of its five tests would have passed against the
broken build** — they set the config key directly, and the framework's fallback
has always worked. Only `test_the_env_variable_reaches_the_key_the_framework_reads`
covers the actual fix, which is why it exists and why removing
`config/trustedproxy.php` fails that one and nothing else.

**The lesson worth carrying:** a setting whose absence is legal cannot be
verified by reading the code that consumes it. Somebody has to set it and watch
the effect. Nobody had.
---
## Conventions

- **A new message the API can return goes into `lang/en/` *and* `lang/es/`**
  (C1.18). A missing key does not fail — it falls back to English and ships as
  an English sentence inside a Spanish screen — so `ApiLocaleTest` compares the
  two key sets, checks Laravel's own `validation.php` against the framework's,
  and flags a Spanish row left as the English text pasted across. The `error`
  code beside a message is **not** a translation: it is the contract, the client
  branches on it, and it never changes. Reach for `Clock::time()` rather than
  `format('h:i A')`: the meridiem is a translated string now.
- **A new string goes into `mobile/lib/l10n/app_en.arb` *and* `app_es.arb`**
  (B6.2). English is the template, so a key missing from the Spanish file falls
  back to the English text and nothing fails — which is exactly why
  `test/locale_test.dart` reads gen_l10n's own `untranslated.json` and fails
  when it is not empty, and separately catches a row left as the English
  sentence pasted across. Reach the strings with `context.t` (trap 19 says where
  not to). `lib/l10n/generated/` is build output and git-ignored: `flutter pub
  get` and every build regenerate it, so a fresh clone needs no extra step.
- **Editing Blade files: use a `php <<'PHPEOF'` heredoc**, not the Edit tool and
  not inline `php -r`. The templates are tab-indented and the strings do not
  round-trip; nested quotes break in Git Bash.
- **Attendance is append-only.** Edit and delete throw; punches are voided
  instead, and every write records actor, source, IP and a full snapshot.
  `ActivityLog` and `AttendanceAuditEvent` refuse updates and deletes too.
  **Deleting an *employee* was the way around this**: `attendance_logs.employee_id`
  is `ON DELETE CASCADE`, so removing somebody took every punch they ever made,
  including the ones a finished payroll run was calculated from. Deletion now
  refuses anyone with history and points at `status = terminated` instead.
  Employees still have no soft delete, so the refusal is the only thing standing
  between a mis-click and a hole in the audit trail — leave it in place.
- **The login throttle fires `Lockout`, not a 429.** `LoginController` counts
  attempts itself rather than wearing `throttle` middleware, because
  `AppServiceProvider` already listens for Laravel's `Lockout` event and writes
  the audit row that the Security panel's "Lockouts (24h)" tile counts. Route
  middleware would block the requests and leave that tile reading zero straight
  through an attack. The counter is keyed on **email *and* IP**: on email alone
  one person's fat fingers would lock out a colleague behind the same office NAT
  address; on IP alone the whole office shares one budget.
- **The offline cache only ever answers for a request that did not arrive.**
  `OfflineCache.fetch` falls back to the saved copy on
  `ApiException.isNetworkFailure` and rethrows everything else, because a
  refusal *is* an answer: serving yesterday's roster over today's 403 hides an
  account that has just lost its employee record. Two more rules go with it.
  **Nothing that takes a decision is cached** — a leave balance from disk talks
  somebody into booking days they no longer have, and an approvals inbox offers
  a manager a request that was settled an hour ago. And **every saved copy is
  labelled on screen** with when it was taken (`OfflineBanner`); a roster that
  is quietly three days old is worse than no roster, because nobody is given a
  reason to doubt it. Today's clock screen has a third rule of its own — it is
  refused unless its `date` is still today, or it would greet somebody with
  "clocked in since 09:00" from yesterday evening.
- **The cache holds PII and is a plain file, so it is cleared with the token.**
  `Session._clearToken` clears it alongside the punch queue, and `restore()`
  clears it when there is no token at all — which is the only sweep that
  catches a session ended by `logout-all` on another device. It never holds the
  bearer token itself: `_cacheProfile` writes the `user` object only, and the
  login response it comes from carries a token beside it. There is a test that
  greps the file for one.
- **Every new policy defaults to off.** `session_idle_timeout_minutes` (0),
  `enforce_geofence` (false), `require_two_factor_for_staff` (false). Each would
  otherwise change behaviour for a working installation on upgrade. They live in
  `Company::POLICY_DEFAULTS` and are edited at `/settings/policies`.
- **Reports return a uniform shape** — `title, subtitle, tiles, headings, rows` —
  so the screen, the PDF and the Excel export are all generic. Row keys must
  match `headings` exactly or the exports throw.
- **The API-docs test walks the route table.** A new endpoint fails the suite
  until it is written up in `API-Reference_v1.md`. That is deliberate.
- **`APP_NAME` is `KEMP`**, and it lives in `.env`, which is gitignored. Only
  `hrms/.env.production.example` carries it in the repository, so an install
  that skips it reads "Laravel" everywhere. It is also the TOTP issuer:
  changing it relabels **new** 2FA enrolments only — existing ones keep the old
  label and keep working, because the shared secret is untouched. It was
  `Klutch Cleaning - Employment Management Portal (EMP)` until the KEMP
  rebrand, so anybody already enrolled still sees that in their authenticator.
- **`MAIL_FROM_NAME` must not inherit `${APP_NAME}`**, even now that the name is
  short enough that it could. The From column should say who is writing, and to
  somebody opening a leave decision on their phone that is their employer, not
  the software. Set it by hand to `Klutch Cleaning`.
- **`App\Support\SqlDumper` must stay free of the container.** `SqlDumperTest`
  is a plain `PHPUnit\TestCase` with no application booted, so a `config()` call
  anywhere in that class dies with *Target class [config] does not exist* and
  takes three backup tests with it. The product name in the dump header is a
  literal for exactly that reason, and a backup must not need a booted framework.
- **The logo `<img>` tags carry explicit `width` and `height`.** The assets are
  PNGs, and above 992px there is **no CSS width for `.logo img` at all** — the
  old SVGs sized themselves through their intrinsic `width="250"`. Swap in an
  asset without those attributes and it renders at natural size and blows the
  sidebar open. `logo-small.png` is a black badge with the mark knocked out
  white because it is the one element shown in *both* light and dark
  mini-sidebar; the template has no dark variant for it.

---

## The four roles

Admin, HR, manager, employee — 18 permissions, all seeded by
`RolePermissionSeeder`.

| | Admin | HR | Manager | Employee |
|---|---|---|---|---|
| Lands on | `/dashboard` | `/dashboard` | `/manager/dashboard` | `/employee/dashboard` |
| Roles, policies, activity log, settings | ✅ | ❌ | ❌ | ❌ |
| Employees, reports, leave register | ✅ | ✅ | ❌ | ❌ |
| Own team: dashboard, attendance, roster, reports | — | — | ✅ | ❌ |
| Team approvals | — | — | ✅ | ❌ |
| Clocks in through the portal | ❌ | ❌ | ✅ | ✅ |

**`manager` is a role *and* a relationship, and both must line up.** The role
grants the gate; `employees.manager_id` decides the scope. Role but no reports →
empty team, not an error. Reports but no role → 403.

The admin app is wrapped in `role:admin|hr`, which runs **before** any
`permission:` middleware on the route inside it. Adding `|approve-leave` to a
route in that group advertises manager access that can never be reached — the
manager holds the permission but not the role.

### The manager area (`/manager/*`)

A parallel route group, **not** part of the admin app, for exactly the reason
above: `role:admin|hr` refuses a manager at the door whatever permission they
hold, so the only way the role can be reached is a group of its own. Gated
`role:manager` **and** `permission:view-team` — the role decides who is in, the
permission decides whether the area exists at all, so the roles editor can
withdraw it without anybody editing the route table. (`view-team` was seeded
from the start and wired to nothing until this area existed.)

Neither gate knows *whose* team. That is **`App\Services\ManagerScope`**, and
every manager query goes through it:

```php
$this->scope->team($manager)            // active direct reports, ordered
$this->scope->teamIds($manager)         // ids, for the whereIn
$this->scope->assertManages($m, $emp)   // 403 unless they report to $m
```

**The scope is direct reports only, and does not recurse.** Leave approval is a
single hop — manager, then HR — which is what `LeaveService::managerApprove`
models, so a manager two levels up is not in the chain and showing them those
records would hand them data they can never act on. Subtree visibility is a
different feature and belongs to whoever also changes the approval chain.

No new tables were added: `employees.manager_id` already carries the reporting
line. A manager↔office or manager↔department pivot would be a second claim on
the same fact, and the two would eventually disagree about who owns whom.

Managers are **read-only** outside approvals — deliberately, not by omission:

- Correcting a punch is `manage-attendance`; attendance is append-only and every
  write records an actor. The routes back are the employee's regularisation
  request (A4.13) or HR's correction screen (A4.12), both of which leave a trail.
- Planning the roster is `manage-shifts`. One planner, one publish step.
- Exporting is `export-reports`. The manager reports print instead.
- The HR-grade PII on an employee record — national ID, home address, date of
  birth, emergency contact, the document vault — is behind `manage-employees`
  and never rendered in `/manager`. A supervisor needs to know who is on shift.

**Two shells, one approvals inbox.** `LeaveApprovalController` serves both
`employee.approvals.index` and `manager.approvals.index`; the view picks its
layout from a `$layout` variable. The approve/reject **writes stay on the portal
routes only** — one write path, already scoped, rather than two places for that
check to be forgotten.

**`homeRoute()` now has three answers, and `landing()` must not assume two.**
It used to compare against `'employee.dashboard'` and send everything else to
`route('dashboard')`, which sent managers somewhere `role:admin|hr` refuses — a
sign-in ending in a 403. `landing()` tests the roles directly now. An admin who
also manages a team lands on `/dashboard`: the bigger screen wins.

**`shiftOn()` is roster-aware but *not* publish-aware.** It reads
`shift_assignments` directly, so it will happily return an unpublished draft's
shift. Anything whose contract is "published only" — the team roster, the app's
`/team/roster` — must fall back to `$employee->shift` (the standing shift) when
there is no *published* assignment, never to `shiftOn()`. That leaked a draft
shift's name onto the roster before it was caught.

**Every mobile API call needs an employee record.** `ApiController::employee()`
aborts 403 "No employee record is linked to this account". A hand-created admin
has no employee row, so it signs in and then 403s on nearly everything.

**An employee record and a login are separate rows, deliberately** — plenty of
staff are on the payroll and never touch the system. The link is
`employees.user_id`, and it is made on the employee's own page under **Sign-in
Account** (`EmployeeAccountController`). Which roles that screen offers depends
on the viewer: `manage-employees` grants `employee` and `manager`, and the
elevated `hr` and `admin` need `manage-roles`. Without that split HR — who hold
`manage-employees`, because onboarding is their job — could mint an account,
make it an admin and sign in as one. The same rule runs in reverse, so HR cannot
demote an existing administrator either.

---

## Where things stand

The web dashboard is **complete except for four deliberate omissions**. The
mobile app and the API are done — B2.8, the launcher quick action, was the last
buildable row on either half, and since it landed **every remaining ⬜ on the
board is parked by decision rather than outstanding**.
`Feature-List_Web-and-App.md` is the live status
board — read it first — and `hrms/config/roadmap.php` drives the phase panel on
the Settings screen.

**Not built, by decision:**

- **AI assistant** (Part D) — out of scope, parked.
- **Multi-company tenancy** (A2.10) — the schema is company-scoped throughout, so
  this is a routing and onboarding job rather than a data-model one.

  **That sentence is now tested rather than asserted.** It had been the premise
  the whole feature rests on and had never been checked.
  `tests/Feature/CrossCompanyIsolationTest` stands up two companies — each with
  an administrator, a manager and a direct report — and attempts **36 real
  crossings**: as an administrator (reads, edits, deletes), as a manager (team
  screens and decisions), as an employee (the self-service withdrawals, where
  the guard is ownership rather than company), over the API, and as a user with
  no company at all. Plus listings, which must not merely refuse but must not
  *contain* the other company's rows — and the `team/*` endpoints, which take no
  id, so what is tested there is an absence rather than a refusal. All hold.
  Mutation-checked each time it grew: pulling a guard out fails exactly the
  tests that should. **Add to it when you add a route that takes a bound model**
  — three different guard idioms are in use (`authorizeCompany`,
  `authoriseCompany`, a bare `abort_unless`), plus ownership and team checks, so
  reading the neighbouring controller is not a substitute for a test.

  **What is genuinely left is smaller than the ⬜ suggests** — the full working
  is in `Multi-Company_Tenancy-Assessment.md`. Two companies can already be
  created (`emp:install --force`, or `--company-id=N` to attach an admin to an
  existing one) and administered separately. What remains:

  1. ~~Close the `?? Office::value('company_id')` fallback.~~ **Done
     2026-09-15.** `companyId()` now lives once, on the base `Controller`, and
     **fails closed**: a user with no company gets a 403 saying so, not
     whichever company owns the first office row. The 19 identical copies of the
     method are gone. **Call `$this->companyId()` — never re-derive it**; the
     old line answered `200` with another company's dashboard, which is why it
     was worth closing while `users.company_id` still happened to be set
     everywhere.
  2. **A product decision, not an engineering task**: creating a company is
     CLI-only and there is deliberately no sign-up route — a public "create your
     company" form on the client's own server would let anyone create tenants on
     it. CLI-only, invite-only or open sign-up is the client's call, and
     building the wrong one is worse than building none.
  3. Spatie's `roles`/`permissions` carry no `company_id`, so all companies share
     one set. Defensible — the four roles and 19 permissions mean the same thing
     everywhere — but record it as a decision rather than leaving it a discovery.
- **Conditional rules engine** (A2.9, A6.6) — the policies are configurable, but
  there is no if-this-then-that builder.
**Built since, and this list used to say otherwise:** the **2FA QR image** was
recorded here as blocked, on the grounds that composer could not resolve a new
dependency because of a `league/commonmark` advisory. On 2026-09-14 composer
resolved `bacon/bacon-qr-code` on the first attempt, and upgraded
`league/commonmark` 2.8.3 → 2.10.1 and `maatwebsite/excel` 3.1.69 → 3.1.70 in
the same pass; `composer audit` is clean. The setup screen now draws the
`otpauth://` URI as inline SVG beside the typed key — see A1.7 in
`Feature-List_Web-and-App.md`. `App\Support\Totp` is still hand-rolled and still
verified against the RFC 6238 vectors, and does not reference the QR package.

A blocker nobody re-tested outlived the thing blocking it. Re-read any note of
that shape against the tool before planning around it.

**The API answers in the caller's language** (C1.18). `SetApiLocale` reads
`Accept-Language` on the API group only, so a web request is untouched. Three
things stay in the language they were typed in, because they are data rather
than vocabulary: leave types, office and department names, and the maintenance
message on `GET /app/status`. `/privacy` and `/account-deletion` are web pages
and are English too.

**Inert until configured — neither is a code change:**

- `MAIL_MAILER` is still `log`. Password resets, leave decisions, scheduled
  reports and document-expiry warnings are all built and tested, and all go
  nowhere until real SMTP is set.
- Push is silent until a Firebase project exists. See `Push-Notifications_Setup.md`.

---

## Deploying

**The app is live at `https://hrams.devonlinetestserver.com`** — permanent, not a
staging step on the way to somewhere else. It is managed webspace (CloudPanel,
Varnish in front, no systemd), so the queue and the scheduler are cron-driven
and `HOSTING_MODE=managed` is what keeps `emp:preflight` from failing a healthy
cron queue on a daemon's five-minute tolerance.

Ship a revision with:

```bash
cd /home/devonlinetestserver-hrams/htdocs/hrams.devonlinetestserver.com
ALLOW_NON_PRODUCTION=1 bash deploy/deploy.sh
```

**The flag is required and is not a workaround.** That box's `.env` says
`APP_ENV=staging`, and `deploy.sh` refuses to run rather than guess which
database to migrate. Leave it saying staging: setting it to `production` would
also switch preflight from advisory to blocking, and it would then fail the
deploy on `MAIL_MAILER=log`.

- `Deployment-Guide_Production.md` — the runbook.
- `deploy/` — nginx config, the systemd worker unit, the cron line, `deploy.sh`.
- `hrms/.env.production.example` — the env template.
- `php artisan emp:preflight` — gates a deploy. Fails on debug-on,
  `MAIL_MAILER=log`, a localhost or http `APP_URL`, the sync queue, no recent
  backup, a bad company timezone, the demo panel left on, seeded passwords, and
  **a critical or high dependency advisory** (`composer audit`, run inside the
  command). Medium and low advisories warn instead — blocking an urgent fix on a
  low-severity advisory in a dev-only tool is how a check gets ignored. It never
  fails because it *could not* look: composer missing, no network or a timeout
  all warn and say which, since the advisory database is fetched over the wire
  and plenty of boxes have no outbound access.

**Preflight is not green on that box, and the failures are real.** As of the
first deploy there: the demo quick-login panel is ON at a public URL and
`admin@hrms.test` still carries the seeded password `password`, which together
are one click to full admin for anybody who finds the URL; `MAIL_MAILER` is
`log`; and `TRUSTED_PROXIES` is unset behind Varnish, so every punch records the
proxy's IP instead of the employee's and neither the audit trail nor the IP
column in exports is telling the truth. All three are `.env` lines plus a
`config:cache`.

**`TRUSTED_PROXIES` is only an `.env` line as of 2026-09-15** — before that it
was an `.env` line plus a bug, and setting it would have changed nothing. See
trap 36: the value was read in `bootstrap/app.php`, which runs before .env is
loaded, so it had never once been true. It now lives in
`config/trustedproxy.php`, which is where the framework's own middleware looks
and the only place `env()` survives `config:cache`. Whoever sets it on that box
should confirm the effect rather than assume it: make a punch and read
`attendance_logs.ip_address`.

**`db:backup --verify` cannot verify on this host.** The panel's database user
cannot `CREATE DATABASE`, so the scratch restore is skipped with a warning and
the dump is written but unproven. That is handled — the command warns rather
than failing the deploy — but it means the nightly dumps are not actually known
to restore. Prove one by hand occasionally on a machine that can.

**The database dump is not the whole backup.** Contracts, ID scans and employee
photos live on disk (`storage/app/employee-documents/`, `storage/app/public/avatars/`).
A restored database with no files behind it is a list of documents that all 404.

**`public/storage` must exist.** Without the symlink every employee photo 404s
with nothing in the log and no error on screen. `deploy.sh` runs `storage:link`
and preflight checks for it.

**There is no asset build step, deliberately.** The UI is the SmartHR Bootstrap 5
template served straight out of `public/assets`; no view uses `@vite`. Laravel's
default `package.json`, `vite.config.js`, `resources/js` and `resources/css` were
scaffolding nothing imported, and were deleted so nobody deploys expecting an
`npm ci && npm run build` that would produce nothing. Deployment is PHP only.

---

## Git

`origin` is `github.com/jasonapp2244/hr_-and_attendance_management_web_application`.

`git push` can hang on a hidden credential-manager dialog —
**`GIT_TERMINAL_PROMPT=0 git push`** completes instantly. `gh` is not installed,
so PRs must be opened through a browser link.
