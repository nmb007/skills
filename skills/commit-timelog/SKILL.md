---
name: commit-timelog
description: >-
  Groups today's git commits into ERP timelog tasks and writes them (max 3h per
  entry) only after explicit user approval. Use when the user runs
  /commit-timelog or asks to turn today's commits into ERP timelogs.
disable-model-invocation: true
---

# Commit → ERP timelog

End-of-day workflow: read **today's commits** in the current repo, **group related work** into a small set of tasks, then **log time in ERP** for each task.

This skill does not create, update, or transition Jira issues.

## Required inputs (ask if missing)

Collect from the user message before doing writes:

| Input | Purpose |
|-------|---------|
| **ERP project name** | Exact name from ERP (or `project_id` if the user provides it). |

Optional (use sensible defaults):

- **Date** — default **today** in the user's local timezone; "today" means commits whose author date falls on that calendar day.
- **Hours per task** — ask if not stated; **each ERP entry must be ≤ 3 hours**. If the user gives one total for the day, split across tasks (each line ≤ 3h).
- **ERP tag (label)** — default **`Coding`** on every timelog entry; use another label only if the user specifies one.

## Preconditions

- Run from the **git repository** whose commits should be counted (current workspace root unless the user names another path).
- **ERP MCP** must be available and authenticated.
- **ERP timelogs require explicit user approval** — never call `timelogs_upsert_entry` until the user clearly approves the proposed entries (see Step 4).

## Step 1 — Fetch today's commits

Use the repo root (or user-specified path). **Author date** defines "today" unless the user gives an explicit `YYYY-MM-DD`.

```bash
# Replace DATE with YYYY-MM-DD (today if not specified)
git log --since="DATE 00:00:00" --until="DATE 23:59:59" \
  --pretty=format:"%H%x09%an%x09%s" --no-merges
```

For each commit hash you plan to cite, pull enough context for grouping and the ERP description:

```bash
git show --stat --format=fuller <hash>
```

If there are **no commits** for that day, stop and tell the user — do not create empty timelogs.

## Step 2 — Group commits into tasks

**Goal:** Fewer, clearer timelog lines — not one line per commit unless each commit is unrelated.

Group when commits:

- Touch the same **plugin / area / feature** (e.g. same top-level folder or frankenstyle component).
- Are part of one **user-visible outcome** (e.g. "Totara notifications migration" + "remove legacy messages.php").
- Share a **theme** (tests, webhooks, settings, lang strings) for the same deliverable.

Keep **separate** groups when work is unrelated (different features, unrelated bugfixes, or different repos/plugins with no shared goal).

For each group, draft:

- **Title** — short, imperative, scannable (≤ ~80 chars if possible). This becomes the ERP task title.

Derive a **repo/plugin label** for ERP from the git root folder name or dominant path in the diff (e.g. `enrol_stripepayment`). Use `[plugin_name]` in ERP descriptions.

Show the user a **grouping table** (task title, commits, proposed hours) and proceed only after they confirm the grouping, or if they already stated hours and grouping in this run.

## Step 3 — Read existing ERP entries for that day

Before proposing writes, call `timelogs_get_day` for the log date.

ERP matches an entry by its **description text** on that date and project. The same text **replaces** that day's hours. Different text creates a **second** entry.

If a proposed description already exists for that day and project, tell the user and ask whether to skip it or replace its hours. Do not upsert a duplicate under a slightly different wording unless the user asks for a separate line.

## Step 4 — ERP timelogs (explicit approval required)

**Hard gate:** Do **not** create or update any ERP timelog until the user **explicitly approves** logging time. Phrases like "go ahead with ERP", "approve timelogs", "yes, log the time", or equivalent count as approval. Generic "run the routine" alone is **not** enough when hours or grouping were not already confirmed.

Present a **proposed timelog table** and **stop**:

| Task | ERP description (preview) | date | hours | tag | ERP project |
|------|---------------------------|------|-------|-----|-------------|

Default **tag** column value: `Coding` (unless the user asked for a different ERP label).

Ask: *"Approve these ERP timelog entries? I will not write to ERP until you confirm."*

- If the user declines, adjusts hours/descriptions, or does not approve: **skip all ERP writes**.
- If the user approves: proceed below for **each** approved row (one timelog line per task unless the user asks otherwise).

1. Resolve project: `timelogs_list_projects` if needed; use the user's **ERP project name** (full name if disambiguation required).
2. Ensure the project is on the **week** containing the log date; if upsert fails, follow MCP guidance (`timelogs_check_week_project`, user adds project in ERP UI).
3. `timelogs_upsert_entry` (only after approval):
   - `date` — same calendar day as the commits (or user override)
   - `hours` — **> 0 and ≤ 3** per entry
   - `label_name` — **`Coding`** by default; resolve via `timelogs_list_labels` if needed. Use a different `label_name` or `label_id` only when the user overrides the default tag.
   - `description` — exact pattern:

```text
[<plugin_or_repo>] <Short title>
```

Example:

```text
[enrol_stripepayment] Add webhook handler and automated test suite
```

Rules:

- Bracket segment is the plugin/repo shorthand from Step 2.
- Title after the bracket is the task title from Step 2 (may be slightly shortened to fit the ERP **500 character** limit).
- **Never** exceed 3 hours on a single entry; split work across multiple tasks or multiple days instead of one 8h line.
- **Always** set tag **`Coding`** on timelog entries unless the user specified another ERP label for this run.
- Sending the same description again on the same date and project **replaces** hours. It does not add to them.

## Step 5 — Summary for the user

Reply with:

1. Table: **task title** | commits (short) | hours logged (or "not logged — awaiting ERP approval" if Step 4 was skipped)
2. The ERP description written for each task
3. Any failures (ERP week/project, ambiguous project name) and what the user must fix manually

## Error handling

- **Duplicate routine same day:** If the user re-runs, compare proposed descriptions with `timelogs_get_day` before writing. Ask whether to skip tasks already logged or replace their hours.
- **Ambiguous ERP project name:** The MCP refuses ambiguous names — list matches from `timelogs_list_projects` and ask once.
- **Project missing from the week:** Report the week from the error. The user adds the project in the ERP web app; this skill cannot attach it.

## What not to do

- Do not create, edit, or transition Jira issues.
- Do not call ERP timelog tools without **explicit user approval** for the proposed entries.
- Do not log more than **3 hours** per timelog entry in the ERP.
- Do not force-push, amend commits, or change git history as part of this skill.
