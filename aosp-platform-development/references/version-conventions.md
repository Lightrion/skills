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

When in doubt about which version to query, use 17. If the user is
working on an OEM platform or forked baseline at an older release,
ask them which one.

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

## How chunks carry version information

Every chunk returned by Lightrion (from `search_code`, `get_chunk`, or
`diff_versions`) carries metadata that makes cross-version archaeology
cheap:

### Compact release_tags

`release_tags` is returned in a **range form**, e.g.
`["r1-r24", "r27-r84"]` instead of the full
`["android-13.0.0_r1", "android-13.0.0_r2", …]` array. About 85% fewer
tokens per result, and gaps in coverage (above, r25/r26 — Google didn't
publish those minor releases) become visible at a glance. The major
version is implicit in the chunk's `version` field, which is why the
ranges drop the `android-XX.0.0_` prefix.

### first_seen_release / last_seen_release

Each chunk also carries `first_seen_release` and `last_seen_release` as
full release strings (e.g. `"android-14.0.0_r29"`). These are pre-computed
boundaries that answer "when did this code first appear?" and "is it
still present in the latest indexed release?" without parsing the
`release_tags` array. They're also valid values you can pass directly
back to `search_code` or `get_file` via the `release_tag` argument.

### Content-addressable chunk_ids

`chunk_id` is a content hash of the chunk body. Two practical
consequences:

1. **Identical code across versions → identical `chunk_id`.** If a
   method exists unchanged in AOSP 14 and AOSP 17, both versions'
   chunks share the same `chunk_id`. The `start_line` / `end_line` may
   differ (the code may have shifted within the file), but the chunk
   itself is byte-identical.

2. **`diff_versions`' `unchanged` bucket is byte-perfect by
   construction.** It's computed as a set intersection on chunk_ids,
   not as a heuristic. When `diff_versions` says a method is unchanged
   between two versions, that's guaranteed equality, not a similarity
   score. The same is true in reverse: the `modified` bucket contains
   pairs with the same `(file_path, symbol_name)` but different
   `chunk_id` — content is guaranteed to differ.

This is why the `verify=true` argument on `diff_versions` is largely
informational rather than corrective: the structural classification is
already deterministic.

## Comparing across versions

When the user wants to know "what changed between X and Y", the
preferred tool is **`diff_versions`** — it runs both searches in
parallel and classifies chunks into 5 buckets:

```
diff_versions(
    query="...",
    version_a="15",
    version_b="16",
    limit=10
)
```

Returns `unchanged`, `modified`, `moved`, `only_in_a`, `only_in_b`
arrays plus a `summary` with counts. Reading the response is much more
useful than reading two raw `search_code` results — the structural
matching is done for you, and the `unchanged` and `modified` buckets
are byte-perfect by construction (see "Content-addressable chunk_ids"
above).

Use the `moved` bucket to spot directory renames, method moves, and
extracted-to-new-file refactors. Use `only_in_b` to see what was added
in the newer version, `only_in_a` to see what was removed.

### Minor-to-minor within a major

`diff_versions` also supports comparing two minor releases of the same
major — useful for questions like "what shipped in the September 2024
quarterly maintenance release of Android 14?" or "what's different
between r45 and r60 of Android 14?". Pass full release tags on **both**
sides:

```
diff_versions(
    query="BroadcastQueue deliverToReceiverLocked",
    version_a="android-14.0.0_r29",
    version_b="android-14.0.0_r75",
    limit=10
)
```

The response includes a `diff_mode` field
(`"major_to_major"` or `"minor_to_minor"`) and echoes
`release_tag_a` / `release_tag_b` so the caller can confirm which
axes were actually diffed. Same classification semantics (chunk_id
content hashes), same byte-perfect guarantees on `unchanged` and
`modified`.

If you only have one release tag (e.g. comparing r29 with the latest
indexed of a major), pass it explicitly on one side and the bare major
on the other — but note that this is a major-to-major diff, since both
sides resolve to the same major and the unspecified side is ambiguous.
For unambiguous results, always pass tags on both sides for
minor-to-minor.

### Broad archaeology across all minor releases

For "in which release_tag did this method first appear / get changed?"
questions where you don't know which two tags to compare a priori,
use `search_code` with `release_tag="*"` and inspect the
`release_tags` array (and the `first_seen_release` /
`last_seen_release` fields) on each chunk:

```
search_code(query="...", version="15", release_tag="*")
```

A gap or boundary in the `release_tags` arrays signals a refactor or
rewrite at that release. The `first_seen_release` is often the
fastest answer when the user's question is "when was this added?".

Or point the user at the Compare tool at
https://search.lightrion.com/compare which does this visually with a
side-by-side view (across majors, latest tag of each).

## Updating this map

The release tags above reflect the snapshot indexed by Lightrion at
the time this skill was published. The MCP's `list_versions` tool
returns the current release tags, which may have advanced (e.g., r4
becomes r5 when a quarterly maintenance release lands). When in
doubt, call `list_versions` to get the current ground truth.
