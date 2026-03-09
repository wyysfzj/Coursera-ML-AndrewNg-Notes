# Native vs 自定义 Agent

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 3/3/2026 0:00:09  
**Updated:** 3/9/2026 1:32:50  
**Exported:** 3/9/2026 8:51:35  
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
Thought for 34m 6s

## Response:
已确认：**Java = Spring Boot + Maven**，**Python = requirements.txt（pip）**。下面我给你“可直接落地”的一整套：**两套 Rules（Java / Python）+ 6‑agent（角色）+ Protocol（协议）+ Skills（技能）**，并把 TDD loop / Run CI / Generate OpenAPI / Generate Mermaid / PR checklist 全部固化到仓库里。

> 关键运行约束：Multi‑agents 需要显式开启；子代理是 **non‑interactive approvals**（不能弹批准框），所以我们要尽量让测试/CI/OpenAPI 生成在“无需临时批准”的路径里闭环。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/?utm_source=chatgpt.com)  
> 另外：Rules 是用来控制“**离开 sandbox** 执行命令”时的 allow/prompt/forbidden；Codex 会在启动时扫描每个 Team Config 位置下的 `rules/`。[OpenAI Developers](https://developers.openai.com/codex/rules/)

* * *

0) 最终目录（你照着放就能用）
----------------

建议把“团队系统”放在 repo 内（团队共享），结构如下：

```
repo/
  AGENTS.md
  .codex/
    config.toml
    agents/
      team_lead.toml
      architect.toml
      be_developer.toml
      fe_developer.toml
      tester.toml
      reviewer.toml
    rules/
      default.rules              # 你选择 Java 或 Python 的那一套（或用合并版）
      java_springboot_maven.rules
      python_requirements.rules
  .agents/
    skills/
      tdd-loop/SKILL.md
      run-ci/SKILL.md
      generate-openapi/SKILL.md
      generate-mermaid/SKILL.md
      pr-checklist/SKILL.md
  scripts/
    openapi/
      generate_openapi.py        # Python OpenAPI 生成脚本（FastAPI 优先）
```

*   `AGENTS.md` 是“协议面”，Codex 会按规则加载（全局→项目→子目录覆盖）。[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/?utm_source=chatgpt.com)
*   Skills 放 `.agents/skills`，Codex 会从当前目录一路扫描到 repo root。[OpenAI Developers+1](https://developers.openai.com/codex/skills/?utm_source=chatgpt.com)
*   `rules/` 放在 `.codex/` 同级下（Team Config 位置），启动时自动扫描。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

* * *

1) 两套 Rules（你要的：Java / Python）
------------------------------

### 1.1 Java / Spring Boot + Maven 版（建议用于 Java 仓库）

保存为：`.codex/rules/java_springboot_maven.rules`  
（如果这是纯 Java repo，把它复制成 `.codex/rules/default.rules`）

> 设计目标：
> 
> *   允许 **只读 git**（status/diff/log/show）
> *   禁止 git commit/push/reset/clean/rebase（保留人工 gate）
> *   允许 Maven **标准化命令**（我们在 skills 里固定用这些命令，避免 options 顺序导致不匹配）
> *   禁止 sudo/ssh/scp/rm 等危险命令
> *   `curl` 默认 prompt（OpenAPI 我们走 MockMvc/插件生成，尽量不依赖 curl）
>     
> 
> Rules 语义与 `prefix_rule()` 的字段定义见官方文档。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

```
# .codex/rules/java_springboot_maven.rules

# -----------------------------
# Hard blocks
# -----------------------------
prefix_rule(
  pattern = ["sudo"],
  decision = "forbidden",
  justification = "Never run sudo from Codex. Do it manually if needed.",
  match = ["sudo ls"],
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
  pattern = ["rm"],
  decision = "forbidden",
  justification = "Block destructive deletes outside sandbox. Use git restore/clean manually if needed.",
  match = ["rm -rf /"],
)

# Network calls: prompt by default (safer)
prefix_rule(
  pattern = ["curl"],
  decision = "prompt",
  justification = "Network calls are risky. Prefer offline OpenAPI generation. If you must, approve explicitly.",
  match = ["curl http://localhost:8080/actuator/health"],
)

# -----------------------------
# Git: allow read-only, forbid write/destructive
# -----------------------------
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

prefix_rule(
  pattern = ["git", ["commit", "push", "reset", "clean", "rebase"]],
  decision = "forbidden",
  justification = "Keep commits/pushes and destructive git ops manual (human gate).",
  match = ["git commit -m x", "git push", "git reset --hard HEAD~1", "git clean -fdx"],
)

# -----------------------------
# Maven: allow only canonical CI commands (used by skills)
# NOTE: prefix match is strict; keep skills commands exactly as below.
# -----------------------------
prefix_rule(
  pattern = ["./mvnw", "-B", "-q", "test"],
  decision = "allow",
  justification = "Allow running unit tests via Maven wrapper (canonical).",
  match = ["./mvnw -B -q test"],
)

prefix_rule(
  pattern = ["./mvnw", "-B", "-q", "verify"],
  decision = "allow",
  justification = "Allow running verify (unit + integration if configured) via Maven wrapper (canonical).",
  match = ["./mvnw -B -q verify"],
)

# Fallback if wrapper not present (still canonical)
prefix_rule(
  pattern = ["mvn", "-B", "-q", "test"],
  decision = "allow",
  justification = "Allow mvn test (canonical).",
  match = ["mvn -B -q test"],
)

prefix_rule(
  pattern = ["mvn", "-B", "-q", "verify"],
  decision = "allow",
  justification = "Allow mvn verify (canonical).",
  match = ["mvn -B -q verify"],
)

# Guardrails: forbid publishing-ish goals by default
prefix_rule(
  pattern = [["./mvnw", "mvn"], ["deploy", "release:perform", "release:prepare"]],
  decision = "forbidden",
  justification = "Publishing/release operations must be manual.",
  match = ["mvn deploy"],
)
```

* * *

### 1.2 Python / requirements.txt 版（建议用于 Python 仓库）

保存为：`.codex/rules/python_requirements.rules`  
（如果这是纯 Python repo，把它复制成 `.codex/rules/default.rules`）

> 设计目标：
> 
> *   放行 **pytest / ruff / black / mypy** 的 canonical 命令
> *   放行 **pip install -r requirements.txt**（你要“一步到位”，否则子代理会因为无法批准而卡住）
> *   仍然禁止 sudo/ssh/scp/rm 等
>     
> 
> 说明：pip 安装确实有供应链风险；但在“已 trust 的仓库”里，一步到位通常就是你要的体验。你如果想更严格，把 pip install 改成 `decision="prompt"` 即可。

