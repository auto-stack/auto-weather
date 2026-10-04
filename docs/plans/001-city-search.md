---
plan_id: PLAN-001
status: reviewed
feature_name: 城市搜索与多城市管理
author: [agent]
created_at: 2026-10-04T04:00:00Z
updated_at: 2026-10-04T05:30:00Z
plan_revision: 1
current_step: 6
total_steps: 6
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P0-01]
touched_goals: [F-P0-01]
---

# PLAN-001 城市搜索与多城市管理

## 0. 变更摘要

为 auto-weather 增加任意城市搜索（Open-Meteo Geocoding）、自定义城市
增/切/持久化，打通"非内置城市也能看真实天气"的核心闭环。后端新增 3 个
`#[api]` 端点（search / cities_get / cities_add / report_at），前端新增
搜索区与动态自定义城市 pill 条。坐标以 2dp 字符串为全链路透传格式。

## 1. 目标

- **Goal**：用户可搜索全球任意城市（中文名/拼音），添加到城市条并查看
  其实时+预报天气；城市列表跨启动持久化。
- **Non-goals**：城市删除/排序（PLAN-002）；定位/GPS（未排期）；搜索历史
  （PLAN-006 设置中心一并考虑）。
- 受影响仓库/模块：auto-weather（`src/back/api.at`、`src/front/app.at`、
  `tests/vm_smoke.py`、README、`docs/specs/weather-app.md`）。
- 成功标志：AC-01..AC-05 全部通过，`vm_smoke` 全绿。

## 2. 架构方案

遵循 `docs/design/weather-v2.md`：后端代理 + 平行 str 数组契约 + 后端
JSON 文件持久化（`Env.local_data_dir()/auto-weather/cities.json`）。
前端 pill 条 = 既有 10 内置城（硬编码渲染不动）+ 尾部动态渲染
`.custom_pills`（for + onclick，风险与回退见 T-05）。

## 3. 技术栈

AutoLang `.at`（`#[api]`/widget/VM merged）、Open-Meteo Geocoding API、
`file.read_text/write_text` + `Env.local_data_dir()`、既有 `vm_smoke.py` MCP 驱动。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04 指令）："设计需求目标和设计文档，分成多个计划，
  先给出第一个实施计划，用 /auto-plan:new 写出计划文件，然后再用
  /auto-plan:work 技能实施"——本计划即"第一个实施计划"，new+work 两段
  授权均已给出；review/merge 未授权。
- 背景证据：`src/back/api.at`（v0.6-dev slice-1，commit `3e029f1`）已实证
  VM 配方（点访问/for-in/平行数组预生成/float 去装箱）；设计文档 §6 七条
  约束为硬约束。
- Geocoding 响应形态（实测）：`{"results":[{"name":"青岛","latitude":36.07,
  "longitude":120.38,"country":"中国","admin1":"山东省",…}]}`；`results`
  可缺席（无结果）。
- 依赖：网络可达 geocoding-api.open-meteo.com（与现役 forecast 同域族）。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- `pub type SearchOut = { ok: bool, error: str, ids: []str, names: []str,
  subs: []str, lats: []str, lons: []str }`——平行 str 数组契约（设计 §4）。
- `pub type CityListOut = { ok: bool, error: str, ids: []str, names: []str,
  lats: []str, lons: []str }`。
- `fn fmt_latlon(f float) str`：`(f+100.0)*100.0` 移位取整（f2i 去装箱），
  拆 `ip = v/100-100`、`fp = v%100`（pad2），负值补 '-'——全整数运算。
- `fn quote_json(s str) str`：os-config 转义配方。
- `fn impl_search(q str) SearchOut`：geocoding `count=8&language=zh`；for-in
  `v.results` 预生成 5 个平行列表（空结果零迭代 → ok:true 空表）。
- `fn cities_path() str`、`fn impl_cities_get() CityListOut`（文件缺省 →
  ok:true 空表；损坏 JSON → ok:false）、`fn impl_cities_add(id, name, lat,
  lon) CityListOut`（读-判重-追加-重写）。
- `fn impl_report_ll(lat str, lon str) ReportOut`：`impl_report` 重构核心
  （`ll` 直接来自参数，不再 `coords.split`）；`impl_report(city)` 薄包装
  内置表；`#[api] report_at(lat str, lon str)` 新端点。
