# FPMS C3.1 Git-native Lean 5.6 Reset 综合设计

- 日期：2026-07-28
- 仓库：`/Users/cfcc/Workshop/myprojects/fpms_mvp1_blueprint_atomic`
- 状态：C3 方向已获用户批准；书面独立复审完成（P0/P1 = 0），待用户文件确认
- 适用模型：GPT-5.6 High 为默认执行档位
- 文档性质：完整讨论、审计和设计依据；仅第 0 节是临时规范核心
- 非目标：本文件不修改产品代码，不关闭任何产品任务，不执行发布

## 0. 规范边界：C3.1 临时执行核心

本节是本文件唯一具有 cutover 规范效力的部分。第 1–20 节用于完整记录讨论、
事实、理由、备选方案和实施解释，均为只读审计附录，不得作为每个产品 story 的
启动上下文，也不得被解释成新的常驻治理手册。

本节持续约束整个 cutover，直到下列临时硬规则的终态条件全部通过独立复审。
第 18 节只是这些条件的展开说明，不增加新的规范义务。lean-governance adoption
commit 通过独立复审后，普通产品 story 的日常运行只加载下列精简活跃权威；
cutover 协调者仍必须遵守本节，直至 cutover 完成：

1. 精简 `AGENTS.md`；
2. `docs/product/v8/domain-contract.md`；
3. `docs/product/v8/source-decision-registry.md`；
4. `docs/product/v8/catalog.frozen.json`；
5. `docs/product/v8/coverage-ledger.json`；
6. 当前 story card；
7. 当前代码、测试和 Git commit。

C3.1 cutover 的临时硬规则：

1. 原 dirty workspace 保持只读 quarantine，不 reset、clean、stash、覆盖或继续开发。
2. preservation commit 只允许存在于 archive-only ref，不得成为 active integration
   branch 的父基线。
3. active lean integration 从固定 clean HEAD
   `afa58429e6b6e80b85f76055139e18fbe38ec9e8` 建立。
4. 当前 dirty 产品修改必须通过可见的 story adoption commit 导入 active branch；
   reviewer 必须能审到对应 preservation hunks。
5. 每个 visible dirty path 必须有唯一 disposition：产品 adoption story、治理/历史
   archive-only、本地忽略项或明确排除项；不得静默继承。
6. 85 个历史 terminal-PASS row 全部只是历史候选；在当前相关 hunks、测试和复审
   完成 adoption 前，不满足 Foundation。
7. 283-row catalog 必须逐行映射到 story/commit/test/disposition；不得伪造旧
   taskctl PASS。
8. 未知、混合或有争议的风险分类一律按 `PROTECTED`。
9. 法律状态、期限、官费、费减、支付、服务应收、谱系、权限、安全、schema、
   migration、seed、破坏性操作、SQLite、客户决定和来源 activation 继续
   fail-closed。
10. 普通开发范围由 isolated worktree 和 commit diff 定义；旧 taskctl/scope/
    evidence 只读，不得扩展。
11. coverage ledger、checker、wave report 和 release check 只能是普通 Git 文档
    或无状态确定性命令；禁止 owners、leases、events、ordinals、compat adapters
    或新的 lifecycle service。
12. PROTECTED story 逐 story 独立 High review；NORMAL 可 per-wave review；
    MECHANICAL 仅在没有任何语义变化时使用。
13. migration、shared hot files、SQLite writes 和 milestone verification 串行；
    默认最多两个实现 lane。
14. Release 永远最后；历史 receipt、summary 或 preservation commit 不能替代
    当前完整验证。
15. Superpowers discovery 调整只是可选外部优化，服从 system/developer authority，
    不是 cutover PASS 条件。
16. 进入 Git、restricted evidence archive 或报告的全部 bytes 必须先通过
    content-aware secret/credential/PII scan；path-only scan 只能辅助。历史
    evidence 必须保存到 Git 外的受限、内容寻址 archive，并生成不含敏感值的
    checksum manifest。
17. `PROTECTED` verification lane 必须在 exact commit 上独立重跑决定性检查，
    不能只读取 implementer 的日志或退出码。
18. `CURRENT_VERIFIED` commit/range 必须可从当前 lean integration SHA 到达；
    rebase、cherry-pick、冲突合并或其他操作一旦改变被审字节，原 review、
    verification 和 `CURRENT_VERIFIED` 即失效。
19. 冻结 catalog 的唯一 tracked 输入是 UTF-8 JSON
    `docs/product/v8/catalog.frozen.json`；其 bytes 必须与 quarantine source
    `artifacts/PD-POSTDEMO-V8-MITIGATION-TASK-MANIFEST-20260712-01/materialization/catalog.json`
    完全一致，SHA-256 为
    `72c849825c9cbd39cb25f743d448b67a2a31bfccf7cfb68a3d2557c7bda178bf`。
20. 不 push，不丢失用户改动，不把 archive checkpoint 冒充验收。
21. cutover 只有在 restricted archive/checksum、archive-only preservation、
    clean-HEAD active integration、精确 catalog/ledger、全部 visible dirty-path
    disposition、lean-governance 零 finding 独立复审、三个 canary 以及本节全部
    安全规则均通过后才完成；任一项缺失都不得进入普通 rolling wave。

## 1. 执行摘要

本项目过去约两周的主要问题不是 GPT-5.6 能力不足，也不是 Git 性能不足，而是
同时运行了两套完整的工作流系统：

1. 仓库自己的 `AGENTS.md`、manifest、taskctl、scope、evidence、owner 和 release
   协议；
2. Superpowers 的 brainstorming、writing-plans、TDD、subagent-driven、
   requesting-review、verification 和 finishing-branch 流程。

两套系统都试图完整管理从设计到验收的生命周期。GPT-5.6 的深推理能力又会继续
把每个异常制度化，最终形成：

```text
产品任务
→ scope/owner/evidence 异常
→ compatibility 治理任务
→ compatibility 的设计、计划、证据和复审
→ 新的 owner/successor 异常
→ 下一轮治理任务
```

这不是质量保证的自然成本，而是治理递归。

C3.1 的结论是：

> 推倒当前治理运行时和实施拆分方式；保留 V8 业务设计、客户/官方来源、产品代码、
> 历史证据和不可削弱的安全红线。用 Git commit 代替共享脏工作区中的 scope
> 重建，用可观察业务故事代替 283 次微任务闭环，用风险分级复审代替统一重量的
> 每任务复审。

新的不可约闭环是：

```text
权威业务规格
→ 一个可观察故事
→ 隔离工作区
→ 定向 TDD 和验证
→ Git commit
→ 风险分级独立复审
→ 集成
→ 283 行覆盖台账
→ Foundation / Full / Final / Release
```

## 2. 本次批准的边界

### 2.1 允许推倒重建

- 当前 manifest/digest/activation 驱动的治理运行时；
- `taskctl`、canonical scope、owner reconciliation 和 per-task evidence bundle
  作为新开发的必经路径；
- 283 个 catalog ID 等于 283 个执行和验收单元的假设；
- 每个普通任务固定执行 implementer、spec reviewer、code reviewer 和 final
  reviewer 的流程；
- Superpowers 自动发现和自动路由为完整开发操作系统的方式；
- 原实施计划中“不 commit，以 evidence bundle 作为 checkpoint”的执行策略；
- 旧 final/release gate 对每个旧 taskctl 状态逐项 PASS 的依赖。

