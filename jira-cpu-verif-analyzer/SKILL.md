# JIRA CPU 验证风险分析 Skill

## 1. Skill 目标

本 Skill 用于分析 CPU 芯片设计开发项目中通过 JIRA 管理的 RTL 缺陷、验证任务、设计任务、风险项和投片阻塞项，帮助验证 leader、模块负责人、项目经理和技术主管快速掌握：

- 当前 CPU 项目的验证收敛状态；
- RTL 缺陷的数量、趋势、严重程度和模块分布；
- 高风险模块、高风险 owner、高风险里程碑；
- 长期未关闭、反复 reopen、临近 tapeout 仍未解决的问题；
- 验证活动是否真正释放风险，而不是只体现“跑了很多用例”；
- 是否存在设计质量、验证质量、流程协同或风险闭环方面的系统性问题。

本 Skill 的输出风格应面向 CPU 项目验证管理场景，结论要直接、分层、可行动，避免只给出普通 JIRA 统计报表。

---

## 2. 适用场景

当用户提出以下需求时，使用本 Skill：

- 分析 JIRA 中的 RTL bug、defect、task、sub-task、risk、story 或 epic；
- 根据 JIRA 导出的 CSV、Excel、JSON、API 结果或用户粘贴的表格数据判断项目风险；
- 给验证 leader、模块 owner、项目管理层生成周报、月报、里程碑评审材料；
- 分析某个 CPU 模块的 bug 收敛情况，例如 IFU、IDU、EXU、LSU、MMU、TLB、Cache、NoC、CHI、Debug、RAS、低功耗、DFT、CDC、Security、Virtualization 等；
- 分析 tapeout 前是否存在验证风险、质量风险、交付风险；
- 对 JIRA 数据做根因归类、风险评分、趋势分析和行动项建议。

---

## 3. 输入数据要求

### 3.1 推荐输入格式

优先支持以下输入：

1. JIRA 导出的 CSV 或 Excel；
2. JIRA REST API 返回的 JSON；
3. 用户粘贴的问题单列表；
4. 用户提供的 JQL 查询结果摘要；
5. 按模块、状态、owner、severity 聚合后的统计表。

### 3.2 推荐字段

如果可用，尽量读取或要求用户提供以下字段：

| 字段类别 | 推荐字段 |
|---|---|
| 基本信息 | Key、Summary、Issue Type、Status、Resolution、Priority、Severity |
| 责任信息 | Assignee、Reporter、Owner Team、Module Owner、Verifier、Designer |
| 模块信息 | Component/s、Module、Sub-module、Block、Feature、Label |
| 版本/里程碑 | Fix Version、Affects Version、Target Version、Milestone、Sprint、Tapeout Phase |
| 时间信息 | Created、Updated、Resolved、Due Date、Status Change Time |
| 缺陷属性 | Root Cause、Bug Category、Found Phase、Injection Phase、Detection Method |
| 验证属性 | Test Name、Test Level、Coverage Item、Checker、Assertion、Regression、Random Seed |
| 风险属性 | Blocker、Escalation、Linked Issues、Dependency、Customer Impact、Waiver |
| 历史信息 | Changelog、Reopen Count、Status Transition、Comment History |

### 3.3 字段缺失时的处理原则

- 如果没有 `Severity`，用 `Priority`、标签、Summary 关键词和模块影响面推断风险等级，但必须注明“Severity 字段缺失，风险等级为推断结果”。
- 如果没有 `Module`，尝试从 Component、Label、Summary、Epic、Fix Version 中推断模块。
- 如果没有 `Resolved` 时间，只能分析打开时长和更新时间，不能精确分析关闭周期。
- 如果没有 Changelog，不能精确统计每个状态停留时间，只能使用 Created、Updated、Resolved 粗略分析。
- 如果没有 Root Cause，不要编造根因；可以根据 Summary、Description、Label、Comment 做“疑似根因”分类，并明确标注为推断。

---

## 4. 分析总流程

执行分析时，按以下流程处理。

### Step 1：理解项目背景和分析目标

先识别用户关心的是哪一类问题：

- 项目整体验证状态；
- 某个模块的缺陷风险；
- 某个 milestone 或 tapeout 阶段的退出风险；
- 某个 owner/team 的任务负载；
- 新增 bug 趋势和收敛速度；
- 高优先级 blocker；
- JIRA 数据质量和流程问题；
- 验证 leader 周报或评审报告。

