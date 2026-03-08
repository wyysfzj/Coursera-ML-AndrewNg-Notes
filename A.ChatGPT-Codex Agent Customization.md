# Codex Agent Customization

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 3/6/2026 9:29:16  
**Updated:** 3/6/2026 20:53:26  
**Exported:** 3/9/2026 0:18:35  
**Link:** [https://chatgpt.com/c/69aa2dec-8040-8321-b760-00da9714f013](https://chatgpt.com/c/69aa2dec-8040-8321-b760-00da9714f013)  

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
我会先核对 Codex 原生与自定义 agent 的最新官方能力边界，再从职责、协作机制、产物、治理与风险几个维度系统对比你的 6-agent team，最后给出可落地的增强版改造方案。

我还确认到一个落地层面的关键点：会顺手审计你这套 6 个 TOML 的字段与当前官方配置入口是否真的兼容，而不只停留在方法论对比。

Thought for 19m 53s

基于官方最新文档和你贴的 6 个 TOML，我的判断是：**Native 4 和自定义 6-agent 不是替代关系，而是两层不同抽象。** 官方 4 个角色更像运行时 primitive：`default` 负责通用回退，`worker` 负责实现，`explorer` 负责读多写少的探索，`monitor` 负责长命令/轮询监控；你这套 6-agent 更像 SDLC 交付组织模型，强调计划、架构、实现、测试、评审和最终合成。官方也明确把 `AGENTS.md`、Skills、MCP、Multi-agents 视为互补层，不是互相替代。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

截至 **2026-03-06**，Codex 的 multi-agent 仍是 **experimental**，需要显式开启；内建角色固定就是 `default / worker / explorer / monitor`，自定义角色通过 `config.toml` 的 `[agents]` 定义，可放用户级或项目级配置中。官方还特别强调：好的角色定义应该**窄、强约束、边界清晰**，并且并行更适合 exploration、tests、triage、summarization 这类**读多写少**任务；多个可写 agent 并行改代码会显著增加冲突和协调成本。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/multi-agent/)