- 端点（全部 thin 体）：`GET /api/weather/search?q=`、
  `GET /api/weather/cities`、`POST /api/weather/cities`（body 字段
  id/name/lat/lon 按名注入）、`GET /api/weather/report_at?lat=&lon=`。

### 5.2 前端（src/front/app.at）

- model 新增：`search_q/search_err str`、`search_ids/search_names/search_subs
  /search_lats/search_lons []str`、`custom_ids/custom_names/custom_lats/
  custom_lons []str`（平行数组，handler 索引循环物化——VG12-14 规则）。
- `use back.api` 增列 4 个新函数。
- Header 区加搜索行：`input{value:.search_q, oninput:.SearchChanged}` +
  "搜索"按钮 `.DoSearch`；结果区 `for h in .search_hits_view`（视图态
  列表，由 handler 从平行数组物化）每行"名称 · 副标题"按钮
  `onclick: .AddHit(h.id)`。
- pill 条尾部：`for c in .custom_view` 动态按钮 `onclick:.SelectCustom(c.id)`。
- handler（全部薄）：`.SearchChanged(q)` 存串；`.DoSearch` 清旧结果→
  `weather_search(.search_q)`→ok 则索引循环 push 视图列表；`.AddHit(id)`
  索引循环定位 → `cities_add(...)` → 刷新 `.custom_*`；`.SelectCustom(id)`
  定位 → `report_at` → ok 则全量覆写（复用既有赋值块）+ `city_zh=name`；
  `.Init` 追加 `cities_get()` 预填 `.custom_*`（失败静默）。
- **风险登记**：视图 for + onclick 携带循环变量实参在 App widget 内无直接
  先例（VG16 之祸在 store 语境）。回退方案：onclick 无参 `.AddHit0..7` 固定
  8 槽位（if 渲染守卫）——T-05 验证时若 MCP 点击后参数为空即触发回退。
- **实施调整（已记录）**：① `--server=vm` 模式存在 E5501 脚本门（nil 字面量
  信号），api.at 首行加 `#[script]` 标注解决，merged 模式回归实证无影响；
  脚本模式严格 let 不可重赋值（fmt_latlon 的 fp 改 var）。②
  `url.encode_query_component` 在 server 模式 VM 上非可用面，搜索 q 改
  原始 UTF-8 透传（Open-Meteo 接受）。③ 视图 for + onclick 循环实参
  **实测通过**（T7 MCP 点击闭环），回退方案未启用。④ 竖屏布局未加搜索区
  （归 PLAN-004 移动端打磨）。

### 5.3 测试（tests/vm_smoke.py）

- 新增 T7：DoSearch"青岛"→ 结果非空 → 点击首个结果 → 断言 `.custom_names`
  含青岛 → 点击该自定义 pill → 断言 `city_zh` 变青岛且
  `source_label=="Open-Meteo"`（真实数据面）。复用既有真实数据前置。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P0-01 行尾标注"PLAN-001 达成（删/排序归 PLAN-002）" | 账实对齐 | AC-05 |
| SD-02 | modify | README.md | 数据节补"城市搜索/自定义城市"用法 | 用户可见能力 | AC-05 |

## 6. 测试设计

- T7（新）见 §5.3；既有 T1-T6 不得回归（首绘重试已 30 次，Init 新增一次
  cities_get HTTP，冷启动首绘预算上调）。
- 后端契约直探（开发期）：`auto run --server=vm` 起后 curl 四个端点看
  `ok/形状`（证据记入 T-01..T-03）。

## 7. 验收标准

| ID | 标准 | 验证 | 结果 |
|---|---|---|---|
| AC-01 | `search?q=青岛` 在 VM merged 下 ok:true 且 names 含"青岛"，坐标为 2dp 文本 | T7 search_names=["青岛",…]+快照渲染 | **pass**（worktree vm_smoke 21/21） |
| AC-02 | `cities_add` 后 `cities_get` 返回该城；重启进程后仍在（文件持久化） | 第二轮 smoke pre-state `custom_names:["青岛"]` | **pass**（wt-smoke5 实证） |
| AC-03 | 选中自定义城市后 model 的 city_zh/temp/updated_at 来自 report_at 真实数据 | T7 SelectCustom 断言 city_zh=青岛+Open-Meteo | **pass** |
| AC-04 | `vm_smoke.py` 全绿（18 旧 + T7） | python tests/vm_smoke.py ×2 全绿（21/21） | **pass** |
| AC-05 | SD-01/SD-02 增量落地 | commit f7eb776 文件检查 | **pass** |

