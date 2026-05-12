# AiPlus Agent Team Design

Status: draft v0.1.2 (post-six-round-acceptance-schema-review)
Acceptance schema (binding): `.aiplus/agent-team/acceptance/v0.1.4/schema.yaml`
Scope: local-first permanent agent team, with adaptive coordinator and on-demand
expert directory, designed to compose with the existing AiPlus plugin stack.

---

## 1. One-Line Positioning

`aiplus-agent-team` installs a permanent virtual team of AI agents into a
project so that an Owner can run a multi-role engineering organization with
two human-facing entry points (Advisor and CEO) and an adaptive set of
internal specialists, all without leaving the local machine.

In plain language:

> A single AI agent in a long session forgets, drifts roles, and pollutes its
> memory across unrelated tasks. AiPlus Agent Team replaces that single agent
> with a small standing team — Advisor, CEO, Architect, PM, two Engineers,
> Reviewer, QA — each with its own workspace, memory, and persona, plus an
> on-demand expert directory the CEO can pull from when a project warrants
> it. The Owner only talks to Advisor and CEO; the CEO orchestrates the rest.

---

## 2. Problem

A real Owner running AI coding agents day to day hits three recurring failures:

1. **Roles drift inside one agent.** A single Codex / Claude Code / OpenCode
   session is asked to plan, then code, then review, then write tests. The
   same prompt-history blends product reasoning, implementation hacks, and
   review nitpicks. The agent quietly slips out of one role into the next
   and produces output that is none of them well.

2. **Memory pollution.** A single shared memory means the Engineer-mode
   transcript ends up in the PM-mode context, and vice versa. Architectural
   decisions get buried under transient debug printlns. Useful long-term
   facts age out of the active window because of irrelevant scratch.

3. **No serious division of labor.** Real engineering teams have a PM, an
   Architect, two Engineers, a Reviewer, and a QA because the work *is* that
   structured. Forcing one agent to wear all hats means it does each hat
   shallowly.

The existing AiPlus plugins each address part of the problem:

- `aiplus-agent-memory` gives one agent a clean, project-local memory.
- `aiplus-compact-reminder` keeps one agent's context recoverable.
- `aiplus-auto-team-consultant` consults a virtual expert team *before* a
  plan is written, but does not execute and does not persist.
- `aiplus-agent-velocity` calibrates one agent's estimates.

None of them address the role-and-execution problem. `aiplus-agent-team` is
the layer that does.

---

## 3. What This Plugin Is Not

To prevent scope drift, four explicit non-goals:

1. **Not a virtual consultant.** That is `aiplus-auto-team-consultant`.
   Consultant *advises* the CEO before a plan; team *executes* the plan.
   A team member is a real persistent agent that touches files; a consultant
   lens fires only at planning time and never owns code.

2. **Not a multi-process daemon.** The team is State-level permanent
   (Section 6), not process-level permanent. There is no agent daemon, no
   IPC, no socket, no service to start at boot.

3. **Not a remote-execution platform.** All agents run inside the Owner's
   existing host runtime (Codex, Claude Code, OpenCode). Nothing uploads,
   nothing syncs to cloud, nothing runs on someone else's machine.

4. **Not a replacement for Owner judgment.** STOP-gated actions
   (push, tag, release, deploy, secret access, external account changes)
   always escalate to the Owner. The team can prepare and recommend, but
   never auto-approve.

---

## 4. Solution Overview — Five Core Decisions

The plugin rests on five architectural decisions, each consciously chosen
over plausible alternatives:

1. **Permanent core team of 8 roles**, installed automatically when the
   plugin is added to a project (§5).
2. **Expert directory** of 11 specialist roles available on-demand, only
   summoned when triggers match (§5.3).
3. **State-level permanence + warm bench** (not process-level daemon, not
   pure ephemeral) — agent identity lives on disk, fast cache for repeat
   calls (§6).
4. **Git worktree workspaces** (not shared branches, not folder copies) —
   parallel, conflict-tracked, atomically revertible (§7).
5. **Three-layer memory** (personal / team / project) with project as the
   most authoritative layer (§8).

A sixth design ingredient — the adaptive coordinator inside CEO — is not a
new invention but a **reuse** of `aiplus-auto-team-consultant`'s scoring +
scaling rules (§9). One algorithm covers consult-time decisions and
execute-time staffing.

---

## 5. Team Structure

### 5.1 Owner-Facing Tier (always 2)

| Role | Responsibility | Default invocation |
|---|---|---|
| **Advisor** | Strategic conversation. *Decides what to do.* Decision support, second opinion, tradeoff explanation. | Owner directly |
| **CEO** | Execution coordination. *Gets it done and reports.* Receives tasks, scores, dispatches, integrates, reports back. | Owner directly |

The Owner sees only these two by default. As a clean rule of thumb:

- "Should I do X or Y?" / "Is it worth doing X?" → **Advisor**
- "Do X" / "What's the status of X?" / "Is X done yet?" → **CEO** (status
  questions stay with CEO even though they may sound like Advisor; CEO
  owns the source of truth for in-flight work)

### 5.2 Internal Core Team (always 6)

