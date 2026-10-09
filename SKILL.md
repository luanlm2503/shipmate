---
name: shipmate
description: Use when the user wants to run multiple coding tasks in parallel using Herdr, asks to delegate work to a crew of agents, says things like "spin up a few agents to do X, Y, Z", wants a first-mate/captain workflow without firstmate or WSL, or asks to supervise/check on/collect results from Herdr-managed worker agents.
---

# Shipmate

## Overview

Shipmate turns the host agent into a "first mate": a single liaison that dispatches bounded tasks to disposable worker agents (crewmates), each in its own git worktree and Herdr pane, supervises them to completion, and reports back. It is built entirely on Herdr's native CLI primitives (`herdr worktree`, `herdr workspace`, `herdr agent`) — no firstmate repo, no WSL, no tmux. It runs on Windows, Linux, and macOS wherever Herdr itself runs.

Core rule: the host agent orchestrates and verifies; crewmates execute. Never silently trust a crewmate's self-reported success — always read its output and/or run independent verification before reporting completion to the user.

Prerequisite: `herdr status` must show a running, compatible server. If not, tell the user to run `herdr` once to start it, then retry.

Prerequisite: `jq` must be on PATH (verify with `jq --version`). Without it, JSON responses from `herdr worktree create` / `workspace create` cannot be parsed for IDs.

Known pitfall (verified by testing): if the chosen `--kind` has more than one global install on PATH and one of them is broken (e.g. a stale npm global leftover with a renamed/missing binary), `herdr agent start` can time out with `{"error":{"code":"timeout","message":"timed out waiting for agent startup"}}` even though a working install of that CLI exists elsewhere on PATH. The pane's PATH resolution order is not guaranteed to match the host shell's. Before dispatching at scale, sanity-check the chosen `--kind` once: start it in one throwaway pane/worktree and confirm `agent_status` comes back `idle` (not a timeout). If it fails, tell the user which install is likely shadowing the working one (do not silently work around it or delete anything without confirmation) and suggest trying a different `--kind` or fixing PATH ordering first.

