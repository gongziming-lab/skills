# UVM 验证环境效率分析与优化建议 Skill

## 1. Skill 名称

`uvm-env-performance-reviewer`

## 2. 适用场景

本 Skill 用于读取和理解一个现有的 UVM-based SystemVerilog 验证环境，并从芯片验证专家视角分析其中可能导致仿真效率下降的编码风格、架构设计和使用方式，最终输出可执行的优化建议。

适用对象包括：

- CPU / GPU / NPU / SoC / Subsystem / IP 级 UVM 验证环境
- 已有 regression 运行较慢的验证平台
- 需要做 testbench performance review 的项目
- 需要定位 UVM monitor、scoreboard、coverage、reference model、sequence、logger 等组件性能瓶颈的场景
- 需要给验证 leader / module owner / testbench owner 输出优化建议的场景

---

## 3. 输入信息

使用本 Skill 时，尽量提供以下输入。

### 3.1 必需输入

- UVM 验证环境源码路径，例如：
  - `tb/`
  - `env/`
  - `agents/`
  - `scoreboard/`
  - `cov/`
  - `ref_model/`
  - `seq/`
  - `tests/`
  - `sim/`
- DUT 接口 monitor / driver / sequencer / scoreboard / coverage 的主要源码文件
- 仿真器和 UVM 版本，例如：
  - VCS + UVM-1.1d / UVM-1.2
  - Xcelium + UVM-1.2
  - Questa + UVM-1.2
- 典型 regression 或慢 test 的运行命令
- 当前性能问题描述，例如：
  - 某类 test 明显变慢
  - coverage 开启后变慢
  - scoreboard 开启后变慢
  - debug log 开启后变慢
  - 某个 agent 接入后变慢
  - 长随机用例后半段越来越慢
  - memory usage 不断上涨

### 3.2 可选但强烈建议输入

- 慢 test 的仿真耗时对比数据
- 开关项配置，例如：
  - `+UVM_VERBOSITY`
  - coverage enable
  - scoreboard enable
  - trace enable
  - waveform enable
  - checker enable
- 仿真 profile 报告，例如：
  - VCS profiler
  - Xcelium profiler
  - Questa profiler
  - 自定义 transaction counter
  - memory usage log
- regression 统计信息：
  - test name
  - seed
  - simulation time
  - wall time
  - CPU time
  - memory peak
  - transaction count
  - coverage sample count

---

## 4. 输出目标

本 Skill 的输出应包括：

1. 验证环境结构理解
2. 潜在仿真效率瓶颈列表
3. 每个瓶颈的代码位置、证据和影响分析
4. 可执行的优化建议
5. 优化优先级排序
6. 修改风险评估
7. 推荐验证方法，确保优化不改变 testbench 行为
8. 可选的代码重构示例

---

## 5. 总体工作流程

### Step 1：建立 UVM 环境结构地图

首先不要急于给优化建议，应先理解验证环境结构。

需要识别：

- top testbench
- base test
- env
- agent
- sequencer
- driver
- monitor
- scoreboard
- coverage collector
- reference model
- virtual sequencer
- virtual sequence
- configuration object
- register model
- analysis port / analysis export / analysis fifo 连接关系
- checker / assertion / bind 文件
- common utility package

输出一份结构地图，例如：

```text
 tb_top
  ├── uvm_test_top
  │   ├── cpu_base_test
  │   └── cpu_env
  │       ├── ifu_agent
  │       │   ├── sequencer
  │       │   ├── driver
  │       │   └── monitor
  │       ├── lsu_agent
  │       │   ├── sequencer
  │       │   ├── driver
  │       │   └── monitor
  │       ├── scoreboard
  │       ├── coverage_collector
  │       ├── ref_model
  │       └── virtual_sequencer
```

重点理解数据流：

```text
 driver -> DUT interface
 DUT interface -> monitor
 monitor -> analysis_port -> scoreboard / coverage / logger / ref_model
 sequence -> sequencer -> driver
```

---

### Step 2：识别高频路径

高频路径是性能优化的重点。

优先检查以下代码区域：

- monitor 的 `run_phase`
- driver 的 `run_phase`
- scoreboard 的 `write()` / `compare()` / `check_phase`
- coverage collector 的 `write()` / `sample()`
- reference model 的 transaction step 函数
- sequence 中的 transaction generation loop
- analysis port fanout 路径
- 每拍执行的 checker
- 每笔 transaction 执行的 callback
- logger / trace collector
- config object 读取路径

需要判断每段代码的调用频率：

