# Lightrion Skills

Claude Agent Skills for Android development, powered by the
[Lightrion AOSP MCP server](https://aosp-rag.lightrion.com).

These skills teach Claude how to work effectively with Android source
code — both AOSP platform work and Android app development. They turn
Claude from a general-purpose assistant into a domain expert that
knows when to query Lightrion, how to read the results, what to flag,
and how to cite findings precisely.

## Available skills

| Skill | Audience | What it does |
|-------|----------|--------------|
| [`aosp-platform-development`](aosp-platform-development/) | Framework engineers, OEM platform teams, AAOS automotive devs, ROM developers | Teaches Claude the MCP-and-local workflow for AOSP work, version-anchored citations, safety-pattern detection, fork-vs-AOSP comparison, and cs.android.com integration |
| [`android-app-development`](android-app-development/) | Android app developers using Kotlin/Java + Jetpack | Teaches Claude when to look at AOSP source for app-dev questions (Doze, broadcasts, services, permissions, behavior changes), how to translate platform code back to app-developer language |

Both skills use the same Lightrion MCP server. They differ in
**audience**, **depth**, and **trigger conditions** — Claude
auto-selects the right one based on the context of your work.

## Installation in Claude Code

### Option A: Clone into your global skills directory

```bash
cd ~/.claude/skills/
git clone https://github.com/Lightrion/skills.git lightrion-skills

# Verify
ls ~/.claude/skills/lightrion-skills/
# Should show: aosp-platform-development/  android-app-development/
```

The skills are picked up automatically by Claude Code on the next
session. No manual loading needed — Claude reads the frontmatter at
startup and activates each skill when its description matches your
current context.

### Option B: Project-local

If you want the skills active only in a specific project (e.g., your
AOSP checkout):

```bash
cd /path/to/your/project
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/Lightrion/skills.git lightrion-skills
```

## Setting up the Lightrion MCP server

Both skills depend on having the Lightrion AOSP MCP server connected
in Claude Code. Free, rate-limited, no signup beyond the token.

1. Get a token at https://aosp-rag.lightrion.com
2. Add to your Claude Code MCP config (`~/.claude.json` or via the
   Claude Code UI):

```json
{
  "mcpServers": {
    "lightrion-aosp-public": {
      "url": "https://aosp-rag.lightrion.com/mcp/",
      "headers": {
        "Authorization": "Bearer <your-token>"
      }
    }
  }
}
```

3. Restart Claude Code. The MCP tools `search_code`, `get_chunk`,
   `get_file`, `list_versions` and `diff_versions` should now be
   available.

## How the skills work together

Claude loads both skills' descriptions into its system prompt at the
start of each session (a few hundred tokens total — negligible). When
you give Claude a task, it reads the descriptions and activates the
one matching your context:

- Working in an AOSP checkout, asking about `CarService` internals?
  → `aosp-platform-development` activates.
- Working in your app's `MainActivity.kt`, asking why a broadcast
  isn't delivered?
  → `android-app-development` activates.

Only the activated skill's full content is loaded, keeping context
usage low.

## What the skills don't do

These skills do **not** include the Lightrion MCP server itself —
they're just instructions for Claude on how to use it well. You need
to install both: the MCP server (one config line) and the skills
(one clone).

These skills also don't replace cs.android.com for **symbol
navigation** (jump-to-definition, call hierarchy, xref). Lightrion is
for semantic retrieval. Both skills explicitly suggest linking to
cs.android.com when navigation is what the user wants — except for
non-AOSP versions such as `lineage-23`, whose fork-only code is not
hosted there.

## Versioning

These skills are versioned together with the indexed AOSP releases
they expect on the Lightrion server:

| Version | Indexed releases                                                     |
|---------|----------------------------------------------------------------------|
| 0.3.0   | 13.0.0_r84, 14.0.0_r75, 15.0.0_r36, 16.0.0_r4, 17.0.0_r1, lineage-23.0 |
| 0.2.0   | 13.0.0_r84, 14.0.0_r75, 15.0.0_r36, 16.0.0_r4, 17.0.0_r1             |
| 0.1.0   | 13.0.0_r84, 14.0.0_r75, 15.0.0_r36, 16.0.0_r4                        |

Lightrion also indexes **per-minor-release coverage** within each
major (r1 through the latest), so queries can target a specific minor
release via the `release_tag` argument — useful for "in which release
did this method change?" questions. This granularity is exposed via
the MCP API; the web UI at search.lightrion.com exposes the latest
tag per major.

When the Lightrion server bumps to newer release tags (e.g., r5 for
quarterly maintenance), the skills' `references/version-conventions.md`
will be updated.

## Contributing

Open an issue or PR at https://github.com/Lightrion/skills if:

- You hit a question where the skill should have helped but didn't —
  share the prompt so we can tighten the description.
- You have a workflow pattern that would benefit other Android
  developers using Claude.
- You spot an error in the safety patterns or version map.

## License

Apache 2.0. You're free to fork, adapt, redistribute. Attribution
appreciated.

## See also

- [Lightrion homepage](https://lightrion.com)
- [search.lightrion.com](https://search.lightrion.com) — web UI to
  try the MCP search interactively (free, rate-limited)
- [Anthropic's Agent Skills docs](https://code.claude.com/docs/en/skills)
- [Agent Skills open standard](https://agentskills.io)
