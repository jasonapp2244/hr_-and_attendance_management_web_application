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
| Permissions declared and used | `INTERNET`, `POST_NOTIFICATIONS` only |
| Android 11 package visibility for links | `<queries>` https VIEW intent |
| Auth token excluded from backup and transfer | `xml/data_extraction_rules.xml`, `xml/backup_rules.xml` |
| Export-compliance answer | `ITSAppUsesNonExemptEncryption = false` in `Info.plist` |
| Apple privacy manifest | `ios/Runner/PrivacyInfo.xcprivacy` — **see caveat below** |
| Cleartext traffic blocked in release | `ApiClient.assertSecureBaseUrl()` refuses a non-https release build |

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
flutter build appbundle --dart-define=API_BASE=https://hr.example.com/api/v1
flutter build ipa       --dart-define=API_BASE=https://hr.example.com/api/v1
```

Play takes the `.aab`, not an APK.

### 4. Screenshots — needs the deployed server

Play: at least 2 phone screenshots, plus a 1024×500 feature graphic (required
for every listing). App Store: 6.7" and 5.5" iPhone sets.

The seeded demo company produces presentable screens. Take them after the server
is up, since the app cannot reach data before then.

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
| Approximate location | **No** — see below | — | — | — |
| Advertising ID / analytics | No | No | — | — |

Answer **no** to tracking on both forms: there is no advertising SDK, no
analytics, and the only host the app contacts is the employer's own server.

Say data is encrypted in transit (yes), and that users can request deletion
(yes, via `/account-deletion`).

> **Location:** the server stores a coordinate when a punch arrives carrying
> one, but the app has no location plugin wired and never sends one — so today
> the honest answer is *not collected*. The moment B2.3 ships, this row, the
> privacy page, `PrivacyInfo.xcprivacy` and both store forms all change
> together. An app that starts collecting location without updating its
> declaration is the single most common cause of an enforcement removal.

### 6. Notification permission — only when push is wired

`POST_NOTIFICATIONS` is declared, but nothing requests it at runtime and
Firebase is never initialised, so no notification can appear. That is consistent
today. When B5 is built, the app must request the permission explicitly on
Android 13+ and the data-safety form must be revisited.

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
