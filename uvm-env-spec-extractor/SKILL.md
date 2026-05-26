---
name: uvm-env-spec-extractor
description: Reverse-engineer an existing SystemVerilog/UVM verification environment and generate a detailed testbench specification document. Use when asked to read a UVM codebase and document environment components, agents, interfaces, scoreboards, reference models, checkers, assertions, tests, sequences, coverage, regressions, and verification gaps. 对现有的SystemVerilog/UVM验证环境进行逆向工程，并生成详细的测试平台规范文档。当被要求研读UVM代码库，并对环境组件、代理（Agent）、接口、记分板、参考模型、检查器、断言、测试用例、序列、覆盖率、回归测试及验证盲点进行文档化记录时，请使用此功能。
---

# UVM Environment Specification Extractor

This skill helps an agent inspect an existing UVM-based verification environment and produce a detailed, evidence-based testbench specification document. It is designed for reverse documentation of a real codebase, not for creating a new UVM testbench from scratch.

The primary output is a verification/testbench specification document that explains what the environment actually implements, what it appears intended to verify, and what gaps or ambiguities remain.


## Output Language Support

This skill supports both English and Simplified Chinese output.

Default behavior:

- If the user does not specify a language, generate the specification document in English.
- If the user asks for Chinese, 中文, 简体中文, Simplified Chinese, or a Chinese-language document, generate the main document in Simplified Chinese and preserve English technical names, then optionally add English aliases in parentheses for major section titles or terms.
- Preserve all source-code identifiers exactly as written in the repository, including class names, instance names, signal names, file paths, macros, plusargs, enum names, interface names, package names, and test names.
- Do not translate code identifiers. Translate only explanatory prose, table headers, section titles, status labels, risk descriptions, and recommendations.

Recommended user-facing language options:

```text
output_language: english
output_language: simplified_chinese
```

When `output_language: simplified_chinese` is selected, use professional verification terminology commonly used in CPU/ASIC verification teams. Keep UVM/SystemVerilog terms readable and avoid over-translating terms that are normally used in English.

Preferred terminology mapping:

| English term | Simplified Chinese term |
|---|---|
| UVM testbench specification | UVM 验证环境 / Testbench 规格说明 |
| verification environment | 验证环境 |
| testbench top | testbench 顶层 / TB 顶层 |
| component hierarchy | 组件层次结构 |
| agent | agent |
| sequencer | sequencer |
| driver | driver |
| monitor | monitor |
| scoreboard | scoreboard / 记分板 |
| reference model | 参考模型 |
| predictor | predictor / 预测器 |
| checker | checker / 检查器 |
| assertion | assertion / 断言 |
| functional coverage | 功能覆盖率 |
| covergroup | covergroup |
| coverpoint | coverpoint |
| cross coverage | 交叉覆盖 |
| sequence | sequence / 序列 |
| virtual sequence | virtual sequence / 虚拟序列 |
| testcase | 测试用例 |
| regression | 回归测试 |
| plusarg | plusarg / 运行时参数 |
| factory override | factory override / 工厂覆盖 |
| configuration knob | 配置开关 / 配置参数 |
| analysis port/export/imp | analysis port/export/imp |
| TLM connection | TLM 连接 |
| expected stream | 期望流 |
| actual stream | 实际流 |
| matching rule | 匹配规则 |
| ordering model | 顺序模型 |
| outstanding transaction | 未完成事务 |
| reset/flush handling | reset/flush 处理 |
| evidence | 证据 |
| confidence | 置信度 |
| confirmed | 已确认 |
| inferred | 推断 |
| unknown / TODO | 未知 / 待确认 |
| gap | 缺口 |
| risk | 风险 |
| recommendation | 建议 |

Chinese writing style requirements:

- Use clear, engineering-style Simplified Chinese suitable for internal CPU/ASIC verification reviews.
- Use concise paragraphs and structured tables.
- Keep sentence tone objective and evidence-based.
- Use `已确认` / `推断` / `未知或待确认` to distinguish evidence levels.
- For ambiguous behavior, say `根据命名/连接关系推断` rather than presenting it as fact.
- For gaps, write actionable recommendations, for example: `建议补充该场景的 checker 或在 coverage model 中增加对应 coverpoint/cross。`
- Preserve original file paths and line references in evidence fields, for example: `证据：blk_dut/env/xxx_env.sv:83-96`.

Chinese evidence classification labels:

| Evidence level | Chinese label | Usage |
|---|---|---|
| Confirmed | 已确认 | Directly found in source code, scripts, documentation, filelists, or reports. |
| Inferred | 推断 | Strongly implied by names, connectivity, comments, transaction fields, or test structure. |
| Unknown / TODO | 未知 / 待确认 | Not enough evidence; needs owner confirmation. |

Chinese traceability status labels:

