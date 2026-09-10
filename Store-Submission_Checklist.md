# Store submission — Google Play & App Store

What the two stores require, what the repository already satisfies, and what is
left. Written against the app in `mobile/` and the Laravel server in `hrms/`.

The **blocking** item is not on this list: none of it can be submitted until the
server is deployed at a real HTTPS domain (`C1.14`). Both stores fetch the
privacy-policy URL during review, and a listing pointing at `localhost` is
rejected without a human looking at it. See `Deployment-Guide_Production.md`.

---

## Done in the repository

| Requirement | Where |
|---|---|
| Privacy policy at a public URL, no login | `GET /privacy` — `hrms/resources/views/legal/privacy.blade.php` |
| Account-deletion route, no login | `GET /account-deletion` — `hrms/resources/views/legal/deletion.blade.php` |
| Both reachable from inside the app | Profile screen → Privacy policy / Delete my account |
| Real launcher icon, all densities | `mobile/android/.../mipmap-*`, `mobile/ios/.../AppIcon.appiconset` |
| Adaptive icon (Android 8+) | `mipmap-anydpi-v26/ic_launcher.xml` + `values/colors.xml` |
| iOS icon with no alpha channel | `remove_alpha_ios: true` in `pubspec.yaml` |
| Play listing icon, 512×512 | `mobile/store/play-listing-icon-512.png` |
| App Store icon, 1024×1024 | `mobile/store/app-store-icon-1024.png` |
| Release signing separate from the debug key | `android/app/build.gradle.kts` reads `key.properties` |
| Permissions declared and used | `INTERNET`, `POST_NOTIFICATIONS`, `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` (B2.3), `USE_BIOMETRIC` (B1.3). No `<uses-feature>` for the sensor — requiring the hardware would hide the app from every device without one |
| iOS usage strings for both prompts | `NSLocationWhenInUseUsageDescription`, `NSFaceIDUsageDescription`. Missing either is a termination on a real device, not a refusal |
| Biometric data leaves nothing to declare | The check is made by the OS; the app is told yes or no and stores only a per-handset on/off flag. Neither store's data form has a row to fill in for it |
| Android 11 package visibility for links | `<queries>` https VIEW intent |
| Auth token excluded from backup and transfer | `xml/data_extraction_rules.xml`, `xml/backup_rules.xml` |
| Export-compliance answer | `ITSAppUsesNonExemptEncryption = false` in `Info.plist` |
| Apple privacy manifest | `ios/Runner/PrivacyInfo.xcprivacy` — **see caveat below** |
| Cleartext traffic blocked in release | `ApiClient.assertSecureBaseUrl()` refuses a non-https release build |
| A way to retire a shipped build | `GET /app/status` (B6.6) — a server-side minimum version and a maintenance flag, both empty/off by default. Set `MOBILE_STORE_URL_ANDROID` / `MOBILE_STORE_URL_IOS` to the real listings once they exist, or the update screen has nowhere to send anybody and the gate declines to fire |
| English and Spanish in the app | `mobile/lib/l10n/app_en.arb` and `app_es.arb` (B6.2). Every string, including the OS's own biometric prompt and the name in the task switcher. The app follows the phone's language by default and offers a picker on the Profile screen |
| Accessibility — WCAG 2.1 AA | `mobile/test/accessibility_test.dart` (B6.4). Contrast is measured, not eyeballed: every colour clears 4.5:1 on the worst surface it meets, including its own tint. The OS font size is respected up to 2× with no clipping, and every icon-only control has a name. Both stores ask; Apple's review has rejected apps for text that vanishes at the larger accessibility sizes |

---

## Left to do, and who has to do it

### 1. Add the privacy manifest to the Xcode target — needs a Mac

`ios/Runner/PrivacyInfo.xcprivacy` exists but is not referenced by
`project.pbxproj`, so it is **not in the built app**. A file in the folder is not
a file in the bundle.

Open `ios/Runner.xcworkspace`, drag the file into the `Runner` group, tick the
`Runner` target, and confirm it appears under Build Phases → Copy Bundle
Resources. This was not done here because editing `project.pbxproj` blind, with
no Xcode to verify against, risks corrupting the project file.

### 2. Create the release keystore — one-off, never committed

```bash
keytool -genkey -v -keystore ~/hrms-release.jks -keyalg RSA \
        -keysize 2048 -validity 10000 -alias hrms
```

Then `mobile/android/key.properties`:

```properties
storePassword=…
keyPassword=…
keyAlias=hrms
storeFile=/absolute/path/to/hrms-release.jks
```

`key.properties` and the `.jks` must stay out of git — anyone holding them can
sign an update Play will accept as genuine. Losing them means the app can never
be updated under the same listing.

### 3. Build against the real server

The default API base is the emulator's view of a development machine. A release
build must override it, and will refuse to start if it does not:

