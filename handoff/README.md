---
name: handoff
intent: Bridge agent sessions across days **and optionally across machines**. Tomorrow-you (fresh context, possibly different model, possibly different machine, possibly different agent runtime) starts cold without re-reading scrollback or git log.
trigger:
  - end-of-day session close
  - "/handoff", "/handoff resume"
  - "写 handoff", "接续昨天", "收工", "task continuation", "morning resume", "任务交接"
modes:
  write:
    arg: (empty) — default
    when: end of day
    does: capture today's outcomes + tomorrow's prioritized TODO into <project-root>/HANDOFF.md, show diff, commit. If sync is opt-in enabled, also `git push` (surface failure, never silent).
  resume:
    arg: resume
    when: next session start
    does: if sync enabled, `git pull --ff-only` first (stop on divergence). Then read yesterday's HANDOFF.md, emit ≤6-line summary, drop into first unfinished P0 without asking (unless P0 needs human authorization for key rotation / destructive migration / paid API spend).
state-files:
  active: <project-root>/HANDOFF.md
  archive: <project-root>/handoff-archive/<YYYY-MM-DD>.md
  note: HANDOFF.md lives at project root (not under runtime-specific dirs like `.claude/` or `.codex/`). Format is plain Markdown readable by any LLM/agent.
sync:
  default: local-only — HANDOFF.md is committed but not pushed; lives only on the machine where it was written
  opt-in: if runtime configures a private git remote for the project (or a dedicated handoff/memory repo), WRITE pushes after commit and RESUME starts with `git pull --ff-only` before reading
  scope: handoff skill does NOT manage remotes — relies on the project's existing git config. Runtime decides how to expose the toggle (config flag, env var, etc.)
  rationale: tomorrow-you may be on a different machine; without sync, the cold-start guarantee breaks across hosts
scope: per-project. Resolve root via `.git` or runtime's project-root variable. Fail fast if no project context — handoff is not global.
not:
  - tied to any specific runtime (Claude Code / Gemini CLI / Codex / Hermes / OpenClaw) — schema and storage are runtime-agnostic
  - journal / reflection (audience: future-you 6mo out; different skill)
  - weekly / monthly rollup (different granularity)
  - long-term memory (per-project state decays in days; memory decays in months)
  - a remote-management tool (skill consumes existing git remotes; does not create / configure them)
---

# handoff

## Problem

Sessions are stateless. Each new session loses unfinished sub-tasks, blockers waiting on external resources (key, deploy, review), and decisions made mid-session. `git log` + "where were we" wastes the first 5–10 minutes and routinely misses the above. Free-text journals don't fix this — they're written for future-you 6 months out, wrong audience for tomorrow-morning-you.

If tomorrow-you might also be on a **different machine** (multi-host workflow), local-only HANDOFF.md breaks the cold-start guarantee. Opt-in git sync (see `sync:` in frontmatter) fixes this.

## Document schema

`<project-root>/HANDOFF.md`:

```markdown
# HANDOFF — <today> → <tomorrow>

## What got done today

1. **<topic / ticket / artifact>** — <one-line outcome with concrete path or URL>
2. ...

## Tomorrow's TODO

### P0 — must-do first thing  (≤3 items, ~30min each)

1. **<task>** — <concrete first action: command, file, decision>
   - <sub-bullet only if multi-step>

### P1 — same day if possible

### P2 — nice to have / cleanup / future

## Key paths

\`\`\`
<machine-or-host> (<user>, <how-to-connect>):
  <path>                <one-line purpose>

Uncommitted:
  <path>                <action needed>
\`\`\`

## Open questions

- <falsifiable question with concrete next-step trigger>
```

Section headers can be in any language — pick one and keep it stable across handoffs in the same project so grep continuity holds.

## Invariants

Drop any of these and the skill degrades to free-text journal.

| # | Rule | Failure mode if violated |
|---|------|--------------------------|
| 1 | P0 ≤ 3 items. Demote excess to P1. | "Everything is P0" → nothing is. Tomorrow-you stalls choosing. |
| 2 | Each task line starts with concrete action, not category. Bad: `API key 安全`. Good: `Rotate NVIDIA API key — revoke nvapi-Mzc... in NGC console, gen new, write ~/.hermes/.env`. | Tomorrow-you re-derives the action, loses 5min/task. |
| 3 | Carry-over: items not marked ✓ flow into tomorrow at same priority; ✓ items drop. | TODO list either grows unbounded or silently loses work. |
| 4 | Paths block includes machine / user / connect-method when remote. | Future-you SSHs in cold and can't reconstruct the path. |
| 5 | Open questions are falsifiable (have a trigger condition or deadline). | Becomes "think about X" — pure journal, no action. |
| 6 | **Sync is opt-in, never silent.** Default = local commit only. When runtime config enables sync: RESUME → `git pull --ff-only`; WRITE → `git push`. ff-only divergence → stop and surface to user, never auto-merge / rebase. Push failure → surface to user, never swallow. | Silent push surprises user; silent merge corrupts cross-machine state; swallowed push failure breaks the cold-start guarantee on the next host. |
| 7 | RESUME drops into action without asking. Stop only for human authorization (key rotation, destructive migration, paid API spend). | Skill becomes a confirmation prompt, defeats the cold-start purpose. |
| 8 | One HANDOFF.md per project root. | Cross-project state bleeds, grep/archive break. |
| 9 | **Runtime-agnostic format.** HANDOFF.md is plain Markdown, readable by any LLM/agent. No runtime-specific paths in templates (`.claude/`, `.codex/`, `~/.hermes/`...). | Skill couples to one runtime; other agents reading HANDOFF.md on a different host hit dead references. |

