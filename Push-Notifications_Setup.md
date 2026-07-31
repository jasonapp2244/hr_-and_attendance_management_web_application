# Push notifications — what is built, and what you have to do

Server-side delivery is built and tested. It is **switched off** until Firebase
credentials exist, and does nothing at all until then — an install with no key
sends no push rather than failing every notification.

Three notifications carry a push version: a leave request arriving for a
manager, a decision reaching the employee, and the reminder for somebody still
clocked in after their shift ended. All three continue to write the in-app bell
and send email exactly as before; push is an addition, not a replacement.

---

## What you have to do

None of this can be done in code — it needs an account and a billing-capable
Google project.

### 1. Create the Firebase project

<https://console.firebase.google.com> → **Add project**. Free tier is sufficient;
FCM has no per-message charge.

Note the **Project ID** (Project settings → General). It is not a secret.

### 2. Register the Android app

Project settings → **Your apps** → Add app → Android.

Package name must be exactly:

```
com.hrattendance.hrms_mobile
```

Download `google-services.json` and place it in the Flutter repo at
`android/app/google-services.json`. It is gitignored there.

### 3. Register the iOS app (only when you have an Apple Developer account)

Same screen → iOS. Bundle ID `com.hrattendance.hrmsMobile`. Download
`GoogleService-Info.plist`.

iOS also needs an **APNs authentication key** (`.p8`) from the Apple Developer
portal, uploaded in Firebase under Project settings → Cloud Messaging. Android
works without any of this.

### 4. Generate the server key

Project settings → **Service accounts** → *Generate new private key*. A JSON
file downloads.

Put it at `hrms/storage/app/firebase/service-account.json`.

**This file is a credential.** Anyone holding it can send a notification to
every installation of the app. It is gitignored, it lives outside the web root,
and it should never be emailed or pasted into a chat.

### 5. Switch it on

In `hrms/.env`:

```env
FCM_ENABLED=true
FCM_PROJECT_ID=your-project-id
```

Then `php artisan config:clear`.

### 6. Run a queue worker

Push is queued along with email. Without a worker nothing is delivered:

```bash
php artisan queue:work
```

---

## How to tell it is working

```bash
php artisan tinker
>>> App\Models\PushDevice::count()      # handsets registered by the app
>>> config('fcm.enabled')               # true
>>> app(App\Services\Push\FcmClient::class)->configured()   # true
```

If `configured()` is false, one of these is wrong: `FCM_ENABLED`,
`FCM_PROJECT_ID`, or the credentials file is missing or unreadable.

Failures are logged to `storage/logs/laravel.log` with the reason FCM gave.

---

## What the server does on its own

**Forgets dead handsets.** When FCM answers `UNREGISTERED` — the app has been
uninstalled — that row is deleted rather than retried forever. A `503` is
treated as Google having a bad minute and the handset is kept, because deleting
it would silently unsubscribe somebody from every future notification.

**Sends to every handset a person has registered.** A phone and a tablet are two
installations and get one request each; FCM's v1 API has no batch endpoint.

**Keeps the OAuth token for an hour.** A short-lived token is minted from the
service-account key and cached, so a batch of notifications does not mint one
each.

---

## Still to do in the app

The Flutter side does not yet register a token or display an incoming
notification. That is the remaining half:

- `firebase_core` and `firebase_messaging` packages
- Ask for notification permission (Android 13+ requires it explicitly)
- Register the token with `POST /api/v1/devices` after sign-in
- Create the Android notification channel `hrms_default` — Android 8+ silently
  drops any notification whose channel does not exist
- Send the token with `POST /auth/logout` so a handset stops receiving the
  previous person's notifications
- Route a tap using the `route` value in the payload: `clock`, `leave` or
  `approvals`

The server sends `route` in every push precisely so the app can do that last
part without parsing message text.