| 代码位置 | 调用频率 | 性能风险 |
|---|---:|---|
| per-cycle monitor loop | 极高 | 高 |
| per-transaction scoreboard write | 高 | 高 |
| per-instruction reference model step | 高 | 高 |
| build_phase config_db get | 低 | 低 |
| end_of_test summary print | 低 | 低 |

原则：

> 不要把重操作放在 per-cycle 或 per-transaction 热路径里。

---

## 6. 重点检查项

### 6.1 无保护的 UVM log 和字符串格式化

重点搜索：

```bash
rg "uvm_info|uvm_warning|uvm_error|uvm_fatal"
rg "\\$sformatf|convert2string|sprint|print\\("
```

重点检查以下模式：

```systemverilog
`uvm_info("MON",
  $sformatf("addr=%0h data=%0h id=%0d", addr, data, id),
  UVM_HIGH)
```

风险：

- 即使 log 不打印，字符串格式化也可能已经发生
- `convert2string()` 可能递归访问复杂 object
- per-cycle / per-transaction log 会显著拖慢仿真

建议改法：

```systemverilog
if (uvm_report_enabled(UVM_HIGH, UVM_INFO, "MON")) begin
  `uvm_info("MON",
    $sformatf("addr=%0h data=%0h id=%0d", addr, data, id),
    UVM_HIGH)
end
```

进一步建议：

- debug log 默认关闭
- 高频路径避免复杂字符串拼接
- 大量 trace 使用专门 trace 开关
- 只在 mismatch 或关键事件发生时打印完整 object

---

### 6.2 monitor 中每拍创建 transaction object

重点搜索：

```bash
rg "type_id::create|new\\("
rg "forever|always"
```

重点检查：

```systemverilog
forever begin
  @(posedge vif.clk);

  tr = my_trans::type_id::create("tr");

  tr.valid = vif.valid;
  tr.addr  = vif.addr;
  tr.data  = vif.data;

  if (vif.valid) begin
    ap.write(tr);
  end
end
```

风险：

- valid 为 0 时仍然创建 object
- 长仿真中 object allocation 数量巨大
- 增加内存和仿真器对象管理压力

建议改法：

```systemverilog
forever begin
  @(posedge vif.clk);

  if (vif.valid && vif.ready) begin
    tr = my_trans::type_id::create("tr");

    tr.addr = vif.addr;
    tr.data = vif.data;
    tr.id   = vif.id;

    ap.write(tr);
  end
end
```

优化原则：

- 只在有效 transaction 发生时创建 object
- packet 完整之后再创建 object
- 对极高频简单事件可考虑轻量 struct
- 对大量临时 object 可考虑 object pool，但要谨慎维护生命周期

---

### 6.3 频繁深拷贝 transaction

重点搜索：

```bash
rg "\\.copy\\(|\\.clone\\(|do_copy|copy\\("
```

典型低效写法：

```systemverilog
function void write(my_trans tr);
  my_trans tr_copy;

  tr_copy = my_trans::type_id::create("tr_copy");
  tr_copy.copy(tr);

  expected_q.push_back(tr_copy);
endfunction
```

风险：

- transaction 中可能包含大数组、payload、metadata、trace 字符串
- scoreboard / coverage / logger 多处 copy 会叠加放大
- 内存增长明显

建议：

- scoreboard 只保存比较所需字段
- coverage 使用轻量 transaction
- logger 不要保存完整 transaction
- ref model 和 scoreboard 之间尽量传递 key 或摘要信息

示例：

```systemverilog
typedef struct packed {
  bit [7:0]  id;
  bit [39:0] addr;
  bit [3:0]  opcode;
} sb_key_t;
```

---

### 6.4 scoreboard 中线性搜索大队列

重点搜索：

```bash
rg "foreach .*q|for .*size\\(|find_index|delete\\(|pop_front"
```

典型低效写法：

```systemverilog
function void write_actual(my_trans act);
  foreach (expected_q[i]) begin
    if (expected_q[i].id == act.id &&
        expected_q[i].addr == act.addr) begin
      compare(expected_q[i], act);
      expected_q.delete(i);
      return;
    end
  end

  `uvm_error("SB", "No expected item found")
endfunction
```

风险：

- 每个 actual 都扫描 expected queue
- outstanding 多时变成 O(N²)
- queue 中间 delete 可能导致大量元素移动
- 长随机用例后期越来越慢

建议使用 associative array 建索引：

```systemverilog
typedef struct packed {
  bit [7:0]  id;
  bit [39:0] addr;
} sb_key_t;

