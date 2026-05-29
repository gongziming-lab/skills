# Jira CPU 验证风险分析 Skill

## 目的

使用本 skill 分析 ASIC CPU 设计项目中的 Jira tickets，并输出面向验证负责人的项目健康度、RTL bug 趋势、验证就绪度、tapeout 风险以及跨团队闭环质量视图。

本 skill 适用于 CPU 项目验证负责人、验证经理、DV lead 或质量 owner，用于回答以下问题：

- 项目是真的在向 tapeout 收敛，还是只是在关闭容易关闭的 bug？
- 哪些 CPU 模块、特性或验证环境仍然存在高风险？
- 关键 bug 是否被正确修复、验证，并通过回归手段防止复现？
- bug reopen / reject / duplicate 模式是否暴露出 root cause 分析不足或验证 signoff 不充分？
- 验证任务、覆盖率收敛、用例开发、formal 证明、emulation/prototype 任务和 debug 活动是否在一致推进？
- 哪些 Jira items 需要管理层升级处理？

输出必须基于证据。不要只做 ticket 数量统计摘要。

---

## 何时使用本 Skill

当用户要求分析 Jira issues、RTL bugs、验证任务、项目状态、bug 趋势、tapeout readiness、风险 dashboard、验证进展、CPU bug 质量、验证闭环或基于 Jira 的项目健康度时，使用本 skill。

典型用户请求：

- “Analyze the Jira bugs for my CPU project and identify verification risks.”
- “Give me a weekly verification status based on Jira.”
- “Which modules are high-risk based on open RTL bugs?”
- “Can you summarize blockers before tapeout?”
- “Analyze reopen bugs and tell me what process problems they indicate.”
- “Generate a verification leader dashboard from Jira tickets.”
- “Review Jira bugs and identify whether verification closure is credible.”

---

## 必需输入

至少收集或请求以下任意一种输入：

1. 通过可用 connector、MCP server、API token、CSV 导出、Excel 导出或粘贴的 issue 表格访问 Jira 数据。
2. Jira project key 和 JQL 查询语句。
3. Jira issue 导出数据，至少包含：
   - Issue key
   - Issue type
   - Summary
   - Status
   - Priority 或 severity
   - Component 或 module
   - Assignee
   - Reporter
   - Created date
   - Updated date
   - Resolution date
   - Fix version 或 milestone
   - Labels
   - Description
   - Comments 或 activity history，如果可用

强烈建议额外包含以下字段：

- Root cause category
- Bug source 或 detection method
- Verification phase
- Found by test / random / regression / formal / emulation / FPGA / silicon / code review
- Escaped from previous phase
- Affected CPU block
- Affected feature
- Reopen count
- Linked issues
- Blocking / blocked-by links
- Testcase link
- Commit link
- Fix commit
- Verification evidence link
- Coverage item link
- Waiver link
- Owner team
- Target tapeout milestone
- Due date
- Aging days
- Last comment date

如果字段缺失，应谨慎地从 summary、description、labels、components 和 comments 中推断，并明确说明不确定性。

---

## 期望的 Jira 分类体系

如果项目使用一致的分类体系，本 skill 效果最好。如果分类体系不存在，应提出建议分类。

### Issue Types

常见 issue 类型：

- RTL Bug
- Design Bug
- DV Bug
- Verification Task
- Coverage Task
- Formal Task
- Assertion Task
- Scoreboard / Checker Task
- Testcase Task
- Regression Failure
- Emulation / FPGA Task
- Prototype Bug
- Performance Bug
- RAS / Safety Bug
- Security Bug
- Spec Issue
- Waiver
- Methodology / Infrastructure Task

### CPU Block / Component

推荐的 CPU component 字段或 label：

- IFU
- BPU
- Fetch
- Decode
- Rename
- Dispatch
- Issue
- Scheduler
- ROB
- EXU
- ALU
- MUL
- DIV
- FPU
- SIMD
- LSU
- Load Queue
- Store Queue
- DCache
- ICache
- L2
- MMU
- TLB
- Page Walk
- Coherence
- CHI / ACE / AXI
- Interrupt / Exception
- CSR / System Register
- Debug / Trace
- RAS
- Power / Clock / Reset
- Security / RME / Isolation
- DFT / MBIST
- Top-level Integration
- Verification Environment
- Regression Infrastructure

### Verification Phase

推荐的 phase label：

