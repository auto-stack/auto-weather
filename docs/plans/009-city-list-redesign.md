---
plan_id: PLAN-009
status: executing
feature_name: 城市体验重设计（主城市单一列表 + 搜索添加 + 排序 + 持久化 + 启动偏好）
author: [agent]
created_at: 2026-10-05T12:00:00Z
updated_at: 2026-10-06T00:00:00Z
plan_revision: 2
current_step: 0
total_steps: 5
supersedes_spec_components: []
new_spec_components: []
touched_goals: [F-P0-01, F-P1-04, F-P1-05]
---

# PLAN-009 城市体验重设计（主城市单一列表 + 搜索添加 + 排序 + 持久化 + 启动偏好）

> UI/UX 重规划：从「10 内置城市 pill + 搜索附加自定义 pill」改为「单一有序城市
> 列表，首位=主城市（缺省北京种子），其余全部搜索添加，↑↓ 可排序，整表持久化；
> 启动落点可由用户设定为主城市或上次选中城市」。
> 基线 main @ 8e075d5（工作区干净）。
> rev 2（2026-10-06 用户决策）：新增启动偏好——settings 契约 3 字段改为 5 字段
> （+`startup`/`last_id`），设置卡加切换行；rev 1 的「启动永远落主城市」放宽为
> 默认主城市、可选上次选中。

## 0. 变更摘要

- **前端**：删除 10 组建内置城市 pill 的 if/else 展开与 `t_city_*` ×10 中英文
  变量（3 个赋值块）；pill 条改为渲染唯一持久化列表 `cities_pills`，首位 pill
  带 ★ 主城市徽记，全 pill 补上选中高亮（现自定义 pill 无 on 态，半遗留）。
  选中态模型从 `city_id`(内置 ASCII id) / `active_custom`(自定义 id) 双字段
  收敛为单一 `cur_id/cur_name/cur_lat/cur_lon`，所有城市（含主城市）统一走
  `weather_report_at` 通道，`SelectCity`/`SelectCustom` 双 handler 合一。
  设置卡新增「启动：主城市/上次选中」轮选行（`s_startup`），每次选中城市
  同步 `s_last_id` 并随 settings 整对象落盘。
- **后端**：`cities.json` 语义从「自定义城市附加表」升格为「唯一有序城市表，
  位置 0 = 主城市」；文件缺失/损坏时读路径合成北京种子
  (`39.90_116.40/北京`)，首次写操作落盘。settings 契约删 `default_city`
  （`coords_for` 内置城白名单校验随之删除），增 `startup`(main|last，缺省
  main，白名单校验) 与 `last_id`(自由 str，缺省空)——合计 5 字段。
- **测试**：T1/T2/T3/T11 按新交互重写；新增 T15（冷启动种子、搜索添加、
  排序持久化、启动偏好、last_id 落盘、空表边界）。
- **持久化选型**：维持 JSON 文件（`local_data_dir/auto-weather/*.json`）。
  SQLite 全仓零先例（.at/Cargo/docs grep 无命中，VM 无 sqlite 面），JSON
  配方已双轨实证，不换存储引擎。

## 1. 目标

- **Goal**：首屏 pill 条只含主城市（全新数据目录 → 仅「★ 北京」）；其他城市
  一律经搜索添加；列表可 ↑↓ 重新排序；整表跨启动持久保留；启动落点用户可
  设定——默认「主城市」（列表首位），可切换为「上次选中城市」（跨启动记忆，
  该城市被删则回落主城市）。
- **Non-goals**：定位能力本体（仅预留主城市=首位语义，未来定位城市插入/上移
  至位置 0 即接入）；SQLite 迁移（决策见 §0）；`weather_report`(内置 id 入参)
  端点下线（前端停用但端点保留，避免 Vue 轨桩面波动）；天气详情/指数/降水等
  非城市区块 UI 不动；账号/云同步（spec §4 非目标不变）。
- 受影响模块：`src/back/api.at`、`src/front/app.at`、`tests/vm_smoke.py`、
  `docs/specs/weather-app.md`、`docs/design/weather-v2.md`、README。
- 成功标志：AC-01..AC-08 全过。

## 2. 架构方案

```
cities.json（唯一有序城市表，位置0=主城市，缺省种子=北京）
   ▲ read-or-seed        │ 写：add（去重追加）/ remove（过滤重写）/ move（邻位交换）
   │  GET /api/weather/cities
settings.json { temp_unit, wind_unit, lang, startup: main|last, last_id }
   ▲ 整对象替换          │ 每次选中城市 → s_last_id 更新随写
front Init：settings_get(5字段) → cities_get → 物化 pills + main_id
   → 空表？空态 : 选启动城市（startup=last 且 last_id 在表 → 该城，否则首位）
   → report_at(lat,lon) 统一取数（不再走 weather_report 内置 id 通道）
```

