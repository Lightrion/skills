# Security bulletins: from a CVE to the public commit

The Android Security Bulletin lists CVEs and the Android versions they
affect. It does not tell a fork maintainer **where the fix is in the
public code of their version**. Lightrion joins every bulletin (and OSV)
with the public `android-security-*` tags of Android 14 to 17, commit by
commit, and exposes the result through three MCP tools. This file is
how to use them and how to read what they return.

`list_versions` tells you which versions have this data: each entry
carries `security_data: true|false`. Today: 14, 15, 16, 17.

## Contents

1. The three tools
2. Reading a resolution: status, method, confidence, provenance
3. Reading a tag: announced, unannounced, follow-ups, platform updates
4. Workflows
5. How to answer
6. Limits

## 1. The three tools

| Tool | Use it for | Key arguments |
|------|-----------|---------------|
| `security_bulletin` | "What did the September bulletin fix on 16?" One month, one major: every CVE that lists the major, with its status and public commits | `version`, `month` (YYYY-MM), filters `severity`, `status`, `projects`, `exclude_projects`, `limit` |
| `security_lookup` | "Is CVE-2026-28609 fixed on 14?" One CVE or bug id, across all covered majors at once | `query` (CVE id, bug id, or OSV id) |
| `security_tag_changes` | "What does android-security-16.0.0_r8 contain?" / "What do I pick between r6 and r8?" Everything a tag (or a range of tags) fixes | `version`, `tag`, `since_tag` (exclusive lower bound), `only_unannounced`, `projects`, `exclude_projects`, `limit` |

Prefer filters over unfiltered calls: a quarterly bulletin has about 80
CVEs per major, and a range of tags several hundred commits.

Every response carries `notes` (definitions) and a `summary_scope`
sentence saying what its counters count. Read them when a number looks
surprising: `security_bulletin` counts the CVEs **published that month**
that list the major; `security_tag_changes` counts what the **tags in
the range** fix, whatever the bulletin month.

## 2. Reading a resolution

Each advisory entry has a `status`:

| Status | Meaning | What to tell the user |
|--------|---------|-----------------------|
| `resolved` | A fixing commit is in the public security tags (or inherited from the release the security line starts from) | Give the commit, the project and the tag; see `confidence` |
| `awaiting-tag` | Announced, no fix found yet, but no security tag of this major was published since the announcement (`awaiting` gives the dates) | Nothing can be concluded yet. The fix may come with the next tag. Do not say "unpatched" |
| `announced-not-published` | Tags exist after the announcement and none contains a fix | The fix is not in the public code of this major. Partners may have it; the public tree does not |
| `code-absent` (probable) | No fix, and none of the code the bulletin's commit patches exists in this major | The vulnerable code is probably not there (removed or reworked before the release). Quote `detail`; say "probably not affected", not "safe" |
| `platform-only` | The fix is in the platform release the security line is compared against, with no equivalent in the security tags | A fork moving from that release to the security tag loses it. Give the commit from `platform_only_commits` |
| `not-covered` | The affected project is not in the manifest, or out of scope | Lightrion cannot check it |
| `before-window` | Published before the first ingested tag of this major | Lightrion cannot check it |
| `not-affected` (lookup only) | The advisory does not list this major | Say so, and mention the majors it does list |

For `resolved`, three more fields matter:

- **`method`**: how the commit was linked to the advisory.
  Certain: `bug_id` (Bug: line), `cve_ref` (the commit message quotes
  the CVE id), `change_id` / `patch_id` (same change as the bulletin's
  public commit), `base` (the advisory's own bug id in the history of
  the base release). Probable: `related_bug` (a bug cited by the
  bulletin's commit), `subject` (same normalized subject, a common
  file), `content` (the bulletin commit's changed lines are present in
  the latest tag's code).
- **`confidence`**: `certain` or `probable`, derived from the method.
- **`public_commits[]`**: each with `method`, `confidence`,
  `evidence` (the proof: reference commit, clue, score) and
  `similarity` (0 to 1, for probable matches).

`provenance` (on `first_fix`, `last_fix` and each commit) is a separate
axis: `security_tag` (an `android-security-*` tag) or `base_release`
(the platform release the security line starts from). Method says how
the link was made; provenance says where the fix lives.

`public_before_announcement_days`: the fix was complete in a public tag
that many days before the first bulletin listing the CVE (measured from
the last fixing commit, a conservative figure).

## 3. Reading a tag (`security_tag_changes`)

| Block | Content |
|-------|---------|
| `announced` | Commits that fix an advisory listing this major, grouped by advisory, with `methods` and `confidence` |
| `unannounced` | Security fixes whose bug ids are in no bulletin and no OSV advisory, grouped by bug id; markers `cve_fix`, `sts`, `restrict_automerge` hint at their nature. Some become CVEs months later |
| `related_to_advisory` | Follow-ups of an announced CVE (their bug id is cited by its public commit) |
| `advisory_not_listing_this_major` | Commits linked to an advisory of another major (often a backport the bulletin does not list for this one) |
| `dropped_from_base` | Commits of the platform release the security line is compared against, with no equivalent in the tag; `dropped_from_base_out_of_scope` holds device config and test suites |
| `summary.platform_update_tags` | Tags that carry a platform update (hundreds of commits): most of their commits are not security fixes |

## 4. Workflows

**"Is CVE-X fixed in my fork?"**
1. `security_lookup(query="CVE-X")`: status per major.
2. For the user's major, take `public_commits[]` (project, sha, tag).
3. If the user has the fork locally: check whether that commit, or an
   equivalent, is in their branch (`git log --grep="Bug: <id>"`,
   `git cherry`, or compare the patched lines). If not local, use
   `get_file` on the fork version (e.g. `lineage-23`) to read the
   patched code.
4. Answer with the evidence, and say whether the link is certain or
   probable.

**"What should we pick for this month's security update?"**
1. `security_tag_changes(version, tag=<new tag>, since_tag=<tag the fork is on>)`.
2. Walk `announced` (CVEs), then `unannounced` (fixes no bulletin
   names, still worth taking), then `related_to_advisory`.
3. Use `exclude_projects` for parts the fork does not ship (TV, car,
   watch apps...).
4. Check `summary.platform_update_tags`: if the range includes one, its
   commits are a platform update, not a patch set.

**"What did this month's bulletin change for us?"**
`security_bulletin(version, month)`, then filter on
`status=["announced-not-published", "awaiting-tag", "platform-only"]` to
see what needs attention.

## 5. How to answer

- **Cite the commit and the tag**: `frameworks/av 8cf7d2e9937b
  (android-security-14.0.0_r29)`, with the googlesource link
  `https://android.googlesource.com/<project>/+/<sha>`.
- **Say how sure you are.** For a probable match, quote `evidence` and
  `similarity`; the user decides whether to verify the code.
- **Absence is not safety.** `announced-not-published` means no public
  fix, not "not vulnerable". `code-absent` means "probably not
  affected", with its evidence. `awaiting-tag` means "too early to tell".
- **Never infer a fix from the security patch level string alone**;
  check the commit.
- **Out of scope stays out**: vendor closed-source components and the
  kernel have no AOSP code to check. Say so instead of guessing.

## 6. Limits

- Coverage: Android 14 to 17, public `android-security-*` tags only.
  Releases and QPR lines outside that lineage are not covered.
- Bulletin revisions are tracked from 2026-09-30 on; earlier removals
  of CVEs from a bulletin are not visible.
- The data refreshes hourly. A bulletin and its tags can be published
  hours or days apart (see `awaiting-tag`).
- A human-readable view of the same data, one page per bulletin:
  https://security.lightrion.com/