### 2.2 明确保留

- 客户原始文档、官方来源及其版本和来源优先级；
- 已批准 V8 业务设计中的三线主流程和四个 deep modules；
- 所有现有产品代码、测试、任务文件和历史 evidence；
- 283 行 catalog 作为“不遗漏”的验收断言集合；
- 法律状态、期限、官费、费减、支付、服务应收、文件谱系、权限、安全、
  schema、migration、seed、破坏性操作和 SQLite 的 fail-closed；
- 实现者不能独立批准自己负责的受保护工作；
- Foundation、Full、Final 和 Release 的独立验收；
- Release 永远最后；
- 未决客户决定只阻塞相关 lane，不得猜测，也不得阻塞无关工作；
- 不 push，不丢弃用户改动，不把 preservation checkpoint 冒充产品验收。

### 2.3 不推倒的内容

本方案不是产品重写，也不是重新做客户需求分析。

以下内容继续作为权威输入：

- `docs/superpowers/specs/2026-07-12-fpms-postdemo-three-lane-mitigation-design.md`
- `docs/superpowers/plans/2026-07-12-fpms-postdemo-v8-mitigation-implementation.md`
  中的业务闭包、来源、依赖和验收意图，但不继承其微任务执行机制；
- `docs/agents/domain-safety.md` 中的实质性安全规则；cutover 时迁入新的 tracked
  V8 domain contract，原文件随后只读；
- `docs/agents/source-authority.md` 中的来源索引；cutover 时迁入新的 tracked
  V8 source/decision registry，原文件随后只读；
- Tasks 01–70 的已验收业务行为；
- V8 已实现代码和现有测试。

### 2.4 讨论和决策历程摘要

以下为本文件所汇总的过程脉络，不是新的启动前置条件：

1. V6 曾能完成 UI demo；V7/V8 阶段根据客户 Word、模板、XML、费率、收费场景和
   三线生命周期展示进行了多轮复审。
2. 复审形成了 V8 三线设计：案件生命周期居中，文书证据线和官费/服务费线并行，
   三者由显式事件、谱系和义务连接。
3. V8 计划被物化为 283 个 task，其中 197 个 Foundation、86 个 gated/full-only。
4. 为保护共享 dirty worktree，项目采用了 task-local baseline、scope、owner、
   evidence 和独立 gate，而不是 Git commit。
5. 长期执行中反复出现 controller 空跑、JSON 兼容、transport reconnect、scope
   长尾和 owner/successor 冲突。
6. 为解决这些问题，又建立 governance reset、scope v3、fast-close、legacy
   compatibility 和三十余个 Delta freeze/compatibility 闭环。
7. 多轮 High/Ultra 切换证明，大部分暂停并不需要 Ultra；High 能解决测试、代码、
   JSON 和 ordinary compatibility。
8. 后续治理已把 `AGENTS.md` 从约 678 行缩至约 98 行，但复杂度迁移到了
   `docs/agents`、taskctl、scope/evidence 脚本和大量 task-specific compatibility。
9. fast-close 降低了部分命令数量，但 canonical scope 仍持续占用 4–40 分钟。
10. 实测证明 Git 很快，瓶颈来自共享脏树上的 owner/accepted-tree 重建。
11. 初步 C2 提议改用 clean worktree、story cohorts、commit 和 skill diet。
12. 对 C2 的反证发现，“183 remaining”来自旧 catalog，且固定“25–40 cohorts”
    和 per-cohort receipt 仍可能重建新治理系统。
13. 对 SDD 的进一步考问明确：保留产品型 Spec-Driven Development；取消
    Superpowers 对每个微任务的强制全流程；Subagent-Driven 仅作为执行策略。
14. 对 C3 的进一步考问确认：方向已经足够，但必须补齐历史状态、review class、
    release supersession、安全 cutover 和 rollback。
15. 用户批准 C3.1，并要求把完整讨论与最终方案写入本文件。
16. 书面独立复审进一步发现：preservation commit 不能成为 active 产品基线，
    否则现有约 47k 行 dirty hunks 会逃逸后续 story review；因此最终采用
    archive-only preservation 与 clean-HEAD adoption commits 的双分支方式。
17. 两名只读独立复审者随后逐项核验 cutover 生命周期、secret/PII archive、
    historical PASS adoption、catalog identity、commit reachability 和
    PROTECTED exact-commit verification；第二轮修正后均未发现剩余 P0/P1。

## 3. 审计事实快照

以下数据来自 2026-07-28 的只读检查。它们用于解释为什么必须重置流程，不用于
直接宣称产品完成。

### 3.1 Git 和工作区

| 项目 | 观察值 |
| --- | ---: |
| 当前分支 | `master` |
| 当前 HEAD | `afa58429e6b6e80b85f76055139e18fbe38ec9e8` |
| 相对 `origin/master` | ahead 9 |
| 可见 dirty paths | 474（包含本综合文档） |
| tracked modified/other | 129 |
| untracked | 345（包含本综合文档） |
| tracked diff | `+41,418 / -6,157` |
| untracked 非忽略文件大小 | 约 3.59 MB |
| `artifacts/` | 约 359 MB 磁盘占用；约 278.5 MB 内容，33,804 个文件 |
| `.playwright-mcp/` | 约 453 MB，属于本地运行缓存 |

Git 原生操作不是瓶颈。此前实测：

- `git status` 约 0.02–0.03 秒；
- 约 2.2 MB 的完整 patch 约 0.07–0.13 秒。

### 3.2 V8 catalog 与状态

权威 catalog 仍是：

```text
283 total = 197 Foundation + 86 gated/full-only
```

当前现代 V8 receipt 扫描结果：

| 分类 | 数量 |
| --- | ---: |
| 有现代 terminal PASS receipt | 85 |
| Foundation 非现代 PASS | 112 |
| gated/deferred | 86 |
| 合计没有现代 terminal PASS | 198 |

112 个 Foundation 非现代 PASS 包括：

- 106 个没有现代 state；
- 4 个 `IMPLEMENTING`；
- 1 个 `READY_FOR_REVIEW`；
- 1 个 `BLOCKED`。

仓库全部现代 state 还包括治理和外部任务，因此不能直接拿 `210 PASS` 当作 V8
产品进度。旧任务 prose 状态同样不能作为权威：存在 receipt 已 PASS 但 prose
仍非 PASS，也存在 prose 写 PASS 但没有 terminal receipt 的情况。

此前使用的“剩余 183”来自旧的 `248 / 183 / 65` 计划数据，不是当前
`283 / 197 / 86` catalog 的可靠剩余量。本方案禁止继续引用“183 remaining”
作为执行基线。

### 3.3 历史 PASS 与当前树的关系

只读审计发现：

- 85 个 receipt-PASS catalog 项的 allowlist 都与当前 dirty/untracked 路径发生
  交集；
- 这些历史 patch 合计约 2.19 MB，覆盖 161 个当前 dirty 路径；
- 217 个 `state.json` 和 136 个 `fast-close-1` receipt 位于 ignored/untracked
  artifacts 中。

因此：

1. 历史 PASS 证明“某个历史候选曾被验收”；
2. 它不自动证明当前共享工作树仍满足同一行为；
3. 新 cutover 不得把 85 项直接标成 `CURRENT_VERIFIED`；
4. 当前整合树必须通过受影响故事回归和最终 milestone 验证重新获得当前有效性。