| English status | Chinese status |
|---|---|
| Covered by code | 代码已覆盖 |
| Partially covered | 部分覆盖 |
| Only inferred | 仅推断 |
| Gap | 缺口 |
| Not assessable | 不可评估 |

## When to Use This Skill

Use this skill when the user asks to:

- Read or analyze a whole UVM/SystemVerilog verification environment.
- Generate a testbench specification, verification environment specification, or verification plan from existing code.
- Document UVM components such as tests, envs, agents, sequencers, drivers, monitors, scoreboards, predictors, reference models, subscribers, virtual sequencers, sequences, assertions, and coverage collectors.
- Build traceability from DUT features to tests, sequences, checkers, scoreboards, and coverage points.
- Summarize an unfamiliar verification environment for onboarding, review, audit, methodology cleanup, or reuse.
- Identify verification gaps from code, tests, coverage, assertions, regressions, and TODO comments.

Do not use this skill when the user only wants a small UVM code snippet, a generic UVM tutorial, or a new testbench implementation. For those tasks, use a UVM coding or UVM verification methodology skill instead.

## Core Principle

Document only what can be supported by evidence in the repository.

Classify each finding as one of:

- **Confirmed**: directly found in source code, scripts, documentation, filelists, or regression results.
- **Inferred**: strongly implied by naming, connectivity, transaction fields, tests, or comments, but not explicitly documented.
- **Unknown / TODO**: not enough evidence; needs owner confirmation.

Never invent missing tests, coverage points, checkers, scoreboards, or requirements. When information is missing, explicitly mark it as a gap.

## Expected Inputs

The agent may receive any combination of:

- A repository or uploaded archive.
- A list of source files.
- Filelists such as `*.f`, `*.flist`, `filelist.f`, `compile.f`, `uvm.f`, `tb.f`.
- Simulation/regression scripts such as Makefiles, shell scripts, Python scripts, YAML/JSON regression lists, VCS/Xcelium/Questa command files.
- UVM source directories such as `tb/`, `sim/`, `verif/`, `uvm/`, `env/`, `agents/`, `seq/`, `sequences/`, `tests/`, `cov/`, `coverage/`, `scoreboard/`, `checker/`, `assertions/`, `sva/`, `ref_model/`, `model/`, `ral/`, `vip/`.
- Project-specific exact paths such as `blk_dut/env/`, `blk_dut/tb/`, `blk_dut/testcase/`, `blk_dut/sim/`, and `blk_dut/cfg/`.
- Design specifications, verification plans, architecture docs, interface protocols, or coverage reports.
- Desired output language, for example `output_language: english`, `output_language: simplified_chinese`.

If the repository is too large, first produce an index and then analyze by priority.


## Project-Specific Repository Paths

When the user knows the exact verification environment layout, list those paths directly in this skill or in a project-local copy of this skill. Treat these paths as first-class scan targets before falling back to generic discovery.

Default project root pattern:

```text
blk_dut/
```

Default UVM/testbench source map:

| Path | Expected contents | Analysis priority |
|---|---|---|
| `blk_dut/env/` | UVM environment classes, agents, monitors, drivers, sequencers, scoreboards, predictors, coverage collectors, reference models, env configuration classes | Highest |
| `blk_dut/tb/` | Testbench top, DUT wrapper, interfaces, clock/reset generation, bind files, package files, common TB utilities | Highest |
| `blk_dut/testcase/` | UVM tests, base tests, virtual sequences, directed/random/stress testcases, testcase-specific configuration | Highest |
| `blk_dut/sim/` | Filelists, Makefiles, simulator command scripts, regression scripts, coverage collection/merge scripts, waveform/debug setup | High |
| `blk_dut/cfg/` | Environment configuration files, YAML/JSON/testlist/regression configuration, plusarg defaults, feature switches, compile/runtime macro definitions | High |

Optional related paths to check if present:

| Path | Expected contents |
|---|---|
| `blk_dut/vip/` | External or internal VIP wrappers and protocol agents |
| `blk_dut/seq/` or `blk_dut/sequences/` | Shared sequence libraries if separated from testcase code |
| `blk_dut/cov/` or `blk_dut/coverage/` | Functional coverage models and coverage reports |
| `blk_dut/checker/`, `blk_dut/assertion/`, `blk_dut/sva/` | SVA, bind checkers, protocol assertions, formal/simulation assertions |
| `blk_dut/ref_model/` or `blk_dut/model/` | Reference models, predictors, DPI/C/C++/Python models |
| `blk_dut/ral/` or `blk_dut/reg_model/` | UVM RAL model, register adapters, predictors, register sequences |
| `blk_dut/doc/` or `blk_dut/docs/` | Specification, verification plan, README, known issues, methodology documents |