另外，Codex app、CLI、IDE extension 共享 agent 配置，但官方也提醒：**实验特性通常先落在 CLI**，所以“配置共用”不等于“多 agent 体验各端完全等价”。[OpenAI Developers+1](https://developers.openai.com/codex/app/settings/)

模型选择上，你现在把核心角色固定在 `gpt-5.3-codex` / `gpt-5.3-codex-spark`，已经不是官方当前最推荐的默认值了。官方现在建议大多数 Codex 任务优先用 **`gpt-5.4`**；`gpt-5.3-codex-spark` 更适合极速、探索、摘要这类读多写少任务，而且目前仍是面向 ChatGPT Pro 的 research preview。[OpenAI Developers+3OpenAI Developers+3OpenAI Developers+3](https://developers.openai.com/codex/cli/features/)

Native 4 vs 你这套 6-agent
-----------------------

**1\. 角色切分逻辑不同。**  
官方内建角色是按“执行形态”切的：实现、探索、监控、通用回退。官方示例里的自定义团队也更偏“任务表面 / 工具边界”，比如 PR review team 用的是 `explorer + reviewer + docs_researcher`，前端集成排障 team 用的是 `explorer + browser_debugger + worker`。这说明官方更推崇**按任务和工具边界切角色**，而不是按人类组织架构头衔切角色。你现在的 6-agent 则明显更像 org chart：Lead、Architect、BE、FE、Tester、Reviewer。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

**2\. Native 4 更偏底层原语，你的 6-agent 更偏流程治理。**  
Native 4 的价值在于 spawn、wait、explore、monitor 这些底层协作能力；你的 6-agent 则把“先计划、再架构、再实现、再测试、再评审、再合成”这套流程写死了。对于复杂特性、跨层改动、需要审计留痕的团队，这是优点；对于小 bug、快速修补、单文件改动，这会显著增加 orchestration 开销。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

**3\. 官方已经内建了一部分你在重复实现的能力。**  
现在 CLI 已有 `/plan`，能先出可审阅计划再动手；也有 `/review`，会启动专门 reviewer 对 diff 做本地代码评审；还有 `monitor` 角色专门盯长任务和轮询。也就是说，你的 `team_lead`、`reviewer`、长任务等待逻辑，并不是从零开始，而是可以站在原生能力之上做“治理层叠加”。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/cli/slash-commands/)

**4\. 你这套 6-agent 的长板是“可审计交付”，短板是“角色过宽”。**  
官方明确说最好的 role definition 要 narrow and opinionated。你现在的每个角色都很能干，但也都很宽：Architect 一次性要图、模块、API、模型、技术选型、Schema；Tester 要单测/集成/E2E/性能/截图/日志/覆盖率；Reviewer 要安全、性能、代码味道、风格、测试、diff 建议。这样会导致 prompt 很强，但边界不清。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

对你这 6 个角色的逐个判断
--------------

**team\_lead**  
这是你这套设计里最有价值的角色，因为它把“任务分解、门控交接、最终合成”显式化了。但它现在太像“项目经理 + 架构审查 + release manager”三合一。官方已经提供了 `/plan` 和长期任务里的 Plan/Implement/Documentation 思路，所以 team\_lead 更适合只做三件事：**出计划、控门、做最终 synthesis**，不要再兼任过多技术判断。[OpenAI Developers+1](https://developers.openai.com/codex/cli/slash-commands/)

**architect**  
这个角色对新功能、跨服务改造、迁移、公共 API 设计很有价值；对小 bugfix 往往过重。建议给它加**触发条件**：只有“新模块 / 新接口 / 跨边界重构 / 数据模型变化”才启动。另一个优化点是把“查文档、核 API 版本行为”从角色人格里拆出去，改成 docs skill 或 MCP。官方示例就是这么做的。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

**be\_developer**  
方向是对的，但还不够“最小改动”。官方长期任务实践特别强调 milestone、validation、scoped diff。你的后端角色应该被强化成：**只动后端相关路径、优先补/改测试、每个 milestone 都做 targeted validation、不要顺手重构**。[OpenAI Developers](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex/)

**fe\_developer**  
也有价值，但当前 prompt 对技术栈假设太强，容易在异构仓库里误导。它更应该强调：尊重 repo 现有栈、设计系统、一致性、a11y、browser evidence。官方前端排障示例里，真正关键的并不是“前端开发者”头衔，而是 `browser_debugger` 这种**带浏览器工具的能力边界**。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

**tester**  
这是当前最需要修的一项。你把它设成了 `read-only`，但指令里又要求它“编写完整单元/集成/E2E 测试”，这在职责和权限上是自相矛盾的。再加上它用的是 `spark`，而官方把 Spark 定位成更快但更浅、偏探索/摘要的模型，不太适合做重型 QA gate。官方对测试的建议也很明确：**测试最好作为单独步骤/单独 session 来做，并在进入实现前先验证新测试会失败。**[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/concepts/multi-agents/)

**reviewer**  
这个角色是应该保留的，但应当更“高信号”。官方对 review 的建议是：优先 correctness、安全、行为回归、缺失测试，避免 style-only 噪音；CLI 还内建了 `/review` 工作流。我的建议是让 reviewer 明确只输出：**Severity、Evidence、Repro、Fix Hint、Ship/No-Ship**，不要再把大量风格建议混进主结论。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/multi-agent/)

我认为最值得做的 7 个升级
--------------

**第一，把 6-agent 从“人格驱动”改成“artifact contract 驱动”。**  
你现在写的是角色人格，但官方长期任务实践真正稳定的关键是 durable project memory：`Prompt.md / Plan.md / Implement.md / Documentation.md` 这类可反复读取的文档，把 spec、done-when、milestone、validation、状态都外化。你这套 gated handoff 最该升级成：**没有 artifact 就不准 handoff**。例如：

*   team\_lead → `work_plan.md`
*   architect → `architecture.md` / `openapi.yaml`
*   developer → `implementation_report.md`
*   tester → `test_report.json`
*   reviewer → `review_report.md`  
    这样门控才不是“口头门控”，而是“文件门控”。[OpenAI Developers](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex/)

**第二，把 repo 常驻规则移到 `AGENTS.md`，不要塞进每个角色。**  
官方明确建议：`AGENTS.md` 负责 repo 的常驻规则，且要保持小而稳定；当 agent 重复犯错、总是读错目录、总收到相同 PR 反馈时，就把规则写进 `AGENTS.md`。像“跑哪些测试”“lint/typecheck 命令”“文档更新要求”“覆盖率规则”这类，都不应该每个 agent 重复声明。[OpenAI Developers+1](https://developers.openai.com/codex/guides/agents-md/)

**第三，把可重复流程做成 Skills，不要全靠 agent persona。**  
官方现在很明确：Skills 更适合可复用 workflow，因为它们支持 richer instructions、脚本、参考资料，而且是 progressive disclosure，不会一开始就把全部上下文塞爆。你这套团队特别适合拆出这些 skill：`docs-research`、`browser-repro`、`coverage-gate`、`perf-smoke`、`release-notes`。这样 6 个 agent 负责“谁来做”，Skills 负责“怎么做”。[OpenAI Developers+1](https://developers.openai.com/codex/concepts/customization/)

**第四，把外部系统接入放到 MCP，而不是角色说明里硬写。**  
官方示例已经给了两个很典型的方向：`docs_researcher` 通过 docs MCP 验证 API/框架行为，`browser_debugger` 通过 Chrome DevTools MCP 复现 UI 问题。更广一点，日志、CI、ticket、监控、设计系统都应该走 MCP。这样你的 roles 会变瘦，能力却会更强。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/multi-agent/)

**第五，执行 one-writer rule，并默认用 worktree 隔离。**  
官方反复强调：并行最适合读任务；多写 agent 会增加冲突。Codex app 和 automations 也都把 worktree 作为隔离手段。你这套 6-agent 最稳的方式不是“BE 和 FE 一起改主工作树”，而是：**同一时刻只有一个代码写手；其他角色并行只读；必要时每个 writer 独立 worktree。**[OpenAI Developers+2OpenAI+2](https://developers.openai.com/codex/concepts/multi-agents/)

**第六，把 `monitor` 当底层系统角色保留，不要试图被 team\_lead 替代。**  
长测试、长 build、轮询 CI、日志追踪、后台等待，本来就是 native `monitor` 的职责。你现在 6 个自定义角色里其实缺了这一层，所以在长任务场景里会出现“所有角色都能开始，但没人适合纯等待和回报状态”的问题。[OpenAI Developers](https://developers.openai.com/codex/multi-agent/)

**第七，刷新模型映射。**  
我会建议：`team_lead / architect / reviewer` 默认上 `gpt-5.4`；`be_developer / fe_developer` 用 `gpt-5.4` 的 medium/high；只有探索、日志摘要、初筛这类子任务才交给 Spark。现在把 tester 放在 Spark 上做全套 QA，性价比不高。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/cli/features/)

我会怎样把它收敛成“更强的 6-agent v2”
-------------------------

我不会把你的 6 个角色砍掉，而是会把它们重新定位成：

*   **team\_lead**：只做计划、门控、最终合成；默认只写 `.agent/` 或 `.codex/artifacts/`。
*   **architect**：只做设计与契约；默认只读；只在新模块/新接口/迁移/大重构触发。
*   **be\_developer**：唯一后端写手；最小可辩护改动；必须带测试。
*   **fe\_developer**：唯一前端写手；强调设计系统、一致性、a11y；优先结合 browser-debug skill。
*   **tester**：二选一：要么改名为 `qa_auditor` 并保持只读；要么升级为 `qa_engineer`，允许只改 `tests/**`、snapshot、report。
*   **reviewer**：只读最终 gate；高信号输出；默认走 `/review` 思路。

同时把 Native 4 当底层 runtime 模板来用：  
`default` 负责兜底，`worker` 承载实际代码写手，`explorer` 承载 architect/tester/reviewer 的证据收集，`monitor` 专门盯长任务。这样 6-agent 不再和 Native 4 打架，而是骑在它们上面。[OpenAI Developers+1](https://developers.openai.com/codex/multi-agent/)

一句话总结：**Native 4 更像“执行内核”，你这 6-agent 更像“团队操作系统”。** 当前版本的主要问题不是“想法错了”，而是**角色太宽、权限与职责有冲突、缺少 artifact contract、没有充分借力 AGENTS.md / Skills / MCP / worktrees / monitor**。把这几件事补上，这套 6-agent 会从“像人在演流程”变成“像系统在跑流程”。

## Prompt:
请提供完整可落地的6-agent 更像“团队操作系统，修复“角色太宽、权限与职责有冲突、缺少 artifact contract、没有充分借力 AGENTS.md / Skills / MCP / worktrees / monitor”，给出全套artifact

## Response:
我先按官方当前能力整理一套可直接落地到仓库的 6-agent“团队操作系统”：包含 agent 配置、AGENTS.md、artifact contract、Skills、MCP 模板、worktree/monitor 脚本与推荐目录结构。

我还会把它做成可退化运行的版本：即使不开 multi-agent，也能按同一套 artifact gate 与脚本单 agent 执行，并补齐 CLI/app 共享配置。

这次我会把 Native `default/worker/explorer/monitor` 的映射、artifact 模板、安装脚本也一起打进 repo 级完整包；刚才生成 bundle 时的 here-doc 分隔符冲突已定位并修复。

Thought for 34m 10s

我已经把整套 **可直接落地的 6-agent Team OS scaffold** 做好了，并打成了可下载的完整包：

完整 ZIP 包  
README  
主配置 `.codex/config.toml`

这套模板是按 Codex 当前官方能力来收敛的：把 **AGENTS.md** 用作持久项目规则，把 **Skills** 用作可复用流程，把 **MCP** 用作外部系统接入，把 **Multi-agents** 用作运行时分工；这些层是互补关系，不是替代关系。官方也明确建议并行 agent 更适合 **读多写少** 的任务，多个写手并行改代码会增加冲突和协调成本。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/concepts/customization/)

我这次做的核心修复有 5 个。

第一，**把角色从“宽人格”改成“窄合同”**。  
现在 6 个自定义角色分别只做：`team_lead` 负责计划/门控/合成，`architect` 只做设计，`be_developer`/`fe_developer` 只做各自实现，`tester` 改成只读 QA 审计，`reviewer` 只做最终 gate。与此同时，我还顺手把 native `explorer` 和 native `monitor` 也调优进来了，因为官方当前固定内建角色就是 `default / worker / explorer / monitor`，其中 `monitor` 就是为长命令、轮询、等待型任务准备的。多 agent 当前仍是 experimental，官方文档里最明确的支持面仍在 CLI。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/multi-agent/)

第二，**修掉了 tester 的“职责/权限冲突”**。  
你原来的 tester 一边是 `read-only`，一边又要求“编写完整单元/集成/E2E 测试”，这在运行时是自相矛盾的。新版本里，tester 默认是 **read-only QA auditor**：负责测试计划、命令执行、失败归因、覆盖缺口和 ship/no-ship 建议；真正需要补测试时，由 Team Lead 指派当前唯一 writer 去补最接近的测试。这是按照官方“并行更适合探索/测试/triage/总结，写代码要谨慎”的方向收敛的。[OpenAI Developers+1](https://developers.openai.com/codex/concepts/multi-agents/)

第三，**把口头 handoff 变成了文件化 artifact contract**。  
每个任务现在都落在 `.codex/tasks/<TASK_ID>/` 下，标准文件链路是：`00-task.md`、`10-plan.md`、`20-architecture.md`、`21-openapi.yaml`、`30-implement.md`、`40-documentation.md`、`50-test-plan.md`、`60-test-report.md`、`70-review.md`、`80-release-checklist.md`。这套命名是刻意对齐官方 long-horizon 实践里的 `Plan.md / Implement.md / Documentation.md` 思路，再加上架构、测试、评审和 release gate。官方明确把 `Plan.md` 定义为 milestone + validation 的检查点文件，把 `Implement.md` 定义为执行 runbook，把 `Documentation.md` 定义为共享状态和审计日志。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex)

第四，**把 AGENTS.md、Skills、MCP 真正拆层用了**。  
官方建议 `AGENTS.md` 保持小而稳定，放 repo 级常驻规则；它会在每次开工前加载，而且按全局到项目目录逐层发现，支持 `AGENTS.override.md` 和 fallback 文件名。Skills 则应该“一技能一工作”，只在命中描述时按 progressive disclosure 加载完整说明；repo 级 skills 放在 `.agents/skills` 下就会被扫描。MCP 则直接在 `config.toml` 里配置，CLI 和 IDE extension 共用。[OpenAI Developers+3OpenAI Developers+3OpenAI Developers+3](https://developers.openai.com/codex/guides/agents-md/)

第五，**补上了 worktree 和 monitor 的运行时策略**。  
这套 scaffold 明确执行 **one-writer rule**，并提供了 `create-worktree.sh` 和 `worktree-bootstrap` skill。这样高风险、长任务、易冲突任务可以先隔离到 worktree；在 Codex app 里，官方本身就支持 Local/Worktree/Handoff，automations 也会在独立 worktree 里跑，避免干扰主 checkout。长测试、长 build、CI 轮询这类等待型工作，则交给 tuned native `monitor`。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/app/worktrees/)

你下载后，最值得先看的 4 个文件是：

*   `README.md`：整体运行方式、安装步骤、角色边界、任务生命周期。
*   `.codex/config.toml`：6 个自定义 agent + 2 个 tuned native roles 的主配置。
*   `AGENTS.md`：仓库级常驻规则模板。
*   `PROMPTS.md`：直接可用的 prompt recipes。

另外我还放进去了这些可直接用的内容：

*   `.codex/scripts/new-task.sh`：一键创建 task contract。
*   `.codex/scripts/check-gates.py`：按 frontmatter `status` 检查 gated handoff。
*   `.codex/mcp.project.example.toml`：Context7 / Chrome DevTools / Figma 的 MCP 示例。
*   `.codex/requirements.example.toml`：给管理员或企业环境用的约束样例。
*   `.agents/skills/*`：`contract-bootstrap`、`docs-research`、`browser-evidence`、`validation-gate`、`release-synthesis`、`worktree-bootstrap` 六个 skills。

模型选择上，我默认把主力都切到了 **`gpt-5.4`**，因为官方当前明确写的是：对 Codex 里的大多数任务，`gpt-5.4` 是推荐模型；`gpt-5.3-codex-spark` 更适合追求速度的额外快速任务，而且还是 research preview。[OpenAI Developers+1](https://developers.openai.com/codex/cli/features/)

安全姿态上，我默认保留了 `workspace-write + on-request + web_search = "cached"`，并显式把 `sandbox_workspace_write.network_access = false` 写进主配置。官方当前文档说明，本地默认就是无网络、受 sandbox 和 approval policy 共同约束；`cached` web search 使用的是 OpenAI 维护的索引结果而不是直接抓 live page，暴露面更小。项目级 `.codex/config.toml` 只有在 **trusted project** 下才会生效，这也是为什么 README 里把“先 trust 项目”放进了第一步。[OpenAI Developers+3OpenAI Developers+3OpenAI Developers+3](https://developers.openai.com/codex/security/)

你可以按这个顺序落地：

1.  把 ZIP 解压到你的 repo 根目录。
2.  检查并补全 `AGENTS.md` 里的 repo-native lint/test/build 命令。
3.  按需把 `.codex/mcp.project.example.toml` 合并到你的 user 或 project config。
4.  从 repo 根目录启动 Codex。
5.  先跑一次 `contract-bootstrap` 或直接执行 `./.codex/scripts/new-task.sh TASK-... "..."`。
6.  用 `PROMPTS.md` 里的第一条 prompt 开始跑。

官方当前也支持 `/plan`、`/agent`、`/review`、`/mcp` 这些命令；其中 `/plan` 适合先出审阅式计划，`/review` 适合对 working tree 做额外 gate，`/agent` 用来切换到 spawned sub-agent 线程。[OpenAI Developers+2OpenAI Developers+2](https://developers.openai.com/codex/cli/slash-commands/)

我还做了一次基本自检：已经在包内成功生成了一个示例任务 `TASK-20260306-demo`，并通过了 `check-gates.py` 对 `00-task.md / 10-plan.md / 30-implement.md / 40-documentation.md` 的状态检查。

发我你当前仓库的技术栈和目录结构，我可以把这套 scaffold 再收敛成你的 **repo-specific 版本**。



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)