### 3.4 scope 和 evidence 性能

从现有 352 次成功 scope 记录得到：

| 指标 | 耗时 |
| --- | ---: |
| p50 | 100.95 秒 |
| p90 | 536.87 秒 |
| p95 | 1,287.21 秒 |
| 最大值 | 2,387.34 秒 |

典型长尾：

| 场景 | scope + candidate |
| --- | ---: |
| Legacy fee import | 约 41.34 分钟 |
| Filing XML legacy | 约 33.55 分钟 |
| Legacy provenance | 约 28.62 分钟 |
| Legacy document import | 约 23.80 分钟 |
| OA notice adapter | 约 9.42 分钟 |

主要根因：

- 一次 Legacy scope 可进行约 12 次完整 peer reconciliation；
- 每次扫描约 215 个 eligible owner；
- 每个 PASS peer 重建 accepted/current tree 并启动 owner-authority 子进程；
- 一次 prepare-review 重复计算约 7 次昂贵 snapshot key；
- 只有约 22 个 owner 与目标路径相交，但旧实现先验证全部 owner；
- artifacts 哈希、临时 index、dirty baseline 和 legacy overlay 相互叠加。

### 3.5 治理体量

| 项目 | 规模 |
| --- | ---: |
| 当前 `AGENTS.md` | 98 行 |
| `AGENTS.md + docs/agents` 活跃规则面 | 约 517 行 |
| `taskctl/evidence/scope/governance` 主要脚本 | 约 2.2 万行 |
| 主要 V8 design | 775 行 |
| 主要 V8 implementation plan | 995 行 |
| `docs/superpowers/specs` | 163 份（包含本综合文档） |
| `docs/superpowers/plans` | 157 份 |
| `tasks/repo` 治理任务 | 102 份 |

现代 PASS 状态中，粗分类约为：

- V8 catalog 产品：85；
- governance/Ultra：100；
- external/other：25。

治理产出已经超过产品产出。

### 3.6 当前激活并未失效

当前：

- `AGENTS.md` SHA-256：
  `37b587f47901eadd85f406a7bbabffa212b5f2d71ce10c03d21f5c08032bbbbf`
- `docs/agents/manifest.json` SHA-256：
  `a3db3079a1c3159432b55be4f7bc1484142e5f52c918ee2052ce0dd9020102bb`

二者与 activation state 的 `activation.kernel_sha256` 和
`activation.manifest_sha256` 精确匹配。某些历史字段中存在不同的 pre/current
manifest hash，它们不是当前激活失败的证据。

结论：当前慢是已激活治理的结构性问题，不是“治理尚未生效”。

## 4. 根因分析

### 4.1 直接根因：共享脏工作树取代 Git checkpoint

原实施计划明确禁止在每个任务后 commit，并要求：

- 捕获 dirty baseline；
- 扣除其他任务的修改；
- 重建 scoped patch；
- 扫描 owner 和 successor；
- 证明 candidate 未吸收其他任务；
- 用 artifacts 代替 Git 历史。

这一选择使 scope 系统必须重新实现 Git 已经提供的内容身份、隔离、差异和恢复
能力。

### 4.2 结构根因：微任务边界小于 deep module 的行为边界

V8 设计正确地提出 Lifecycle、Document Evidence、Fee Obligation 和 Overlay
四个 deep modules，但实施计划把它们继续拆成：

- 一个 lifecycle event 一个 task；
- 一个 API endpoint 一个 task；
- 一个 adapter 一个 task；
- 一个表或 carrier 一个 task；
- 一个旧表单一个 manifest activation task；
- 一个 QA 检查一个 task。

这导致同一接口、同一实现文件和同一测试面被重复打开、关闭、scope 和复审。

典型共享热点：

- `backend/scripts/seed_dev.py`：27 个 catalog row；
- `backend/app/modules/documents/official_notice_catalog.py`：24 个 row；
- `backend/app/modules/cases/lifecycle_rules.py`：24 个 row；
- `backend/app/modules/fees/official_rate_book.py`：11 个 row。

### 4.3 流程根因：两套 workflow operating system 叠加

当前 `AGENTS.md` 要求“task-appropriate Superpowers workflow”，而 Superpowers
又要求：

- creative work 必须 brainstorming；
- brainstorming 后必须设计文档、commit、spec review、用户 review；
- 然后 writing-plans；
- 执行中每个 task 使用 implementer、spec review、code review；
- 完成前再次 verification；
- branch 收口再执行 finishing workflow。

仓库自身同时要求 task file、evidence、scope、review、task gate、atomic evidence
和 release gate。两套系统职责重叠，并互相放大。

### 4.4 模型根因：为旧模型设计的外部脚手架叠加 GPT-5.6 深推理

GPT-5.0 时代，细粒度任务和外部证据可以弥补模型稳定性不足。GPT-5.6 更擅长：

- 长上下文理解；
- 跨文件一致修改；
- 复杂测试诊断；
- 在冻结合同内自主完成完整故事。

继续把每个不确定点物化为新治理任务，会让强模型把治理细节继续制度化，形成
“双重放大”，而不是“双重保险”。

### 4.5 工具根因：JSON 和 transport 状态成为业务关键路径

过去反复出现：

- JSON quoting/schema 兼容错误；
- candidate/reviewer binding 格式错误；
- owner/successor 兼容；
- controller 长时间 running 但无 diff/artifact；
- transport disconnect 后重复已完成步骤；
- scope 进程长时间扫描；
- summary prose 与 state receipt 不一致。

这些问题来自自定义控制面过大。C3.1 不再把自定义 JSON 状态机放在产品开发的
关键路径。

## 5. 方案演进与选择

### 5.1 方案 A：继续热修 scope 引擎

可能措施：

- 一次 scope 共享 reconciliation context；
- 先筛选相交 owner；
- invocation-local memoization；
- scope 结果直接携带 snapshot key；
- 合并 Legacy 三分区扫描。

预期可把 scope 降至约 10–40 秒，甚至个位数秒。

拒绝作为最终方案的原因：

- 仍然保留最多 198 个非现代 PASS row 的逐项固定关闭成本；
- 仍保留 owner/successor/receipt compatibility 架构；
- 仍然鼓励为下一类异常继续扩展 scope engine；
- 解决性能，不解决治理递归。

### 5.2 方案 B：C2 Lean 5.6 Reset

C2 的改进：

- preservation snapshot；
- clean worktree；
- story cohort；
- cohort-level TDD、commit、review 和 receipt；
- old taskctl/evidence read-only；
- lean AGENTS 和 skill diet。

C2 的剩余问题：

- “25–40 cohorts”是新的任意规划配额；
- 每个 cohort 一份 receipt 可能重建新的 evidence 系统；
- 仍可能为 lean governance 建 activation、compatibility 和 release adapter；
- 初版错误使用“183 remaining”；
- 未充分处理 ignored receipt 和历史 PASS 与当前 dirty tree 的交叉；
- 默认每 cohort 独立 review 仍可能对普通工作施加统一过程税。

### 5.3 方案 C：C3.1 Git-native Lean 5.6 Reset

C3.1 不再创建新的治理运行时。

选择理由：

- Git commit 已经是稳定 candidate identity；
- isolated worktree 已经提供 scope 隔离；
- `git show` 已经提供精确 reviewer diff；
- targeted tests 已经提供行为证据；
- commit 和受控 review 已经提供断线恢复；
- 一个轻量覆盖台账足以保留 283 行不遗漏责任；
- 复杂度集中在产品 deep module，而不是开发流程。