- Architecture review
- Microarchitecture review
- Testplan
- Unit test
- Subsystem test
- CPU top
- SoC integration
- Formal
- Simulation regression
- Emulation
- FPGA prototype
- Performance validation
- Low power validation
- RAS validation
- Security validation
- Gate-level simulation
- Silicon validation

### Bug Source / Detection Method

推荐的 detection label：

- Directed test
- Random test
- Stress test
- Architectural test
- ISA test
- Formal property
- Assertion
- Checker
- Scoreboard
- Functional coverage
- Code review
- Spec review
- Lint
- CDC/RDC
- Low-power check
- Emulation
- FPGA
- Silicon
- Customer scenario
- Regression triage

### Root Cause Category

推荐的 root cause category：

- Spec ambiguity
- Spec mismatch
- Microarchitecture corner case
- RTL implementation error
- FSM/state machine bug
- Ordering bug
- Coherency bug
- Exception/flush/replay bug
- Reset/power/clock bug
- Timing/configuration dependency
- Interface protocol bug
- Testbench/model bug
- Checker/scoreboard bug
- Coverage hole
- Missing test
- Environment/configuration issue
- Integration issue
- Tool/script issue
- Legacy bug
- Incomplete fix
- Duplicate/invalid

---

## 数据收集流程

### 1. 确定范围

分析前，先识别：

- Jira project key
- CPU project name
- Tapeout 或 milestone 名称
- Date range
- Scope 内的 blocks 或 components
- Scope 内的 issue types
- 是否包含 closed issues
- 是否除 bugs 外还包含 verification tasks
- 是否包含 DV infrastructure issues
- 是否包含来自 emulation、FPGA 或 silicon validation 的 bugs

如果用户没有指定范围，使用较宽的默认范围：

```jql
project = <PROJECT_KEY>
AND issuetype in ("RTL Bug", "Design Bug", Bug, Task, Sub-task)
AND created >= -180d
ORDER BY priority DESC, updated DESC
```

### 2. 查询 Jira

使用 JQL 获取相关 issues。优先先做宽查询，然后再细化。

通用 bug 查询示例：

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND created >= -180d
ORDER BY created DESC
```

open critical bug 查询示例：

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND statusCategory != Done
AND priority in (Blocker, Critical, Highest, High)
ORDER BY priority DESC, created ASC
```

verification task 查询示例：

```jql
project = <PROJECT_KEY>
AND issuetype in (Task, Sub-task, "Verification Task", "Coverage Task", "Formal Task")
AND statusCategory != Done
ORDER BY due ASC, updated ASC
```

stale open issue 查询示例：

```jql
project = <PROJECT_KEY>
AND statusCategory != Done
AND updated <= -14d
ORDER BY updated ASC
```

如果存在 reopen count 字段，reopen 相关查询示例：

```jql
project = <PROJECT_KEY>
AND "Reopen Count" > 0
ORDER BY "Reopen Count" DESC, updated DESC
```

如果 Jira 中没有 reopen-count 字段，在可用时检查 changelog/history。

### 3. 获取详细信息

对风险分析所需的每个 issue，获取：

- Summary
- Description
- Status
- Priority/severity
- Labels/components
- Fix version/milestone
- Assignee/reporter
- Created/updated/resolved dates
- Linked issues
- Comments
- Changelog/history，如果可用
- Attachments 或 links，如果相关
- Test evidence、commit link、coverage link 或 waiver link，如果存在

不要只依赖 issue summary。

### 4. 数据归一化

归一化 status、priority、component、phase 和 root cause。

建议的归一化状态分组：

| Normalized Status | Jira Status Examples |
|---|---|
| New/Open | Open, New, To Do, Backlog |
| In Analysis | Triage, Analyzing, Need Info |
| In Fix | In Progress, Fixing, Coding |
| Fixed Pending DV | Fixed, Resolved, Ready for Verify |
| In Verification | Verifying, In Review |
| Closed | Closed, Done, Verified |
| Rejected | Invalid, Won't Fix, Duplicate, Not a Bug |

建议的 severity 分组：

| Severity | Meaning |
|---|---|
| S0 Blocker | 阻塞重大验证、tapeout、boot 或架构正确性 |
| S1 Critical | 功能正确性、数据破坏、死锁、安全、RAS 或一致性风险 |
| S2 Major | 重要 feature failure 或高影响 corner case |
| S3 Minor | 有 workaround 或功能影响较低的局部问题 |
| S4 Cosmetic | 非功能问题或文档问题 |

