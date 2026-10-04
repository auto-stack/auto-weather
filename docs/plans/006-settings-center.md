---
plan_id: PLAN-006
status: reviewed
feature_name: 设置中心（单位/主题/默认城市/城市管理排序）
author: [agent]
created_at: 2026-10-04T13:00:00Z
updated_at: 2026-10-04T13:00:00Z
plan_revision: 1
current_step: 5
total_steps: 5
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P1-04, docs/specs/weather-app.md#F-P1-05]
touched_goals: [F-P1-04, F-P1-05]
---

# PLAN-006 设置中心（单位/主题/默认城市/城市管理排序）

> 编号说明：文件取 006 对齐 roadmap 与用户口径；005 预留给 QWeather
> provider（受 JWT 阻塞未立项）。编号非 max+1 的偏离在此记录。

## 0. 变更摘要

新增设置中心（齿轮按钮展开设置卡，横竖屏共享）：温度单位 °C/°F、风速
单位 km/h/m/s、默认城市（点击轮选内置城）、城市管理（自定义城市
↑↓ 排序 + ✕ 删除收纳）。设置经后端 `settings.json` 持久化
（local_data_dir），Init 先取设置再按默认城市+单位拉取报文；单位作为
report/report_at 的查询参数贯穿全部 6 个取数点。主题沿用既有
ToggleTheme（设置卡内同步入口）。语言（i18n）为跨切面工程，明确拆出
本计划另立（登记 spec 偏差）。

## 1. 目标

- **Goal**：单位/默认城市/排序设置持久化并即时生效；设置入口对双布局
  可用。
- **Non-goals**：语言切换（i18n 拆独立计划）；accent 色板 UI（SetAccent
  已有 handler 无 UI，维持）；自定义城市设为默认（default 仅内置城）。
- 受影响模块：`src/back/api.at`、`src/front/app.at`、`tests/vm_smoke.py`、
  README、`docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-06 全过，vm_smoke 全绿。

## 2. 架构方案

设置面 = 后端 JSON（settings_get/settings_set 整对象替换，契约
`SettingsOut{ok,error,temp_unit,wind_unit,default_city}`）；单位下发 =
report/report_at 增 `ut/uw` 查询参数（后端 temp_s/风速拼装点分支）；
排序 = `POST /api/weather/cities_move {id,dir}` 读-交换-重写。城市/设置
双 JSON 同目录。Init 顺序：settings_get → 应用模型 → weather_report
（default_city + 单位）。

## 3. 技术栈

AutoLang .at、`#[api]`/VM merged、local_data_dir JSON、vm_smoke MCP。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04"考虑下一个计划 006"）：沿用 new+work 两段
  授权；review/merge 未授权。
- 前置：PLAN-001..004 已归档（751bc3/dbfff44/ba88d40/f7ed637）。
- 关键事实：温度格式化单点 `temp_s`（全温度出口经此）；风速拼装在
  impl_report_ll 单点；Init 硬编码 beijing（需改默认城市消费）；select
  组件 VM 不渲染（os-config V4）→ 默认城市用轮选按钮而非下拉。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- `pub type SettingsOut = { ok: bool, error: str, temp_unit: str,
  wind_unit: str, default_city: str }`；`impl_settings_get/set`（settings.json
  缺省 `{"temp_unit":"c","wind_unit":"kmh","default_city":"beijing"}`；
  set=读-合-写整对象）。
- `weather_report(city str, ut str, uw str)` / `weather_report_at(lat, lon,
  ut, uw)`：#[api] 增参（GET 取 query；旧调用方=前端全量更新）。
- 单位分支：`temp_s` 加 ut 参数（f → (v*9/5+32)  fmt + "°F"；fmt_i 复合
  算术经 let 中转）；风速 `uw=="ms"` → `(spd/3.6)` fmt + " m/s"。
  hourly/daily 温度同走 temp_s ✓ 单点收口。
- `impl_cities_move(id str, dir str) CityListOut`：读表-定位-与相邻交换
  （up=前邻，down=后邻）-重写-返回。
- 端点：`GET /api/weather/settings`、`POST /api/weather/settings`、
  `POST /api/weather/cities_move`。

### 5.2 前端（src/front/app.at）

