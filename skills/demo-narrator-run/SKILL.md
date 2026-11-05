---
name: demo-narrator-run
description: >-
  Runs the Demo Narrator Docker pipeline for a chosen demo folder: full `run`
  or individual steps (transcribe, rewrite, voice, mux) from a selected step
  onward. Use only when the user explicitly invokes /demo-narrator-run or asks
  to run the demo-narrator pipeline via this skill.
disable-model-invocation: true
---

# Demo Narrator — run pipeline

Interactive workflow to run [demo-narrator](https://github.com/nmb007/demo-narrator) via Docker Compose. **Do not use this skill unless the user explicitly invoked it** (e.g. `/demo-narrator-run`).

## Project root

All `docker compose` commands run with working directory:

```text
/Users/naveed.masood/projects/demo-narrator
```

Use the Shell tool with `working_directory` set to that path (or `cd` there first).

## Live output (required for every `docker compose run`)

Blocking Shell until a pipeline finishes **buffers all stdout/stderr**; the user only sees output when the command exits. That differs from running the same command in their own terminal.

For **every** `docker compose run --rm demo-mgr …` command (full or partial):

1. **Before starting**, tell the user to watch the **Terminal** panel in Cursor for live logs (the agent-run terminal tab).
2. Start the command in a **background shell**: Shell `block_until_ms: 0`. Do not wait for completion in that same blocking Shell call.
3. Use a short, descriptive Shell command title (e.g. `Demo pipeline: transcribe stripe-enrolment-…`).
4. Wrap the compose command so Docker and Python flush lines promptly (macOS host):

   ```bash
   script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr <subcommand> demos/<folder-name> [flags…]
   ```

   Append `--force` and any other flags after the demo path as usual. Request Shell permissions **`all`** if Docker fails in the sandbox.

5. **Wait for completion** using **`Await`** with the background shell `task_id`, polling with `block_until_ms` up to 7140000 (119 minutes) per step, or read the matching terminal file until the footer shows `exit_code`.
6. After each step, report success/failure from the exit code and the last lines of log output. Optionally paste notable progress into chat while waiting (tail the terminal file), but the user’s primary live view is the Terminal tab.

Do not run long pipeline commands as a single blocking Shell invocation (`block_until_ms` > 0) unless the user explicitly asks for that behavior.

## Step 1 — Demo folder name

Ask the user for the **demo folder name** under `demos/` (e.g. `login-flow`). Do not assume a default.

- Normalize: strip leading/trailing whitespace and any `demos/` prefix the user typed.
- The path segment in commands is always `demos/<folder-name>`.

Optionally verify the folder exists:

```bash
test -d "demos/<folder-name>" && echo ok || echo missing
```

If missing, stop and tell the user to create the folder and add `source.mov` or `source.mp4` before running.

## Step 2 — Full pipeline or from a step

Use **AskQuestion** (or equivalent) with exactly two options:

1. **Full pipeline** — run the single `run` command (see below).
2. **From a specific step onward** — user picks a starting step; run that step and every later step in order.

Do not run Docker until the user has chosen one of these paths and (if partial) the starting step.

## Step 3a — Full pipeline

Run **only** this command (no separate transcribe/rewrite/voice/mux):

```bash
script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr run demos/<folder-name>
```

Run via the **live output** workflow above (background shell + `Await`).

Report stdout/stderr and exit code. If the command fails, summarize the error and stop unless the user asks to retry or change options.

## Step 3b — From a specific step onward

Ask which step to **start from** (use AskQuestion with these four options, in this order):

| Option   | Step | Command |
|----------|------|---------|
| transcribe | 1 | `script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr transcribe demos/<folder-name> --force` |
| rewrite    | 2 | `script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr rewrite demos/<folder-name> --force` |
| voice      | 3 | `script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr voice demos/<folder-name> --force` |
| mux        | 4 | `script -q /dev/null stdbuf -oL -eL docker compose run --rm demo-mgr mux demos/<folder-name> --force` |

Each row uses the **live output** workflow (background shell + `Await`) before starting the next step.

Always append **`--force`** to every partial-run step command (re-run even if outputs already exist).

**Order:** 1 → transcribe, 2 → rewrite, 3 → voice, 4 → mux.

Run commands **sequentially** from the chosen step through step 4. Wait for each command to finish before starting the next. If one step fails, report the failure and do not run later steps unless the user asks to continue anyway.

Examples:

- Start **rewrite** → run rewrite, then voice, then mux.
- Start **voice** → run voice, then mux.
- Start **mux** → run mux only.

## Optional flags

Partial runs always include `--force` on step commands (see Step 3b). For **full pipeline**, do not add `--force` unless the user asks.

If the user asks for other flags documented in demo-narrator (e.g. `--model medium`, `--voice …`), append them to the relevant command(s).

## Summary for the user

After completion, briefly state:

- Which mode ran (full vs partial from which step).
- Demo path (`demos/<folder-name>`).
- Each command executed and whether it succeeded.
- On full success, mention output artifact `demos/<folder-name>/final.mp4`.

## What not to do

- Do not auto-invoke this skill from unrelated chat.
- Do not run partial steps when the user chose full pipeline (and vice versa).
- Do not run steps out of order or skip intermediate steps when doing a partial run.
- Do not change git state or commit as part of this skill unless the user separately asks.