| Role | Responsibility | Default state |
|---|---|---|
| **Architect** | System design, data flow, tech selection, "what is hard to undo later" | Active |
| **PM** | Requirements decomposition, scope, user-visible behavior, acceptance criteria | Active |
| **Engineer A** | Implementation. Vacant specialty by default; CEO tags per task (backend / frontend / data / infra) | Active |
| **Engineer B** | Same as Engineer A. **Dormant by default**; activates only when CEO explicitly parallelizes (HEAVY tier or Owner request) | Dormant unless invoked |
| **Reviewer** | Code review and design review against acceptance criteria; not a substitute for QA | Active |
| **QA** | Tests, dogfood, regression, "did this actually meet acceptance" | Active |

Total core team = 2 owner-facing + 6 internal = **8 roles**.

### 5.3 Expert Directory (on-demand, not core)

Eleven specialists are pre-defined and ready to summon. **For v0.1, six
"high-frequency" experts ship implemented; the remaining five ship as
config stubs and become functional in v0.2.** All are State-level permanent
(persona, memory, workspace template exist as files), but only enter a
project's *active* team when summoned.

| Expert | Trigger conditions | v0.1 status |
|---|---|---|
| **AI Integration Specialist** | `llm`, `prompt`, `model`, `agent`, `context`, `embed` | shipped |
| **Security Reviewer** | `auth`, `payment`, `pii`, `secret`, `compliance`, `crypto` | shipped |
| **Tech Writer** | `readme`, `docs`, `api reference`, public-facing release | shipped |
| **DevOps / SRE** | `deploy`, `ci`, `prod`, `monitoring`, `incident` | shipped |
| **UI Designer** | Any task touching visible UI surface | shipped |
| **Researcher** | "look up", "best practice for", "benchmark", new domain | shipped |
| **Data Analyst** | `metric`, `experiment`, `cohort`, `funnel`, `analytics` | v0.2 stub |
| **Customer Researcher** | `interview`, `feedback`, `user study`, `persona` | v0.2 stub |
| **Performance Engineer** | `slow`, `latency`, `throughput`, `profile` | v0.2 stub |
| **Accessibility Specialist** | `a11y`, `screen reader`, `wcag`, `contrast` | v0.2 stub |
| **Compliance Reviewer** | `gdpr`, `pci`, `hipaa`, `audit`, regulatory keyword | v0.2 stub |

A summon is reversible — when the project no longer needs the expert, the
CEO drops it via `aiplus agent dismiss`. State files remain on disk for
re-summoning.

### 5.4 Recommended Initial Activation by Project Type

| Project type | Core 8 + summoned experts |
|---|---|
| AI / agent product (e.g. AiPlus itself) | + AI Integration + Security Reviewer + Tech Writer |
| SaaS billing backend | + Security Reviewer + DevOps + Compliance (v0.2) |
| Open-source CLI tool | + Tech Writer + Accessibility (v0.2) + DevOps |
| Internal tool / script | core 8 only |
| Research prototype | + Researcher + Data Analyst (v0.2) |
| Mobile app | + UI Designer + Accessibility (v0.2) + Performance (v0.2) |

---

## 6. Permanence Model: State + Warm Bench

### 6.1 What "permanent" means here

An agent is **State-level permanent**: its persona, memory, workspace, and
last-known state live as files on disk. The agent process itself is
ephemeral — spawned on demand, runs the task, writes new state to disk,
exits.

This is the opposite of process-level permanent (a daemon that lives in
RAM between calls). The trade-off in plain terms:

- **Process-level wins** on raw response speed (no disk read on each call)
  and theoretical "always-on" awareness.
- **Process-level loses** on cost (always burning RAM/CPU), recoverability
  (process death = state loss until daemon restart), portability (cannot
  cleanly move between machines), and host-runtime fit. The AiPlus host
  runtimes (Codex, Claude Code, OpenCode) are themselves ephemeral-process
  tools, so a State-level agent is the natural fit.

We choose State-level. We recover speed via the warm bench (§6.2).

### 6.2 Warm Bench (hot cache)

Every agent's state stays cached in memory for a configurable TTL after
each invocation. While on the bench, a subsequent invocation bypasses the
disk-read step and resumes instantly.

This gives the experience of a colleague who is still in the room, without
the cost of a permanent residence.

### 6.3 Warm Bench under an ephemeral CLI — important caveat

Each `aiplus agent route` (and similar) is its own CLI process. When that
process exits, in-memory cache is gone. So **pure in-memory warm bench only
helps when multiple agent calls happen inside a single CLI invocation**
(e.g., CEO orchestrating several team members within one `aiplus agent
route` call) — not when the user repeatedly types `aiplus agent route` at
their shell.

Two options to address this:

| Option | Behavior | v0.1 plan |
|---|---|---|
| **A — In-process only** | Cache lives only inside one CLI process. Repeat shell calls always cold-start. Simple, no extra disk footprint. | **default** |
| **B — On-disk fast cache** | Cache snapshots persist under `~/.cache/aiplus-agent-team/<project>/<role>.cbor` with TTL. Survives across CLI invocations within a session. Requires Owner consent because cached state contains the agent's prompt + memory snapshot. | **v0.2 opt-in** via `aiplus agent cache --enable-disk` |

v0.1 ships Option A. Owner who wants the full "30-minute warm" experience
across shell calls will opt into Option B in v0.2 after explicitly
acknowledging the disk-cache tradeoff. **This is a meaningful difference
from the colloquial "暖凳子 30 分钟" mental model — Owner should be aware:
in v0.1, only intra-process speedup; in v0.2, true cross-call warmth, but
with disk footprint for cached state.**

### 6.4 Default TTLs

Per-role default warm-bench TTL, calibrated to typical work pattern:

| Role | Default TTL | Rationale |
|---|---|---|
| Advisor | 60 min | Long Owner conversations |
| CEO | 60 min | Long-running orchestration |
| Architect | 30 min | Long technical reasoning |
| Engineer A / B | 30 min | Coding flow continuity |
| PM | 15 min | Conversation-style work |
| Reviewer | 15 min | Reviews finish in one pass |
| QA | 15 min | Test cycles are bounded |
| **All experts** | **5 min** | One-shot lookups; experts rarely sit across multiple turns |

Each role's TTL is overridable in `.aiplus/agents/<role>.toml`.

### 6.5 Cache Invalidation Triggers

Long TTL only works safely with active invalidation. The cache is dropped
for the affected agent(s) when any of these events fire:

| Event | Invalidates |
|---|---|
| `aiplus memory add` / `forget` | All agents (project memory changed) |
| Any agent commits to its `agent/<role>` branch | Reviewer (must reread diff) |
| **CEO merges any `agent/<role>` branch into `main`** | **All agents whose worktree reads `main`** (project state changed) |
| CEO assigns an agent to a *new task type* (not a continuation) | That agent (avoid stale context bleed) |
| Owner edits `.aiplus/agents/<role>.toml` | That agent (persona changed) |
| `aiplus agent reset <role>` | That agent (manual escape hatch) |
| Process exit | All (cache is process-scoped in v0.1) |

---

## 7. Workspace Model: git worktree

### 7.1 Why git worktree

Each non-trivial agent that writes code needs an isolated workspace so two
Engineers can work without stepping on each other. Three options:

- Independent folders: simple but merging is manual diff hell
- Shared branch with checkouts: serial, no parallelism
- **Git worktrees**: each agent gets its own working directory backed by
  the same repo, all changes tracked, conflicts surfaced by git itself,
  atomic rollback per agent

Worktrees win because we get parallel filesystem isolation *plus* automatic
conflict detection *plus* per-agent revert without human diffing.

### 7.2 Layout

```
project-root/
├── .git/                           # main repo
├── .aiplus/
│   ├── agents/
│   │   ├── advisor.toml            # persona + TTL + memory namespace
│   │   ├── ceo.toml
│   │   ├── architect.toml
│   │   ├── pm.toml
│   │   ├── engineer-a.toml
│   │   ├── engineer-b.toml
│   │   ├── reviewer.toml
│   │   ├── qa.toml
│   │   ├── personas/               # markdown persona files referenced by .toml
│   │   │   ├── advisor.md
│   │   │   ├── ceo.md
│   │   │   └── ...
│   │   └── _experts/               # expert directory (not yet summoned)
│   │       ├── ai-integration.toml
│   │       ├── security-reviewer.toml
│   │       └── ...
│   ├── agent-team.toml             # team-wide config + active expert list
│   ├── agent-memory/
│   │   ├── shared/                 # team decision log + role assignments
│   │   └── <role>/                 # per-agent personal memory
│   └── agent-workspaces/
│       └── README.md               # explains worktree layout (v0.1: pointer
│                                   #   only; v0.2 may move worktrees here)
├── ../project-root.engineer-a/     # git worktree for Engineer A
├── ../project-root.engineer-b/     # git worktree for Engineer B
├── ../project-root.architect/      # git worktree for Architect
└── ...                             # one worktree per code-touching role
```

Owner-facing roles (Advisor, CEO, PM) do not need a worktree — they reason
and report, they do not write code. Engineers, Architect, Reviewer, QA each
get a worktree.

Worktrees live in sibling directories (`../project-root.<role>/`) by
default to keep the project root clean. This is configurable.

### 7.3 Worktree lifecycle

- Created lazily on first agent task that requires writing code
- Branch naming: `agent/<role>` (e.g. `agent/engineer-a`, `agent/reviewer`)
- CEO integrates by merging `agent/<role>` back into `main` (or whatever
  branch the project uses) when the task completes — explicit via `aiplus
  agent integrate <role>`
- Stale worktrees pruned by `aiplus agent prune-worktrees` (manual; never
  automatic, to avoid losing in-flight work)

### 7.4 Conflict policy

- Two agents claiming the same file: second one STOPs and returns blocked.
  CEO redistributes.
- Two agents in different worktrees both touch the same file in their own
  branch: handled by `git merge` at integration time. **CEO arbitrates**;
  if the conflict requires a design call (not a mechanical merge), CEO
  consults Architect for an opinion but the decision and the merge commit
  remain CEO's responsibility.
- Owner can read any worktree at any time, but should not write to a
  worktree directly while the agent is on the warm bench (would force a
  cache invalidation).

---

## 8. Memory Model: Three-Layer

### 8.1 Layers

| Layer | Path | Who writes | Who reads |
|---|---|---|---|
| **Personal** | `.aiplus/agent-memory/<role>/` | That agent only | That agent only |
| **Team** | `.aiplus/agent-memory/shared/` | CEO only | All team members |
| **Project** | `.aiplus/memory/` (existing) | Any agent via `aiplus memory add` | All team members + Owner |

### 8.2 What goes where

- **Personal**: Agent's working notes, intermediate reasoning, role-specific
  conventions ("I am Engineer A, I prefer to write tests first"). Not seen
  by other agents. Useful for keeping each agent's context clean.
- **Team**: Decisions made by CEO that everyone needs to follow. Current
  task assignments. Open questions awaiting Owner. Architectural decisions
  that crossed from Architect's personal notes into binding policy.