my_trans expected_by_key[sb_key_t][$];

function void write_expected(my_trans exp);
  sb_key_t key;

  key.id   = exp.id;
  key.addr = exp.addr;

  expected_by_key[key].push_back(exp);
endfunction

function void write_actual(my_trans act);
  sb_key_t key;
  my_trans exp;

  key.id   = act.id;
  key.addr = act.addr;

  if (expected_by_key.exists(key) &&
      expected_by_key[key].size() > 0) begin
    exp = expected_by_key[key].pop_front();
    compare(exp, act);

    if (expected_by_key[key].size() == 0) begin
      expected_by_key.delete(key);
    end
  end else begin
    `uvm_error("SB", "No expected item found")
  end
endfunction
```

优化原则：

- in-order 比较可用 FIFO
- out-of-order 比较应使用 key-based lookup
- 同 key 多 outstanding 时使用 associative array + queue
- 避免大 queue 中间 delete
- 对 timeout / aging 检查使用周期性低频扫描，而不是每笔 transaction 全量扫描

---

### 6.5 coverage 过度采样

重点搜索：

```bash
rg "covergroup|coverpoint|cross|sample\\("
```

典型低效写法：

```systemverilog
forever begin
  @(posedge vif.clk);
  cg.sample();
end
```

风险：

- 每拍 sample，即使没有有效事件
- cross coverage bin 数爆炸
- covergroup 参数过多
- coverage collector 接收所有 transaction，但实际只关心少数事件

建议改法：

```systemverilog
if (vif.valid && vif.ready) begin
  cg.sample();
end
```

进一步建议：

```systemverilog
if (enable_cov && tr.is_architecturally_visible()) begin
  cg.sample(tr.opcode, tr.size, tr.priv_mode);
end
```

检查 cross coverage：

```systemverilog
cross cp_opcode, cp_src, cp_dst, cp_size, cp_priv;
```

风险：

- 多维 cross 可能产生巨大 bin 数
- 很多组合没有验证意义
- illegal / ignore bins 不完整

建议：

- 只 cross 与验证目标强相关的字段
- 使用 ignore_bins 排除无意义组合
- 按 feature 拆分 covergroup
- 对性能敏感 regression 关闭非必要 coverage
- 区分 smoke / nightly / weekly coverage 模式

---

### 6.6 analysis port fanout 过大

重点搜索：

```bash
rg "\\.connect\\(.*analysis|analysis_port|analysis_export|uvm_analysis_imp|uvm_tlm_analysis_fifo"
```

典型结构：

```systemverilog
monitor.ap.connect(scoreboard.analysis_export);
monitor.ap.connect(coverage.analysis_export);
monitor.ap.connect(logger.analysis_export);
monitor.ap.connect(ref_model.analysis_export);
monitor.ap.connect(trace_collector.analysis_export);
```

风险：

- 每笔 transaction 被广播到多个组件
- 每个 subscriber 可能 copy、sample、print、write file
- 某些 test 不需要所有 subscriber 仍然执行

建议：

- 增加 enable 开关
- 使用 dispatcher 过滤 transaction
- 对不同消费者使用不同粒度 transaction
- coverage 不一定需要完整 transaction
- logger 默认不接收高频 transaction

示例：

```systemverilog
function void write(my_trans tr);
  if (enable_sb) begin
    sb_export.write(tr);
  end

  if (enable_cov && tr.need_cov_sample()) begin
    cov_export.write(tr);
  end

  if (enable_trace && tr.is_interesting()) begin
    trace_export.write(tr);
  end
endfunction
```

---

### 6.7 高频路径中调用 config_db / resource_db

重点搜索：

```bash
rg "config_db|resource_db"
```

低效写法：

```systemverilog
forever begin
  @(posedge vif.clk);

  void'(uvm_config_db#(int)::get(this, "", "enable_check", enable_check));

  if (enable_check) begin
    do_check();
  end
end
```

风险：

- config_db 查找有层次路径匹配开销
- per-cycle 查询会严重浪费
- 开关状态不应每拍从 database 读取

建议：

```systemverilog
function void build_phase(uvm_phase phase);
  super.build_phase(phase);

  if (!uvm_config_db#(int)::get(this, "", "enable_check", enable_check)) begin
    enable_check = 1;
  end
endfunction
```

原则：

- config_db 用于 build/connect/start_of_simulation
- run_phase 热路径只访问本地变量
- runtime 可变配置应使用专门 lightweight 变量或 event 机制

---