```
# .codex/rules/python_requirements.rules

# -----------------------------
# Hard blocks
# -----------------------------
prefix_rule(
  pattern = ["sudo"],
  decision = "forbidden",
  justification = "Never run sudo from Codex.",
  match = ["sudo ls"],
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
  pattern = ["rm"],
  decision = "forbidden",
  justification = "Block destructive deletes outside sandbox.",
  match = ["rm -rf /"],
)

prefix_rule(
  pattern = ["curl"],
  decision = "prompt",
  justification = "Network calls are risky. Prefer offline OpenAPI generation by importing the app.",
  match = ["curl http://localhost:8000/health"],
)

# -----------------------------
# Git: allow read-only, forbid write/destructive
# -----------------------------
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

prefix_rule(
  pattern = ["git", ["commit", "push", "reset", "clean", "rebase"]],
  decision = "forbidden",
  justification = "Keep commits/pushes and destructive git ops manual (human gate).",
  match = ["git commit -m x", "git push", "git reset --hard HEAD~1", "git clean -fdx"],
)

# -----------------------------
# Python tooling (canonical commands used by skills)
# -----------------------------
prefix_rule(
  pattern = ["python", "-m", "pytest", "-q"],
  decision = "allow",
  justification = "Allow running pytest (canonical).",
  match = ["python -m pytest -q"],
)

prefix_rule(
  pattern = ["python", "-m", "ruff", "check", "."],
  decision = "allow",
  justification = "Allow ruff check (canonical).",
  match = ["python -m ruff check ."],
)

prefix_rule(
  pattern = ["python", "-m", "black", "--check", "."],
  decision = "allow",
  justification = "Allow black --check (canonical).",
  match = ["python -m black --check ."],
)

prefix_rule(
  pattern = ["python", "-m", "mypy", "."],
  decision = "allow",
  justification = "Allow mypy (canonical).",
  match = ["python -m mypy ."],
)

# Install deps (one-step): allow requirements.txt installs
prefix_rule(
  pattern = ["python", "-m", "pip", "install", "-r", "requirements.txt"],
  decision = "allow",
  justification = "Allow installing Python deps from requirements.txt (one-step).",
  match = ["python -m pip install -r requirements.txt"],
)

# OpenAPI generator script
prefix_rule(
  pattern = ["python", "scripts/openapi/generate_openapi.py"],
  decision = "allow",
  justification = "Allow generating OpenAPI offline by importing the app.",
  match = ["python scripts/openapi/generate_openapi.py"],
)
```

* * *

### 1.3 你在 Codex 里怎么验证 Rules 是否命中（强烈建议做）

官方提供了 `codex execpolicy check`。[OpenAI Developers+1](https://developers.openai.com/codex/rules/?utm_source=chatgpt.com)

例如（Java）：

```
codex execpolicy check --pretty --rules .codex/rules/java_springboot_maven.rules -- ./mvnw -B -q test
```

例如（Python）：

```
codex execpolicy check --pretty --rules .codex/rules/python_requirements.rules -- python -m pip install -r requirements.txt
```

* * *

2) “一步到位”落地方案（Step-by-step）
---------------------------

### Step 1：启用 Multi‑agents + 注册 6 个角色（项目级 .codex/config.toml）

Multi‑agents 目前是 experimental，必须显式开启并重启。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/?utm_source=chatgpt.com)

保存为：`.codex/config.toml`