- **主城市定义**：不做独立字段——主城市 ≡ `cities.json` 位置 0，pill 以
  `c.id == .main_id`（每次列表重灌时镜像首位 id）打 ★。排序至顶即换主城市，
  语义直给且复用现有 `cities_move` 端点；未来定位：定位城市插入/移动至位置 0。
- **启动偏好**：`startup` 存用户选择；`last_id` 存最近一次选中（前端在一切
  `cur_*` 变更点同步：pill 点选/添加自动选中/删除回落）。Init 只做一次表内
  查找：命中用 `last_id`，落空（空串/城市已删）回落首位。校验在前端落点处，
  后端 `last_id` 仅 verbatim 存储。
- **种子策略**：后端读路径合成（文件缺失/解析失败 → 返回 `[北京]`；文件存在
  且 `cities` 为空数组 → 返回空，即「删光」是合法持久态），首次写操作以合成
  表为基落盘。前端零特判；空表时 UI 给空态提示。
- **settings 契约**：5 字段（temp_unit/wind_unit/lang/startup/last_id），
  整对象替换。旧 4 字段文件（含 default_city）兼容：`impl_settings_get` 按名
  取值，缺项回缺省（startup=main、last_id=空），多余字段自然忽略；
  `impl_settings_set` 改发 5 字段 JSON。前后端 `settings_get/set` 实参同步
  4→5（typed 调用，编译面强制对齐）。
- **VM 约束遵守**（设计文档 §6，违反即回归）：契约仍扁平标量+平行数组；
  `#[api]` 体 thin；`cities_move` 继续经 `impl_cities_get` push-only 通道出
  响应（数组句柄退化实证 api.at:1060-1063 注记）；前端 handler 保持薄。

## 3. 技术栈

AutoLang .at widget（VM merged 主面 / Vue 轨桩后端随 api.at 契约自动再生成）、
`tests/vm_smoke.py`（vm_smoke MCP；autoui_screenshot 备用）。

## 4. 需求分析与背景调查

- **授权**：用户 2026-10-05「请重新规划一下 app 的 UI/UX……」授权新建计划
  （new 段）；2026-10-06 用户对 rev 1 待澄清 #1 拍板：「让用户设定是『用主
  城市』还是『上次选中城市』」——即本次 rev 2 修订授权。执行（work）、复审、
  合并未授权，需用户另行指示。
- **需求原文拆解**：① 一上来只显示主城市，默认北京，未来定位后默认定位
  城市 → 种子+首位语义+定位钩子；② 其他城市都通过搜索添加 → 现有
  `DoSearch`/`AddHit`/`weather_search` 链路保留，pill 条数据源换为唯一列表；
  ③ 列表可重新排序 → 现有设置卡 ↑↓/`cities_move` 保留并补 pill 选中高亮；
  ④ 列表记下来（本地文件或 SQLite 皆可）→ `cities.json` 已是有序持久化，
  选型维持 JSON（§0 决策）；⑤ 启动落点可配（rev 2）→ `startup`+`last_id`。
