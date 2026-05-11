# AiPlus Agent Team
[中文 README](README.zh-CN.md)

## The pain

You ask the agent to architect a schema, then implement it, then review the
code, then write tests. By the third task it has drifted: the same prompt
history now contains product reasoning, implementation hacks, and review
nitpicks, and the output is none of them well.

Worse, the shared context pollutes across roles. The engineer-mode transcript
ends up in the PM-mode context. Architectural decisions get buried under
transient debug printlns. Useful long-term facts age out because irrelevant
scratch fills the window.

You try to compensate by giving the agent more hats. But one agent wearing
PM, Architect, Engineer, Reviewer, and QA does each hat shallowly. Real
engineering teams divide labor because the work *is* that structured.

## What we do about it

AiPlus Agent Team installs a permanent virtual team of eight core roles into
your project: Advisor, CEO, Architect, PM, two Engineers, Reviewer, and QA.
Each role has its own persona, workspace, and memory namespace. The Owner
talks only to Advisor and CEO; the CEO orchestrates the rest.

The records cover:

- **Role isolation** — each agent loads only its own persona and personal
  memory. An Engineer does not see the PM's reasoning, and vice versa.
- **Git worktree workspaces** — code-touching roles get isolated working
  directories so two Engineers can work in parallel without stepping on each
  other. Conflicts surface through git, not silent overwrites.
- **Three-layer memory** — personal (per-agent), team (CEO-shared), and
  project (existing `.aiplus/memory/`). Project memory wins on conflict, so
  a team-of-the-day decision never overrides durable project consensus.
- **Expert directory** — eleven specialists (AI Integration, Security
  Reviewer, Tech Writer, DevOps, UI Designer, Researcher, and others) sit
  dormant until the CEO summons them for tasks that match their triggers.
- **Adaptive routing** — the CEO scores each task (LIGHT, MEDIUM, HEAVY) and
  staffs only the roles that are needed. A quick question gets a direct
  answer; a high-risk change gets the full council.

No daemon. No cloud sync. No upload. Each agent is state-level permanent —
its files live on disk, but the process is ephemeral, spawned only when the
CEO routes a task.

## Quick start

If you already have AiPlus installed:

```bash
cd MyProject
aiplus install codex          # or: claude-code, opencode, all
aiplus agent status
```

`aiplus agent status` shows the team roster, active experts, and warm-bench
state.

Then route a task through the CEO:

```text
aiplus agent route "refactor the billing module"
```

The CEO scores the task, picks the right team members, and reports back.

Other everyday commands:

```bash
aiplus agent doctor            # validate configs, worktrees, memory layout
aiplus agent list              # list all roles (core + expert)
aiplus agent talk architect    # direct conversation with one role
aiplus agent invite security-reviewer   # add an expert to the active team
aiplus agent integrate engineer-a       # merge engineer-a's branch into main
```

## What's inside

- `core/templates/` — TOML configs and markdown personas for all 8 core
  roles, plus the team-wide `agent-team.toml`
- `core/templates/personas/` — role persona prompts (advisor, ceo,
  architect, pm, engineer, reviewer, qa)
- `core/docs/` — design rationale, routing protocol, memory model,
  worktree policy, safety boundaries
- `adapters/codex/` — Codex plugin and skill assets
- `adapters/claude-code/` — Claude Code project-local commands and agents
- `adapters/opencode/` — OpenCode project-local config, commands, and prompts
- `examples/` — synthetic examples for all three runtimes

## Safety boundaries

AiPlus Agent Team does not:

- upload agent state, persona, memory, or transcript to any service
- run as a background daemon or persistent process
- store secrets in any agent's persona, memory, or workspace
- modify global agent configuration (~/.codex, ~/.claude, etc.)
- modify another project's `.aiplus/`
- automatically approve Owner-gated actions (push, tag, release, deploy)
- introduce new network calls beyond what the host runtime already makes

## More

- Main platform: [aiplus](https://github.com/izhiwen/aiplus)
- Canonical source: https://github.com/izhiwen/aiplus-agent-team

## License

[Apache-2.0](LICENSE)
