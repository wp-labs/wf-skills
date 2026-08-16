---
name: wf-perf-bench-verification
description: Warp Fusion nexmark 性能基准的测量纪律与输出正确性验证方法论。用于跑 bench 得出可信 EPS/RSS 数字（计时口径、A/B 交错、RSS 相位配对、首跑剔除）和验证规则引擎输出正确性（确定性 ground truth 模拟器、逐 alert 对拍、语义考古、非确定性残差定性）。覆盖 metrics.ndjson 计数提取、bench.sh 用法、q5 类窗口规则的对拍陷阱。
---

# AI Agent 指南：wfusion 性能基准测量与正确性验证

> 供 AI agent 在 `wf-examples/performance/nexmark_pk`（及同类 bench）上得出**可信的**
> 性能数字并**验证输出正确性**时遵循。核心原则：数字要经得起"换一天重跑还在不在"
> 的检验；验证要以确定性 ground truth 为基准，不以"看起来对"为基准。
>
> 配套：bench 操作细节见 `wf-examples/performance/nexmark_pk/README.md`，
> 引擎权威设计见 `wp-reactor/docs/design/`。

## 何时用本指南

- 跑 nexmark_pk（或任意 cont 模式）bench 并对外引用 EPS/RSS 数字
- 验证规则引擎输出条数是否正确（尤其带窗口/聚合的规则）
- A/B 对比两个 commit / 两个配置的性能
- 排查"引擎输出与预期差百分之几"类问题

## 第一部分：测量纪律（数字可信的前提）

### 1. 计时口径 = append_total

- EPS 分母用 metrics.ndjson 的 `append_total`（counter，**跨 1s 区间求和**）追平
  TOTAL 行，不是 ingress 预读游标（预读会超前于消费，数字虚高——曾因此全量
  作废过历史 EPS）。
- 提取脚本：`nexmark_pk/scripts/extract_emitted.py data/metrics.ndjson`
  （counter 求和、gauge 取峰值；`val = int(float(...))` 处理浮点行）。
- **counter 与 gauge 不可混用**：counter 跨区间求和，gauge（memory_bytes/rows）
  取单样本峰值。

### 2. A/B 对比三条件（缺一不可）

1. **不限速**：`RATE=10000000`（限速会封顶，测出的是注入速率不是引擎能力）；
2. **同时段交错**：ABAB 或 ABBA 交错跑，不串行分时段；
3. **RSS 相位配对**：bench 机呈双峰相位（EPS 与 RSS_peak 强相关，±8%），
   高低相位不可直接比。同相位配对后才可下结论。

⚠️ 批式向量化后**高 EPS 与低 RSS 解耦**——相位仍在（机器层面），但旧"高相=高
RSS"的绑定失效：仍按 RSS 配对，但不能用 RSS 反推相位。

### 3. 必须剔除的跑批

- **stash 重建后的第一跑系统性偏低**（曾三次复现 ~3.75M vs 正常 4.7M+），
  重建后先跑一轮预热再计数据；
- 正确性计数器非零的跑批作废：`serialize_failed` / `dropped_late` /
  `cursor_gap` / `memory_evicted` 必须为 0（`time_evicted` 有值属正常窗口关闭）。

### 4. 沙箱环境适配（macOS）

- `ps` 执行被拒 → RSS 采样用 macOS `footprint <pid>`（bench.sh 已内置回退）；
- 勿用 `/usr/bin/time -l` 包装 daemon（搞乱 PID/端口管理）；
- 无 `timeout` 命令 → 后台启动 + sleep + kill 模式；
- shell 会话 cwd 每条命令重置 → 统一用 `--manifest-path` / `git -C` / 绝对路径，
  不依赖 `cd &&` 前缀。

### 5. 结果合理性自检（防相位混杂）

同一轮内自查逻辑一致性：**q2 语义是 q1 的超集，q2 不可能比 q1 快**。出现
q2 > q1 即相位混杂的直接证据（曾实测 q1 4.34M 撞低相、q2 5.13M 撞高相）。
单轮单 query 的数字只能算"探摸"，不可对外。

## 第二部分：正确性验证方法论

### 1. 前提：确定性数据

wfgen 确定性生成（`wfgen gen-nexmark <count>`，默认 seed=1，phase-major 顺序）。
同一 count 重新生成的 frames **字节一致**——这意味着可以用 Python 流式重放
JSONL 计算 ground truth（`verify_ground_truth.py` 模式）。

### 2. 语义考古：从代码确认语义，不要假设 DSL 语义