如果用户没有指定分析目标，默认从“验证状态 + 风险识别 + 行动建议”三个层次输出。

### Step 2：字段标准化

将不同 JIRA 项目中的字段映射到统一字段：

```text
issue_key       = Key / Issue key / id
issue_type      = Issue Type / Type
summary         = Summary / Title
status          = Status
status_category = To Do / In Progress / Done
priority        = Priority
severity        = Severity / customfield_severity / Bug Severity
module          = Component/s / Module / Block / Label
assignee        = Assignee / Owner
reporter        = Reporter
created         = Created
updated         = Updated
resolved        = Resolved / Resolution Date
fix_version     = Fix Version/s / Target Version / Milestone
labels          = Labels
root_cause      = Root Cause / Cause Category
found_phase     = Found Phase / Detection Phase
reopen_count    = Reopen Count / status transition back to Open
links           = Linked Issues / Blocks / Is blocked by
```

字段名称不一致时，应优先保留原字段，并在报告中说明字段映射关系。

### Step 3：数据质量检查

先检查 JIRA 数据本身是否可信：

- 是否存在大量缺失 Assignee 的问题单；
- 是否存在大量没有 Component/Module 的 bug；
- 是否存在 Severity/Priority 未填写；
- 是否存在关闭但 Resolution 为空；
- 是否存在更新时间很久以前但状态仍为 In Progress；
- 是否存在相同 Summary 或相似 Summary 的重复单；
- 是否存在任务被错误标记为 Done 但 comment 中仍有未解决事项；
- 是否存在 target milestone 为空或 fixVersion 不一致；
- 是否存在 bug 和 task 混用，导致统计口径失真。

如果数据质量较差，必须先在结论中提示“JIRA 数据质量本身已构成管理风险”。

### Step 4：基础统计

至少统计以下指标：

- 总问题单数量；
- Open / In Progress / Resolved / Closed 数量和比例；
- 新增 bug 数、关闭 bug 数、净增 bug 数；
- 未关闭 bug backlog；
- 按 Severity/Priority 分布；
- 按模块分布；
- 按 owner/team 分布；
- 按 milestone/fixVersion 分布；
- 平均打开时长、P50/P90/P95 打开时长；
- 最近 7 天、14 天、30 天新增和关闭趋势；
- Reopen 数量和 reopen rate；
- Blocker/Critical 问题数量；
- 临近 tapeout 未关闭问题数量。

### Step 5：CPU 验证风险识别

从 CPU 验证视角识别以下风险。

#### 5.1 缺陷收敛风险

重点关注：

- 高等级 bug 未关闭；
- 新增 bug 数持续高于关闭 bug 数；
- P0/P1 bug 在 milestone 后期仍新增；
- 同一模块在后期持续产生高等级 bug；
- bug 修复后 reopen 率高；
- 大量 bug 长期停留在 In Progress、Review、Ready for Verification；
- 关闭速度依赖少数 owner，存在资源瓶颈。

典型结论表达：

> 当前缺陷 backlog 未呈现稳定下降趋势，且高等级 bug 在后期仍有新增，说明该模块仍处于功能/微架构不稳定状态，不建议将其视为 tapeout 风险已释放。

#### 5.2 模块热点风险

按 CPU 模块聚类，识别高风险模块：

- IFU：取指顺序、自修改代码、I-cache invalidation、branch prediction、异常入口；
- Decode/Dispatch：指令解码、非法指令、异常优先级、rename/dispatch backpressure；
- OOO/ROB/Issue：年龄选择、flush、commit、exception、replay、starvation；
- LSU：load/store ordering、store buffer、MMU interaction、TLBI、cacheability、atomic/exclusive；
- MMU/TLB：page table walk、permission、ASID/VMID、TLBI、speculation、translation fault；
- Cache/Coherence：MESI/MOESI、CHI/ACE、snoop、eviction、dirty data、barrier；
- Interrupt/Exception：优先级、嵌套、精确异常、debug entry/exit；
- Low Power：retention、power gating、reset sequence、clock gating、CDC/RDC；
- RAS：error injection、parity/ECC、fault reporting、recovery、poison propagation；
- Security/Virtualization：TrustZone/RME、stage-2 translation、permission isolation、side-effect；
- DFT/Debug：scan、MBIST、debug access、halt/resume、trace；
- SoC Interface：AXI/CHI/APB、QoS、deadlock、backpressure、ordering。