---

## 核心分析流程

按以下顺序执行分析。

### Step 1: 数据质量检查

在给出结论前，先检查 Jira 数据是否可用于分析。

需要报告：

- 缺失 component/module 字段
- 缺失 severity/priority
- 缺失 fix version/milestone
- 缺失 root cause
- 缺失 verification evidence
- 长时间没有更新的 stale tickets
- 已关闭但没有 testcase 或 regression evidence 的 bugs
- 状态与 comments 不一致的 bugs
- ownership 不清晰的 tickets
- 没有链接到 spec/testplan/commit 的 tickets

将弱数据质量标记为一种验证管理风险。

### Step 2: 整体 Bug 趋势

分析：

- 每周新增 bugs
- 每周关闭 bugs
- open bugs 净变化趋势
- open critical bug 趋势
- 按 severity 的关闭率
- bug aging 的 median 和 P90
- reopen 趋势
- duplicate/invalid 趋势
- fixed-pending-verification backlog

解释指导：

- open count 下降本身不够说明问题。
- 检查 critical bugs 是否真的在关闭，而不是只关闭低严重级别 bugs。
- 后期 S0/S1 bugs 增加，说明 design maturity 或 verification completeness 有风险。
- “fixed pending verification” 队列增长，说明 DV bandwidth、regression 或 evidence 可能成为瓶颈。
- 大量 rejected 或 duplicate issues，可能说明 triage 噪声大或 bug report 质量差。
- 大量 reopened issues，可能说明 fix 不完整、root-cause 分析不足或 regression tests 不充分。

### Step 3: Block / Feature 风险热力图

按 CPU block、feature 和 owner team 对 issues 分组。

对每个 block，计算：

- Open S0/S1 bug count
- Total open bug count
- 最近 7/14/30 天新增 bugs
- Average aging
- P90 aging
- Reopen count
- Fixed pending verification count
- Verification task backlog
- Coverage task backlog
- Formal task backlog
- Unresolved blockers 数量
- 在 integration/prototype/silicon phase 发现的未解决 bugs 数量

风险解释：

- open critical count 高：直接功能/tapeout 风险。
- late-discovery rate 高：testplan 或早期验证阶段存在薄弱点。
- reopen count 高：fix 质量差或缺少 regression。
- stale count 高：ownership 或 triage 问题。
- closed count 高但 evidence quality 低：存在 false closure 风险。
- 复杂 block bug 数少不一定代表低风险；需要和 coverage 与 verification activity 交叉检查。

### Step 4: 验证闭环证据审查

对 resolved 或 closed bugs，检查闭环是否可信。

高质量闭环应包含：

- 清晰 root cause
- Fix commit 或 RTL change link
- Focused testcase 或 replay testcase
- Regression evidence
- 如适用，checker/assertion 更新
- 如适用，coverage 更新
- 如果 bug 暴露了遗漏场景，应更新 spec/testplan
- Impact scope 说明
- 确认已搜索类似场景
- 确认没有 waiver 掩盖问题

如果存在以下情况，应标记为弱闭环：

- Status 是 Closed，但没有 verification evidence
- Comment 只写 “fixed” 或 “verified”，没有 testcase/regression reference
- 没有 root cause
- 没有 fix link
- 对 escaped bug 没有新增 regression
- 没有 follow-up coverage owner
- 关闭后又 reopen
- 作为 duplicate 关闭，但 parent issue 未关闭或未验证

### Step 5: 风险分类

将每个风险项归入一个或多个类别。

推荐风险类别：

1. Functional correctness risk
2. CPU architectural compliance risk
3. Data corruption risk
4. Deadlock/livelock/starvation risk
5. Exception/interrupt/flush/replay ordering risk
6. Memory ordering/coherency risk
7. MMU/TLB/page-table risk
8. RAS/safety diagnostic risk
9. Security/isolation risk
10. Power/reset/clock risk
11. Performance risk
12. Integration risk
13. Verification environment risk
14. Coverage closure risk
15. Formal convergence/signoff risk
16. Emulation/FPGA/prototype readiness risk
17. Regression stability risk
18. Process/ownership risk

每个 risk 应包含：

- Jira 中的证据
- Impact
- Likelihood
- Current owner
- Required action
- Escalation recommendation
- Confidence level

