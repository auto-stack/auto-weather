---
plan_id: PLAN-002
status: reviewed
feature_name: 生活指数、空气详情与城市管理（删除）
author: [agent]
created_at: 2026-10-04T06:30:00Z
updated_at: 2026-10-04T08:10:00Z
plan_revision: 1
current_step: 6
total_steps: 6
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P0-03, docs/specs/weather-app.md#F-P0-04, docs/specs/weather-app.md#F-P1-05]
touched_goals: [F-P0-03, F-P0-04, F-P1-05]
---

# PLAN-002 生活指数、空气详情与城市管理（删除）

## 0. 变更摘要

在 PLAN-001 城市能力之上补全"常见天气 App 的标配三件套"：① 生活指数
（穿衣/洗车/运动/感冒/紫外线，由报文数据本地推导，零额外外呼）；② 空气
详情（PM2.5/PM10/O3 组分，复用既有 AQI 软失败通道扩展字段）；③ 自定义
城市删除（后端 cities_remove + 前端 pill ✕）。顺带兑现 PLAN-001 复审
发现 F-001（搜索结果渲染副标题）。排序明确归 PLAN-006（设置中心）。

## 1. 目标

- **Goal**：报文一次取回指数与空气组分；用户可删除自定义城市；搜索结果
  展示"名称 · 副标题"。
- **Non-goals**：城市排序/拖拽（PLAN-006）；指数点击详情页（P1 未排期）；
  竖屏布局新增区块（归 PLAN-004，与 PLAN-001 同口径）。
- 受影响模块：`src/back/api.at`、`src/front/app.at`、`tests/vm_smoke.py`、
  README、`docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-06 全过，vm_smoke 全绿。

## 2. 架构方案

遵循 `docs/design/weather-v2.md`：指数在 `impl_report_ll` 内由已有变量
推导（不新增外呼）；AQI 通道 `current=us_aqi,pm2_5,pm10,ozone` 一次取齐；
`ReportOut` 扩展 `pm25/pm10/o3 str` + `indices: []LifeIndex`（向后兼容：
Vue 桩 Default 不受影响）。删除走 `DELETE /api/weather/cities?id=`（
http-server 规范：DELETE 额外参数取 query；340 改写器已修 DELETE query 发射）。

## 3. 技术栈

AutoLang .at（`#[api]`/widget/VM merged）、Open-Meteo Air Quality API
（扩展参数）、既有 vm_smoke MCP 驱动。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04"继续计划002"）：沿用 PLAN-001 指令的
  new+work 两段授权；review/merge 未授权。
- 前置：PLAN-001 已归档交付（7511bc3）：report/report_at/search/cities
  端点、自定义城市 pill、T7 全链路。
- 复审发现 F-001（结果副标题未渲染）在本计划兑现。
- Open-Meteo air-quality `current` 变量名实证：`us_aqi, pm2_5, pm10,
  ozone`（μg/m³）；与 forecast 同软失败语义。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- `pub type LifeIndex = { name: str, level: str, note: str }`；
  `ReportOut` 增 `pm25: str, pm10: str, o3: str, indices: []LifeIndex`。
- `fn index_dressing(feels float) str`、`fn index_carwash(rain12h bool,
  precip float) str`、`fn index_sport(t float, aqi int, precip float) str`、
  `fn index_cold(tmax float, tmin float) str`——纯标量推导（边界见
  §6）；紫外线复用 `uv_level()`。
- `impl_report_ll`：AQI 块扩展取 pm2_5/pm10/ozone（软失败 → "—"）；
  24h 扫描期记录 `rain12h`（code 51-67/80-82/95-99 命中前 12 条）；
  今日 tmax/tmin 存 float；尾部 push 5 个 LifeIndex。
- `impl_cities_remove(id) CityListOut`：读-过滤-重写-返回新表；
  `#[api(method="DELETE", path="/api/weather/cities")] cities_remove(id str)`。

### 5.2 前端（src/front/app.at）