## 6. C3.1 目标运行模型

### 6.1 权威层级

新权威层级：

1. 当前 system/developer/user 指令；
2. 精简 `AGENTS.md`；
3. V8 domain contract 和 source/decision registry；
4. 当前 story card；
5. 代码、测试和 commit；
6. 非规范性的历史计划、task files、artifacts 和 examples。

不再存在：

- governance manifest activation 前置条件；
- governance digest 自举；
- rule owner selector；
- task owner accepted-tree 重建；
- candidate hash triple；
- successor overlay；
- ordinary task 的 atomic evidence bundle。

### 6.2 活跃治理面

目标活跃治理只包含：

| 文件/机制 | 目的 |
| --- | --- |
| 一个约 30–50 行的 `AGENTS.md` | 执行、安全、复审和 release 红线 |
| `docs/product/v8/domain-contract.md` | 从旧 domain-safety 提取的法律、费用、期限、谱系、权限和数据不变量 |
| `docs/product/v8/source-decision-registry.md` | 从旧 source-authority 迁入的客户和官方来源、版本、决定及未决 gate |
| `docs/product/v8/catalog.frozen.json` | 唯一、只读、哈希绑定的 283-row catalog |
| `docs/product/v8/coverage-ledger.json` | 283 row 到 story/commit/test/disposition 的映射 |

旧 `docs/agents/**`、`scripts/taskctl`、`scripts/evidence_scope.py`、
`scripts/evidence_validate.py`、compat specs/plans 和 `tasks/repo/REPO-*` 保留为
只读历史，不删除、不继续扩展。旧 domain/source 文件只在新 tracked contract
完成逐条迁移和独立复审前继续提供输入；adoption commit 通过后不再是活跃入口。

### 6.3 Story 是唯一普通执行单元

一个 story 至少包含：

- `story_id`；
- 业务 outcome；
- explicit non-goals；
- 对应的旧 catalog IDs；
- authority/source/decision refs；
- dependencies 和 customer gates；
- 预计修改的 deep module seam 和路径；
- observable tests；
- review class；
- rollback boundary。

Story 合并条件：

- 共享同一个可观察用户结果；
- 共享同一个 deep module interface；
- 共享相同权威来源和风险等级；
- 可以一起回滚；
- 测试可以从同一个外部 seam 证明。

Story 强制拆分条件：

- 不同 migration；
- 不同客户决定；
- 不同法律或官方费率来源；
- 不同独立回滚边界；
- 不同权限/安全边界；
- 同时修改相互冲突的共享文件；
- 一个 story 无法用单一清晰 outcome 描述。

不设置“每个 story 必须包含 3–12 个旧 ID”或“总数必须 25–40”。

### 6.4 TDD 和验证

新行为或 bugfix：

```text
可观察 RED
→ 最小实现
→ focused GREEN
→ 受影响回归
→ scoped lint/type/diff
```

继承 WIP 或已经存在实现的 correction：

- 不删除已有实现来人为制造 RED；
- 先用失败测试、现有失败、差异重放或已保存 evidence 证明缺口；
- 只重跑受修复影响的最小验证；
- 在 milestone 再执行广测。

禁止：

- 为每个新函数强制单独测试；
- 为了形式化 RED 删除正确代码；
- 邻接重构、统一改名、全仓格式化；
- 未到 milestone 提前运行 repo-wide check。

### 6.5 复审分级

#### PROTECTED

以下为受保护故事：

- 法律状态和案件生命周期；
- 官方期限和 deadline preview；
- 官费、费减、滞纳金、支付、官方凭证；
- 服务费和服务应收；
- 文书/附件/回执/派生/版本谱系；
- authentication、authorization、permission、401/403；
- schema、migration、seed、destructive operation；
- SQLite compatibility；
- 客户决定和来源 activation；
- Foundation、Full、Final、Release。

要求：

- implementer 不能批准自己；
- 每个 story 的精确 commit/range 必须独立 High review；
- reviewer 检查来源、fail-closed、observable tests、权限/事务/谱系；
- 独立 verification lane 必须在 exact commit 上重新运行决定性的 PROTECTED
  checks；不能只读取 implementer 写下的命令和 exit status；
- P0/P1/P2 均为零后才可集成。

#### NORMAL

冻结合同内、没有改变上述语义的普通 API/UI/adapter/story：

- 每个 story 必须 focused GREEN；
- 一个 wave 可由同一独立 reviewer 一次审查多个明确 commit；
- reviewer 必须逐 story 给出 verdict；
- 不能用代表样本覆盖未列出的 story。

#### MECHANICAL

纯格式、索引、无运行时文档、显然机械的更新：

- 精确 check 通过即可；
- 不要求独立 reviewer；
- 不得借机械分类吸收产品语义变化。

任何未知、混合、有争议或无法从冻结合同明确分类的 story，一律按
`PROTECTED`。不得为了提速把 auth、fee、lineage、migration、customer/source
语义降为 `NORMAL` 或 `MECHANICAL`。

### 6.6 证据模型

普通 story 的 durable evidence 仅包括：

- exact commit SHA 或 commit range；
- covered catalog IDs；
- exact verification commands；
- observed exit status；
- review class；
- independent reviewer/verdict（如适用）；
- 未关闭 residual。

记录方式：

- commit 是代码和 scope 的权威；
- 每个 wave 允许一份简洁 review/verification record；
- `docs/product/v8/coverage-ledger.json` 记录 row → story → commit → tests；
- 失败日志按需保存，成功命令不再生成几十份重复 artifact；
- 不复制完整 worktree、baseline、peer artifact tree。

本地没有 CI 时：

- 使用当前 commit 上的新鲜本地验证；
- NORMAL reviewer 核对 commit 和验证记录；
- PROTECTED verification lane 在 exact commit 上独立重跑决定性 tests/checks，
  reviewer 再核对结果；
- milestone 重新执行累计验证；
- Final/Release 必须重新运行完整 gate，不能只相信历史命令文本。

coverage ledger、wave record、milestone report 和 checker 都只是普通 Git-tracked
文档或无状态、确定性的命令。它们不得拥有 controller、mutable lifecycle、
events、owners、leases、ordinals、candidate generation 或 compatibility adapter；
不得形成第二套 taskctl。

未来若启用远程 CI：

- CI 结果绑定 commit SHA；
- branch protection/CODEOWNERS 可承接独立验收；
- CI 建设不是 C3.1 cutover 的前置项目。

### 6.7 并行策略

默认：

- 2 个实现 lane；
- 1 个独立 review/verification lane；
- 主线程负责 integration、dependency 和 conflict 检查。

仅在明确无冲突时启用第 3 个实现 lane。

始终串行：

- migration；
- schema/seed；
- SQLite-writing tests；
- shared router/registry；
- 同一 deep module 热点文件；
- Foundation/Full/Final/Release；
- repo-wide/full E2E。

并发不是目标；最短关键路径才是目标。

### 6.8 High 与 Ultra

- GPT-5.6 High 是默认且强制的实现、诊断、测试和 review 档位；
- 普通测试失败、JSON、scope、兼容、工具调用和代码缺陷不得作为 Ultra 理由；
- 只有冻结合同、权威来源、客户决定和代码都无法消解的重大法律、官费、期限、
  谱系、schema/migration 或架构语义歧义，才允许建议 Ultra；