## 8. 执行步骤

- [x] T-01 后端 search 端点 + SearchOut + fmt_latlon（AC-01）
  - 验证：`auto run --server=vm` 直探（server 模式调通后 merged T7 复验）；
    证据：T7 search_names 非空（`f2i/fmt_latlon` 全整数路径，merged 21/21）
- [x] T-02 后端 cities_get/cities_add + cities_path + quote_json（AC-02）
  - 验证：cities_get 空表瞬回（server 直探）+ 第二轮 smoke pre-state 持久化实证
- [x] T-03 后端 report_at + impl_report 重构（AC-03 前置）
  - 验证：server 直探 青岛 18°C/晴/风11km/h（真实数据，亚秒返回）
- [x] T-04 前端搜索区（input/按钮/结果列表/DoSearch/AddHit）（AC-01/03）
  - 验证：T7 前半段（录入→搜索→结果渲染，autoui_type）
- [x] T-05 前端动态 pill + SelectCustom + Init 预填（AC-03）
  - 验证：T7 后半段；循环实参实测通过，回退方案未启用
- [x] T-06 vm_smoke T7 + SD-01/SD-02 + 全量回归（AC-04/05）
  - 验证：`python tests/vm_smoke.py` 21/21 ×2（worktree plan-001-dev @ f7eb776）

依赖：T-01→T-04；T-02→T-05；T-03→T-05；T-06 最后。

## 9. 复审记录

- `stage: new | PLAN-001 | rev 1 | outcome: pass | next: work`——授权内
  （用户已批 new+work），任务覆盖 AC-01..05 与 SD-01/02，路径/命令已对照
  仓库核实（src/back/api.at、src/front/app.at、tests/vm_smoke.py 均在册）。
- `stage: work | PLAN-001 | rev 1 | outcome: pass | code_commit: f7eb776
  (on plan-001-dev, base 327bcc7) | task_ids: T-01..T-06 | evidence:
  worktree vm_smoke 21/21 ×2（wt-smoke4/5 全绿；AC-02 跨重启 pre-state 实证）;
  server 模式直探 cities_get/report_at 通过 | blockers: 无（--server=vm 下
  search 曾现 encode 原生件缺失，已改 UTF-8 透传并实证） | next: review`——
  所有任务与 AC 映射已核销，变更已提交，worktree 保留待复审。
- `stage: review | PLAN-001 | rev 1 | outcome: pass |
  reviewed_commit: f7eb776a376dd0ee3f58b8dfd4959f831dcbd53e |
  base_commit: 327bcc70e5fd42cbb72afd3c55a1345c1f3d3171 |
  dependency_revisions: auto-lang 168b56923（仅验证工具链 auto.exe，无源码改动）|
  spec_inputs: docs/specs/weather-app.md（worktree 含 SD-01）、README.md
  （worktree 含 SD-02）——增量已核：文本描述当前行为、持久决策，未发布（待 merge）|
  acceptance_results: AC-01 pass（T7 search_names 非空+快照含青岛；2dp 子项由
  fmt_latlon 源码+T-03 直探 36.07/120.38+add payload id 36.07_120.38 三角实证）/
  AC-02 pass（复审复现：pre-state custom_names["青岛"] 跨进程重启）/
  AC-03 pass（T7 SelectCustom：city_zh=青岛+source_label=Open-Meteo）/
  AC-04 pass（复审复现 worktree vm_smoke 21/21 exit 0）/
  AC-05 pass（SD-01/02 文件核查在案）|
  findings: F-001 info——计划 §5.2 文案"名称 · 副标题"与实现（label 仅 name）
  有漂移，非 AC 约束，建议 PLAN-002 渲染副标题；F-002 info——SelectCustom
  不置 .city_id，内置 pill 高亮与显示城市可能短暂不一致（纯视觉），建议
  PLAN-004 一并处理。均无阻塞 |
  evidence: 复审复现 /tmp/review-smoke1.out（21/21，pre-state 持久化）；
  worktree 零脏文件（status 0 行）；worktree 定位 git worktree list 实证 |
  next: merge`。复审局限声明：复审与实施同会话，结论全部经工件与复现命令重建。

## 10. 待澄清事项

- 无（删除/排序、搜索历史、GPS 已明确归入后续计划；竖屏搜索区归 PLAN-004）。