- **Project**: The shared facts of the project (already managed by
  `aiplus-agent-memory`). Naming conventions, accepted patterns, owner
  preferences. Survives across team membership changes.

### 8.3 Read order — weakest to strongest authority

When an agent's context is loaded, files are read in this order. **Later
layers override earlier on conflict** — so the most authoritative layer
(Project) wins:

1. **Personal memory** — read first, gives the agent its private baseline
   and preferences
2. **Team memory** — overrides personal when CEO has issued a binding team
   decision
3. **Project memory** — overrides everything; the project consensus wins
   over any individual agent's preference or any single team-of-the-day
   decision

Why this direction: an Engineer's private "I like `var`" preference must
not silently override the project-level "always use `const`" decision.
Project-wide consensus is the highest authority because it represents
durable Owner intent. Individual agent preferences are the lowest because
they are easily polluted by a single bad session.

Conflicts are surfaced to the agent (and logged in team memory), not
silently resolved.

### 8.4 Owner writing to team memory

Owner does not write directly to team memory (that is CEO's namespace).
Owner has two paths to broadcast guidance:

- **For project-level facts**: `aiplus memory add` — goes to project layer,
  which is the highest-authority layer and reaches everyone
- **For team-of-the-day decisions**: ask CEO ("CEO, please tell the team
  X") — CEO writes the decision into team memory and broadcasts cache
  invalidation

### 8.5 Redaction

All three layers go through the existing 12 `aiplus-agent-memory` redaction
patterns before write. Adding a team member does not change this contract.

---

## 9. Coordinator (lives inside CEO)

### 9.1 Reuse Auto Team Consultant's algorithm

The CEO does not invent a new dispatcher. It reuses the
`aiplus-auto-team-consultant` Coordinator scoring + scaling rules. The same
complexity (1–5) and risk (0.0–1.0) scoring applies. The output is just
different: consultant scaling tells CEO *which lenses to consult*; team
scaling tells CEO *which team members to staff*.

### 9.2 Default scaling for team staffing

| Tier | Trigger | Who staffs the task | Consultant fires? |
|---|---|---|---|
| **LIGHT (no-code)** | complexity ≤ 2 AND no file write expected (e.g., a quick question, status check) | CEO answers directly | no |
| **LIGHT (code)** | complexity ≤ 2 AND requires code change | One Engineer | no |
| **MEDIUM** | complexity 3–4 | One Engineer + Reviewer; Architect if design impact | yes |
| **HEAVY** | complexity ≥ 5 OR risk ≥ 0.7 | PM + Architect + 2 Engineers + Reviewer + QA + applicable experts | yes |

Risk ≥ 0.7 always pulls in Security Reviewer (or any expert whose triggers
match), regardless of tier — even for a LIGHT-code task.

### 9.3 The two-step CEO flow

For MEDIUM and HEAVY tasks (LIGHT skips this for speed):

1. **Plan step (consult)**: CEO fires `aiplus-auto-team-consultant` to get
   the lens-based critique. This is read-only advice; consultant lenses do
   not touch files.
2. **Execute step (staff)**: CEO uses the team-staffing scaling to pick
   actual team members and worktrees. These members write the code.

So consultant is the advisor before staffing; team is the workforce.

LIGHT tasks bypass consultant to keep small work fast. The CEO can
optionally fire consultant on a LIGHT task by Owner request.

---

## 10. Configuration Schema

### 10.1 Per-agent file: `.aiplus/agents/<role>.toml`

Path resolution rules (referenced from this file):
- `system_prompt_file` is resolved **relative to `.aiplus/agents/`**
- `worktree_path` template variable `{project_name}` is the **basename of
  the project root directory** (not the absolute path)

```toml
schema_version = "1.0"

[agent]
role = "engineer-a"
display_name = "Engineer A"
tier = "internal"          # owner_facing | internal | expert
default_specialty = ""     # CEO sets per task: backend | frontend | data | infra
warm_bench_ttl_seconds = 1800   # 30 min default

[persona]
system_prompt_file = "personas/engineer.md"   # resolved under .aiplus/agents/
voice = "engineer"
escalation_target = "ceo"  # who this agent escalates to when blocked

[workspace]
needs_worktree = true
worktree_branch = "agent/engineer-a"
# {project_name} = basename of project root, e.g., "aiplus-public"
worktree_path = "../{project_name}.engineer-a"

[memory]
personal_dir = ".aiplus/agent-memory/engineer-a"
read_team_memory = true
read_project_memory = true
write_team_memory = false   # only CEO writes shared team memory

[invocation]
# v0.1: only no-space aliases; v0.2 may add quoted-arg parsing for spaces
chinese_aliases = ["工程师A", "EngineerA"]
english_aliases = ["engineer", "engineerA", "eng-a"]
```

### 10.2 Team-wide file: `.aiplus/agent-team.toml`

```toml
schema_version = "1.0"

[team]
project_name = "billing-backend"
core_roles = [
  "advisor", "ceo",
  "architect", "pm",
  "engineer-a", "engineer-b",
  "reviewer", "qa",
]

# Active experts for this project. Add by `aiplus agent invite <expert>`.
active_experts = [
  "security-reviewer",
  "ai-integration",
]

[coordinator.light]
complexity_max = 2
fire_consultant = false

[coordinator.medium]
complexity_min = 3
complexity_max = 4
fire_consultant = true

[coordinator.heavy]
complexity_min = 5
risk_threshold = 0.7
fire_consultant = true

[owner_interface]
# Owner sees only these by default. Other roles addressable via `aiplus agent talk`.
default_visible = ["advisor", "ceo"]
allow_direct_talk_to_others = true   # escape hatch
```

### 10.3 Expert directory entries: `.aiplus/agents/_experts/<expert>.toml`

Same schema as 10.1, but with `tier = "expert"`. Sits dormant until
`aiplus agent invite <expert>` adds the expert's role to `active_experts`.

---

## 11. CLI Surface

```
aiplus agent status             # show team roster, active experts, warm bench
aiplus agent doctor             # validate configs, worktrees, memory layout
aiplus agent list               # list all roles (core + expert)
aiplus agent invite <role>      # add an expert to active team
aiplus agent dismiss <role>     # remove expert from active team (state preserved)
aiplus agent disable <role>     # temporarily turn off a core agent w/o removal
aiplus agent enable <role>      # re-enable a previously disabled agent
aiplus agent route <task>       # CEO scores and dispatches; default entry point
aiplus agent talk <role>        # Owner direct conversation with one role
aiplus agent reset <role>       # invalidate warm bench cache for one role
aiplus agent integrate <role>   # explicit merge of agent/<role> branch into main
aiplus agent transcript <role>  # show recent activity for one agent (audit)
aiplus agent prune-worktrees    # clean up stale worktrees (asks confirm)
```

**No `aiplus agent install`.** Provisioning the team into a project goes
through the existing top-level command:

```
aiplus install <runtime>        # provisions all enabled AiPlus modules,
                                # including agent-team if the module is added
aiplus add agent-team           # adds agent-team to the project's module set
                                # (then `aiplus install` provisions it)
```

This matches how `agent-memory`, `compact-reminder`, etc. are installed.
There is no separate `agent install` flow.

Chinese aliases (per the recent CLI translation work):

| Chinese | English |
|---|---|
| `团队` / `团队状态` | `agent status` |
| `派单` / `分配任务` | `agent route` |
| `跟 X 聊` / `找 X` | `agent talk <role>` |
| `召唤 X` / `请 X` | `agent invite <role>` |
| `让 X 走` / `解散 X` | `agent dismiss <role>` |
| `合并 X` / `集成 X` | `agent integrate <role>` |
| `看 X 的活` / `X 的记录` | `agent transcript <role>` |
| `清理团队工作区` | `agent prune-worktrees` |

---

## 12. Integration with the Existing AiPlus Stack

| Plugin | How agent-team uses it |
|---|---|
| `aiplus-agent-memory` | Each agent gets a memory namespace under `.aiplus/agent-memory/<role>/`. Existing redaction applies unchanged. |
| `aiplus-compact-reminder` | Each long-running agent (Engineer, Architect) runs its own compact cycle. CEO tracks compact state per agent. |
| `aiplus-agent-velocity` | Each agent has its own velocity records (Engineer's estimate model differs from PM's). |
| `aiplus-auto-team-consultant` | CEO fires consultant before any MEDIUM- or HEAVY-tier task. Consultant lens findings flow into the staffed team's brief. |

Stack diagram — the four existing plugins are **shared infrastructure**;
agent-team is the orchestration layer that sits on top. Velocity is not
"below" memory and compact; all three are peer infrastructure that any
agent uses.

```
                   aiplus-agent-team               ← orchestration layer
                          ↓ uses
              aiplus-auto-team-consultant          ← decision-support layer
                          ↓ uses
   aiplus-agent-memory   aiplus-compact-reminder   aiplus-agent-velocity
              ←——————— shared infrastructure layer ———————→
```

---

## 13. Owner Interaction Model

### 13.1 Default

Owner sees and addresses two roles: Advisor and CEO. Everything else is
hidden behind CEO orchestration.

- "Should I do X or Y?" → Advisor
- "Do X" / "What's the status of X?" → CEO

### 13.2 Escape hatches

- `aiplus agent talk <role>` opens a direct one-to-one conversation with
  any role, bypassing CEO. Use case: Owner does not understand a piece of
  code Engineer A wrote and wants to ask Engineer A directly.
- `aiplus agent reset <role>` forces a cache flush + clean reload of that
  role. Use case: agent confused; restart its short-term context.
- `aiplus agent disable <role>` temporarily takes a core agent out of the
  CEO's dispatch list without losing its state.
- Any time Owner runs a normal `aiplus memory add` it broadcasts an
  invalidation to all team caches. Owner does not need a special command.

---

## 14. Failure Modes & Fallbacks

| Failure | Fallback |
|---|---|
| Cache miss on agent invocation | Read state from disk (5s overhead); proceed |
| Agent config TOML malformed | Skip that agent, log to `agent doctor`, surface in `aiplus agent status` |
| Worktree corrupted | Mark agent unavailable, ask Owner to manually inspect |
| Two agents claim same file | Second blocks; CEO redistributes |
| **Same role invoked twice concurrently** | Per-role lock; second invocation waits or errors with "agent busy"; never silently interleaved |
| **Disk full / sibling worktree dir unwriteable** | Fail-fast with explicit `WORKSPACE_PROVISIONING_ERROR` and Owner-facing remediation hint |
| **Personal memory exceeds context window** | Auto-truncate oldest personal records (keep latest 200 by default, configurable); never silently drop team or project memory |
| **Owner kills host runtime mid-CEO-orchestration** | All worktrees and personal memory survive (State-level); Owner can recover by re-routing the same task. Team memory may have one orphan "in-flight assignment" entry; `aiplus agent doctor` flags it |
| CEO crashes mid-task | Other agents' worktrees and personal memory survive; Owner can recover by re-routing the task |
| Personal memory file conflicts with team memory | Team memory wins; conflict logged, not silently overridden |
| Expert summoned but never used | No-op cost; expert's state files exist but are not loaded into context |

---

## 15. Privacy & Safety Boundaries

`aiplus-agent-team` does not:

- upload any agent state, persona, memory, or transcript to any service
- write cache to disk in v0.1 (warm bench is RAM-only)
- store secrets in any agent's persona, memory, or workspace
- modify global agent configuration (~/.codex, ~/.claude, etc.)
- modify another project's `.aiplus/`
- automatically prune worktrees (in-flight work might be lost)
- spawn agents in another project's directory
- expose agent state through telemetry, logging, or analytics
- send any data to a remote LLM that is not already configured by the host
  runtime (Codex / Claude Code / OpenCode); the plugin does not introduce
  new network calls

Honest caveat on RAM: we do not deliberately persist warm-bench cache to
disk, but the OS may swap process memory. Truly preventing swap requires
`mlock()`-class primitives that we do not use in v0.1. If Owner is in a
high-sensitivity context and swap-to-disk is unacceptable, mitigation is
operational (disable swap on the host) rather than in-plugin.

### 15.1 Honest caveat on GPG strong_mode (sentinel soft anchor)

The Auditor's GPG-signed release manifest (see §22 and the acceptance
schema at `.aiplus/agent-team/acceptance/v0.1.4/schema.yaml`) protects
against schema tampering and unauthorized manifest writes by agents. It
does **NOT** provide hard isolation in lifestyle-solo deployments where
the agent runtime shares an OS user with Owner.

Concrete limits in same-OS-user mode:

- Agent technically can read `~/.gnupg/` private key files.
- Agent technically can use `gpg-agent` cached passphrase.
- Agent technically can shell out to simulate Owner actions, including
  `touch`-ing the first-run setup sentinel file.

The sentinel mechanism for first-run setup is a **soft trust anchor**:
it relies on the prior that a helpful, task-focused agent will not
spontaneously perform unprompted privilege-escalation setup (the agent
has not been asked to, and "create sentinel then run setup-gpg" is not
in its instruction space). This prior holds in practice for current
LLM agents but is not a cryptographic guarantee.

Owners deploying high-sensitivity projects should consider:

- Running the AiPlus agent runtime under a separate OS user that lacks
  read access to the Owner's `~/.gnupg/` directory (advanced; out of
  v0.1 scope).
