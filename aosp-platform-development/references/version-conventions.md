# AOSP version conventions

AOSP behavior changes substantially between Android releases. Every
citation in an answer should anchor to a specific release tag, not
just a major version number.

## Release tags currently indexed by Lightrion

| Major version | Release tag        | Notes                                          |
|---------------|--------------------|------------------------------------------------|
| 13            | android-13.0.0_r84 | Last common base for older OEMs; many ROMs    |
| 14            | android-14.0.0_r75 | Predictive back gesture, broadcast batching   |
| 15            | android-15.0.0_r36 | 16 KB pages, partial screen sharing, AAOS UX  |
| 16 (default)  | android-16.0.0_r4  | Latest. New VHAL APIs, refined Car APIs       |

When in doubt about which version to query, use 16. If the user is
working on an OEM platform that has frozen at an older release, ask
them which one.

## Why version-anchoring matters

Examples where the wrong version answer is actively misleading:

### VHAL subscription API

- **AOSP 13/14:** `CarPropertyManager.registerCallback()` only.
- **AOSP 15:** adds `subscribePropertyEvents()` (early form, limited).
- **AOSP 16:** adds `registerSupportedValueChangeCallback()` and
  full `subscribePropertyEvents()` with per-area rates.

An answer that says "VHAL subscription is done via `registerCallback`"
is correct for 13/14, misleading for 16 (where it's the legacy API).

### Driving state restriction model

- **AAOS 13/14:** bit-flag based, multiple individual restrictions
  (UX_RESTRICTIONS_NO_VIDEO, UX_RESTRICTIONS_NO_KEYBOARD, etc.).
- **AAOS 15/16:** similar model but with refined per-display logic
  and passenger zone awareness in 16.

A claim about how distraction-optimization works is wrong if the
release isn't specified.

### init.rc service ordering

- **AOSP 13:** specific ordering for `boot.car_service_created=1`
  sysprop.
- **AOSP 14/15/16:** ordering reorganized; the sysprop still exists
  but is set at a different point in the boot sequence.

If the user is debugging a boot timing issue, the release tag determines
which approach actually works.

## Comparing across versions

When the user wants to know "what changed between X and Y", do two
searches:

```
search_code(query="...", version="15", limit=5)
search_code(query="...", version="16", limit=5)
```

Compare the results manually. If they differ in meaningful ways (file
moved, signature changed, behavior reversed), surface that as the
finding.

Or point the user at the Compare tool at
https://search.lightrion.com/compare which does this visually with a
side-by-side view.

## Updating this map

The release tags above reflect the snapshot indexed by Lightrion at
the time this skill was published. The MCP's `list_versions` tool
returns the current release tags, which may have advanced (e.g., r4
becomes r5 when a quarterly maintenance release lands). When in
doubt, call `list_versions` to get the current ground truth.