```bash
flutter build appbundle --dart-define=API_BASE=https://emp.klutchcleaning.com/api/v1
flutter build ipa       --dart-define=API_BASE=https://emp.klutchcleaning.com/api/v1
```

Play takes the `.aab`, not an APK.

### 4. Screenshots — needs the deployed server

Play: at least 2 phone screenshots, plus a 1024×500 feature graphic (required
for every listing). App Store: 6.7" and 5.5" iPhone sets.

The seeded demo company produces presentable screens. Take them after the server
is up, since the app cannot reach data before then.

**Both consoles list a language per set, and both let a listing declare more
than one.** The app ships English and Spanish (B6.2), so the listing should say
so — Play under *Store listing → Manage translations*, App Store under
*Localizations* — and each language wants its own screenshots. A store page in
one language for an app that opens in another is not a rejection, but it is the
reason somebody uninstalls before the first sign-in.

Switching the app for a Spanish set is a phone setting, not a build: change the
handset's language, or pick Spanish on the Profile screen.

### 5. Fill in the data-safety and privacy forms

Both must agree with `/privacy` and with `PrivacyInfo.xcprivacy` — a reviewer
compares them, and a mismatch is a rejection.

| Data | Collected | Shared | Purpose | Linked to identity |
|---|---|---|---|---|
| Name | Yes | No | App functionality | Yes |
| Email address | Yes | No | App functionality, account management | Yes |
| Employee ID, department, job title | Yes | No | App functionality | Yes |
| Attendance times, worked hours, leave | Yes | No | App functionality | Yes |
| IP address | Yes | No | Security / fraud prevention | Yes |
| Precise location | Yes — see below | No | App functionality | Yes |
| Crash logs / diagnostics | Yes — B6.5 | No | App functionality | Yes |
| Advertising ID / analytics | No | No | — | — |

Answer **no** to tracking on both forms: there is no advertising SDK, no
analytics, and the only host the app contacts is the employer's own server.

> **Crash logs (B6.5) are collected, and that answer stays "no third party".**
> Crashes are written on the handset and posted to the employer's own server,
> where an administrator reads them. There is no Crashlytics and no Sentry, and
> **there should not be**: a stack trace routinely carries fragments of whatever
> the app was holding, and sending those to a third party would falsify the "not
> shared" column above, the `/privacy` page and `NSPrivacyTracking` in one go.
> If a crash service is ever added, all four declarations change with it.

Say data is encrypted in transit (yes), and that users can request deletion
(yes, via `/account-deletion`).

> **Location — declare it, and declare it as precise.** B2.3 shipped: the app
> reads a fix through `geolocator` at the moment of a punch and sends it with
> the punch. It asks for `ACCESS_FINE_LOCATION` and
> `NSLocationWhenInUseUsageDescription`, and the declaration describes what is
> *asked for* rather than what the user then grants — so this is precise, not
> approximate, even though a punch is recorded perfectly well without any fix
> at all.
>
> It is **when in use** only. There is no `ACCESS_BACKGROUND_LOCATION` and no
> `NSLocationAlwaysAndWhenInUseUsageDescription`, which is what keeps this off
> the Play Console's sensitive-permission declaration path. Do not add either.
>
> Four descriptions of one behaviour have to agree, and a reviewer compares
> them: this row, the `/privacy` page on the server, `PrivacyInfo.xcprivacy`,
> and the data forms on both consoles. **This row said "not collected" for
> some time after B2.3 shipped, and the Apple manifest omitted the entry
> entirely** — nothing about that fails at build time, and an app that collects
> location without declaring it is the single most common cause of an
> enforcement removal. Re-read all four against the code before every
> submission rather than trusting any one of them.

### 6. Notification permission — wired

`POST_NOTIFICATIONS` is declared and the app now requests it at runtime, at
sign-in rather than at first launch, so the prompt arrives with a reason
visible. Android 13+ requires that explicit request; below 13 it is granted on
install.

Whether a notification can actually appear depends on the build: without
`google-services.json` Firebase does not initialise, nothing is registered and
nothing arrives. Both states are consistent with the declaration — the
permission may be asked for and unused.

**Before submitting a build that has the Firebase config in it**, revisit the
data-safety form: an FCM token is a device identifier, and both stores treat
"registers a push token" as data collection even when the notifications
themselves carry no personal data.

---

## Checked and deliberately not changed

- **No ATS exception on iOS.** The default blocks plain HTTP, which is what a
  release build should do. Development reaches `http://10.0.2.2` through
  `dart:io`, which does not consult ATS.
- **`allowBackup` left on.** Only the sign-in token needed excluding, and it is
  excluded by name. Turning backup off wholesale would be a bigger promise than
  the privacy policy makes.
- **Account deletion is a support route, not an in-app button.** Accounts are
  provisioned by an employer and an employee cannot delete their own attendance
  record — that is the point of an audit trail. Apple's in-app deletion rule is
  written for apps that let users *create* accounts; this one does not. The page
  says plainly who to contact and what will and will not be erased.