- Hardware-backed signing key (YubiKey / Secure Enclave) — under
  consideration for v0.3.
- Layer 7 cross-provider auditor (different LLM, different machine) —
  under consideration for v0.3.
- Or accept the v0.1 boundary and supplement with periodic Owner
  spot-checks via `aiplus agent audit owner-feedback`.

This caveat is intentionally surfaced in the schema's
`gpg_security_limitations` block as `acknowledged: true` so Owners are
informed of the boundary, not silently exposed.

The Owner can audit:

- Persona files: `cat .aiplus/agents/<role>.toml`
- Team memory: `cat .aiplus/agent-memory/shared/*`
- Personal memory: `cat .aiplus/agent-memory/<role>/*`
- Worktree state: `git -C ../project-name.<role> log`
- Cache contents: `aiplus agent status --verbose` (memory dump of warm bench)
- Recent agent activity: `aiplus agent transcript <role>`

---

## 16. STOP Gates

Without explicit Owner approval, the plugin and its agents do NOT:

- `git push` to any remote
- create git tags
- publish to any package manager (crates.io / npm / Homebrew / Docker /
  marketplace)
- delete or archive any GitHub repository
- modify host-machine global configuration
- introduce any network call beyond what the host runtime already makes
- copy private profile data into agent state
- automatically decline an Owner gate without surfacing it to Owner
- enable disk-cache (Option B in §6.3) without an explicit
  `aiplus agent cache --enable-disk` consent step