- 建议 Ultra 前必须列出精确冲突、已检查来源、受影响 lane 和最小 freeze；
- 未受影响 lane 继续；
- 未经用户确认不自行切换。

## 7. SDD 在 C3.1 中的位置

### 7.1 Spec-Driven Development 继续有效

C3.1 保留：

- V8 业务设计规定“做什么”和“为什么”；
- source/decision registry 规定事实从哪里来；
- story card 规定当前可观察 outcome 和 non-goals；
- tests 规定可以执行的行为合同；
- coverage ledger 证明规格断言没有遗漏。

语义或来源变化时：

```text
先更新 domain/source contract
→ 独立确认
→ 再更新 story 和实现
```

冻结合同内的正常实现不重新 brainstorm、design 或 plan。

### 7.2 Subagent-Driven Development 改为策略而非治理

子代理继续用于：

- 真正无冲突的并行实现；
- PROTECTED story 的独立 review；
- 定向来源核验；
- 性能/故障诊断；
- final audit。

不再要求每个微任务固定创建：

- implementer；
- spec reviewer；
- code reviewer；
- final reviewer。

主线程可以直接完成普通故事；implementer 仍不能独立批准自己的 PROTECTED 工作。

### 7.3 Superpowers 的处理

建议：

- 移除 `/Users/cfcc/.agents/skills/superpowers` 自动发现 symlink；
- 保留 `/Users/cfcc/.codex/superpowers/skills` 源码；
- 不删除、不过度修改 Superpowers；
- 需要时可恢复 symlink 或显式选择单个能力。

移除的是自动接管，不是 TDD、review、diagnose 或多代理能力。

独立的 `diagnose`、`review`、轻量 TDD 和领域技能继续按需使用。任何 skill 不得
在冻结产品 story 上自行增加第二套设计、计划、审批和验收流程。

## 8. 283 行覆盖台账

### 8.1 台账目的

coverage ledger 只回答：

1. 283 行是否每行恰好出现一次；
2. 每行当前是什么 disposition；
3. 哪个 story/commit/test 覆盖该行；
4. 是否存在客户/来源 blocker；
5. Final/Release 是否允许通过。

它不是：

- task runner；
- owner registry；
- scope engine；
- artifact scanner；
- agent state machine；
- 新的治理 manifest。

### 8.2 建议 disposition

| 状态 | 含义 |
| --- | --- |
| `HISTORICAL_PASS_CANDIDATE` | 有历史 terminal receipt，但当前整合树尚未重新验证 |
| `INHERITED_EVIDENCE` | 由 Tasks 01–70 或其他批准基线提供历史行为证据 |
| `CURRENT_VERIFIED` | 已映射到当前 lean integration commit 和新鲜验证 |
| `WIP` | 当前树包含未完成或未验收实现 |
| `PENDING` | 依赖满足但尚未进入 story |
| `CUSTOMER_BLOCKED` | 缺少明确客户决定，仅阻塞对应 lane |
| `AUTHORITY_BLOCKED` | 缺少法律/官方来源或来源冲突 |
| `SUPERSEDED_BY_STORY` | 旧微任务由明确 story、commit 和 tests 完整替代 |
| `DEFERRED_FULL_ONLY` | 按批准设计属于 Full/decision-gate lane |

限制：

- `SUPERSEDED_BY_STORY` 不是 PASS；只有关联 story `CURRENT_VERIFIED` 后才满足覆盖；
- `HISTORICAL_PASS_CANDIDATE` 不是当前 release 证明；
- 不允许无依据的 `NOT_APPLICABLE`；
- 一个 catalog ID 不能映射到两个相互独立的 story；
- 一个 story 可以映射多个 ID，但必须逐行列出 observable test；
- 未决客户项不能伪造成 safe default 的完成。

### 8.3 最小机器检查

允许一个小型、确定性的 checker：

- 输入仅为 UTF-8 JSON `docs/product/v8/catalog.frozen.json` 和
  `docs/product/v8/coverage-ledger.json`；
- frozen catalog 必须逐 byte 等于
  `artifacts/PD-POSTDEMO-V8-MITIGATION-TASK-MANIFEST-20260712-01/materialization/catalog.json`
  的 cutover source copy，并绑定 SHA-256
  `72c849825c9cbd39cb25f743d448b67a2a31bfccf7cfb68a3d2557c7bda178bf`；
- 验证 283 个唯一 ID、合法 disposition、commit 引用和无重复覆盖；
- 验证每个 `SUPERSEDED_BY_STORY` 都解析到 `CURRENT_VERIFIED` story、当前 commit、
  observable test，以及适用的 PROTECTED review；
- 验证每个 `CURRENT_VERIFIED` commit/range 可从本次检查所声明的当前 lean
  integration SHA 到达；
- 如果 rebase、cherry-pick、冲突合并或其他操作改变了被审字节，要求重新验证和
  复审，不接受原 commit 上的 verdict；
- 按 milestone 验证 disposition eligibility；
- 不扫描 worktree owner；
- 不读取完整 artifacts；
- 不启动 taskctl；
- 不重建 accepted tree；
- 正常目标耗时小于数秒。

checker 只读输入并向 stdout 返回结果；除退出码外不写 state。不得把它扩展为
新的控制面。

Milestone eligibility：

| Disposition | Foundation | Full | Final/Release |
| --- | --- | --- | --- |
| `CURRENT_VERIFIED` | 允许 | 允许 | 允许 |
| `SUPERSEDED_BY_STORY` | 仅当 successor 当前验证完成 | 同左 | 同左 |
| `HISTORICAL_PASS_CANDIDATE` | 阻塞 | 阻塞 | 阻塞 |
| `INHERITED_EVIDENCE` | 仅作回归输入，不单独放行 | 同左 | 同左 |
| `WIP` / `PENDING` | 阻塞 | 阻塞 | 阻塞 |
| `CUSTOMER_BLOCKED` / `AUTHORITY_BLOCKED` | 仅非 Foundation lane 可隔离 | 阻塞对应 Full | 阻塞最终完成 |
| `DEFERRED_FULL_ONLY` | 可排除 Foundation | 必须先转为当前验证或明确 blocker | 阻塞最终完成 |

## 9. 安全 cutover

### 9.1 Phase 0：停写和取证

在任何写操作前：

- 停止旧 taskctl/scope/evidence 进程；
- 确认没有 SQLite-writing test、migration、frontend build 或 Playwright 在运行；
- 记录 Git HEAD、branch、status 和 visible paths；
- 记录 217 个 state、136 个 fast-close receipt 和 activation receipt 的摘要；
- 对将进入 Git 的全部 visible bytes 执行 content-aware secret/credential/PII scan；
  scanner 不得把命中的敏感值原样输出到日志；
- path-only scan 只作辅助，不能替代内容扫描；
- 记录 ignored `.env`、数据库、附件和本地缓存，只做排除清单，不读取敏感值；
- 不删除 stale lock；先确认没有对应 live process。

### 9.2 Phase 1：原工作区作为 quarantine source

原始工作区：

- 保持当前 dirty bytes；
- 保留 ignored artifacts；
- 不 clean、reset、stash、checkout 或覆盖；
- 不在其中继续产品开发；
- 作为 cutover 的恢复源和历史 evidence 仓库。

历史 evidence 不能永久只有这一份。cutover 还必须：