> 重点：我在这里**默认开启 workspace-write 的 network\_access**，让 Maven/pip 能在 sandbox 内直接拉依赖，不必“离开 sandbox”。默认情况下 workspace-write 网络是关闭的，需要显式打开。[OpenAI Developers](https://developers.openai.com/codex/security/)

```
# .codex/config.toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"

model = "gpt-5.3-codex"
model_reasoning_effort = "medium"

[features]
multi_agent = true
collaboration_modes = true

# 让依赖下载/CI 更顺滑（可按公司安全策略关闭）
[sandbox_workspace_write]
network_access = true

[agents]
max_threads = 6
max_depth = 1

[agents.team_lead]
description = "团队Lead：先产出可执行计划与门禁，再调度架构/实现/测试/评审，最终汇总PR建议与合并checklist。"
config_file = "agents/team_lead.toml"

[agents.architect]
description = "架构师：Mermaid图 + 模块边界 + OpenAPI契约策略 + 风险点。默认只改docs/契约文件。"
config_file = "agents/architect.toml"

[agents.be_developer]
description = "后端开发：严格TDD实现（Spring Boot Maven / Python requirements）。跑测试并产出证据。"
config_file = "agents/be_developer.toml"

[agents.fe_developer]
description = "前端开发：若仓库包含前端则实现UI/联调，TS + 现有组件库，跑lint/test。"
config_file = "agents/fe_developer.toml"

[agents.tester]
description = "测试工程：补齐单测/集成/E2E，跑CI同款命令并输出证据（覆盖率/失败复现/日志）。"
config_file = "agents/tester.toml"

[agents.reviewer]
description = "资深评审：只读审查 correctness/security/perf/test gaps，给可执行建议与✅/❌结论。"
config_file = "agents/reviewer.toml"
```

* * *

### Step 2：写 AGENTS.md（协议 = Protocol）

AGENTS.md 是你们“角色 + 门禁 + 输出 schema”的核心承载面。[OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/?utm_source=chatgpt.com)

保存为：`AGENTS.md`

```
# Team Protocol for Codex (6-agent) — Java Maven + Python requirements

## 1) Canonical Commands (skills must use these exact forms)
### Java / Spring Boot (Maven)
- Unit tests: `./mvnw -B -q test`  (fallback: `mvn -B -q test`)
- Full verification: `./mvnw -B -q verify` (fallback: `mvn -B -q verify`)

### Python (requirements.txt)
- Install deps: `python -m pip install -r requirements.txt`
- Unit tests: `python -m pytest -q`
- Lint (if installed): `python -m ruff check .`
- Format check (if installed): `python -m black --check .`
- Type check (if configured): `python -m mypy .`

## 2) Output Schema (mandatory)
Every role must output the following sections (either in chat or in file):
- Goal
- Changes (files)
- Commands Run (with results)
- Evidence (paths + key symbols)
- Risks & Mitigations
- Handoff (next role + exact next step)

## 3) Gated Handoff (门禁交接)
Artifacts are required:
- Team Lead: `docs/ai/PLAN.md`
- Architect: `docs/architecture/ARCHITECTURE.md` (+ OpenAPI strategy under docs/openapi/)
- Dev: `docs/ai/DEV_NOTES.md`
- Tester: `docs/ai/TEST_REPORT.md`
- Reviewer: ✅/❌ conclusion + actionable diffs

No upstream artifact => no downstream phase.

## 4) Safety
- Do not run `git commit` / `git push` (human gate)
- Prefer offline OpenAPI generation (no curl) whenever possible
```

* * *

### Step 3：放 6 个 agent TOML（角色 = Roles）

你之前那套 6-agent TOML 我不重复全部贴“长版本”，这里只给**关键改动点**（确保 Maven / requirements 流程被 skills 驱动）。

保存路径：`.codex/agents/*.toml`

**(A) tester.toml（非常关键：要能闭环）**

*   子代理不能弹批准框，所以 tester 必须使用我们在 Rules + AGENTS.md 固定的 canonical 命令。
*   tester 建议用 `gpt-5.3-codex-spark` 提速（跑日志/汇总多）。

```
# .codex/agents/tester.toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Tester】。必须严格使用 AGENTS.md 的 canonical commands，否则可能触发规则不匹配/审批失败。

必交付：
- docs/ai/TEST_REPORT.md（按 Output Schema）
- 覆盖率目标 >80%（不可达要给原因与风险）

优先使用 skills：
- $run-ci
- $tdd-loop（补测试）
"""
```

**(B) be\_developer.toml（TDD + OpenAPI 变更触发）**

```
# .codex/agents/be_developer.toml
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"

developer_instructions = """
你是【Backend Developer】（Spring Boot Maven / Python requirements）。

硬规则：严格 TDD（red → green → refactor），每个循环都记录“命令+结果”到 docs/ai/DEV_NOTES.md。

如果变更了 API（路径/请求响应模型/鉴权）：
- 必须调用 $generate-openapi 更新 docs/openapi/openapi.json 或 openapi.yaml
"""
```

其它四个角色你可以沿用你现在的版本（Lead/Architect/FE/Reviewer），只要把“输出协议”和“文件门禁”对齐 AGENTS.md 即可。

* * *

### Step 4：落 Skills（技能 = Skills）——把 best practice 固化成可复用工作流

Skills 的“progressive disclosure”机制是 Codex 官方推荐的上下文治理方式：先读元信息，真正用到才加载 SKILL.md。[OpenAI Developers+1](https://developers.openai.com/codex/skills/?utm_source=chatgpt.com)

把下面 5 个 skill 目录创建到 `.agents/skills/` 下即可（每个目录一个 `SKILL.md`）。

* * *

3) 5 个 Skills（完整可直接复制）
----------------------

### 3.1 `tdd-loop`（TDD 红绿重构，Java+Python）

保存：`.agents/skills/tdd-loop/SKILL.md`

```
---
name: tdd-loop
description: "Red-Green-Refactor TDD loop for Spring Boot (Maven) and Python (requirements.txt)."
---

# TDD Loop (Red → Green → Refactor)

## Canonical commands (MUST use exactly)
- Java: `./mvnw -B -q test`
- Python: `python -m pytest -q`

## Workflow
1) **RED**: write the smallest failing test describing behavior.
2) **GREEN**: minimal implementation to pass.
3) **REFACTOR**: improve structure without changing behavior.

## Spring Boot (Maven) best practice
- Prefer JUnit 5
- Controller tests: `@WebMvcTest` + MockMvc (fast slice)
- Service tests: plain unit tests + mocks
- Use `./mvnw -B -q test` after each iteration.

## Python best practice
- Prefer pytest
- Use Arrange → Act → Assert
- Run `python -m pytest -q` each iteration.

## Protocol output
Update:
- Developer writes `docs/ai/DEV_NOTES.md`
- Tester writes `docs/ai/TEST_REPORT.md`
Include: files changed, command run, result, what behavior is guaranteed.
```

* * *

### 3.2 `run-ci`（本地复刻 CI：Maven / requirements）

保存：`.agents/skills/run-ci/SKILL.md`

```
---
name: run-ci
description: "Run CI-equivalent checks locally for Maven Spring Boot and Python requirements.txt."
---

# Run CI Locally

## Step 1: Read CI definition first
If `.github/workflows/*.yml` exists, follow it.
Otherwise, fall back to AGENTS.md canonical commands.

## Java (Maven)
1) `./mvnw -B -q test`
2) `./mvnw -B -q verify`  (if your repo uses integration tests / failsafe)

## Python (requirements.txt)
1) `python -m pip install -r requirements.txt`
2) `python -m pytest -q`
3) Optional if installed:
   - `python -m ruff check .`
   - `python -m black --check .`
   - `python -m mypy .`

## Output
Write `docs/ai/TEST_REPORT.md` with:
- commands + results
- failures: repro steps + short logs
- coverage if available
```

* * *

### 3.3 `generate-openapi`（OpenAPI 生成：Springdoc / Python 导入 app）

保存：`.agents/skills/generate-openapi/SKILL.md`

```
---
name: generate-openapi
description: "Generate or update docs/openapi/openapi.json (or .yaml). Spring Boot (springdoc) + Python (import app)."
---

# Generate OpenAPI

## Output target
- `docs/openapi/openapi.json` (preferred)
- or `docs/openapi/openapi.yaml`

## Spring Boot (Maven) — community standard: springdoc
- springdoc 默认 OpenAPI 端点是 `/v3/api-docs`（JSON），也可 `/v3/api-docs.yaml`。 
- 典型依赖是 `org.springdoc:springdoc-openapi-starter-webmvc-ui`（或只要 API 则用 starter-webmvc-api）。
(See springdoc docs/README.) 

### Option A (recommended): Maven plugin in a dedicated profile
Baeldung 介绍过 `springdoc-openapi-maven-plugin`，它配合 `spring-boot-maven-plugin` 在 integration-test 阶段生成 OpenAPI。  
做法：只在 `-Popenapi` profile 下执行，避免每次 test 都生成文件。

### Option B (lighter): Snapshot test using MockMvc
在测试里调用 `/v3/api-docs`，把 JSON 写入 `docs/openapi/openapi.json`，并在 CI 做 diff gate。

## Python — offline generation (FastAPI-first)
Use `python scripts/openapi/generate_openapi.py`:
- set env `OPENAPI_APP_IMPORT="module:app"` (e.g. `app.main:app`)
- script imports the app and writes openapi JSON without running a server.

## Output protocol
- Update docs/openapi/openapi.json (or yaml)
- Update docs/openapi/README.md (how to regenerate)
- Record in docs/ai/DEV_NOTES.md
```

> 上面 Springdoc 的 `/v3/api-docs` 默认值与依赖坐标可以直接从 springdoc 文档/README 看到。[OpenAPI 3 Library for spring-boot+2GitHub+2](https://springdoc.org/)  
> Baeldung 也明确提到 springdoc 的 Maven plugin 能生成 JSON/YAML，并在 integration-test 阶段运行。[Baeldung on Kotlin+1](https://www.baeldung.com/spring-rest-openapi-documentation)

* * *

### 3.4 `generate-mermaid`（架构图：组件图 + 关键序列图）

保存：`.agents/skills/generate-mermaid/SKILL.md`

```
---
name: generate-mermaid
description: "Generate Mermaid component + sequence diagrams into docs/architecture/ARCHITECTURE.md."
---

# Generate Mermaid

## Output
Update `docs/architecture/ARCHITECTURE.md`:
- Component diagram (C4-ish)
- 1–3 key sequence diagrams

## Best practice
- Diagrams must match real code boundaries (packages/modules)
- Keep each diagram small; split by bounded context
- For each diagram, add a short rationale + invariants

## Protocol
Always reference:
- module/package names
- key classes/functions
- endpoints/events involved
```

* * *

### 3.5 `pr-checklist`（PR 描述 + 合并门禁）

保存：`.agents/skills/pr-checklist/SKILL.md`

```
---
name: pr-checklist
description: "Produce PR description + merge checklist aligned with Team Protocol."
---

# PR Checklist Builder

## Output (paste into PR)
### What
### Why
### How

### Tests
- [ ] Java: `./mvnw -B -q test`
- [ ] Java: `./mvnw -B -q verify` (if applicable)
- [ ] Python: `python -m pytest -q`

### OpenAPI / Docs
- [ ] OpenAPI regenerated (if API changed)
- [ ] Mermaid/docs updated (if architecture changed)

### Risk & Rollback
- Risk:
- Rollback plan:

### Merge checklist
- [ ] All required artifacts exist (PLAN/ARCH/DEV_NOTES/TEST_REPORT)
- [ ] Reviewer conclusion is ✅
```

* * *

4) OpenAPI “可直接使用”的代码与配置（Java Maven / Python requirements）
----------------------------------------------------------

### 4.1 Java Spring Boot：springdoc 依赖（若你项目还没接）

springdoc README 给出了 Spring Boot 3 的依赖示例，例如 `springdoc-openapi-starter-webmvc-ui`。[GitHub](https://github.com/springdoc/springdoc-openapi)

在 `pom.xml` 加（版本用你们统一 BOM 或最新稳定版）：

```
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version><!-- your version --></version>
</dependency>
```

并且默认会暴露 `/v3/api-docs`。[GitHub+1](https://github.com/springdoc/springdoc-openapi)

* * *

### 4.2 Java Spring Boot：Maven profile + springdoc-openapi-maven-plugin（推荐“一步到位”）

Baeldung 描述了 springdoc 的 Maven plugin 会在 integration-test 阶段生成 OpenAPI，并且与 spring-boot-maven-plugin 配合。[Baeldung on Kotlin](https://www.baeldung.com/spring-rest-openapi-documentation)

你可以在 `pom.xml` 里加一个 profile（示例结构，参数按你们路径改）：

```
<profiles>
  <profile>
    <id>openapi</id>
    <build>
      <plugins>

        <!-- Ensure app is available during integration-test (Spring Boot plugin) -->
        <plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <executions>
            <execution>
              <id>start</id>
              <phase>pre-integration-test</phase>
              <goals><goal>start</goal></goals>
            </execution>
            <execution>
              <id>stop</id>
              <phase>post-integration-test</phase>
              <goals><goal>stop</goal></goals>
            </execution>
          </executions>
        </plugin>

        <!-- springdoc OpenAPI generator -->
        <plugin>
          <groupId>org.springdoc</groupId>
          <artifactId>springdoc-openapi-maven-plugin</artifactId>
          <version><!-- use your pinned version --></version>
          <executions>
            <execution>
              <phase>integration-test</phase>
              <goals><goal>generate</goal></goals>
              <configuration>
                <apiDocsUrl>http://localhost:8080/v3/api-docs</apiDocsUrl>
                <outputFileName>openapi.json</outputFileName>
                <outputDir>${project.basedir}/docs/openapi</outputDir>
              </configuration>
            </execution>
          </executions>
        </plugin>

      </plugins>
    </build>
  </profile>
</profiles>
```

然后开发/CI 用下面命令更新契约：

```
./mvnw -B -q -Popenapi verify
```

> `/v3/api-docs` 的默认 URL 也在 Baeldung 中提到。[Baeldung on Kotlin+1](https://www.baeldung.com/spring-rest-openapi-documentation)

* * *

### 4.3 Python：离线生成 OpenAPI（FastAPI 优先），可直接放脚本

保存为：`scripts/openapi/generate_openapi.py`

```
#!/usr/bin/env python3
import importlib
import json
import os
import sys
from pathlib import Path

OUT = Path("docs/openapi/openapi.json")

def import_object(spec: str):
    """
    spec: "module:attr"  e.g. "app.main:app"
    """
    if ":" not in spec:
        raise ValueError('OPENAPI_APP_IMPORT must be like "module:attr"')
    mod_name, attr = spec.split(":", 1)
    mod = importlib.import_module(mod_name)
    obj = getattr(mod, attr)
    return obj

def main():
    app_import = os.getenv("OPENAPI_APP_IMPORT", "").strip()
    if not app_import:
        print('ERROR: set OPENAPI_APP_IMPORT="module:app" (e.g. app.main:app)', file=sys.stderr)
        sys.exit(2)

    app = import_object(app_import)

    # FastAPI-style: app.openapi() -> dict
    if hasattr(app, "openapi") and callable(getattr(app, "openapi")):
        schema = app.openapi()
    else:
        print("ERROR: imported object has no callable .openapi(). "
              "If you are not using FastAPI, adapt this script to your framework.",
              file=sys.stderr)
        sys.exit(3)

    OUT.parent.mkdir(parents=True, exist_ok=True)
    OUT.write_text(json.dumps(schema, indent=2, ensure_ascii=False) + "\n", encoding="utf-8")
    print(f"Wrote OpenAPI to {OUT}")

if __name__ == "__main__":
    main()
```

使用方式（开发者/CI 都一致）：

```
export OPENAPI_APP_IMPORT="app.main:app"   # 你们实际入口
python scripts/openapi/generate_openapi.py
```

* * *

5) CI 工作流（可选，但我建议你们直接加上）
------------------------

你提到“运行 CI best practice”，除了 `$run-ci` 技能，本质上还是要把 CI 固化到 `.github/workflows`，这样 agent 只要“读 workflow → 复刻命令”即可。

### 5.1 Java Maven GitHub Actions（含缓存）

`actions/setup-java` README 给了 `actions/setup-java@v5`、`actions/checkout@v6`，并支持 `cache: maven`。[GitHub+1](https://github.com/actions/setup-java)

```
# .github/workflows/ci-java.yml
name: ci-java

on:
  push:
    paths:
      - "**/*.java"
      - "pom.xml"
      - ".github/workflows/ci-java.yml"
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Unit tests
        run: ./mvnw -B -q test

      - name: Verify
        run: ./mvnw -B -q verify
```

（如果你启用 `-Popenapi`，再加一段生成并 diff 检查即可。）

* * *

### 5.2 Python requirements GitHub Actions（含 pip 缓存）

`actions/setup-python` README 给了 `actions/setup-python@v6`、`actions/checkout@v6`，并支持 `cache: pip`（默认会用 requirements.txt 的 hash）。[GitHub+1](https://github.com/actions/setup-python)

```
# .github/workflows/ci-python.yml
name: ci-python

on:
  push:
    paths:
      - "**/*.py"
      - "requirements.txt"
      - ".github/workflows/ci-python.yml"
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-python@v6
        with:
          python-version: "3.13"
          cache: pip

      - name: Install deps
        run: python -m pip install -r requirements.txt

      - name: Unit tests
        run: python -m pytest -q
```

* * *

6) 最后一步：把 rules 选成 default.rules（每个 repo 只用一个）
----------------------------------------------

Codex 启动会扫描 team config 下的 `rules/`；最不容易踩坑的做法是：你把当前 repo 真正要用的那套复制为 `default.rules`。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

*   Java repo：

```
cp .codex/rules/java_springboot_maven.rules .codex/rules/default.rules
```

*   Python repo：

```
cp .codex/rules/python_requirements.rules .codex/rules/default.rules
```

然后**重启 Codex**（Rules / multi-agent 开关都需要重启加载）。[OpenAI Developers+1](https://developers.openai.com/codex/rules/)

* * *

7) 你现在怎么用（推荐的“标准指令”）
--------------------

在 Codex 里直接发（中文也 OK）：

> 用我们的 6-agent team 做 XXX。
> 
> 1.  Team Lead 先产出 `docs/ai/PLAN.md`（门禁+任务分解）
> 2.  Architect 产出 `docs/architecture/ARCHITECTURE.md` + OpenAPI 策略
> 3.  Backend Developer 严格 `$tdd-loop` 实现
> 4.  Tester 用 `$run-ci` 跑全套并写 `docs/ai/TEST_REPORT.md`
> 5.  如果 API 变更，执行 `$generate-openapi` 更新 `docs/openapi/openapi.json`
> 6.  Reviewer 给 ✅/❌ + diff 建议
> 7.  Team Lead 用 `$pr-checklist` 汇总 PR 描述与合并 checklist
>     

* * *

如果你希望我把上面这套再“更一步到位”到**一键脚本**（执行一次就把所有文件生成出来，并按仓库检测 `pom.xml` / `requirements.txt` 自动选择 default.rules），我也可以把脚本完整贴出来（你复制执行即可）。当前我先给的是“每个文件可直接复制粘贴”的最稳定版本，避免脚本在不同 OS 上踩坑。

## Prompt:
是的，我可以给你一个更强化的“原生 4 角色全流程模板”。
这个模板专门针对 Codex CLI 未开启 multi_agent 配置 的场景优化，利用内置的 default / explorer / worker / monitor 这四个原生角色，来尽可能接近（甚至在某些方面超越）之前自定义 6-Agent Team 的结构化、纪律性流程。
原生 4 角色能力快速回顾（基于当前 Codex 行为）

default：主代理，通用思考、决策、汇总结果（相当于“Team Lead”的统筹角色）
explorer：只读探索代码库、文档、依赖、架构（类似 Architect + 部分 Reviewer 的分析角色）
worker：实际写代码、改文件、跑命令（相当于 BE/FE Developer 的执行角色）
monitor：擅长长任务等待、轮询状态、进度跟踪、汇总子结果（强协调 + Tester/Reviewer 的把关角色）

原生模式下，Codex 会自动决定何时 spawn 子代理（通常 1–4 个并发），你可以通过明确指令引导它使用特定角色。
强化版全流程 Prompt（直接复制粘贴到 Codex CLI 使用）
这个 Prompt 已经把之前 6-Agent 的 gated handoff、traceability、风险评估等最佳实践“注入”到原生 4 角色里，让它表现得更像一个有纪律的团队。
使用你的原生 agent team（default + explorer + worker + monitor）来执行 FPMS 项目的最终增强工作。

角色严格映射与行为规则如下（请严格遵守，不得随意切换或省略）：

- default（主代理） → 扮演 Team Lead / Orchestrator
  - 负责整体规划、任务分解、风险评估、最终汇总
  - 必须先输出完整计划（步骤、依赖、预计时间）
  - 所有输出以 【default / Team Lead】 开头标注
  - 最后合成所有子代理结果，给出 PR-ready 建议 + 风险 checklist

- explorer → 扮演 Architect + 初步 Reviewer
  - 只读模式：分析代码、docs/FPMS SPEC 2.0.md、审查报告
  - 先完整提取 Partially Implemented 清单
  - 输出架构影响分析、缺失点细节、建议的文件/模块变更
  - 输出以 【explorer】 开头

- worker（可 spawn 多个） → 扮演 BE Developer + FE Developer
  - 实际修改代码、写 patch、新增测试
  - 前后端并行处理（如果任务允许）
  - 必须遵循 TDD、先写测试、遵守项目现有栈（FastAPI + Vue）
  - 每步改动后说明文件路径 + diff 意图
  - 输出以 【worker - BE/FE】 开头

- monitor → 扮演 Tester + Final Reviewer + Progress Tracker
  - 监控子任务进度、等待完成
  - 运行测试、验证修复是否关闭 Partially Implemented 项
  - 最终安全/性能/规范审查
  - 输出以 【monitor】 开头，包含覆盖率/失败用例/最终通过结论

执行 Scope（严格限制）：
- 只针对 docs/FPMS_Code_Review_Report_SPEC_2.0_20260227.md 中 Functional Compliance Matrix 的 Partially Implemented 条目进行修复
- 完全排除任何 document generation 相关需求
- 其他条目（Fully / Missing / N/A）一律不触碰

完整流程（必须按顺序 gated handoff）：
1. default 先读取两个文档，输出 Partially Implemented 完整清单 + 整体 Enhancement Plan（包括依赖图、风险）
2. explorer 分析每个 Partially Implemented 项，给出详细设计建议（前后端）
3. default 根据 explorer 输出，分配具体任务给 worker（可并行 spawn 多个 worker）
4. worker 执行代码变更、写测试、提交 patch
5. monitor 等待所有 worker 完成 → 运行验证（测试、lint、安全检查）→ 给出 review 结论
6. default 收集所有结果 → 生成最终报告到 docs/FPMS_Final_Enhancement_Plan_Native_20260308.md
   报告结构必须包含：
   - Executive Summary（覆盖条目数、总工作量、符合度提升）
   - Partially Implemented 修复矩阵（表格：User Story | 原状态 | 修复方式 | 验证结果）
   - Detailed Changes（文件路径 + 变更描述 + diff 意图）
   - Validation Results（测试覆盖、通过情况）
   - Ready-for-Merge Checklist
   - Next Steps

现在立即开始：
第一步：default 输出 Partially Implemented 清单和 Enhancement Plan。
全程使用角色标注，每步完成后明确说“等待下一角色”或“已完成，交给 XXX”。
完成后告诉我“✅ 原生团队增强计划与报告已生成在 docs/FPMS_Final_Enhancement_Plan_Native_20260308.md”为什么这个模板“更强”？

强制角色标注：输出像团队聊天一样清晰（【default】、【explorer】等），你一眼就能追踪谁在干嘛。
注入 gated handoff：即使原生不强制顺序，也通过 Prompt 强引导按步骤走。
保留最佳实践：TDD、traceability matrix、风险 checklist、PR-ready 输出都保留。
零配置友好：完全不依赖 multi_agent = true 或自定义 toml。
可扩展：如果你后期想加 MCP（Playwright 测试）或 /review，可以在 Prompt 里加一句“monitor 阶段使用 Playwright 验证 UI”。

直接把上面 Prompt 扔进 Codex，它就会开始用原生 4 角色“组队”给你干活。
如果你跑完后觉得某个角色表现不够理想（比如 monitor 没认真等），可以再加一句“monitor 必须每 30s 报告一次进度”来微调。请审核这个方案，给出你的版本

## Response:
我先对照 Codex 官方关于原生角色、multi-agent 开关、子代理权限和 plan/协作模式的实际边界审一遍，再把你这套模板改成更稳、更贴近真实运行时的版本。

另外还要补一句现实边界：你这版里关于 worker 并行、monitor 轮询、`/agent` 切线程这些描述，只有真正开启 multi-agent 时才更可信。若保持 zero-config，我会把它改成“优先按这些工位组织输出；运行时若支持再并行”，避免过度承诺。

Thought for 12m 44s

我审过了。结论很明确：

这个方案**作为“提示词脚手架”是好的**，但它把一件事说得过头了：\*\*在未开启 `multi_agent` 时，不应该把 `default / explorer / worker / monitor` 当成已经会被真实 spawn 的 4 个子代理。\*\*官方目前把 `multi_agent` 标成 experimental，默认值也是 `false`；而这 4 个名字出现在官方的 Multi-agents 文档里，属于那套多代理机制的内置角色。更稳的做法是：**未开 `multi_agent` 时，把这 4 个名字当“阶段标签/职责标签”；把真实的计划和最终审查分别交给 `/plan` 和 `/review`。**`/plan` 所属的 collaboration modes 是 stable，`/review` 会启动一个专门的 reviewer，而且不会改你的 working tree。[OpenAI Developers+4OpenAI Developers+4OpenAI Developers+4](https://developers.openai.com/codex/config-basic/)

所以，我对你这版模板的判断是：

它的优点在于角色标注清楚、scope 锁得严、gated handoff 明确、最后产物也有 traceability。问题在于它把“组织结构”写得很好，但把“运行时能力”假定得太强。官方对 Codex 的 prompting 建议是：复杂任务要拆成更小、更聚焦的步骤，并明确验证方法与完成标准；对长流程任务，GPT-5.4 也更稳定地遵循“显式输出合同、工具预期、完成条件”。所以这类工作，**三段式（Plan → Execute → Review）会比一条巨型 prompt 更可靠。**[OpenAI Developers+1](https://developers.openai.com/codex/prompting/)

我建议你改 6 个点：

1.  把“spawn 原生 4 角色”改成“**如果当前运行时支持多代理则 spawn；否则仅作为输出标签**”。这能避免 Codex 在单线程会话里“口头扮演成已开多代理”。[OpenAI Developers+1](https://developers.openai.com/codex/config-basic/)
2.  不要把计划、实现、验证、最终报告塞进一个回合。官方更推荐**先 plan，再执行，再 review**。`/plan` 是现成能力。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/config-basic/?utm_source=chatgpt.com)
3.  不要让 `monitor` 同时承担“测试工程师 + 最终 reviewer + 进度协调员”三种职责。**最终审查直接交给 `/review` 更稳**，因为这是 CLI 的内建专门工作流。[OpenAI Developers](https://developers.openai.com/codex/cli/features/)
4.  把“document generation 相关需求完全排除”改成更精确的句子：  
    **排除产品功能范围内的 document generation 需求；允许生成本次修复报告文件。**  
    否则你后面要求落盘 `docs/FPMS_Final_Enhancement_Plan_Native_20260308.md`，前后是冲突的。
5.  把“FastAPI + Vue”从硬编码改成\*\*“遵循仓库实际技术栈”\*\*。如果你这版模板要跨 Java / Python 复用，就不能把技术栈写死。
6.  把“预计时间”改成**复杂度 / 风险 / 依赖 / 并行性评估**。时间估计对 Codex 没什么执行价值，反而占上下文。

下面是我的版本。这个版本的原则是：

*   **零配置可用**
*   **对未开 `multi_agent` 完全诚实**
*   **如果以后开启 `multi_agent`，同一套提示词可以自动升级**
*   **稳定能力优先：`/plan` + 正常执行 + `/review`**
*   **四个原生角色继续保留，但默认只当职责标签**

* * *

我的版本：推荐用法（3 回合）
---------------

### 第 1 回合：先用 `/plan`

把下面整段贴进 Codex CLI 的 `/plan`：

```
使用“原生 4 角色工作流”来规划 FPMS 项目的最终增强工作。

运行时规则（必须遵守）：
- 如果当前运行时未启用 multi_agent，请把 default / explorer / worker / monitor 仅作为输出标签与职责标签，不要声称已经实际 spawn 子代理。
- 只有在当前运行时明确支持多代理时，才允许真实 spawn 对应角色；否则用单线程 gated handoff 模拟同样流程。
- 本阶段只允许读取代码和文档，不允许修改文件、不允许生成 patch。

角色映射：
- default → Team Lead / Orchestrator
- explorer → Architect / Evidence Gatherer
- worker → Implementer（本阶段不执行）
- monitor → Validator（本阶段不执行）

执行 Scope（严格限制）：
- 只处理 docs/FPMS_Code_Review_Report_SPEC_2.0_20260227.md 中 Functional Compliance Matrix 的 “Partially Implemented” 条目
- 完全排除产品功能范围内的 document generation 需求
- 允许本任务最终生成审计/修复报告文件
- Fully / Missing / N/A 条目不得作为修复目标；只有在修复 Partially Implemented 时必须触及共享类型、共享校验或共享测试夹具时，才允许最小必要修改，并必须解释原因

本阶段输出合同（必须按这个结构输出）：

【default / Team Lead】
1. 任务边界摘要
2. 依赖与风险概览
3. 建议的执行批次（Batch 1 / 2 / 3 ...）
4. 每个批次的进入条件与完成条件

【explorer】
1. Partially Implemented 完整清单
2. 每一项的证据来源（文档章节/表格/代码位置）
3. 受影响模块与建议修改文件
4. 不应触碰的模块/文件
5. 测试建议（单测/集成/E2E）

【default / Team Lead】
输出最终执行计划，必须包含：
- Fix Matrix：Item | Current Gap | Proposed Fix | Files | Tests | Risk
- 批次顺序
- 回滚点
- Done Definition：
  - 仅修复 Partially Implemented 项
  - 每项都有验证证据
  - 不引入对 Fully / Missing / N/A 的意外改动
  - 最终需生成 docs/FPMS_Final_Enhancement_Plan_Native_20260308.md

最后停在计划阶段，不要执行代码修改。
```

* * *

### 第 2 回合：执行计划

等计划出来后，再发这一段：

```
按刚才批准的计划进入执行阶段。

继续使用“原生 4 角色工作流”，并遵守以下运行时规则：
- 未启用 multi_agent 时，default / explorer / worker / monitor 只作为输出标签，不要假装存在真实并发子代理。
- 如果当前运行时支持多代理，可以按批次并行，但不得让两个执行单元修改同一文件。

执行规则（必须遵守）：
1. 【explorer】先重新确认本批次的目标文件、共享依赖和禁改区域。
2. 【worker】按批次小步修改，遵循 TDD：
   - 先写最小 failing test
   - 再做最小实现
   - 再回归验证
3. 每轮修改后必须输出：
   - 文件路径
   - 变更意图
   - 对应修复的是哪个 Partially Implemented 项
   - 跑了什么验证
4. 【monitor】只负责验证，不负责设计新方案：
   - 运行最小必要测试 / lint / 回归检查
   - 检查是否误改 Fully / Missing / N/A 范围
   - 标注未解决风险
5. 每完成一个批次，先输出阶段总结，再进入下一批次。
6. 最终把汇总结果写入：
   docs/FPMS_Final_Enhancement_Plan_Native_20260308.md

最终报告必须包含：
- Executive Summary
- Partially Implemented 修复矩阵
- Detailed Changes（文件路径 + 变更描述 + diff 意图）
- Validation Results
- Ready-for-Merge Checklist
- Next Steps

输出要求：
- 所有分段都用角色标注：
  【default / Team Lead】
  【explorer】
  【worker】
  【monitor】
- 不要宣称“全部修复完成”，除非逐项给出验证结果。
- 如果发现某一项需要超出范围的大改，立即停下并在【default / Team Lead】中说明原因与替代方案。
```

* * *

### 第 3 回合：最后用 `/review`

实现结束后，不要再让“monitor 兼 final reviewer”，直接跑 CLI 内建的 `/review`。用这个指令最合适：

```
/review
Focus on:
1. regressions outside the Partially Implemented scope
2. missing tests or weak assertions
3. auth/permission boundary mistakes
4. API contract drift
5. performance hotspots or accidental N+1 / extra roundtrips
6. unsafe shared-model or migration changes

Only report prioritized actionable findings.
```

官方的 `/review` 就是为这个场景做的：它会启动 dedicated reviewer，读你选定的 diff，并返回优先级明确、可执行的问题，而且不会动 working tree。[OpenAI Developers](https://developers.openai.com/codex/cli/features/)

* * *

如果你坚持“单贴运行”，这是我的一体化版本
---------------------

它比你原版更准确，因为它不会误导 Codex 以为“没开 multi\_agent 也一定有真实 4 agent”。

```
使用“原生 4 角色工作流”来执行 FPMS 项目的最终增强工作。

重要运行时规则（必须遵守）：
- 如果当前运行时未启用 multi_agent，请把 default / explorer / worker / monitor 仅作为输出标签与职责标签，不要声称已经实际 spawn 子代理。
- 只有在当前运行时明确支持多代理时，才允许真实 spawn 对应角色；否则按单线程 gated handoff 顺序执行。
- 若未启用多代理，不要伪造并发、不要伪造等待中的子线程状态。

角色映射：
- default → Team Lead / Orchestrator
- explorer → Architect / Evidence Gatherer
- worker → Implementer
- monitor → Validator

严格 Scope：
- 只修复 docs/FPMS_Code_Review_Report_SPEC_2.0_20260227.md 中 Functional Compliance Matrix 的 Partially Implemented 条目
- 排除产品功能范围内的 document generation 需求
- 允许生成本次修复报告 docs/FPMS_Final_Enhancement_Plan_Native_20260308.md
- Fully / Missing / N/A 不作为修复目标；若必须触及共享代码，必须解释为“最小必要变更”

强制流程（不得跳步）：
1. 【default / Team Lead】先输出：
   - 任务边界
   - 风险与依赖
   - 批次化执行计划
2. 【explorer】输出：
   - Partially Implemented 完整清单
   - 每项证据
   - 建议修改文件
   - 禁改区域
   - 测试建议
3. 【default / Team Lead】确认批次与进入条件
4. 【worker】按批次执行小步改动，遵循 TDD：
   - 先最小 failing test
   - 再最小实现
   - 再验证
5. 【monitor】每批次后做验证：
   - 测试 / lint / 回归
   - 检查是否误改范围
   - 输出未解决风险
6. 【default / Team Lead】最终汇总并写入：
   docs/FPMS_Final_Enhancement_Plan_Native_20260308.md

输出合同（所有角色都必须遵守）：
- Goal
- Changes
- Commands Run
- Evidence
- Risks & Mitigations
- Handoff

最终报告必须包含：
- Executive Summary
- Partially Implemented 修复矩阵
- Detailed Changes
- Validation Results
- Ready-for-Merge Checklist
- Next Steps

现在开始第一步，只做计划与证据收集；不要跳到实现，除非计划阶段已经完整结束。
```

* * *

为什么我这版更稳
--------

第一，它**符合官方边界**：不把未开启的 `multi_agent` 说成已激活；真要用多代理，就显式依赖该 feature。[OpenAI Developers+1](https://developers.openai.com/codex/config-basic/)

第二，它把**稳定能力**放前面：`/plan` 属于 stable 的 collaboration modes，而不是把所有控制都押在 experimental multi-agent 上。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/config-basic/)

第三，它把**最终审查**交给 `/review`，这是官方现成的 dedicated reviewer 工作流，比让 `monitor` 在一个超长 prompt 里兼职 reviewer 更稳定。[OpenAI Developers](https://developers.openai.com/codex/cli/features/)

第四，它更符合官方的 prompting 建议：**任务拆小、验证优先、完成条件明确**。这类长链路任务，三回合通常比一条 mega prompt 更可靠。[OpenAI Developers+1](https://developers.openai.com/codex/prompting/)

第五，如果你将来真的开了多代理，这套文案**几乎不用重写**；只是把“角色标签”自动升级为“真实角色线程”。而且官方也支持多个线程并行，只是明确提醒不要让两个线程改同一批文件。[OpenAI Developers+1](https://developers.openai.com/codex/prompting/)

如果你愿意，我下一条可以直接给你一个 **“FPMS 专用最终版 prompt”**：把上面模板进一步压缩成一段可直接粘贴进 Codex CLI 的生产版文案。

## Prompt:
AGENTS.md

File

由于agents.md  存在，codex只能执行atomic task， one bye one, 现在请修改，目的 1. 单个task 还是遵循 atomic task的原则，2.允许muti agents，3 其他约束不变

## Response:
我先看一下你现在的 `AGENTS.md` 约束点，尤其是“atomic task”具体怎么写的，然后我会把上一版 prompt 改成兼容它的版本。

我已经确认关键改法了：不动其他约束，只把规则从“单次 execution 只能一个 atomic task”改成“每个 agent 一次只处理一个明确 task file path”。接下来我会直接给你可替换的 prompt/AGENTS.md 最小修改版。

Thought for 12m 5s

可以，核心不是把 atomic task 取消，而是把它从“**整次 session 只能一个 task**”改成“**每个 agent 只能一个 task，但一个 session 允许多个 agent 并行，各自负责一个 atomic task**”。你当前的 `AGENTS.md` 确实把 atomic 绑定到了整次 execution；而 `AGENTS.md` 又会在 Codex 开始工作前自动注入并被严格遵循，所以它会强烈压制 multi-agent 编排。官方也明确说，multi-agent 需要显式开启，且并发上限由 `[agents].max_threads` 控制；内置角色就是 `default / worker / explorer / monitor`。[OpenAI Developers+3OpenAI Developers+3OpenAI Developers+3](https://developers.openai.com/codex/guides/agents-md/)

你现在的 `config.toml` 已经有 `multi_agent = true`，所以**主要改 `AGENTS.md`**。不过你当前 `max_threads` / `max_depth` 只是注释，还没有真正生效；要让并发行为更稳定，建议补一个真实的 `[agents]` 表。官方文档里这两个键都在 `[agents]` 下，`max_threads` 控制并发线程数，`max_depth` 控制嵌套深度。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

下面给你的是**最小改动版**：只改 `Section 1`、`Section 11`、`Section 13`，其余约束全部保持不变。

* * *

1) `config.toml` 最小补丁
---------------------

把这段加到你的 `config.toml` 里。你已有 `multi_agent = true`，这段是补并发控制和内置角色说明。

```
[agents]
max_threads = 6
max_depth = 1

[agents.default]
description = "主线程编排者。负责批次规划、任务分配、冲突消解、最终验收。"

[agents.explorer]
description = "只读分析代理。负责规范分析、契约冻结、影响面与风险识别，不改代码。"

[agents.worker]
description = "执行代理。一次只拥有一个 atomic task file，做最小必要改动并运行任务级验证。"

[agents.monitor]
description = "监控与验证代理。负责等待/轮询、收集 evidence、运行 task gate、汇总 per-task 状态。"
```

* * *

2) 直接替换 `AGENTS.md` 的 Section 1
-------------------------------

把你现在的 **Section 1** 整段替换成下面这一版：

```
## 1) Atomic Task Discipline

- The atomic unit remains EXACTLY ONE task file path.
- One atomic task = one endpoint OR one service OR one doc change.
- Atomicity is enforced at the AGENT level, not at the whole-session level.
- In a multi-agent run, the lead/main thread MAY coordinate multiple atomic task files in one session, but each spawned agent MUST own exactly one explicit task file path.
- No single agent may implement more than one task file in the same execution.
- Do NOT implement multiple endpoints inside one atomic task unless the task file explicitly defines them as a single unit.
- Do NOT proactively refactor unrelated code.
- When asked “which task to implement”, request/confirm either:
  - one exact task file path, OR
  - an explicit batch manifest listing one exact task file path per agent.
- If no explicit batch manifest exists, default to ONE task file only.
- Two agents MUST NOT concurrently edit the same task file or the same shared ownership file.
- Shared ownership files (for example: router wiring, shared schemas, permission registries, common exports) require serialized ownership.
```

### 这段改动的意义

你原文里最卡的是这句：

```
Implement EXACTLY ONE task file per execution.
```

它把 atomic 绑定到了**整次 execution**。改完后，atomic 仍然存在，但被绑定到**每个 agent**，这样主线程就可以一次编排多个 task file，只要做到“一 agent 一 task”。

* * *

3) 直接替换 `AGENTS.md` 的 Section 11
--------------------------------

把 **Section 11** 替换成下面这一版：

```
## 11) Required Agent Output (End of each execution)

Every agent MUST explicitly state:
- Which task/runbook was executed
- Which role executed it
- Which file(s) were modified
- Verification commands + expected status codes
- Evidence path under `artifacts/<TASK-ID>/**`
- Final per-task status: PASS / FAIL / BLOCKED

If the main thread coordinates multiple agents in one session, it MUST additionally state:
- Agent-to-task mapping
- Execution wave / batch order
- Any serialized shared-file decisions
- Per-task completion summary
```

### 为什么要改这里

不改这一段的话，即使允许 multi-agent，最后汇总仍然是“整次 run 一个结论”，不利于 task-level 审计。你现在的 evidence 体系本来就是按 `TASK-ID` 组织的，所以输出也要切换成 **per-task**。

* * *

4) 直接替换 `AGENTS.md` 的 Section 13
--------------------------------

把 **Section 13** 整段替换成下面这一版：

```
## 13) Codex Multi-Agent Atomic Execution (MVP1 Execution)

This repository supports multi-agent execution with strict atomic-task discipline.

Team behavior is enforced by this file + task files + evidence gates.

### 13.1 Role Mapping (Supported)

| Business Role | Codex Executor | Responsibility |
|---|---|---|
| Team Lead / Project Manager | Main thread / `default` | Plan, assign task IDs, group tasks into safe waves, coordinate dependencies, final acceptance |
| Architect / Designer | `explorer` agent | Spec/contract analysis, module boundary checks, design decisions, impact analysis |
| Backend Developer | `worker` agent | Implement exactly one backend atomic task file per agent |
| Frontend Developer | `worker` agent | Implement exactly one frontend atomic task file per agent |
| Tester / Progress Monitor | `monitor` agent | Wait/poll long-running work, run task gates, collect evidence, report pass/fail/blockers |
| Reviewer | `explorer` agent or main thread | Independent scope review, acceptance validation, risk notes |

Notes:
- Built-in roles may be used directly (`default`, `explorer`, `worker`, `monitor`).
- One session MAY contain multiple atomic tasks only when each task has an explicit task file path and is assigned to a distinct agent or serialized wave.
- Acceptance remains per-task, not per-session.

### 13.2 Hard Constraints (Iron Rules + Quality Gate)

- Preserve all rules in Sections 1-12; this section is additive, not a replacement.
- Each spawned agent MUST target exactly one atomic task file path.
- Lead MAY coordinate multiple task file paths in one run only through an explicit batch manifest.
- No single agent may own more than one task file concurrently.
- No two agents may concurrently edit the same ownership file or shared support file.
- Shared ownership files that require serialization include:
  - `backend/app/api/router.py`
  - shared schema files
  - permission registry / permission constants
  - common exports / index files
  - any file explicitly listed in more than one task allowlist
- If two tasks require the same shared file, split out a dedicated follow-up task OR serialize them into different waves.
- Backend/Frontend dev agents MUST NOT modify each other's ownership files unless task explicitly allows.
- Reviewer cannot mark a task complete unless required gates pass and evidence exists.

### 13.3 Coordination Protocol (Wave-based, Multi-Agent Safe)

1) Lead defines an explicit batch manifest:
   - one row = one task file path
   - owner role
   - allowed files
   - required verification
   - dependency notes