---

## 17. MVP Roadmap

Estimated effort, not committed dates. Reality may extend.

### v0.1 — Static team (estimate: ~1 week)

- Define the 8 core roles with persona files and TOML configs
- Define the 6 high-frequency expert directory entries (functional) +
  5 stub entries
- Implement git worktree provisioning per code-touching role
- Implement three-layer memory (personal / shared / project)
- Implement `aiplus agent status / doctor / list / talk / route / reset /
  invite / dismiss / disable / enable / integrate / transcript /
  prune-worktrees`
- Owner manually invokes `aiplus agent route engineer-a "implement payment endpoint"`
- CEO routing is rule-based simple; no consultant integration yet
- Warm bench is **in-process only** (Option A in §6.3); cross-CLI calls
  cold-start
- Acceptance: Owner can sequentially route Engineer A on task 1 and
  Engineer B on task 2, both worktrees coexist, diffs visible separately,
  CEO can integrate one and reject the other. (True intra-session
  parallelism is v0.4 work; v0.1 demonstrates the *workspace isolation
  primitive* the parallelism will later use.)

### v0.2 — Plugin integration + opt-in disk cache (estimate: ~1 week)

- Hook into `aiplus-agent-memory` for namespaced memory
- Hook into `aiplus-compact-reminder` for per-agent compact
- Hook into `aiplus-agent-velocity` for per-agent estimates
- Hook into `aiplus-auto-team-consultant` for pre-staffing consult on
  MEDIUM and HEAVY tasks
