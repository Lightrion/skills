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

## Distribution of the 30 eval queries

| Expected skill | Count | Rationale |
|----------------|-------|-----------|
| `aosp-platform-development` | 12 | Framework engineers, AAOS, ROM devs, OEM, SELinux, VHAL, init.rc, etc. |
| `android-app-development`   | 11 | App devs with platform-impacting issues, behavior changes, SDK internals |
| Neither (should not trigger) | 7 | Pure app dev, DevOps, library questions, code review — should not activate either skill |

Includes intentional edge cases:

- **Overlap zone**: "How does Activity.finish() work internally?" — could
  go either way; expected is app-dev because user says "I want to use
  it in my checkout flow".
- **Borderline neither**: Retrofit / OkHttp question (no platform context),
  Hilt DI question (architecture only).
- **Multi-trigger plat**: "CarUxRestrictionsManagerService in AAOS 16, fault
  defaults, AAOS 14 comparison" — hits multiple platform-dev triggers
  intentionally.

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

### v0.1.0 (initial)

- 30 eval queries created
- Both skills' descriptions are ~1100 chars with extensive trigger lists
- Manual eval not yet run
