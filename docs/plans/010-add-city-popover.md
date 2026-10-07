---
plan_id: PLAN-010
status: reviewed
feature_name: 添加城市弹出面板（＋按钮入口，添加即选中）
author: [agent]
created_at: 2026-10-06T00:00:00Z
updated_at: 2026-10-07T00:00:00Z
plan_revision: 1
current_step: 4
total_steps: 4
supersedes_spec_components: []
new_spec_components: []
touched_goals: [F-P0-01]
---

# PLAN-010 添加城市弹出面板（＋按钮入口，添加即选中）

> 用户 2026-10-06 走查反馈：搜索逻辑改为"添加城市"逻辑——主页不再显示搜索
> 框，改为主页右上角 `＋` 图标按钮，点击弹出悬浮面板（搜索拼音框 + 搜索
> 按钮 + 结果列表），点结果后：popover 关闭、城市入列**并成为当前选中项**。
> 基线 main(v0.6-dev) @ b694892（PLAN-009 已交付）。

## 0. 变更摘要

- **前端**：删除共享顶部区的搜索输入行/错误行/结果行（现 app.at:285-299，
  横竖屏共享单份——portrait 分支无第二拷贝，:284 注释陈旧）；header 右侧
  图标排（布局切换/刷新/主题/⚙️）新增 `＋` 按钮（icon_btn 同款）；新增
  `add_open` 门控的**内联弹出面板**（header 行之后、pill 条之前，与设置卡
  同构的 if 块卡片）：标题「添加城市」+ ✕ 关闭行、输入框 + 搜索按钮行、
  错误行、结果 for 循环。`AddHit` 语义变更：**添加成功后始终选中新城市**
  （pick_city 语义：cur_* + s_last_id + settings 落盘 + 取数）、关闭面板、
  清空搜索区；添加失败（r2.ok=false）面板保持打开。
- **VM 悬浮约束适配**：字面"悬浮 popover"（absolute 定位浮动层）在 VM 轨
  不可用（PLAN-008 A1 实证「VM 无 absolute 遮罩」；os-config VG16 确认层
  同款普通 if 块）。本计划采用与设置卡同构的内联弹出面板，置于 header
  正下方、视觉上自 `＋` 按钮展开——双布局共享，交互语义（点击外部不收
  敛、✕/完成关闭、点结果即关闭）与用户需求逐项对齐；Vue 轨未来真 popover
  属另一计划。
- **面板互斥**：`ToggleAddCity` 打开时置 `settings_open=false`，
  `ToggleSettings` 打开时置 `add_open=false`——两面板不叠显。
- **后端小修（顺手清 PLAN-009 复审 R2）**：`impl_cities_add` 判重早退
  （返回部分表）改为"置重复标记 + 继续扫描 + 返回全表"——修复重复添加
  已有城市时前端内存列表瞬断的预存缺陷（文件从未损坏，纯内存态）。
- **i18n**：新增 `t_add_city`（zh「添加城市」/en「Add city」，四处赋值块）；
  空态 hint `t_cities_empty` 改「点 ＋ 添加城市」/「Tap + to add a city」
  （入口变了，hint 指路）。
- **测试**：T3/T7/T8/T10/T15 的搜索步骤改经"点 ＋ → 面板内搜索"；新增
  T16（面板开合/添加即选中/重复添加选中/互斥）。

## 1. 目标

- **Goal**：主页零搜索组件；右上角 `＋` 一键开合添加面板；面板内完成
  搜索→点结果→面板关闭、城市入列并选中取数；与设置卡互斥；双布局一致；
  重复添加不再截断内存列表。
- **Non-goals**：真 absolute 悬浮 popover（VM 轨不支持，见 §0）；搜索
  后端契约不变（`weather_search`/`cities_add` 形状不动，仅 add 判重返回
  语义修正）；键盘回车触发搜索（VM input 事件面无 keybinding 实证，不
  做）；选中态高亮/详情浮层等 PLAN-008 在途能力（不并入）。
- 受影响模块：`src/front/app.at`、`src/back/api.at`、`tests/vm_smoke.py`、
  README、`docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-07 全过。

## 2. 架构方案

```
header 行（右上）：布局切换 │ 刷新 │ 主题 │ ＋(ToggleAddCity) │ ⚙️(ToggleSettings)
        │ 点 ＋
        ▼