### Step 6: 识别验证盲区

不要只看 open bugs。

潜在盲区信号：

- 复杂 block bug 很少，同时 verification tasks 也很少
- 最近没有来自 random stress tests 的 bugs
- control-heavy logic 没有来自 formal/assertions 的 bugs
- integration-heavy features 没有来自 emulation/prototype 的 bugs
- 很多 bugs 是 integration 阶段发现，而不是 unit test 阶段发现
- feature freeze 后仍发现大量 bugs
- 同一个 block 反复出现相同 root cause
- closed bugs 没有新增 coverage
- waiver 数量高
- coverage tasks 长期 stale
- 大量 environment bugs 阻塞真实 RTL bug 发现
- regression failures 经常被标记为 infrastructure，但没有 root-cause proof
- bug description 提到 “corner case”，但没有新增 directed 或 random regression

### Step 7: 升级清单

创建一份需要管理层关注的简洁 ticket 列表。

需要升级的问题包括：

- S0/S1 且 open
- 阻塞 regression、bring-up、emulation 或 prototype
- 超过约定 SLA
- 多次 reopened
- Fixed 后长期未 verified
- 缺少 owner
- 缺少 closure evidence
- 影响 architectural correctness、memory ordering、coherency、security、RAS 或 deadlock
- 链接到多个下游 bugs
- 导致反复 regression failures
- 设计与验证团队之间存在争议
- 依赖 spec clarification

### Step 8: 推荐行动

针对每个风险，推荐具体的验证动作，不要给泛泛建议。

示例：

- 为 exception priority corner case 增加 architectural directed test。
- 增加 random stress constraint，覆盖 flush + replay + TLB invalidate overlap。
- 为 valid/ready stability 增加 SVA protocol assertion。
- 在 fairness assumptions 下增加 request eventually drains 的 formal liveness property。
- 为 IC invalidate crossing outstanding fill 增加 coverage bin。
- 为 same-address replay 下的 load-store ordering 增加 scoreboard check。
- 增加 multi-core coherency + interrupt storm 的 emulation stress test。
- 将 Jira ticket 拆分为 RTL fix、testcase、coverage 和 regression evidence subtasks。
- 要求 design owner 提供 root cause 和 similar-case analysis。
- 在关联 fix commit 和 regression evidence 前，阻止 closure。
- 将 stale S1 bug 升级给 block owner 和 project lead。

---

## 风险评分模型

使用透明的风险评分。不要把评分当作绝对事实；评分用于排序和聚焦管理注意力。

### Ticket-Level Risk Score

建议公式：

```text
risk_score =
  severity_weight
+ age_weight
+ reopen_weight
+ late_phase_weight
+ block_criticality_weight
+ evidence_gap_weight
+ dependency_weight
+ trend_weight
```

建议权重：

| Factor | Condition | Weight |
|---|---:|---:|
| Severity | S0 | +50 |
| Severity | S1 | +35 |
| Severity | S2 | +20 |
| Severity | S3 | +8 |
| Age | open > 30 days | +15 |
| Age | open > 14 days | +8 |
| Reopen | each reopen | +10 |
| Late phase | found in CPU top / SoC / emulation / FPGA / silicon | +15 |
| Critical block | MMU, LSU, coherence, exception, security, RAS, reset/power | +10 |
| Evidence gap | closed/resolved without verification evidence | +20 |
| Dependency | blocks other issues or milestone | +15 |
| Trend | same root cause appears repeatedly | +10 |

风险等级：

| Score | Level |
|---:|---|
| >= 70 | Red |
| 40-69 | Amber |
| < 40 | Green |

### Block-Level Risk Score

对每个 block：

```text
block_risk =
  5 * open_S0
+ 3 * open_S1
+ 2 * open_S2
+ 2 * reopened_bugs
+ 2 * stale_open_bugs
+ 2 * fixed_pending_verification
+ 2 * late_found_bugs
+ 1 * open_verification_tasks
+ 1 * evidence_gap_closed_bugs
- 1 * high_quality_recent_closures
```

需要结合工程判断解释。复杂 block 如果 bug 数低但验证活动也低，不应自动判断为 Green。

---

## 输出格式

根据用户请求选择对应输出格式。

### 1. Executive Verification Status

用于经理级状态汇报。

