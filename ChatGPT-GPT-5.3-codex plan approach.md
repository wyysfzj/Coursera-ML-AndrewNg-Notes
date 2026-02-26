# GPT-5.3-codex plan approach

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 2/26/2026 21:56:15  
**Updated:** 2/26/2026 22:08:05  
**Exported:** 2/26/2026 22:15:30  
**Link:** [https://chatgpt.com/c/69a050ff-2e60-8321-ac81-a64aaaff16fa](https://chatgpt.com/c/69a050ff-2e60-8321-ac81-a64aaaff16fa)  

## Prompt:
Claude\_enhance.md

File

Atomic Task Writing Specification v1.0.md

File

我有一个基于Spring boot 的项目，只能用github copilot + gpt-5.3-codex. 我希望 也能让 gpt-5.3-codex 在plan模式下生成 类似的Claude_enhance.md 的plan，重点是 1.有iron rules,2有定义好的quality  GATE ,3 plan 有atomic task 组成 还有其他符合atomic 要求的要素。请给出一个可操作的approach

## Response:
我会先抽取你给的 Claude\_enhance.md 与 Atomic Task 规范里的关键结构（iron rules / quality gate / atomic要素），再把它翻译成适用于“GitHub Copilot + gpt-5.3-codex（plan 模式）”的可落地流程：包括仓库内文件约定、Plan 模板、GATE 命令、以及如何把 Plan 自动拆成原子任务并执行。

Thought for 11m 48s

下面给你一套**能落地**的做法：把“Claude\_enhance.md 的计划骨架 + Atomic Task 规范 + 质量门禁”固化成仓库里的**常驻上下文**（Copilot 指令文件 + 脚本 + 模板），再用 gpt‑5.3‑codex 的 **PLAN MODE → TASK MODE → EXECUTE MODE** 三段式输出，保证计划里天然包含 **Iron Rules / Quality Gate / Atomic Tasks**，并且每个 task 都符合 atomic 要求。

我会用你给的两个标准作为源：

*   Claude 风格的批次计划模板（Iron Rules、Quality Gate、批次/门禁纪律等）
    Claude\_enhance
*   Atomic Task 写作规范（1 task=1 file=1 responsibility、禁止条件语、固定结构等）
    Atomic Task Writing Specificati…

* * *

1) 先把“规则”写进仓库：让 Copilot / Codex 每次都读得到
--------------------------------------

### 1.1 仓库级指令文件（关键）

GitHub Copilot 支持在仓库里放自定义指令文件，最通用的是：

*   `.github/copilot-instructions.md`（仓库级，Copilot Chat 读取） [GitHub Docs+2Visual Studio Code+2](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot?utm_source=chatgpt.com)
*   如果你用 Copilot coding agent（后台 agent），还可以加 `AGENTS.md`（根目录或子目录生效） [The GitHub Blog](https://github.blog/changelog/2025-08-28-copilot-coding-agent-now-supports-agents-md-custom-instructions/)
*   还可以做路径级指令（对 tasks/、src/、docs/ 分别定规则） [GitHub Docs](https://docs.github.com/en/copilot/tutorials/customization-library/custom-instructions/your-first-custom-instructions?utm_source=chatgpt.com)

**为什么要这样做**：  
Plan 不是“临时 prompt 工程”，而是“仓库规范”。你把 Iron Rules、Quality Gate、Atomic Task 结构写进去后，gpt‑5.3‑codex 每次对话都在同一约束框架里工作，输出就稳定很多。

* * *

2) 建议的仓库结构（最小可行）
----------------

在 repo 里增加这些文件/目录（名字你可按需改，但建议保持稳定）：

```
.github/
  copilot-instructions.md
docs/
  ai/
    PLAN_SPEC.md
    QUALITY_GATE.md
    ATOMIC_TASK_SPEC.md   (可直接复制你现有规范/或引用)
  design/
    ...（你的设计文档：DB/API/RULES 单一真相源）
tasks/
  backend/
    BE-xx-yy.md           (Atomic Tasks 文件都放这里)
plans/
  ENHANCE-YYYY-MM-DD.md   (每次 plan 输出一个快照)
scripts/
  quality-gate.sh
  atomic-task-lint.py     (可选但强烈建议)
```

**设计文档（docs/design）是关键**：Atomic Task 规范要求“Design references 是单一真相源”，task 不能凭空发明字段/行为 。  
所以你至少需要为每个模块准备 2~3 个 design doc（DB / API / Rules），哪怕是很短的 markdown。

* * *

3) Spring Boot 的 Quality Gate：把“门禁”变成一条命令
-----------------------------------------