如果某个模块同时满足“bug 数多 + 高等级 bug 多 + 长期未关闭 + reopen 多 + 后期仍新增”，应标记为 Top Risk Module。

#### 5.3 验证充分性风险

检查 JIRA 是否体现出有效的风险释放证据：

- 是否只有大量 test/task，但缺少 bug root cause、coverage closure、checker/assertion 证据；
- 是否有新增 directed/random/regression 用例对应到具体风险点；
- 是否有 assertion、formal、scoreboard、reference model、coverage 的闭环记录；
- 是否有复杂场景的 stress test，例如 flush + exception + TLBI + cache miss + backpressure；
- 是否有跨模块问题的 owner 和 closure 证据；
- 是否有 waiver，但缺少影响分析和签核；
- 是否存在 bug 已关闭但没有验证回归证据。

结论应区分：

- “用例执行量充分”；
- “覆盖率达到目标”；
- “关键风险点已有证据释放”；
- “缺陷趋势已经收敛”。

不要把前三者混为一谈。

#### 5.4 设计稳定性风险

识别以下信号：

- 同一模块持续有新 bug；
- 同一 root cause 反复出现；
- spec unclear / design change / late ECO 类问题比例高；
- 修复一个 bug 引入多个新 bug；
- reopen 多，说明修复质量或验证确认不足；
- 多个模块被同一个架构变更影响；
- milestone 后期仍有大量 design task 未关闭。

#### 5.5 流程协同风险

识别以下信号：

- bug 长期停留在 Waiting for Design / Waiting for Verification；
- verification owner 和 design owner 责任不清；
- blocker 链接关系混乱；
- comment 中有争议但状态被关闭；
- 缺少明确的 next action、owner、due date；
- 关键 bug 没有升级记录；
- 模块之间互相阻塞，缺少项目级协调。

---

## 5. 风险评分模型

对每个问题单计算风险分，默认 0–100 分。可以根据项目习惯调整权重。

### 5.1 单个问题单风险分

```text
IssueRiskScore =
  SeverityWeight
+ StatusWeight
+ AgingWeight
+ MilestoneWeight
+ ReopenWeight
+ ModuleCriticalityWeight
+ BlockerWeight
+ VerificationEvidenceWeight
+ CustomerImpactWeight
```

推荐权重：

| 因子 | 规则 | 分值 |
|---|---|---:|
| Severity | Blocker/Critical/P0 | +25 |
| Severity | Major/High/P1 | +18 |
| Severity | Medium/P2 | +10 |
| Status | Open / In Progress / Reopened | +15 |
| Status | Ready for Verification | +8 |
| Aging | 打开超过 30 天 | +8 |
| Aging | 打开超过 60 天 | +15 |
| Milestone | 目标版本为当前 tapeout/milestone | +15 |
| Reopen | reopen >= 1 | +8 |
| Reopen | reopen >= 2 | +15 |
| Module Criticality | MMU/LSU/Cache/Coherence/OOO/RAS/Security 等关键模块 | +10 |
| Blocker | blocks 其他 issue 或被标记 blocker | +15 |
| Verification Evidence | 缺少回归/覆盖/assertion/formal 证据 | +10 |
| Customer Impact | 影响架构正确性、数据一致性、安全、RAS 或死锁 | +15 |

风险等级：

| 分数 | 风险等级 | 处理建议 |
|---:|---|---|
| >= 70 | Red | 必须项目级跟踪，明确 owner、due date、验证证据和退出标准 |
| 45–69 | Amber | 模块负责人跟踪，要求下个周期给出闭环计划 |
| 20–44 | Yellow | 常规跟踪，但需要确认不影响 milestone |
| < 20 | Green | 低风险，按常规流程处理 |

### 5.2 模块风险分

```text
ModuleRiskScore =
  0.30 * HighSeverityOpenRatio
+ 0.20 * OpenBacklogTrend
+ 0.15 * AgingRisk
+ 0.15 * ReopenRate
+ 0.10 * LateBugArrivalRate
+ 0.10 * VerificationEvidenceGap
```

