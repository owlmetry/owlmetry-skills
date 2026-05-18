---
name: owlmetry-investigate-issues
description: >-
  Triage open Owlmetry issues for the current project. Detects the project
  from the codebase (or asks), lists open + regressed issues, fans out
  sub-agents in parallel to read each issue's occurrences and breadcrumb
  timeline, posts findings back as comments, then walks the user through
  claim → fix → resolve-with-version. Use when you want to know what's
  broken in production and decide what to fix first.
allowed-tools: Bash, Read, Grep, Glob, Task
---

## What this skill does

Owlmetry clusters error events into **issues** (fingerprinted by source + message; network errors additionally by `METHOD host/path`). This skill turns the "scroll the issues list, click into each, read 30 occurrences, cross-reference fingerprints" loop into a single semi-automated pass:

1. Detect which Owlmetry project the current codebase belongs to (or ask).
2. Pull every open + regressed issue.
3. Fan out a sub-agent per issue. Each sub-agent reads the latest 10 occurrences via `investigate-event` (full breadcrumb + cross-app session timeline), forms a hypothesis, and posts findings as a comment on the issue.
4. Compile a single prioritized breakdown across all issues.
5. Recommend what to tackle first — fix, downgrade, silence, snooze, or merge.
6. Walk through the chosen issue: claim → locate call site → help fix → bump version → resolve.

**Human in the loop.** No status transitions happen without explicit confirmation. The skill never auto-fixes code, never auto-resolves, never auto-bumps the app version.

## Prerequisite

You need **either** the `owlmetry` MCP server connected (preferred — every tool below is available as `mcp__owlmetry__<name>`) **or** the `owlmetry` CLI authenticated. If neither is set up, load the `owlmetry-cli` skill first and run through its Setup section.

> **Prefer MCP tools when available.** CLI fallback is provided for every step in the table at the bottom of this file. Do not mix unless you have to — pick one transport per run.

This skill is **read-mostly**: the only writes it ever performs are `add-issue-comment` (during investigation, always), `claim-issue` / `resolve-issue` / `silence-issue` / `snooze-issue` / `merge-issues` (during action, only after user confirmation).

## Step 1 — Determine the project

Try these heuristics in order. Stop at the first hit.

**a. Swift / iOS consumer codebase:**

```bash
grep -rE 'Owl\.configure' --include='*.swift' . | head -20
grep -rEoh 'owl_client_[a-f0-9]{56}' --include='*.swift' . | head -5
```

