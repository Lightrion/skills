# Safety patterns to flag

When working on safety-relevant AOSP code — particularly AAOS automotive
code, but also permission systems, isolation boundaries, and fault
handlers in the broader platform — be alert for divergences that
indicate a regression or an inconsistent choice. These are the
high-value findings users care about.

## Pattern 1: Inverted fault defaults

The most consequential class of finding. AOSP's `CarUxRestrictionsManagerService`
defaults to the **most restrictive** restriction set when the driving
state is ambiguous. A forked or third-party product that defaults to
the **least restrictive** set is less safe under the same conditions.

Concrete example to watch for: `getCurrentUxRestrictions()` called
before the driving state is initialized.

- AAOS 15 (`android-15.0.0_r36`):
  `CarUxRestrictionsManagerService.java:928-932` returns the
  MAX_SPEED-equivalent restriction set on ambiguous state.
- A product fork: might return a Park-equivalent (less restrictive)
  set in the same code path.

If you encounter this pattern in a user's fork or downstream code,
flag it explicitly as a safety finding, not as a comment.

## Pattern 2: Removed safety nets

A check that exists in an older release but is removed in a newer one.
This can happen legitimately (refactor moved it elsewhere) or as a
genuine regression. Always confirm where the equivalent check now
lives before assuming "the platform got safer".

How to detect: search the same concept across two versions and look
for code that has no obvious analog in the newer one.

## Pattern 3: Permission gates dropped under refactor

A method that required `PERMISSION_CONTROL_CAR_INTERFACE` (or any
signature-level permission) in one release may have lost its
`@RequiresPermission` annotation in a refactor. The runtime check may
still be present elsewhere, or it may have been silently dropped.

How to detect: search for the API name across two versions, compare
the surrounding signature and annotations.

## Pattern 4: VHAL callback leaks

`CarPropertyManager.registerCallback()` paired with no
`unregisterCallback()` in the same scope leaks the callback in AOSP's
implementation. This isn't just a memory leak — for VHAL it can cause
the binder thread pool on the HAL side to bloat over time, leading
to delayed property updates.

How to detect: find `registerCallback` calls; walk the lifecycle
(onCreate/onDestroy, onResume/onPause) to confirm the matching
`unregisterCallback` exists.

## Pattern 5: Process isolation boundaries

`Process.myUid()` vs `Binder.getCallingUid()` is a classic AOSP
mistake. Code that thought it was checking the caller's UID may
actually be checking its own (always the same answer, always true).

How to detect: search for `myUid` in security-sensitive code paths.
If found, verify whether the intent was to check the caller.

## Pattern 6: Hidden API behavior changes

`@hide` APIs in `frameworks/base/core/java/android/...` can be used
by app code via reflection (or by privileged apps directly), but they
have weak compatibility guarantees. A method signature that's stable
across 14/15 may change in 16, breaking downstream code that depended
on it.

How to detect: when the user is debugging "this worked in 14 but
doesn't in 16", check if the API touched is `@hide` and whether its
signature or behavior changed.

## How to communicate findings

When you find one of these patterns, structure the finding clearly:

1. **What you found** (the divergent code, the leak, the dropped
   check). Cite file:line:release_tag.
2. **What the consequence is** (less safe under ambiguity, callback
   leak, etc.). Be specific.
3. **Where the related code lives** (the version where it's correct,
   so the user can compare).

Don't bury safety findings in a list of incidental observations. They
deserve their own section in your response.
