---
name: sandbox-fetch-optimize
description: >-
  Sandbox-first data-fetch optimization: measure baseline cost, compare multiple
  schemes in an isolated sandbox, and adopt a change only when the comparison
  shows a real win. Use when reviewing pull/fetch/cache-fill pipelines, reducing
  ClickHouse/MCP/API query cost, replacing OFFSET pagination, or the user asks
  to 优化拉取 / 降低查询消耗 / 沙盒对比.
---

# Sandbox Fetch Optimize

优化拉取必须先建沙盒、再对比、有收益才落地。可同时评估多个方案；对比没有赢的方案不采用。

## 何时启用

- 回顾或优化「从远程库/API 拉数 → 本地缓存」的路径
- 用户提到：优化拉取、降低消耗、少打库、分页、MCP/ClickHouse 查询成本
- 准备改 `fill_*` / `fetch_*` / 列表回填，且改动会影响查询次数或扫描量

不要用于：只改 UI、只改本地 SQLite 读、已经量过且方案已选定的小修补。

## 工作流

复制并勾选：

```
Fetch optimize:
- [ ] 1. 画出现状拉取路径（谁触发、打几次、每页多大、有无 OFFSET）
- [ ] 2. 用廉价聚合摸清一天的量（count / uniq / avg payload），不要先全日拉明文
- [ ] 3. 建立沙盒：用真实分布做成本模型，或拦截 query 层只记账
- [ ] 4. 列出 ≥2 个方案，各自算出 queries / scan_rows / payload
- [ ] 5. 与 baseline 比；只有 score 严格更优才进入实现
- [ ] 6. 实现赢家；用回归测试锁住正确性（低频项、分页、对账）
- [ ] 7. 再跑同一套沙盒数字，确认落地后的模型仍赢
```

### 1. 画出现状

先读缓存回填入口，再读远程查询。记清：

| 项 | 要回答 |
|----|--------|
| 触发 | 缺日 / 定时 / 用户下钻 |
| 形状 | 先 count 再按 name 循环，还是一日一份 |
| 分页 | `LIMIT/OFFSET` 还是 keyset |
| 列 | 是否把 input/output 全文拉回 |
| 上限 | MCP `maxRows`、默认 1000、token QPS |
| 正确性约束 | 低频 name 不能丢；`expected_count` 要对得上 |

本仓库的回填入口：`app/cache.py` 的 `fill_day` / `fill_agent_day`；查询：`app/source.py`。

### 2. 廉价摸底

只跑聚合，禁止把这一步写成全日明细拉取。

需要的数字：

- 一日 `uniqExact(id)`、name 数
- 每个 name 的 n（用来算「按 name 分页」的查询次数）
- `avg(length(input))` / `avg(length(output))` / max（用来判断截断有没有意义）

有 MCP / 只读连接就用它；没有就用缓存里最近一天的直方图，并标明「样本不是线上」。

### 3. 建立沙盒

沙盒要和线上拉取隔离：

- **成本模型**（优先）：用摸底直方图纯算，不打库。本仓库：`app/fetch_cost.py` + `tests/test_fetch_sandbox.py`
- **拦截记账**：monkeypatch `query_json` / `execute_query`，数 SQL 次数和是否带 `OFFSET`
- **禁止**：用生产回填当实验；禁止为了对比把同一天全日重拉两遍明文

沙盒输出至少包含每个方案的：

- `queries`：远程往返（含一次 cheap count）
- `scan_rows`：OFFSET 累加扫描 vs keyset 每行一次
- `payload_bytes`：拉回的 input/output 估算
- `rpc_sec`：若客户端有最小间隔（本仓库 MCP 约 1.1s/次），用 `queries * interval`

默认 score：`(queries, scan_rows, payload_bytes)`，越小越好。先比查询次数（墙钟往往被 RPC 间隔主导），再比扫描，再比字节。

### 4. 同时评估多个方案

至少覆盖这些方向（按需改名，不要只做一个就改代码）：

| 方案 | 想法 | 常见风险 |
|------|------|----------|
| per_name + OFFSET | 现状：每个 name 一串 OFFSET 页 | 查询数 = 1 + Σ ceil(n_i/page)；大 OFFSET 重复扫描 |
| unscoped + OFFSET | 一日一份，少查询 | 查询少，但大 OFFSET 扫描可能比按 name 更差 → **可拒绝** |
| unscoped + keyset | `(call_start, id)` 游标，每行扫一次 | 必须自己分页到空，不能依赖单次 `LIMIT` |
| unscoped + keyset + slim | 截断列表用不到的大列 | 截断不得破坏 parse / 成功判断；详情仍要能现查全文 |
| unscoped + 缺数补拉 | 先全日拉，count 对账后再按缺的 name 补 | 完整日不增加查询；MCP 截断时仍能保住低频项 |

还可以加、但默认不当赢家，除非沙盒证明更优：

- 丢掉 cheap count（对账会瞎）
- 列表阶段不拉 output 全文却仍要在 SQL 外判断 `"success": false`
- 为省字节把详情也截断且不提供现查

### 5. 采用规则

只采用同时满足的方案：

1. `score` 严格小于 baseline（元组逐项，先比较的项更优即赢）
2. 正确性不回退：低频 name、分页完整性、`expected_count` 对账仍在
3. 若 A 查询更少但扫描明显更差，换用「查询少且扫描不差」的变体（例如拒绝 unscoped+OFFSET，改用 keyset）

并列第一时：选实现更简单、正确性兜底更强的（本仓库选「keyset + slim + 缺数补拉」）。

对比没有赢：保持现状，把否决原因写进回复，不要为了「做过优化」而改。

### 6. 落地时锁住的行为

- 分页：禁止再引入 `OFFSET`；页大小仍要低于上游行上限（本仓库 500 < MCP 默认 1000）
- 先一日 keyset，再用当日 count 对账；缺的 name 才按 name 补拉
- 列表/回填可截断 input；**不要截断用来算成功失败的 output**，除非成功标志已在 SQL 里算完且详情另有现查
- 单测至少覆盖：分页跨页、unscoped 被截断时低频项仍在、unscoped 已完整则不再按 name 打

### 7. 复跑沙盒

实现后用同一份直方图再算一遍。数字应与采用前的赢家一致（完整日查询数 = 1 + ceil(total/page)）。

## 本仓库对照（2026-09-02 摸底）

工具日：16243 行 / 23 个 name。Baseline `50` 次查询、`72743` 行 OFFSET 扫描；采用后 `34` 次、`16243` 行。Agent 日约 `16 → 2` 次。细节见 [reference.md](reference.md)。

## 反模式

- 不建沙盒，直接改线上 SQL
- 只比「感觉更快」，没有 queries/scan/payload
- 用单次 unscoped `LIMIT` 代替分页（会丢掉低频项）
- 为减查询而去掉 count 对账
- 同时采用所有方案，包括沙盒里已经输的