If a `owl_client_…` key is found, the API key is the project handle. Call `mcp__owlmetry__whoami` (with the calling agent's existing auth) to identify the team, then `mcp__owlmetry__list-projects` to find which project owns an app whose `client_secret` matches.

**b. Node / TypeScript consumer codebase:**

```bash
grep -rE 'Owl\.configure' --include='*.{js,mjs,ts,cjs}' . | head -20
grep -hE 'OWLMETRY_API_KEY|OWL_API_KEY|owl_client_' .env* 2>/dev/null
```

**c. iOS bundle ID lookup:**

```bash
grep -h PRODUCT_BUNDLE_IDENTIFIER *.xcodeproj/project.pbxproj 2>/dev/null | head -3
```

Cross-reference the bundle ID against `mcp__owlmetry__list-apps` — the matching app's `project_id` is the answer.

**d. Single-project fallback:**

If `mcp__owlmetry__list-projects` returns exactly one project for the user's team, just use it.

**e. Otherwise — ask:**

List projects and prompt the user to pick:

```
Found 4 Owlmetry projects on your team:
  1. Lofi               (slug: lofi)
  2. Signature Creator  (slug: signature-creator)
  3. Demo Project       (slug: demo)
  4. Owlmetry           (slug: owlmetry)
Which project should I investigate?
```

Once a project is selected, also note whether to filter by a specific app (multi-app projects only). Default to "all apps" unless the codebase clearly points at one (e.g. detected bundle ID matches one app).

> **Always print the resolved project name + ID at the top of the run.** This is the user's one chance to spot a wrong-project misfire before any state changes.

Do not cache the project across runs — the user might `cd` between repos.

## Step 2 — Pull the issue inventory

Default filter: `status` ∈ `{new, regressed}`. **Include `in_progress` only if the user asks for it** (those are already being worked on, no point spawning sub-agents on them).

```text
mcp__owlmetry__list-issues  project_id=<id>  status=new      [app_id=<id>] [is_dev=false]
mcp__owlmetry__list-issues  project_id=<id>  status=regressed [app_id=<id>] [is_dev=false]
```

Follow the cursor through all pages — issue counts are bounded by `issue_scan` clustering, not unbounded.

Before fanning out, print a one-line preview:

```
Found 14 issues to investigate in Lofi:
  11 new
   3 regressed ⚠️
   (4 in_progress excluded — pass --include-in-progress to include them)
```

The `⚠️` is meaningful: a regressed issue was thought fixed and came back, so it usually wins the priority race.

If the count is **zero**, return "Nothing to investigate — all clear" and stop. Do not spawn any sub-agents.

## Step 3 — Fan out investigation sub-agents

For each issue in the inventory, spawn an investigation sub-agent via the Task tool. **Cap parallelism at 5.** If there are more than 5 issues, batch them: launch 5 in one message (parallel), wait for completion, launch the next 5, and so on. Do **not** launch a 6th sub-agent before the first batch returns — sub-agents have their own context windows and you don't want to exhaust them.

Use this prompt template verbatim, interpolating `<issue_id>`, `<project_id>`, `<today>`:

> You are investigating Owlmetry issue `<issue_id>` in project `<project_id>`.
>
> 1. Call `mcp__owlmetry__get-issue` with `project_id=<project_id>`, `issue_id=<issue_id>`, `occurrence_limit=10`. Note from the response: `title`, `fingerprints[]`, `occurrence_count`, `affected_user_count`, `first_seen_at`, `last_seen_at`, `first_seen_app_version`, `last_seen_app_version`, `environment` distribution, `status`, `app_name`, `app_id`.
>
> 2. From the returned `occurrences[]` (already newest-first, capped at 10), call `mcp__owlmetry__investigate-event` for each occurrence's `event_id` with the default `window_minutes` and `compact=true`. This returns the full session + cross-app breadcrumb timeline merged and sorted.
>
> 3. Look across the 10 timelines for:
>    - **Shared preceding event** that consistently appears in the seconds before the error (likely root cause).
>    - **Shared `_error_stack`** or **`_error_type`** custom attribute on the error event (call site + exception class).
>    - **Network signature**: `_http_url` + `_http_method` + `_http_status_code` on `sdk:network_request` errors.
>    - **`_unhandled` attribute** present (Node SDK auto-captured an uncaughtException / unhandledRejection — usually higher priority because the consumer didn't expect it).
>    - **Cross-app context**: events from sibling apps in the same project + same session window (Owlmetry's `investigate-event` merges these in by `user_id`).
>    - **Version concentration**: are all 10 occurrences on the same app version, or spread across multiple?
>
> 4. Compare `last_seen_app_version` against the app's `latest_app_version` (read via `mcp__owlmetry__get-app` with `app_id` if not on the issue payload). If all occurrences are on an older version, the issue may already be fixed in the latest release — flag for `resolve-current-version`.
>
> 5. Return a markdown report with these exact fields:
>    - **Title**: issue title
>    - **Severity**: `<occurrence_count> occurrences`, `<affected_user_count> users`, last seen `<relative-time>`
>    - **Versions**: `<first_seen_app_version>` → `<last_seen_app_version>` (app latest: `<latest_app_version>`, on-latest: yes/no)
>    - **Environment**: production / development split
>    - **Pattern**: 1–2 sentence hypothesis of root cause derived from the 10 timelines
>    - **Network**: yes (with `METHOD host/path` summary) / no
>    - **Unhandled**: yes / no
>    - **Suggested action**: one of `fix-now` | `silence` | `snooze` | `resolve-current-version` | `merge-into:<other-issue-id>` | `downgrade-sdk-call`
>    - **Rationale**: one line justifying the suggested action
>
> 6. Finally — and **always** — call `mcp__owlmetry__add-issue-comment` with the markdown report as the body. Prefix the body with `### Investigation (Claude, <today>)` so it's clearly authored and dated. This persists findings on the issue itself for future readers.
>
> **You are read + comment only.** Do not call `claim-issue`, `resolve-issue`, `silence-issue`, `snooze-issue`, or `merge-issues`. Status transitions are the parent agent's responsibility, made after the user confirms.

When all sub-agents have returned, collect their reports into a single in-memory list.

## Step 4 — Compile the breakdown

Print one sorted table to the user. Sort primarily by `affected_user_count desc`, secondarily by `is_regressed desc`. Cap at the top 20 rows; if there are more, list the rest by ID + title underneath as "Also seen".

```
| # | Issue                                                | Users | Occ. | Versions          | Suggested            | Why |
|---|------------------------------------------------------|------:|-----:|-------------------|----------------------|-----|
| 1 | ⚠️ Regressed: Crash in `SignatureRenderer.export`     |    42 |  318 | 1.4.0 (latest)    | fix-now              | Affects newest version; no UX fallback |
| 2 | Network error: POST api.revenuecat.com/v1/receipts   |    28 |  121 | 1.3.5–1.4.0       | downgrade-sdk-call   | SDK already retries; UX shows toast |
| 3 | TypeError: Cannot read 'id' of undefined             |    14 |   44 | 1.4.0 (latest)    | fix-now              | Unhandled, top of session |
| 4 | EAI_AGAIN getaddrinfo api.x.com                      |     6 |   12 | 1.3.0 only        | resolve-current-version | All occurrences on stale version |
```

Include `⚠️` on the row for any regressed issue. Append a one-line legend below the table if not obvious.

## Step 5 — Recommend a starting point

Below the table, output a short prose recommendation block. Aim for 3–6 bullet points:

- **Top fix-now candidate**: the highest-severity issue not marked for silence / downgrade / resolve.
- **Batch downgrade candidates**: every issue tagged `downgrade-sdk-call` — offer to silence-or-resolve them as a single batch (one prompt, multiple actions).
- **Merge candidates**: any pair of issues where sub-agent reports identify the same root cause or overlapping `_error_stack` — offer `merge-issues`.
- **Stale candidates**: issues tagged `resolve-current-version` — offer to resolve them all at the app's current `latest_app_version` in one batch. If they ever come back at the same error level, regression detection catches it.

Then **stop and ask** which to tackle first. Phrase the prompt with the issue number from the table, e.g. "Pick a row (1–14), or type `batch-downgrade` / `batch-resolve-stale` / `merge X→Y`."

> **Do not auto-claim, do not auto-fix.** This is the human-in-the-loop checkpoint.

## Step 6 — Fix flow (per chosen issue)

Once the user picks issue X:

1. **Confirm claim**: ask "Claim issue X (sets status to `in_progress`)? This is visible to the team." On yes, call `mcp__owlmetry__claim-issue` with `project_id=<id>`, `issue_id=<X>`.

2. **Locate the call site**: re-read the sub-agent's report for the `_error_stack` and `_error_type` from the latest occurrence. Then grep the codebase:
   - For Swift: `grep -rnE '<class-or-message-substring>' --include='*.swift' .`
   - For Node: `grep -rnE '<class-or-message-substring>' --include='*.{js,mjs,ts}' .`
   - Cross-reference against the symbol or function name in the stack frame nearest to user code.

3. **Help the user write the fix.** Suggest the code change but **stop after editing** — do not assume tests pass. Let the user run their own verification.

4. **Ask about version bump**: "What version will this fix ship in? I can bump now."
   - For Swift apps, look at `Info.plist` `CFBundleShortVersionString` or `MARKETING_VERSION` in `.pbxproj` / `.xcconfig`.
   - For Node packages, look at `package.json` `version`.
   - Suggest the next semver (patch for bug fix; ask if minor/major).
   - **Wait for explicit user OK before editing version files.** The user may want to batch this fix with other changes in the same release.

5. **Resolve with version**: once the user confirms the fix is committed (or is about to ship) at version `Y`, call `mcp__owlmetry__resolve-issue` with `project_id=<id>`, `issue_id=<X>`, `version=Y`. The `version` field is **mandatory** — Owlmetry returns 400 if missing. If the user can't commit to a version yet, suggest `snooze` or `silence` instead and skip the resolve.

## Step 7 — Downgrade flow

For issues the sub-agent tagged `downgrade-sdk-call` (typically network errors where the SDK already retries and the UX handles failure gracefully), there are two real-world options. Explain both, let the user pick per issue:

**a. Silence in dashboard only** — stop alerts, keep recording occurrences:

```text
mcp__owlmetry__silence-issue  project_id=<id>  issue_id=<X>
```

`silenced` is terminal — Owlmetry will not re-alert even if the error keeps happening. Use when you want forensic data without notification noise.

**b. Downgrade the SDK call + resolve** — fix the noise at the source:

1. Find the `Owl.error(...)` call site (same grep approach as Step 6.2).
2. Change it to `Owl.warn(...)` so future occurrences no longer cluster into issues.
3. Resolve the existing issue at the app's current `latest_app_version`:
   ```text
   mcp__owlmetry__resolve-issue  project_id=<id>  issue_id=<X>  version=<latest_app_version>
   ```

Option (b) is usually the right call for network errors with graceful UX, because the regression detector still catches the case where the same call site ever emits `error` again — you've reduced noise without dropping the safety net.

## Step 8 — Snooze flow (suspected one-offs)

For issues where the sub-agent saw a single occurrence, single user, no consistent breadcrumb pattern — suggest `snooze`:

```text
mcp__owlmetry__snooze-issue  project_id=<id>  issue_id=<X>
```

Owlmetry auto-reverts a snoozed issue to `new` and re-fires the `issue.new` push on the **very next** occurrence. So if the one-off assumption holds, you never hear about it again. If it returns, you find out immediately. No version commitment required, no claim of "fix".

## Step 9 — Merge duplicates

If two issues describe the same underlying bug (the sub-agent reports overlap on `_error_stack` or root cause), pick the **older** issue as the target so its history is preserved:

```text
mcp__owlmetry__merge-issues  project_id=<id>  target_issue_id=<older>  source_issue_id=<newer>
```

This moves every fingerprint, occurrence (deduped), and comment from source into target, then deletes source. Confirm with the user first — merges are not reversible through the API.

## Step 10 — End-of-session recap

After the user is done (they say "done", "stop", or stop responding), print a tight summary:

```
Session summary for Lofi (project owl_proj_…):
  Claimed:        2  → OWL-A12, OWL-B7
  Resolved:       1  → OWL-A12 at v1.4.1
  Silenced:       3  → OWL-N1, OWL-N4, OWL-N5
  Snoozed:        1  → OWL-S2
  Merged:         1  → OWL-D3 into OWL-D1
  Comments added: 14 (one per investigation)
```

This makes it easy to reconstruct the session without scrolling.

## Action / MCP / CLI cross-reference

Every step has both a MCP tool and a CLI fallback. Pick one transport per run — don't mix.

| Action | MCP tool | CLI |
|---|---|---|
| List projects | `mcp__owlmetry__list-projects` | `owlmetry projects --format json` |
| List apps for project | `mcp__owlmetry__list-apps` | `owlmetry apps --project-id <id> --format json` |
| Get app (for `latest_app_version`) | `mcp__owlmetry__get-app` | `owlmetry apps view <app-id> --format json` |
| List issues (paginated) | `mcp__owlmetry__list-issues` | `owlmetry issues list --project-id <id> --status new --format json` |
| Get issue + occurrences | `mcp__owlmetry__get-issue` | `owlmetry issues view <issue-id> --project-id <id> --format json` |
| Breadcrumb timeline | `mcp__owlmetry__investigate-event` | `owlmetry events investigate <event-id> --format json` |
| Add comment | `mcp__owlmetry__add-issue-comment` | `owlmetry issues comment <issue-id> --project-id <id> --body "..."` |
| Claim (→ in_progress) | `mcp__owlmetry__claim-issue` | `owlmetry issues claim <issue-id> --project-id <id>` |
| Resolve (version required) | `mcp__owlmetry__resolve-issue` | `owlmetry issues resolve <issue-id> --project-id <id> --version <v>` |
| Silence (terminal) | `mcp__owlmetry__silence-issue` | `owlmetry issues silence <issue-id> --project-id <id>` |
| Snooze (auto-revives) | `mcp__owlmetry__snooze-issue` | `owlmetry issues snooze <issue-id> --project-id <id>` |
| Reopen (→ new) | `mcp__owlmetry__reopen-issue` | `owlmetry issues reopen <issue-id> --project-id <id>` |
| Merge two | `mcp__owlmetry__merge-issues` | `owlmetry issues merge <target-id> --project-id <id> --source <source-id>` |

## Guardrails (re-stated for the agent's benefit)

- **No auto-claim, no auto-resolve, no auto-silence, no auto-snooze, no auto-merge.** Every status transition is user-confirmed.
- **Resolve requires a version.** If the user can't or won't give one, switch to `snooze` or `silence` — never invent a placeholder.
- **Don't auto-bump the app version.** Ask first; the user may want to batch with other changes.
- **Sub-agents are read + comment only.** They never transition issue status. The parent agent does, and only after explicit OK.
- **Always print the resolved project name + ID at the top of the run** so the user can spot a wrong-project misfire before any state changes.
- **Cap sub-agent parallelism at 5.** Batch the rest sequentially.

## Typical workflow (end-to-end)

```text
User: /owlmetry-investigate-issues

You:  Detected project Lofi (owl_proj_8f3…). Pulling issues…
      Found 14 issues to investigate: 11 new, 3 regressed ⚠️
      Spawning 5 investigation sub-agents (batch 1 of 3)…
      [sub-agents run in parallel, each posts a comment per issue]
      …batch 2 of 3 done
      …batch 3 of 3 done

      | # | Issue | Users | Occ. | Versions | Suggested | Why |
      | 1 | ⚠️ Regressed: Crash in SignatureRenderer.export | 42 | 318 | 1.4.0 (latest) | fix-now | … |
      | 2 | Network error: POST api.revenuecat.com/v1/receipts | 28 | 121 | 1.3.5–1.4.0 | downgrade-sdk-call | … |
      …

      Recommendation:
      - Start with #1 — regressed, affects current release.
      - Batch-downgrade candidates: #2, #5, #8 (network errors with graceful UX).
      - Merge candidates: #11 → #4 (same _error_stack).
      - Stale: #4 (all occurrences on 1.3.0; current is 1.4.0).

      Pick a row (1–14), or `batch-downgrade`, `batch-resolve-stale`, or `merge 11→4`.

User: 1

You:  Claim issue #1 (OWL-A12) — sets status to in_progress, visible to team?

User: yes
      [claim-issue called]
      Searching for the call site of SignatureRenderer.export…
      Found at SignatureCreator/Renderers/SignatureRenderer.swift:142
      The stack points at a force-unwrap on `selectedFont`. Suggested fix: guard let, fall back to system font.
      [edits made]

      What version will this ship in? Current Info.plist version is 1.4.0; I can bump to 1.4.1.

User: yes 1.4.1
      [version bumped, user commits + ships]

You:  Mark OWL-A12 as resolved at 1.4.1?

User: yes
      [resolve-issue called]
      Resolved.

      Next issue, or done?

User: done

You:  Session summary for Lofi:
      …
```
