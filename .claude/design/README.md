# Design Artifacts

Visual design specs and exported images authored by
[`@designer`](../agents/designer.agent.md) and consumed by
[`@fullstack-dev`](../agents/fullstack-dev.agent.md).

## Layout

Each feature gets:

- `.claude/design/<feature-slug>.design.md` — the spec file. Follows the
  `# Design Spec:` template emitted by `@designer`. Visual/behavior only,
  **no technical direction**.
- `.claude/design/<feature-slug>/` — folder of images exported from Stitch
  via the Stitch MCP server. The spec file references these with relative
  markdown image links.

## Rules

- **Committed.** These files are project truth for visual direction. Commit
  them to the repo.
- **Local only.** Never link to Stitch's hosted UI from a spec file —
  hosted URLs expire or require auth. Every image referenced must live in
  the feature's sibling folder.
- **Do not hand-edit specs.** Specs are owned by `@designer`. If the tech
  lead or developer needs clarification, route feedback through
  `@project-manager` so the designer updates the spec.
- **No secrets, no PII.** Generalize before committing.

## Authored by
[`@designer`](../agents/designer.agent.md)

## Consumed by
[`@tech-lead`](../agents/tech-lead.agent.md) (as planning input) and
[`@fullstack-dev`](../agents/fullstack-dev.agent.md) (as implementation truth).