- Implement the 5 v0.1-stub experts as functional
- Add `aiplus agent cache --enable-disk` (Option B in §6.3) with explicit
  Owner consent
- `aiplus agent doctor` validates the integration
- Acceptance: Owner enables disk cache, exits and re-opens CLI within 30
  min, second `aiplus agent route` for the same role hits cache (visible
  in `aiplus agent status --verbose` as `cache_source=disk_warm`).

### v0.3 — Adaptive coordinator (estimate: ~1–2 weeks)

- CEO scores tasks (complexity + risk) and applies scaling rules
- Auto-summoning of experts when triggers match
- Per-agent TTL configuration honored
- Acceptance: Owner says "实现支付接口" → CEO scores 4/0.85 → fires
  consultant lenses → staffs Engineer A + Reviewer + Security Reviewer →
  all run in their own worktrees → CEO integrates and reports back without
  Owner intervention.

### v0.4 — Cross-runtime parallelism (estimate: TBD)

- One Engineer in Codex, another in Claude Code, simultaneously
- Requires solving inter-runtime message passing
- Out of scope for initial release; flagged as future work

---

## 18. Known Limitations

- **No true concurrency in v0.1–v0.3.** Agents take turns inside a single
  host-runtime session. v0.4 explores parallelism across runtimes.
- **Worktree disk cost.** Each code-touching agent gets its own worktree.
  For a large repo, eight worktrees may use ~2–4 GB. Owner can restrict
  worktree creation to tasks that need it.
- **Cache memory cost.** Default warm-bench cache is small (~50 MB total
  across team) but a heavy session can push higher. Owner can lower TTLs
  per role.
- **In-process-only cache (v0.1).** Cross-CLI calls cold-start. Disk-warm
  cache is v0.2 opt-in.
- **No automatic expert inference for novel domains.** Triggers are
  keyword-based; a project in a domain whose vocabulary is not in any
  expert's trigger list will not auto-summon. Owner can manually invite.
- **Persona drift.** A long-lived agent's personal memory accumulates
  preferences and idiosyncrasies over months. `aiplus agent reset <role>`
  is the manual lever; auto-pruning is provided only at the
  context-overflow boundary (§14), not on a schedule, in v0.1.

---

## 19. Decisions (formerly "Open Questions")

The questions raised during design review are now decided. All defaults
below were approved by the Owner; the doc records them as binding for
v0.1 and may be revisited per release.

| # | Question | DECISION |
|---|---|---|
| 1 | Worktree placement default | **Sibling** `../project.role/`. Clean root tree. Revisit after v0.1 if Owner finds parent-dir clutter unacceptable. |
| 2 | CEO ↔ Owner update cadence | **Respond to Owner queries + brief async note when a HEAVY task completes.** No proactive chatter. Refine in v0.3. |
| 3 | Expert summoning permanence | **Stays for project duration** once invited. Owner can `aiplus agent dismiss` to remove explicitly. |
| 4 | Persona file format | **Persona prompt in markdown, all metadata in TOML.** `system_prompt_file` in `.toml` points to a `.md` file under `.aiplus/agents/personas/`. |
| 5 | Agent-vs-agent disagreement | **CEO arbitrates using consultant lenses as tiebreaker.** If still tied, escalate to Owner via Advisor. Refine in v0.3. |
| 6 | Personal memory expiry | **No scheduled auto-prune in v0.1.** Manual `aiplus agent reset <role>`. Auto-truncation only at context-overflow boundary (§14, keep latest 200 personal records). |

---

## 20. Glossary

- **Permanent agent**: a role whose persona, memory, workspace, and state
  persist across sessions on disk. Process is ephemeral.
- **State-level permanent**: see §6.1. The opposite of process-level
  permanent (no daemon).
- **Warm bench**: in-memory cache of an agent's state for a configurable
  TTL after invocation. See §6.2 and the in-process caveat in §6.3.
- **Core team**: the 8 roles installed by default in every project.
- **Expert directory**: the 11 specialist roles defined globally; only
  active in a project after `aiplus agent invite`.
- **Worktree**: a git worktree, one per code-touching agent. See §7.
- **Three-layer memory**: personal / team / project. Project layer wins
  on conflict. See §8.
- **Coordinator**: the scoring + scaling logic inside CEO that decides
  which roles staff a task. Reuses `aiplus-auto-team-consultant`'s
  algorithm.
- **Summoning**: CEO adding an expert from the directory to the project's
  active team for the duration of the project (Owner may dismiss).
- **Integration**: CEO merging an `agent/<role>` branch back into the
  project's main branch via `aiplus agent integrate <role>`.

---

## 21. Out of Scope for v0.1

To keep MVP shipping discipline:

- No web UI for team status (CLI only)
- No persistent process (no daemon)
- No remote agents (everything local)
- No agent self-modification of own persona (Owner edits config files)
- No automatic Owner-gate approval (always escalates)
- No multi-project team-sharing (one team per project). The
  `aiplus-public` install-projects registry tracks *which projects have
  AiPlus*, not *which agents staff which project* — these are
  complementary but separate concerns. The agent-team config lives entirely
  inside `.aiplus/` of each project.
- No metrics collection (use `aiplus-agent-velocity` per agent if needed)
- No 5 stub experts as functional implementations (deferred to v0.2)
- No disk-warm cache (deferred to v0.2 opt-in)

---

## 22. Acceptance Criteria & Auditor (frozen)

The binding acceptance criteria schema for v0.1 lives at:

