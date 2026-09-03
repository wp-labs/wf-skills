---
name: wf-integration-authoring
description: Warp Fusion 产品集成（把引擎作为检测子系统接入自有系统）。用于搭建/修改/排查一个完整检测任务工程——按固定五步次序装配：①设置数据来源 → ②定义输入窗口 → ③定义告警输出窗口 → ④设置告警输出（sink 路由）→ ⑤编写计算规则；以及目录接线、文件格式陷阱与端到端验证（wfl lint/test + wfusion batch，产物 alerts.ndjson 断言）。
---

# AI Agent 指南：WarpFusion 产品集成（5 步搭建检测任务）

> 供 AI agent 在把 WarpFusion 作为检测/分析子系统集成进自有产品、为某个业务
> 新建检测任务、或仿照最小工程写配置/排查集成问题时遵循。核心原则：
> **数据流先于规则**——先让数据进得来、出得去，再写规则；规则只是把前四步
> 串起来的那一层。
>
> 配套与分工：完整叙述见 warp-fusion `docs/useage/integration.md`；最小可跑
> 工程 `examples/rules/hello_detection`（含一键 `run.sh`）。单点细节交给相邻
> 技能：`wf-config-authoring`（wfusion.toml/source 参数）、`wf-wfs-authoring`
> （window schema）、`wf-wfl-authoring`（规则语法）、`wf-test-pipeline-debugging`
> （wfl test 管线排障）。

## 何时用本指南

- 把引擎接入自有系统：先选来源，再定窗口，再接输出，最后写规则
- 新建一个检测任务工程（目录骨架、接线、跑通）
- 给用户提供「照抄可跑」的最小集成示例
- 排查集成工程跑不起来（配置加载失败 / 没产出告警 / 产出不对）

## 固定五步次序（不要倒过来写规则）

```
① 设置数据来源  →  ② 定义输入窗口  →  ③ 定义告警输出窗口  →  ④ 设置告警输出  →  ⑤ 编写计算规则
```

各步职责与「下一步依赖什么」：

| 步 | 交付物 | 关键点 |
|---|---|---|
| ① 数据来源 | `wfusion.toml [[sources]]` 或 `topology/sources/*.toml` | 决定事件怎么进引擎；定 `stream_tag`（固定）或 `stream_tag_field`（逐行分发） |
| ② 输入窗口 | `schemas/*.wfs` 的输入 window | 每个 stream_tag 对应一个输入窗；声明字段类型/时间字段/保留时长 |
| ③ 告警输出窗口 | 同文件里的输出 window | `over = 0`；字段 = 一条告警的结构，由规则的 `yield` 逐字段填充 |
| ④ 告警输出 | `topology/sinks/business.d/*.toml` + connector | sink group `windows` 命中第 ③ 步输出窗名，接到消费通道（file/kafka/自定义） |
| ⑤ 计算规则 | `rules/*.wfl` | 绑定 ②→聚合/匹配→评分→`yield` ③（由 ④ 送出）；可带内联 `test` |

判断写到了哪一步：第 ⑤ 步之前，engine 应当能 bootstrap 且数据能落窗
（batch 跑完有正常关闭日志）；第 ⑤ 步之后才有告警产出。

## 工程布局（照抄骨架）

```text
your_detection_project/
├── wfusion.toml               # 引擎配置（sources/sinks/windows 目录 + runtime）
├── windows.toml               # 窗口资源/时间策略（条目可省，文件被引用、不能缺）
├── schemas/*.wfs              # ②输入窗口 + ③告警输出窗口定义
├── rules/*.wfl                # ⑤计算规则（可多文件，可有 _global.wfl preset）
├── topology/sources/          # ①数据来源声明（或写在 wfusion.toml [[sources]]）
└── topology/sinks/            # ④告警输出路由
    ├── business.d/            #   业务告警 sink group（windows = [输出窗名]）
    ├── connectors/sink.d/     #   connector 定义（[[connectors]] 数组形式）
    └── infra.d/               #   兜底 windows=["*"] / 错误通道（可选约定）
```

真实最小工程 = warp-fusion `examples/rules/hello_detection`（3 行 failed 登录 →
`auth_events` → `match<sip:1m>` 计数≥3 → `mini_alerts` 输出窗 →
`alerts.ndjson` 恰好 1 条 `brute_login_mini`）。

## 每步要点

### 第 1 步：数据来源

- 推荐 **TCP + ndjson**：发送方任何语言逐行发 JSON（行内可带 tag 字段）；
  与 warp-parse 联动用 `arrow_framed`（帧级 tag 路由）；回放历史数据用
  **file source**（`mode="batch"`）。
- 两种声明位置等价：daemon + TCP 写在 `wfusion.toml` 的 `[[sources]]`；
  batch + file 回放写在 `topology/sources/*.toml`（顶层键
  `connect/enable/key/path/stream_tag/data_format`，无 `type` 字段）。
- ndjson 行内 `_stream` 字段与 source 的固定 `stream_tag` 要一致
  （示例行：`{"_stream":"auth",...,"event_time":"..."}`）。

### 第 2 步：输入窗口

```wfs
// stream_tag 与第 1 步来源对齐；time = 事件时间字段（每条事件必须携带）
// over = 窗口保留时长
window auth_events {
    stream_tag = "auth"
    time = event_time
    over = 1h
    fields {
        sip: ip
        user: chars
        action: chars
        event_time: time
    }
}
```