- model 增：`s_temp_unit/s_wind_unit/s_default_city str`（默认 c/kmh/
  beijing）、`settings_open bool=false`。
- 所有 report 调用点（Init/SelectCity/SelectCustom/Refresh 双分支）传
  `.s_temp_unit/.s_wind_unit`。
- Init：先 `settings_get()` → ok 则三设置入模型（失败保默认）→ 默认城市
  替换原硬编码 beijing（city_id + city_zh + mock 链 id）。
- 视图：header 齿轮按钮 `⚙️` → `.ToggleSettings`；`if .settings_open`
  设置卡（app_frame 内、布局分支前 → 双布局共享）：温度单位两 pill、
  风速两 pill、默认城市轮选按钮（label "默认城市：${.s_default_city}"→
  onclick 循环内置 10 城）、城市管理（for custom_pills：↑ ↓ ✕ 三钮，
  onclick MoveCustom/RemoveCustom）。
- handler：`SetTempUnit(u)/SetWindUnit(u)` → 模型 + settings_set 持久化；
  `CycleDefaultCity` → 内置表循环 + 持久化；`MoveCustom(id,dir)` →
  cities_move → 重建 pills；`ToggleSettings`。

### 5.3 测试（tests/vm_smoke.py）

- 新增 T11：① 设置往返（切 °F + m/s → state s_temp_unit=="f"、
  s_wind_unit=="ms"）；② 单位生效（刷新 → 快照含 "°F" 与 " m/s"）；
  ③ 排序：添加大连 → ↑ 移至青岛前（custom_names 中大连位置 < 青岛）；
  ④ 默认城市轮选（label 变化 + state s_default_city 变更持久）。
  既有 T1–T10 不回归（T1 初始城市断言受默认城市影响——保持默认 beijing
  不被 T11 前段修改；T11 ④ 的轮选先设置再断言，不干扰 T1 已完成的断言）。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P1-04 标"单位/主题/默认城市达成（006）；语言拆独立 i18n 计划"；F-P1-05 标"排序达成（006）" | 账实对齐 | AC-06 |
| SD-02 | modify | README.md | 数据节补设置中心说明 | 用户可见能力 | AC-06 |

## 6. 测试设计

- T11（新）见 §5.3；设置持久化跨重启依赖与 cities 同口径（本进程内断言
  + 文件重写证据）。

## 7. 验收标准

| ID | 标准 | 验证 | 结果 |
|---|---|---|---|
| AC-01 | 设置面持久化（settings.json 读写往返） | T11 ①（单位 f/ms 入 state）+ ④（轮选持久） | **pass**（40/40 ×2） |
| AC-02 | °F/m/s 单位端到端生效（快照见证） | T11 ②（刷新后快照含 °F/m/s） | **pass** |
| AC-03 | 自定义城市 ↑↓ 排序生效（顺序断言） | T11 ③（大连先于青岛） | **pass** |
| AC-04 | 默认城市设置项存在且 Init 消费（源码锚定 + 轮选断言） | T11 ④ + Init settings_get 锚定 | **pass** |
| AC-05 | vm_smoke 全绿（34 旧 + T11 新 6 项 = 40）×2 | python tests/vm_smoke.py ×2 | **pass** |
| AC-06 | SD-01/02 落地 | 文件核查（commit ecdbbcc） | **pass** |

## 8. 执行步骤

- [x] T-01 后端：SettingsOut + get/set + 单位参数分支（AC-01/02/04 前置）
  - 验证：T11 ①②；单位经 temp_s 单点（ut）与风速拼装（uw）收口
- [x] T-02 后端：cities_move（AC-03 前置）
  - 验证：T11 ③；三轮修复见 §9 实施调整（.set/整体赋值的退化实证）
- [x] T-03 前端：设置卡 + 齿轮 + 6 调用点传参 + Init 默认城市（AC-01..04）
  - 验证：T11 全项；s_default_label 唯一前缀消歧；发现并修复 PLAN-004
    竖屏双渲染（banner/搜索/pills 本属共享区，portrait 副本删除）
- [x] T-04 tests：T11 + 全量回归（AC-05）
  - 验证：vm_smoke 40/40 ×2（w6-smoke9/A）；启动前运行时状态卫生
    （settings/cities JSON 重置——跨运行 hermetic，消除 T1 beijing
    缺省与遗留设置污染的跨运行冲突）