```markdown
# CPU Verification Jira Risk Summary

## Overall Status

- Overall risk: Red / Amber / Green
- Main conclusion: <one-sentence conclusion>
- Tapeout readiness concern: <yes/no/conditional>
- Data quality confidence: High / Medium / Low

## Key Metrics

| Metric | Value | Interpretation |
|---|---:|---|
| Total open bugs |  |  |
| Open S0/S1 bugs |  |  |
| New bugs in last 7 days |  |  |
| Closed bugs in last 7 days |  |  |
| Fixed pending verification |  |  |
| Reopened bugs |  |  |
| Stale open bugs >14 days |  |  |
| Closed bugs without evidence |  |  |

## Top Risks

| Rank | Risk | Evidence | Impact | Owner | Action |
|---:|---|---|---|---|---|
| 1 |  |  |  |  |  |
| 2 |  |  |  |  |  |
| 3 |  |  |  |  |  |

## Block Heatmap

| Block | Open Critical | Aging | Reopen | Evidence Quality | Risk | Comment |
|---|---:|---:|---:|---|---|---|
| LSU/MMU |  |  |  |  | Red/Amber/Green |  |
| IFU/BPU |  |  |  |  | Red/Amber/Green |  |
| ROB/Issue |  |  |  |  | Red/Amber/Green |  |

## Required Management Actions

1. <action>
2. <action>
3. <action>
```

### 2. Weekly Verification Leader Report

```markdown
# Weekly Jira-Based CPU Verification Status

## This Week's Signal

- New issues:
- Closed issues:
- Net open change:
- New S0/S1:
- S0/S1 closed:
- Main risk movement:

## What Improved

- <evidence-based improvement>

## What Got Worse

- <evidence-based deterioration>

## Open Critical Items

| Issue | Block | Severity | Age | Status | Risk | Next Action |
|---|---|---|---:|---|---|---|

## Verification Closure Quality

| Area | Observation | Risk | Required Action |
|---|---|---|---|

## Next-Week Focus

1. <specific target>
2. <specific target>
3. <specific target>
```

### 3. Ticket-Level Risk Table

```markdown
| Issue | Summary | Block | Severity | Status | Age | Risk Level | Why Risky | Recommended Action |
|---|---|---|---|---|---:|---|---|---|
```

### 4. Root-Cause and Escape Analysis

```markdown
# Root-Cause Pattern Analysis

## Dominant Root Causes

| Root Cause | Count | Blocks | Late Found? | Interpretation |
|---|---:|---|---|---|

## Escaped Bugs

| Issue | Found Phase | Should Have Been Found In | Escape Reason | Prevention |
|---|---|---|---|---|

## Verification Methodology Improvements

1. <methodology action>
2. <test/checker/formal/coverage action>
3. <process action>
```

### 5. Tapeout Readiness Gate Review

```markdown
# Jira Evidence Review for Tapeout Readiness

## Gate Result

- Gate recommendation: Pass / Conditional Pass / Fail
- Reason:

## Blocking Conditions

| Condition | Evidence | Required Closure |
|---|---|---|

## Conditional Pass Items

| Item | Risk | Required Mitigation | Owner | Deadline |
|---|---|---|---|---|

## Evidence Gaps

| Area | Missing Evidence | Risk | Action |
|---|---|---|---|

## Final Recommendation

<clear recommendation with rationale>
```

---

## CPU 验证分析启发式规则

应用 CPU-specific 判断。不要把所有 bugs 等同看待。

### 高风险 CPU Bug 主题

优先关注与以下内容相关的 bugs：

- Architectural state corruption
- Wrong instruction retirement
- Exception priority 或 precise exception violation
- Interrupt delivery 或 masking
- Flush/replay interaction
- Speculation side effects
- Load-store ordering
- Same-address load/store hazard
- MMU/TLB invalidation
- Page-table permission 和 attribute checks
- I-cache/D-cache coherency
- Multi-core coherence
- Deadlock/livelock/starvation
- RAS error logging 或 containment
- Security/isolation boundaries
- Debug/trace architectural visibility
- Reset/power domain crossing
- Clock gating 和 enable sequencing
- Performance counters，如果用于 architecture-visible behavior
- External protocol violations，例如 AXI/CHI ordering 或 response rules

### 验证流程异味

将以下模式标记为 process risk：

