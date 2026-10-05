---
plan_id: PLAN-008
status: executing
feature_name: 交互补全（滚动/高亮/详情浮层/设置卡语义/加载态）
author: [agent]
created_at: 2026-10-05T01:00:00Z
updated_at: 2026-10-05T01:00:00Z
plan_revision: 1
current_step: 0
total_steps: 6
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P1-06]
touched_goals: [F-P1-06]
---

# PLAN-008 交互补全（滚动/高亮/详情浮层/设置卡语义/加载态）

> UI/UX 走查结论落地的第一批"小快灵"。基线 v0.6-dev（daebff6，与 origin
> 同步）。注意：PLAN-007（i18n）在 plan-007-dev 分支在途——本计划与其
> 都重度改 app.at，**后合并者需 rebase**（两计划已在各自文件中登记）。

## 0. 变更摘要

五项交互缺陷修复：① 横屏根 col 包 scroll-y（内容超高时底部区块可滚
可见——此前有裁掉风险）；② 自定义城市 pill 选中高亮（F-002 视觉半遗
留收口：active_custom 非空时对应 pill 用 on 样式）；③ 日卡/小时卡
drill-down 详情浮层（遮罩 + 数据卡 + 完成关闭——os-config VG16 普通 if
块确认层同款，禁 popover）；④ 设置卡底部"完成"按钮 + 顶部 ✕（backdrop
行点击同 handler）；⑤ 起屏加载注记（booting 哨兵，Init 取数完毕隐藏）。

## 1. 目标

- **Goal**：首屏无裁切、选中态全程可见、预报可点看详情、设置卡可显式
  关闭、起屏有加载反馈。
- **Non-goals**：skeleton 渐显（中成本，后续）；横屏信息架构重排
  （PLAN-009/010 候选）；i18n（007 在途，本计划文案仍中文）。
- 受影响模块：`src/front/app.at`、`tests/vm_smoke.py`、README、
  `docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-06 全过。

## 2. 架构方案

全部前端工作，无契约变更。浮层走多参 onclick 直传标量（030-video-
player "review_card 循环项多参数 onclick" 实证），handler 零 model 反查
（绕开 VG4 handler 读循环变量 map 限制）。遮罩 = 全屏 `bg-black/50` 行
onclick 关闭，叠在 app_frame 内布局分支之后（双布局共享）。

## 3. 技术栈

AutoLang .at widget/VM merged、vm_smoke MCP（autoui_screenshot 备用）。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-05"建立计划008并实施"）：沿用 new+work 两段
  授权；review/merge 未授权。
- 问题清单来自 2026-10-04 UI/UX 走查（本会话分析）：横屏 scroll 缺失、
  custom pill 无高亮、无 drill-down、设置卡无显式关闭、起屏无加载反馈。
- 风险锚点：Plan 512 教训（scroll 需明确高度，max-w 塌陷史）——scroll-y
  包裹用 h-full 并靠 smoke 快照地标断言守回归。

## 5. 详细设计

### 5.1 前端（src/front/app.at）

- 横屏：`if .layout_mode == "landscape" { scroll (axis: "y", style: "w-full h-full") { col { …原内容… } } }`。
- pill：custom for 循环按钮样式改条件——`if .active_custom == c.id { city_pill_on } else { city_pill_off }`。
- model 增：`det_open bool = false`、`det_icon/det_title/det_l1/det_l2/det_l3 str`、
  `booting bool = true`。
- msg 增：`ShowDayDetail(str,str,str,str,str)`（day/icon/cond/tmin/tmax）、
  `ShowHourDetail(str,str,str)`（time/temp/icon）、`CloseDetail`、
  `CloseSettings`。
- 视图：daily/hourly 卡 onclick 绑多参 msg（日卡：d.day/d.icon/d.cond/
  d.tmin/d.tmax；小时卡：h.time/h.temp/h.icon）；浮层条件块放 app_frame
  尾部（布局分支后）：遮罩行 + 居中大卡（icon 大字 + title + l1/l2/l3 行
  + "完成"按钮 onclick .CloseDetail）；设置卡顶部加 ✕ 行（onclick
  .CloseSettings）、底部"完成"按钮（同 handler）；header 区 `if .booting`
  显示 hint_text "加载实时天气中…"。
- handler：ShowDayDetail/ShowHourDetail 五/三参直存 det_* + det_open=true；
  CloseDetail/CloseSettings 置 false；Init 尾部 `.booting = false`（失败
  路径同——handler 尾部统一置）。

### 5.2 测试（tests/vm_smoke.py）

- 新增 T14：① 详情浮层——切 5日 tab 点"今天"卡 → 快照含 det_cond 实际
  值（cond 级）→ 点"完成"→ 浮层消失（"完成"缺席快照）；② 设置卡 ✕ 关闭
  ——开齿轮 → 点 ✕ → settings_open=false；③ booting 注记——起屏早期
  快照含"加载"字样（首绘断言窗口内，弱断言：T1 快照若含则记 PASS，含
  于首绘重试循环天然覆盖——改为断言 booting 字段存在且最终 false）。
  既有 T1–T13 全回归。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | add | docs/specs/weather-app.md | 新增 F-P1-06「交互品质」行：滚动/选中态/drill-down/设置语义/加载态达成（008） | 账实对齐 | AC-06 |
| SD-02 | modify | README.md | 功能清单补交互特性一行 | 用户可见能力 | AC-06 |

## 6. 测试设计

- T14 见 §5.2；单配置 ×2 全绿（42+3=45 项/轮）。

## 7. 验收标准

| ID | 标准 | 验证 |
|---|---|---|
| AC-01 | 横屏 scroll-y 包裹后全部地标与区块可见（无裁切回归） | T1/T9 快照断言保持绿 |
| AC-02 | 自定义城市 pill 选中高亮（active_custom 对应 on 样式） | T14 state + 样式条件在码 |
| AC-03 | 日/小时卡详情浮层可开可关 | T14 ① |
| AC-04 | 设置卡 ✕/完成关闭 + booting 加载注记 | T14 ②③ |
| AC-05 | vm_smoke 全绿（42 旧 + T14 新 3 项 = 45）×2 | python tests/vm_smoke.py |
| AC-06 | SD-01/02 落地 | 文件核查 |

## 8. 执行步骤

- [ ] T-01 横屏 scroll-y 包裹（AC-01）
- [ ] T-02 custom pill 高亮（AC-02）
- [ ] T-03 详情浮层（卡 onclick + 遮罩 + det_* model/handler）（AC-03）
- [ ] T-04 设置卡 ✕/完成 + booting 注记（AC-04）
- [ ] T-05 tests：T14 + 全量回归（AC-05）
- [ ] T-06 docs：SD-01/02（AC-06）

依赖：T-01..T-04 并行度高；T-05/06 最后。

## 9. 复审记录

- `stage: new | PLAN-008 | rev 1 | outcome: pass | next: work`——授权内。

## 10. 待澄清事项

- skeleton 渐显、温度曲线、日出日落弧、横屏信息架构重排：UI/UX 走查
  P2/P3 项，归 PLAN-009/010 候选（本计划不扩大范围）。
