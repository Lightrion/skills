# SDK API to AOSP file mapping

A cheatsheet for finding the AOSP source behind common Android SDK APIs.
When you need to look up what an SDK API actually does, these mappings
get you to the right file fast.

## API to platform location

| App developer asks about... | Look in AOSP at... |
|------------------------------|--------------------|
| `Activity` lifecycle | `frameworks/base/core/java/android/app/Activity.java` (the SDK side) and `frameworks/base/services/core/java/com/android/server/am/ActivityTaskManagerService.java` (server side) |
| `Service` (incl. foreground) | `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java` |
| `BroadcastReceiver` delivery | `frameworks/base/services/core/java/com/android/server/am/BroadcastQueue*.java` |
| `ContentProvider` | `frameworks/base/services/core/java/com/android/server/am/ContentProviderHelper.java` |
| `WorkManager` (Jetpack) | App-side: AndroidX repo. Platform-side: `frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java` |
| `JobScheduler` | `frameworks/base/services/core/java/com/android/server/job/...` |
| `AlarmManager` | `frameworks/base/services/core/java/com/android/server/alarm/AlarmManagerService.java` |
| `PackageManager` queries | `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java` |
| Runtime permissions | `frameworks/base/services/core/java/com/android/server/pm/permission/...` |
| `NotificationManager` | `frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java` |
| Doze and standby buckets | `frameworks/base/services/core/java/com/android/server/DeviceIdleController.java` and `frameworks/base/services/core/java/com/android/server/usage/AppStandbyController.java` |
| `ConnectivityManager` | `frameworks/base/services/core/java/com/android/server/ConnectivityService.java` |
| `LocationManager` | `frameworks/base/services/core/java/com/android/server/location/LocationManagerService.java` |
| Battery and power | `frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java` |
| `WindowManager` (window flags, insets) | `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java` |
| `InputManager` (touch, keyboard) | `frameworks/base/services/core/java/com/android/server/input/InputManagerService.java` |
| `MediaSession`, audio focus | `frameworks/base/services/core/java/com/android/server/audio/AudioService.java` and `MediaSessionService.java` |
| `ShortcutManager` | `frameworks/base/services/core/java/com/android/server/pm/ShortcutService.java` |
| Bound services (`bindService`) | `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java` (look for `bringUpServiceLocked`) |

## targetSdk to AOSP version

When you want to read the code that runs on a device with a given API
level, look at the matching AOSP release:

| Android API level | AOSP version | Latest release tag  |
|-------------------|--------------|---------------------|
| 33 (Android 13)   | 13           | android-13.0.0_r84  |
| 34 (Android 14)   | 14           | android-14.0.0_r75  |
| 35 (Android 15)   | 15           | android-15.0.0_r36  |
| 36 (Android 16)   | 16           | android-16.0.0_r4   |
| 37 (Android 17)   | 17           | android-17.0.0_r1   |

Note: a device running Android 14 has some patches applied from later
quarterly releases (r75 vs the initial r1). For most app-dev
purposes, the differences within a major version are minor; the
release tag indexed by Lightrion is the most recent that captures
the bulk of fixes.

If you need to target an intermediate minor release (e.g. r50 of
Android 14, to investigate a specific monthly security patch), pass
`release_tag="android-14.0.0_r50"` to `search_code`. Per-minor-release
queries are MCP-only.

## "SDK side" vs "server side"

Most Android APIs you call from your app are thin facades that
forward to the system_server via Binder IPC. Knowing this dichotomy
saves time:

- **SDK side** (`frameworks/base/core/java/android/...`): the class
  you import in your app. Usually just unpacks arguments, calls
  through a Binder proxy, and returns the result. Looking here is
  rarely useful for understanding behavior.

- **Server side** (`frameworks/base/services/core/java/com/android/server/...`):
  the actual implementation. This is where the platform makes
  decisions, enforces policies, applies throttling, etc. **This is
  where to read** if you want to understand what really happens.

When you search Lightrion for the SDK class name, you'll usually see
results from both sides. The `services/core` ones are the substantive
ones.

## Behavior changes to watch for

For each major version, here's a non-exhaustive list of platform
behavior changes that frequently surprise app developers:

### Android 14 (API 34)

- Foreground services require declared types in the manifest.
- Implicit intents to internal components are blocked.
- BroadcastReceiver delivery may be batched for non-ordered broadcasts.
- `ACTION_BOOT_COMPLETED` not delivered to apps that haven't been
  opened since install.

### Android 15 (API 35)

- 16 KB page size support — native code may behave differently.
- Partial screen sharing API formalized.
- Stricter foreground service restrictions for shorter-running types.

### Android 16 (API 36)

- More predictive cache management for background apps.
- Updated permission grant flows for some sensitive permissions.

### Android 17 (API 37)

- Most recent major. Behavior changes catalog is still emerging — many
  LLMs predate this release entirely. When a user mentions targeting
  API 37 and asks why behavior differs from API 36, search Lightrion
  with `version="17"` rather than relying on training knowledge.

When the user says "this used to work, now it doesn't", check whether
they updated their `targetSdk` recently and whether the API they're
using is affected by the version's behavior changes.
