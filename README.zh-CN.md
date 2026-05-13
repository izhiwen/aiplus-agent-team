# AiPlus Agent Team
[![CI](https://github.com/izhiwen/AiPlus-Agent-Team/actions/workflows/ci.yml/badge.svg)](https://github.com/izhiwen/AiPlus-Agent-Team/actions/workflows/ci.yml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

[English README](README.md)

## 痛点

你让同一个 Agent 先设计 schema，再实现代码，然后做 review，最后写测试。
到第三步时，角色已经**漂移**了：同一段 prompt 历史里混着产品思路、实现细节和
review 挑刺，结果哪一样都没做好。

更糟的是，共享上下文会在不同角色之间互相**污染**。工程师模式的对话记录会流进
PM 模式的上下文里。架构决策被临时的 debug print 埋掉。有价值的长效信息因为
无关的草稿占满窗口而被挤出去。

你试图让 Agent 多戴几顶帽子来弥补。但一个 Agent 同时扮演 PM、架构师、工程师、
Reviewer 和 QA，每一顶帽子都戴得很**浅**。真正的工程团队之所以分工，是因为工作
本身就是如此结构化。

### 不是这些其他痛点

AiPlus Agent Team 专门解决**角色分离与执行**问题。其他 AiPlus 插件解决相邻但
不同的问题：

| 插件 | 解决的痛点 | 为什么它不是 Agent Team |
|---|---|---|
| [AiPlus-Agent-Memory](https://github.com/izhiwen/AiPlus-Agent-Memory) | **失忆** — 跨 session 忘记上下文 | 给单个 Agent 记忆；不拆分角色 |
| [AiPlus-Auto-Team-Consultant](https://github.com/izhiwen/AiPlus-Auto-Team-Consultant) | **忽略** — plan 时漏掉上手、安全、执行陷阱 | 在规划前**建议**；不执行、不持久化角色 |
| [AiPlus-Compact-Reminder](https://github.com/izhiwen/AiPlus-Compact-Reminder) | **断片** — compact 后上下文恢复 | 恢复单个 Agent 的上下文；不做角色分离 |
| [AiPlus-Agent-Velocity](https://github.com/izhiwen/AiPlus-Agent-Velocity) | **瞎报** — 估时锚定在人类工程师小时数 | 校准单个 Agent 的估时；不做团队结构 |

Agent Team 是安装**常驻执行团队**、拥有隔离工作区和内存的那一层。它位于其他
四个插件之上，将它们用作共享基础设施。

## 解决方案

**用常驻团队取代单 Agent 的角色漂移。**

AiPlus Agent Team 在你的项目里安装一支常驻虚拟团队，包含 8 个核心角色：
Advisor、CEO、Architect、PM、两名 Engineer、Reviewer 和 QA。每个角色拥有
独立的人设、工作区和内存命名空间。Owner 只需与 Advisor 和 CEO 对话；
CEO 负责调度其余成员。

核心能力：

- **角色隔离** — 每个 Agent 只加载自己的人设和个人记忆。工程师看不到 PM 的
  推理过程，反之亦然。
- **Git worktree 工作区** — 需要写代码的角色拥有独立的工作目录，两名工程师
  可以并行工作而不会互相踩脚。冲突通过 git 暴露，而非静默覆盖。
- **三层记忆** — 个人（按 Agent）、团队（CEO 共享）和项目（现有的
  `.aiplus/memory/`）。项目记忆在冲突时优先级最高，因此临时的团队决策不会
  覆盖长期的项目共识。
- **专家目录** — 十一位专家（AI Integration、Security Reviewer、Tech Writer、
  DevOps、UI Designer、Researcher 等）平时待命，当 CEO 识别到匹配触发条件时
  才会召唤。
- **自适应路由** — CEO 为每个任务打分（LIGHT、MEDIUM、HEAVY），只派遣需要的
  角色。一个简单问题直接回答；一个高风险改动调动全团队。

没有守护进程。没有云端同步。没有上传。每个 Agent 是"状态级常驻"——它的文件
保存在磁盘上，但进程是临时的，仅在 CEO 派发任务时启动。

## 安装

将模块添加到你的项目：

```bash
cd MyProject
aiplus add agent-team
aiplus install codex          # 或：claude-code、opencode、all
```

## 快速开始

```bash
aiplus agent status              # 显示团队名册、活跃专家和热板凳状态
aiplus agent route engineer-a    # 分配任务给 engineer-a
aiplus agent integrate engineer-a # 将 engineer-a 的分支合并回 main
aiplus agent audit run           # 运行验收审计
aiplus 团队                      # 中文别名：显示团队状态
aiplus 审计 跑                   # 中文别名：运行验收审计
```

通过 CEO 派发任务：

```text
aiplus agent route "重构 billing 模块"
```

CEO 会为任务打分、挑选合适的团队成员并汇报结果。

其他常用命令：

```bash
aiplus agent doctor            # 验证配置、工作区和内存布局
aiplus agent list              # 列出所有角色（核心 + 专家）
aiplus agent talk architect    # 与某个角色直接对话
aiplus agent invite security-reviewer   # 召唤专家加入活跃团队
aiplus agent dismiss security-reviewer  # 将专家从活跃团队移除
aiplus agent transcript        # 显示近期活动记录用于审计
aiplus agent prune-worktrees   # 清理过时工作区
```

## 架构概览

```
                    AiPlus-Agent-Team               ← 编排层
                           ↓ 使用
               AiPlus-Auto-Team-Consultant          ← 决策支持层
                           ↓ 使用
    AiPlus-Agent-Memory  AiPlus-Compact-Reminder  AiPlus-Agent-Velocity
               ←——————— 共享基础设施层 ———————→
```

Agent Team 是编排层。它位于四个现有 AiPlus 插件之上，将它们用作共享基础设施：

- **AiPlus-Agent-Memory** — 每个 Agent 拥有独立的内存命名空间，位于
  `.aiplus/agent-memory/<role>/`
- **AiPlus-Compact-Reminder** — 每个长期运行的 Agent 拥有自己的 compact
  周期；CEO 按 Agent 跟踪 compact 状态
- **AiPlus-Agent-Velocity** — 每个 Agent 拥有自己的 velocity 记录
- **AiPlus-Auto-Team-Consultant** — CEO 在 MEDIUM 和 HEAVY 任务前触发
  consultant；consultant 的发现流入执行团队的 brief

### 五大核心设计决策

1. **8 个核心角色的常驻团队** — 插件添加到项目时自动安装。
2. **专家目录** — 11 位专家角色按需召唤，仅在触发条件匹配时出现。
3. **状态级常驻 + 热板凳** — Agent 身份保存在磁盘上；进程是临时的，
   仅在 CEO 路由任务时启动。
4. **Git worktree 工作区** — 每个需要写代码的角色拥有独立工作目录，
   两名工程师可以并行工作而不会静默覆盖。
5. **三层内存** — 个人（按 Agent）、团队（CEO 共享）和项目（现有的
   `.aiplus/memory/`）。项目内存在冲突时优先级最高。

详细设计原理、路由协议、内存模型、工作区策略和验收标准见
[`DESIGN.md`](DESIGN.md)。

## 仓库结构

- `core/templates/` — 全部 8 个核心角色的 TOML 配置和 markdown 人设，
  以及团队级配置 `agent-team.toml`
- `core/templates/personas/` — 角色人设提示词（advisor、ceo、architect、
  pm、engineer、reviewer、qa）
- `core/docs/` — 设计原理、路由协议、内存模型、工作区策略、安全边界
- `adapters/codex/` — Codex 插件和技能资源
- `adapters/claude-code/` — Claude Code 项目本地命令和 Agent
- `adapters/opencode/` — OpenCode 项目本地配置、命令和提示词
- `examples/` — 三个运行时的合成示例

## 贡献指南

欢迎保持在插件范围内的贡献（角色分离与执行，不是咨询顾问或估时校准）。

1. **先开 issue** — 对于超过 typo 修复的改动，`AiPlus-Agent-Team` 的范围
    tightly bounded，我们希望避免与其他 AiPlus 插件冲突的善意 PR。
2. **遵循现有的 TOML + markdown 人设模式** — 每个 Agent 的配置在
   `.aiplus/agents/<role>.toml`，人设提示词在
   `.aiplus/agents/personas/<role>.md`。
3. **保持 adapter 一致性** — 如果修改 CLI 界面，同时更新三个 adapter
   （`adapters/codex/`、`adapters/claude-code/`、`adapters/opencode/`）。
4. **配置变更后运行 `aiplus agent doctor`** — 验证工作区、内存布局和
   TOML 模式。
5. **验收标准是 binding 的** — 见
   `.aiplus/agent-team/acceptance/v0.1.4/schema.yaml`。任何行为变更必须
   更新 schema 及其配套的 `.test.sh`。

## 安全边界

AiPlus Agent Team 不会：

- 向任何服务上传 Agent 状态、人设、记忆或对话记录
- 以后台守护进程或常驻进程运行
- 在任何 Agent 的人设、记忆或工作区中存储密钥
- 修改全局 Agent 配置（~/.codex、~/.claude 等）
- 修改其他项目的 `.aiplus/`
- 自动批准 Owner 把关的操作（push、tag、release、deploy）
- 引入主机运行时之外的新网络请求

## 更多

- 主平台：[AiPlus](https://github.com/izhiwen/AiPlus)
- 官方源码：https://github.com/izhiwen/AiPlus-Agent-Team

## 许可证

[Apache-2.0](LICENSE)
