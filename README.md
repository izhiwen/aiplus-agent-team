# AiPlus Agent Team
[![CI](https://github.com/izhiwen/AiPlus-Agent-Team/actions/workflows/ci.yml/badge.svg)](https://github.com/izhiwen/AiPlus-Agent-Team/actions/workflows/ci.yml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

[中文 README](README.zh-CN.md)

## The pain

You ask the agent to architect a schema, then implement it, then review the
code, then write tests. By the third task it has **drifted**: the same prompt
history now contains product reasoning, implementation hacks, and review
nitpicks, and the output is none of them well.

Worse, the shared context **pollutes** across roles. The engineer-mode transcript
ends up in the PM-mode context. Architectural decisions get buried under
transient debug printlns. Useful long-term facts age out because irrelevant
scratch fills the window.

You try to compensate by giving the agent more hats. But one agent wearing
PM, Architect, Engineer, Reviewer, and QA does each hat **shallowly**. Real
engineering teams divide labor because the work *is* that structured.

### Not these other pains

AiPlus Agent Team is specifically about **role separation and execution**.
Other AiPlus plugins solve adjacent but different problems:

| Plugin | Pain it solves | Why it is not Agent Team |
|---|---|---|
| [AiPlus-Agent-Memory](https://github.com/izhiwen/AiPlus-Agent-Memory) | **amnesia** — agent forgets context between sessions | Gives one agent a memory; does not split roles |
| [AiPlus-Auto-Team-Consultant](https://github.com/izhiwen/AiPlus-Auto-Team-Consultant) | **overlooks** — agent misses onboarding, security, execution pitfalls at plan time | Advises *before* planning; does not execute or persist roles |
| [AiPlus-Compact-Reminder](https://github.com/izhiwen/AiPlus-Compact-Reminder) | **forget** — context recovery after compact | Recovers one agent's context; does not separate roles |
| [AiPlus-Agent-Velocity](https://github.com/izhiwen/AiPlus-Agent-Velocity) | **mis-bills** — estimates anchor on human-engineer hours | Calibrates one agent's estimates; does not structure a team |

Agent Team is the layer that installs a **permanent, executing team** with
isolated workspaces and memory. It sits on top of the other four plugins and
uses them as shared infrastructure.

## What we do about it

**Replace single-agent drift with a permanent team.**

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

## Install

Add the module to your project:

```bash
cd MyProject
aiplus add agent-team
aiplus install codex          # or: claude-code, opencode, all
```

## Quick start

```bash
aiplus agent status              # Show team roster, active experts, warm bench
aiplus agent route engineer-a    # Assign task to engineer-a
aiplus agent integrate engineer-a # Merge engineer-a's branch back into main
aiplus agent audit run           # Run acceptance audit
```

Route a task through the CEO:

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
aiplus agent dismiss security-reviewer  # remove expert from active team
aiplus agent transcript        # show recent activity for audit
aiplus agent prune-worktrees   # clean up stale worktrees
```

## Architecture overview

```
                    AiPlus-Agent-Team               ← orchestration layer
                           ↓ uses
               AiPlus-Auto-Team-Consultant          ← decision-support layer
                           ↓ uses
    AiPlus-Agent-Memory  AiPlus-Compact-Reminder  AiPlus-Agent-Velocity
               ←——————— shared infrastructure layer ———————→
```

Agent Team is the orchestration layer. It sits on top of the four existing
AiPlus plugins and uses them as shared infrastructure:

- **AiPlus-Agent-Memory** — each agent gets a namespaced memory under
  `.aiplus/agent-memory/<role>/`
- **AiPlus-Compact-Reminder** — each long-running agent runs its own compact
  cycle; CEO tracks compact state per agent
- **AiPlus-Agent-Velocity** — each agent has its own velocity records
- **AiPlus-Auto-Team-Consultant** — CEO fires consultant before MEDIUM and
  HEAVY tasks; consultant findings flow into the staffed team's brief

### Five core design decisions

1. **Permanent core team of 8 roles** — installed automatically when the
   plugin is added to a project.
2. **Expert directory** — 11 specialist roles available on-demand, only
   summoned when triggers match.
3. **State-level permanence + warm bench** — agent identity lives on disk;
   process is ephemeral, spawned only when CEO routes a task.
4. **Git worktree workspaces** — each code-touching role gets an isolated
   working directory so two Engineers can work in parallel without silent
   overwrites.
5. **Three-layer memory** — personal (per-agent), team (CEO-shared), and
   project (existing `.aiplus/memory/`). Project memory wins on conflict.

See [`DESIGN.md`](DESIGN.md) for the full design rationale, routing protocol,
memory model, worktree policy, and acceptance criteria.

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

## Contributing

We welcome contributions that stay within the plugin's scope (role separation
and execution, not advisory consulting or estimate calibration).

1. **Open an issue first** for anything larger than a typo fix — the
   `AiPlus-Agent-Team` scope is tightly bounded and we want to avoid
   well-intentioned PRs that collide with other AiPlus plugins.
2. **Follow the existing TOML + markdown persona pattern** — per-agent config
   lives in `.aiplus/agents/<role>.toml`, persona prompt in
   `.aiplus/agents/personas/<role>.md`.
3. **Add adapter parity** — if you change CLI surface, update all three
   adapters (`adapters/codex/`, `adapters/claude-code/`, `adapters/opencode/`).
4. **Run `aiplus agent doctor`** after config changes to validate worktrees,
   memory layout, and TOML schema.
5. **Acceptance criteria** are binding — see
   `.aiplus/agent-team/acceptance/v0.1.4/schema.yaml`. Any behavioral change
   must update the schema and its sibling `.test.sh`.

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

- Main platform: [AiPlus](https://github.com/izhiwen/AiPlus)
- Canonical source: https://github.com/izhiwen/AiPlus-Agent-Team
- Sibling module (applied-economics research):
  [AiEconLab (AEL)](https://github.com/izhiwen/AiEconLab) — 8 research
  roles (Advisor / PI / Theorist / PM / RA-Stata / RA-Python / Referee /
  Replicator) plus 12 experts including an LLM-as-Measurement
  Specialist. Built on the same agent-team architecture; replaces the
  default SWE consultant team with a research-tuned 5-seat consultant
  on install. Coexists in the same project via opt-in.

## Note on the consultant team

`aiplus install` ships `.aiplus/consultant-team.toml` with the
SWE-tuned 5-seat default from
[AiPlus-Auto-Team-Consultant](https://github.com/izhiwen/AiPlus-Auto-Team-Consultant)
(Architecture / UX / Security / Pitfall / AI Integration). If you also
`aiplus add aieconlab` for research workflows, AEL overwrites this
config with its own research-tuned 5 seats (Design Credibility /
Contribution Framing / Day-1 Reproducibility / IRB Gate / LLM-as-
Measurement). v0.2 will let both configs coexist per role tier; v0.1
takes the simpler stance that the most-recently-added team wins.

## Honest disclosure

- Single maintainer (Steve Zhiwen Wang, izhiwen on GitHub).
- v0.1.x. Structural acceptance schema passes; install flow works on
  5 platforms. Some Phase D wiring (real consultant-fire on dispatch,
  owner-gate enforcement at dispatch time) is documented in design but
  not yet wired into the CLI — open issues for the gaps.

## License

[Apache-2.0](LICENSE)