来源-窗口靠 `stream_tag` 对齐；未知 tag 进内置 miss 诊断（不崩引擎）。

### 第 3 步：告警输出窗口

`yield` 的目标也是一个 window：`over = 0` 的输出窗，字段 = 告警结构。

```wfs
window mini_alerts {
    over = 0
    fields { sip: ip; alert_type: chars; detail: chars; }
}
```

输出窗不存事件、不做窗口聚合，只是「告警管道」的类型声明。

### 第 4 步：告警输出路由

- `business.d/*.toml`：`windows = ["mini_alerts"]` 命中第 3 步输出窗名；
  每条 sink 的 `connect = "file_json"` 引用 connector id。
- `connectors/sink.d/*.toml`：connector 定义**必须是 `[[connectors]]` 数组
  形式**（`id`/`type`/`allow_override` + `[connectors.params]`）。写错成
  `[connector] name/sink_type` 会在 bootstrap 报 "load sink config"。
- `infra.d/default.toml`（`windows = ["*"]` 兜底）与 `error.toml`（错误通道）
  是约定目录，去掉不影响跑通。
- 业务输出行 = 规则 `yield` 的业务字段 + `__wfu_*` 元字段
  （`__wfu_rule_name` / `__wfu_score` / `__wfu_entity_id` / `__wfu_fired_at`…）。

### 第 5 步：计算规则

```wfl
// events = 绑定 ② 输入窗（别名 a）；match<sip:1m> = 按 sip 开 1 分钟匹配窗
// entity → __wfu_entity_id；yield → ③ 输出窗；limits 为护栏（可省）
rule brute_login_mini {
    events { a : auth_events && action == "failed" }
    match<sip:1m> {
        on event { a | count >= 3; }
    } -> score(60.0)
    entity(ip, a.sip)
    yield mini_alerts (
        sip = a.sip,
        alert_type = "brute_login_mini",
        detail = "3 failed logins in 1m"
    )

    limits {
        max_memory = "16MB";
        max_instances = 1000;
        on_exceed = throttle;
    }
}
```

- `limits{}` 可省（防失控护栏，示例故意展示）；
- 内联测试的 `row(a, ...)` 别名 = `events { a : ... }` 的绑定别名；
  `expect { hits == 1; }` 的 `hits` = 组/实例触发次数（勿按每事件一条断言）。

## 端到端验证（命令用真实子命令）

```bash
# 规则静态检查 + 内联测试（注意：wfl 没有 --work-dir，用位置参数 FILE）
wfl lint  rules/<rule>.wfl --schemas "schemas/*.wfs"    # -> No issues found.
wfl test  rules/<rule>.wfl --schemas "schemas/*.wfs"    # -> 1 tests: 1 passed

# 离线回放（batch）跑通整条链路
wfusion batch --config wfusion.toml --work-dir .        # rows=N，正常关闭即通过

# 常驻接实时数据
wfusion daemon --config wfusion.toml --work-dir .
```

- 产物在 work-dir 下 `data/out_dat/alerts.ndjson`（file sink 默认落点）：
  断言行数 + 关键字段（`__wfu_entity_id`、`alert_type` 等）；`error.ndjson`
  应为空；
- `run.sh` 一键惯例：lint + 内联测试 + 清理产物 + batch + 断言（照抄
  hello_detection / match_expr_key_demo 的 run.sh）；`examples/rules/run_all.sh`
  会遍历每个示例做 lint/test 并对登记的 case 跑 batch 断言。

## 已踩过的坑（接线层）

1. connector 定义用了 `[connector] name/sink_type` → 改 `[[connectors]]
   id/type/allow_override` + `[connectors.params]`（否则 "load sink config"）。
2. `windows.toml` 文件被 `wfusion.toml` 引用却缺失 → "configuration parse
   error"；文件必须有，条目可省（落 `[window_defaults]` 默认）。
3. 旧字段名：`stream =`（→ `stream_tag`）、`format =`（→ `data_format`）、
   `wfusion run`（→ `wfusion batch` / `wfusion daemon`）。
4. sink group 的 `windows` 写成了 schema 窗口名之外的任意名 → 无路由、静默
   丢告警；必须命中第 3 步输出窗名。
5. `over = 0` 输出窗在 `windows.toml` 常配 `over_cap = "0s"` 与 1MB 上限
   （不是必须，但能避免输出窗按输入窗口径囤数据）。
6. 给用户的可抄文件里别放注释行进 JSON/ndjson 代码块（照抄会坏文件）。

## 检查清单（对外交付前）

- [ ] 五步次序齐全：来源 → 输入窗 → 输出窗 → sink 路由 → 规则
- [ ] 来源 `stream_tag` 与输入窗 `stream_tag` 一致（ndjson 行内 `_stream` 同步）
- [ ] 输入窗有 `time` 字段；输出窗 `over = 0`
- [ ] sink group `windows` 命中输出窗名；`connect` id 与 connector 定义一致
  （`[[connectors]]` 数组形式）
- [ ] `wfl lint` / `wfl test` 通过；`wfusion batch` 正常退出
- [ ] `data/out_dat/alerts.ndjson` 行数与内容断言通过；`error.ndjson` 为空
- [ ] 工程可整目录照抄（无对真实路径的隐式依赖）；运行产物已忽略/清理
