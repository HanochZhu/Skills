# 对照：daily_tools 拉取优化

- 日期：2026-09-03
- 样本日：2026-09-02（只读聚合，未拉全日明文）

## 现状路径

`fill_day` / `fill_agent_day`：

1. `fetch_day_stats` / `fetch_day_agent_stats`：一日一份 cheap count（不拉 input/output）
2. 旧实现：对每个 name 调用 `fetch_day_calls`，`LIMIT 500 OFFSET n`
3. 写入 SQLite；`expected_count` 来自步骤 1，用来发现少拉

约束（不可破）：

- SRE MCP 默认约 1000 行；单次 unscoped 会被截断，低频 name 会丢（见 `test_fill_day_keeps_rare_tool_when_unscoped_query_is_capped`）
- MCP RPC 间隔约 1.1s，查询次数 ≈ 墙钟
- 成功/失败靠 `level` 和 output 里的 `"success": false`，回填阶段 output 不能砍

## 摸底数字（工具）

| name | n | avg_in | avg_out |
|------|---:|-------:|--------:|
| execute_csharp_script | 5508 | 2032 | 116 |
| unity_editor | 4339 | 76 | 46 |
| unity_console | 1552 | 152 | 21 |
| unity_scene | 1463 | 85 | 31 |
| 其余 19 个 name | 3381 | — | — |
| **合计** | **16243** | **815** | **70** |

`max(input) = 40172`，所以 input 截断只打尾部，打不动当天均值。

Agent：207 行 / 15 个 name。

## 沙盒结果

模型：`app/fetch_cost.py`。score = `(queries, scan_rows, payload)`。

| 方案 | queries | scan_rows | payload | 结论 |
|------|--------:|----------:|--------:|------|
| per_name_offset（baseline） | 50 | 72743 | 14.4MB | 对照 |
| unscoped_offset | 34 | 再扫更多 | 同左 | **拒绝**（查询少、扫描更差） |
| unscoped_keyset | 34 | 16243 | 同左 | 赢 |
| unscoped_keyset_slim | 34 | 16243 | 均值持平，尾部更小 | 赢（实现便宜） |
| unscoped_keyset_slim_backfill | 34（完整日） | 16243 | 同上 | **采用**（缺数再补拉） |

Agent：`16 → 2` 次查询。

RPC 下限：工具日 `50 × 1.1s ≈ 55s` → `34 × 1.1s ≈ 37s`。

## 采用后的实现要点

- `fetch_day_calls` / `fetch_day_agent_calls`：`(call_start, id)` keyset，禁止 OFFSET
- `substring(input, 1, 4000)`；output 保持全文
- `fill_day` 先 `fetch_day_calls(day, None)`，再用 stats 对账，缺的 name 才按 name 补
- 回归：`tests/test_fetch_sandbox.py`、`test_fill_day_keeps_rare_tool_when_unscoped_query_is_capped`、`test_fill_day_skips_per_tool_when_unscoped_is_complete`

## 未采用

- 去掉 stats 查询：省 1 次，但对账和补拉失去基准
- 回填阶段截断 output：详情和失败判断会坏
- 把 trace 现查（`fetch_call_trace` / nearby generation）并进这次回填优化：路径不同，应另开沙盒