输出时不要只按 bug 数排序。模块 bug 数多不一定风险最高；后期新增高等级 bug、长期不关闭、reopen 多、缺少验证证据的模块风险更高。

---

## 6. 推荐 JQL 模板

以下 JQL 需要根据项目实际字段名调整。

### 6.1 当前未关闭高风险 bug

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, Defect)
AND statusCategory != Done
AND priority in (Blocker, Critical, Highest, High)
ORDER BY priority DESC, updated ASC
```

### 6.2 当前 milestone 未关闭问题

```jql
project = <PROJECT_KEY>
AND fixVersion = <MILESTONE>
AND statusCategory != Done
ORDER BY priority DESC, created ASC
```

### 6.3 长期未更新问题

```jql
project = <PROJECT_KEY>
AND statusCategory != Done
AND updated <= -14d
ORDER BY updated ASC
```

### 6.4 后期新增高等级 bug

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, Defect)
AND created >= <DATE>
AND priority in (Blocker, Critical, Highest, High)
ORDER BY created DESC
```

### 6.5 某模块未关闭 bug

```jql
project = <PROJECT_KEY>
AND issuetype in (Bug, Defect)
AND component = <MODULE_NAME>
AND statusCategory != Done
ORDER BY priority DESC, created ASC
```

### 6.6 Reopened 问题

```jql
project = <PROJECT_KEY>
AND status changed TO Reopened
ORDER BY updated DESC
```

### 6.7 未分配 owner 的问题

```jql
project = <PROJECT_KEY>
AND statusCategory != Done
AND assignee is EMPTY
ORDER BY priority DESC, created ASC
```

### 6.8 Blocker/依赖问题

```jql
project = <PROJECT_KEY>
AND statusCategory != Done
AND (priority in (Blocker, Critical) OR labels in (blocker, tapeout_blocker, risk))
ORDER BY priority DESC, updated ASC
```

---

## 7. 分析输出模板

输出报告应包含以下结构。

### 7.1 一页管理层摘要

```markdown
# JIRA 验证状态与风险分析摘要

## 总体结论
- 当前项目状态：Green / Yellow / Amber / Red
- 主要风险判断：...
- 是否建议通过当前 milestone：建议通过 / 有条件通过 / 不建议通过

## 关键数据
- 总问题单：N
- 未关闭问题：N，占比 X%
- P0/P1 未关闭：N
- 最近 7/14/30 天新增 bug：N / N / N
- 最近 7/14/30 天关闭 bug：N / N / N
- Reopen 数量：N
- 超过 30 天未关闭：N

## Top 5 风险模块
1. <Module A>：风险原因...
2. <Module B>：风险原因...

## Top 5 必须升级问题
1. <JIRA-KEY>：原因、owner、建议动作
2. <JIRA-KEY>：原因、owner、建议动作

## 下个周期建议动作
- 动作 1：owner / due date / exit criteria
- 动作 2：owner / due date / exit criteria
```

### 7.2 验证 leader 视角详细报告

```markdown
# CPU 项目 JIRA 缺陷与验证风险分析报告

## 1. 分析范围
- 项目：...
- 时间范围：...
- JIRA 查询条件或数据来源：...
- 纳入 issue 类型：...
- 主要字段：...
- 数据质量限制：...

## 2. 总体缺陷收敛状态
- 缺陷总量与状态分布
- 新增/关闭趋势
- Backlog 趋势
- 高等级 bug 收敛情况
- 结论：...

## 3. 模块风险分析
| 模块 | Open Bug | P0/P1 Open | Aging Bug | Reopen | 后期新增 | 风险等级 | 主要风险 |
|---|---:|---:|---:|---:|---:|---|---|
| LSU/MMU | ... | ... | ... | ... | ... | Red | ... |

## 4. 高风险问题单清单
| Key | Summary | Module | Severity | Status | Age | Owner | Risk | 建议动作 |
|---|---|---|---|---|---:|---|---|---|

## 5. 验证充分性和风险释放证据
- 已有证据：...
- 缺失证据：...
- 需要补充的 regression/formal/assertion/coverage：...

## 6. 流程和协同风险
- Owner 缺失：...
- 状态长期停滞：...
- 跨模块依赖：...
- 关闭标准不一致：...

## 7. 行动项建议
| 优先级 | 动作 | Owner | Due Date | Exit Criteria |
|---|---|---|---|---|
```