- 建立 restricted、content-addressed evidence archive；
- archive 位于 Git 之外、用户批准的位置，并使用最小文件权限；
- 默认只收录 terminal state、acceptance receipt、review、scoped patch 和被结果引用的
  必要日志；
- 先执行 secret/PII 检查；`.env`、数据库、客户附件和原始敏感材料不得自动打包；
- 生成不含秘密内容的 checksum manifest；
- archive 路径和 checksum 写入 cutover report，但 archive 内容不进入 Git。

如果没有获得安全 archive 位置或内容扫描失败，停止 cutover；不能以“原目录还在”
代替 durable evidence preservation。

### 9.3 Phase 2：archive-only preservation worktree

从固定 commit：

```text
afa58429e6b6e80b85f76055139e18fbe38ec9e8
```

创建 archive sibling worktree/ref，并把 quarantine 中的 visible nonignored 开发树
复制进去。

只迁移：

- tracked modifications；
- Git 未忽略的 untracked source/test/docs/task/reference 文件；
- 必须纳入新树的 migration 和数据文件。

明确排除：

- `.env` 和 credentials；
- `*.db`、`*.sqlite*`；
- `backend/storage/attachments/**`；
- `artifacts/**`；
- `.playwright-mcp/**`；
- `node_modules/**`；
- local caches、reports、临时锁。

迁移后：

- 对全部 visible nonignored paths 做路径、mode、size、hash 对账；
- archive worktree 与 quarantine source 的可见开发树必须一致；
- 对 staged bytes 再执行 content-aware secret/PII scan；
- 不一致即停止，不猜测、不继续 commit。

随后仅在 archive ref 创建：

```text
checkpoint: preserve V8 pre-lean state (not acceptance)
```

该 commit：

- 只证明当前可见开发树被保存；
- 不证明测试通过；
- 不把六个非终态任务关闭；
- 不把 85 个历史 receipt 升级为当前 PASS；
- 不包含 secrets、数据库、附件或 artifacts；
- 不 push。

archive ref 永远不是 active integration branch 的父提交。它只提供：

- 当前可见开发树的恢复点；
- adoption story 的来源；
- dirty hunk 对账输入。

### 9.4 Phase 3：建立 active lean integration

另建 active sibling worktree/branch，直接从 fixed clean HEAD
`afa58429e6b6e80b85f76055139e18fbe38ec9e8` 开始。

active branch：

- 不继承 preservation commit；
- 初始不包含当前 uncommitted 产品代码；
- 每一段 dirty 产品行为只能通过后续 adoption story commit 进入；
- reviewer 的 commit/range 必须展示被导入的 preservation hunks；
- shared file 中若包含多个不相干故事，按 hunk 拆分或由一个明确 coherent story
  完整承担，不能整文件静默吸收。

### 9.5 Phase 4：一次性事实和 dirty-path 迁移

生成 coverage ledger 初始版本：

- 283 行恰好一次；
- 85 个现代 receipt 项标为 `HISTORICAL_PASS_CANDIDATE`；
- 112 个 Foundation 非现代 PASS 项逐项分类；
- 86 个 gated/deferred 项保留明确 gate；
- 4 个 IMPLEMENTING 和 1 个 READY_FOR_REVIEW 标为 `WIP`；
- V8 的 1 个 BLOCKED 项保留 blocker；
- repo governance BLOCKED 不计入产品 catalog；
- Tasks 01–70 映射为 `INHERITED_EVIDENCE`，但不重复计算 catalog row；
- 任何 prose/receipt 冲突写入 reconciliation note。

这是一次性 cutover 事实迁移，不是 283 次 task closure。

同时生成 visible dirty-path disposition：

- 每个 path 恰好属于一个 adoption story、governance/history archive-only、
  local-excluded 或 generated-excluded 分类；
- 每个产品 path 记录对应 story 和预期 commit；
- 非 catalog path 也必须分类，不能因为没有旧 task ID 而遗漏；
- path classification 只是迁移清单，不替代 hunk review；
- 所有 85 个 receipt-PASS row 都必须通过当前 adoption tests 和相关 hunks review
  后才能满足 Foundation；不要求重做历史 RED，也不允许只引用旧 receipt。

### 9.6 Phase 5：Lean governance direct adoption

用户的本次明确批准是旧治理之上的高位授权。新治理采用：

- 一个精简 `AGENTS.md`；
- 新 tracked domain contract；
- 新 tracked source/decision registry；
- coverage ledger；
- Git-native execution。

不得：

- 用旧 taskctl 为新治理创建 activation task；
- 为 activation 再建 design/plan/materialization/compat tasks；
- 要求新治理验证自己的旧 manifest digest；
- 因 sibling worktree 不含 ignored activation receipt 而重新进入旧 bootstrap。

Lean governance commit 必须独立审查，但只审：

- 是否保留所有安全红线；
- 是否正确废止旧控制面；
- 是否保留 283 行覆盖责任；
- 是否保持旧 evidence read-only；
- 是否具备明确 rollback。

### 9.7 Phase 6：按 story 导入当前 dirty 产品修改

adoption 顺序根据实际依赖和 rollback boundary 决定，通常先：

1. schema/migration foundations；
2. lifecycle/document/fee deep-module contracts；
3. services/rules；
4. API/FE adapters；
5. UI 和 E2E；
6. legacy reconciliation。

每个 adoption story：

- 从 archive ref 提取精确 path/hunks；
- 在 active story worktree 中形成可见 diff；
- 运行当前行为 tests；
- PROTECTED verification lane 在 exact commit 独立重跑决定性检查；
- reviewer 审查导入 hunks 与任何 correction；
- 集成后确认 exact commit/range 可从新的 lean integration SHA 到达，且集成未改变
  被审字节；改变字节则重新验证和复审；
- 最后更新 coverage ledger 和 dirty-path disposition；
- 集成后不再依赖 archive parent。

六个当前非终态产品任务只作为 WIP 来源；它们必须被 adoption story 当前验证，
不得走旧 taskctl 机械 close，也不得因进入 archive commit 而视为完成。

### 9.8 Phase 7：可选的 Superpowers 自动发现调整

在用户/系统批准外部文件操作后：

- 只移除自动发现 symlink；
- 不删除源码；
- 记录原路径和 target；
- 验证其他独立 skills 仍可发现；
- 下一会话确认 Superpowers 不再自动进入 mandatory stack。

该操作是可逆的外部优化，不影响仓库历史，也不是 C3.1 cutover、canary 或产品
PASS 的完成条件。若 system/developer 仍发现或要求相关 skill，以更高 authority
为准；repository policy 不声称可以覆盖它。

## 10. Canary 与速度门槛

### 10.1 三个 canary

选择：

1. 一个 NORMAL 后端故事；
2. 一个 PROTECTED 法律/费用/谱系故事；
3. 一个前端/UI 或 BE/FE 纵向故事。

每个 canary 必须：

- 从 clean story branch/worktree 开始；
- 使用明确 story card；
- targeted TDD；
- commit；
- 按 review class 复审；
- 集成到 lean branch；
- 确认 exact commit/range 可从新的 lean integration SHA 到达，且集成未改变被审
  字节；
- 最后更新 coverage ledger；若集成改变字节，先使旧 verdict 失效并重新验证/复审。

### 10.2 成功标准