if .add_open 内联弹出面板（header 下、pill 条上；与设置卡同构卡片）
  ├ 标题行：「添加城市」            ✕(关闭+清空)
  ├ 输入框(oninput SearchChanged) + 搜索按钮(DoSearch)
  ├ 错误行(search_err)
  └ for h in .search_hits：button h.label → AddHit(h.id)
        │ 点结果
        ▼
AddHit 新语义：cities_add(判重返全表) → 重灌 → 定位 id 索引 → pick_city 语义
  （cur_*+s_last_id+settings_set 落盘）→ report_at 取数 → add_open=false、
  search_q/search_hits/search_err 清空；!r2.ok → 面板保持打开
```

- **选中逻辑**：添加后按 `id` 在新表内索引定位（重复添加时 id 已在表内，
  直接选中）——`was_empty` 特判删除（"始终选中"涵盖空表场景）。
- **互斥**：两 Toggle handler 互置对方门控 false；关闭面板即清搜索区，
  保证再次打开是干净态。
- **VM 配方约束**（设计文档 §6）：`#[api]` thin 不动；前端 handler 薄、
  索引循环物化；视图条件双节点展开；文案 handler 预拼。

## 3. 技术栈

AutoLang .at widget（VM merged 主面 / Vue 轨 api_gen 桩随契约再生成）、
`tests/vm_smoke.py`（vm_smoke MCP）。

## 4. 需求分析与背景调查

- **授权**：用户 2026-10-06 走查后口述需求 + 同日消息"这个功能可能需要一
  个新的计划文件跟踪"——授权新建计划（new 段）；work/review/merge 未授
  权，待用户指示。
- **现状关键事实**（基线 b694892 行号）：
  - 搜索块 app.at:285-299（共享顶部区，`input` 全仓唯一视图拷贝）；
    header 图标排 :253-281（`ml-auto` 右侧组，⚙️ 在 :280）。
  - `DoSearch` :1088-1115（结果物化 `search_hits`，`label=name · sub`）；
    `AddHit` :1117-1170（现语义：空表才 pick_city(0)，非空保持选中）；
    `ToggleSettings` :1346。
  - 后端 `impl_cities_add` 判重早退返回部分表（api.at 内，PLAN-009 复审
    R2 记录）；空表/首次添加的种子基表逻辑（cities_read_or_seed）不变。
  - 测试 T3/T7/T8/T10/T15 全部经主页搜索框走 DoSearch/AddHit；输入驱动
    方式沿用 harness 既有机制（vm_smoke MCP 对 input 元素的 action）。
- **风险锚点**：PLAN-008 A1「col onclick 不派发」→ 面板关闭用独立 ✕
  按钮（不绑容器点击）；A3「条件块按钮 aura 映射偶发延迟」→ 关键按钮
  （＋/✕/搜索）唯一文案 + 点击重试纪律；A2「scroll h-full 挤压尾部内联
  元素」→ 面板放 header 后、pill 条前的共享区，不进 scroll 容器。

## 5. 详细设计

### 5.1 前端（src/front/app.at）

- **删除**：:285-299 搜索输入行 + search_err 行 + search_hits 行（含
  :284 陈旧注释）。
- **model**：新增 `add_open bool = false`、`t_add_city str = "添加城市"`；
  `t_cities_empty` 默认值改「点 ＋ 添加城市」。
- **header 图标排**：`⚙️` 之前插 `button "＋" { onclick: .ToggleAddCity,
  style: icon_btn }`。
- **弹出面板**（header 行后、pill 条前，`if .add_open { col { 卡片样式同
  设置卡（w-full gap-2 bg-card border border-border rounded-2xl p-3）：
  ├ 行：text .t_add_city(section_title 同款) + button "✕"(onclick
  .ToggleAddCity，icon_btn 右对齐 ml-auto)
  ├ 行：input(placeholder .t_search_ph, value .search_q, oninput
  .SearchChanged，原输入样式) + button .t_search(onclick .DoSearch)
  ├ if .search_err != "" { text .search_err(hint_text) }
  └ 行(flex-wrap)：for h in .search_hits { button h.label(onclick
  .AddHit(h.id), city_pill_off) } } }`）。
- **handler**：
  - `.ToggleAddCity`：`add_open = !.add_open`；开 → `settings_open =
    false`；关 → `search_q/search_err = ""`、`search_hits = []`。
  - `.ToggleSettings`：开合时同步 `add_open = false`（互斥）。
  - `.AddHit(id)` 重写：定位 search_ids → `cities_add` → `r2.ok` 后
    重灌五列表 + `main_id`/`cities_empty` 同步 → **按 id 在 city_ids 内
    索引定位** → pick_city 语义（cur_* + `s_last_id=id` + settings_set 5
    参）→ `report_at` 取数（失败保现状）→ `add_open=false` + 清
    `search_q/search_hits/search_err`；`!r2.ok` 面板不动。
  - `.DoSearch`/`.SearchChanged` 不变（结果渲染地移到面板内）。