- bugs 反复在没有 testcase links 的情况下关闭
- bug comments 显示 design 与 verification 之间存在分歧
- critical bugs 没有 owner 或 owner 错误
- 老 bugs 被移动到未来 milestone，但没有 documented risk acceptance
- 大量 bugs 被标记为 duplicate，但 parent issue 没有 closure evidence
- Regression failures 反复被 waiver 成 “environment issue”
- 同样 root cause 跨项目或跨 CPU 代际反复出现
- 声称 coverage closure 后仍发现新的 critical bugs
- 大量 prototype/silicon bugs 本应在 simulation/formal 中发现
- verification tasks 晚于 RTL fixes 关闭
- coverage tasks 没有链接到 bug root causes
- formal tasks 以 “inconclusive” 关闭，但没有 residual-risk statement

---

## 推荐 JQL 查询库

将 `<PROJECT_KEY>`、`<MILESTONE>` 和字段名替换为项目实际值。

### All active CPU verification bugs

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND statusCategory != Done
ORDER BY priority DESC, created ASC
```

### Open critical bugs

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND statusCategory != Done
AND priority in (Blocker, Critical, Highest, High)
ORDER BY priority DESC, created ASC
```

### Bugs fixed but not verified

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND status in ("Fixed", "Resolved", "Ready for Verify", "Pending Verification")
ORDER BY updated ASC
```

### Stale open bugs

```jql
project = <PROJECT_KEY>
AND statusCategory != Done
AND updated <= -14d
ORDER BY updated ASC
```

### Recently created bugs

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND created >= -7d
ORDER BY created DESC
```

### Recently closed bugs

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
AND statusCategory = Done
AND resolved >= -7d
ORDER BY resolved DESC
```

### Bugs by milestone

```jql
project = <PROJECT_KEY>
AND fixVersion = <MILESTONE>
AND issuetype in (Bug, "RTL Bug", "Design Bug")
ORDER BY priority DESC, status ASC
```

### Verification tasks not done

```jql
project = <PROJECT_KEY>
AND issuetype in (Task, Sub-task, "Verification Task", "Coverage Task", "Formal Task")
AND statusCategory != Done
ORDER BY priority DESC, due ASC
```

### Regression-related tickets

```jql
project = <PROJECT_KEY>
AND (
  labels in (regression, nightly, failure)
  OR summary ~ "regression"
  OR description ~ "regression"
)
ORDER BY updated DESC
```

### Formal verification tasks

```jql
project = <PROJECT_KEY>
AND (
  labels in (formal, fpv, sva, assertion)
  OR summary ~ "formal"
  OR summary ~ "assertion"
)
ORDER BY status ASC, updated DESC
```

### Emulation / FPGA / prototype tickets

```jql
project = <PROJECT_KEY>
AND (
  labels in (emulation, fpga, prototype)
  OR summary ~ "emulation"
  OR summary ~ "FPGA"
  OR summary ~ "prototype"
)
ORDER BY priority DESC, updated DESC
```

---

## 证据提取规则

阅读 ticket 文本和 comments 时，提取以下信息。

### Extracted Fields

```yaml
issue_key:
summary:
issue_type:
status:
priority:
severity:
component:
subsystem:
owner:
reporter:
created:
updated:
resolved:
age_days:
fix_version:
milestone:
labels:
found_phase:
detection_method:
root_cause:
impact:
linked_issues:
blocking_status:
has_fix_commit:
has_testcase:
has_regression_evidence:
has_coverage_update:
has_formal_property_update:
has_spec_update:
has_waiver:
reopen_count:
risk_category:
risk_score:
risk_level:
recommended_action:
confidence:
```

### Root-Cause Extraction

如果 root cause 没有显式标注，则从措辞中推断：

- “flush”, “replay”, “exception”, “retire”, “commit” → ordering/precise-state risk
- “TLB”, “page”, “permission”, “attribute”, “ASID”, “VMID” → MMU/TLB risk
- “snoop”, “coherent”, “shareable”, “invalidate”, “ownership” → coherence risk
- “valid/ready”, “response”, “ID”, “ordering” → interface protocol risk
- “deadlock”, “hang”, “no forward progress” → liveness risk
- “X”, “reset”, “clock gate”, “power” → reset/power/X-propagation risk
- “RAS”, “SError”, “poison”, “ECC”, “parity” → RAS/safety risk
- “secure”, “realm”, “permission”, “isolation” → security risk
- “coverage”, “checker”, “scoreboard”, “env” → verification environment 或 closure risk

始终区分显式字段数据与推断分类。

---

## 输出风格指南

使用验证负责人风格：

- 清晰、基于证据、面向风险。
- 优先表达 “risk and action”，而不是泛泛总结。
- 突出 blockers 和 weak evidence。
- 不要仅基于 closed-ticket count 得出乐观结论。
- 当 Jira 字段不完整时，不要夸大置信度。
- dashboard 使用简洁表格。
- 只有在技术细节会改变风险判断时，才展开技术细节。

推荐措辞：

- “This is a closure-evidence risk, not only a bug-count risk.”
- “The LSU/MMU area remains Amber because the open critical count is small, but recent bugs were found late and closure evidence is weak.”
- “The fixed-pending-verification backlog suggests DV bandwidth or regression stability is now the bottleneck.”
- “The reopen pattern points to incomplete root cause analysis rather than isolated implementation mistakes.”

避免：

- “Project looks good because many bugs are closed.”
- “No bugs means no risk.”
- “Closed means verified.”
- “Low priority means safe.”
- “Jira data proves tapeout readiness.”

---

## 最终回答检查清单

回答用户前，确认答案包含：

- Overall risk status
- Key Jira metrics
- Trend interpretation
- High-risk blocks/features
- Open critical tickets or categories
- Stale/reopened/fixed-pending-verification issues
- Evidence quality assessment
- Escalation list
- Concrete recommended actions
- Data limitations and confidence level

如果输入数据不完整，包含以下内容：

```markdown
## Data Limitations