If the user provides a different root directory, replace `blk_dut/` with the provided root and preserve the same logical role mapping.

### How to Use Exact Paths

Use exact paths as the initial scan boundary:

1. Start with the listed project paths instead of scanning the entire repository.
2. Build a source inventory per path.
3. Record missing paths as `Not found`, not as errors, unless the user explicitly says every path must exist.
4. If a needed artifact is not found in the listed paths, expand search to nearby conventional paths and clearly label the expansion.
5. Preserve user-provided path names in the generated specification so reviewers can trace findings back to the real project structure.

Recommended path-specific scan commands:

```bash
PROJECT_ROOT=blk_dut
SCAN_DIRS="$PROJECT_ROOT/env $PROJECT_ROOT/tb $PROJECT_ROOT/testcase $PROJECT_ROOT/sim $PROJECT_ROOT/cfg"

for d in $SCAN_DIRS; do
  if [ -d "$d" ]; then
    echo "[SCAN] $d"
    find "$d" -type f \
      \( -name "*.sv" -o -name "*.svh" -o -name "*.v" -o -name "*.vh" \
         -o -name "*.f" -o -name "*.flist" -o -name "Makefile" \
         -o -name "*.mk" -o -name "*.sh" -o -name "*.py" \
         -o -name "*.pl" -o -name "*.tcl" \
         -o -name "*.yml" -o -name "*.yaml" -o -name "*.json" \
         -o -name "*.cfg" -o -name "*.ini" -o -name "*.txt" \
         -o -name "README*" -o -name "*.md" \)
  else
    echo "[MISSING] $d"
  fi
done
```

Recommended symbol scan over exact paths:

```bash
rg -n "class\s+\w+\s+extends\s+uvm_|`uvm_(component|object)_utils|run_test|uvm_config_db|analysis_port|analysis_export|analysis_imp|uvm_tlm_analysis_fifo|covergroup|coverpoint|cross|assert\s+property|assume\s+property|cover\s+property|bind\s+|sequence\s+\w+|property\s+\w+|\+UVM_TESTNAME|\$value\$plusargs|\$test\$plusargs" \
  blk_dut/env blk_dut/tb blk_dut/testcase blk_dut/sim blk_dut/cfg
```

### Path-to-Document Mapping

When generating the final testbench specification, map each path to document sections as follows:

| Source path | Primary document sections |
|---|---|
| `blk_dut/tb/` | DUT/testbench top integration, interfaces, clock/reset, bind/checker setup, package/import setup |
| `blk_dut/env/` | UVM architecture, component hierarchy, agents, scoreboards, reference models, coverage collectors, checkers, configuration strategy |
| `blk_dut/testcase/` | Testcase specification, sequences, stimulus strategy, factory overrides, plusargs, test intent |
| `blk_dut/sim/` | Regression and execution flow, compile/run commands, simulator options, seed strategy, coverage merge, debug/wave options |
| `blk_dut/cfg/` | Configuration matrix, feature switches, runtime knobs, testlists, regression grouping, environment modes |

### Exact File Path Mode

If the user provides exact file paths rather than directories, analyze exactly those files first. Then use include statements, filelists, package imports, and script references to discover dependent files.

Use this priority order:

1. User-provided exact file paths.
2. Files referenced by user-provided filelists or package includes.
3. Files under the user-provided directories.
4. Conventional nearby paths, only when required to close obvious gaps.

Always report the final analyzed file set in the `Repository and Source Inventory` section.

## High-Level Workflow

### 1. Build the repository map

Identify important files and directories before reading details. If `Project-Specific Repository Paths` are listed, scan those paths first and use generic discovery only to fill gaps.

Look for:

- Top-level README and documentation.
- Filelists and package files.
- Testbench top modules.
- UVM package include order.
- Environment, agent, sequence, test, scoreboard, checker, coverage, RAL, and reference model directories.
- Regression scripts and test lists.
- Coverage databases or coverage reports, if present.

Suggested search commands:

```bash
find . -maxdepth 4 -type f \
  \( -name "*.sv" -o -name "*.svh" -o -name "*.v" -o -name "*.vh" \
     -o -name "*.f" -o -name "*.flist" -o -name "Makefile" \
     -o -name "*.mk" -o -name "*.sh" -o -name "*.py" \
     -o -name "*.yml" -o -name "*.yaml" -o -name "*.json" \
     -o -name "README*" -o -name "*.md" \)