- model 增：`pm25/pm10/o3 str`（默认 "—"）、`indices` 列表。
- 四个赋值块（Init/SelectCity/Refresh/SelectCustom）各增 4 行
  （pm25/pm10/o3/indices）。
- 视图（横屏）：①"当前详情"下新增第四行 PM2.5/PM10/O3 三个 metric_card；
  ②新增"生活指数"区（section_title + row，for it in .indices 渲染
  name/level/note 三行 mini 卡）；③搜索结果按钮 label 改
  `h.name + " · " + h.sub`（F-001）；④自定义 pill 行内并列 ✕ 小按钮
  `onclick: .RemoveCustom(c.id)`。
- handler `.RemoveCustom(id)`：cities_remove → 用响应重建 custom_* 平行
  数组与 pills（同 AddHit 模式）。

### 5.3 测试（tests/vm_smoke.py）

- 新增 T8（在 T7 之后，青岛已选中）：① 快照含"穿衣"且 state indices 非空；
  ② state pm25/pm10/o3 在线非"—"；③ 搜索"青岛"→ 快照含副标题"山东"
  （F-001）；④ 添加"大连"→ 点击其 ✕ → state custom_names 不再含大连。

### 指数推导规则（验收锚点）

| 指数 | 规则 |
|---|---|
| 穿衣 | 体感 <0 羽绒服 / <10 厚外套 / <18 薄外套 / <24 长袖衬衫 / <30 短袖 / else 夏装 |
| 洗车 | 当前降水>0 或 未来12h 有雨码 → 不宜；else 适宜 |
| 运动 | 5–32°C 且 AQI≤150 且 无降水 → 适宜（10–28 且 AQI≤100 → 佳）；else 不宜 |
| 感冒 | 今日温差 ≥10 或 tmin ≤5 → 较易发；else 少发 |
| 紫外线 | uv_level 复用 + 原值注记 |

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P0-03/F-P0-04 标 PLAN-002 ✅；F-P1-05 改"删除达成（002）/排序归 PLAN-006" | 账实对齐 | AC-06 |
| SD-02 | modify | README.md | 数据节补指数/空气详情/删除说明 | 用户可见能力 | AC-06 |

## 6. 测试设计

- T8（新）见 §5.3；既有 T1–T7 不回归。AQI 通道在线依赖与 T6 同口径。

## 7. 验收标准

| ID | 标准 | 验证 | 结果 |
|---|---|---|---|
| AC-01 | report/report_at 返回 5 项指数，横屏渲染"穿衣"等 | T8 ①（快照含穿衣+indices 5×vmref） | **pass**（26/26） |
| AC-02 | 在线时 pm25/pm10/o3 非"—"且 UI 渲染 | T8 ②（o3 36/pm10 22/pm25 21 实测） | **pass** |
| AC-03 | cities_remove 生效并持久化；✕ 后 pill 消失 | T8 ④（删大连留青岛，末位 ✕ 定向） | **pass** |
| AC-04 | 搜索结果含"名称 · 副标题"（快照见"山东"） | T8 ③ | **pass** |
| AC-05 | vm_smoke 全绿（21 旧 + T8 新 5 项 = 26） | python tests/vm_smoke.py ×2 全绿 | **pass** |
| AC-06 | SD-01/02 落地 | commit 8ee25b1 文件核查 | **pass** |

## 8. 执行步骤

- [x] T-01 后端：ReportOut 扩展 + AQI 组分 + 指数推导（AC-01/02）
  - 验证：T8 ①②；修复记录：块级 let 不跨 try（UndefinedVariable×5）→
    推导用原始值提升函数级 var
- [x] T-02 后端：cities_remove（AC-03 前置）
  - 验证：T8 ④
- [x] T-03 前端：PM 行 + 生活指数区 + 搜索副标题（AC-01/02/04）
  - 验证：T8 ①②③；调整：视图按钮 label 不支持 paren-expr → handler 预拼
    `label` 字段（h.label）；按钮块父子污染 → find_smallest_clickable
