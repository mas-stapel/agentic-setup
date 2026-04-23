# agentic-setup

> A reusable Claude Code agent workflow for fullstack Node.js development, designed to be installed as a git submodule in any project.

## What's included

### Agents

- `@project-manager` — Orchestrator. Receives feature requests, decomposes them into an ordered delivery plan, and coordinates the other agents end-to-end.
- `@tech-lead` — Architect. Performs adversarial review of feature requests and produces detailed, step-by-step implementation plans.
- `@fullstack-dev` — Implementer. Executes implementation plans step-by-step, writing production-quality TypeScript and Jest tests.

### Skills (shared expertise loaded automatically by relevant agents)

- `react-expert` — React 18+, Next.js App Router & Pages Router patterns, hooks, state management, testing, and accessibility.
- `typescript-expert` — Strict TypeScript, Express typed handlers, Jest AAA testing, AirBnb ESLint, and project file conventions.

### Memory system

- Role-specific, append-only memory files per agent (`project-manager`, `tech-lead`, `fullstack-dev`).
- Memory is project-local — each project accumulates its own architectural decisions, requirements, and postmortems.
- Committed to the repo so knowledge persists across sessions and team members.

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
> ln -s ../../.agentic-setup/.claude/agents/tech-lead.agent.md .claude/agents/tech-lead.agent.md
> ln -s ../../.agentic-setup/.claude/agents/fullstack-dev.agent.md .claude/agents/fullstack-dev.agent.md
> ln -s ../../.agentic-setup/.claude/skills/react-expert .claude/skills/react-expert
> ln -s ../../.agentic-setup/.claude/skills/typescript-expert .claude/skills/typescript-expert
> ```

### 3. Initialize project memory

Copy the memory templates into your project. Do **not** symlink — memory must stay project-local so each project accumulates its own knowledge independently.

```bash
mkdir -p .claude/memory
cp .agentic-setup/.claude/memory/*.md .claude/memory/
```

### 4. Add read permissions

Create or update `.claude/settings.json` to allow the agents to read the submodule:

```json
{
  "permissions": {
    "allow": [
      "Read(.agentic-setup/.claude/**)"
    ]
  }
}
```

### 5. Commit the setup

```bash
git add .gitmodules .agentic-setup .claude
git commit -m "Add agentic-setup submodule"
```

---

## Using the agents

Open Claude Code in your project and hand off a feature request to the project manager:

```text
@project-manager I want to add user authentication with email + password. Use JWT for sessions.
```

The pipeline runs automatically:

```text
You → @project-manager → @tech-lead → @project-manager → @fullstack-dev → @project-manager → You
```

1. **Project Manager** decomposes your request, asks clarifying questions if needed, and produces a Feature Request Document.
2. **Tech Lead** performs an adversarial review and produces a detailed implementation plan.
3. **Project Manager** forwards the plan to **Fullstack Dev**.
4. **Fullstack Dev** implements each step, writes tests, self-reviews, and reports back.
5. **Project Manager** verifies completion and reports the final outcome to you.

---

## Keeping the submodule up to date

Pull the latest agent definitions and skills from this repo:

```bash
git submodule update --remote .agentic-setup
git add .agentic-setup
git commit -m "Update agentic-setup submodule"
```