```

```bash
rg -n "class\s+\w+\s+extends\s+uvm_|`uvm_(component|object)_utils|run_test|uvm_config_db|analysis_port|analysis_export|analysis_imp|covergroup|assert\s+property|cover\s+property|sequence\s+\w+|property\s+\w+" .
```

### 2. Identify testbench entry points

Find the simulation entry and DUT connection layer:

- `module tb_top`, `module testbench`, `module *_tb`, or project-specific top.
- DUT instantiation.
- Interface instantiations.
- Clock/reset generation.
- `run_test()` call.
- `uvm_config_db::set()` calls for virtual interfaces, knobs, configuration objects, or plusargs.
- Bind files for SVA/checkers.
- Wave dumping, timeout, reset sequencing, and global simulation controls.

Document:

- TB top module name and file path.
- DUT instance path.
- Interface instances and which DUT ports they connect to.
- Clock/reset domains and frequencies, if available.
- Config DB setup and run-time options.
- Any conditional compilation macros that change TB behavior.

### 3. Parse UVM package and include order

Find packages such as:

- `*_pkg.sv`
- `*_uvm_pkg.sv`
- `tb_pkg.sv`
- `env_pkg.sv`
- `test_pkg.sv`

Extract:

- Imported packages.
- Included files and order.
- Dependencies between transaction classes, sequences, agents, env, tests, coverage, and scoreboards.
- Macro definitions affecting environment structure.

Document any risk from include-order dependencies or duplicate class names.

### 4. Build the UVM class inventory

For every class extending a UVM base class, capture:

- Class name.
- File path.
- Base class.
- Factory registration macro.
- Key member variables.
- UVM phases implemented.
- Constructor default name.
- Configuration object or virtual interface usage.
- Analysis ports/exports/imps/FIFOs.
- Sequence item type parameters.
- Comments or TODOs that explain intent.

Common class categories:

| Category | Common base classes / patterns |
|---|---|
| Test | `uvm_test` |
| Environment | `uvm_env` |
| Agent | `uvm_agent` |
| Sequencer | `uvm_sequencer #(item_t)` |
| Driver | `uvm_driver #(item_t)` |
| Monitor | `uvm_monitor` |
| Sequence | `uvm_sequence #(item_t)` |
| Transaction | `uvm_sequence_item`, `uvm_object` |
| Scoreboard | `uvm_scoreboard`, sometimes `uvm_component` |
| Coverage | `uvm_subscriber`, `uvm_component`, embedded `covergroup` |
| Reference model / predictor | `uvm_component`, `uvm_subscriber`, project-specific model classes |
| RAL | `uvm_reg_block`, `uvm_reg`, `uvm_reg_adapter`, `uvm_reg_predictor` |
| Virtual sequence | `uvm_sequence`, contains handles to multiple sequencers |
| Virtual sequencer | `uvm_sequencer`, contains handles to sub-sequencers |

### 5. Recover component hierarchy

Use `type_id::create()` calls in `build_phase()` to determine hierarchy.

Search patterns:

```systemverilog
x = some_class::type_id::create("instance_name", this);
x = new("instance_name", this);
```

Document hierarchy as both a tree and a table.

Example tree format:

```text
uvm_test_top
└── <test_class>
    └── env : <env_class>
        ├── axi_mst_agent : <agent_class> [active]
        │   ├── sequencer
        │   ├── driver
        │   └── monitor
        ├── axi_slv_agent : <agent_class> [passive/active]
        ├── scoreboard
        ├── coverage_collector
        └── virtual_sequencer
```

For each component, include:

- Instance name.
- Type/class name.
- Parent component.
- Active/passive mode if known.
- Created conditionally? If yes, list condition.
- Confidence level.

### 6. Recover connectivity

Use `connect_phase()` and other setup code to extract communication paths.

Search for:

- `connect(`
- `.analysis_port.connect(`
- `.analysis_export.connect(`
- `uvm_tlm_analysis_fifo`
- `uvm_analysis_imp_decl`
- `seq_item_port.connect(sequencer.seq_item_export)`
- Virtual sequencer handle assignment.
- RAL predictor/adapter connections.

Document:

- Driver-to-sequencer connections.
- Monitor-to-scoreboard connections.
- Monitor-to-coverage connections.
- Monitor-to-reference-model/predictor connections.
- Scoreboard expected/actual channels.
- RAL predictor and adapter paths.
- Any sideband event, mailbox, queue, or custom TLM communication.

Connectivity table format:

| Source | Port/export | Destination | Purpose | Evidence | Confidence |
|---|---|---|---|---|---|

### 7. Analyze interfaces and transaction types

For each SystemVerilog `interface` and UVM transaction:

- Identify signals, modports, clocking blocks, reset, protocol-specific fields.
- Identify transaction fields, constraints, enums, helper methods, compare/copy/print/convert2string methods.
- Map transaction fields to interface signals when possible.
- Capture randomization constraints and legal value ranges.
- Capture protocol assumptions implied by monitor/driver behavior.

Transaction table format:

| Transaction | Fields | Randomized fields | Constraints | Used by | Purpose |
|---|---|---|---|---|---|

Interface table format:

| Interface | Clock/reset | Signals | Modports/clocking blocks | Connected DUT ports | Used by components |
|---|---|---|---|---|---|

### 8. Analyze agents

For each agent:

- Protocol/interface type.
- Active/passive control.
- Driver behavior.
- Monitor sampling behavior.
- Sequencer and sequence item type.
- Config object fields.
- Virtual interface retrieval.
- Analysis output transactions.
- Reset handling.
- Error injection or protocol violation support.

Agent section format:

```markdown
### <agent_name> Agent

- **Class**: `<class_name>`
- **Files**: `<paths>`
- **Protocol/interface**: `<interface or inferred protocol>`
- **Mode**: Active / Passive / Configurable / Unknown
- **Subcomponents**: driver, monitor, sequencer, coverage, etc.
- **Input/output transactions**: `<transaction classes>`
- **Driver responsibilities**: ...
- **Monitor responsibilities**: ...
- **Configuration knobs**: ...
- **Reset behavior**: ...
- **Known gaps / TODOs**: ...
```

### 9. Analyze tests, sequences, and stimulus strategy

Find all classes extending `uvm_test` and `uvm_sequence`.

For tests, extract:

- Environment creation.
- Factory overrides.
- Config DB settings.
- Plusargs.
- Sequence starts.
- Virtual sequence usage.
- Objection handling.
- Drain time, timeouts, watchdogs.
- Test intent from comments/log messages/class names.

For sequences, extract:

- Sequence item type.
- Randomization constraints.
- Directed behavior.
- Error injection.
- Stress/random loops.
- Register access patterns.
- Dependencies on virtual sequencer or config.
- Any `pre_body`, `body`, `post_body` behavior.

Testcase table format:

| Test name | File | Main sequence(s) | Configuration/plusargs | Verification intent | Expected checking | Coverage target | Confidence |
|---|---|---|---|---|---|---|---|

Sequence table format:

| Sequence | File | Item type | Behavior | Constraints | Started by tests | Notes |
|---|---|---|---|---|---|---|

If test intent is inferred only from naming, explicitly say so.

### 10. Analyze checkers and assertions

Find checkers implemented as:

- SVA `assert property`, `assume property`, `cover property`.
- `checker` constructs.
- Bound assertion modules.
- Immediate assertions.
- UVM monitor protocol checks using `uvm_error`, `uvm_fatal`, or custom error counters.
- Scoreboard comparison checks.
- End-of-test checks in `check_phase()` or `report_phase()`.

Search patterns:

```bash
rg -n "assert\s+property|assume\s+property|cover\s+property|\bchecker\b|bind\s+|`uvm_error|`uvm_fatal|check_phase|report_phase|compare\(" .
```

Checker/assertion table format:

| Checker/assertion | Type | Location | Trigger/clock | What it checks | Severity/action | Related feature | Confidence |
|---|---|---|---|---|---|---|---|

Classify assertions:

- Protocol compliance.
- Data integrity.
- Ordering.
- Liveness/progress.
- Reset behavior.
- X/Z checking.
- Error handling.
- Security/safety/RAS.
- Low-power behavior.
- Performance/fairness.

### 11. Analyze scoreboards, predictors, and reference models

For each scoreboard/reference model:

- Input transaction streams.
- Expected-vs-actual matching rule.
- Ordering assumption: in-order, out-of-order, per-ID ordering, per-address ordering, per-flow ordering.
- Queues, associative arrays, FIFOs, scoreboarding keys.
- Data transformation or prediction model.
- Error conditions.
- End-of-test checks for outstanding transactions.
- Reset/flush behavior.
- Known limitations.

Scoreboard section format:

```markdown
### <scoreboard_name>

- **Class/file**: ...
- **Inputs**: expected stream(s), actual stream(s), monitor(s)
- **Matching key**: ID/address/tag/order/transaction content/unknown
- **Ordering model**: in-order / out-of-order / partial order / unknown
- **Reference model/predictor**: ...
- **Mismatch reporting**: ...
- **End-of-test criteria**: ...
- **Outstanding transaction handling**: ...
- **Reset/flush handling**: ...
- **Gaps or risks**: ...
```

### 12. Analyze functional coverage

Find coverage in:

- `covergroup` declarations.
- `coverpoint`, `bins`, `ignore_bins`, `illegal_bins`, `cross`.
- `uvm_subscriber` classes.
- Transaction classes with embedded covergroups.
- Monitor sampling methods.
- Cover properties.
- Coverage reports if present.

Coverage table format:

| Coverage model | Location | Sample event | Coverpoints | Crosses | Related feature | Related tests/sequences | Gaps |
|---|---|---|---|---|---|---|---|

For each covergroup, document:

- Sampling trigger.
- Data source.
- Coverpoints and bin intent.
- Cross coverage intent.
- Illegal/ignore bins.
- Whether it is connected and sampled in the actual environment.

