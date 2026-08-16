---
name: wf-wfl-authoring
description: Warp Fusion WFL 检测规则编写、修改、检查与 lint 指南。用于处理 `.wfl` 规则文件、events/match/on each/yield/entity/limits/score 语法、schema 对齐、规则不触发和编译错误排查。
---

# AI Agent 指南：WFL 规则编写

> 供 AI agent 编写、修改、检查 `.wfl` 检测规则时遵循。
>
> 适用版本：`wf-lang` v2.1+。配套 schema 编写见 `wf-wfs-authoring`。

## 规则文件结构

```wfl
use "network.wfs"              // 声明依赖的 schema 文件

rule port_scan {               // 规则名：小写 + 下划线
    events {                   // 1. 事件声明
        c : conn_events && c.action == "syn"
    }
    match<sip:5m> {            // 2. 匹配窗口
        on event { port_scan: c.dport | distinct | count >= 10; }
        and close { final_conn: c | count >= 10; }
    } -> score(80.0)           // 3. 分数
    entity(ip, c.sip)          // 4. 实体
    yield network_alerts (     // 5. 输出
        sip = c.sip,
        alert_type = "port_scan",
        distinct_ports = stat.count(match_distinct(port_scan)),
        final_count = stat.value(final(final_conn)),
        detail = ">=10 ports in 5min"
    )
    limits {                   // 6. 治理（v2.1 必填）
        max_memory = "64MB";
        max_instances = 10000;
        on_exceed = throttle;
    }
}
```

## 核心规则

### 1. `events` — 事件过滤

```
events { alias : window && filter }
```

| 写法 | 含义 |
|------|------|
| `e : auth_events` | 所有 auth_events |
| `e : auth_events && e.result == "failed"` | 过滤失败事件 |
| `e : conn_events && e.dport == 22` | 过滤 SSH 连接 |
| `e : conn_events && (dport == 22 \|\| dport == 445)` | 多端口 OR |

**重要**：只能用 `==`、`!=`、`&&`、`||`，不支持 `>`、`<`（这些放 match 阶段）。

### 2. `match` — 匹配窗口

```
match<key:window> { on event { ... } and close { ... } }
```

- **`key`**：分组字段（`sip`、`sip,dip`），每个分组独立窗口
- **`window`**：时间窗（`5m`/`30m`/`1h`），不超过 schema `over`
- **`on event`**：触发条件，每次事件都评估
- **`and close`**：关闭条件（可选），窗口结束时评估

**常见模式**：

| 模式 | 写法 | 场景 |
|------|------|------|
| 频率阈值 | `e \| count >= 10` | brute force |
| 去重计数 | `e.dport \| distinct \| count >= 5` | port scan |
| 求和 | `e.bytes_out \| sum >= 10000000` | data exfil |
| 全部命中 | `a\|count>=1; b\|count>=1; c\|count>=1` | 多步攻击链 |

**多别名事件**（非同一数据源时）：

```wfl
events {
    scan  : conn_events && bytes_out < 1000
    login : auth_events && result == "success"
    xfer  : conn_events && bytes_out >= 10000
}
match<sip,dip:30m> {
    on event {
        scan | count >= 1;
        login | count >= 1;
        xfer | count >= 1;
    }
}
```

#### `on event seq` / `on event any` — 序列与共现

`on event` 的 `seq` / `any` 修饰符声明步骤的**排序模式**。裸 `on event { ... }` 等价 `seq`，向后兼容。

**`on event seq { ... }` — 有序序列（攻击链）**：步骤按书写顺序完成，支持 `has` / `within` / `not` / `consec` / `skip`：

```wfl
match<sip,dip:30m> {
    on event seq {
        has scan;                  // 存在性步骤：等价 count >= 1
        has login within 10m;      // login 必须在 scan 完成后的 10m 内
        not has failed within 5m;  // 否定步：scan 后 5m 内不得出现失败登录
        scan.dport | distinct | count >= 5;  // 聚合步骤复用 pipe
    }
}
```