```
.aiplus/agent-team/acceptance/v0.1.4/schema.yaml
```

This file is the contract between Team CEO's claims and what counts as
"actually done". It was developed across six independent review passes
(`acceptance_criteria_schema_v0.1.0.yaml` → `v0.1.4`) and is now frozen
as the binding spec for v0.1 implementation.

### 22.1 What the schema specifies

- **3 acceptance modes** — `deterministic` / `llm_judge` / `owner_review`,
  with strict hierarchy: deterministic is authoritative; llm_judge always
  routes to Owner; owner_review skips Auditor entirely.
- **3 check kinds** — `exit_code` / `file_exists` / `shell_output_match`.
  Forbidden by design: `ask_user`, raw LLM judgment, arbitrary shell
  primitives that don't reduce to a deterministic verdict.
- **7 working deliverables** as canonical examples, including the v0.1
  acceptance scenario from §17.
- **`bin_aliases`** for cross-platform commands (`{sha256}`, `{timeout}`,
  etc.) so audit scripts work on Linux and macOS without rewrites.
- **Auditor independence**: 6 layers per §6.3 plus a 7th (cross-provider)
  in v0.3 — task source, info source, weak tool isolation, output path,
  prompt injection defense, fail-closed enforcement, optional model
  independence.

### 22.2 Critical anti-overclaim mechanisms

- **`canary_replay`** — weekly count-based + risk-weighted re-execution
  of historical PASS audits against current state. Owner-in-loop on
  candidate overclaims (no automatic penalty without Owner confirmation).
- **`audit_reproducibility_drift`** — same deliverable + same script +
  same file state but different verdict across audit runs = LLM
  hallucination signal. Routes both runs to Owner spot-check queue.
- **`owner_spot_check`** — Owner can flag any deliverable's verdict via
  one-line command; ground-truth feedback flows into Auditor accuracy
  metrics over a 30-day window.
- **`owner_feedback_inline`** — every Audit packet ends with a 5-second
  feedback prompt that does NOT require Owner to remember commands;
  copy-paste is enough.

### 22.3 First-run trust anchor (sentinel + GPG)

The first time `aiplus agent audit setup-gpg` runs, the wizard refuses
to proceed unless Owner has manually created the sentinel file at
`.aiplus/agent-team/.owner-setup-authorized` containing Owner's name
and email in YAML. This is a soft anchor (see §15.1) but blocks the
agent-self-setup attack: an agent invoking setup-gpg cannot complete
the trust chain without out-of-band Owner action.

Generated GPG key uses EDDSA + Curve25519 (fallback RSA-4096 on older
GPG versions), non-empty passphrase required, gpg-agent cache TTL
recommended ≤600 seconds.

### 22.4 Goodhart-resistant `.test.sh`

Every audit script has a sibling `.test.sh` exercising the script
against ≥1 pass-fixture and ≥4 fail-fixtures covering the 4 fail modes
(absent / partial / corrupted / **behavioral**). The behavioral fail
mode catches "structure correct but runtime wrong" — the most common
LLM-agent failure mode and the one most likely to escape simpler
testing. `.test.sh` must be reviewed by a Reviewer agent that did not
author the corresponding audit script.

If a deliverable genuinely has no behavioral fail mode (rare; e.g.,
pure file-existence checks), it may mark `behavioral=not_applicable`
but MUST list which existing fail modes substitute and provide a
1-line justification.

### 22.5 30-day data-driven evolution

After v0.1.4 lands in production, the following metrics drive v0.1.5
and v0.2 patches (not engineering intuition):

| Metric | Threshold | Action if tripped |
|---|---|---|
| `auditor_overclaim_rate_window_30d` | > 5% | Upgrade `audit_passes` default |
| `canary_replay_false_positive_rate` | > 20% | Recalibrate canary |
| `canary_dropped_this_run` (≥3 weeks) | sustained | Propose raising `sample_size_cap` |
| % deliverables marking `behavioral=not_applicable` | > 30% | Escape hatch abuse — strict default |
| `ownership_unverified` events | high frequency | Sentinel UX problem |
| GPG passphrase entry failure rate | high | `gpg-agent default-cache-ttl` too short |

The schema explicitly documents that thresholds (5%, 20%, 30%) are
engineering intuition at freeze time; production data will revise them.

### 22.6 Out-of-scope for v0.1 (Auditor scope only)

The following are noted in the schema as deferred:

- v0.1.5 (data-driven, 4 weeks after freeze): refusal_message URL-only
  hardening; any 30-day data-driven small patches.
- v0.2 (after 30-day data): `substituted_by` references to specific
  fixture id; mutation testing; `fixture_provenance` from real failure
  history; `stochastic` mode; `regression_diff` mode.
- v0.3: Layer 7 cross-provider auditor (different LLM provider for
  Auditor than CEO); hardware-backed signing (YubiKey / Secure
  Enclave); `temporal_durable` mode for continuous-running assertions.

These are tracked in the schema's `Deferred` section at the top.

---

End of design v0.1.2 draft.

Next steps: Team CEO implementation prompt for v0.1 — implement the
Auditor + sentinel + setup-gpg wizard + canary_replay + Goodhart-resistant
`.test.sh` per `.aiplus/agent-team/acceptance/v0.1.4/schema.yaml`,
with the v0.1 core team and worktree provisioning from earlier sections.
Once Team CEO produces a binary that passes its own acceptance schema
(self-audit), v0.1 is shippable.