| 指标 | 目标 |
| --- | --- |
| Git scope/diff 生成 | p95 小于 5 秒 |
| custom owner/scope scan | 0 次 |
| 新 taskctl/evidence artifact | 0 |
| 非测试、非复审的流程管理开销 | 每 story 中位数小于 5 分钟 |
| transport 恢复 | 从 Git/process 状态恢复，不重复已完成 commit |
| canary review | P0/P1/P2 为零 |
| 旧 catalog 覆盖 | 每个 story 精确更新映射，无遗漏/重复 |

如果 canary 未达标：

- 先修精确、可测的本地瓶颈；
- 最多一次流程调整；
- 不创建新的治理项目；
- 不回到 per-task scope engine；
- 产品安全问题必须先修，不能用性能理由绕过。

### 10.3 预期改善

流程开销将从：

```text
每个 catalog ID：
scope 4–40 分钟
+ candidate
+ reviewer binding
+ accept
+ artifacts
```

变为：

```text
每个业务 story：
实际测试耗时
+ Git commit
+ 风险分级 review
+ 一次台账更新
```

真实 ETC 在三个 canary 后重新估计。禁止在事实迁移前继续使用旧“183 remaining”
推算工期。

## 11. 持续开发波次

### 11.1 Rolling wave，而非一次性重规划

不预先 materialize 所有 story。

每个 wave：

1. 从 coverage ledger 选择 dependency-ready 项；
2. 按 deep module seam 聚合成最少必要 story；
3. 检查 source/customer gate；
4. 检查文件、migration、SQLite 冲突；
5. 启动最多两个默认实现 lane；
6. story commit 后立即进入 review；
7. reviewer slot 完成后立即复用；
8. wave 集成后更新 ledger 和下一 wave。

一次只规划当前 wave 和紧邻依赖，不重新写完整项目计划。

### 11.2 建议的高层故事域

这些是路由域，不是固定数量承诺：

- Lifecycle 与法律状态；
- Document/Evidence 与工作包；
- Filing/OA/Grant workflow；
- Fee obligation、fee reduction 与 annuity；
- PayList、payment 与 official evidence；
- Customer decision/source activation；
- Legacy migration/reconciliation；
- Three-lane overlay 与 UI；
- Foundation/Full/Final/Release。

### 11.3 共享热点处理

对于 `seed_dev.py`、`lifecycle_rules.py`、`official_notice_catalog.py`、
`official_rate_book.py` 等热点：

- 同一时间只有一个 owner；
- 尽量在一个 coherent story 中一次完成同来源、同接口的表驱动行为；
- 不为每一行数据或每一事件类型重复开 task；
- 不因聚合而跨越不同法律来源、customer gate 或 rollback boundary。

## 12. Foundation、Full、Final 和 Release

### 12.1 Foundation

Foundation 必须证明：

- 所有 Foundation-required catalog row 已由当前 story/commit/test 覆盖；
- 当前 85 个 `HISTORICAL_PASS_CANDIDATE` 全部已通过相应 adoption story 审查
  preservation hunks，并在 active commit 上运行当前验证；旧 receipt 只作参考，
  不要求重做历史 RED；
- schema/migration/seed/SQLite contract 通过；
- lifecycle、document、fee 三条主线的关键集成通过；
- 未决 customer lanes 明确隔离；
- 无 required Foundation row 仅靠 prose 或旧 summary 宣称完成。

### 12.2 Full

Full 只能纳入：

- 客户决定已确认；
- source 已激活；
- gate scope/version/effective time 明确；
- 对应 story 已 `CURRENT_VERIFIED`。

缺少客户决定时：

- 仅对应 lane 保持 `CUSTOMER_BLOCKED`；
- 其他 lane 继续；
- 不猜测；
- 不宣称 Full complete。

### 12.3 Final

Final 独立审计：

- 283 行唯一且无遗漏；
- 所有 required row 有当前 commit/test；
- superseded row 有精确 successor story；
- 所有 PROTECTED story 有独立 review；
- 无未解释 WIP；
- 无 secrets/PII evidence 泄露；
- clean migration、backend、frontend 和真实 E2E 通过；
- residual 与 customer blockers 如实报告。

### 12.4 新 release gate

旧 release gate 依赖每个旧 taskctl PASS，必须由 lean release gate 取代。

新 release gate 输入：

- tracked frozen catalog `docs/product/v8/catalog.frozen.json`，其 SHA-256 必须为
  `72c849825c9cbd39cb25f743d448b67a2a31bfccf7cfb68a3d2557c7bda178bf`；
- `docs/product/v8/coverage-ledger.json`；
- lean integration commit；
- Foundation/Full/Final 的 Git-tracked verification/review reports；
- 当前完整测试结果；
- migration/seed/SQLite 结果；
- frontend lint/type/build；
- named Playwright/real E2E；
- independent Final review。

release eligibility 必须遵循第 8.3 节的 disposition 表。任何
`HISTORICAL_PASS_CANDIDATE`、`WIP`、`PENDING`、未解析 blocker 或没有解析到
`CURRENT_VERIFIED` story 的 `SUPERSEDED_BY_STORY` 都阻止 Release。
任何 `CURRENT_VERIFIED` commit/range 若无法从当前 lean integration SHA 到达，
或者当前可达字节与被独立验证/复审的字节不同，也阻止 Release。

新 release gate 不得：

- 遍历旧 owner tree；
- 调用 canonical scope；
- 要求被 `SUPERSEDED_BY_STORY` 的旧 taskctl 写 PASS；
- 读取全部历史 artifacts 计算 current scope；
- 在 Final 之前运行。

Release 永远最后。

## 13. Transport、agent 和 Goal 恢复

### 13.1 Transport failure

断线后按顺序检查：

1. 当前 branch/worktree；
2. `git status`；
3. 最新 commit；
4. test/build 进程；
5. reviewer/agent 状态；
6. coverage ledger；
7. 第一个未完成 story step。

不得：

- 重复已有 commit；
- 重跑已完成且未受影响的验证；
- 重新分析冻结 source/design；
- 仅因没有新 diff 就中止正在运行的测试；
- 用新 agent 完整复制超长历史。

### 13.2 Agent liveness

- agent prompt 使用最小上下文；
- 每个 implementer 一次只负责一个 story；
- review agent 只接收精确 commit/range 和 story card；
- 无 diff 但有 active test/process 不算 stall；
- 两次相隔至少 30 秒且总计至少 90 秒没有 process、diff、commit、log 或消息进展，
  才可判断 stall；
- stall 后最多替换一次；
- replacement 再 stall，由主线程接管或报告，不递归创建 agent。

### 13.3 Goal

Goal 只表达：

```text
完成 V8 剩余开发、验收和 release close
```

Goal 不保存 283 个微任务状态。Git 和 coverage ledger 才是 durable truth。

- 不因一个 story/wave 完成而停止 Goal；
- 一个 lane blocked 时继续其他 lane；
- 只有 required work 全部完成才标记 complete；
- 只有全部剩余工作都被真实外部 blocker 阻断，才标记 blocked；
- registry 缺失时不得凭空声称已有 Goal；需按用户明确指令创建或恢复。

## 14. 回滚和恢复

### 14.1 回滚资产

- 原 quarantine workspace；
- 原 ignored artifacts；
- restricted content-addressed evidence archive 及 checksum manifest；
- pinned pre-cutover HEAD；
- archive-only preservation checkpoint；
- active clean-HEAD integration ref；
- lean governance commit；
- visible dirty-path disposition；
- coverage ledger 初始版本；
- 每个 story commit；
- wave integration commits。