### 6.8 复杂 randomize 被频繁调用

重点搜索：

```bash
rg "randomize\\(|constraint|dist|inside|foreach"
```

重点关注：

```systemverilog
repeat (1000000) begin
  tr = my_trans::type_id::create("tr");
  assert(tr.randomize() with {
    addr inside {[base:base+size]};
    foreach (data[i]) data[i] dist {
      8'h00 := 1,
      8'hff := 1,
      [8'h01:8'hfe] := 10
    };
    !(addr inside {forbidden_addr_q});
  });
end
```

风险：

- constraint solver 开销大
- 动态数组 / queue membership 约束昂贵
- `dist`、`inside`、`foreach`、复杂 implication 叠加会很慢
- 大量 transaction 生成时成为瓶颈

建议：

- 简单字段使用 procedural generation
- 复杂约束预生成 pool
- 减少动态数组约束
- 避免在 tight loop 中重复求解不变约束
- 将常量计算提前
- 对大规模 random traffic 使用 lightweight generator

示例：

```systemverilog
assert(tr.randomize() with {
  id inside {[0:15]};
  opcode inside {OP_LOAD, OP_STORE, OP_AMO};
});

tr.addr = gen_addr_fast(region_cfg);
tr.data = gen_data_pattern(pattern_mode);
```

---

### 6.9 滥用 UVM field automation macro

重点搜索：

```bash
rg "uvm_field_|UVM_ALL_ON|UVM_DEFAULT|UVM_NOCOMPARE|UVM_NOPRINT"
```

典型写法：

```systemverilog
`uvm_object_utils_begin(my_trans)
  `uvm_field_int(addr, UVM_ALL_ON)
  `uvm_field_int(data, UVM_ALL_ON)
  `uvm_field_queue_int(payload, UVM_ALL_ON)
  `uvm_field_string(debug_str, UVM_ALL_ON)
`uvm_object_utils_end
```

风险：

- `UVM_ALL_ON` 打开 copy、compare、print、pack、record 等行为
- 对大数组和 queue 很慢
- 默认 compare / print 可能递归处理大量字段

建议：

```systemverilog
`uvm_object_utils(my_trans)
```

并手写必要方法：

```systemverilog
function void do_copy(uvm_object rhs);
  my_trans r;

  if (!$cast(r, rhs)) begin
    `uvm_fatal("COPY", "cast failed")
  end

  addr = r.addr;
  data = r.data;
  id   = r.id;
endfunction
```

或者关闭不必要字段：

```systemverilog
`uvm_field_int(addr, UVM_DEFAULT)
`uvm_field_int(data, UVM_NOCOMPARE | UVM_NOPRINT)
```

原则：

- 大型 transaction 不建议无脑 `UVM_ALL_ON`
- 高频 compare 路径手写关键字段比较
- print 只在 debug 模式启用

---

### 6.10 checker 每拍扫描全局状态

重点搜索：

```bash
rg "assert\\(|check_|foreach|for .*size\\("
```

低效写法：

```systemverilog
always @(posedge clk) begin
  assert(check_whole_system_state());
end
```

其中：

```systemverilog
function bit check_whole_system_state();
  foreach (rob[i]) begin
    foreach (lsq[j]) begin
      foreach (iq[k]) begin
        if (some_relation(rob[i], lsq[j], iq[k])) begin
          return 0;
        end
      end
    end
  end

  return 1;
endfunction
```

风险：

- 每拍 O(N²) / O(N³) 扫描
- CPU 微结构队列较多时特别慢
- checker 越写越复杂，仿真后期急剧变慢

建议：

- 改成事件触发检查
- 改成增量检查
- 对状态建立索引
- 每 N 个 cycle 低频检查一次
- smoke test 开轻量检查，nightly 开全量检查

示例：

```systemverilog
if (alloc_event) begin
  check_alloc_entry(alloc_idx);
end

if (commit_event) begin
  check_commit_order(commit_idx);
end

if (flush_event) begin
  check_flush_state();
end
```

---

### 6.11 文件 IO 和 trace 过重

重点搜索：

```bash
rg "\\$fwrite|\\$fdisplay|\\$fflush|\\$swrite|\\$sformat"
```

风险写法：

```systemverilog
forever begin
  @(posedge clk);
  $fwrite(fd, "%0t addr=%0h data=%0h\n", $time, addr, data);
end
```

风险：

- 每拍 IO 很慢
- `$fflush` 更慢
- 字符串格式化和文件写入叠加
- NFS / 网络盘上尤其明显