- `stage: review | PLAN-006 | rev 1 | outcome: pass |
  reviewed_commit: ecdbbccf488bc6e63b25f19a49cc6b04ab27bedf |
  base_commit: 94fcbf7（docs(plan): PLAN-006 contract）|
  dependency_revisions: auto-lang 168b56923（仅验证工具链）|
  spec_inputs: docs/specs/weather-app.md（worktree 含 SD-01）、README.md
  （worktree 含 SD-02）——增量已核：描述当前行为与持久决策，未发布 |
  acceptance_results: AC-01 pass（T11 单位 f/ms 入 state + 轮选持久）/
  AC-02 pass（T11 ② 刷新后快照含 °F/m/s）/ AC-03 pass（T11 ③ 大连先于
  青岛）/ AC-04 pass（T11 ④ 轮选 + Init settings_get 消费锚定）/
  AC-05 pass（复审复现 worktree vm_smoke 40/40 exit 0）/ AC-06 pass
  （SD-01/02 文件核查，commit ecdbbcc）|
  findings: F-001 info——默认城市仅内置城（自定义为默认在 non-goals 明确）；
  F-002 info——i18n 拆独立计划（SD-01 登记）；F-003 info——A6 偶发进程
  退出 1 例（无 panic 日志，重跑即绿，环境/工具链偶发）；F-004 info——
  设置卡内联渲染于布局分支上方（popover VM 脆弱，内联为正确取舍）。
  均无阻塞 |
  evidence: 复审复现 /tmp/review6-smoke.out（40/40）；worktree 零脏文件 |
  next: merge`。复审局限声明：复审与实施同会话，结论全部经工件与
  复现命令重建。
- [x] T-05 docs：SD-01/02（AC-06）
  - 验证：commit ecdbbcc 文件核查

依赖：T-01/T-02 → T-03；T-04/05 最后。

## 9. 复审记录

- `stage: new | PLAN-006 | rev 1 | outcome: pass | next: work`——授权内
  （"考虑下一个计划 006"沿用 new+work）；编号取 006 对齐 roadmap（005
  预留 QWeather），编号非 max+1 的偏离已记录。
- `stage: work | PLAN-006 | rev 1 | outcome: pass | code_commit: ecdbbcc
  (on plan-006-dev, base 94fcbf7) | task_ids: T-01..T-05 | evidence:
  worktree vm_smoke 40/40 ×2（w6-smoke9/A 全绿）| blockers: 无 |
  next: review`——所有任务与 AC 映射已核销，变更已提交，worktree 保留待复审。
- **实施调整记录**：
  A1 **list .set 与"本地方列表整体赋给响应字段"在 VM 边界退化为数值
  句柄**（[1360,1325] 与 [1361,1360] 双实证）——响应列表一律 push-only
  重建，重排实现改为"新列表按交换序 push + 写文件 + impl_cities_get
  出响应"；已写入 api.at 头注（design-v2 §6 第 9 条）。
  A2 设置卡按钮与 pill 条同名城市的快照定位冲突 → s_default_label
  唯一前缀（"默认:xx"）handler 预拼（视图 paren-expr 受限的既定配方）。
  A3 T11 ④对当前默认城市鲁棒 + 还原循环带状态验证（上轮崩溃遗留
  shanghai 的污染实证）。
  A4 测试启动前重置运行时 JSON（settings/cities）——跨运行 hermetic；
  持久化正确性由 run 内断言覆盖（跨重启持久化已在 PLAN-001 时代实证）。
  A5 发现并修复 PLAN-004 遗留缺陷：共享区（banner/搜索/pills）在
  portrait 分支被重复渲染（T10 存在性断言未暴露）；本次删除 portrait
  副本，PLAN-004 的"竖屏对齐"语义不受影响（共享区本就双端可见）。
  A6 偶发环境故障一记：w6-smoke4 运行中 app 进程无 panic 退出
  （前后文无异常日志），重跑即绿——登记为环境/工具链偶发，非代码缺陷。

## 10. 待澄清事项

- 语言/i18n 拆独立计划（跨切面：视图全部文案 + 后端映射表双语化）；
  自定义城市设为默认、accent 色板 UI 未排期。