写模拟器前**逐层读引擎代码**确认实际语义。经验教训：DSL 字面语义与实现可能
存在微妙偏差，且这些偏差就是偏差来源本身。关键位置：
`wf-engine/src/match_engine/match_engine/mod.rs`（advance/fire/reset/expiry）、
`wf-runtime/src/engine_task/rule_task.rs`（扫描时机）。
已冻结的语义档案：`wp-reactor/docs/design/match-expiry-semantics.md`。

### 3. 逐级收窄的对拍策略（差 2% → 0.0036% → 0）

发现全量偏差后的收窄路径，每级都保持可复现：

1. **全量模拟**（30M）：定位偏差量级（如 +2%）；
2. **中探针**（~28k 事件 slice）：复现偏差比例（+13.5%），获得可逐条对拍的
   alert 列表；
3. **单 key trace**（单 auction 逐事件）：找**第一个分歧事件**，读代码核对
   该事件的引擎处理路径；
4. 根因通常是**一个精确的语义细节**（见下"已踩过的坑"）。

### 4. 已踩过的坑（模拟器陷阱清单）

- **手推小例会引入自己的错误**：手推时隐含了正确语义，而模拟代码用错语义——
  两者相互"确认"会掩盖根因。手推和模拟要分别与引擎对拍。
- **fire/reset 语义**：引擎 plain `on event` fire 后 `instance.reset(plan,
  fire_time)`——实例保留、created_at=**fire 事件时间**（非下一条事件时间）、
  聚合清零。模拟器 fire 时销毁实例会少算。
- **pending_expiry 每 key 单条目去重**：fire/reset 后的 push 被去重丢弃，实例
  过期由旧 heap 条目驱动（pop 时才 re-read 修正）→ 乱序流下实例系统性多活
  ~2%。模拟器无条件 push 新条目会少算 3.4 万条。
- **JSONL 键序两种格式**：wfgen 紧凑（`"k":v`）vs Python json.dumps 带空格
  （`"k": v`）→ 正则一律 `:\\s*`。
- **entity_id 类型**：sink 输出里是字符串，模拟侧是 int → int() 后再比。
- **snapshot join miss 不丢事件**：仅跳过 enrich（Anti join 才过滤）。

### 5. 残差定性：找到非确定性来源即可结案

确定性模拟器原理上无法复现引擎的非确定性成分。已知来源：
`scan_timeouts` 周期 tick 用 `watermark + 墙钟 elapsed` 推进过期（运行时长
相关）+ 多 shard 各 machine watermark 子序列效应。**量级吻合（如 0.0036%）
+ 中探针级 100% 精确吻合 = 可结案**，不必追求全量逐条相等。

### 6. 假设排除的纪律

每个候选假设必须**模拟或实验证伪后才排除**，并留档（工具脚本入库）。曾经历
六轮假设排除（全局水位线/并行度/排序流/accu/span 单位/数据陈旧）才命中真因。
工具链在 `nexmark_pk/scripts/`：`verify_ground_truth.py`（全量模拟）、
`q5_diff_v2.py`（逐 alert 对拍）、`q5_trace_auc.py`（单 key trace）、
`extract_emitted.py`（metrics 提取）。

## 第三部分：性能排查教训（关联设计文档）

- **crossbeam-epoch 延迟析构陷阱**：skiplist `remove` 只 unlink，value 析构需
  epoch+2；静息期无人 pin → 已删除数据永久驻留（RSS 回归 13.9GB 的真凶）。
  归因工具链：`std::alloc::System` + `MallocStackLogging=1` +
  `malloc_history <pid> -callTree`。
- **profile 归因会虚高**：float 格式化曾显示 ~6%，实际每事件仅 3 次调用——
  归因要有调用频次佐证。
- **零效应方向先查档案再动手**：`hot-path-vectorization-design.md` §5（recv_many
  批排空 / ALERT_BATCH_SIZE 调大 / parse 合并消息 / float 快路径均实测零效应）。
- **列批/向量化改造必须配行级等价测试**（总数一致不够，列错位要靠行级对比抓）。
- **优化通用性优先**：按"是否让生产工作负载普遍受益"排序改动，拒绝 benchmark
  特化；参照系（Flink/VVR）不是优化靶心。

## 检查清单（对外引用数字前）

- [ ] append_total 口径 + counter 求和
- [ ] 不限速、同时段交错（如为 A/B）
- [ ] RSS 相位配对（如为 A/B）；同轮逻辑自检（q2≤q1 等）
- [ ] 非 stash 重建首跑
- [ ] 正确性计数器全零 + appended == TOTAL
- [ ] 数字与既有基线方向一致；不一致能解释
