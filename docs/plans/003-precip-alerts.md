---
plan_id: PLAN-003
status: executing
feature_name: 24小时降水展示与天气预警展示位
author: [agent]
created_at: 2026-10-04T09:00:00Z
updated_at: 2026-10-04T09:00:00Z
plan_revision: 1
current_step: 0
total_steps: 4
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P1-01, docs/specs/weather-app.md#F-P1-02]
touched_goals: [F-P1-01, F-P1-02]
---

# PLAN-003 24小时降水展示与天气预警展示位

## 0. 变更摘要

新增"24小时降水"区：逐小时降水概率（%）+ 降水量（mm）条形图（24 根，
从当前小时起，与 24h 温度条同窗切出）；条形高度按量分档、颜色按概率
分档，全部由后端算好 class 字符串，前端 `style: p.bar` 直接绑定（
031-paint/038-minesweeper 实证动态 style 绑定可用）。同时落地天气预警
**数据契约与展示位**：`ReportOut` 增 `alert_level/alert_title/alert_text`
（默认空串=无预警，banner 不渲染），真实预警内容待 PLAN-005（QWeather
JWT）接入。分钟级降水超出 Open-Meteo 免费面，明确归 QWeather。

## 1. 目标

- **Goal**：报文带回 24h 降水序列并渲染条形图；预警契约字段就位、
  展示位按条件渲染（默认隐藏）。
- **Non-goals**：分钟级降水（QWeather）；真实预警内容（PLAN-005）；
  预警点击详情；竖屏新区块（PLAN-004 口径延续）。
- 受影响模块：`src/back/api.at`、`src/front/app.at`、`tests/vm_smoke.py`、
  README、`docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-05 全过，vm_smoke 全绿。

## 2. 架构方案

遵循 design-v2 §6 配方：降水概率走 **raw 文本 key 扫描**（
`"precipitation_probability":` 可能为 null——PLAN-002 A3 教训：parse
结果数字面只信字符串通道；null 段→-1 哨兵→前端 "—"）；降水量走
for-in 元素 push（数值元素面已实证安全，与 htemps 同构）。条形 class
后端拼装（高度 6 档 × 颜色 5 档），前端绑定。预警字段常驻报文、空串
缺省。

## 3. 技术栈

AutoLang .at、`#[api]`/VM merged、Open-Meteo forecast
`hourly=precipitation,precipitation_probability`（实测可用）、vm_smoke MCP。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04"继续用技能规划计划3,并实施"）：沿用
  new+work 两段授权；review/merge 未授权。
- 前置：PLAN-001/002 已归档（7511bc3、dbfff44）。
- 实证：① `style: .field`/`style: obj.field` 动态绑定先例存在
  （031-paint、038-minesweeper）；② 降水两参数 probe 返回正常单位
  （%/mm）；③ precipitation_probability 可能为 null（文档与社区口径），
  必须 null 安全。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- `pub type PrecipPoint = { time: str, prob: str, amount: str, bar: str }`；
  `ReportOut` 增 `precip: []PrecipPoint`、`alert_level/alert_title/
  alert_text: str`（默认 ""）。
- `fn scan_raw_segments(raw str, key str) []str`：scan_numlist 的原始段
  变体（不做 2dp 截断）；impl 内对每段判 `"null"` → prob -1，否则
  `str.to_uint` → int。
- `impl_report_ll`：URL hourly 增 `precipitation,precipitation_probability`；
  24h while 循环内同窗推 `PrecipPoint`：
  - prob：`<30 "灰"`、30-59、60-84、≥85 四档文案（"—"当 -1）；
  - bar class：`"w-4 rounded-t shrink-0 "` + 色（prob 分档 sky-300/400/
    500/600 或 null 时 muted/40）+ 高（amount 6 档 h-1..h-6）；
  - amount：mm 原值 2dp 截断 + "mm"。
- 预警字段：本期不赋值（保留 ""），头注说明 PLAN-005 接 QWeather 填充。

### 5.2 前端（src/front/app.at）

- model 增：`precip` 列表、`alert_level/alert_title/alert_text str`。
- 四个赋值块各增 5 行（precip + alert×3，replace_all + SelectCustom 补）。
- 视图（横屏，生活指数区之后）：`text "24小时降水"` + scroll(x) row
  `for p in .precip`：每点 col（w-10 items-center gap-0.5）= time(9px) +
  bar col（`style: p.bar`，min-h 兜底 h-1）+ prob(9px) + amount(8px)。
- 预警 banner：app_frame 顶部 `if .alert_title != ""` 条件块（badge
  `.alert_level` + `.alert_title`）——默认空串不渲染。

### 5.3 测试（tests/vm_smoke.py）

- 新增 T9：① 快照含"24小时降水"；② state `precip` 恰 24 项（vmref
  计数）；③ 快照含"mm"；④ `alert_title` 为 ""（banner 默认隐藏，
  快照不含"天气预警"）；既有 T1–T8 不回归。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P1-01 标"逐小时降水达成（003），分钟级归 QWeather"；F-P1-02 标"展示位+契约达成（003），内容待 PLAN-005" | 账实对齐 | AC-05 |
| SD-02 | modify | README.md | 数据节补降水区/预警位说明 | 用户可见能力 | AC-05 |

## 6. 测试设计

- T9（新）见 §5.3；网络依赖与 T6 同口径（概率段 null 时对应点显示 "—"，
  不判单点值）。

## 7. 验收标准

| ID | 标准 | 验证 |
|---|---|---|
| AC-01 | 报文带 24 点降水序列（time/prob/amount/bar），概率 0-100 或 -1 哨兵 | T9 ②（state 计数 24） |
| AC-02 | 横屏"24小时降水"区渲染（含 mm 标注） | T9 ①③ |
| AC-03 | 预警契约字段默认 ""、banner 默认隐藏、无崩溃 | T9 ④ |
| AC-04 | vm_smoke 全绿（26 旧 + T9 新 4 项 = 30）×2 | python tests/vm_smoke.py |
| AC-05 | SD-01/02 落地 | 文件核查 |

## 8. 执行步骤

- [ ] T-01 后端：PrecipPoint + 扫描/分档/bar 拼装 + alert 字段（AC-01/03）
  - 验证：T9 ②④ / merged smoke
- [ ] T-02 前端：降水区 + 预警 banner + 赋值块（AC-02/03）
  - 验证：T9 ①③④
- [ ] T-03 tests：T9 + 全量回归（AC-04）
  - 验证：vm_smoke ×2 全绿
- [ ] T-04 docs：SD-01/02（AC-05）
  - 验证：文件核查

依赖：T-01→T-02；T-03/04 最后。

## 9. 复审记录

- `stage: new | PLAN-003 | rev 1 | outcome: pass | next: work`——授权内
  （"规划计划3,并实施"沿用 new+work），任务覆盖 AC-01..05 与 SD-01/02。

## 10. 待澄清事项

- 分钟级降水与真实预警内容依赖 QWeather JWT（PLAN-005）；降水区交互
  （点击某小时看详情）未排期。
