# Agent Memory

Project-scoped, append-only, role-specific memory for the four sub-agents in
[../agents/](../agents/). Each file accumulates durable knowledge across
sessions so agents don't start cold on every task.

## Files

| File | Owner | Purpose |
|------|-------|---------|
| [project-manager.memory.md](project-manager.memory.md) | `@project-manager` | Client functional requirements: branding, color schemes, theming, UX / a11y constraints, NFRs. |
| [designer.memory.md](designer.memory.md) | `@designer` | Durable design-system decisions: palette tokens, type scale, spacing rhythm, motion principles, named component patterns, iconography direction. |
| [tech-lead.memory.md](tech-lead.memory.md) | `@tech-lead` | Tech stack and architectural decisions / pivots with rationale and trade-offs. |
| [fullstack-dev.memory.md](fullstack-dev.memory.md) | `@fullstack-dev` | Debugging postmortems — only for genuinely hard problems the agent got stuck on. |

## How it works

- **Agent-driven.** Each agent reads its own memory file as the first step of
  its workflow and appends new entries as the final step. No hooks, no MCP
  server.
- **Project-local.** Memory lives in `.claude/memory/` inside each project.
  There is no global store — client preferences, stack decisions, and bug
  histories don't cross project boundaries.
- **Committed.** These files are team knowledge; commit them to the repo.
  Do not put secrets, NDAs, or PII in them — generalize first.
- **Append-only, newest-on-top.** Agents prepend new entries below the H1
  header so the most recent (and usually most relevant) context is the
  cheapest to read.

## Entry Templates

### `fullstack-dev.memory.md`
```markdown
## YYYY-MM-DD — <short symptom headline>
- **Symptom**: error / unexpected behavior
- **Context**: files, stack excerpt, runtime conditions
- **Failed approaches**:
  1. A — why it failed
  2. B — why it failed
  3. C — why it failed
- **Root cause**: actual underlying reason
- **Solution**: what fixed it (file path / snippet reference)
- **Tags**: #typescript #prisma #nextjs-app-router
```

### `tech-lead.memory.md`
```markdown
## YYYY-MM-DD — <decision headline>
- **Decision**: chosen approach
- **Old approach**: previous (or "N/A — new")
- **New approach**: replacement
- **Rationale**: why
- **Trade-offs**: what we accept in return
- **Affected areas**: paths / modules / domains
- **Tags**: #state-mgmt #orm #auth
```

### `project-manager.memory.md`
```markdown
## YYYY-MM-DD — <requirement headline>
- **Requirement**: the constraint / preference
- **Source**: which user message or feature surfaced this
- **Category**: branding | theming | ux | a11y | perf | compliance | other
- **Notes**: scope, known exceptions
```

### `designer.memory.md`
```markdown
## YYYY-MM-DD — <decision headline>
- **Decision**: chosen approach (e.g., "8pt spacing rhythm with 4pt half-step")
- **Old approach**: previous (or "N/A — new")
- **New approach**: replacement
- **Rationale**: why
- **Trade-offs**: what we accept in return
- **Affected areas**: which kinds of screens / components
- **Tags**: #palette #typography #spacing #motion #a11y #components
```

## When to Write

- **fullstack-dev**: only after ~3 genuinely distinct failed attempts on a
  non-obvious problem. Skip typos, lint errors, and first-try fixes.
- **tech-lead**: only when a stack / library / pattern is introduced or
  pivoted. Skip routine per-feature choices already covered by existing
  entries.
- **project-manager**: only for durable client preferences that span multiple
  features (branding, palette, theming, a11y bar, UX principles). Skip
  one-off acceptance criteria.
- **designer**: only for durable design-system decisions that span multiple
  features (palette tokens, type scale, spacing rhythm, motion principles,
  named component patterns, iconography direction). Skip per-feature layout
  decisions and one-off copy choices. Brand requirements sourced from the
  client belong in the PM's memory, not here.
- **All agents**: de-duplicate first. If a matching entry exists, update or
  reference it instead of adding a new one.

## When NOT to Write

- Trivial or one-off knowledge.
- Anything that belongs in the actual codebase (code comments, README,
  ADRs the team already maintains).
- Anything sensitive (credentials, client PII, contract details).