- **i18n 四处赋值块**（model 默认 / Init en / SetLang en / SetLang zh）：
  `t_add_city` 与 `t_cities_empty` 新文案（en: "Add city" / "Tap + to
  add a city"）。

### 5.2 后端（src/back/api.at）

- `impl_cities_add`：判重分支由「早退返回部分表」改「置 `dup bool` +
  继续扫描完整个循环」；循环后 `if dup { out.ok = true; return out }`
  （out 已是全表）；非重复路径（追加+重写）不变。注释更新（R2 修复
  说明）。

### 5.3 测试（tests/vm_smoke.py）

- **新增 helper** `open_add_panel(client)`：快照找「＋」按钮 → press →
  断言面板标题「添加城市」出现（唯一文案定位）。
- **改 T3/T7/T8/T10/T15**：所有"主页搜索"步骤改为 open_add_panel →
  输入（沿用 harness 既有 input 驱动）→ 点「搜索」→ 点结果；断言增补：
  点击结果后快照中「添加城市」缺席（面板关闭）+ `cur_id`=新城市 id。
  T15.8 空态 hint 断言改新文案「点 ＋ 添加城市」。
- **新增 T16**（面板专项）：① ＋ 开/关（✕ 关闭、快照面板的存/缺席）；
  ② 与设置卡互斥（开设置卡 → 点 ＋ → 设置卡关、面板开）；③ 添加即
  选中（青岛：cur_id/s_last_id=青岛、settings.json 落盘、面板关闭）；
  ④ 重复添加已有城市（再搜索青岛点结果：面板关闭、cur_id=青岛、
  `city_names` 无重复且**无截断**——R2 修复实证，列表含此前全部城市）。
- 全量 ×2 轮绿（69 旧项数内步骤变更 + T16 新 4 项）。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P0-01 行补「添加入口=＋弹出面板（内联弹出，VM 无 absolute 悬浮），点结果即添加并选中（010）」 | 账实对齐 | AC-01/02 |
| SD-02 | modify | README.md | 城市列表行：搜索入口描述改「右上角 ＋ 弹出面板（搜索拼音/中文 → 点结果添加并选中）」 | 用户可见能力 | AC-07 |

## 6. 测试设计

- T16 见 §5.3；×2 轮全绿；Init 无新增外呼（add_open 纯前端门控）。
- 面板元素定位纪律：「添加城市」标题/「＋」/「✕」在各自快照上下文唯一；
  复用 find_smallest_clickable 优先。

## 7. 验收标准

| ID | 标准 | 验证 |
|---|---|---|
| AC-01 | 主页无搜索输入/结果行；header 右上 ＋ 按钮；点开面板含输入框+搜索按钮+标题 | T16 ①；T1 快照（搜索框缺席断言） |
| AC-02 | 面板内搜索 → 点结果：面板关闭、城市入列且成为当前选中（cur_id/s_last_id 落盘）并取数 | T16 ③；T3/T7 改写段 |
| AC-03 | 面板 ✕ 关闭并清空搜索区；与设置卡互斥不叠显 | T16 ①② |
| AC-04 | 重复添加已有城市：面板关闭、该城选中、列表无重复无截断 | T16 ④ |
| AC-05 | 竖屏功能对齐（portrait 下 ＋/面板可用，主页无搜索块） | T10 改写段 + 快照 |
| AC-06 | vm_smoke 全量 ×2 轮绿 | python tests/vm_smoke.py |
| AC-07 | SD-01/02 落地 | 文件核查 |

## 8. 执行步骤

- [x] T-01 前端：删搜索块、＋ 按钮、add_open 面板、AddHit 重写、
  Toggle 互斥、i18n 四处（AC-01/02/03/05）
  - 验证：`950cf77`（app.at +113/−66）；自查 grep 全过（主页零搜索残留、
    add_open 三处一致、settings_set 仍 8 处 5 参、i18n 四处全补）
- [x] T-02 后端：impl_cities_add 判重返全表（R2 修复）（AC-04）
  - 验证：`fe2cd3b`（api.at 8+/3−，dup 标记+全表返回，不重写文件语义不变）；
    T16.4 两轮 PASS
