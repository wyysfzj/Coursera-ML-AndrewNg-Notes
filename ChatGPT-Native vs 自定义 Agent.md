# Native vs 自定义 Agent

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 3/3/2026 0:00:09  
**Updated:** 3/3/2026 9:05:31  
**Exported:** 3/3/2026 9:05:41  
**Link:** [https://chatgpt.com/c/69a5b408-b74c-8320-a192-026d0c7ee744](https://chatgpt.com/c/69a5b408-b74c-8320-a192-026d0c7ee744)  

## Prompt:
Codex 是有原生的agent。固定 4 个（永远可用）
• default（通用）
• worker（执行实现）
• explorer（代码探索）
• monitor（监控长任务。 最新版的也support 自定义的 agent，现在follow 社区的best practice 定义了6 agent team，如下 #!/bin/bash
set -e

# 创建目录（全局配置，推荐）
mkdir -p ~/.codex/agents

echo "🚀 开始创建 6 个 Agent 角色 toml 文件..."

# ==================== 1. team_lead.toml ====================
cat > ~/.codex/agents/team_lead.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"

developer_instructions = """
你是团队 Lead（项目总负责人）。你必须严格遵守以下流程：

1. 收到需求后，**先输出完整计划**：
   - 任务分解（步骤 + 依赖关系）
   - 时间/复杂度估计
   - 明确分配给每个角色的具体任务

2. 使用 gated handoff：
   - 只有当前角色完成并产出 artifact（文件/代码/Mermaid 图等）后，才允许下一个角色启动
   - 必须检查前置文件是否存在

3. 收集所有子代理结果后：
   - 做最终合成
   - 风险评估
   - 给出 PR 建议和合并 checklist

4. 所有输出必须清晰标注角色和步骤，例如：
   【Team Lead】计划已生成 → 分配给 Architect...
"""
EOF

# ==================== 2. architect.toml ====================
cat > ~/.codex/agents/architect.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"

developer_instructions = """
你是系统架构师。必须产出以下完整内容：
- Mermaid 架构图（C4 / 序列图 / 组件图）
- 模块划分与每个模块职责
- API 契约（OpenAPI 格式）与数据模型
- 技术选型理由 + 可扩展性/可维护性/性能考虑
- 数据库 schema（如果需要）
"""
EOF

# ==================== 3. be_developer.toml ====================
cat > ~/.codex/agents/be_developer.toml << 'EOF'
model = "gpt-5.3-codex"

developer_instructions = """
你是后端开发者。负责实现服务端逻辑、API、数据库操作、业务逻辑、性能优化。
- 必须使用当前项目技术栈
- 严格 TDD：先写测试再写实现
- 代码必须遵循 SOLID、干净架构、日志完整、安全校验
- 每次实现后自动运行测试
"""
EOF

# ==================== 4. fe_developer.toml ====================
cat > ~/.codex/agents/fe_developer.toml << 'EOF'
model = "gpt-5.3-codex"

developer_instructions = """
你是前端开发者。负责 UI/UX 实现、组件开发、状态管理、响应式适配、与后端联调。
- 严格遵循现有 UI 组件库和设计系统（shadcn / AntD / Element 等）
- 使用 TypeScript + 现代框架（React / Vue / Svelte）
- 优先原子组件、可复用 Hook、响应式设计
- 每次修改后运行 lint + test
"""
EOF

# ==================== 5. tester.toml ====================
cat > ~/.codex/agents/tester.toml << 'EOF'
model = "gpt-5.3-codex-spark"
sandbox_mode = "read-only"

developer_instructions = """
你是测试工程师。必须：
- 编写完整单元测试、集成测试、E2E 测试
- 目标覆盖率 > 80%
- 包含正常、边界、异常、性能测试用例
- 运行测试并报告详细结果（失败用例 + 截图/日志）
- 发现 bug 立即复现并反馈给对应开发者
"""
EOF

# ==================== 6. reviewer.toml ====================
cat > ~/.codex/agents/reviewer.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"

developer_instructions = """
你是资深代码评审员。必须检查：
- 安全漏洞（OWASP Top 10）
- 性能问题与瓶颈
- 代码异味、重构建议
- 命名/风格/注释一致性
- 测试覆盖率与质量
给出具体修改建议 + diff 示例代码
最终输出「✅ 通过」或「❌ 需要修改」结论
"""
EOF

echo "✅ 全部 6 个角色 toml 文件创建完成！路径：~/.codex/agents/"
ls -1 ~/.codex/agents/*.toml

echo ""
echo "📋 下一步："
echo "1. 在 ~/.codex/config.toml 中加入以下 [agents] 配置（如果还没有）："
echo "   （我可以再给你一键添加 config.toml 的命令）"
echo "2. 启动 codex → 输入你的需求，例如："
echo '   "使用我们的 6-agent team 开发用户登录功能"'
echo "3. 项目级配置？把 ~/.codex/agents/ 整个复制到项目下的 .codex/agents/ 即可覆盖"  请研究Codex 原生 Agent（Native） vs 我们之前按业界 Best Practice 配置的自定义 6-Agent Team 完整对比。 并探究 6个自定义agent 的进一步完善的可能性

## Response:
Thought for 19m 28s

## Response:
下面按“机制/能力 → 组织/流程 → 落地可运行性 → 进一步优化空间”的顺序，把 **Codex 原生 4-Agent（Native roles）** 与你们的 **自定义 6-Agent Team** 做一次“可执行级别”的完整对比，并给出针对 6 个自定义 agent 的具体改造建议（含配置片段、权限/并发/模型选择的坑位修复）。

* * *

1) 先澄清：Codex 的“原生 4 个 Agent”到底是什么
---------------------------------

你提到的 **4 个固定角色**：

*   `default`（fallback）
*   `worker`（实现/修复）
*   `explorer`（读多写少的代码探索）
*   `monitor`（长任务/轮询等待）

这在官方定义里属于 **Codex CLI 的 Multi‑agents（多代理协作）机制**的“内置角色集合”。它的要点是：Codex 可以在一个任务里 **并行 spawn 多个子代理**，最后 **汇总**成一个回答；并且你也可以在 `[agents]` 里定义自己的角色与每个角色的独立配置。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

但注意两个“落地前提”：

1.  **Multi‑agents 目前是 experimental**，需要显式开启（CLI `/experimental` 或 `[features] multi_agent = true`）。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)
2.  **子代理的权限模型**跟主线程不一样：
    *   子代理 **继承当前 sandbox**，但以 **non‑interactive approvals** 运行；任何需要“新批准”的动作会直接失败，并把错误回传给父流程。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)
    *   可以对某个角色单独覆盖 sandbox（例如把 reviewer 强制成 `read-only`）。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

* * *

2) Native 4-Agent vs 你们 6-Agent：本质差异（一句话）
-----------------------------------------

*   **Native 4**：更像“功能型分工”（探索/实现/监控/兜底），目标是 **降低主线程噪音与上下文衰减**、提升并行效率。[OpenAI Developers+1](https://developers.openai.com/codex/concepts/multi-agents/)
*   **自定义 6**：更像“组织型分工”（Lead/架构/前后端/测试/评审），目标是把 **SDLC 流程与质量门禁**固化进 agent 行为。

这两套并不冲突，关键在于：**你们的 6-agent 要“适配 Codex 的多线程 + 权限 + 上下文隔离”运行时约束**，否则会出现“看起来很完整，但实际上跑不起来/跑不稳”的问题（下面会点名）。

* * *

3) 维度级完整对比
----------

### 3.1 角色颗粒度与“自动选角”能力

**Native 4 roles**

*   由 Codex 内置，且是 Codex 自动 spawn 时最稳定的“词表”。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)
*   你也可以用同名覆盖（例如你定义 `[agents.explorer]` 会覆盖内置 explorer）。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

**你们 6 roles**

*   优点：覆盖 SDLC，天然适合 gated handoff（规划→架构→实现→测试→评审）。
*   风险：如果你只“创建 toml 文件”，但 **没在 `[agents]` 里声明 role + description + config\_file**，Codex **不会稳定地自动选用**（甚至可能根本 spawn 不了这个角色）。角色声明与 config\_file 的校验规则也很严格：路径必须存在，否则 spawn 可能失败。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

**建议**

*   如果你希望 Codex **自动选择**“你们的 6-agent”，一定要把每个 role 在 `[agents.<name>]` 配上清晰 `description`（越窄越好，官方也强调“角色定义要 narrow & opinionated”）。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

* * *

### 3.2 上下文治理：为什么 Native 4 的范式很重要

官方把多代理的核心动机讲得很直白：主线程塞满探索笔记、测试日志、堆栈、命令输出，会造成：

*   **Context pollution**：关键决策被噪音淹没
*   **Context rot**：随着对话增长，整体可靠性下降[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)

多代理的正确姿势是把“噪音型工作”外包给子代理，让它们 **返回摘要**，主线程只保留需求、决策与最终产出。[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)

你们 6-agent 的思路是对的，但需要把“谁产生噪音、谁做摘要、谁写最终文件/代码”写进契约，否则 6 个 agent 并行输出会把噪音放大 6 倍。

* * *

### 3.3 并行与写冲突：CLI vs App 的现实差别

官方建议：

*   并行优先做 **read-mostly**（探索、测试/triage、总结），
*   **并行写代码要谨慎**：多个 agent 同时编辑会带来冲突与协调成本。[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)

而在 **Codex app** 里，OpenAI 明确说 app 有 **worktrees 内建支持**，每个 agent 在隔离的代码副本上工作，从而减少冲突。[OpenAI](https://openai.com/index/introducing-the-codex-app/)

**结论**

*   如果你们主要跑在 **CLI 的 multi-agent**：建议把并行限制在“探索/评审/测试分析”，写代码尽量串行或单写手。
*   如果你们主要跑在 **Codex app**：可以更大胆地并行实现（因为 worktrees 提供天然隔离）。[OpenAI](https://openai.com/index/introducing-the-codex-app/)

* * *

### 3.4 权限与 approvals：你们当前 6-agent 里最大的“运行时硬伤”

多代理里子代理是 **non‑interactive approvals**：需要新批准的动作会失败。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)  
再结合 sandbox 语义（read‑only / workspace‑write / …），你们当前配置会出现一个非常典型的矛盾：

> 你们的 `tester` 设成 `sandbox_mode = "read-only"`，但又要求“写测试 + 运行测试 + 报告日志”。

在 Codex 的安全模型里，“read‑only”通常意味着：只能读文件；要编辑文件、运行命令、访问网络都需要批准/或被限制。[OpenAI Developers](https://developers.openai.com/codex/security/)  
而子代理又不能弹批准窗口。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

**所以：tester 这个角色在多代理场景下，按现配置很可能无法完成其职责。**

* * *

### 3.5 模型路由与 reasoning effort：你们现在“全高配”的成本/收益不平衡

官方给了非常实用的建议：

*   `gpt-5.3-codex`：适合强推理/评审/安全分析/多步实现/需要提 patch 的 agent
*   `gpt-5.3-codex-spark`：适合更快的探索、扫描、总结型并行 worker[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)

你们现在：

*   team\_lead / architect / reviewer 都用 `gpt-5.3-codex` 且 lead/architect/reviewer 还开了 `high` reasoning（这部分合理）
*   但 BE/FE dev 也默认用 full model（可以）
*   tester 用 spark 但 read-only（模型快，但权限把它“掐死”）

更好的做法是：

*   把“读多写少”的角色（architect / reviewer / explorer 类）更多用 spark 或 read-only 限制
*   把“写代码/改文件”的角色（BE/FE/test-implement）用 full model，reasoning effort 多数用 `medium`，只有在复杂问题上再提到 `high`（节省成本、也减少过度思考导致的偏航）

* * *

### 3.6 配置可移植性：AGENTS.md / Skills / Rules 才是“团队级最佳实践”的承载面

如果你把所有规则都塞进每个 agent 的 `developer_instructions`，会带来两个问题：

*   维护成本高（每次改规范要改 6 份）
*   不同入口（CLI / IDE / App）可能存在注入差异

官方推荐的“稳定承载面”是 **AGENTS.md**：Codex 在做任何工作前会读取，并按全局→项目→子目录覆盖的方式拼接指令链。[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/)

而 **Codex app** 本身也强调了 **skills**：把指令、资源、脚本打包，让 agent 更稳定复用团队工作流。[OpenAI](https://openai.com/index/introducing-the-codex-app/)

另外，很多“让 tester 能跑起来”的问题，需要靠 **Rules（exec policy rules）**：用 `prefix_rule` allowlist 一些安全命令，让 agent 在不升级 sandbox 的情况下也能顺畅执行。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

* * *

4) 你们现有 6-agent 的“可运行性审计”（最关键的几处）
---------------------------------

### A. 你们现在只是“创建了 toml”，但还缺一个关键：`[agents]` 声明层

Multi‑agents 明确：角色要在 `[agents]` 里声明，并用 `description` 指导 Codex 何时使用，还可用 `config_file` 载入该角色专属配置；且 `config_file` 必须存在、会在加载时校验。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

### B. tester 的 `read-only` 与职责冲突（这是最大 blocker）

如上所述，在子代理 non-interactive approvals 的约束下，你要求 tester 写/跑测试会直接受阻。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

### C. 并行写冲突风险未被显式控制

官方提醒并行写要谨慎。[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)  
你们目前 BE/FE 都是写手角色，若并行 spawn，CLI 场景会很容易冲突；App 场景可用 worktrees 缓解。[OpenAI](https://openai.com/index/introducing-the-codex-app/)

### D. team\_lead 的“gated handoff”想法很好，但最好直接对齐 Codex 的 Plan Mode 规范

Plan Mode 的模板明确：

*   3 阶段（ground→intent→implementation），
*   允许非 mutating 的探索/测试/构建以完善计划，
*   **禁止写文件/打补丁/执行会改变 repo 的动作**，
*   最终输出必须是 decision-complete 的 `plan` block。[GitHub+1](https://raw.githubusercontent.com/openai/codex/main/codex-rs/core/templates/collaboration_mode/plan.md)

你们的 team\_lead 可以直接复用这套范式，把“先计划后执行”的门禁变成 Codex 原生理解的协作模式，而不是靠口头约束。

* * *

5) 进一步完善：把 6-agent 变成“Codex 友好”的版本（具体改造建议）
------------------------------------------

下面给出一个我认为最实用、最贴近 Codex runtime 的改造方向：**“一套团队契约 + 六个窄角色 + 读写隔离 + rules/skills 配套”**。

### 5.1 把“团队级规范”从 6 份 instructions 里抽出来：用 AGENTS.md 承载

建议至少做两层：

1.  `~/.codex/AGENTS.md`：个人偏好/通用规范（例如不要加依赖需确认）
2.  repo 根目录 `AGENTS.md`：项目命令、测试入口、目录约定、代码风格、PR 流程

Codex 的读取与覆盖顺序、以及 `AGENTS.override.md` 的优先级与 size cap，都有明确规范。[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/)

这样你们每个 agent 的 `developer_instructions` 就可以更“窄”，避免漂移。

* * *

### 5.2 在 config.toml 注册 6 个角色，并给出“让 Codex 会选”的 description

示例（放 `~/.codex/config.toml` 或 `.codex/config.toml`；后者便于团队共享）：[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

```
[features]
multi_agent = true

[agents]
max_threads = 6
max_depth = 1

[agents.team_lead]
description = "项目总负责人：先在计划/架构层收敛需求与方案，再按门禁调度实现、测试、评审。默认不直接改代码，除非进入执行阶段。"
config_file = "agents/team_lead.toml"

[agents.architect]
description = "系统架构与接口契约：输出模块划分、Mermaid 图、OpenAPI、数据模型与风险点。优先读代码与现有文档。"
config_file = "agents/architect.toml"

[agents.be_developer]
description = "后端实现：按现有栈做最小必要改动，TDD 实现 API/业务/DB，运行测试。"
config_file = "agents/be_developer.toml"

[agents.fe_developer]
description = "前端实现：按现有设计系统与组件库实现 UI/交互，TypeScript，跑 lint/test。"
config_file = "agents/fe_developer.toml"

[agents.tester]
description = "测试工程：补齐单测/集成/E2E，跑测试并输出失败用例、复现步骤与日志证据。"
config_file = "agents/tester.toml"

[agents.reviewer]
description = "资深代码评审：只读审查，聚焦 correctness/security/perf/test gaps，给可执行的 diff 建议并出结论。"
config_file = "agents/reviewer.toml"
```

补充要点：

*   `description` 是 Codex 决策“用谁”的关键提示。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)
*   `max_depth` 建议先保持 1（避免子代理再 spawn 子代理导致“组织爆炸”）；官方默认也就是 1。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

> 如果你们希望“尽量复用 Codex 原生自动选角逻辑”，可以考虑**把 `team_lead` 改名为 `default` 并覆盖内置 default**（同名覆盖会生效）。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

* * *

### 5.3 修正 tester：让它在多代理里真的能“写+跑+报”

你们现在的 tester = read-only 是不匹配职责的。推荐两种可选路线：

#### 路线 1（更实用）：tester 改为 `workspace-write`，但限制“只动测试域”

*   `sandbox_mode = "workspace-write"`
*   `developer_instructions` 明确：
    *   只允许修改 `test/ __tests__/ e2e/` 等目录或 `*.spec.*` / `*.test.*` 文件
    *   不改业务代码（除非为了修复测试可观测性，并且先提请 team\_lead）
    *   跑测试只跑规定命令

> 子代理遇到需要新批准的动作会失败，所以 tester 的工作流必须“在当前 sandbox + rules 允许范围内闭环”。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

#### 路线 2（更保守）：tester 仍 read-only，只输出“测试补丁方案”，由 worker 统一落盘

*   好处：避免并行写冲突
*   坏处：失去 tester 自己跑测试、出证据的闭环（你们原目标是闭环）

你问“进一步完善的可能性”，我更建议路线 1，并配合 rules（下一节）。

* * *

### 5.4 用 Rules 给 tester/worker 打通“非交互执行”的最后一公里

Rules 的官方定位就是：**控制哪些命令可以在 sandbox 外/或提升权限时自动执行**，而且能用 `prefix_rule` 精确到命令前缀。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

典型做法：allowlist 你们常用的“测试/构建/静态检查”命令（而不是允许 `bash -lc "...一串..."` 这种不可控的包装命令——官方也解释了 shell wrapper 的风险处理）。[OpenAI Developers](https://developers.openai.com/codex/rules/)

示例 `~/.codex/rules/default.rules`（按你们栈改）：[OpenAI Developers](https://developers.openai.com/codex/rules/)

```
# Allow common test commands (tight prefixes, no shell scripts)
prefix_rule(
  pattern = ["pnpm", "test"],
  decision = "allow",
  justification = "Allow running unit tests"
)

prefix_rule(
  pattern = ["npm", "test"],
  decision = "allow",
  justification = "Allow running unit tests"
)

prefix_rule(
  pattern = ["pytest"],
  decision = "allow",
  justification = "Allow running Python tests"
)

prefix_rule(
  pattern = ["go", "test"],
  decision = "allow",
  justification = "Allow running Go tests"
)
```

你还可以用 `codex execpolicy check` 去验证规则是否覆盖到具体命令（官方提供了这个检查方式）。[OpenAI Developers](https://developers.openai.com/codex/rules/)

* * *

### 5.5 让 team\_lead 的“门禁交接”对齐 Plan Mode（会更稳）

你们 team\_lead 的 gated handoff 很接近官方 Plan Mode 的精神，但 Plan Mode 更“可机器执行”：

*   明确禁止 mutating 行为（写文件/打补丁/运行会改 repo 的 codegen 等）
*   允许做“非 mutating 的探索/测试/构建”来完善计划
*   三阶段推进，最终输出必须 decision-complete 的 `plan` block[GitHub](https://raw.githubusercontent.com/openai/codex/main/codex-rs/core/templates/collaboration_mode/plan.md)

而且 collaboration\_modes 在配置里是 stable 默认开启的（plan mode 属于其中）。[OpenAI Developers+1](https://developers.openai.com/codex/config-basic/)

**建议你对 team\_lead.toml 的改造方向：**

*   明确：默认处于 plan 阶段（不直接写代码），除非用户/父线程显式切到执行
*   将“artifact 门禁”具体化：
    *   Phase 1 产出 `PLAN.md`（或输出 plan block）
    *   Phase 2 产出 `ARCH.md` / `openapi.yaml`（可由 architect 产出，lead 落盘）
    *   Phase 3 才允许 worker 修改代码

* * *

### 5.6 模型与 reasoning 的再平衡：把钱花在刀刃上

结合官方建议：[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)

*   team\_lead：`gpt-5.3-codex` + `high`（合理，负责决策闭环与综合）
*   architect：很多任务是“读+抽象+输出图/契约”，可考虑 `spark` + `medium`（速度更划算），遇到复杂再升级
*   be\_developer / fe\_developer：通常 `gpt-5.3-codex` + `medium` 就够；只有复杂重构/疑难 bug 才 `high`
*   tester：若走路线 1（可写可跑），模型可以 `spark`（快）或 `codex`（更稳），建议把 \*\*“失败 triage”\*\*交给 spark，把 \*\*“最终测试补丁”\*\*交给 codex（但你们现在只有一个 tester role，先选一个更稳的默认也行）
*   reviewer：`gpt-5.3-codex` + `high`（合理）

* * *

### 5.7 “进一步完善”的高阶方向：把 6-agent 变成“角色 + 协议 + 技能”的系统

你们现在主要在做“角色定义”。下一步提升会来自两件事：

1.  **输出协议（contracts）**：让每个 agent 输出可被 team\_lead 机械合并的结构化结果
    *   explorer/architect：必须带文件路径/符号名/证据点
    *   dev：必须带“改动清单 + 风险 + 回滚方式 + 测试结果”
    *   tester：必须带“复现步骤 + failing test + 日志证据 + 覆盖范围”
    *   reviewer：必须带“严重度分级 + 建议 diff + 通过/不通过结论”
2.  **Skills 化**：把你们最常重复的流程（TDD loop、运行 CI、生成 OpenAPI、生成 Mermaid、PR checklist）打包成 skills，让 agent 自动复用。Codex app 明确强调 skills 可以把指令/资源/脚本打包并在 app 内管理，也可跨 app/CLI/IDE 使用。[OpenAI](https://openai.com/index/introducing-the-codex-app/)

* * *

6) 一个你们可以直接采用的“升级策略”（不增加角色数，但显著更稳）
----------------------------------

1.  **先不扩编角色**，只做“读写隔离”：
    *   architect + reviewer → `read-only`
    *   tester → `workspace-write`（只写测试域）
    *   be/fe developer → `workspace-write`
2.  **并行只跑 read-mostly 子任务**（explore/review/triage），写代码尽量串行（特别是 CLI）。[OpenAI Developers](https://developers.openai.com/codex/concepts/multi-agents/)
3.  **落地 Rules allowlist**（测试/构建/静态检查命令）让子代理闭环执行。[OpenAI Developers](https://developers.openai.com/codex/rules/)
4.  **把通用规范迁移到 AGENTS.md**，让每个 agent 的 instructions 变“窄”。[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/)
5.  **team\_lead 对齐 Plan Mode**：先 plan block 再执行，保证门禁可控。[GitHub](https://raw.githubusercontent.com/openai/codex/main/codex-rs/core/templates/collaboration_mode/plan.md)

* * *

7) 你可能会踩的一个“现实坑”：Codex App 的指令注入差异
----------------------------------

OpenAI 说 app 会继承 CLI/IDE 的 session history 和 configuration。[OpenAI](https://openai.com/index/introducing-the-codex-app/)  
但与此同时，有用户在 2026-02-07 提了一个 issue：**Codex App 发起的线程没有注入 `developer_instructions`（config.toml）**。[GitHub](https://github.com/openai/codex/issues/11004)

这意味着：如果你们非常依赖 `developer_instructions` 来实现团队规范，\*\*在 App 侧可能出现“规则失效”\*\*的体验差异。  
务实建议是：**把关键规范放到 AGENTS.md**（因为 issue 描述里 App 仍会注入 AGENTS.md）。[GitHub+1](https://github.com/openai/codex/issues/11004)

* * *

如果你愿意把“进一步完善”推进到更工程化的一步，我建议你们下一版直接做两件事（仍不增加角色）：

*   给 6 个 agent 定义一套**统一输出 schema**（例如每个 agent 必须输出：`Findings / Evidence / Suggested Next Step / Risks / Files touched`），让 team\_lead 汇总变成“可审计”的 PR checklist。
*   把 tester 改成可执行闭环（workspace-write + rules），否则这个角色在 multi-agent 里基本只是“口头 QA”，不是“证据驱动 QA”。

你如果把你们现有 `~/.codex/config.toml`（去掉敏感信息）和项目栈（Node/Go/Python/Java + 测试命令）贴一点出来，我可以基于你们的真实命令集把 `default.rules` 做成“最小可用且安全”的 allowlist（避免过宽导致安全面扩大）。

## Prompt:
config.toml

File

分别给出两套rules。JAVA/SprintBoot. 和 python. 我要一步到位 “进一步完善”的高阶方向：把 6-agent 变成“角色 + 协议 + 技能”的系统，支持 JAVA/SprintBoot. 和 python. 采用社区或业界的best practice of （TDD loop、运行 CI、生成 OpenAPI、生成 Mermaid、PR checklist）.请给出step by step 的详细落地方案，过程，可直接使用的配置

## Response:
Thought for 38m 26s

## Response:
先说明一下：你刚上传的 `config.toml` 我这边当前无法读取（系统提示文件已过期）。不影响我给你“一步到位”的**完整可用模板**；如果你希望我把下面的方案“精准贴合你现有 config.toml 并做增量 patch”，你再把文件重新上传即可。

下面我给你：

1.  **两套可直接用的 Rules（execpolicy）**：
    *   `JAVA / Spring Boot`
    *   `Python`
2.  一套“角色 + 协议 + 技能”落地体系（可直接拷到仓库）：
    *   `.codex/`：多代理角色与配置（6-agent）
    *   `AGENTS.md`：团队协议（protocol）
    *   `.agents/skills/`：Skills（TDD loop / Run CI / Generate OpenAPI / Generate Mermaid / PR checklist）

并且按 **step-by-step** 给你落地流程、目录结构、文件内容、校验方法。

> 关键事实依据：Multi‑agents 需要 `[features] multi_agent = true`，角色在 `[agents]` 定义并用 `config_file` 指向存在的 TOML；子代理是 non‑interactive approvals（不能弹出批准窗口）；skills 扫描 `.agents/skills`；rules 用 `prefix_rule()` 并可用 `codex execpolicy check` 校验。[OpenAI Developers+3OpenAI Developers+3OpenAI Developers+3](https://developers.openai.com/codex/multi-agent/)

* * *

Step 0：你要的目标状态（落地后怎么用）
======================

你最终会得到一个仓库内可共享的“AI 团队系统”：

```
repo/
  AGENTS.md                         # 协议（Protocol），全员遵守
  .codex/
    config.toml                     # 6-agent team 注册 + 默认安全策略
    agents/
      team_lead.toml
      architect.toml
      be_developer.toml
      fe_developer.toml
      tester.toml
      reviewer.toml
    rules/
      java_springboot.rules
      python.rules
  .agents/
    skills/
      tdd-loop/
        SKILL.md
        agents/openai.yaml          # 可选：限制隐式触发
      run-ci/
        SKILL.md
        agents/openai.yaml
      generate-openapi/
        SKILL.md
        agents/openai.yaml
      generate-mermaid/
        SKILL.md
      pr-checklist/
        SKILL.md
  docs/
    architecture/                   # Mermaid 架构图建议放这里
    openapi/                        # OpenAPI 产物建议放这里
```

*   Codex **项目级配置**放 `.codex/config.toml`（团队共享，且只在“trusted project”生效）。[OpenAI Developers](https://developers.openai.com/codex/config-basic/)
*   Skills 放 `.agents/skills`（Codex 会从当前目录向上扫描到 repo root）。[OpenAI Developers](https://developers.openai.com/codex/skills/)
*   Rules 放 `.codex/rules/*.rules`（Codex 启动会扫描每个 Team Config 位置下的 `rules/`）。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

* * *

Step 1：一键生成所有文件（推荐）
===================

把下面脚本保存为 `scripts/setup_codex_ai_team.sh`，在 repo 根目录执行：

```
#!/usr/bin/env bash
set -euo pipefail

echo "==> Create directories"
mkdir -p .codex/agents .codex/rules
mkdir -p .agents/skills/{tdd-loop,run-ci,generate-openapi,generate-mermaid,pr-checklist}/{agents,assets,scripts,references} || true
mkdir -p docs/{architecture,openapi}

echo "==> Write .codex/config.toml"
cat > .codex/config.toml << 'EOF'
# =========================
# Codex Project Config
# =========================
# 注意：项目级 .codex/config.toml 只在该项目被标记为 trusted 时加载
# 详见 Codex config basics
# =========================

# 建议：默认用 workspace-write（可改代码），网络默认关闭（更稳妥）
sandbox_mode = "workspace-write"
approval_policy = "on-request"

# 默认模型（按你们团队习惯）
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"

[features]
multi_agent = true
collaboration_modes = true

[agents]
max_threads = 6
max_depth = 1

[agents.team_lead]
description = "团队Lead：先产出可执行计划与门禁，再按协议调度架构/实现/测试/评审，最后汇总PR建议与合并checklist。"
config_file = "agents/team_lead.toml"

[agents.architect]
description = "架构师：产出Mermaid图、模块划分、OpenAPI契约/数据模型与关键风险点；尽量只改docs与契约文件。"
config_file = "agents/architect.toml"

[agents.be_developer]
description = "后端开发：严格TDD实现服务端逻辑/API/DB，按协议跑测试并产出证据。支持 Spring Boot 与 Python 后端。"
config_file = "agents/be_developer.toml"

[agents.fe_developer]
description = "前端开发：TS + 现有组件库实现UI与联调；每次修改跑lint/test（如项目包含前端）。"
config_file = "agents/fe_developer.toml"

[agents.tester]
description = "测试工程：补齐单测/集成/E2E，目标覆盖率>80%，跑测试并输出失败复现与日志证据。"
config_file = "agents/tester.toml"

[agents.reviewer]
description = "资深评审：只读审查 correctness/security/perf/test gaps，给可执行改动建议与结论。"
config_file = "agents/reviewer.toml"
EOF

echo "==> Write 6 agent configs"

cat > .codex/agents/team_lead.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
# Lead 默认不跑命令，避免把噪音带进主线程；必要时可让 worker/tester 跑
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Team Lead】（项目总负责人）。必须遵守《AGENTS.md》中“Team Protocol”。

强制流程（gated handoff）：
1) 先产出计划：写入 docs/ai/PLAN.md（如果没有 docs/ai 目录就创建）。
2) 计划必须包含：任务分解、依赖、风险、每个角色的交付物路径（文件名）。
3) 未看到上游角色交付物文件存在，不得启动下游角色。

你可以并行启动 read-only 型子任务（架构评审、代码探索），但写代码/写测试尽量串行或分区执行，避免冲突。

最终汇总必须输出：
- PR 概述（What/Why/How）
- 风险评估（回滚方案）
- 合并 checklist（必须可勾选）
"""
EOF

cat > .codex/agents/architect.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Architect】。遵守《AGENTS.md》中“Protocol & Output Schema”。

你必须产出（写入文件）：
1) docs/architecture/ARCHITECTURE.md
   - Mermaid：组件图（C4/组件图）+ 关键序列图（1-3条）
   - 模块边界、职责、依赖方向
   - 关键非功能性：可维护性/扩展性/性能/安全
2) docs/openapi/README.md（说明 OpenAPI 生成策略：Spring Boot / Python）
3) 如项目已有 OpenAPI 规范文件：更新 docs/openapi/openapi.yaml 或 openapi.json（按项目约定）

注意：
- Architect 只允许改 docs/**、openapi/**、*.md、*.yaml、*.json（契约相关）
- 不要直接改业务代码，除非 Team Lead 明确授权
"""
EOF

cat > .codex/agents/be_developer.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Backend Developer】。支持 Java Spring Boot 与 Python 后端。

硬规则（TDD）：
- 先写测试（red），再最小实现（green），再重构（refactor）
- 每个循环都要跑测试并记录证据（命令 + 关键输出）
- 改动必须小步提交到“工作区变更集”（不要自己 git commit/push）

交付物（必须写文件）：
- docs/ai/DEV_NOTES.md（记录：做了什么、跑了什么、结果、风险）
- 新增/更新的测试文件
- 实现代码

优先使用 skills：
- $tdd-loop
- $run-ci
- $generate-openapi（如果涉及 API 变更）
"""
EOF

cat > .codex/agents/fe_developer.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Frontend Developer】（若仓库包含前端才启用）。

规则：
- TypeScript + 现有UI库；组件可复用、可测试
- 每次修改后跑 lint/test（按 AGENTS.md 的命令约定）
- 输出 docs/ai/FE_NOTES.md：实现清单、联调点、截图/录屏（如可用）、风险

优先使用 skills：
- $run-ci
- $pr-checklist（用于整理PR说明）
"""
EOF

cat > .codex/agents/tester.toml << 'EOF'
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Tester】。子代理无法交互批准（non-interactive approvals），所以：
- 尽量只跑“项目允许的测试命令”，避免触发需要额外批准/越权的动作
- 若测试依赖网络/外部服务，先在 docs/ai/TEST_REPORT.md 写明前置条件与替代方案

必须交付：
- docs/ai/TEST_REPORT.md
  - 覆盖范围（单测/集成/E2E）
  - 关键命令与结果（失败要含复现步骤、日志片段、最小重现）
  - 覆盖率（如可获得）与未覆盖风险点
- 补齐必要测试（目标 >80% 覆盖率是目标，不可行要解释）

优先使用 skills：
- $run-ci
- $tdd-loop（用于补测试）
"""
EOF

cat > .codex/agents/reviewer.toml << 'EOF'
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
sandbox_mode = "read-only"

developer_instructions = """
你是【Reviewer】（只读）。按《AGENTS.md》输出“Review Report Schema”。

必须检查：
- 安全（OWASP Top 10 思路：输入校验、鉴权、敏感信息、注入、反序列化等）
- Correctness（边界/异常/并发）
- 性能热点（N+1、IO、缓存、事务）
- 可维护性（结构、命名、重复、复杂度）
- 测试质量（覆盖率、断言质量、易碎测试）

输出：
- 在聊天中给出结论：✅通过 / ❌需要修改
- 提供具体可执行的修改建议（最好带 diff 片段）
"""
EOF

echo "==> Write AGENTS.md (only if not exists)"
if [ ! -f AGENTS.md ]; then
cat > AGENTS.md << 'EOF'
# Team Protocol for Codex (6-agent)

## 1) Repository Commands (keep in sync with CI)
### Java / Spring Boot (Maven or Gradle)
- Unit tests:
  - Maven: `./mvnw test` (fallback: `mvn test`)
  - Gradle: `./gradlew test` (fallback: `gradle test`)
- Lint/format (if configured):
  - Maven/Gradle plugin tasks as defined by repo (spotless/checkstyle)
- OpenAPI:
  - Preferred output: `docs/openapi/openapi.json` or `docs/openapi/openapi.yaml`

### Python
- Install deps (human-run, one-time): `python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
- Unit tests: `python -m pytest -q`
- Lint: `python -m ruff check .` (if ruff exists)
- Format: `python -m black .` (if black exists)
- Type check: `python -m mypy .` (if configured)
- OpenAPI:
  - Preferred output: `docs/openapi/openapi.json` or `docs/openapi/openapi.yaml`

## 2) Output Protocol (必须遵守)
所有角色输出必须包含以下结构（无论写在聊天里还是写文件里）：

### Output Schema (Markdown)
- Goal
- Changes (files)
- Commands Run (with results)
- Evidence (paths + key symbols)
- Risks & Mitigations
- Handoff (next role + exact next step)

## 3) Gated Handoff (门禁交接)
- Team Lead 必须先写 `docs/ai/PLAN.md`
- Architect 必须写 `docs/architecture/ARCHITECTURE.md`
- Developer 必须写 `docs/ai/DEV_NOTES.md`
- Tester 必须写 `docs/ai/TEST_REPORT.md`
- Reviewer 必须给出 ✅/❌ 结论与明确修改项

没有上游交付物文件 => 不允许进入下游阶段。

## 4) Safety Defaults
- 不允许 agent 执行 `git commit/push`（保持人为把关）
- 不允许改 `.codex/**` 与 `.agents/**`（这些路径在 workspace-write 下通常也是受保护只读）
EOF
fi

echo "==> Write rules: Java Spring Boot + Python"
cat > .codex/rules/java_springboot.rules << 'EOF'
# Java / Spring Boot execpolicy rules
# Control which commands Codex can run outside the sandbox.
# Docs: https://developers.openai.com/codex/rules/

# --- Hard blocks (outside sandbox) ---
prefix_rule(
  pattern = ["sudo"],
  decision = "forbidden",
  justification = "Never run sudo from Codex. Do it manually if needed.",
  match = ["sudo ls"],
)

prefix_rule(
  pattern = ["rm"],
  decision = "forbidden",
  justification = "Block destructive deletes outside sandbox. Use git restore/clean manually if needed.",
  match = ["rm -rf /"],
)

prefix_rule(
  pattern = ["ssh"],
  decision = "forbidden",
  justification = "No SSH from Codex.",
  match = ["ssh user@host"],
)

prefix_rule(
  pattern = ["scp"],
  decision = "forbidden",
  justification = "No SCP from Codex.",
  match = ["scp file user@host:/tmp/"],
)

prefix_rule(
  pattern = ["curl"],
  decision = "prompt",
  justification = "Network calls are risky. Prefer offline generation. If you must, approve explicitly.",
  match = ["curl http://localhost:8080/actuator/health"],
)

# --- Read-only Git operations (safe) ---
prefix_rule(
  pattern = ["git", ["status", "diff", "log", "show"]],
  decision = "allow",
  justification = "Allow read-only git commands.",
  match = ["git status", "git diff", "git log -n 20", "git show HEAD"],
)

prefix_rule(
  pattern = ["git", "rev-parse"],
  decision = "allow",
  justification = "Allow git metadata reads.",
  match = ["git rev-parse --show-toplevel"],
)

# --- Block destructive git operations outside sandbox ---
prefix_rule(
  pattern = ["git", ["commit", "push", "reset", "clean", "rebase"]],
  decision = "forbidden",
  justification = "Keep commits/pushes and destructive operations manual (human gate).",
  match = ["git commit -m x", "git push", "git reset --hard HEAD~1", "git clean -fdx"],
)

# --- Java build/test tools (often needed when sandbox blocks network or .git writes) ---
prefix_rule(
  pattern = ["./mvnw"],
  decision = "allow",
  justification = "Allow Maven wrapper outside sandbox when necessary for CI-like runs.",
  match = ["./mvnw test"],
)

prefix_rule(
  pattern = ["mvn"],
  decision = "allow",
  justification = "Allow Maven outside sandbox when necessary for CI-like runs.",
  match = ["mvn test"],
)

prefix_rule(
  pattern = ["./gradlew"],
  decision = "allow",
  justification = "Allow Gradle wrapper outside sandbox when necessary for CI-like runs.",
  match = ["./gradlew test"],
)

prefix_rule(
  pattern = ["gradle"],
  decision = "allow",
  justification = "Allow Gradle outside sandbox when necessary for CI-like runs.",
  match = ["gradle test"],
)

# --- Java runtime (occasionally used for codegen/formatters) ---
prefix_rule(
  pattern = ["java"],
  decision = "prompt",
  justification = "Allow java invocations only with explicit approval when outside sandbox.",
  match = ["java -version"],
)
EOF

cat > .codex/rules/python.rules << 'EOF'
# Python execpolicy rules
# Control which commands Codex can run outside the sandbox.
# Docs: https://developers.openai.com/codex/rules/

# --- Hard blocks (outside sandbox) ---
prefix_rule(
  pattern = ["sudo"],
  decision = "forbidden",
  justification = "Never run sudo from Codex.",
  match = ["sudo ls"],
)

prefix_rule(
  pattern = ["rm"],
  decision = "forbidden",
  justification = "Block destructive deletes outside sandbox.",
  match = ["rm -rf /"],
)

prefix_rule(
  pattern = ["ssh"],
  decision = "forbidden",
  justification = "No SSH from Codex.",
  match = ["ssh user@host"],
)

prefix_rule(
  pattern = ["curl"],
  decision = "prompt",
  justification = "Network calls are risky. Prefer offline generation (import app) for OpenAPI.",
  match = ["curl http://localhost:8000/health"],
)

# --- Read-only Git operations (safe) ---
prefix_rule(
  pattern = ["git", ["status", "diff", "log", "show"]],
  decision = "allow",
  justification = "Allow read-only git commands.",
  match = ["git status", "git diff", "git log -n 20", "git show HEAD"],
)

prefix_rule(
  pattern = ["git", "rev-parse"],
  decision = "allow",
  justification = "Allow git metadata reads.",
  match = ["git rev-parse --show-toplevel"],
)

# --- Block destructive git operations outside sandbox ---
prefix_rule(
  pattern = ["git", ["commit", "push", "reset", "clean", "rebase"]],
  decision = "forbidden",
  justification = "Keep commits/pushes and destructive operations manual (human gate).",
  match = ["git commit -m x", "git push", "git reset --hard HEAD~1", "git clean -fdx"],
)

# --- Python test/lint tools (outside sandbox) ---
prefix_rule(
  pattern = [["python", "python3"]],
  decision = "allow",
  justification = "Allow python invocations outside sandbox (tests, linters, codegen).",
  match = ["python -V", "python -m pytest -q"],
)

prefix_rule(
  pattern = ["pytest"],
  decision = "allow",
  justification = "Allow pytest outside sandbox for CI-like runs.",
  match = ["pytest -q"],
)

prefix_rule(
  pattern = ["ruff"],
  decision = "allow",
  justification = "Allow ruff outside sandbox.",
  match = ["ruff check ."],
)

prefix_rule(
  pattern = ["black"],
  decision = "allow",
  justification = "Allow black outside sandbox.",
  match = ["black --check ."],
)

prefix_rule(
  pattern = ["mypy"],
  decision = "allow",
  justification = "Allow mypy outside sandbox if configured.",
  match = ["mypy ."],
)

# --- Package managers: prompt (supply-chain / install scripts risk) ---
prefix_rule(
  pattern = [["pip", "pip3"]],
  decision = "prompt",
  justification = "Installing packages can execute arbitrary build scripts. Approve explicitly.",
  match = ["pip install -r requirements.txt"],
)

prefix_rule(
  pattern = ["poetry"],
  decision = "prompt",
  justification = "Poetry may install packages / run scripts. Approve explicitly.",
  match = ["poetry install"],
)

prefix_rule(
  pattern = ["uv"],
  decision = "prompt",
  justification = "uv may install packages. Approve explicitly.",
  match = ["uv sync"],
)
EOF

echo "==> Write skills"

# Skill 1: TDD loop
cat > .agents/skills/tdd-loop/SKILL.md << 'EOF'
---
name: tdd-loop
description: "Use when implementing a feature or bugfix with tests-first. Enforces Red-Green-Refactor. Supports Java Spring Boot and Python."
---

# TDD Loop (Java Spring Boot + Python)

## Trigger
Use this skill whenever:
- You add a new endpoint / business rule / bug fix
- Behavior must be proven with tests
Do NOT use for pure formatting or documentation-only changes.

## Inputs
- Feature description / bug reproduction
- Target module/service path (if monorepo)

## Workflow (Red → Green → Refactor)
1) **Red**: write the smallest failing test that captures the behavior.
2) **Green**: implement minimal code to make it pass.
3) **Refactor**: clean up (no behavior changes), keep tests green.

## Java / Spring Boot recipe (best practice)
- Prefer JUnit 5 + Mockito
- Choose the lightest test slice:
  - Controller: `@WebMvcTest` + MockMvc
  - Service: plain unit test + mocks
  - Integration: `@SpringBootTest` only when needed
- Run: `./mvnw test` or `./gradlew test`
- Keep test names behavior-driven and assertions explicit.

## Python recipe (best practice)
- Prefer pytest (+ pytest-mock as needed)
- Structure: Arrange → Act → Assert
- Run: `python -m pytest -q`
- If API framework is FastAPI:
  - Use TestClient/httpx and assert status + payload

## Required output (Protocol)
After each loop, record:
- files changed
- command run + result
- what behavior is now guaranteed
Write updates into `docs/ai/DEV_NOTES.md` (developer) or `docs/ai/TEST_REPORT.md` (tester).
EOF

cat > .agents/skills/tdd-loop/agents/openai.yaml << 'EOF'
policy:
  allow_implicit_invocation: false
EOF

# Skill 2: run-ci
cat > .agents/skills/run-ci/SKILL.md << 'EOF'
---
name: run-ci
description: "Use to run the same checks as CI locally (tests + lint + typecheck where applicable). Supports Java Spring Boot and Python."
---

# Run CI Locally (Java + Python)

## Trigger
Use when:
- Before handing off to Reviewer
- After a non-trivial change
- When a failure needs reproduction

## Workflow
1) Inspect CI definition:
   - If `.github/workflows/*.yml` exists, read it and mimic the same steps locally.
2) If no workflow is present, fall back to the project commands in `AGENTS.md`.

## Java defaults
- `./mvnw test` (or `mvn test`)
- or `./gradlew test` (or `gradle test`)
- If repo has format/lint tasks (spotless/checkstyle), run them as CI specifies.

## Python defaults
- `python -m pytest -q`
- If configured: `python -m ruff check .`
- If configured: `python -m black --check .`
- If configured: `python -m mypy .`

## Output
Write `docs/ai/TEST_REPORT.md`:
- Commands run + pass/fail
- Failure reproduction steps
- Key logs (short excerpts)
EOF

cat > .agents/skills/run-ci/agents/openai.yaml << 'EOF'
policy:
  allow_implicit_invocation: false
EOF

# Skill 3: generate-openapi
cat > .agents/skills/generate-openapi/SKILL.md << 'EOF'
---
name: generate-openapi
description: "Use when API endpoints/contracts change. Produces/updates docs/openapi/openapi.json or .yaml. Supports Spring Boot (springdoc) and Python (FastAPI-style import)."
---

# Generate OpenAPI (Spring Boot + Python)

## Trigger
Use whenever:
- You add/modify REST endpoints
- Request/response schema changes
- Auth rules affecting API behavior changes

## Output target (team standard)
- `docs/openapi/openapi.json` (preferred for generated)
- or `docs/openapi/openapi.yaml` (if your ecosystem is YAML-first)

## Spring Boot best practice (contract stays in sync)
**Preferred approach**: add a **snapshot test** that generates OpenAPI from the app context:
- Ensure you use `springdoc-openapi` (common: `/v3/api-docs`)
- In a test, call `/v3/api-docs` via MockMvc and write to `docs/openapi/openapi.json`
- Then CI can ensure the file is up to date (diff check)

Rationale: avoids requiring a running server or curl, keeps contract versioned.

## Python best practice (FastAPI-style)
**Preferred approach**: import app and write `app.openapi()` output:
- Find the module where `app = FastAPI()` (commonly `app.main:app` or `src.main:app`)
- Generate JSON without running a server
- Write to `docs/openapi/openapi.json`

## Validation (optional but recommended)
- Validate OpenAPI with an OpenAPI validator in CI (Spectral/openapi-cli) if your org uses it.

## Output protocol
Update:
- `docs/openapi/openapi.json` (or yaml)
- `docs/openapi/README.md` describing how to regenerate
Record in `docs/ai/DEV_NOTES.md` what changed and why.
EOF

cat > .agents/skills/generate-openapi/agents/openai.yaml << 'EOF'
policy:
  allow_implicit_invocation: false
EOF

# Skill 4: generate-mermaid
cat > .agents/skills/generate-mermaid/SKILL.md << 'EOF'
---
name: generate-mermaid
description: "Use when documenting architecture or complex flows. Produces Mermaid diagrams (C4/component/sequence) in docs/architecture."
---

# Generate Mermaid Diagrams

## Trigger
Use when:
- Architecture changes
- New cross-module flow
- Non-trivial async/transaction flow

## Output
Write/update `docs/architecture/ARCHITECTURE.md` with Mermaid code blocks:
- Component diagram (C4-ish)
- 1-3 key sequence diagrams

## Best practice
- Diagrams must match real code boundaries (packages/modules)
- Keep diagrams small and focused; split by bounded context
- Put the decision rationale under each diagram (why this boundary)

## Protocol
Always cite:
- modules/packages involved
- key classes/functions
- endpoints/events involved
EOF

# Skill 5: pr-checklist
cat > .agents/skills/pr-checklist/SKILL.md << 'EOF'
---
name: pr-checklist
description: "Use when preparing a PR summary/checklist. Produces a PR-ready description and merge checklist."
---

# PR Checklist Builder

## Trigger
Use when:
- Team Lead is ready to open PR
- Reviewer requests a clearer summary
- You need a risk/rollback checklist

## Output (paste into PR)
### What
### Why
### How
### Tests
- [ ] Unit
- [ ] Integration
- [ ] E2E (if applicable)
### OpenAPI / Docs
- [ ] OpenAPI regenerated (if API changed)
- [ ] Mermaid/docs updated (if architecture changed)
### Risk & Rollback
### Monitoring / Metrics (if applicable)

Also produce a "Merge checklist" at the end with explicit commands run and results.
EOF

echo "==> Done."
echo "Next: trust the project in Codex, restart Codex, then use /skills to see skills."
```

执行后你会得到完整体系。

* * *

Step 2：确认 Codex 能加载（必须做）
========================

2.1 信任项目（Trusted project）
-------------------------

项目级 `.codex/config.toml` 只有在项目被标记为 trusted 时才会加载。[OpenAI Developers](https://developers.openai.com/codex/config-basic/)

2.2 启用 Multi‑agents
-------------------

你已经在 `.codex/config.toml` 里加了：

```
[features]
multi_agent = true
```

Multi‑agents 是实验特性，需要显式开启并重启。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

* * *

Step 3：两套 Rules（你要的 JAVA / Python），以及如何验证
=========================================

Rules 的语义：**控制哪些命令可以在“离开 sandbox（outside the sandbox）”时自动允许/提示/禁止**。规则用 `prefix_rule()`，并且可以写 `match/not_match` 做加载时校验。[OpenAI Developers](https://developers.openai.com/codex/rules/)

> 为什么我们要 rules：在 `workspace-write` 下，`.git/.codex/.agents` 往往是受保护只读路径；有些操作（例如 `git commit`）会触发“需要 outside sandbox”的请求。我们通过 rules 把这些高风险操作直接禁掉。[OpenAI Developers+1](https://developers.openai.com/codex/security/)

3.1 Java / Spring Boot rules
----------------------------

文件：`.codex/rules/java_springboot.rules`  
（上面脚本已生成；你也可直接复制我给的内容）

3.2 Python rules
----------------

文件：`.codex/rules/python.rules`  
（上面脚本已生成）

3.3 用命令验证规则是否按你预期匹配
-------------------

官方推荐用：

```
codex execpolicy check --pretty \
  --rules .codex/rules/java_springboot.rules \
  -- mvn test
```

或：

```
codex execpolicy check --pretty \
  --rules .codex/rules/python.rules \
  -- python -m pytest -q
```

`codex execpolicy check` 会输出“最严格 decision”与命中规则列表，用来避免写错 prefix。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

* * *

Step 4：把“6-agent”升级成“角色 + 协议 + 技能”的系统（落地运行方式）
=============================================

这里是你要的“进一步完善”的核心：**把质量门禁从“口头要求”变成“协议化 + 技能化 + 可复用资产”**。

4.1 角色（Roles）：让 6-agent 真能跑、且互不干扰
---------------------------------

你要达成的运行原则（最佳实践）：

*   **Team Lead**：计划与门禁，不做实现（减少上下文污染）
*   **Architect**：只产出 docs/契约，不碰业务代码
*   **Dev / Tester**：唯一写代码的人群（串行或分区）
*   **Reviewer**：只读、给结论与可执行建议
*   子代理无法弹批准窗口（non‑interactive approvals），所以你要尽量让子代理的动作**不依赖临时批准**。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

> 这就是为什么我们把“危险命令”用 rules 禁掉，并且把技能中“会触发网络/系统副作用”的动作改为**显式调用**（`allow_implicit_invocation: false`）。

4.2 协议（Protocol）：用 AGENTS.md 固化“交付物 + 输出结构 + 门禁”
------------------------------------------------

Codex 会在开始工作前读取 `AGENTS.md`，并按全局→项目→子目录叠加规则。[OpenAI Developers+1](https://developers.openai.com/codex/guides/agents-md/?utm_source=chatgpt.com)  
所以把协议写在 `AGENTS.md` 是最稳的“团队级承载面”。

你已经在脚本里落了：

*   `docs/ai/PLAN.md`（Lead）
*   `docs/architecture/ARCHITECTURE.md`（Architect）
*   `docs/ai/DEV_NOTES.md`（Dev）
*   `docs/ai/TEST_REPORT.md`（Tester）
*   Reviewer 输出 ✅/❌ + diff 建议

这就实现了你要的 gated handoff（检查文件存在才允许推进）。

4.3 技能（Skills）：把 best practice 变成“可复用工作流”
-----------------------------------------

Skills 在 Codex CLI/IDE/App 都可用，且支持 progressive disclosure（只在触发时注入完整 SKILL.md）。[OpenAI Developers+1](https://developers.openai.com/codex/skills/)  
并且 Codex 会从 repo 的 `.agents/skills` 扫描技能。[OpenAI Developers](https://developers.openai.com/codex/skills/)

你现在有 5 个核心技能，对应你点名的 best practice：

1.  `tdd-loop`：强制 Red‑Green‑Refactor（Java + Python）
2.  `run-ci`：本地复刻 CI（优先读 `.github/workflows`）
3.  `generate-openapi`：API 变更必更新契约
4.  `generate-mermaid`：架构/流程必文档化
5.  `pr-checklist`：PR 描述与合并门禁模板化

并且对“会触发命令执行/副作用”的技能（TDD/CI/OpenAPI）我都建议禁用隐式触发：  
`policy.allow_implicit_invocation: false`（需要你显式 `$skill` 调用）。[OpenAI Developers](https://developers.openai.com/codex/skills/)

* * *

Step 5：日常使用流程（你要的“一步到位”工作方式）
============================

5.1 典型开发需求（推荐提示词模板）
-------------------

在 Codex 里直接说：

> 使用我们的 6-agent team：实现 XXX。
> 
> 1.  Team Lead 先生成 docs/ai/PLAN.md（按协议拆分）
> 2.  Architect 输出 docs/architecture/ARCHITECTURE.md + OpenAPI 策略
> 3.  BE Developer 严格 TDD 实现
> 4.  Tester 补齐测试并产出 TEST\_REPORT
> 5.  Reviewer 给 ✅/❌ + diff 建议
> 6.  Team Lead 汇总 PR checklist
>     

5.2 使用技能（建议显式调用）
----------------

在 CLI/IDE 里可 `/skills` 查看，然后：

*   `用 $tdd-loop 按红绿重构实现这个 endpoint`
*   `用 $run-ci 跑一遍跟 CI 一样的检查，并写 TEST_REPORT`
*   `API 改动后用 $generate-openapi 更新 docs/openapi/openapi.json`
*   `用 $generate-mermaid 更新 docs/architecture/ARCHITECTURE.md`
*   `最后用 $pr-checklist 生成 PR 描述与合并 checklist`

Skills 支持显式提及触发（CLI/IDE：`/skills` 或 `$` 提及）。[OpenAI Developers](https://developers.openai.com/codex/skills/)

* * *

你现在就可以直接复制用的“最终交付清单”
====================

你要的“可直接使用配置”，在上面脚本里已经包含全部：

*   ✅ `.codex/config.toml`（multi\_agent + 6 roles 注册，且 `config_file` 路径相对 `.codex/config.toml`，符合 Codex 规则）[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)
*   ✅ 6 个 `.codex/agents/*.toml`（角色职责、门禁、技能调用点）
*   ✅ `AGENTS.md`（协议与门禁）[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/?utm_source=chatgpt.com)
*   ✅ 两套 rules：
    *   `.codex/rules/java_springboot.rules`
    *   `.codex/rules/python.rules`  
        并提供 `codex execpolicy check` 校验方式[OpenAI Developers+1](https://developers.openai.com/codex/rules/)
*   ✅ 5 个 skills：`.agents/skills/*/SKILL.md`（并对高副作用技能禁用隐式触发）[OpenAI Developers](https://developers.openai.com/codex/skills/)

* * *

我建议你下一步做的唯一“微调”（不需要我问你问题，你照做即可）
-------------------------------

1.  **如果你希望 tester/CI 不被网络限制卡住**：考虑在项目 `.codex/config.toml` 加上（风险更高）：
    ```
    [sandbox_workspace_write]
    network_access = true
    ```
    workspace-write 默认网络是关闭的；需要显式开启。[OpenAI Developers](https://developers.openai.com/codex/security/)
2.  **把 PR checklist 固化为仓库模板**（业界常用做法）：  
    让 Team Lead 每次 PR 自动有 checklist
    *   `.github/PULL_REQUEST_TEMPLATE.md`（你可以让 `$pr-checklist` 技能产出后落盘）

如果你把你们 Java 项目当前用的是 **Maven 还是 Gradle**、Python 是 **requirements.txt / Poetry / uv / pip-tools** 哪一种（只要一句话），我可以把 `run-ci` 和 `generate-openapi` 这两项技能再做一次“完全贴合你们栈的命令与路径约定”，让触发更稳定、输出路径更统一。

## Prompt:
Java 是 Maven Python实是requirements.txt /

## Response:
Pro thinking • Java Maven vs Python

Answer now



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)