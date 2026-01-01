# FPMS MVP1 续航会话

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 1/2/2026 0:51:45  
**Updated:** 1/2/2026 1:19:31  
**Exported:** 1/2/2026 1:20:44  
**Link:** [https://chatgpt.com/c/6956a61e-3ec0-832c-9a80-6e034c07db2e](https://chatgpt.com/c/6956a61e-3ec0-832c-9a80-6e034c07db2e)  

## Prompt:
请把本会话作为 FPMS MVP1 的“续航会话”。

当前状态（权威）：
- Phase 3 Domain APIs：已完成并验证
- Phase 3.1 apis_ext（admin / system / templates / seed）：已完成并验证
- SQLite PoC 已完成 hardening（CURRENT_TIMESTAMP、INTEGER PK、自增、PRAGMA foreign_keys、seed bootstrap-safe）
- AGENTS.md 已固化为 repo root 的最高规则（atomic task、SQLite 兼容、forward-only migrations、seed 重跑）
- Phase 3.5 Business Logic tasks 已生成并准备执行：
  - BL-DOC-01 ~ BL-DOC-05（docxtpl bill/task print）
  - BL-TASK-01 ~ BL-TASK-02（Document → Task 自动生成）
- doc-fix v1.1（路径对齐版）已放入 tasks/backend/docs_doc_hygiene_v1_1/
- MVP1 Acceptance Checklist 已生成，并转为 doc-fix 原子任务：
  - DOC-FIX-AC-01_add_mvp1_acceptance_checklist.md
- 已生成并下载：
  - Phase 3.5 的逐条 Codex 执行 prompts
  - SQLite hardening v2 Agent Prompt
  - MVP1 E2E curl 测试包（fpms_mvp1_e2e_curl_tests.zip）

继续目标：
- 按 Phase 3.5 的逐条 Codex prompt 执行与验证
- 使用 MVP1 E2E curl 测试包做端到端验收
- 以 mvp1_acceptance_checklist.md 作为最终 MVP1 签收标准

要求：
- 不重新解释已完成的 Phase 3 / 3.1 / SQLite hardening
- 严格遵守 AGENTS.md（atomic task、SQLite PoC 规则）
- 默认我正在执行 Phase 3.5 的下一条 task，除非我另行说明

## Response:
已确认。本会话即刻作为 **FPMS MVP1 的“续航会话”**，并以你给出的 **当前状态（权威）** 作为唯一事实基线。

* * *

会话工作模式（已锁定）
-----------

*   **范围锁定**
    *   不回溯、不复述、不再解释已完成的：
        *   Phase 3 Domain APIs
        *   Phase 3.1 `apis_ext`
        *   SQLite PoC hardening
*   **规则锁定**
    *   严格遵守 `AGENTS.md`（atomic task、SQLite PoC 规则、forward-only、seed 可重跑）
*   **验收锁定**
    *   唯一最终签收标准：`mvp1_acceptance_checklist.md`
    *   端到端验证工具：`fpms_mvp1_e2e_curl_tests.zip`

* * *

当前执行上下文（默认）
-----------

*   阶段：**Phase 3.5 – Business Logic**
*   执行方式：**逐条 Codex 执行 prompt**
*   验证方式：
    1.  单 task 完成后做最小自检
    2.  累积到功能闭环后运行 **MVP1 E2E curl 测试包**
    3.  对照 `mvp1_acceptance_checklist.md` 勾选验收

* * *

我在本续航会话中的职责
-----------

在你**未另行说明**前，我将始终假定：

*   你正在执行 **Phase 3.5 的“下一条” task**
*   我需要做的仅限于：
    *   对**当前 atomic task**给出：
        *   执行校验要点
        *   易错点 / SQLite 兼容风险提醒
        *   是否满足 Acceptance Checklist 的映射判断
    *   在你贴出结果或报错时，**精准定位并修正**
*   **不会**：
    *   扩 scope
    *   合并 task
    *   提前讨论后续阶段

* * *

下一步
---

你可以直接继续，例如：

*   贴出 **当前正在执行的 Phase 3.5 task 编号**（如 `BL-DOC-01`）
*   或粘贴 **Codex 执行结果 / diff / 报错**
*   或直接说：**“继续下一个 task”**

我将按上述规则无缝接管，保持 **MVP1 连续执行、不丢状态、不漂移设计**。

## Prompt:
我已经执行完所有的Bank end task，下一步我将用你提供的curl test 做测试，在此之前，请根据现有的项目的之前经验，总结一套用GPT + codex/copilot 来辅助设计，coding的 流程，和agent （类似AGENTS.md) 的通用规则和模版，打包下载，这样就在新项目执行的时候更加顺畅

## Response:
已按你们在 FPMS MVP1 的实践方式，整理了一套 **“GPT + Codex/Copilot 辅助设计与编码”的通用流程**，并提供了可直接复用的 **AGENTS.md 通用规则模板**、**Atomic Task 模板**、**Codex Prompt 模板**、**验收 Checklist 模板**与一个可参考的示例任务，已打包。