Claude\_enhance 的核心是：**每个 batch 结束必须跑同一套质量门禁** 。  
在 Spring Boot 项目里，我建议把门禁固化成脚本：`scripts/quality-gate.sh`，并且在计划里规定“每批必须跑它”。

### 3.1 一个可直接用的 Quality Gate（通用 Maven/Gradle 版本）

你可以先

Atomic Task Writing Specificati…

见）：

```
#!/usr/bin/env bash
set -euo pipefail

# 0) 基础编译 + 单测
./mvnw -B -q test

# 1) 打包（确保可打包）
./mvnw -B -q -DskipTests package

# 2) 启动校验（可选但推荐：保证启动 + health）
# 如:contentReference[oaicite:8]{index=8}agement.endpoints.web.exposure.include=health,info
#   management.endpoint.health.probes.enabled=true
JAR="$(ls -1 target/*.jar | head -n 1)"
(java -jar "$JAR" > /tmp/app.log 2>&1 & echo $! > /tmp/app.pid)
sleep 5
curl -sf http://localhost:8080/actuator/health > /dev/null
kill "$(cat /tmp/app.pid)"
echo "QUALITY GATE: PASS"
```

然后在计划里写死：**每个 batch 结束必须执行**：

```
bash scripts/quality-gate.sh
```

> 说明：你项目如果是 Gradle，把 `./mvnw ...` 换成 `./gradlew test build` 就行。关键是：**门禁必须唯一且可复现**。

### 3.2 强化门禁（把格式化/静态检查纳入）

如果你项目已经有 Spotless / Checkstyle / SpotBugs / OWASP 等插件，就把它们纳入脚本；如果没有，建议在第一个 batch 里加上并固定为 `./mvnw -Pci verify`（更稳）。

* * *

4) Iron Rules：直接照 Claude\_enhance 的“执行纪律”写进指令
---------------------------------------------

你想要的三个重点里，Iron Rules 是“保证 plan 不漂移”的核心。可以直接把 Claude\_enhance 的理念迁移到 Spring Boot：

建议放进 `.github/copilot-instructions.md`（或 docs/ai/PLAN\_SPEC.md），典型 Iron Rules（可复制）：

1.  **Batch 顺序强制**：不得跳批、不得重排（防止依赖倒置）。
2.  **每批必须过 Quality Gate**：只要门禁没过，不允许进入下一批。
3.  **一批一 PR**：每个 batch 对应一个可合并 PR（One concern per PR）。
4.  **迁移只前进**：Flyway/Liquibase/SQL migration 不回滚；修复用新 migration。
5.  **禁止 Scope Creep**：Frozen Features 列表写死，不出现在任何 batch。
6.  **证据必留**：每批记录 `scripts/:contentReference[oaicite:13]{index=13}如` artifacts/B1/quality-gate.txt\`）。
    Claude\_enhance
    优先/契约优先\*\*：先写 design docs（OpenAPI/DTO/DB schema），再写代码，再写测试。
    Claude\_enhance
    是唯一执行单元\*\*：实现时必须引用某个 task 文件；不得“自由发挥”。

* * *

5) 让 plan “由 Atomic Tasks 组成”：核心设计
----------------------------------

### 5.1 关键落地规则

你提供的

Claude\_enhance

*   **一个 task 只改一个文件（Target = exactly one file）**
*   **结构固定**（Title/Design references
    Claude\_enhance
    n FIXED/Exact items/Non-scope/Prompt）
*   **禁止条件语言**（不要出现 “optional/if you want/…”）
*   **字段/行为必须完全枚举**  
    Atomic Task Writing Specificati…
    ot 下怎么做到“一 task 一文件”  
    Spring Boot 的 feature 往往跨多层（Entity/Repo/Service/Controller/Test/Migration）。解决方法是：
*   **把“一个 feature”拆成多条 atomic tasks**，每条只锚定一个文件
*   用依赖顺序串起来（Migration → Entity → DTO → Repo → Service → Controller → Test → Docs）

例子（同一个功能：新增 Customer 地址）：

*   BE-01-01：新增 Flyway migration `V202602:contentReference[oaicite:22]{index=22}r_address.sql`（只改这一个文件）
*   BE-01-02：新增实体 `CustomerAddressEntity.java`（只改这一个文件）
*   BE-01-03：新增 repository `CustomerAddressRepository.java`
*   BE-01-04：新增 service `CustomerAddressService.java`
*   BE-01-05：新增 controller `CustomerAddressController.java`
*   BE-01-06：新增测试 `CustomerAddressControllerTest.java`

**plan 里每个 batch 只列 task ID 和路径**，执行时严格按 task 顺序。

* * *

6) 关键：把“PLAN MODE / TASK MODE / EXECUTE MODE”写成可复制的 Prompt
----------------------------------------------------------

你要让 gpt‑5.3‑codex “在 plan 模式下输出 Claude\_enhance 风格 plan + atomic tasks”，最稳的方式是：**控制词 + 输出格式约束 + 引用仓库规范文件**。

下面给你 3 段可直接在 Copilot Chat 里粘贴的 prompt（建议固定用这三段）。

* * *

### 6.1 PLAN MODE Prompt（输出增强计划，不写代码）

> 目标：生成一个类似 Claude\_enhance.md 的 plan（含 Iron Rules / Quality Gate / Frozen Features / Batches / Dependency graph / 每批 atomic task 列表）。

```
PLAN MODE.

