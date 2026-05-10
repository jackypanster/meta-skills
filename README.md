---
repo: meta-skills
intent: Catalog of agent skills described as portable design, with all runtime detail (frontmatter syntax, hook wiring, MCP config, shell idioms) stripped. Each skill = one README.md; agent reads it and implements natively.
audience: agent / LLM (humans secondary — if a human can't follow, they ask an LLM)
consume: read <skill>/README.md → implement on your runtime as slash command / prompt / tool / persona / whatever your platform calls it
contribute: add <skill>/README.md following the schema demonstrated in handoff/
schema-per-skill:
  - frontmatter: name, intent, trigger, modes, state-files, scope, not
  - body: Problem, Design (modes + file layout + document schema), Invariants, Anti-patterns, Implementation (one sentence handing last mile to agent)
non-goals:
  - multi-runtime adaptation layers
  - installer scripts
  - platform-specific code
  - human marketing copy
license: MIT
---

# meta-skills

## When to read a skill from here

| User intent | Skill |
|---|---|
| Bridge agent sessions across days / 接续昨天 / 收工 / end-of-day handoff | [handoff](./handoff) |

## When NOT to use this repo

- You want a working drop-in for one specific runtime → use that runtime's own skill marketplace, not this.
- You want generated code → meta-skills are specs, not implementations.

## Reading order for an agent

1. Match user request → skill via the table above (or grep `intent:` / `trigger:` across `*/README.md`)
2. Read frontmatter first — decide relevance in <5 lines
3. Read body only if relevant
4. Implement on your runtime; preserve every item under `Invariants`