- [x] T-04 前端：pill ✕ + RemoveCustom（AC-03）
  - 验证：T8 ④
- [x] T-05 tests：T8 + 全量回归（AC-05）
  - 验证：vm_smoke 26/26 ×2（w2-smokeB/C）；T8 含横屏回切（T4 留竖屏）
- [x] T-06 docs：SD-01/02（AC-06）
  - 验证：commit 8ee25b1 文件核查

依赖：T-01→T-03；T-02→T-04；T-05/06 最后。

## 9. 复审记录

- `stage: new | PLAN-002 | rev 1 | outcome: pass | next: work`——授权内
  （"继续计划002"沿用 new+work 授权），任务覆盖 AC-01..06 与 SD-01/02。
- `stage: work | PLAN-002 | rev 1 | outcome: pass | code_commit: 8ee25b1
  (on plan-002-dev, base a147755) | task_ids: T-01..T-06 | evidence:
  worktree vm_smoke 26/26 ×2（w2-smokeB/C 全绿）| blockers: 无 |
  next: review`——所有任务与 AC 映射已核销，变更已提交，worktree 保留待复审。
- **实施调整记录**（均在授权范围内，已随任务核销）：
  A1 块级 let 不跨 try 块——推导原始值提升函数级 var（UndefinedVariable×5
  实证）；A2 视图按钮 label 的 paren-expr 解析失败（20×RBrace 级联）——
  改 handler 预拼 label 字段；A3 **for-in 循环变量数字字段读出面中毒**
  （`as int` 饱和 21474836.47 / `""+` 垃圾 926338546 双实证；字符串字段
  与同对象点访问正常）——search 坐标改走 raw 文本 key 扫描
  （scan_numlist，纯字符串面）。该发现已写回 api.at 头注，属 design-v2
  §6 配方清单第 8 条，后续 PLAN 直接遵守；A4 快照父子块污染——
  find_smallest_clickable + 末位 ✕ 选择；A5 测试期 cities.json 污染
  数据按运行时状态清理（非仓库数据）。
- `stage: review | PLAN-002 | rev 1 | outcome: pass |
  reviewed_commit: 8ee25b1ed51f4c11245f38b61dff4fd2f4bcf458 |
  base_commit: a14775591150d36cfbdc4549a032142719da1f88 |
  dependency_revisions: auto-lang 168b56923（仅验证工具链，无源码改动）|
  spec_inputs: docs/specs/weather-app.md（worktree 含 SD-01）、README.md
  （worktree 含 SD-02）——增量已核：文本描述当前行为与持久决策，未发布 |
  acceptance_results: AC-01 pass（T8 ① 快照含穿衣 + indices 5×vmref）/
  AC-02 pass（T8 ② o3 36/pm10 22/pm25 21 实测）/ AC-03 pass（T8 ④ 删大连
  留青岛，末位 ✕ 定向 + 响应重建持久化）/ AC-04 pass（T8 ③ 快照含山东）/
  AC-05 pass（复审复现 worktree vm_smoke 26/26 exit 0）/ AC-06 pass
  （SD-01/02 文件核查，commit 8ee25b1）|
  findings: F-001 info——指数 note 为通用文案（"未来12小时降水"），未随
  数据个性化，非 AC 约束；F-002 info——PM 行/指数区仅横屏（口径内，
  竖屏归 PLAN-004）；F-003 info——scan_numlist 依赖 geocoding 紧凑 JSON
  形态（`"latitude":` 后紧跟数值逗号），有守卫降级（失配 → "0.00"），
  换 provider 时需重估。均无阻塞 |
  evidence: 复审复现 /tmp/review2-smoke.out（26/26）；worktree 零脏文件；
  worktree 定位 git worktree list 实证 | next: merge`。复审局限声明：
  复审与实施同会话，结论全部经工件与复现命令重建。

## 10. 待澄清事项

- 无（排序归 PLAN-006；竖屏新区块归 PLAN-004；指数详情页未排期）。
