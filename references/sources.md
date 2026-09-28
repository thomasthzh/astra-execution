# 研究来源与设计取舍

核查日期：2026-09-29。共 12 类项目/方案，另核对 OpenAI 官方能力、Astra skill 指导与 GitLab 自部署适配。读取一手文档，未部署这些项目、未做吞吐或费用基准；下面“采纳”列是本 skill 的综合设计，不是上游性能承诺。链接/主分支会变化，真正引入依赖时应再核对版本。

## 编排与隔离

| 项目 / 一手来源 | 文档证实的机制 | 本 skill 采纳与边界 |
| --- | --- | --- |
| [Agent Orchestrator](https://github.com/Untrivial-ai/agent-orchestrator)，原 ComposioHQ | 项目 orchestrator，单任务 worker，branch/worktree，PR/CI/review 反馈跟随 owner | 主代理保留跨任务决策，反馈回原负责人；不需要安装它才能执行此协议 |
| [Vibe Kanban](https://github.com/BloopAI/vibe-kanban) | 任务看板、workspace、diff/预览与 PR 流程 | 任务和可审查证据关联；[官方关闭公告](https://www.vibekanban.com/blog/shutdown)称公司 2026-04-10 关闭、转社区维护，不能依赖旧远程服务 |
| [Claude Squad](https://github.com/smtg-ai/claude-squad) | tmux 会话、worktree 隔离、TUI/diff 管理 | 小规模只需明确 owner 与隔离；不是完整 issue 调度/发布系统，不照搬实验性自动接受 |
| [Xum](https://github.com/coder/xum)，原 Mux | 多种工作区运行时，分歧与用量可视化；[worktree 文档](https://xum.coder.com/runtime/worktree)说明共享 Git 数据 | 隔离文件不等于隔离所有副作用；[Best of N](https://xum.coder.com/agents/best-of-n)会增加开销，不作为每项默认 |
| [Gas Town](https://github.com/gastownhall/gastown) | Mayor/Polecats/Witness/Refinery 分工，独立合并队列；[scheduler 设计](https://github.com/gastownhall/gastown/blob/main/docs/design/scheduler.md)包含容量、ready 过滤与失败熔断 | 提炼独立集成、背压与有界重试；不照搬所有角色和后台巡检，也不抄存在歧义的配置值 |
| [Symphony](https://github.com/openai/symphony) / [规范](https://github.com/openai/symphony/blob/main/SPEC.md) | issue 驱动、有界并发、持久 workspace、退避、状态核对、WORKFLOW.md 契约 | 先对账再重试，运行结束与交付成功分开；engineering preview，调度器不自动提供业务验收或发布授权 |

## 任务、规格与持久化

| 方案 / 一手来源 | 文档证实的机制 | 本 skill 采纳与边界 |
| --- | --- | --- |
| [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) | lead/teammates、独立上下文、任务依赖与认领、额外 token/协调开销 | 小任务包和 ready 队列；不把 Claude 的实验性团队/hook/恢复能力当 Codex 功能 |
| [Superpowers](https://github.com/obra/superpowers) / [原始工作流](https://raw.githubusercontent.com/obra/superpowers/main/skills/subagent-driven-development/SKILL.md) | fresh context、规格/质量审查、磁盘 ledger、base/head 范围、失败升级 | 用最小上下文、证据交接和完整变更范围；不照搬固定轮数、强制串行或普遍重审 |
| [GitHub Spec Kit](https://github.com/github/spec-kit) / [Agentic SDD](https://github.com/github/spec-kit/blob/main/docs/reference/agentic-sdd.md) / [Lean](https://github.com/github/spec-kit/blob/main/presets/lean/README.md) | 规格→计划→依赖任务→实现，覆盖/一致性检查；issue 导出可选 | 验收可追踪，按规模用规格；不要求每次生产大批文档或外部 issue |
| [OpenHands](https://github.com/OpenHands/OpenHands) / [Persistence](https://docs.openhands.dev/sdk/guides/convo-persistence) / [Condenser](https://docs.openhands.dev/sdk/guides/context-condenser) | 会话 ID、状态与事件持久化、较旧上下文压缩 | 账本保留目标、owner、产物和恢复入口，日志按需读；Markdown 不能替代 SDK 的执行持久性 |
| [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence) / [Checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers) | checkpoint、状态重放、不同 durability；重放可能再次执行 API/LLM 调用 | 对外部副作用先核对已有结果，区别恢复与幂等；只有跨进程可靠执行需要时才考虑框架 |
| [Beads](https://github.com/gastownhall/beads)，原 steveyegge/beads | ready 依赖过滤、原子 claim、当前 Dolt 状态源；embedded 单 writer、server 并发 writer | 单一真相与认领；未采用时不用为本 skill 安装数据库，单调度者账本即可 |

## OpenAI 官方能力校准

- [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)：支持专门子代理与不同模型配置；额外代理自身消耗模型/工具用量。用于确认上下文拆分和能力检测，不承诺多代理天然更省。
- [Git worktrees](https://learn.chatgpt.com/docs/environments/git-worktrees)：独立检出与共享 Git 元数据。具体创建/归档/未提交修改行为以宿主工具说明为准。
- [Build skills](https://learn.chatgpt.com/docs/build-skills)：技能以 SKILL.md 和按需资源组织，可显式/隐式发现；因此本包保持短入口、按阶段读参考文件。
- [Rethinking skills and prompts for GPT-6 Astra](https://learn.chatgpt.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra)，2026-09-11：强调精确触发、按需披露和减少过度规定。本 skill 保留并发/验收不变量，让具体实现依任务选择。

## 由截图到最终协议

自部署 GitLab 的官方来源与能力边界集中在 [git-forges.md](git-forges.md)：保留实例/API 基址、嵌套 namespace、项目 ID 与 iid；区分普通/合成 Pipeline；按实际版本和套餐处理审批、自动合并与 merge train。此项为用户明确要求的适配，不是从截图推导的默认平台。

| 截图经验 | 采用方式 |
| --- | --- |
| 用户只与 Astra 沟通 | 主代理负责用户沟通，worker 给证据摘要；不人为限制用户查看/干预其他任务 |
| Sol 领任务、Luna 实现 | 作为可用时的模型路由起点；复杂实现直接用更强执行者，不强制最低模型 |
| 另一个 Sol 管 merge/PR/CI | 多分支工作由独立集成角色收口，验证最终组合；外部动作仍依实际请求 |
| 每条需求上 GitHub issue | 复用已有任务系统；本地账本也能执行，不额外强制外部写入 |
| 10 个 agent / 10 worktrees | 并发取决于依赖、槽位、机器与集成吞吐，按需增加 |
| 每 issue 只问两次 | 两轮无新增证据时换策略的软提醒，不能代替关键澄清或权限 |
| 每两三小时执行一次 | 优先真实完成事件；跨轮依赖宿主排程，不能靠 skill 文本持续运行 |
| 一晚 8%、每小时 1% | 未证实、不可归因的个人经验，不写成性能保证 |

没有照搬上游完整提示词或运行命令；这些项目是研究来源，不是本包依赖。运行中的授权、宿主能力和用户目标优先于所有模板。