---

## 8. CPU 项目专用风险规则

### 8.1 Tapeout 前 Red Flag

如果出现以下情况，应标记为 Red Flag：

- 当前 milestone 仍有未关闭 P0/P1 bug；
- 最近两周仍持续新增 P0/P1 bug；
- MMU/LSU/Cache/Coherence/OOO/RAS/Security 模块存在未解释清楚的高等级 bug；
- 存在 deadlock、data corruption、security violation、precise exception violation、memory ordering violation、cache coherency violation；
- bug 关闭但缺少 regression seed、test name、checker、coverage 或 formal 证据；
- 大量 bug 处于 Ready for Verification，但验证侧没有明确回归计划；
- 关键模块 bug reopen 率高于 10%；
- 同一 root cause 跨多个模块重复出现；
- waiver 数量多且缺少签核或影响分析；
- 关键 feature 的 JIRA task 已关闭，但 coverage/plan/issue link 无法证明风险释放。

### 8.2 CPU 高危关键词

分析 Summary、Description、Comment、Labels 时，重点关注以下关键词：

```text
deadlock, hang, starvation, livelock, timeout,
data corruption, wrong data, mismatch, ordering,
exception, interrupt, precise, priority,
flush, replay, rollback, commit, retire,
TLB, TLBI, MMU, PTW, ASID, VMID, page fault,
cache, snoop, coherence, dirty, eviction, invalidate,
atomic, exclusive, barrier, fence,
RAS, ECC, parity, poison, error injection,
security, permission, isolation, realm, secure, non-secure,
low power, retention, reset, clock gating, power gating,
CDC, RDC, glitch, X-propagation,
DFT, scan, MBIST, debug, trace
```

出现这些关键词的问题单，即使 Severity/Priority 不高，也应检查是否存在低估风险。

---

## 9. 输出风格要求

### 9.1 结论要管理化

不要只输出：

> LSU 有 35 个 bug，MMU 有 22 个 bug。

应输出：

> LSU 是当前最高风险模块。虽然 bug 总数不是最高，但 P1 未关闭数、后期新增数和 reopen 数同时偏高，并且问题集中在 load/store ordering、TLBI interaction 和 replay/flush 场景，说明该模块仍存在架构级风险释放不足的问题。建议将 LSU 设为下个周期项目级专项收敛对象。

### 9.2 建议要可执行

每条建议尽量包含：

- 具体问题；
- 责任 owner；
- 截止时间；
- 退出标准；
- 需要补充的验证证据。

示例：

```markdown
建议将 JIRA-1234/JIRA-1256/JIRA-1290 合并为 LSU TLBI + flush 风险专项，由 LSU design owner 和 MMU verification owner 联合闭环。退出标准包括：
1. 所有关联 bug 关闭且无 reopen；
2. 增加 directed test 覆盖 TLBI 与 outstanding transaction 并发；
3. regression 连续 3 轮无失败；
4. 增加 SVA/scoreboard 检查 stale translation 不可被使用；
5. 覆盖率点 tlbi_flush_overlap 达到目标。
```

### 9.3 明确不确定性

如果数据不足，应明确说明：

- 哪些结论是基于字段直接统计；
- 哪些结论是基于文本关键词推断；
- 哪些结论因为缺少 changelog、root cause、coverage link 而无法确认；
- 需要用户补充哪些字段才能进一步分析。

---

## 10. 可选深度分析

当数据足够时，进一步做以下分析。

### 10.1 缺陷趋势分析

- 每周新增 bug；
- 每周关闭 bug；
- 净增 bug；
- 累积 open backlog；
- 高等级 bug 趋势；
- 后期 bug arrival rate。

判断标准：

- 如果新增曲线下降且关闭曲线稳定高于新增曲线，说明缺陷正在收敛；
- 如果新增曲线在 milestone 后期反弹，说明验证深水区仍在暴露新问题；
- 如果关闭曲线很高但 reopen 也很高，说明关闭质量存疑。

### 10.2 Root Cause 分析

建议分类：

