# ConnectLite — starter project

An Android messaging app with email/password + Google login, dark/light mode,
file & image sharing, 1:1 video calling, and a hidden admin panel.

**What this is:** a real, working source-code starter you open in Android
Studio — not a compiled APK. It compiles conceptually and follows correct
patterns, but you should expect a few small fixes during your first Gradle
sync (normal for any hand-written project), and there's real engineering
still ahead (message backend wiring, testing, Play Store prep, signing).

---

## 1. Open the project
Open the `ConnectLite/` folder in Android Studio (Koala or newer) and let it
sync. It will fail to build until you add the two things below.

## 2. API keys — where they go and how to get them

### 🔑 Firebase config file (`google-services.json`)
This single file carries your Firebase project's public identifiers (not a
secret key you need to hide — it's meant to ship inside the app).

- **Get it:** [Firebase Console](https://console.firebase.google.com) → Create
  a project → Add an Android app with package name `com.lite.connect` → download
  `google-services.json`.
- **Place it at:** `app/google-services.json`
- While there, also turn on:
  - **Authentication → Sign-in method → Email/Password** (enable)
  - **Authentication → Sign-in method → Google** (enable) — this is required
    for the "Continue with Google" button to work, and is what auto-generates
    `default_web_client_id` into your strings once you sync Gradle again.
  - **Authentication → Settings → your app's SHA-1/SHA-256 fingerprints** —
    Google Sign-In will silently fail without this. Get your debug SHA-1 by
    running `./gradlew signingReport` in the project root.
  - **Firestore Database** (create in production mode) and **Storage** (enable).

### 🔑 Agora App ID (video calling)
- **Get it:** [Agora Console](https://console.agora.io) → Project Management
  → Create a project → copy the **App ID** shown (not the App Certificate).
- **Place it:** copy `local.properties.example` → `local.properties`, then set:
  ```
  AGORA_APP_ID=your_app_id_here
  ```
  `local.properties` is git-ignored, so this never gets committed.
- Note: joining calls with a `null` token (as this starter does) only works
  while your Agora project is in **testing mode**. For a public release,
  generate short-lived RTC tokens from a Cloud Function using your Agora App
  Certificate — never embed the App Certificate in the app itself.

### 🔑 Admin setup secret (Cloud Functions)
Used once to grant yourself admin access. Before deploying functions:
```
firebase functions:config:set admin.setup_secret="pick-something-long-and-random"
```

**Send me any of the values above (or the `google-services.json` contents) and
I'll drop them straight into the right files for you** — none of these are
meant to be kept secret from your own codebase, though I'd still avoid
pasting the Agora App Certificate or the Cloud Functions admin secret into a
public/shared chat.

## 3. Deploy the backend
```
cd functions
npm install
firebase deploy --only functions,firestore:rules
```
This deploys `registerUser` (enforces the 1-account-per-7-days rule) and
`grantAdminOnce`, plus the Firestore rules that actually restrict who can
read every user's data.

## 4. Make yourself an admin
1. Register a normal account in the app with the email you want to use as admin.
2. Call `grantAdminOnce` once (e.g. via `firebase functions:shell` in the
   `functions/` folder) with your `uid` (find it in Firebase Console →
   Authentication) and the `setupSecret` you configured above.
3. In the app, go to **Settings → hold down "ADMIN LGN"** at the bottom → log
   in with that same email/password → you'll land on the Admin Panel showing
   every registered user's email, phone, and sign-up date.

## 5. Before you ship this publicly
- **Add a real launcher icon** (Android Studio → right-click `res` → New →
  Image Asset). The manifest currently points at a system placeholder.
- **Write a privacy policy.** You're collecting emails and phone numbers and
  giving an admin visibility into them — most app stores require a disclosed
  privacy policy for this, and depending on your users' location you may have
  obligations under laws like GDPR or CCPA (data access/deletion requests,
  lawful basis for storing phone numbers, etc.). This isn't legal advice —
  worth a real look before launch.
- **The 7-day registration limit is a soft deterrent, not a hard wall** — it's
  keyed on email, so someone determined can still use a new email address
  each time. Real anti-abuse (device attestation, phone verification, etc.)
  is a deeper project on its own.
- **Wire up real messaging.** This starter ships the chat *UI* (input bar,
  file attach, call button) but not a live message backend — the natural next
  step is a Firestore collection per room with a real-time listener.

## Project layout
```
app/src/main/java/com/lite/connect/
  ui/splash        – animated launch screen
  ui/auth          – login & register screens, Google Sign-In
  ui/home          – chat list
  ui/chat          – message thread + file attach + call button
  ui/call          – Agora video call screen
  ui/settings      – dark mode / notification toggles + hidden admin entry
  ui/admin         – admin login + user list panel
  data/repository  – Firebase Auth / Firestore / Storage wrappers
functions/         – Cloud Functions (registration throttle, admin claim)
firestore.rules    – who can read/write what
```