- `has <alias>`：存在性步骤，等价 `count >= 1`
- `within <dur>`：本步完成时刻 − 上一步完成时刻 ≤ dur
- `not has <alias> within <dur>`：否定步，自上一完成步骤起 dur 内不得出现匹配事件
- `consec`：严格相邻修饰符（默认允许步骤间夹带无关事件）；`skip = past_last | to_next`（`to_next` 延后 L3）

**`on event any { ... }` — 无序共现**：所有 step 并行评估，全部满足即触发，顺序无关：

```wfl
match<sip,dip:10m> {
    on event any {
        scan | count >= 1;
        login | count >= 1;
        xfer | count >= 1;
    }
}
```

- `any` 不支持 `within` / `not` / `consec` / `skip`（依赖顺序，编译期拒绝）
- `seq` / `any` 不支持用于 pipeline stages

#### `|>` 多级管道

```wfl
match<sip,dport:5m> {
    on event { d | count >= 1; }
    on close { d | count >= 3; }
}
|> match<sip:10m> {
    on event { _in | count >= 1; }
    on close { _in | count >= 10; }
} -> score(80.0)
```

- 中间 stage 不允许带 `-> score(...)`；最终 stage 必须带 `-> score(...)`
- 下游 stage 通过 `_in` 读取上一 stage 输出
- `_in` 字段可被 `entity`、`yield` 直接引用（如 `entity(ip, _in.sip)`）

### 3. `on each` — 逐条外部查询

```wfl
on each e where external("password_check", e.password_hash) -> score(75.0)
```

- 每条事件调一次 `external()`
- 返回值参与 `where` 判断（true 则命中）
- 依赖 `knowdb.toml` 中的 `[fun.<name>]` 配置
- **限制**：不能用于 `match` 内（`match<key>` 只能用聚合）

### 3.1 `join` — 关联

支持 `snapshot` / `asof` / `asof within` / `anti`：

```wfl
join geo_lookup snapshot on sip == geo_lookup.ip
join conn_risk asof within 24h on sip == conn_risk.ip
join blocked_list anti on sip == blocked_list.ip
```

| 模式 | 语义 |
|------|------|
| `snapshot` | 取右表当前快照 |
| `asof` | 按事件时间回看最近一条 `ts <= event_time` |
| `asof within` | 在指定时间范围内回看 |
| `anti` | 排除式关联（白名单排除），仅保留右表无匹配的左记录 |

- join 位于 `match` / `on each` stage 之后、`entity` 之前
- 支持多条件：`join t snapshot on sip == t.ip && dport == t.port`

### 4. `yield` — 输出声明

```wfl
yield security_alerts (
    sip = e.sip,           // 字段赋值
    alert_type = "name",   // 字符串字面量
    detail = "text"
)
```

- 目标 window 必须在 `.wfs` 中声明，`over = 0`
- 字段名必须存在于目标 window schema 中
- `__wfu_*` 是系统保留前缀，不能作为业务字段

### 4.1 输出稳定统计上下文

在 `yield` 中输出“为什么触发”时，优先使用稳定统计上下文，而不是直接依赖内部字段名：

```wfl
match<sip:5m> {
    on event {
        failed_hits: fail | count >= 3;
        port_scan: conn.dport | distinct | count >= 10;
    }
    and close {
        final_hits: fail | count >= 1;
    }
} -> score(70.0)

yield security_alerts (
    window_events = stat.count(window_event(fail)),
    matched_events = stat.count(match_event(failed_hits)),
    distinct_ports = stat.count(match_distinct(port_scan)),
    trigger_count = stat.value(trigger(port_scan)),
    final_count = stat.value(final(final_hits))
)
```

规则：
- `stat.count(...)` / `stat.value(...)` 只允许在 `yield` 表达式里使用。
- selector 参数是静态符号，不加引号：`window_event(fail)`，不是 `window_event("fail")`。
- `window_event(alias)` 引用 `events` 中的 source alias。
- `match_event(label)` / `match_distinct(label)` / `trigger(label)` 只能引用 `on event` step label。
- `final(label)` 只能引用 `and close` step label；不要在 `on close` OR 模式里使用 `stat.value(final(...))`。
- `match_event(label)` 要求对应 branch 使用 `count` measure。
- `match_distinct(label)` 要求对应 branch 使用 `distinct | count`。