建议：

- trace 默认关闭
- 只在 valid transaction 时写
- 支持 trace filter
- 批量缓存后 flush
- 避免每行 `$fflush`
- 使用二进制或简化格式记录高频 trace

---

### 6.12 过多 fork/join_none 或 zero-time loop

重点搜索：

```bash
rg "fork|join_none|forever|while"
```

严重问题：

```systemverilog
fork
  forever begin
    if (condition) begin
      do_something();
    end
  end
join_none
```

风险：

- 没有等待事件，形成 zero-time busy loop
- 可能直接卡死仿真
- 过多长期 process 增加调度开销

建议：

```systemverilog
forever begin
  @(posedge clk);

  if (condition) begin
    do_something();
  end
end
```

对于多个 entry：

```systemverilog
forever begin
  @(posedge clk);

  foreach (entries[i]) begin
    if (entries[i].valid) begin
      check_entry(i);
    end
  end
end
```

原则：

- 每个 forever loop 必须有明确等待条件
- 不要为每个 transaction 创建长期 process
- 不要为每个 ID / entry 无脑 fork 独立线程
- 用集中式 event loop 替代大量小线程

---

### 6.13 每笔 transaction raise/drop objection

重点搜索：

```bash
rg "raise_objection|drop_objection"
```

低效写法：

```systemverilog
repeat (100000) begin
  phase.raise_objection(this);

  start_item(req);
  finish_item(req);

  phase.drop_objection(this);
end
```

建议：

```systemverilog
phase.raise_objection(this);

repeat (100000) begin
  start_item(req);
  finish_item(req);
end

phase.drop_objection(this);
```

原则：

- objection 管理 test / sequence 生命周期
- 不应管理每一笔 transaction
- 对大量 traffic generation 尤其需要避免频繁 objection 操作

---

### 6.14 reference model 过重

重点检查：

- ISA model 是否每条指令全状态比较
- memory model 是否每笔访存全量扫描
- cache model 是否每次访问全 cache 搜索
- MMU/TLB model 是否重复翻译
- scoreboard 是否每笔 transaction 都触发 full architectural compare

低效模式：

```systemverilog
function void write_commit(commit_tr tr);
  ref_model.step_full_system(tr);
  ref_model.compare_full_arch_state();
endfunction
```

建议：

```systemverilog
if (enable_isa_check) begin
  ref_model.step_instruction(tr);
end

if (enable_mem_order_check && tr.is_load_store()) begin
  mem_model.check_order(tr);
end

if (enable_full_state_compare && tr.is_sync_point()) begin
  ref_model.compare_arch_state();
end
```

原则：

- reference model 分层
- full-check 不应默认用于所有 test
- long random stress 可使用 lightweight check
- full architectural compare 可放在 checkpoint / sync point

---

## 7. 推荐搜索命令

在代码库根目录下，可以先执行以下命令辅助定位热点：

```bash
rg "uvm_info|uvm_warning|uvm_error|\\$sformatf|convert2string|sprint|print\\("
rg "type_id::create|new\\("
rg "copy\\(|clone\\(|compare\\(|pack\\(|unpack\\("
rg "config_db|resource_db"
rg "covergroup|coverpoint|cross|sample\\("
rg "analysis_port|analysis_export|uvm_analysis_imp|uvm_tlm_analysis_fifo"
rg "randomize\\(|constraint|dist|inside"
rg "foreach|find_index|delete\\(|pop_front|push_back"
rg "\\$fwrite|\\$fdisplay|\\$fflush|\\$swrite"
rg "fork|join_none|forever|while"
rg "raise_objection|drop_objection"
```

注意：

- 搜索结果不能直接等同于问题
- 必须结合上下文判断是否在高频路径
- build_phase 中的 `config_db::get` 通常没问题
- end_of_test 中的 print 通常没问题
- 关键是判断调用频率和数据规模

---

## 8. 性能问题严重程度分级

### P0：严重性能问题

满足以下任意条件：

- zero-time loop
- 每拍大量 object create
- 每拍无条件 coverage sample
- scoreboard O(N²) 且 outstanding 很大
- 每拍文件 IO
- 每笔 transaction 大量 `copy/clone/print`
- long random 后内存持续增长
- 打开某组件后仿真变慢数倍

### P1：高优先级优化

- 高频路径中 `$sformatf`
- analysis fanout 过多
- coverage cross 过大
- reference model 全量比较过频繁
- queue 中间 delete 频繁
- complex randomize 在大循环中调用

### P2：中优先级优化

