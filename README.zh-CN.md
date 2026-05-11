# AiPlus Agent Team
[English README](README.md)

## 痛点

你让同一个 Agent 先设计 schema，再实现代码，然后做 review，最后写测试。
到第三步时，角色已经漂移了：同一段 prompt 历史里混着产品思路、实现细节和
review 挑刺，结果哪一样都没做好。

更糟的是，共享上下文会在不同角色之间互相污染。工程师模式的对话记录会流进
PM 模式的上下文里。架构决策被临时的 debug print 埋掉。有价值的长效信息因为
无关的草稿占满窗口而被挤出去。

你试图让 Agent 多戴几顶帽子来弥补。但一个 Agent 同时扮演 PM、架构师、工程师、
Reviewer 和 QA，每一顶帽子都戴得很浅。真正的工程团队之所以分工，是因为工作
本身就是如此结构化。

## 解决方案

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

## 快速开始

如果你已经安装了 AiPlus：

```bash
cd MyProject
aiplus install codex          # 或：claude-code、opencode、all
aiplus agent status
```

`aiplus agent status` 会显示团队名册、活跃专家和热板凳状态。

然后通过 CEO 派发任务：

```text
aiplus agent route "重构 billing 模块"
```

CEO 会为任务打分、挑选合适的团队成员并汇报结果。

常用命令：

```bash
aiplus agent doctor            # 验证配置、工作区和内存布局
aiplus agent list              # 列出所有角色（核心 + 专家）
aiplus agent talk architect    # 与某个角色直接对话
aiplus agent invite security-reviewer   # 召唤专家加入活跃团队
aiplus agent integrate engineer-a       # 将 engineer-a 的分支合并到 main
```

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

- 主平台：[aiplus](https://github.com/izhiwen/aiplus)
- 官方源码：https://github.com/izhiwen/aiplus-agent-team

## 许可证

[Apache-2.0](LICENSE)
