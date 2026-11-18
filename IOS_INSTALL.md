# SafeRoute — iOS install context (for Claude on the Mac)

Goal: build the Flutter app in this repo and install it on a real iPhone **for free** (no $99 Apple Developer account), using a free Apple ID. Then keep it alive with AltStore (or re-run weekly from Xcode).

Please walk me through this step by step, run commands for me where you can, and stop to ask when something needs my action (Xcode GUI, iPhone settings, Apple ID).

## Project facts (verified from the repo)

- Flutter app, package name `saferoute`, Dart SDK ^3.13.5, iOS deployment target **15.0**
- iOS bundle ID: `com.mohit.saferoute` (RunnerTests: `com.mohit.saferoute.RunnerTests`)
- Backend: Firebase Auth + Firestore (Spark plan), Google Sign-In, OpenRouteService (OSRM fallback)
- iOS Info.plist already has: location usage strings, `UIBackgroundModes` = location, fetch, processing, `BGTaskSchedulerPermittedIdentifiers`, and the Google Sign-In reversed-client-ID URL scheme (`com.googleusercontent.apps.120489674470-brfgsg6ga952t6bb3q0hl3vftajgtfne`)
- Firebase iOS app id `1:120489674470:ios:a8bbc663ee95b099e39d27`, iosBundleId `com.mohit.saferoute` (in `lib/firebase_options.dart`)
- iPhone guardians get alerts via the **ntfy.sh app** (iOS can't run the always-on monitor). iPhone travelers upload location from the main isolate in the background.

## ⚠️ Step 0 — secret files are NOT in the git clone

These are git-ignored, so a fresh clone is missing them and the build will fail. I must copy them from my Linux machine to the Mac (AirDrop / USB / cloud drive, never commit them):

| File | Destination in repo |
|---|---|
| `lib/config.dart` | `lib/config.dart` (copy of `lib/config.example.dart` with real `orsKey`) |
| `lib/firebase_options.dart` | `lib/firebase_options.dart` |
| `ios/Runner/GoogleService-Info.plist` | `ios/Runner/GoogleService-Info.plist` |

(`android/app/google-services.json` and keystore are Android-only; not needed on the Mac.)

Claude: first check these three files exist; if not, tell me to copy them before continuing. If `GoogleService-Info.plist` is missing it must be added to the Runner target in Xcode (drag into Runner group, "Copy items if needed", Runner target checked) — or just placed in `ios/Runner/` if the project already references it.

## Step 1 — Mac prerequisites

Check/install, and report versions:
1. Xcode (latest from App Store) + open it once to accept license and install iOS platform: `sudo xcodebuild -license accept`, `xcode-select --install`
2. Flutter SDK (`flutter doctor` must show iOS toolchain ✓)
3. CocoaPods: `brew install cocoapods` (or `sudo gem install cocoapods`)
4. Make sure `flutter doctor` has no iOS errors.

## Step 2 — Prepare the project

```bash
cd saferoute
flutter clean
flutter pub get
cd ios && pod install --repo-update && cd ..
```
If `pod install` complains about the minimum iOS version, ensure `ios/Podfile` has `platform :ios, '15.0'`.

## Step 3 — Signing with a free Apple ID

1. `open ios/Runner.xcworkspace` (the **.xcworkspace**, not .xcodeproj)
2. Xcode → Settings → Accounts → add my Apple ID
3. Select the Runner target → Signing & Capabilities → tick "Automatically manage signing" → Team = my Personal Team
4. Bundle ID `com.mohit.saferoute` may already be taken by someone else's free account → if Xcode says it can't register it, change it to something unique (e.g. `com.mohit.saferoute2`) in **Xcode only**. NOTE: if changed, the Firebase iOS app must be re-registered with the new bundle ID, and `GoogleService-Info.plist`, `lib/firebase_options.dart` (iosBundleId/appId) and the Google URL scheme must be regenerated (`flutterfire configure`). Prefer keeping the original ID if it works.
5. Free-account limits: the "Background Modes" capability works with free accounts, but Push Notifications / some entitlements don't. The app should not need Push (alerts go via ntfy). If Xcode complains about a capability, tell me which one before removing anything.

## Step 4 — iPhone setup

1. Plug the iPhone in with a cable, unlock it, tap "Trust This Computer"
2. iPhone: Settings → Privacy & Security → **Developer Mode** → On (restarts phone)
3. Select the iPhone as the run destination in Xcode (or `flutter devices` to confirm it shows up)

## Step 5 — Build & install

```bash
flutter run --release -d <iphone-device-id>
```
(`--release` so it keeps running when unplugged; debug builds on iOS crash when launched from the home screen without the debugger.)

First launch will fail with "Untrusted Developer": iPhone → Settings → General → **VPN & Device Management** → my Apple ID → Trust. Then open the app again.

If the first build is slow (10+ min) that's normal (Firebase pods).

## Step 6 — Keep it installed (free certs expire every 7 days)

Choose one:
- **AltStore (recommended in the plan):** install AltServer on the Mac, install AltStore to the iPhone via cable, then sideload the app. To get an .ipa: `flutter build ipa --release` may refuse without a paid account, so alternative: `flutter build ios --release --no-codesign`, then
  ```bash
  mkdir Payload && cp -R build/ios/iphoneos/Runner.app Payload/ && zip -r SafeRoute.ipa Payload
  ```
  and open that .ipa in AltStore (AltStore re-signs it with my Apple ID). AltServer must stay running on the Mac on the same Wi-Fi so it can auto-refresh every 7 days.
- **Or Xcode weekly:** re-run `flutter run --release` every 7 days (max 3 free apps).
- Upgrade path: $99/yr Apple Developer → TestFlight (90-day builds, install by link).

## Step 7 — Verify on the iPhone

- Google Sign-In works (if it crashes/does nothing → URL scheme/GoogleService-Info.plist mismatch)
- Allow location → choose **Always** (Settings → SafeRoute → Location → Always, Precise On, and Background App Refresh On)
- Start a test trip as traveler, lock the phone, walk/drive a bit, confirm location updates keep arriving in Firestore (Firebase console → Firestore) and on a guardian's phone
- iPhone guardians: install the **ntfy** app from the App Store and subscribe to the topic shown in the app (see `lib/services/alert_service.dart` / `group_admin_screen.dart`) so alerts arrive
- Don't swipe-kill the app during a trip (iOS will stop background location)

## Known gotchas

- Always open `Runner.xcworkspace`, never `Runner.xcodeproj`
- iOS gives speed `-1` when unknown — app treats <0 as unknown
- Each group needs at least one **Android** guardian for fully automatic route monitoring
- Never commit `lib/config.dart`, `lib/firebase_options.dart`, `GoogleService-Info.plist`
- Package APIs change between versions; exact versions are in `pubspec.lock` — use those if debugging build errors

## If something fails

Paste the full error. Common ones: CocoaPods version/repo errors (`pod repo update`), "No profiles for com.mohit.saferoute" (bundle ID taken → Step 3.4), "Developer Mode disabled" (Step 4), Firebase pod build errors (`flutter clean && rm -rf ios/Pods ios/Podfile.lock && pod install`).