- UVM field macro 使用过重
- log 开关粒度不足
- checker 可进一步事件化
- configuration 读取不够集中
- trace filter 不够细

### P3：低优先级优化

- build/connect 阶段小规模低效代码
- end_of_test summary 较重
- 非默认 debug 功能较慢
- 只影响少数 debug test 的问题

---

## 9. 输出报告格式

最终报告建议使用以下格式。

### 9.1 Executive Summary

```markdown
# UVM 验证环境效率分析总结

## 总体判断

当前验证环境的主要性能风险集中在：

1. monitor / scoreboard 高频路径存在较多 object create 和 copy
2. coverage collector 存在过度 sample 和大型 cross
3. scoreboard 使用线性 queue 匹配，outstanding 增大后复杂度较高
4. log / trace 在部分高频路径缺少开关保护
5. reference model 默认执行粒度偏重

## 优化优先级

| 优先级 | 问题 | 预期收益 | 修改风险 |
|---|---|---:|---:|
| P0 | scoreboard queue 线性搜索 | 高 | 中 |
| P0 | monitor 每拍创建 transaction | 高 | 低 |
| P1 | coverage 每拍 sample | 中到高 | 低 |
| P1 | log 无保护 sformatf | 中 | 低 |
| P2 | UVM field macro 过重 | 中 | 中 |
```

### 9.2 环境结构理解

```markdown
# 验证环境结构

## 顶层结构

- `tb_top.sv`
- `cpu_base_test.sv`
- `cpu_env.sv`
- `ifu_agent`
- `lsu_agent`
- `scoreboard`
- `coverage_collector`
- `ref_model`

## 主要 transaction 流

```text
lsu_monitor.ap
  -> lsu_scoreboard.exp_export
  -> mem_cov_collector.analysis_export
  -> transaction_logger.analysis_export
```

## 高频组件

| 组件 | 高频原因 |
|---|---|
| lsu_monitor | 每个 LSU request / response 都会调用 |
| cpu_scoreboard | 每条 commit / memory event 都会调用 |
| coverage_collector | 每笔 transaction sample |
| ref_model | 每条指令 step |
```

### 9.3 单个问题报告模板

每个问题应使用固定格式。

```markdown
## Issue P0-001：scoreboard 使用线性 queue 搜索导致 O(N²)

### 位置

- 文件：`env/scoreboard/cpu_scoreboard.sv`
- 函数：`write_actual()`
- 代码区域：line 120 - line 168

### 代码现象

当前 scoreboard 对每个 actual transaction 都遍历 expected queue：

```systemverilog
foreach (expected_q[i]) begin
  if (match(expected_q[i], actual)) begin
    compare(expected_q[i], actual);
    expected_q.delete(i);
    return;
  end
end
```

### 性能影响

当 outstanding transaction 数量较大时，每笔 actual 都需要线性搜索 expected queue。
如果 queue size 达到数百或数千，长随机 test 中该函数会成为主要性能瓶颈。

### 优化建议

使用 associative array 建立 key-based index：

```systemverilog
my_trans expected_by_key[sb_key_t][$];
```

匹配时通过 key 直接查找，避免全队列扫描。

### 修改风险

中等。

需要确认 key 的唯一性规则：

- 是否允许相同 ID 多 outstanding
- 是否需要 addr / opcode / txn_type 共同作为 key
- 是否存在乱序返回
- 是否需要支持 retry / replay / cancel

### 验证建议

1. 使用原 scoreboard 和新 scoreboard 跑同一批 directed tests
2. 打开 compare debug，对比匹配顺序
3. 增加相同 key 多 outstanding 测试
4. 增加 out-of-order response 测试
5. 对 mismatch case 注入错误，确认仍能报错
```

---

## 10. 优化建议原则

### 10.1 不要只给笼统建议

错误示例：

```markdown
建议优化 scoreboard。
```

正确示例：

```markdown
scoreboard 的 `write_actual()` 函数对 `expected_q` 进行线性搜索。
该函数位于 actual transaction 高频路径中。
当 outstanding 数量增长时，复杂度从 O(1) 退化为 O(N)，整个 test 可能变成 O(N²)。
建议使用 associative array，以 `{id, addr, txn_type}` 作为 key，将 expected transaction 存入 `expected_by_key[key][$]`。
```

---

### 10.2 每条建议必须包含证据

每个性能问题应尽量包含：

- 文件名
- 类名
- 函数名
- 调用路径
- 代码片段
- 为什么这是高频路径
- 为什么该写法慢
- 建议修改方式
- 修改风险
- 验证方法