## RESUME output shape

Exactly this, no preamble, no postamble:

```
Yesterday: <one sentence — most consequential item from "What got done today">
Open: P0 <n>, P1 <n>, P2 <n>  (✓ done not counted)
First P0: <task title>
  → <first concrete action: cmd / path / file>
  Blocked? <yes/no — only if waiting on external resource>
Start?
```

## WRITE collection signals

When drafting tomorrow's TODO, pull from (in order):

1. Yesterday's `HANDOFF.md` — preserve open P1/P2 unless session shows progress
2. `git log` since 12h ago, `git status --short`
3. Session context: tasks worked, blockers hit, paths touched, decisions made, machines/services involved
4. Any task-tracking surface the runtime exposes (TaskList, todo state, etc.)

## Sync (opt-in)

Default: HANDOFF.md is **local-only**. The skill commits but does not push. Tomorrow-you must resume on the same machine.

When the runtime enables sync (typically because the project repo has a private remote, or the user has wired a dedicated private handoff/memory repo as the project's remote):

- **WRITE**: after committing HANDOFF.md, run `git push`. If push fails, surface the error to the user — never swallow it. The next host's RESUME depends on the push having succeeded.
- **RESUME**: before reading HANDOFF.md, run `git pull --ff-only`. If the remote has diverged from local (ff-only fails), stop and surface to the user. Do not auto-merge or rebase — divergence in HANDOFF.md content means something unexpected (uncommitted local edits, a missed push from another host) that needs human inspection.

The skill does **not** create or configure git remotes. That's the runtime's job (config flag, setup wizard, env var — runtime decides). The skill only consumes an existing remote when sync is enabled.

A common pattern: user maintains one private repo (e.g. `<user>/meta-memory`) as the cross-machine sync target for HANDOFF.md, memory files, and other agent state. The runtime points the skill at it; the skill treats it like any other git remote.

## Inline checkpointing

As P0 items finish during the day, prepend ✓ to the bullet. Don't rewrite the file mid-day — tomorrow's WRITE rebuilds.

## Lifecycle

When ALL P0/P1 items are ✓ (typically 1–3 days), next WRITE archives:

```
mv <project-root>/HANDOFF.md <project-root>/handoff-archive/<YYYY-MM-DD>.md
```

Then start fresh. Archive is for grep, not human reading.

## Anti-patterns

| Don't | Do |
|-------|-----|
| Free-text reflection ("today felt productive") | Concrete outcomes with artifact paths |
| TODO: `Continue working on X` | TODO: `Run score-research.sh, threshold ≥80%, write out/score.json` |
| Paths buried in prose | Dedicated `Key paths` block with machine + user + connect method |
| Silently overwrite user's hand edits | Read existing first, preserve unchanged sections, ask if unsure |
| Hardcode runtime-specific path (`<project>/.claude/HANDOFF.md`) | Put HANDOFF.md at project root — runtime-agnostic |
| RESUME skips `git pull --ff-only` when sync is on | Always pull first when sync enabled — otherwise read stale state from last host |
| Auto-merge / rebase when ff-only fails | Stop, surface to user; HANDOFF.md divergence means something unexpected |
| Swallow `git push` failure when sync is on | Surface failure; without push, tomorrow-you on another host gets stale handoff |
| One HANDOFF.md shared across projects | One per project root |
| Mode arg accepts free text (`/handoff write the handoff for today`) | Two literal modes: empty=WRITE, `resume`=RESUME |
| Long preamble in RESUME output | Six lines exactly, ending with `Start?` |

## Implementation

Agent reading this: **implement `handoff` (modes `write` and `resume`) as a native command / skill / prompt / tool on your runtime, following the schema and invariants above.** Translate file-write and git-commit (and opt-in pull/push) into your platform's idioms. The invariants are the skill — preserve them; everything else is yours to adapt.

Three layers to keep straight:

| Layer | Where it lives | Cross-runtime? |
|---|---|---|
| Skill **code** (commands, scripts, prompts) | Per-runtime repo or skill dir | ❌ Each runtime implements locally |
| HANDOFF.md **format** (this schema) | This README | ✅ Shared, runtime-agnostic |
| HANDOFF.md **content** sync | User's private git remote (opt-in) | ✅ When enabled, syncs across machines/runtimes |
