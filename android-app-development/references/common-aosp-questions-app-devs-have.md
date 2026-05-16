# Common AOSP-rooted questions app developers ask

A reference of common scenarios where an Android app developer needs
to peek into AOSP source to fully understand what's happening, along
with example workflows.

## "Why is my BroadcastReceiver not being called?"

### Likely platform-side causes

1. **Implicit broadcast restrictions** — since Android 8 (and tightened
   in 14), receivers registered in the manifest don't receive most
   implicit broadcasts unless explicitly exempted.
2. **Package visibility restrictions** — since Android 11, your app
   needs `<queries>` in the manifest to see packages.
3. **App standby bucket** — if your app is in a restricted bucket,
   broadcasts may be delayed or dropped.
4. **Boot-completed restrictions** — `ACTION_BOOT_COMPLETED` requires
   the user to have opened your app since install (Android 14+).
5. **Foreground state requirements** — some broadcasts only deliver
   to foreground apps now.

### Workflow

```
search_code(
    query="BroadcastReceiver delivery restrictions package visibility",
    version="<user's targetSdk version>"
)
```

Look in `BroadcastQueue*.java` for the actual delivery logic. The
filtering happens before your receiver is even consulted.

## "Why does my foreground Service get killed?"

### Likely platform-side causes

1. **Type mismatch** — Android 14 requires the service type declared
   in the manifest to match the actual work (e.g.,
   `dataSync`, `mediaPlayback`). Mismatches lead to kills.
2. **Short-service timeout** — Android 14 introduced a 3-minute
   timeout for `shortService` type.
3. **User-initiated requirement** — some types require the service
   to be started from a foreground app interaction.
4. **Memory pressure** — system_server may still kill foreground
   services under extreme pressure; check `lowmemorykiller` logic.

### Workflow

```
search_code(
    query="foreground service type validation timeout kill",
    version="14"  // or wherever the user's problem started
)
```

Look in `ActiveServices.java` for the validation and lifecycle.

## "Why is WorkManager not running my work?"

### Likely platform-side causes

1. **JobScheduler constraints** — WorkManager translates Work to
   Jobs. Network, charging, idle constraints all impact dispatch.
2. **Doze** — at Doze maintenance windows, only jobs matching certain
   criteria run.
3. **App standby bucket** — restricted buckets cap jobs per day.
4. **Battery saver** — when active, many job categories are deferred.

### Workflow

```
search_code(
    query="JobScheduler Doze constraint check exemption WorkManager",
    version="<user's targetSdk version>"
)
```

Look in `JobSchedulerService.java` and `DeviceIdleController.java`
for the actual decision logic.

## "Why is my permission grant being denied silently?"

### Likely platform-side causes

1. **App ops** — runtime permissions are gated by AppOps as well as
   the permission flag. Even a granted permission can be revoked at
   the AppOps level.
2. **One-time permissions** — Android 11+ "Only this time" grants
   expire on app process death.
3. **Auto-reset** — Android 11+ auto-revokes permissions for unused
   apps after several months.
4. **Permission re-prompts limits** — the user can permanently deny
   after two declines.

### Workflow

```
search_code(
    query="runtime permission grant AppOps auto-reset one-time",
    version="<user's targetSdk version>"
)
```

Look in `PermissionManagerService.java` and `AppOpsService.java`.

## "Why is the user seeing 'This app keeps stopping' on my service?"

This is an ANR (Application Not Responding). The system detected your
main thread or a binder thread didn't respond within a timeout.

### Likely platform-side causes

1. **Main thread block** — anything I/O on main thread.
2. **Binder timeout** — your service didn't respond to a binder call
   in time.
3. **BroadcastReceiver timeout** — Receiver's `onReceive` exceeded
   10 seconds (foreground) or 60 seconds (background).

### Workflow

```
search_code(
    query="ANR detection timeout binder broadcast receiver",
    version="<user's targetSdk version>"
)
```

Look in `AnrHelper.java` or `ActivityManagerService.java` (search for
`appNotResponding`).

## "What does targetSdk X actually change?"

The "behavior changes" page in the Android docs lists the changes,
but if the user wants to read the actual code that gates the new
behavior, they're looking for `CompatChanges.isChangeEnabled()` calls
in the platform.

### Workflow

```
search_code(
    query="CompatChanges isChangeEnabled <feature name>",
    version="<user's target>"
)
```

Each behavior-change is gated by a change ID that's checked against
the calling app's targetSdk. Reading the gates tells you exactly when
your app is affected.

## "How does this Jetpack/AndroidX library actually work?"

Jetpack libraries (WorkManager, Lifecycle, Compose, etc.) are
**not in AOSP**. They live in the AndroidX repo (a separate Google
codebase, also open source). Lightrion does not currently index
AndroidX.

When the user asks about Jetpack internals, point them at:

- The AndroidX source on cs.android.com/androidx/platform/frameworks/support
- The class's `@param` and `@throws` JavaDoc in the SDK
- For WorkManager specifically: it builds on top of JobScheduler (which
  **is** in AOSP), so understanding JobScheduler answers many "why is
  my Work delayed" questions.

Don't try to search Lightrion for AndroidX class names — you'll get
no relevant results.