---

### 10.3 优先做低风险高收益优化

优先级建议：

1. 关闭无用 log / trace
2. 避免无效周期创建 transaction
3. coverage 只在有效事件 sample
4. config_db get 移出 run_phase
5. scoreboard queue 改 key-based lookup
6. reference model 分层 enable
7. transaction copy/compare/print 手写轻量版本
8. complex randomize 拆分或预生成
9. 大型 checker 改事件触发
10. 大规模架构重构

---

## 11. 回归验证建议

所有性能优化都必须证明没有改变验证语义。

推荐验证方式：

### 11.1 Golden regression 对比

选择一组代表性 test：

- smoke test
- directed test
- random test
- long random stress
- error injection test
- coverage test
- scoreboard mismatch test

比较：

- pass/fail 是否一致
- UVM error/warning 数量是否一致
- 关键 scoreboard compare 次数是否一致
- coverage sample 数是否合理一致
- transaction count 是否一致
- end-of-test summary 是否一致

### 11.2 性能指标对比

优化前后记录：

| 指标 | 优化前 | 优化后 | 改善 |
|---|---:|---:|---:|
| wall time | | | |
| CPU time | | | |
| peak memory | | | |
| transaction count | | | |
| coverage sample count | | | |
| scoreboard queue max depth | | | |
| log line count | | | |

### 11.3 防止功能退化

优化后应特别检查：

- monitor 是否漏采 transaction
- scoreboard 是否漏报 mismatch
- coverage 是否少采关键事件
- ref model 是否跳过必要检查
- logger / trace 是否仍可 debug
- out-of-order 场景是否仍能正确匹配
- timeout / drain / end-of-test 行为是否一致

---

## 12. 推荐最终交付物

使用本 Skill 后，建议输出以下文件或章节：

```text
uvm_env_perf_review.md
 ├── 1. Executive Summary
 ├── 2. UVM Environment Architecture
 ├── 3. Major Transaction Flow
 ├── 4. Performance Hotspot Candidates
 ├── 5. Detailed Issues
 │   ├── P0 Issues
 │   ├── P1 Issues
 │   ├── P2 Issues
 │   └── P3 Issues
 ├── 6. Recommended Refactoring
 ├── 7. Risk Assessment
 ├── 8. Regression Validation Plan
 └── 9. Appendix: Search Results and Code Evidence
```

---

## 13. 分析时的专家判断准则

### 13.1 判断是否真的是性能问题

不是所有看起来“重”的代码都是问题。

需要综合判断：

- 是否在高频路径
- 调用次数是否大
- object 是否大
- queue 是否长
- coverage bin 是否多
- 是否默认开启
- 是否影响所有 test
- 是否只在 debug 模式启用
- 是否已有仿真 profile 证据

### 13.2 不要误伤可读性

有些 UVM 写法虽然不是最高效，但如果只在低频路径中使用，可以保留。

例如：

- build_phase 中使用 config_db
- end_of_test 中打印 summary
- debug test 中使用 full transaction print
- 小规模 queue 中使用线性搜索
- directed test 中少量 randomize

优化不应为了微小收益牺牲可维护性。

### 13.3 区分 testbench 性能与 DUT 性能

分析时需要区分：

- DUT RTL 本身仿真慢
- gate-level simulation 慢
- waveform dump 慢
- SVA assertion 慢
- UVM testbench 慢
- coverage database 慢
- DPI-C reference model 慢
- license / farm / filesystem 慢

本 Skill 重点分析 UVM/SystemVerilog testbench 编码和架构导致的性能问题，但报告中可以指出非 UVM 因素。

---

## 14. CPU / SoC UVM 环境中特别关注的性能瓶颈

### 14.1 Commit / Retire Scoreboard

CPU 验证环境中，commit / retire scoreboard 往往是最高频组件之一。

重点检查：

- 是否每条指令都 full architectural compare
- 是否保存完整 instruction trace object
- 是否对 ROB / commit queue 做线性扫描
- 是否每条 commit 都打印 trace
- 是否每条 commit 都调用重型 reference model

建议：

- 按 instruction class 分层检查
- full architectural compare 放在 checkpoint 或 sync point
- commit trace 默认关闭或压缩记录
- 使用 retire index / sequence number / pc 作为 key 建索引

### 14.2 LSU Memory Order Checker

LSU / memory ordering checker 容易出现 O(N²) 或 O(N³) 扫描。

重点检查：