### 14.2 回滚场景

#### cutover 复制不一致

- 停止；
- 删除或放弃 archive sibling 前先保存诊断；
- 原 workspace 不受影响；
- 修正复制规则后重试一次。

#### Lean governance 缺少安全红线

- 不启动产品 canary；
- 修正同一治理 commit；
- 重新独立 review；
- 不回到旧 taskctl activation。

#### Canary 质量失败

- 修产品 story；
- 重新 review；
- 不削弱 PROTECTED 规则；
- 不因一次失败恢复全部旧流程。

#### Canary 流程性能失败

- 测量精确瓶颈；
- 只允许一个小型工具修复；
- 不增加新 owner/scope/evidence 状态机。

#### C3.1 整体不可用

- 停止 lean branch；
- 原 quarantine workspace 和 artifacts 保持可用；
- archive-only preservation commit 提供完整可见开发树；
- active branch 可直接回到 fixed clean HEAD，不包含隐藏的 dirty parent；
- 用户决定回退或采用其他方案；
- 不自动覆盖原 `master`。

## 15. 明确禁止的反复模式

以后不得再次：

- 为治理 bug 建设计、计划、materialization、compat 和 activation 五层任务；
- 为普通工具 JSON 失败创建产品外治理项目；
- 用任意 cohort 数量作为完成指标；
- 把所有不确定性升级 Ultra；
- 为冻结 story 重复客户资料分析；
- 为每个旧 task ID 单独生成 scope/evidence/review；
- 在共享 dirty tree 中依赖 baseline subtraction 区分并发 owner；
- 把 summary prose 当作 terminal authority；
- 把历史 PASS 当作当前整合树 PASS；
- 把 preservation commit 当作验收；
- 为“未来可能需要”预建抽象、checker 或 adapter；
- 在 release 之前运行 release gate；
- 因用户暂时离开而停止可安全推进的 lane。

## 16. 质量为什么不会削弱

C3.1 删除的是重复证明和控制面，不是产品验证。

| 质量目标 | 旧机制 | C3.1 |
| --- | --- | --- |
| 精确 scope | baseline subtraction + owner scan | isolated worktree + commit diff |
| 内容身份 | candidate hashes | commit SHA |
| 行为验证 | per-task logs | targeted tests + milestone tests |
| 独立验收 | 每任务统一 review | PROTECTED per-story，NORMAL per-wave |
| 不遗漏 | 283 task states | 283-row coverage ledger |
| 断线恢复 | JSON state/ordinal | branch、commit、process、ledger |
| 法律/费用安全 | domain rules | 原样保留并加强可执行测试 |
| 最终质量 | taskctl/release scripts | complete tests + independent Final + lean release |

减少约束数量可以提高质量，因为：

- reviewer 更专注于产品语义而不是 hash binding；
- commit 隔离消除 shared dirty scope 歧义；
- story 对齐 deep module interface，测试更接近真实行为；
- governance code 变少，工具 bug 和 compatibility 分支变少；
- 广测在正确 milestone 执行，不被数百次重复小验收稀释。

## 17. 实施顺序

本综合设计获文件确认后，只执行以下一次性切换包：

1. quiesce 和只读 inventory；
2. content-aware secret/PII scan；
3. 建立 quarantine/cutover 摘要和 restricted evidence archive；
4. archive sibling worktree；
5. 可见开发树 hash 对账；
6. archive-only preservation checkpoint；
7. 从 fixed clean HEAD 建 active lean worktree；
8. 283-row 初始 coverage ledger 和 dirty-path disposition；
9. 精简 `AGENTS.md`、新 domain contract 和新 source/decision registry；
10. Lean governance direct-adoption 独立 High review；
11. 按 dependency/story 把 dirty 产品 hunks 作为可见 adoption commits 导入；
12. 三个 canary；
13. 依据 canary 更新 ETC；
14. 自动进入 rolling waves；
15. Foundation；
16. eligible Full；
17. Final；
18. Release。

Superpowers discovery 调整可在获得外部权限后独立执行，但不占据 cutover 关键路径。

实施时不再创建：

- C3 design task catalog；
- C3 plan materialization task；
- C3 governance activation task；
- C3 scope compatibility task；
- 每个 cutover step 的独立 evidence bundle。

允许一份简洁 cutover report 和一份 independent review。

## 18. C3.1 完成定义

C3.1 governance cutover 完成必须同时满足：

- 原 workspace 和 artifacts 完整保留；
- restricted evidence archive 和 checksum manifest 可验证；
- archive sibling 可见开发树对账一致；
- preservation commit 位于 archive-only ref，并明确标为非验收；
- active integration 从 fixed clean HEAD 开始，不继承 preservation parent；
- 每个 visible dirty path 有唯一 disposition；
- 当前 dirty 产品 hunks 只能通过可见 adoption commit 进入 active branch；
- 283-row ledger 唯一、完整、状态语义正确；
- 六个非终态产品任务没有被伪关闭；
- Lean `AGENTS.md` 保留全部 fail-closed 红线；
- old taskctl/evidence/scope 明确 read-only；
- ordinary story 不再依赖旧 scope/accept；
- reviewer 对 lean-governance adoption commit 给出零 finding；
- PROTECTED verification lane 可在 exact commit 独立重跑决定性 checks；
- 三个 canary 达到质量和性能门槛；
- 没有 push、reset、clean、stash 或用户改动丢失。

V8 Goal 完成定义仍然是：

- 所有 required catalog assertion 有当前可验证覆盖；
- 所有可执行故事完成；
- customer/authority blocker 如实处理；
- Foundation、Full、Final、Release 按顺序完成；
- 最终没有 required work。

## 19. 决策记录

| 决策 | 结论 |
| --- | --- |
| 是否继续修旧 scope engine | 否；只保留历史读取 |
| 是否采用原 C2 | 否；方向继承，去掉 cohort 配额和新 receipt 系统 |
| 是否采用 C3.1 | 是，用户已批准方向 |
| 是否保留 283 catalog | 是，作为覆盖断言，不作为 283 次执行 |
| 是否保留 SDD | 是；保留产品型 Spec-Driven，取消流程型强制编排 |
| 是否保留 subagents | 是；按并行和独立 review 需要使用 |
| 是否保留 Superpowers 源码 | 是 |
| 是否保留 Superpowers 自动发现 | 建议关闭，但属于可选外部配置，不是 cutover PASS 条件 |
| 是否默认 High | 是 |
| 是否自动切换 Ultra | 否 |
| 是否 commit | C3.1 执行使用 Git-native commit；当前文档落盘本身不自动 commit |
| 是否 push | 否 |
| 是否保留原 dirty tree | 是，作为 quarantine source |
| 是否重新做 V8 客户/法律设计 | 否，除非来源或客户决定真实变化 |
| 是否立即搭建完整 CI | 否，不作为 cutover 前置 |

## 20. 最终结论

C3.1 是当前条件下“足够简单、足够安全、可显著提速”的最终方案。

它不是降低标准，而是把标准放回正确位置：

- 业务事实由来源和 domain contract 约束；
- 实现范围由 isolated Git commit 约束；
- 正确性由可观察测试约束；
- 高风险由独立 review 约束；
- 不遗漏由 283-row ledger 约束；
- 最终质量由 milestone 和 release gate 约束。

除此之外，不再建立第三套治理系统。