Read and follow:
- docs/ai/PLAN_SPEC.md
- docs/ai/QUALITY_GATE.md
- docs/ai/ATOMIC_TASK_SPEC.md
- all docs/design/** as single source of truth

Output a single Markdown file content for: plans/ENHANCE-YYYY-MM-DD.md

Hard requirements:
1) Must include "Iron Rules" section.
2) Must include "Quality Ga:contentReference[oaicite:24]{index=24}t runnable commands (use scripts/quality-gate.sh).
3) Must include "Frozen Features" section.
4) Must include "Current Codebase Reality (Audit Results)" section based on repository reading.
5) Must include "Batch Execution Plan" section with:
   - dependency graph
   - batches in strict order
   - each batch lists atomic tasks (TaskID + target file path + short title)
   - each batch has acceptance criteria + quality gate evidence requirement

Constraints:
- Plan only: do not output code patches.
- No vague wording; be explicit and deterministic.
- If information is missing, add Batch A0 "Baseline Verification" to generate facts first (commands, checks, outputs).
```

> 解释：你用这段 prompt，会强制 Codex 输出“Claude 风格计划”，并且 plan 里只包含 atomic task 列表，不让它直接动代码。

* * *

### 6.2 TASK MODE Prompt（把某个 batch 展开成 Atomic Task 文件）

> 目标：把 plan 里的某一批（比如 Batch A1）展开成多个 `tasks/backend/BE-xx-yy.md` 文件，且每个文件**严格符合 Atomic Task Writing Specification**。

```
TASK MODE.

Input:
- plans/ENHANCE-YYYY-MM-DD.md (Batch A1 section)
- docs/design/** referenced by Batch A1

Generate Atomic Task files under:
- tasks/backend/

Rules (STRICT):
- Each task modifies/creates ONLY ONE target file.
- Follow the exact section order from docs/ai/ATOMIC_TASK_SPEC.md.
- No conditional :contentReference[oaicite:26]{index=26}cision (MVP1 – FIXED)" must deny expansion.
- "Models / APIs / Components to implement (EXACT)" must fully enumerate fields/methods/endpoints relevant to that one file.
- "Non-scope" must list at least 3 explicit exclusions.

Output format:
For each task, output:
1) file path line: ### FILE: tasks/backend/<TaskID>.md
2) then the full markdown content for that file.
```

* * *

### 6.3 EXECUTE MODE Prompt（真正写代码：一次只执行一个 Atomic Task）

> 目标：你用 Copilot + Codex 真开始写代码时，避免“越写越多/跨文件漂移”。Atomic Task 里的 Target 已经限定为一个文件，所以执行 prompt 必须再次强调“只动 Target”。

```
EXECUTE MODE.

You are executing exactly ONE Atomic Task:
- tasks/backend/BE-xx-yy.md

Rules:
- Modify/create ONLY the file in the Target section.
- Implement EXACTLY what the task enumerates.
- Implement NOTHING from other tasks/batches.
- After editing, run the Quality Gate commands from docs/ai/QUALITY_GATE.md and report pass/fail.

Output:
- A short diff summary of the single target file change
- Quality gate result (PASS/FAIL) with the exact commands run
```

* * *

7) 建议的 `.github/copilot-instructions.md`（可直接用的骨架）
-------------------------------------------------

这是让 Copilot/Codex“常态化按你的制度输出”的关键文件（GitHub 官方支持的路径） [GitHub Docs+1](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot?utm_source=chatgpt.com)：

```
# Copilot Instructions (Repo-wide)

## Control words
- When the user says "PLAN MODE": output plan only, no code changes.
- When the user says "TASK MODE": output atomic task markdown files only.
- When the user says "EXECUTE MODE": implement exactly one atomic task, only the target file.

## Iron Rules
1. Batch order is mandatory. No skipping/reordering.
2. One batch = one merge-ready PR.
3. Every batch must pass Quality Gate (scripts/quality-gate.sh) before next batch.
4. Forward-only DB migrations. Never edit applied migrations.
5. No scope creep. Frozen features are forbidden.
6. Evidence required: record quality gate output per batch.

## Atomic Task Standard
- Atomic Tasks follow docs/ai/ATOMIC_TASK_SPEC.md exactly.
- One task modifies/creates exactly ONE file.
- No conditional language. No placeholders. Fully enumerate all fields/behaviors.
- Design docs under docs/design/** are the single source of truth.

## Quality Gate
Use scripts/quality-gate.sh as the single entry point.
```

> 如果你还会用 Copilot coding agent，可以把同样的内容放进 `AGENTS.md`（GitHub changelog 明确支持） [The GitHub Blog](https://github.blog/changelog/2025-08-28-copilot-coding-agent-now-supports-agents-md-custom-instructions/)。

* * *

8) “其他符合 atomic 的要素”你应该强制写进 plan 的清单
------------------------------------

除了你点名的 3 个（Iron Rules / Quality Gate / Atomic Tasks），为了让 atomic 真能跑起来，plan 里还应强制包含这些（都能从你的规范推导出来） ：

1.  **Design references 的路径规则**：每个 batch 必须列出引用的 design doc 路径（单一真相源）。
2.  **Scope 冻结区（Frozen Features）**：明确哪些不做，防止任务漂移。
3.  **任务 ID & 命名约定**：例如 `BE-<Batch>-<Seq>`，保证可追踪。
4.  **任务依赖顺序**：明确 Migration → Entity → Repo → Service → Controller → Test。
5.  **验收标准（Acceptance Criteria）**：每个 batch 明确“可自动验证”的验收点。
6.  **证据留存规范**：每个 batch 产出 `artifacts/<batch>/quality-gate.txt`。

* * *

9) 一条最短落地路径（你今天就能开始用）
---------------------

1.  把上面“仓库结构”建起来（至少 `.github/copilot-instructions.md`、`script:contentReference[oaicite:33]{index=33}ocs/ai/*`）。 [GitHub Docs+1](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot?utm_source=chatgpt.com)
2.  在 `docs/ai/ATOMIC_TASK_SPEC.md` 里放入你现有的 Atomic Task 规范（或直接复制）。
    Claude\_enhance
    LAN MODE prompt\*\* 让 gpt‑5.3‑codex 输出 `plans/ENHANCE-YYYY-MM-DD.md`。
3.  选第一个 batch，用 **TASK MODE prompt** 生成 `tasks/backend/BE-*.md`。
4.  开始实现：每次只用 **EXECUTE MODE** 跑一个 Atomic Task（只动 T
    Claude\_enhance
    ipts/quality-gate.sh\`，记录证据，提 PR。

* * *

如果你愿

Claude\_enhance

目录树/关键设计文档贴一下即可），把 `docs/ai/PLAN_SPEC.md` 和 `docs/ai/QUALITY_GATE.md` 写成**完全贴合你项目**的版本，并给你一份“首批 A0/A1 的 atomic task 列表示例”（依旧保持每 task 一文件的硬约束）。

Atomic Task Writing Specificati…

Claude\_enhance



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)