2) Architect (`explorer`) freezes API contract / acceptance checklist for each task or for the batch if the contract is shared and identical.

3) Lead groups only NON-CONFLICTING tasks into the same execution wave.

4) Spawn one execution agent per task in the current wave:
   - backend task → `worker`
   - frontend task → `worker`
   - validation / long-wait / evidence tracking → `monitor`

5) Monitor tracks completion and records evidence per task under:
   - `artifacts/<TASK-ID>/results.jsonl`
   - `artifacts/<TASK-ID>/summary.md`
   - `artifacts/<TASK-ID>/git/diff.patch`

6) Reviewer validates each completed task independently:
   - scope compliance
   - gate pass
   - no regression
   - no cross-task contamination

7) Lead closes tasks individually.
   - A session may complete multiple tasks.
   - Each task must independently pass before being marked done.

Mandatory handoff artifacts per task:
- `artifacts/<TASK-ID>/results.jsonl`
- `artifacts/<TASK-ID>/summary.md`
- `artifacts/<TASK-ID>/git/diff.patch`

### 13.4 Required Verification by Role

- Backend task:
  - `ruff check --fix .`
  - `ruff format .`
  - `ruff check .`
  - `pytest -q` (or task-defined targeted tests)

- Frontend task:
  - `npm run lint`
  - `npm run typecheck`
  - `npm run build`