- **现状关键事实**（行号为基线 8e075d5）：
  - 10 内置 pill 字面量展开 app.at:332-381；`t_city_*` ×10 三赋值块
    (:178-187, :1022-1031, :1405-1414)；中文名 if 链 :237-269、:1460-1480。
  - 双通道选中：`city_id`/`active_custom`（F-002：`active_custom` 非空时
    `city_id=""`，SelectCustom :1214）；Refresh 双分流 :1667-1714。
  - 默认城市轮选：`s_default_city`/`s_default_label`(:139-141)、设置卡行
    :424-428、`CycleDefaultCity` :1301-1325、Init 消费 :1037-1058；
    `settings_set` 四参整对象替换 (:1293 等 4 处)。
  - 后端：`coords_for` 10 城白名单 api.at:127-140；`impl_settings_set` 用其
    校验 default_city (:1011)；`cities_*` 端点 :789-941；`cities_move` 邻位
    交换 :1031-1123；`weather_search` :727-786。
  - 演示样本兜底：Init 重置样本后真实取数，失败静默保留（`live` 守门）；
    `SelectCity` 中文键演示分支 :1481-1611 为死代码（实参恒 ASCII id）。
  - 测试：T3 点上海 pill、T11 默认城市 10 城轮选表 (:783-820)、T1/T2
    `city_id=beijing` 断言——均需重写；T7/T8/T10 搜索/删除/排序断言可保留。
  - 数据目录实证：Windows `%APPDATA%\auto-weather\`（vm_smoke 卫生删除
    :262-269）；SQLite 全仓零先例。
- **风险锚点**：Init 起屏真实 HTTP 首绘 10s+（测试首绘重试 ≥30 次，设计文档
  §6.6）——Init 内新增 `cities_get` 串行调用注意重试预算；删除城市若删的是
  当前选中 → 须回落首位（否则 header/取数悬空）；pill ★ 依赖 `main_id` 镜像
  时机，重灌后立即刷新；`last_id` 持久化使每次 pill 点选多一次本地文件写
  （可接受，JSON 小文件）。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- **种子**：新增 `fn seed_cities() -> str`（JSON 文本常量，单条北京
  `{"id":"39.90_116.40","name":"北京","lat":"39.90","lon":"116.40"}`）；
  `fn cities_read_or_seed() -> str`：文件缺失/解析失败 → 返回种子文本；文件
  存在且 `cities` 为空数组 → 返回文件文本（空表合法）。
- `impl_cities_get`(:798-833)：改经 `cities_read_or_seed()` 出表。
- `impl_cities_add`(:836-889)：基表改 `cities_read_or_seed()`；判重、追加、
  `quote_json` 重写落盘不变。
- `impl_cities_remove`(:892-941)：确认空表写 `{"cities":[]}` 路径可用；返回
  新表。
- `impl_cities_move`(:1031-1123)：逻辑不变（邻位交换、边界不动、push-only
  出响应），确认空表/单元素不越界。
- **settings 改 5 字段**：`SettingsOut`/`impl_settings_get` 删 `default_city`，
  增 `startup`（缺省 `"main"`）与 `last_id`（缺省 `""`），按名取值兼容旧文件；
  `impl_settings_set` 删 `coords_for` 校验 (:1011)，新增 `startup ∈
  {main,last}` 白名单校验（非法值回退 main 或返回 ok:false——沿用 temp_unit
  既有校验风格），手写 JSON 改 5 字段。`coords_for` 表本身保留（
  `weather_report` 内置 id 端点仍注册，前端停用）。
- 端点形状不变：`GET/POST/DELETE /api/weather/cities`、`POST cities_move`
  入出参契约不动（Vue 轨桩面零波动）。

### 5.2 前端（src/front/app.at）

- **model 变更**：
  - 删除：`city_id`、`city_zh`、`active_custom`、`s_default_city`、
    `s_default_label`、`t_city_beijing…t_city_sanya`（×10）。
  - 新增：`cur_id/cur_name/cur_lat/cur_lon str`（当前城市，取数唯一依据）、
    `main_id str`（= 列表首位 id，★ 判定）、`cities_empty bool`（空态门控）、
    `s_startup str = "main"`、`s_last_id str = ""`、`s_startup_label str`
    （handler 预拼，模式同 rev 1 前的 `s_default_label`）。
  - `custom_ids/custom_names/custom_lats/custom_lons` + `custom_pills` 更名
    为 `city_ids/city_names/city_lats/city_lons` + `cities_pills`（单列表）。
- **Init 重排**（:979-1137）：settings_get(5 字段) → `cities_get()` 物化
  `cities_pills` + `main_id` → 空表则 `cities_empty=true`、跳取数（保演示
  样本+注记）；非空则**选启动城市**：`startup=="last"` 时扫描 `city_ids` 找
  `s_last_id`，命中用该城、否则回落首位；存 `cur_*` → 重置演示样本 →
  `weather_report_at(.cur_lat,.cur_lon,…)` → `.booting=false`。
- **选中即记忆**：新增 `fn pick_city(i)`（索引 → 存 `cur_*` + `s_last_id =
  city_ids[i]` + `settings_set` 5 参落盘）；`SelectCity`/`AddHit` 自动选中/
  `RemoveCity` 删除回落——凡 `cur_*` 变更一律经 `pick_city`，保证 `last_id`
  账实一致。
- **pill 条**（:324-389 共享顶部区）：10 组内置 if/else 整段删除；改为
  `for c in .cities_pills { button → .SelectCity(c.id) }`，样式
  `if c.id == .cur_id { city_pill_on } else { city_pill_off }`，名首拼接：
  `if c.id == .main_id { "★ " + c.name } else { c.name }`；✕ 保留
  （`.RemoveCity(c.id)`）；空态 `if .cities_empty` 渲染 hint（新 t_key
  `t_cities_empty`，zh「搜索添加城市」/en「Search to add a city」）。
- **handler 合一**：`SelectCity(id)`（新语义=原 SelectCustom）：定位 →
  `pick_city`；删除 `SelectCustom` 与 `SelectCity` 旧内置分支（含 :1481-1611
  死代码）。`RemoveCity`（=原 RemoveCustom）：remove → 重灌 → 同步 `main_id`；
  若删的是 `cur_id` → `pick_city(0)`（空表置 `cities_empty` 跳取数并清
  `s_last_id` 为 ""）。`MoveCity`（=原 MoveCustom）：move → 重灌 → 同步
  `main_id`；当前城市不因移动而改选。`AddHit`：add → 重灌 → 若此前空表则
  `pick_city(0)`。`Refresh`：单通道 `report_at(.cur_lat,.cur_lon,…)`。
  删 `CycleDefaultCity`；新增 `CycleStartup`：main↔last 互切 → 重拼
  `s_startup_label` → `settings_set` 落盘。
- **设置卡**（:392-455）：删「默认:`<id>`」轮选行；新增「启动:`<主城市|上次
  选中>`」行（`button .s_startup_label → .CycleStartup`）；城市管理行保留
  （↑↓✕ 全列表）；设置持久化调用 4→5 实参（5 处 handler：温度/风速/语言/
  启动/选中记忆）。
- **i18n**：删 `t_city_*` 三赋值块与 header 城市名 if 链（:237-269），
  header 城市名绑 `.cur_name`；新增 `t_cities_empty`、
  `t_startup_main`（zh「启动:主城市」/en「Startup: Main city」）、
  `t_startup_last`（zh「启动:上次选中」/en「Startup: Last selected」），
  zh/en/SetLang 三处赋值照 :1405-1414 模式。
- **演示样本**：保留现有失败兜底语义（取数失败不清 `live` 注记）；样本内容
  仍为北京形态，与主城市种子一致，无需改造。

### 5.3 测试（tests/vm_smoke.py）

- **重写 T3**：改为「搜索 上海 → AddHit → pills 含上海 → 点上海 pill →
  `cur_name=上海`、`s_last_id` 为上海 id，且取真实数据」。
- **重写 T1/T2 断言**：`city_id=beijing` → `cur_id=39.90_116.40` +
  `cur_name=北京`；地标快照「★ 北京」；默认 `s_startup=main`。
- **重写 T11**：删 10 城轮选段（:783-820）与「还原 beijing」尾巴；保留单位
  持久化与 ↑↓ 排序断言（大连/青岛序）；新增 `main_id` 断言（首位不变除非
  移至顶）。
- **新增 T15**（城市列表端到端）：① 卫生删除 json 后冷启动 → 快照恰 1 个
  pill「★ 北京」、`s_startup=main`；② 搜索青岛添加 → 2 pills、青岛选中
  高亮、settings.json `last_id`=青岛 id；③ 青岛点 ↑ → `main_id` 变青岛；
  断言 cities.json 文本顺序=[青岛,北京]；④ 设置卡点「启动：主城市」切到
  「上次选中」→ settings.json `startup=last`；⑤ 点北京 pill → `last_id`
  变北京 id；⑥（vm_smoke 支持进程内重启则重启）→ 启动落北京（startup=last
  实证）；不支持则以 settings.json 落盘 + Init 选路代码在码为证；⑦ 删北京
  （当前选中+last_id）→ 回落青岛（首位）、`last_id` 变青岛 id；⑧ 删光全部
  → 空态 hint、跳取数；再添加上海 → 自动选中、`last_id`=上海。
- 回归：T4-T10、T12、T13 全绿（T7/T8/T10 的搜索/删除断言基于新 pill 渲染
  仍成立，元素定位文案注意 ★ 前缀）。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P0-01 行：由「10 内置+自定义 pill」改为「唯一有序城市表：首位=主城市（缺省北京种子），搜索添加，排序/删除达成（009）」 | 账实对齐 | AC-01/02/03 |
| SD-02 | modify | docs/specs/weather-app.md | F-P1-04 行：设置中心删「默认城市轮选」，新增「启动偏好：主城市/上次选中（startup+last_id 持久化）」（009） | 轮选 UI 换启动偏好 | AC-05/08 |
| SD-03 | modify | docs/specs/weather-app.md | F-P1-05 行：城市管理=单列表 ↑↓ 排序与删除（含主城市），不再区分内置/自定义（009） | 账实对齐 | AC-03/04 |
| SD-04 | modify | docs/design/weather-v2.md | §4 CityListOut 语义补「有序、位置 0=主城市、读路径种子」；§5 补空表与种子规则；settings 契约改 5 字段 | 契约知识沉淀 | AC-01/04/05 |
| SD-05 | modify | README.md | 数据节城市搜索/设置中心两行改写为新城市模型+启动偏好；顺带修正运行节「pac.at 声明 api:"rust"」陈旧表述 | 用户可见能力 | AC-07 |

## 6. 测试设计

- T15 见 §5.3；全量 46 旧项（T14 后计）+ T15 新 8 断言，×2 轮全绿。
- Init 新增一次 `cities_get` 串行调用：首绘重试预算 ≥30 次纪律不变，T15 ①
  放在既有首绘重试循环内复用。
- 持久化跨重启实证以 cities.json/settings.json 文件文本断言为主（卫生删除后
  全程可控）；进程内重启支持时补 T15 ⑥ 启动态断言，不支持则以文件断言 +
  Init 选路代码在码为证。

## 7. 验收标准

| ID | 标准 | 验证 |
|---|---|---|
| AC-01 | 全新数据目录首启：pill 条恰「★ 北京」一座，header 北京，自动取实时数据 | T15 ①；T1 快照 |
| AC-02 | 搜索添加城市进入 pill 条（尾部追加、去重）；点任一 pill 选中高亮并按其坐标取数 | T15 ②；T3 重写 |
| AC-03 | ↑↓ 排序即时生效并落盘，重启后顺序保留；首位=主城市带 ★ | T15 ③；T11 排序段 |
| AC-04 | 任一城市可删；删当前城市回落首位；删光→空态提示不取数；空表再添加自动选中 | T15 ⑦⑧；T8/T10 回归 |
| AC-05 | 设置卡无默认城市轮选行，有「启动:主城市/上次选中」行；settings.json 5 字段；旧 4 字段文件读取不报错 | T11/T15 ④；文件核查 |
| AC-06 | vm_smoke 全量绿（旧回归 + T15）×2 轮 | python tests/vm_smoke.py |
| AC-07 | SD-01..05 落地 | 文件核查 |
| AC-08 | startup=last 时启动落在 last_id 城市并跨重启保持；该城市被删/不在表 → 回落主城市；startup=main（默认）启动恒落主城市 | T15 ④⑤⑥⑦ |

## 8. 执行步骤

- [ ] T-01 后端：种子/read-or-seed、cities_add/remove/move 空表边界、settings
  改 5 字段（删 default_city 校验，增 startup 白名单 + last_id）（AC-01/03/04/05）
  - 验证：T15 相关断言过；既有 T7/T8/T10/T11 绿
- [ ] T-02 前端 model/handler：cur_*/main_id/cities_empty/s_startup/s_last_id，
  pick_city 选中记忆，SelectCity/RemoveCity/MoveCity/AddHit/Refresh 合一，
  CycleStartup，Init 重排选路（AC-01/02/04/05/08）
  - 验证：T2/T11/T15 断言过
- [ ] T-03 前端视图：pill 条单 for 渲染+★+高亮+✕、空态 hint、设置卡删轮选行
  加启动行、删 t_city_* ×10 与 header if 链、i18n 三处增 3 个 t_key（AC-01/02/03/05）
  - 验证：T1 快照地标过
- [ ] T-04 测试：重写 T1/T2/T3/T11、新增 T15、全量 ×2（AC-03/04/06/08）
  - 验证：vm_smoke 全绿 ×2
- [ ] T-05 文档：SD-01..05（AC-07）
  - 验证：commit 文件核查

依赖：T-01∥T-02∥T-03 可并行（契约已锁）；T-04 依赖前三；T-05 最后。

## 9. 复审记录

- `stage: new | PLAN-009 | rev 1 | outcome: pass | next: work`——授权内。
- `stage: new | PLAN-009 | rev 2 | outcome: pass | next: work`——用户
  2026-10-06 拍板启动偏好（主城市/上次选中可选）授权修订；变更面：settings
  契约 3→5 字段（+startup/last_id）、设置卡新增切换行（CycleStartup）、
  选中即记忆（pick_city 统一入口）、Init 选路、SD-02、新增 AC-08、T15 扩至
  8 断言。work/review/merge 仍未授权，待用户指示。

## 10. 待澄清事项

- 定位接入时的主城市策略细节（定位城市强插位置 0 vs 仅首次询问；与
  startup=last 的交互）——归未来定位计划。
- `weather_data.at` 孤儿文件与 `weather_report` 内置 id 端点的最终下线——
  不在本计划范围，登记备查。
