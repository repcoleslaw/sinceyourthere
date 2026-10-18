---
name: Location Task Reminder App
overview: Build a cross-platform (iOS and Android) task app with Expo and Supabase. Each task is tied to a store chain like "Target", and you get a reminder notification whenever you come near any branch of that chain.
todos:
  - id: scaffold
    content: Scaffold Expo + TypeScript + Expo Router project, configure app.config.ts permissions and EAS dev builds
    status: pending
  - id: emulator-setup
    content: Install Android Studio, create a Google Play Pixel emulator, run first dev build with npx expo run:android
    status: pending
  - id: supabase
    content: "Set up Supabase project: auth, tables (chains, tasks, branch_cache), RLS policies"
    status: pending
  - id: edge-fn
    content: Write nearby-branches Edge Function using Google Places Text Search with caching
    status: pending
  - id: ui
    content: Build sign-in, task list, task create/edit with chain picker, settings screens
    status: pending
  - id: local-cache
    content: Add expo-sqlite local cache and sync with Supabase
    status: pending
  - id: geofence
    content: "Implement background geofence task: refresh algorithm, 19 nearest + refresh region, enter handler with cooldown"
    status: pending
  - id: notifications
    content: Wire expo-notifications local notifications, staged permission flow, and dev-only Simulate arrival button
    status: pending
  - id: device-test
    content: Build with EAS and field-test on iPhone (TestFlight/ad-hoc) and Pixel
    status: pending
isProject: false
---

# Location-Based Task Reminder App

## Stack
- **App:** React Native with Expo (TypeScript), Expo Router for navigation, built as an Expo development build. Expo Go can't run background location, so it won't work for this.
- **Location:** `expo-location` handles geofencing and significant-change updates. `expo-task-manager` runs that code in the background.
- **Notifications:** `expo-notifications` shows **local** notifications. The geofence fires on the phone itself, so no push server is needed. The user still sees a normal notification banner.
- **Backend:** Supabase provides authentication (email magic link or Apple/Google sign-in), Postgres with row-level security, and Edge Functions.
- **Store lookup:** Google Places API (New) Text Search, called through a Supabase Edge Function so the API key stays off the device.
- **Builds:** EAS Build for both platforms. To install on the iPhone you'll need an Apple Developer account ($99/year), then distribute through TestFlight or an ad-hoc build. On the Pixel you can install the APK directly.

## How chain-based reminders work
The hard limit is that iOS lets an app monitor at most **20 geofences** at a time; Android allows 100. Since there can be hundreds of Target stores, the app can't watch all of them. Instead it keeps re-registering the geofences for the branches closest to you:

```mermaid
flowchart TD
    move["Significant location change or exit of refresh region"] --> chains["Collect chains that have open tasks"]
    chains --> fn["Edge Function: nearby branches for those chains within ~15 km"]
    fn --> pick["Pick the 19 nearest branches"]
    pick --> reg["Register a ~150 m geofence on each, plus 1 large refresh region around the user"]
    reg --> enter["Geofence enter event in background task"]
    enter --> cache["Read open tasks for that chain from local cache"]
    cache --> cooldown{"Notified for this branch in last 2h?"}
    cooldown -->|no| notify["Local notification: 3 tasks at Target - milk, batteries..."]
    cooldown -->|yes| skip[Skip]
```

- The "refresh region" is a geofence with a radius of about 5 km centered on you. When you leave it, the background task runs again and re-registers the geofences. Together with significant-change updates, this keeps the 19 branch geofences close to wherever you are.
- Tasks and branches are cached on the device with `expo-sqlite`, so the background task works without a network connection. A per-branch cooldown prevents repeated notifications.

## Data model (Supabase)
- `chains`: `id`, `name` (for example "Target"), `places_query`, `brand_match` (the name pattern used to filter Places results).
- `tasks`: `id`, `user_id`, `chain_id`, `title`, `notes`, `done`, `created_at`. Row-level security limits each user to their own rows.
- `branch_cache`: `place_id`, `chain_id`, `lat`, `lng`, `name`, `fetched_at`. This is shared and keeps Places API costs down.
- Edge Function `nearby-branches(lat, lng, chain_ids[])` returns cached branches and fills any gaps from Google Places.

## App structure
- `app/(auth)/sign-in.tsx`: sign-in screen.
- `app/(tabs)/index.tsx`: task list grouped by chain, with a "near you" badge.
- `app/task/[id].tsx` and `app/task/new.tsx`: create and edit tasks. The chain picker searches existing chains or creates a new one.
- `app/settings.tsx`: permission status, notification test, and the cooldown setting.
- `src/lib/supabase.ts`, `src/lib/db.ts` (local SQLite cache and sync).
- `src/location/geofenceTask.ts`: the background task definitions and the refresh algorithm above.
- `src/location/permissions.ts`: a staged permission flow. It asks for foreground location, then "Always"/background location, then notifications, explaining each step.
- `app.config.ts`: iOS `NSLocationAlwaysAndWhenInUseUsageDescription`, `UIBackgroundModes: [location]`; Android `ACCESS_BACKGROUND_LOCATION` and `POST_NOTIFICATIONS`.

## Platform caveats
- iOS only delivers geofence events if the user grants "Always" location access. Android needs "Allow all the time", and the Pixel's battery optimization can delay events by a minute or two.
- If the user force-quits the app on iOS, geofence events still relaunch it. Android does the same unless the app is force-stopped from settings.
- A ~150 m radius is about as small as is reliable. Accuracy is typically 50 to 200 m.

## Local development on Windows
- **Android Emulator (primary):** Install Android Studio and create a Pixel virtual device from a "Google Play" system image (needed for Play Services geofencing). Run `npx expo run:android` to build and install the dev build; after that, `npx expo start` hot-reloads code changes.
- **Faking location:** Use the emulator's Extended Controls > Location to drop a pin on a store or play back a GPX/KML route that drives past one. You can also run `adb emu geo fix <lng> <lat>` from a script.
- **Dev-only "Simulate arrival" button** in `app/settings.tsx`: calls the geofence enter handler directly for a chosen branch, so you can test notification logic instantly without waiting on real geofence events.
- **Backend:** Use a free hosted Supabase project (no Docker needed). Run Edge Functions locally with `supabase functions serve` if Docker is available, otherwise deploy them to the hosted project.
- **iOS:** The iOS Simulator only runs on macOS, so it isn't available on Windows. Do all day-to-day work on the Android Emulator, then test iOS on the real iPhone with an EAS dev build near the end.

## Out of scope for v1
- Task sharing between users.
- Specific single-location tasks. This can be added later as a "custom place" chain with one branch.
- App Store and Play Store publishing. v1 is for personal installs.