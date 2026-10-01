# AOSP version conventions

AOSP behavior changes substantially between Android releases. Every
citation in an answer should anchor to a specific release tag, not
just a major version number.

## Release tags currently indexed by Lightrion

| Major version | Latest release tag | Notes                                          |
|---------------|--------------------|------------------------------------------------|
| 13            | android-13.0.0_r84 | Long-lived OEM/automotive baseline; many ROMs |
| 14            | android-14.0.0_r75 | Predictive back gesture, broadcast batching   |
| 15            | android-15.0.0_r36 | 16 KB pages, partial screen sharing, AAOS UX  |
| 16            | android-16.0.0_r4  | New VHAL APIs, refined Car APIs               |
| 17 (default)  | android-17.0.0_r1  | Latest. Most LLMs predate it — Lightrion is   |
|               |                    | one of the few sources that knows it exists   |
| lineage-23    | lineage-23.0       | LineageOS 23, a downstream fork of AOSP 16    |

When in doubt about which version to query, use 17. If the user is
working on an OEM platform or forked baseline at an older release,
ask them which one.

## Non-AOSP versions

Not every indexed version is an AOSP major. `lineage-23` is a
downstream fork — LineageOS 23, derived from AOSP 16. Three things
follow, and getting them wrong produces confidently wrong answers:

**Version ids are opaque strings, not numbers.** Do not parse,
increment or compare them arithmetically. `list_versions` is the only
authority on what exists.

**cs.android.com does not host fork code.** Paths that exist only in
the fork — `lineage-sdk/`, `hardware/lineage/`, `vendor/lineage/`,
`packages/apps/Aperture` — have no cs.android.com equivalent. Linking
there produces a dead link. Only surface a cs.android.com link when the
result comes from an AOSP version, or when the file is one the fork
inherits unmodified from AOSP (and say so if you cannot tell). For
LineageOS specifically, the upstream is github.com/LineageOS.

**A fork's release tag is not an AOSP tag.** `lineage-23.0` is a
branch name, not an `android-XX.0.0_rN` tag, and the fork's projects
do not all sit on the same AOSP base. Do not state which AOSP release
a given fork file derives from unless you have verified it.

### What a fork is actually useful for

The question a fork maintainer asks is rarely "what did they change" —
it is "what can I leave alone at the next rebase". `diff_versions`
answers that directly:

```
diff_versions(query="charging control", version_a="16",
              version_b="lineage-23", limit=10)
```

Measured on LineageOS 23 against AOSP 16: of 1,133 manifest projects,
870 are unmodified AOSP; across the 103 forked projects, 96.99% of
code files are untouched. Expect the interesting delta to be narrow
and concentrated — and read `only_in_a` with the caveat below in
mind.

## Per-minor-release coverage (release_tag argument)

Lightrion indexes **every minor release** within each major (e.g.,
AOSP 13 covers r1 through r84). Each chunk carries a `release_tags`
array listing every minor release where that code appears. This
enables precise archaeology queries like:

- "In which minor release of AOSP 15 did `PropertyHalService.getProperty()`
  change?" — search with version=15, look at the `release_tags` on
  the returned chunks. Pre-r20: one form. r20 and later: another.
- "Is this method present from r1 of AOSP 14, or was it added later?"
- "Which lines of `zygote.te` are stable across the entire release
  history?"

To target a specific minor release, pass `release_tag`:

```
search_code(query="...", version="15", release_tag="android-15.0.0_r20")
```

To query across the full minor-release history of a major (useful for
"when did this change?" archaeology):

```
search_code(query="...", version="15", release_tag="*")
```

When `release_tag` is omitted, the server uses the latest tag for that
major (the table above).

Note: per-minor-release granularity is exposed via MCP only. The web
UI at search.lightrion.com sticks to the latest tag per major.

## Why version-anchoring matters

Examples where the wrong version answer is actively misleading:

### VHAL subscription API

- **AOSP 13/14:** `CarPropertyManager.registerCallback()` only.
- **AOSP 15:** adds `subscribePropertyEvents()` (early form, limited).
- **AOSP 16:** adds `registerSupportedValueChangeCallback()` and
  full `subscribePropertyEvents()` with per-area rates.
- **AOSP 17:** APIs from 16 carry forward; verify in `list_versions`
  for any 17-specific refinements.

An answer that says "VHAL subscription is done via `registerCallback`"
is correct for 13/14, misleading for 16+ (where it's the legacy API).

### Driving state restriction model

- **AAOS 13/14:** bit-flag based, multiple individual restrictions
  (UX_RESTRICTIONS_NO_VIDEO, UX_RESTRICTIONS_NO_KEYBOARD, etc.).
- **AAOS 15/16/17:** similar model but with refined per-display logic
  and passenger zone awareness in 16+.

A claim about how distraction-optimization works is wrong if the
release isn't specified.

### init.rc service ordering

- **AOSP 13:** specific ordering for `boot.car_service_created=1`
  sysprop.
- **AOSP 14/15/16/17:** ordering reorganized; the sysprop still exists
  but is set at a different point in the boot sequence.

If the user is debugging a boot timing issue, the release tag determines
which approach actually works.

### Method-level evolution within a major

For one concrete example surfaced by recent testing:
`CarPropertyService.getProperty()` was substantially refactored
between AOSP 13 and 14 (synchronized lock → async lambda + latency
histogram tracing), and then received only a one-character refinement
between 14 and 15 (static → instance field, for testability). Knowing
this lets you tell the user "your fork at AAOS 13 r84 is built on a
materially different API surface than AAOS 14 r1+".

## Comparing across versions

When the user wants to know "what changed between X and Y", two
options:

**Major-to-major** — use `diff_versions`, not two manual searches:

```
diff_versions(query="...", version_a="15", version_b="16", limit=10)
```

It runs both searches and classifies the results into five buckets:
`unchanged`, `modified`, `moved`, `only_in_a`, `only_in_b`.
`unchanged` is an intersection of content-addressable chunk ids —
a hash guarantee, not a similarity score — so "this file is identical
between the two releases" is something you can state as fact.

**One caveat that matters.** `diff_versions` compares the top-N
semantic results of each version, not whole trees. So:

- `only_in_b` means "in B's top-N, absent from A's top-N". A good
  detector of additions, not an exhaustive list.
- `only_in_a` does **not** mean "removed in B". The entry may well
  exist in B and simply have ranked lower. Never report `only_in_a`
  as a deletion without confirming with a targeted `search_code` on
  version B.

Only the `unchanged` bucket carries a hard guarantee. Say so when you
rely on it, and hedge on the rest.

The `verify` parameter is a no-op kept for API stability — the chunk
id intersection already gives byte-equality. Passing it costs nothing
and changes nothing.

**Minor-to-minor** within a major — use the `release_tags` array on
each chunk:

```
search_code(query="...", version="15", release_tag="*")
```

The returned `release_tags` arrays tell you which minor releases
each chunk appears in. A gap or boundary in the arrays = a refactor
or rewrite at that release.

Or point the user at the Compare tool at
https://search.lightrion.com/compare which does this visually with a
side-by-side view (across majors, latest tag of each).

## Updating this map

The release tags above reflect the snapshot indexed by Lightrion at
the time this skill was published. The MCP's `list_versions` tool
returns the current release tags, which may have advanced (e.g., r4
becomes r5 when a quarterly maintenance release lands). When in
doubt, call `list_versions` to get the current ground truth.