Important: a covergroup definition does not prove coverage is sampled. Confirm where `.sample()` is called or where the covergroup event is triggered.

### 13. Analyze regression and execution flow

Inspect Makefiles, scripts, CI configs, and regression lists.

Extract:

- Compile command and simulator.
- Filelist used.
- Test invocation style, such as `+UVM_TESTNAME=<test>`.
- Random seed control.
- Plusargs and runtime knobs.
- Regression test list.
- Timeout and pass/fail criteria.
- Coverage collection/merge commands.
- Wave/debug options.
- Lint/formal/static checks, if included.

Regression table format:

| Regression target | Tests included | Seeds | Plusargs | Coverage enabled | Pass/fail criteria | Script/file |
|---|---|---|---|---|---|---|

### 14. Build feature-to-verification traceability

Create a traceability matrix using all available evidence.

Traceability table format:

| Feature / scenario | Stimulus / sequence | Test(s) | Checker / scoreboard | Coverage point(s) | Status | Evidence |
|---|---|---|---|---|---|---|

Status values:

- **Covered by code**: stimulus, checking, and coverage found.
- **Partially covered**: at least one of stimulus/checking/coverage missing.
- **Only inferred**: naming or comments imply coverage, but implementation evidence is weak.
- **Gap**: expected feature has no visible verification artifact.
- **Not assessable**: design requirement is unknown.

### 15. Generate the final specification document

The final document should be readable by verification engineers, design owners, project leads, and reviewers.

Use this structure unless the user requests another format:

```markdown
# <DUT/Project> UVM Testbench Specification

## 1. Document Control
- Generated from repository/version:
- Date:
- Author/tool:
- Scope:
- Inputs analyzed:
- Confidence summary:

## 2. Executive Summary
- Environment purpose:
- Main protocols/interfaces:
- Main verification strategy:
- Key checkers/scoreboards:
- Main coverage strategy:
- Major gaps/risks:

## 3. Repository and Source Inventory
- User-provided scan root/path map:
- Directory map:
- Analyzed file set:
- Missing expected paths, if any:
- Key packages/filelists/scripts:
- Important compile macros:

## 4. DUT and Testbench Top Integration
- DUT instance and top module:
- Interfaces and port connections:
- Clock/reset generation:
- Config DB setup:
- Bind/assertion setup:

## 5. UVM Architecture Overview
- Component hierarchy diagram/tree:
- Build/connect/run phase summary:
- Configuration strategy:
- Factory override strategy:

## 6. Environment Components
- Environment class:
- Agents:
- Virtual sequencer:
- Scoreboards:
- Reference models/predictors:
- Coverage collectors:
- Utility components:

## 7. Agent Specifications
- One subsection per agent.

## 8. Transaction and Interface Models
- Transaction classes:
- Interfaces:
- Mapping between interface signals and transaction fields:

## 9. Stimulus and Sequence Strategy
- Sequence library:
- Directed tests:
- Random/stress tests:
- Error injection tests:
- Reset/low-power/security/performance scenarios, if present:

## 10. Testcase Specification
- Test list table:
- Per-test intent:
- Main sequences/configuration:
- Expected pass/fail behavior:

## 11. Checker and Assertion Specification
- SVA/bind checkers:
- Monitor checks:
- Scoreboard checks:
- End-of-test checks:

## 12. Scoreboard and Reference Model Specification
- Expected/actual data paths:
- Matching rules:
- Ordering model:
- Outstanding transaction handling:
- Reset/flush handling:

## 13. Functional Coverage Specification
- Coverage architecture:
- Covergroups/coverpoints/crosses:
- Sampling points:
- Coverage-to-feature mapping:
- Missing coverage:

## 14. Regression and Execution Flow
- Compile/run commands:
- Test selection:
- Seeds:
- Coverage collection:
- Debug/wave options:
- CI/regression targets:

## 15. Feature Traceability Matrix
- Feature -> stimulus -> tests -> checker/scoreboard -> coverage.

## 16. Known Gaps, Risks, and Recommendations
- Missing documentation:
- Unconnected components:
- Unsampled coverage:
- Tests with weak checking:
- Scoreboard limitations:
- Assertion gaps:
- Suggested next actions:

## 17. Appendix
- Class inventory:
- File inventory:
- Search terms used:
- Assumptions and inferred items:
```

### Simplified Chinese Final Specification Template

When the user requests Simplified Chinese output, use this structure unless the user requests another format:

```markdown
# <DUT/Project> UVM 验证环境 / Testbench 规格说明

## 1. 文档控制
- 仓库 / 版本来源：
- 生成日期：
- 作者 / 工具：
- 分析范围：
- 已分析输入：
- 置信度总结：
- 输出语言：简体中文

## 2. 执行摘要
- 验证环境目标：
- 主要协议 / 接口：
- 主要验证策略：
- 关键 checker / scoreboard：
- 主要覆盖率策略：
- 主要缺口 / 风险：

## 3. 仓库与源码清单
- 用户提供的扫描根目录 / 路径映射：
- 目录结构说明：
- 已分析文件集合：
- 缺失的预期路径：
- 关键 package / filelist / script：
- 重要编译宏：

## 4. DUT 与 Testbench 顶层集成
- DUT 实例与顶层模块：
- 接口与端口连接：
- 时钟 / 复位生成：
- Config DB 设置：
- bind / assertion 接入方式：

## 5. UVM 架构总览
- 组件层次结构图 / 树：
- build/connect/run phase 摘要：
- 配置策略：
- factory override 策略：

## 6. 验证环境组件
- env 类：
- agents：
- virtual sequencer：
- scoreboards：
- reference models / predictors：
- coverage collectors：
- utility components：

## 7. Agent 规格说明
- 每个 agent 一个小节。

## 8. Transaction 与 Interface 模型
- transaction 类：
- interfaces：
- interface 信号与 transaction 字段映射：

## 9. 激励与 Sequence 策略
- sequence library：
- directed tests：
- random / stress tests：
- error injection tests：
- reset / low-power / security / performance 场景：

## 10. 测试用例规格说明
- 测试列表：
- 每个测试的验证意图：
- 主要 sequence / 配置：
- 预期 pass/fail 行为：

## 11. Checker 与 Assertion 规格说明
- SVA / bind checkers：
- monitor checks：
- scoreboard checks：
- end-of-test checks：

## 12. Scoreboard 与 Reference Model 规格说明
- expected / actual 数据路径：
- 匹配规则：
- 顺序模型：
- 未完成事务处理：
- reset / flush 处理：

## 13. 功能覆盖率规格说明
- 覆盖率架构：
- covergroups / coverpoints / crosses：
- 采样点：
- coverage-to-feature 映射：
- 缺失覆盖：

## 14. 回归测试与执行流程
- 编译 / 运行命令：
- 测试选择方式：
- seed 策略：
- 覆盖率收集：
- debug / wave 选项：
- CI / regression targets：

## 15. Feature Traceability Matrix
- feature -> stimulus -> tests -> checker/scoreboard -> coverage。

## 16. 已知缺口、风险与建议
- 缺失文档：
- 未连接组件：
- 未采样 coverage：
- 检查较弱的测试：
- scoreboard 限制：
- assertion 缺口：
- 建议后续动作：

## 17. 附录
- class inventory：
- file inventory：
- 使用的搜索关键字：
- 假设与推断项：
```

Recommended Chinese table headers:

```markdown
| 项目 | 说明 | 文件 / 位置 | 证据 | 置信度 |
|---|---|---|---|---|
```

```markdown
| 测试用例 | 文件 | 主要 sequence | 配置 / plusargs | 验证意图 | 预期检查方式 | 覆盖目标 | 置信度 |
|---|---|---|---|---|---|---|---|
```

```markdown
| Coverage model | 位置 | 采样事件 | Coverpoints | Crosses | 关联 feature | 关联测试 / sequence | 缺口 |
|---|---|---|---|---|---|---|---|
```

```markdown
| Feature / 场景 | Stimulus / sequence | 测试 | Checker / scoreboard | Coverage points | 状态 | 证据 |
|---|---|---|---|---|---|---|
```

## Evidence and Citation Style

When producing the document, include evidence for important findings.

Preferred evidence format:

```markdown
- `axi_mst_agent` is created by `soc_env` in `build_phase()` and appears to be active when `cfg.axi_mst_active == 1`. Evidence: `verif/env/soc_env.sv:83-96`.
```

For Simplified Chinese output, use the same evidence discipline with Chinese prose and original code identifiers:

```markdown
- `axi_mst_agent` 由 `soc_env` 在 `build_phase()` 中创建；当 `cfg.axi_mst_active == 1` 时推断为 active 模式。证据：`blk_dut/env/soc_env.sv:83-96`。
```

For tables, use an `Evidence` column with file paths and line ranges where possible.

If exact line numbers are unavailable, cite the file path and symbol name.

## Handling Large Repositories

For large codebases, analyze in passes:

1. **Index pass**: source tree, packages, filelists, class inventory.
2. **Architecture pass**: TB top, env hierarchy, agent inventory, connect-phase graph.
3. **Behavior pass**: drivers, monitors, sequences, scoreboards, coverage, assertions.
4. **Regression pass**: tests, scripts, plusargs, coverage flow.
5. **Spec pass**: write document with evidence and gaps.

If time is limited, prioritize:

1. TB top and UVM package.
2. Environment build/connect phases.
3. Agents and transaction classes.
4. Scoreboards/checkers.
5. Tests/sequences.
6. Coverage.
7. Regression scripts.

## Important Heuristics

### UVM class extraction

Search for:

```systemverilog
class <name> extends uvm_<base>
`uvm_component_utils(<name>)
`uvm_object_utils(<name>)
`uvm_component_utils_begin(<name>)
`uvm_object_utils_begin(<name>)
```

### Component creation

Search for:

```systemverilog
<handle> = <class>::type_id::create("<instance>", this);
<handle> = new("<instance>", this);
```

### Virtual interface configuration

Search for:

```systemverilog
uvm_config_db#(virtual <if_type>)::set(...)
uvm_config_db#(virtual <if_type>)::get(...)
```

### Factory overrides

Search for:

```systemverilog
set_type_override_by_type(...)
set_inst_override_by_type(...)
set_type_override(...)
set_inst_override(...)
```

### Sequence starts

Search for:

```systemverilog
seq.start(...)
`uvm_do(...)
`uvm_do_on(...)
`uvm_do_with(...)
start_item(req)
finish_item(req)
```

### Analysis connections

Search for:

```systemverilog
uvm_analysis_port
uvm_analysis_export
uvm_analysis_imp
uvm_tlm_analysis_fifo
.analysis_port.connect(...)
.write(...)
```

### Coverage

Search for:

```systemverilog
covergroup
coverpoint
bins
ignore_bins
illegal_bins
cross
.sample()
cover property
```

### Checkers

Search for:

```systemverilog
assert property
assume property
cover property
bind
`uvm_error
`uvm_fatal
check_phase
report_phase
```

### Tests and plusargs

Search for:

```systemverilog
class <name> extends uvm_test
$value$plusargs
$test$plusargs
+UVM_TESTNAME
raise_objection
drop_objection
```

## Quality Checklist Before Finalizing the Spec

Before delivering the document, verify:

- Every named component in the hierarchy has a class and file path.
- Every major analysis connection is captured.
- Every scoreboard has input streams and matching rules documented or marked unknown.
- Every coverage model is checked for actual sampling.
- Every test has main sequence/configuration captured or marked unknown.
- Assertions/checkers are separated from scoreboards and monitor checks.
- Inferred test intent is clearly labeled as inferred.
- Unused, unconnected, or dead code is not presented as active without evidence.
- Compile macros and configuration knobs that change topology are documented.
- Feature traceability distinguishes stimulus, checking, and coverage.
- Gaps are explicit and actionable.

## Common Pitfalls

Avoid these mistakes:

- Treating file names as proof of behavior without reading the code.
- Assuming every `covergroup` is sampled.
- Assuming every sequence is used by a test.
- Assuming every monitor output is connected to a scoreboard.
- Ignoring factory overrides that replace base components.
- Ignoring compile-time macros that change environment topology.
- Confusing monitor protocol checks with scoreboard end-to-end checks.
- Missing bind-based assertions because they live outside the UVM directory.
- Missing C/C++/Python reference models called through DPI or scripts.
- Missing tests defined only in regression YAML, Makefile variables, or command-line plusargs.
- Presenting incomplete inferred feature coverage as confirmed coverage.

## Recommended Final Response to User

When returning the generated specification, include:

- A short summary of what was analyzed.
- The generated document or a link to it.
- A confidence statement.
- The top gaps/risks.
- Suggested next actions, such as owner review, requirement mapping, coverage closure, or adding missing checkers.
- If Simplified Chinese output was requested, write the response summary in Simplified Chinese and call out that code identifiers/file paths were preserved in their original form.

Example:

```markdown
I generated the UVM testbench specification from the provided repository. The strongest evidence is around the UVM hierarchy, agent connections, tests, and scoreboards. Coverage intent is partially documented, but several covergroups appear unsampled and should be reviewed.

Top gaps:
1. No confirmed end-to-end checker for outstanding transaction drain on reset.
2. Several sequences are compiled but not referenced by any regression test.
3. Coverage model exists for burst type, but no cross with error response is sampled.
```


Simplified Chinese response example:

```markdown
已基于提供的 UVM 验证环境生成中文 Testbench 规格说明。本文档保留了原始 class、signal、file path、plusarg 和 macro 名称，并将说明性内容整理为简体中文。

当前证据最充分的部分包括 UVM 组件层次、agent 连接关系、测试用例与 scoreboard。coverage 部分存在若干仅定义但未确认采样的 covergroup，建议 owner 进一步 review。

主要缺口：
1. reset 后 outstanding transaction drain 的 end-to-end checker 尚未确认。
2. 若干 sequence 已编译但未发现被 regression test 调用。
3. 已发现 burst type coverage，但未确认与 error response 的 cross coverage。
```
