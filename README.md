# agentic-setup

> A reusable Claude Code agent workflow for fullstack Node.js development, designed to be installed as a git submodule in any project.

## What's included

### Agents

- `@project-manager` — Orchestrator. Receives feature requests, decomposes them into an ordered delivery plan, and coordinates the other agents end-to-end.
- `@designer` — Visual UI specialist. Ideates multiple design variations, generates concrete visuals via the Stitch MCP server, and writes a visual/behavior design spec for the fullstack-dev to implement against. Runs for any feature with frontend scope.
- `@tech-lead` — Architect. Performs adversarial review of feature requests and produces detailed, step-by-step implementation plans. References (not duplicates) the design spec for UI-producing steps.
- `@fullstack-dev` — Implementer. Executes implementation plans step-by-step, writing production-quality TypeScript and Jest tests. Treats the design spec as non-negotiable visual truth.

### Skills (shared expertise loaded automatically by relevant agents)

- `react` — React 18+, Next.js App Router & Pages Router patterns, hooks, state management, testing, and accessibility.
- `typescript` — Strict TypeScript, Express typed handlers, Jest AAA testing, AirBnb ESLint, and project file conventions.

### Memory system

- Role-specific, append-only memory files per agent (`project-manager`, `designer`, `tech-lead`, `fullstack-dev`).
- Memory is project-local — each project accumulates its own architectural decisions, requirements, design-system choices, and postmortems.
- Committed to the repo so knowledge persists across sessions and team members.

### Design artifacts

- Visual design specs live at `.claude/design/<feature-slug>.design.md` alongside a sibling folder of exported images at `.claude/design/<feature-slug>/`.
- Authored by `@designer`, consumed by `@tech-lead` (as planning input) and `@fullstack-dev` (as implementation truth).
- Specs are visual/behavior only — no libraries, file paths, or component APIs. Those belong to the tech lead's plan.
- Committed to the repo so designs persist and can be re-run through Stitch later.

---

## Using this setup in your project

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- Git

### 1. Add as a git submodule

```bash
git submodule add https://github.com/mas-stapel/agentic-setup.git .agentic-setup
git submodule update --init --recursive
```

This clones the setup into a hidden `.agentic-setup/` directory at your project root, keeping it out of the way.

### 2. Symlink agents and skills

Claude Code discovers agents from `.claude/agents/` and skills from `.claude/skills/` in your project root. Symlink these directories to the submodule so updates to the submodule propagate automatically.

```bash
mkdir -p .claude
ln -s ../.agentic-setup/.claude/agents .claude/agents
ln -s ../.agentic-setup/.claude/skills .claude/skills
```

> If your project already has a `.claude/agents/` or `.claude/skills/` directory, symlink individual files instead:
>
> ```bash
> ln -s ../../.agentic-setup/.claude/agents/project-manager.agent.md .claude/agents/project-manager.agent.md
> ln -s ../../.agentic-setup/.claude/agents/designer.agent.md .claude/agents/designer.agent.md
> ln -s ../../.agentic-setup/.claude/agents/tech-lead.agent.md .claude/agents/tech-lead.agent.md
> ln -s ../../.agentic-setup/.claude/agents/fullstack-dev.agent.md .claude/agents/fullstack-dev.agent.md
> ln -s ../../.agentic-setup/.claude/skills/react .claude/skills/react
> ln -s ../../.agentic-setup/.claude/skills/typescript .claude/skills/typescript
> ```

### 2.5. Create the design artifacts directory

```bash
mkdir -p .claude/design
```

Design specs produced by `@designer` are written here as `<feature-slug>.design.md` with a sibling folder of exported Stitch images. They are committed with the rest of the project.

### 3. Initialize project memory

Copy the memory templates into your project. Do **not** symlink — memory must stay project-local so each project accumulates its own knowledge independently.

```bash
mkdir -p .claude/memory
cp .agentic-setup/.claude/memory/*.md .claude/memory/
```

### 4. Add read permissions

Create or update `.claude/settings.json` to allow the agents to read the submodule and the design artifacts directory:

```json
{
  "permissions": {
    "allow": [
      "Read(.agentic-setup/.claude/**)",
      "Read(.claude/design/**)"
    ]
  }
}
```

### 4.5. Configure the Stitch MCP server (required for `@designer`)

The `@designer` agent generates visuals through the Stitch MCP server. It is a **hard requirement** — the agent will refuse to run if Stitch MCP is not configured and no design spec will be produced without it. The rest of the pipeline (`@project-manager`, `@tech-lead`, `@fullstack-dev`) still works for non-design features without it.

Copy the example config and fill in Stitch's actual server details:

```bash
cp .agentic-setup/.mcp.json.example .mcp.json
# edit .mcp.json and replace the REPLACE_ME placeholders with the
# command / args / env published by the Stitch team
```

If `.mcp.json` contains an API key, add it to `.gitignore` and document the required env vars in your project README instead of committing secrets.

### 5. Commit the setup

```bash
git add .gitmodules .agentic-setup .claude
git commit -m "Add agentic-setup submodule"
```

The `.claude/design/` directory is tracked even when empty (add a `.gitkeep` if needed) so design specs can be committed into it as features ship.

---

## Using the agents

Open Claude Code in your project and hand off a feature request to the project manager:

```text
@project-manager I want to add user authentication with email + password. Use JWT for sessions.
```

The pipeline runs automatically:

```text
You → @project-manager → [@designer if frontend scope] → @project-manager → @tech-lead → @project-manager → @fullstack-dev → @project-manager → You
```

1. **Project Manager** decomposes your request, asks clarifying questions if needed, and produces a Feature Request Document.
2. **Designer** (only if the feature has any frontend scope) asks further clarifying questions, presents three distinct design variations, generates visuals through the Stitch MCP server, and writes a design spec at `.claude/design/<feature-slug>.design.md`.
3. **Tech Lead** performs an adversarial review, reads the design spec if one was produced, and produces a detailed implementation plan that references the spec rather than duplicating visual direction.
4. **Project Manager** forwards the plan to **Fullstack Dev**.
5. **Fullstack Dev** implements each step, reads the design spec when UI is in scope, writes tests, self-reviews, and reports back.
6. **Project Manager** verifies completion and reports the final outcome to you.

You can also invoke `@designer` directly for standalone design work (no feature pipeline required), e.g.:

```text
@designer I want to redesign our dashboard's empty state. Make it feel more welcoming.
```

---

## Keeping the submodule up to date

Pull the latest agent definitions and skills from this repo:

```bash
git submodule update --remote .agentic-setup
git add .agentic-setup
git commit -m "Update agentic-setup submodule"
```
