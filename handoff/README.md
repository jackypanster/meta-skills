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
    does: capture today's outcomes + this year's completed goals + tomorrow's prioritized TODO into HANDOFF.md, keep it concise, show diff, commit locally (always). If a remote is configured AND reachable, also `git push` — failure surfaces as warning but does NOT fail WRITE.
  resume:
    arg: resume
    when: next session start
    does: if remote configured AND reachable, `git pull --ff-only` first (stop on divergence; warn-and-continue on network failure). Then read yesterday's HANDOFF.md, emit ≤6-line summary. After user confirms with `Start?`, if other projects in the store have HANDOFFs modified in the last 48h, surface cross-project borrow/deploy candidates and WAIT for user approval before executing them — separate from own-project P0, which still drops in without asking (unless P0 needs human authorization for key rotation / destructive migration / paid API spend).
state-files:
  active: <handoff-store>/<project>/HANDOFF.md
  archive: <handoff-store>/<project>/archive/<YYYY-MM-DD>.md
  store: a local-first git repo (e.g. ~/workspace/meta-memory). May be lazily `git init`'d if absent. Optional remote enables cross-machine sync.
  note: HANDOFF.md never lives under runtime-specific dirs like `.claude/` or `.codex/`. Format is plain Markdown readable by any LLM/agent.
sync:
  contract: local-first, sync best-effort. Local commit is the durable boundary; push/pull are opportunistic on top.
  remote-absent: skill works fully offline. WRITE commits locally; RESUME reads local copy.
  remote-present-online: WRITE additionally pushes; RESUME additionally pulls --ff-only.
  remote-present-offline: push/pull fail → surface warning with unpushed-commit count → WRITE/RESUME still complete using local state. Unpushed commits flush on next successful push.
  divergence: ff-only fail on RESUME stops the skill (cross-host conflict requires human inspection) — distinguished from plain network failure.
  scope: skill never creates or configures remotes. Runtime / user wires `git remote add` once; skill consumes it.
  rationale: tomorrow-you may be on a different machine — but only sometimes. Make the cross-machine path work when conditions allow, and degrade gracefully (offline, no remote yet, first-time machine) without losing content.
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

## Year-to-date completed goals

1. **<goal>** — <concise done signal with concrete artifact path or URL; completed <YYYY-MM-DD or month>>
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

`What got done today` doubles as the **cross-machine / cross-runtime bulletin**: other agents reading another project's HANDOFF via the Cross-project surface in RESUME pull the first 1–2 bullets to judge whether to borrow or deploy. Write the bullets for an LLM/agent on a different host (concrete artifact path, command, JD-ID), not for human reflection.

## Invariants

Drop any of these and the skill degrades to free-text journal.

| # | Rule | Failure mode if violated |
|---|------|--------------------------|
| 1 | Keep the whole handoff concise: "What got done today" ≤5 bullets, "Year-to-date completed goals" ≤7 bullets, each one line unless a sub-bullet is essential. Move raw MR/commit lists and long audit trails to linked artifacts, not HANDOFF.md. | The file becomes a journal/audit log; future-you has to summarize before acting. |
| 2 | P0 ≤ 3 items. Demote excess to P1. | "Everything is P0" → nothing is. Tomorrow-you stalls choosing. |
| 3 | Each task line starts with concrete action, not category. Bad: `API key 安全`. Good: `Rotate NVIDIA API key — revoke nvapi-Mzc... in NGC console, gen new, write ~/.hermes/.env`. | Tomorrow-you re-derives the action, loses 5min/task. |
| 4 | Carry-over: items not marked ✓ flow into tomorrow at same priority; ✓ items drop. | TODO list either grows unbounded or silently loses work. |
| 5 | Paths block includes machine / user / connect-method when remote. | Future-you SSHs in cold and can't reconstruct the path. |
| 6 | Open questions are falsifiable (have a trigger condition or deadline). | Becomes "think about X" — pure journal, no action. |
| 7 | **Local-first, sync best-effort.** Local commit is the durable boundary and always runs. If a remote is configured: WRITE additionally pushes (failure → warn with unpushed-commit count, NEVER fail WRITE); RESUME additionally pulls --ff-only (network failure → warn + fall back to local; ff-only divergence → STOP and surface, never auto-merge / rebase). | Failing WRITE on push error destroys content the user just spent the day producing. Auto-merging cross-host divergence silently corrupts handoff state. Refusing RESUME because the network is down leaves the user with no recoverable session. |
| 8 | RESUME drops into action without asking. Stop only for human authorization (key rotation, destructive migration, paid API spend). | Skill becomes a confirmation prompt, defeats the cold-start purpose. |
| 9 | One HANDOFF.md per project root. | Cross-project state bleeds, grep/archive break. |
| 10 | **Runtime-agnostic format.** HANDOFF.md is plain Markdown, readable by any LLM/agent. No runtime-specific paths in templates (`.claude/`, `.codex/`, `~/.hermes/`...). | Skill couples to one runtime; other agents reading HANDOFF.md on a different host hit dead references. |
| 11 | **Cross-project actions need explicit user approval.** RESUME may surface candidates from other projects' HANDOFFs (mtime ≤48h), but must NOT execute borrow/deploy actions without user disposition. Distinct from invariant 8 (own-project P0 drops in automatically) — different machines have different roles (main-driver / LAN-restricted / GPU host / cloud), and a borrow that helps machine A may break machine B. | Skill silently mutates the local machine based on another machine's session; environments diverge in unpredictable ways. |

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