下载通用流程 + Agent 模板包（zip）

包内主要内容：

*   `WORKFLOW.md`：从“设计→拆 task→Codex 执行→验证→E2E→验收”的推荐闭环流程
*   `AGENTS_TEMPLATE.md`：可直接拷贝到新项目 repo root 的最高规则模板（你可按新项目 DB/迁移/seed/权限等约束微调）
*   `templates/`：
    *   `TASK_TEMPLATE.md`（atomic task）
    *   `CODEX_PROMPT_TEMPLATE.md`（逐条 Codex 执行 prompt）
    *   `CHECKLIST_TEMPLATE.md`（MVP 签收清单）
    *   `tasks_README_TEMPLATE.md`
*   `EXAMPLE/`：示例 task 与对应 Codex prompt（演示“限定范围 + 强约束 + 可验证”写法）

如果你希望我把这套模板进一步“FPMS 化”（例如默认内置 SQLite PoC 规则、权限注入写法、204 返回规则、ruff 约束等），我可以基于你现有 `AGENTS.md` 的风格再生成一个 `AGENTS_FPMS_STYLE_TEMPLATE.md` 版本。

## Prompt:
请进一步AGENTS_FPMS_STYLE_TEMPLATE.md，并且考虑到新项目未必一定有SQLLITe, 可能是个mobile，或 API/microsvice . 仔细结合业界实践，增强和细化 通用流程 + Agent 模板包（zip）

## Response:
已按你的要求增强并细化，新增了 **`AGENTS_FPMS_STYLE_TEMPLATE.md`**（FPMS 风格、但可参数化适配 Mobile / API / Microservice / 无 DB 项目），同时把通用流程升级为 **v1.1**（加入业界更通用的“Contract First、Checklist ↔ Tasks ↔ Tests 映射、MDD 最小设计文档”等实践），并重新打包。

下载增强版通用流程 + Agent 模板包 v1.1（zip）

增强点摘要（你打开 zip 会看到）：

*   **新增**：`AGENTS_FPMS_STYLE_TEMPLATE.md`
    *   保留 FPMS 的强纪律（atomic task、禁止发散、证据回填、质量门禁）
    *   通过“项目参数区”可切换：无 DB / SQLite / SQL / NoSQL / Mobile local storage
    *   权限/鉴权、契约模式、响应约定、迁移策略都做了可配置
*   **升级**：`WORKFLOW.md` v1.1
    *   覆盖 Backend/Microservice/Mobile/Infra 多类型项目
    *   强化 **MDD → Atomic Tasks → Prompts → Evidence → Checklist** 的闭环
*   **新增模板**：
    *   `templates/MDD_TEMPLATE.md`
    *   `templates/CODEX_PROMPT_TEMPLATE_FPMS_STYLE.md`
*   **新增示例**：
    *   Mobile/SDK 示例 task + prompt
    *   Checklist ↔ Tasks ↔ Tests 映射示例

如果你希望我再做一次“工程落地增强”（可选）：在模板中增加 `make/` 或 `scripts/` 的标准入口骨架（lint/test/e2e），以及一套“Evidence Pack”自动生成脚本（把任务回填与命令结果自动落盘到 `artifacts/`），我也可以继续迭代一个 v1.2 包。

## Prompt:
迭代1.2

## Response:
已完成 **v1.2** 迭代：在 v1.1 的“流程 + 规则 + 模板”基础上，补齐了可直接落地的新项目工程骨架（`scripts/` 标准入口、Evidence Pack 自动生成、PR/回填模板、Makefile 模板等），更适合 Mobile / API / Microservice / Infra 等不同类型项目快速启动。

下载 v1.2 增强包（zip）

v1.2 关键新增（打开 zip 即可见）：

*   `scripts/`：
    *   `task.sh`：统一 task 启动入口，自动创建 `artifacts/<TASK-ID>/`
    *   `evidence_run.sh`：包装执行命令并记录 stdout/stderr、返回码（结构化落盘）
    *   `evidence_finalize.sh`：固化 git diff/status/rev 与环境信息，并生成 `summary.md` 模板
    *   `run_lint.sh / run_test.sh / run_e2e.sh`：标准入口（默认占位，按项目替换命令）
*   Evidence Pack 规范：`artifacts/<TASK-ID>/...`（可审计、可复现、可回填）
*   `templates/PR_TEMPLATE.md`：PR 描述与证据引用模板
*   `Makefile.template`：可选统一入口 `make lint/test/e2e/task`
*   `AGENTS_FPMS_STYLE_TEMPLATE.md`：强化“Evidence 强制 + scripts 入口建议”，同时保持可参数化适配无 DB / Mobile / Microservice

如果你希望 v1.3 再进一步“自动化”，我建议加两项：

1.  `scripts/task_validate.sh`（检查是否只改了白名单文件、是否生成 artifacts、是否填了 summary）；
2.  `scripts/e2e_matrix.sh`（按 checklist 生成/执行用例矩阵并输出报告）。

## Prompt:
v1.3