| Root Cause | 含义 |
|---|---|
| Spec unclear | 规格不清或规格变更 |
| RTL logic bug | RTL 实现错误 |
| Micro-architecture gap | 微架构方案缺陷 |
| Verification env bug | 验证环境问题 |
| Test issue | 用例错误或约束错误 |
| Checker/model issue | checker、scoreboard、reference model 问题 |
| Integration issue | 集成、配置、接口连接问题 |
| Low power/reset issue | 低功耗、reset、clock gating 问题 |
| CDC/RDC issue | 跨时钟/跨复位问题 |
| Tool/IP issue | EDA 工具或第三方 IP 问题 |

### 10.3 验证方法有效性分析

按 Detection Method 统计：

- directed test；
- random regression；
- formal/SVA；
- emulation/FPGA/prototype；
- code review；
- architecture review；
- static/lint/CDC；
- post-silicon feedback。

重点判断：

- 哪些类型 bug 主要靠随机回归发现；
- 哪些高危 bug 应前移到 formal/assertion/static；
- 是否存在“后期才由 prototype 或 silicon-like 场景发现”的前端验证缺口。

---

## 11. 与用户交互策略

### 11.1 数据足够时

直接分析，不要反复追问。即使字段不全，也先给出基于当前数据的初步结论，并列出需要补充的数据。

### 11.2 数据不足时

可以请用户补充以下最小字段：

```text
Key, Summary, Issue Type, Status, Priority/Severity, Assignee,
Component/Module, Created, Updated, Resolved, Fix Version/Milestone, Labels
```

如果用户只提供部分字段，也要尽力完成分析。

### 11.3 用户要求生成报告时

优先输出中文报告，适合直接放入项目周报、质量复盘、milestone review 或 tapeout readiness review。

---

## 12. 默认输出检查清单

每次分析完成前，检查是否已经回答以下问题：

- 当前项目验证状态是 Green、Yellow、Amber 还是 Red？
- 最大的 3 个风险是什么？
- 哪些模块最危险，为什么？
- 哪些问题单必须升级跟踪？
- 哪些 bug 长期未关闭或长期未更新？
- 是否存在 reopen 高、关闭质量差的问题？
- 是否存在 owner/模块/里程碑信息缺失导致的管理风险？
- 是否能证明关键风险已通过 regression、coverage、assertion、formal、prototype 等手段释放？
- 下个周期最应该做的 3–5 个行动项是什么？

---

## 13. 示例输出片段

```markdown
## 总体判断

当前项目验证状态评估为 Amber，主要原因不是 open bug 总量过高，而是高等级问题集中在 LSU/MMU/Cache 这类架构关键模块，并且最近两周仍有 P1 bug 新增。虽然关闭数量上升，但 reopen 率偏高，说明修复质量和验证确认流程仍不稳定。

## Top 风险

1. LSU/MMU 交互风险：多个问题集中在 TLBI、outstanding transaction、flush/replay 场景，属于架构正确性风险。
2. Cache/Coherence 收敛风险：后期仍有 ordering/coherence 相关 bug 新增，建议增加协议级 checker 和压力回归。
3. 流程风险：多个高等级 bug 缺少明确 verification evidence，关闭标准不一致。

## 建议行动

- 将 LSU/MMU 相关 bug 建立专项 risk epic，统一跟踪 root cause、修复方案和验证证据。
- 对所有 P0/P1 关闭单补充 regression seed、test name、checker/coverage link。
- 对 reopen >= 2 的问题进行设计负责人和验证负责人联合复盘。
```

---

## 14. 安全和保密要求

- 不要在报告中泄露 JIRA token、用户账号、内部服务器地址或未脱敏客户信息。
- 如果用户提供的是内部项目数据，输出时尽量使用模块名、问题单号和风险描述，不扩散敏感实现细节。
- 对外部汇报版本应自动建议脱敏：项目代号、人员姓名、内部 IP 名称、客户名称、具体漏洞细节。

---

## 15. Skill 使用方式建议

用户可以这样调用本 Skill：

```text
请使用 JIRA CPU 验证风险分析 Skill，分析我上传的 JIRA CSV，输出验证 leader 视角的风险报告。
```

```text
请分析这些 JIRA bug，判断 tapeout 前还有哪些验证风险没有释放。
```

```text
请按模块统计 JIRA 缺陷，并给出 Top 风险模块和下周收敛建议。
```

```text
请把这个 JIRA 查询结果整理成 CPU 项目验证周报。
```

```text
请基于 JIRA 数据检查当前 milestone 是否具备退出条件。
```