- load/store queue shadow model 是否每笔访存全量扫描
- memory dependency 检查是否有 key-based index
- store buffer / invalidate / snoop 事件是否触发全局扫描
- 是否每次 load 都扫描所有 older store

建议：

- 使用 address range index
- 使用 page / cache-line 粒度索引
- 对 ordering violation 做事件触发检查
- 对全量一致性检查降频或放到 checkpoint

### 14.3 Cache Coherence Monitor

多核 / coherent subsystem 环境中，coherence monitor 可能非常重。

重点检查：

- 是否每笔 snoop 都扫描所有 core/cache line 状态
- 是否对每个 transaction 都 clone 完整 cache-line object
- 是否对所有 channel 做无条件 cross coverage
- 是否存在大量 per-line forked process

建议：

- 按 cache line address 建 associative array
- 按 transaction type 做 filter
- 使用 per-line state summary，而不是完整 history object
- 对 cross coverage 按 feature 拆分

### 14.4 ROB / IQ / LSQ Shadow Model

乱序 CPU 验证常会维护 shadow ROB / IQ / LSQ。

重点检查：

- 是否每拍扫描所有 entry
- 是否每个 entry 一个 forever process
- 是否每次分配 / 唤醒 / 发射都做全局一致性检查
- 是否大量使用 queue 中间 delete

建议：

- 使用 valid entry list
- 使用 head/tail pointer 或 free-list
- 对 wakeup / issue / commit 做事件触发局部检查
- 全量 invariant check 可降频或只在 debug 模式开启

---

## 15. 常见高收益优化清单

| 类型 | 优化动作 | 收益 | 风险 |
|---|---|---:|---:|
| Log | `$sformatf` 增加 `uvm_report_enabled` 保护 | 中 | 低 |
| Monitor | valid 后再 create transaction | 高 | 低 |
| Scoreboard | queue 线性搜索改 associative array | 高 | 中 |
| Coverage | 每拍 sample 改事件 sample | 高 | 低 |
| Coverage | 拆分大型 cross | 中 | 中 |
| TLM | analysis fanout 增加 enable/filter | 中 | 低 |
| Config | run_phase 中 config_db get 移到 build_phase | 中 | 低 |
| Random | complex randomize 改 procedural generation | 中 | 中 |
| Object | 减少 clone/copy/print | 中 | 中 |
| Field Macro | 避免 `UVM_ALL_ON` | 中 | 中 |
| Ref Model | full compare 改 checkpoint compare | 高 | 中 |
| Checker | 全局扫描改事件触发 | 高 | 中 |
| IO | trace 默认关闭或批量写 | 中 | 低 |
| Process | 减少 fork/join_none 小线程 | 中 | 中 |

---

## 16. 示例最终结论

```markdown
# 结论

当前 UVM 验证环境的主要性能风险不是单点问题，而是多个高频路径中的重操作叠加：

1. monitor 在无效周期也创建 transaction object
2. scoreboard 对 expected queue 进行线性搜索
3. coverage collector 对每拍或每笔 transaction 进行大型 cross sample
4. 多个 analysis subscriber 接收完整 transaction
5. debug log 中存在未保护的 `$sformatf()` 和 `convert2string()`
6. reference model 默认执行 full architectural compare

建议优先处理 P0/P1 问题，尤其是 monitor object create、scoreboard matching、coverage sample 条件和 log 保护。这些修改通常能在较低功能风险下显著改善仿真效率。

优化后必须通过 golden regression 对比，确认 transaction count、scoreboard compare count、coverage sample 行为和 error reporting 行为没有异常变化。
```

---

## 17. 使用本 Skill 的注意事项

- 不要在没有代码证据的情况下猜测瓶颈
- 不要只给抽象原则，必须给出具体文件、类、函数、代码模式
- 不要默认关闭 checker / scoreboard / coverage 来换性能，必须说明验证风险
- 不要把 debug-only 慢代码误判为默认性能问题
- 对于可能改变验证语义的优化，必须给出验证计划
- 对 CPU / SoC 验证环境，特别关注 long random、multi-outstanding、out-of-order、retry/replay、flush、exception、interrupt 等场景下 scoreboard 和 reference model 的复杂度

---

## 18. 一句话原则

UVM 环境效率优化的核心不是“少写 UVM”，而是：

> 把重操作从 per-cycle / per-transaction 热路径中移出去；
> 用事件触发代替无条件执行；
> 用 key-based lookup 代替线性扫描；
> 用轻量数据结构代替完整 object 复制；
> 用可配置分层 check 代替默认 full-check。