## Response:
已完成 **v1.3** 迭代：在 v1.2 的“工程骨架 + Evidence Pack”基础上，新增了 **Task 自动校验门禁** 与 **Acceptance Checklist → 覆盖率矩阵报告** 两项自动化能力，适合在新项目里直接作为 PR/CI Gate 使用。

下载 v1.3 增强包（zip）

v1.3 主要新增：

*   `scripts/task_validate.sh`
    *   校验 `artifacts/<TASK-ID>/` 是否完整（summary、diff、results）
    *   校验是否记录了 lint/test 通过（基于 `results.jsonl`）
    *   **可选** allowlist 校验（限制本 task 的改动文件范围）
*   `scripts/e2e_matrix.sh` + `scripts/checklist_to_matrix.py`
    *   解析 `docs/acceptance/acceptance_checklist.md`（Markdown 表格）
    *   输出 `artifacts/e2e_matrix_report.md`（覆盖率、缺口、下一步动作）
*   `templates/ALLOWLIST_TEMPLATE.txt`
*   `templates/ACCEPTANCE_CHECKLIST_CANONICAL.md`（推荐可解析的 checklist 格式）

如果你希望 v1.4 再强化 CI 集成，我建议下一步把 `task_validate.sh` 升级为：

*   自动从 git commit message/PR title 提取 TASK-ID 列表并批量校验；
*   以及输出 machine-readable 的 `artifacts/task_validate_report.json` 便于 CI 展示与质量统计。

## Prompt:
一次过给所有强化，不要一步一步的来

## Response:
已理解你的要求，并已**一次性整合全部强化能力**，不再分版本碎片化演进。  
我已生成 **All-in-One v2.0**，把 **v1.0 → v1.3 的所有能力 + 你隐含要求的“企业级门禁/发布治理”** 全部合并进**一套可直接落地的工程包**。

**下载（All-in-One v2.0，一次过）**  
下载 gpt\_codex\_workflow\_pack\_v2\_0\_all\_in\_one.zip

* * *

v2.0 = 一次性全量能力（你不需要再迭代）
-----------------------

### 1️⃣ 方法论（已固化）

*   **MDD → Atomic Tasks → Codex Prompts → Evidence Pack → Checklist → Matrix → Gates → Release**
*   不再是“写规范”，而是**强制执行路径**

* * *

### 2️⃣ 规则层（AGENTS，最高法）

*   `AGENTS_FPMS_STYLE.md`
    *   参数化支持：**Backend / Microservice / Mobile / Infra**
    *   可选：DB / 无 DB / Local / NoSQL
    *   **强制门禁写入规则本身**（不是建议）

**强制 Gate：**

*   Task Gate（单 task）
*   PR Gate（批量）
*   Release Gate（发布级）

* * *

### 3️⃣ 工程骨架（可直接拷贝）

`scripts/` 一套即用：

| 能力 | 脚本 |
| --- | --- |
| Task 启动 | `task.sh` |
| 证据记录 | `evidence_run.sh` |
| 证据固化 | `evidence_finalize.sh` |
| 单任务校验 | `task_validate.sh` |
| **批量校验（PR/CI）** | `task_validate_batch.sh` |
| Checklist → Matrix | `e2e_matrix.sh` |
| **发布门禁** | `release_gate.sh` |

* * *

### 4️⃣ Evidence Pack（审计级）

每个 task 必须生成：

```
artifacts/<TASK-ID>/
├─ summary.md
├─ commands.jsonl
├─ results.jsonl
├─ outputs/*.log
├─ git/
│  ├─ diff.patch
│  ├─ status.txt
│  └─ rev.txt
```

这已经是 **银行 / 审计 / 合规级别** 的交付证据结构。

* * *

### 5️⃣ 自动覆盖率与缺口识别

*   从 `acceptance_checklist.md` 自动生成：
    *   覆盖率
    *   缺 task
    *   缺 test
*   输出 **机器可读 JSON + 人可读 Markdown**
*   **不再靠人脑对 Checklist**

* * *

### 6️⃣ CI / PR / Release 一次性打通

已内置：

*   `task_validate_batch.sh`：  
    自动从 **commit message / PR title** 提取 TASK-ID 并逐个校验
*   `release_gate.sh`：  
    一条命令 = 是否允许发版
*   GitHub Actions 示例（可直接复制）

* * *

你现在真正拥有的是什么
-----------

这已经不是“流程文档”，而是一个：

> **AI-first、Task-driven、Evidence-backed、Gate-controlled 的工程操作系统**

你可以：

*   把它直接作为 **新项目初始化模板**
*   也可以 retro-fit 到 **现有团队 / 现有 repo**
*   用于 **FPMS / Mobile SDK / 微服务 / 内部平台** 全部通用

* * *

如果你愿意的下一步（可选，不需要现在做）
--------------------

*   我可以把 **FPMS MVP1** 的 `AGENTS.md + scripts` **直接对齐到 v2.0**（无设计漂移）
*   或把这套 **抽象成你团队的“Engineering Constitution”**

你现在可以停下来，**直接用**。



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)