The Jira data is incomplete in the following ways:
- <missing field or missing evidence>
- <missing field or missing evidence>

Therefore, the risk assessment confidence is <High/Medium/Low>.
```

---

## 面向用户的示例回答骨架

```markdown
Based on the Jira tickets provided, I would not judge verification status only by total open/closed count. The main risk signal is the combination of critical open bugs, fixed-but-not-verified backlog, reopen rate, stale ownership, and lack of closure evidence.

## Executive Summary

Overall risk: Amber

The project appears to be functionally converging, but verification closure is not yet strong enough for a clean tapeout recommendation because several high-impact bugs are fixed but not verified, and closure evidence is missing for multiple CPU-memory-system issues.

## Top Verification Risks

| Risk | Evidence | Impact | Action |
|---|---|---|---|
| Fixed-pending-verification backlog | 12 bugs fixed but not verified, 5 older than 14 days | RTL fixes may not be regression-proven | Allocate DV owner and require testcase/regression links |
| LSU/MMU late bugs | Recent S1 bugs found at CPU-top | Unit-level testplan may have blind spots | Add focused random stress and formal checks |
| Weak closure evidence | 18 closed bugs lack testcase links | False closure risk | Block future closure without evidence |

## Recommended Management Actions

1. Treat S0/S1 fixed-pending-verification as tapeout blockers until regression evidence is attached.
2. Run a focused escape analysis for late LSU/MMU and coherency bugs.
3. Require each reopened bug to include root cause, similar-case search, and regression prevention.
4. Build a block-level risk dashboard combining bug trend, coverage gaps, and closure evidence.
```

---

## 可选自动化思路

如果具备 Jira API 访问能力，可以自动化以下内容：

1. Daily open critical bug digest.
2. Weekly verification risk dashboard.
3. Stale ticket escalation.
4. Fixed-pending-verification queue report.
5. Reopened bug root-cause review.
6. Closed-without-evidence audit.
7. Block heatmap generation.
8. Tapeout readiness gate report.
9. 从 UT 到 CPU-top/prototype/silicon 的 bug escape analysis。
10. Regression failure trend analysis.

---

## Tool Integration 注意事项

如果使用 Jira Cloud REST API 或 Jira MCP server：

- 优先使用 JQL 进行初始 issue retrieval。
- 对大型项目使用 pagination。
- 尽可能只请求分析所需字段。
- 当需要分析 reopen count、status transitions 或 aging 时，获取 changelog/history。
- 当需要分析 closure evidence 和 root cause quality 时，获取 comments。
- 构造复杂 JQL 前，先解析 custom field names。
- 由于项目特定字段名和允许值会变化，需要在 Jira instance 中验证生成的 JQL。
- 永远不要在输出中暴露 API tokens 或 credentials。
- 如果项目保密性重要，应谨慎总结敏感 ticket 内容。