### 4.2 `yield preset` — 输出模板复用

公共输出字段集合抽成 preset，降低每条规则重复填写告警字段的成本：

```wfl
yield preset base_alerts <severity, source = "wfusion"> (
    rule_name = @__wfu_rule_name,
    score = @score,
    severity = $severity,
    source = $source
)

rule scan {
    events { e : auth_events }
    on each e -> score(50.0)
    entity(ip, e.sip)
    yield scan_alerts : base_alerts<"high"> (
        alert_type = "scan",
        ioc_value = e.dip
    )
}
```

- `<severity, source = "wfusion">` 声明参数；`source` 带默认值，`$severity` / `$source` 在 preset 体内引用
- 调用时 `yield scan_alerts : base_alerts<"high"> (...)` 按位置传参；实参数量过多、未知 `$param` 都是编译错误
- 多个 preset 按顺序组合：`yield out : base_alerts, ioc_fields (...)`
- 必填参数不能排在带默认值参数之后；缺少必填参数是编译错误
- preset 不单独输出，也不绑定某个目标 window；展开后的字段仍按目标 window 做存在性/类型校验
- 项目级公共 preset 集中放规则根目录 `_global.wfl`（该文件只允许 `yield preset` 声明，不会自动启用普通 `rule`）

### 5. `entity` — 实体标识

```wfl
entity(ip, c.sip)         // entity_type, entity_id_expr
entity(user, e.user)
```

- 第一参数：实体类型（`ip`/`user`/`host`/`domain`）
- 第二参数：从哪个字段取值

### 6. `limits` — 治理（v2.1 必填）

```wfl
limits {
    max_memory = "64MB";
    max_instances = 10000;
    on_exceed = throttle;
}
```

| 字段 | 说明 |
|------|------|
| `max_memory` | 单规则最大内存 |
| `max_instances` | 最大状态机实例数 |
| `on_exceed` | 超限策略：`throttle`（唯一支持值） |

### 7. `score()` — 风险评分

```wfl
-> score(75.0)                           // 固定分数
-> score(if e.count > 10 then 90.0 else 50.0)  // 条件分数
```

**不允许**：`score()` 作为 `yield` 字段——score 是规则级元数据，不是输出字段。

## 常见错误

### ❌ 错误 1：`external()` 放 match 内

```wfl
match<sip:5m> {
    on event { e && external("check", e.hash) | count >= 1; }  // ❌ external 不能放在聚合里
}
```

**✅ 正确**：用 `on each`：
```wfl
on each e where external("check", e.hash) -> score(75.0)
```

### ❌ 错误 2：`yield` 窗口用了 `over > 0`

```wfl
window security_alerts {    // ❌ 告警窗口 should be over = 0
    over = 5m
    ...
}
```

**✅ 正确**：`over = 0`。

### ❌ 错误 3：缺少 `limits` 块

v2.1 要求每条规则必须含 `limits`。`wfl lint` 会拒绝。

### ❌ 错误 4：字段名与 schema 不一致

`.wfs` 里声明 `alert_type: chars`，但 `.wfl` 里写 `alert_type = e.type` ——如果 `e.type` 在 events 中不存在，编译时报错。

## 验证

```bash
# 单规则检查
wfl lint rules/my_rule.wfl -s "schemas/*.wfs"

# 全量检查
wfl lint rules/*.wfl -s "schemas/*.wfs"

# 编译计划查看
wfl explain rules/my_rule.wfl -s "schemas/*.wfs"

# 单元测试
wfl test rules/my_rule.wfl -s "schemas/*.wfs"
```

## 参考

- Schema 编写 → `wf-wfs-authoring`
- 配置编写 → `wf-config-authoring`
- 规则库 → `wf-rules/`
- 数据契约 → `wf-rules/DATA_CONTRACT.md`
