# Workflow: MCP and local tools, complementary not competing

When working on AOSP, the Lightrion MCP server and your local tools
(Read, Grep, Glob, Bash on an AOSP checkout) serve different purposes.
Combining them properly is what makes the difference between a
shallow guess and a verified answer.

## The four stages

### DISCOVER — semantic retrieval

**Use:** `search_code` from the Lightrion MCP.

**Why:** AOSP is ~100 GB of source. Local grep is impractical and
keyword-matching misses concepts that aren't named exactly. Semantic
search finds the right places by meaning, not by string match.

**How:** rephrase the user's question to emphasize concepts. Use the
`version` parameter matching the AOSP release the user targets. When
the user is asking about a specific minor release ("in r20 of
AOSP 15..."), add the `release_tag` argument to narrow the search.

```
User: "How does WorkManager keep running in Doze?"

search_code(
    query="WorkManager Doze mode JobScheduler constraint exemption",
    version="17",
    limit=8
)
```

You get back 5-10 file paths with line ranges. This is your map of
the territory.

### CONFIRM — read the actual code

**Use:** `get_chunk` for the MCP (returns the full text of a chunk by
ID) or `get_file` if you want the surrounding context. If the user
has an AOSP checkout, `Read` on the local file also works.

**Why:** search snippets are truncated. The real semantics are in the
20 lines around the cited line, not in the 80 characters returned by
search. Don't write conclusions from a snippet alone — open the chunk.

```
search_code returned a chunk in JobSchedulerService.java around
line 14289. Before claiming "JobScheduler exempts work-driven jobs
from Doze", read the actual logic:

get_chunk(chunk_id="<from search result>", version="17")
```

### EXPAND — follow the graph

**Use:** local `Grep` and `Read` if the user has an AOSP checkout, or
the cs.android.com link surfaced by Lightrion otherwise.

**Why:** answering "how does X work" usually means walking the call
graph — who calls this, what does it call, where's the corresponding
test, what's the related XML/manifest. Lightrion handed you the entry
point; now you need to navigate from there.

```
# Local
Grep "mBackgroundJobsDelay" frameworks/base/services/core/...

# Or on cs.android.com, click through hyperlinks from the file
# Lightrion already linked you to.
```

**Across a Binder boundary, use `binder_edges`.** Grep and
cs.android.com stop at the interface: the dispatch from `IFoo` to its
implementation goes through generated code (`BnFoo`, `IFoo.Stub`) that
is not in the source tree. `binder_edges(anchor="IFoo", method="bar")`
returns the classes that actually implement `bar`, with file:line, and
`direction="upstream"` returns the Java call sites that bind `IFoo`.

### SYNTHESIZE — answer with citations

**Use:** your own words, with file:line:release_tag citations.

**Why:** the user needs to verify your answer or follow up. Vague
explanations like "AOSP uses a delay mechanism in Doze" are useless.
Concrete citations make the answer auditable.

Good:
> "In android-17.0.0_r1, the staggered exit-Doze delay lives in
> `JobSchedulerService.java:14289-14315`. The constant
> `mBackgroundJobsDelay` (line 14302) is checked against the job's
> rush_eligible flag; non-rush-eligible jobs are deferred by the
> configured delay to spread the wake spike across cores."

Bad:
> "There's a delay mechanism in JobSchedulerService that handles Doze
> wake-up fairness."

## Worked example: end-to-end

**User question:** "How does the cluster app subscribe to vehicle speed
in AAOS 17?"

### DISCOVER

```
search_code(
    query="cluster app vehicle speed subscription VehiclePropertyManager CarPropertyManager",
    version="17",
    limit=8
)
```

Returns top hits including:
- `packages/services/Car/car-lib/src/.../CarPropertyManager.java:425` (registerCallback)
- `packages/services/Car/cluster/.../ClusterCarPropertyClient.kt:88`
- `hardware/interfaces/automotive/vehicle/.../VehiclePropertyIds.java:42` (PERF_VEHICLE_SPEED_DISPLAY)

### CONFIRM

```
get_chunk(chunk_id="<ClusterCarPropertyClient one>", version="17")
```

You see the cluster registers a callback for property
`PERF_VEHICLE_SPEED_DISPLAY` at `1.0f` Hz, with handler on the
cluster's main thread.

### EXPAND

If the user is in an AOSP checkout, run:

```
Grep "registerCallback.*PERF_VEHICLE_SPEED" packages/services/Car/
```

to find all callers across AAOS. You learn the cluster, the speedometer
widget, and the HVAC display all subscribe — useful context for the
user's actual concern.

### SYNTHESIZE

> "In android-17.0.0_r1, the cluster app subscribes to vehicle speed via
> `CarPropertyManager.registerCallback()` at
> `packages/services/Car/car-lib/src/android/car/hardware/property/CarPropertyManager.java:425`.
> The cluster-specific binding is in
> `ClusterCarPropertyClient.kt:88` — it registers for property
> `PERF_VEHICLE_SPEED_DISPLAY` (defined at `VehiclePropertyIds.java:42`)
> at 1 Hz on the cluster's main thread.
>
> Note: starting in AOSP 16, `subscribePropertyEvents()` was introduced
> as a richer alternative to the older `registerCallback()`. If you're
> writing new code, prefer the new API — it supports variable update
> rates and per-area subscriptions."

The user has a verifiable answer, knows what's new in 16, and can
follow up by clicking the cs.android.com link surfaced in the
search results.

## When to skip Lightrion

Lightrion shines at **conceptual search**. It's overkill for:

- "What's the file path of X?" if the user already knows roughly where
  to look. Just use Read directly.
- "Show me the imports of file Y" — use Read.
- "List all files in directory Z" — use Glob.
- "What changed between two commits of file W?" — use git, not
  Lightrion. Lightrion indexes release tags (and per-minor-release
  coverage for archaeology), not arbitrary commit diffs.

Use Lightrion when the user's question requires understanding the code,
not just inspecting it.