- [x] T-03 测试：helper + 改 T3/T7/T8/T10/T15 + 新增 T16 + 全量 ×2（AC-04/06）
  - 验证：`8f15a55`（vm_smoke.py +341/−35）；**92/92 ×2 轮全绿 exit 0**
    （T16 11 项断言两轮全过，含 `add_open:false` state 直断言）；覆盖扩及
    T11/T13（任务清单未列但含搜索步骤，一并改写）
- [x] T-04 文档：SD-01/02（AC-07）
  - 验证：`1053e6a`（specs F-P0-01 行 + README 城市列表行）

依赖：T-01∥T-02 可并行；T-03 依赖前两；T-04 最后。

## 9. 复审记录

- `stage: new | PLAN-010 | rev 1 | outcome: pass | next: work`——授权内
  （用户 2026-10-06 需求 + "需要一个新的计划文件跟踪"）；work/review/
  merge 未授权，待用户指示。
- `stage: work | PLAN-010 | rev 1 | outcome: pass | code_commit: 8f15a55
  (on plan-010-dev, base 8340a95; T-01 950cf77 / T-02 fe2cd3b / T-04
  1053e6a) | task_ids: T-01..T-04 | evidence: vm_smoke 92/92 ×2 轮全绿
  exit 0（T16 11 项断言两轮全过）| blockers: 无 | next: review`。
- **实施调整记录**：A1 测试覆盖扩及 T11/T13（任务清单未列但其搜索步骤
  随交互迁移，不改则红；属同一变更面）；A2 首轮 T10 两红为测试前置
  过期——AddHit 新语义使 T8④ 删大连后当前城回落北京，T8 末追加重选
  青岛恢复 T10 前置（src 行为符合既有契约，非 bug）；A3 T16③ 硬编码
  QD_ID 经 curl 实证系 Open-Meteo「青岛」首条命中截断值，依赖 geocoding
  首条排序（若 API 结果集变化会显式 FAIL，脆弱点登记备查）；A4 面板 ✕
  按钮采用完整 style 字面量（icon_btn 无对齐类，视图拼接受限）。
- `stage: review | PLAN-010 | rev 1 | outcome: pass | reviewed_commit:
  8f15a55a4c48dccd4383c678ac82d74c4c8db394 | base_commit:
  8340a95ecab2b34fe0689cc69dba10f3b0c17952 | dependency_revisions:
  auto-lang auto.exe（D:/autostack/auto-lang/target/debug，2026-10-06
  16:50 构建）| spec_inputs: docs/specs/weather-app.md @1053e6a 内 SD-01/02
  冻结 diff（README 随附）| acceptance_results: AC-01..AC-07 全 pass
  （AC-01..06 = 复审复跑 2 vm_smoke 92/92 exit 0 运行时实证，T16 全过 +
  T1 新地标 + T3/T7/T8/T10/T11/T13/T15 改写段；AC-07 = SD-01/02 diff
  逐条核对）| findings: R1 `panel_is_closed` 以「添加城市」缺席判定、空表
  hint「点 ＋ 添加城市」含同子串——仅适用非空表上下文（T16 位于 T15 后
  成立，docstring 已标注；N 级非阻塞）；R2 T8 面板交互时序 flake（复跑 1
  双红、复跑 2 全绿；VM 日志实证该轮 DoSearch HTTP 仅 270ms——属 UI
  快照/aura 时序族，PLAN-008 A4 同族先例；N 级非阻塞，频率升高再硬化重
  试窗）| evidence: 复审复跑 2 `python tests/vm_smoke.py` Total 92 Failed
  0 exit 0（bash-bhisqk5a，/tmp/plan010-review-run2.log，worktree 零未提
  交）；复跑 1 T8 红完整日志 /tmp/plan010-review-run.log + per-boot VM 日
  志；src diff 逐面人工核对（面板构型/AddHit 新语义/Toggle 互斥/判重全表/
  i18n 四处）| next: merge`。局限声明：复审与实施同会话，结论以 git 产物
  + 两轮独立复跑重建，未依赖实施 agent 总结。

## 10. 待澄清事项

- 字面 absolute 悬浮 popover 待 Vue 轨/a2r 支持 absolute 层后再评估
  （本计划内联弹出面板已在交互语义上等价）。
- 面板开启时是否自动聚焦输入框：VM 焦点面无实证，暂不承诺；若走查需要
  再立项。
- 键盘回车触发搜索：同上，待 input keybinding 面确认。
