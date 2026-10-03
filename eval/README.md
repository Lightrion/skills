# Skill trigger evaluation

This directory contains evaluation queries for the two Lightrion
skills. They exist to verify that Claude activates the right skill
(or no skill) when given realistic prompts.

## Why this matters

Anthropic's own guidance (skill-creator):

> Claude has a tendency to "undertrigger" skills — to not use them
> when they'd be useful. To combat this, please make the skill
> descriptions a little bit "pushy".

Conversely, an over-pushy description causes a skill to fire on
irrelevant prompts (over-triggering), wasting context and biasing
Claude's reasoning. The eval set catches both failure modes.

## Distribution of the 46 eval queries

| Expected skill | Count | Rationale |
|----------------|-------|-----------|
| `aosp-platform-development` | 23 | Framework engineers, AAOS, ROM devs, OEM, SELinux, VHAL, init.rc, cross-major (incl. AOSP 17), fork-vs-AOSP comparison, CVE / security bulletin work on a fork, Binder/HAL implementation lookup |
| `android-app-development`   | 12 | App devs with platform-impacting issues, behavior changes, SDK internals, minor-release deltas, which system service handles an SDK call |
| Neither (should not trigger) | 11 | Pure app dev, DevOps, library questions, code review, end-user ROM support, end-user patch level, non-Android CVE — should not activate either skill |

Includes intentional edge cases:

- **Overlap zone**: "How does Activity.finish() work internally?" — could
  go either way; expected is app-dev because user says "I want to use
  it in my checkout flow".
- **Borderline neither**: Retrofit / OkHttp question (no platform context),
  Hilt DI question (architecture only).
- **Multi-trigger plat**: "CarUxRestrictionsManagerService in AAOS 16, fault
  defaults, AAOS 14 comparison" — hits multiple platform-dev triggers
  intentionally.
- **Fork negative controls**: two queries mention LineageOS and expect
  *neither* skill — a TWRP flashing problem and a privacy opinion. The
  platform skill lists "LineageOS" and "custom ROM" among its triggers,
  so the brand name on its own must not activate it. These two are the
  ones to watch when tuning that description: they are the cheapest way
  to notice it has become too pushy.

- **Security negative controls**: an end-user "security patch level" question
  and a log4j CVE. The platform skill now triggers on CVE ids and security
  patch levels; these two check that the words alone, without Android
  platform or fork code, do not activate it.

## How to run the evals manually

Anthropic's skill-creator suggests a manual approach when an automated
evaluator isn't available:

1. Open a fresh Claude Code session with **both** skills installed.
2. Paste a query from `eval-queries.json`.
3. Watch which skill (if any) Claude activates. Claude Code shows
   loaded skills in the session header / context indicator.
4. Compare to the `expected` field. Record pass/fail.
5. Aim for >= 90% match on should-trigger queries and >= 90% on
   should-not-trigger.

For a 30-query set, this is ~30 minutes of human-in-the-loop work
per iteration of the skill descriptions.

## How to run the evals automatically

If you have the `skill-creator` plugin installed (Anthropic-shipped):

```bash
/plugin install skill-creator@anthropic-agent-skills
```

It can run an eval-driven optimization loop:

1. Splits queries 60/40 into train/test.
2. Runs trigger probes against the current SKILL.md descriptions.
3. Generates improved descriptions.
4. Re-evaluates on the test set.
5. Returns the description with the highest test score.

Point it at this `eval-queries.json` and the two `SKILL.md` files.

## What to do if a query fails

### A "should-trigger" query didn't trigger the skill

**Cause**: the description doesn't match the query's vocabulary.

**Fix**: identify the keyword/phrase the query uses that the
description doesn't cover, and add it to the description's trigger
list. Be specific: "WorkManager" is more triggering than "background
work".

Example: if "I'm forking AOSP 15 to ship an OEM build" doesn't
trigger `aosp-platform-development`, add "forking AOSP", "OEM build",
"shipping a custom Android" to the trigger phrases in the description.

### A "should-not-trigger" query triggered a skill

**Cause**: the description is too broad — captures app dev questions
that don't need AOSP context.

**Fix**: narrow the description. Add explicit anti-triggers
("Do NOT use this skill for: pure Compose layout questions, library-only
questions about Retrofit/OkHttp/Hilt, ..."). The description can
include negatives.

### A query that should trigger one skill triggered the other

This is the most subtle failure mode. Usually means the trigger
boundaries between the two skills aren't sharp enough.

**Fix**: clarify what distinguishes the two audiences in **both**
descriptions. The `aosp-platform-development` description should say
"NOT for app developers using the SDK". The `android-app-development`
should say "NOT for platform engineers modifying AOSP source".

## Iteration log

Track changes here as you iterate.

### v0.2.0

- Added AOSP 17 (`android-17.0.0_r1`) to indexed releases. Default
  version bumped from 16 to 17.
- Added per-minor-release coverage feature: each indexed chunk carries
  a `release_tags` array; `release_tag` can be passed to `search_code`
  to target a specific minor release or set `release_tag="*"` for
  archaeology.
- Trigger descriptions in both SKILL.md files extended to mention
  AOSP 17 explicitly, minor-release queries, and API 37 mapping.
- 3 new eval queries added covering AOSP 17 (platform fork, AAOS API
  evolution) and minor-release deltas (app-dev BroadcastQueue
  scenario). Total now 33.

### v0.1.0 (initial)

- 30 eval queries created
- Both skills' descriptions are ~1100 chars with extensive trigger lists
- Manual eval not yet run