- Tester / Monitor:
  - `./scripts/task_validate.sh <TASK-ID>`
  - targeted regression checks
  - release gate only when explicitly requested for batch close

- Reviewer:
  - Verify status-code semantics, permission enforcement, envelope consistency,
    SQLite compatibility, task allowlist compliance, and no cross-task/shared-file regression.
```

* * *

5) 保持不变的部分
----------

下面这些不用动：

*   Section 2–10
*   Section 12
*   Section 14

也就是说，你现有的：

*   Phase 约束
*   Router wiring 约束
*   Permission 写法
*   HTTP status code 语义
*   Ruff / lint discipline
*   SQLite PoC 兼容要求
*   Evidence / Gate 体系
*   前端中文 UI 铁律

全部继续生效。

* * *

6) 改完之后，Codex 的行为会变成什么
----------------------

改完后，Codex 在这个 repo 下会遵循下面这个模型：

*   **主线程 / default**：可以一次接收一个 batch manifest
*   **每个 worker**：仍然只做一个 atomic task
*   **monitor**：只负责等待 / gate / evidence / per-task 状态
*   **explorer**：先做 contract freeze / risk analysis，再做 reviewer

也就是说，你的目标就变成了：

> **single task 仍然 atomic；multi-agent 在 session 级别并行；验收仍然是 per-task。**

这与官方 multi-agent 机制是对齐的：Codex 支持并行 spawn specialized agents，`default / worker / explorer / monitor` 是内置角色；`max_threads` 控制并发上限；`AGENTS.md` 会在工作前被自动注入并强烈影响行为。[OpenAI Developers+4OpenAI Developers+4OpenAI Developers+4](https://developers.openai.com/codex/multi-agent/)

* * *

7) 配套调用方式也要顺手改一下
----------------

以后不要再用这种口径：

> implement exactly one task in this run

而改成这种：

```
Use multi-agent execution under AGENTS.md.

Create a safe batch manifest for the following task files.
Each spawned agent must own exactly one atomic task file.
Group only non-conflicting tasks into the same wave.
Serialize any task that touches shared ownership files.
Collect evidence per TASK-ID and report PASS/FAIL/BLOCKED per task.
```

这样它就不会再被“整次 run 只能一个 task”的旧语义锁死。

如果你要，我下一条可以直接把**你这份 `/mnt/data/AGENTS.md` 完整重写成最终版全文**，你可以整份覆盖，不需要自己手工替换段落。



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)