Known pitfall (verified by testing): `herdr agent prompt --wait` can return `agent_prompt_stalled` (no `working`/`blocked` observed within 5000ms) or `timeout` (no terminal status observed within the given timeout) **even when the crewmate received the prompt and completed the task correctly**. This is a detection race, not evidence of failure. Treat either error as "unknown, go check" — never as "the prompt failed" and never as "the task failed." Immediately run `herdr agent get "<task-id>"` to read the real current `agent_status`, and if it shows `idle`/`done`, proceed straight to [Collect and verify](#4-collect-and-verify) as normal. Only escalate or retry the prompt if `agent get` also looks stuck (e.g. still `working` far longer than expected, or `unknown`).

Known pitfall (verified by testing): if you `cd` into a worktree path (or otherwise open a terminal there) before or outside of `herdr worktree create`, Herdr can register a second, separate workspace pointing at the same repo path alongside the one `worktree create` manages. `herdr worktree remove --workspace <id>` only removes the worktree-backed workspace; the stray plain workspace is left behind. After cleanup, always run `herdr workspace list` and `herdr workspace close <id>` on any leftover workspace that still points at a path you just finished with, before telling the user cleanup is done.

## When to use

Use Shipmate when:
- The user has 2+ independent subtasks (fixes, investigations, refactors, audits) that can run in parallel without touching the same files.
- The user explicitly asks to "spin up agents", "run a crew", "delegate to workers", or similar.
- A single task is long-running and the user wants to keep working in the main session while it proceeds in the background.

Do not use Shipmate for a single small task, or when subtasks share heavy file overlap (worktrees isolate git state, not merge conflicts — plan around that).

## Workflow

### 1. Plan

Break the user's request into bounded, independent task units. For each unit, write:
- A short task id (`[a-z][a-z0-9_-]{0,31}`, must be unique among live agents).
- A concrete, evidence-based objective and stop condition (same discipline as any agent-orchestration prompt: no "looks good", only observable conditions).
- Which agent kind to use: `claude`, `opencode`, or `codex` — default to the one already configured/authenticated unless the user specifies otherwise, or pick by task fit (reasoning-heavy → claude; repo-aware/review-heavy → codex; otherwise opencode).

Ask the user where each worktree should live (this project does not use a fixed default) unless they already told you, e.g.:

```bash
git rev-parse --show-toplevel
```

then confirm a path like `<repo-root>/../worktrees/<task-id>` or let the user name it explicitly.

Exception — Telegram-relayed requests (text-only, no interactive UI): never use question dialogs or approval prompts. Use defaults instead (standby objective when none is given; the established repo worktree convention for the path) and state assumptions in the reply.

### 2. Dispatch

For each task, in order:

```bash
# 1. Create an isolated git worktree + workspace, no stolen focus
created=$(herdr worktree create --cwd "<repo-root>" --branch "<task-id>" --path "<confirmed-path>" --label "<task-id>" --no-focus)
workspace_id=$(printf '%s\n' "$created" | jq -r '.result.workspace.workspace_id')
pane_id=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')

# 2. Start the crewmate agent in that pane (must be at an idle shell prompt).
# Always pass the kind's trust-all flag after `--` so crewmates never block on
# permission prompts (worktrees are disposable — this is the user default):
#   claude:   herdr agent start "<task-id>" --kind claude --pane "$pane_id" -- --dangerously-skip-permissions
#   opencode: herdr agent start "<task-id>" --kind opencode --pane "$pane_id" -- --auto
#   codex:    herdr agent start "<task-id>" --kind codex --pane "$pane_id" -- --dangerously-bypass-approvals-and-sandbox

# 3. Hand off the bounded prompt and wait for it to settle
herdr agent prompt "<task-id>" "<prompt-skeleton below>" --wait --until idle --until blocked --timeout 1800000
```

Prompt skeleton (fill in brackets, keep it outcome-based):

```text
Objective: <one concrete outcome>.

Work only inside this worktree. Do not touch files outside it. Preserve any existing uncommitted changes you find.

Before coding, inspect the current repo state and relevant files. Do not make unrelated refactors.

After each meaningful change, run relevant verification (tests/build/lint as applicable). If blocked, stop and explain the blocker — do not claim success without evidence.

Stop only when: <observable completion condition>.

When done, summarize exactly what changed and how you verified it.
```

If `agent start` returns `agent_not_ready` because detection saw `blocked` during startup, inspect with `agent read` before sending the prompt.

### 3. Supervise

Do not poll tightly in a loop burning turns. Prefer:

```bash
herdr agent wait "<task-id>" --until idle --until blocked --until done --timeout <ms>
```

This blocks server-side until the crewmate settles, so you spend no tokens between dispatch and completion. Repeat per task, or dispatch all tasks first (loop step 2 for each) and then wait on each in turn.

If a crewmate reports `blocked` (waiting on an approval/question dialog):

```bash
herdr agent read "<task-id>" --source recent-unwrapped --lines 80
```

Inspect what it's asking, then either:
- Approve/respond via `herdr agent send-keys "<task-id>" <key...>` (e.g. `enter`, or arrow keys + `enter` for a menu), or
- Send a clarifying instruction via `herdr agent prompt "<task-id>" "<correction>" --wait` once unblocked.

Never approve blindly — read the dialog first.

### 4. Collect and verify

Reach this step either because `agent wait`/`agent prompt --wait` returned a terminal status normally, or because you followed the `agent_prompt_stalled`/`timeout` pitfall above and confirmed via `agent get` that the crewmate is actually `idle`/`done`.

When a crewmate reaches `idle`/`done`:

```bash
herdr agent read "<task-id>" --source recent-unwrapped --lines 150
```

Then independently verify in that worktree — do not trust the summary alone:

```bash
cd "<worktree-path>" && git status --short && git log --oneline -5
# plus whatever test/build/lint command fits the project
```

Classify each task as **Mergeable**, **Needs follow-up**, or **Blocker found**, same standard as any agent-review workflow.

### 5. Report

Give the user one consolidated report across all dispatched tasks: task id, agent kind, branch, verification result, and recommended next action (merge, follow-up prompt, or manual look). Never summarize from memory — always re-check state right before reporting if any time has passed.

### 6. Clean up

Only after the user confirms a task's result is handled (merged, or explicitly discarded):

```bash
herdr worktree remove --workspace "$workspace_id"
```

Add `--force` only if the user accepts discarding uncommitted changes in that worktree. Never clean up a worktree whose changes have not been merged or explicitly discarded by the user.

Then check for and close any stray leftover workspace per the `cd`-before-`worktree create` pitfall above:

```bash
herdr workspace list
herdr workspace close "<stray-id>"   # only for workspaces pointing at a path you just finished with
```

## Steering mid-run

If a crewmate drifts (touches unrelated files, repeats a failed fix, optimizes the wrong thing), don't let it keep going:

```bash
herdr agent read "<task-id>" --source recent-unwrapped --lines 150
herdr agent prompt "<task-id>" "Stop. <bounded correction>. Verify with <specific check>. Stop only when <condition>." --wait --until idle --until blocked
```

Prefer one sharply bounded correction prompt over manually taking over implementation yourself.

## Safety

- Always confirm the worktree path with the user before creating it (no fixed default in this setup).
- Never run destructive git commands (`reset --hard`, force-push) in a worktree without explicit user confirmation.
- Never remove a worktree with unmerged, unreviewed changes without the user's explicit go-ahead.
- Treat crewmate self-reported success as a claim, not evidence — always verify independently (tests, git diff, build).
- Keep every dispatch prompt outcome-based with a concrete stop condition — never "make it work" without a check.
- If `herdr status` shows no running/compatible server, stop and tell the user to start Herdr first — do not try to start a server yourself without asking.