## Cross-project surface (RESUME, post-`Start?`)

After the user answers `Start?` and **before** dropping into own-project P0, scan
`<handoff-store>/*/HANDOFF.md` for projects other than the current one with
mtime ≤48h. This is the channel by which agents on different machines /
different runtimes notice each other's outcomes (new skills, new tools, new
patterns) and propose borrowing or deploying them locally.

Steps:

1. Enumerate `<handoff-store>/*/HANDOFF.md`, exclude the current project,
   filter to mtime ≤48h.
2. For each, read "What got done today" (first 1–2 bullets).
3. Judge **borrow / deploy value against THIS machine's role and environment**
   — main-driver vs LAN-restricted vs GPU host vs cloud, etc. Machine role
   comes from the runtime's own context (SOUL.md, memory store, env). Skip
   items already present locally or irrelevant to the role.
4. List surviving items as numbered **concrete actions** (clone X, install
   skill Y, port pattern Z) — not vague "consider".
5. **Wait for user approval before executing.** User picks ("1,3" / "skip" /
   "all"). Skipped candidates drop; next RESUME cycle will resurface them if
   still active in source HANDOFFs.
6. Execute approved actions, then drop into own-project P0.

If no candidates survive (no other projects, all >48h old, or nothing matches
this machine's role) — emit nothing. Silence is the default.

Output format (only when at least one candidate exists, appended after the
six-line block once the user has answered `Start?`):

```
Cross-project (last 48h):
  [1] <project>: <one-line outcome> — <concrete action for this machine>
  [2] ...
Approve (e.g. "1,3" / "skip" / "all"):
```

## WRITE collection signals

When drafting tomorrow's TODO, pull from (in order):

1. Yesterday's `HANDOFF.md` — preserve open P1/P2 unless session shows progress
2. `git log` since 12h ago, `git status --short`
3. Session context: tasks worked, blockers hit, paths touched, decisions made, machines/services involved
4. Current-year completed goals from prior handoffs/session context only; do not invent goals from vague history. Keep this as a stable YTD milestone list, not a daily changelog.
5. Any task-tracking surface the runtime exposes (TaskList, todo state, etc.)

## Sync — local-first, best-effort

The contract: **local commit always runs and is the durable boundary**. Network sync is layered on top.

| State | WRITE | RESUME |
|---|---|---|
| No remote configured | commit local; print "saved locally only" hint | read local; print hint |
| Remote configured, network OK | commit local + `git push` | `git fetch` + `git merge --ff-only` + read |
| Remote configured, network DOWN | commit local; warn + show unpushed commit count | warn + fall back to read local |
| Remote configured, ff-only diverges | (push will be rejected — same as offline path: warn, retry next time) | **STOP**, surface conflict to user, do not auto-merge |

Concrete rules:

- **WRITE** never fails because of remote issues. A failed `git push` is a warning, not an error. Unpushed commits accumulate locally and flush on the next successful push. Show the unpushed-commit count so the user knows state is pending.
- **RESUME** falls back to local copy on plain network failure (offline, DNS fail, auth glitch) — local content is always usable. Distinguish this from ff-only divergence, which means another host pushed a conflicting handoff and human inspection is required; auto-merge here would corrupt cross-host state.
- The skill **does not** create or configure git remotes. That's the runtime's / user's job (one-off `git remote add origin git@...`). The skill consumes whatever remote already exists.
- The skill **may** lazily `git init` the handoff store on first use, so a fresh machine works offline immediately. The user opts into sync later by adding a remote.

A common pattern: user maintains one private repo (e.g. `<user>/meta-memory`) as a dedicated handoff-content store, namespaced by project — clean separation from project repos and from runtime-specific memory paths.

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
| Hardcode runtime-specific path (`<project>/.claude/HANDOFF.md`) | Store in a dedicated handoff dir (e.g. `~/workspace/meta-memory/<project>/`), runtime-agnostic |
| Fail WRITE because `git push` failed | Local commit is the durable boundary; push failure is a warning, content is safe |
| Refuse RESUME when offline or remote missing | Fall back to local copy; local is always readable |
| Auto-merge / rebase when ff-only fails | Stop, surface — divergence means cross-host conflict needing human inspection |
| Silently swallow `git push` failure | Surface with unpushed-commit count so user knows there's state to flush later |
| One HANDOFF.md shared across projects | One per project root |
| Mode arg accepts free text (`/handoff write the handoff for today`) | Two literal modes: empty=WRITE, `resume`=RESUME |
| Long preamble in RESUME output | Six lines exactly, ending with `Start?` |

## Implementation

Agent reading this: **implement `handoff` (modes `write` and `resume`) as a native command / skill / prompt / tool on your runtime, following the schema and invariants above.** Translate file-write and git-commit (and opt-in pull/push) into your platform's idioms. The invariants are the skill — preserve them; everything else is yours to adapt.

Three layers to keep straight:

| Layer | Where it lives | Cross-runtime? | Failure mode |
|---|---|---|---|
| Skill **code** (commands, scripts, prompts) | Per-runtime repo or skill dir | ❌ Each runtime implements locally | — |
| HANDOFF.md **format** (this schema) | This README | ✅ Shared, runtime-agnostic | — |
| HANDOFF.md **content** local store | Local handoff repo (lazy-init OK) | ✅ Same Markdown anywhere | Local commit always succeeds — this is the durable boundary |
| HANDOFF.md **content** sync | User's private git remote (opt-in) | ✅ When configured & online | Network/auth failure surfaces as warning, never fails WRITE; divergence stops RESUME |
