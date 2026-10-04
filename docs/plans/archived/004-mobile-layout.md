---
plan_id: PLAN-004
status: archived
completion_kind: delivered
feature_name: 竖屏布局正式化（手机版功能对齐）与 F-002 修复
author: [agent]
created_at: 2026-10-04T11:00:00Z
updated_at: 2026-10-04T11:00:00Z
plan_revision: 1
current_step: 4
total_steps: 4
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P1-03]
touched_goals: [F-P1-03]
---

# PLAN-004 竖屏布局正式化（手机版功能对齐）与 F-002 修复

## 0. 变更摘要

把横屏独占的能力全部下沉到竖屏（portrait）分支，使手机预览成为功能
完备的布局：预警 banner、搜索区+结果、城市 pill（内置+自定义+✕）、
PM 行、生活指数区、24小时降水区（窄屏用 scroll-x）。同时修复 PLAN-001
复审发现 F-002：SelectCustom 置空 `.city_id`（内置 pill 不再残留高亮），
并新增 `.active_custom` 使"刷新"对自定义城市走 report_at 通道
（可观察性：Refresh 先置 `--:--`，取数成功才回填）。平板口径：沿用横屏
自适应（w-full/flex 布局随窗宽拉伸），不设第三布局模式。

## 1. 目标

- **Goal**：竖屏快照可见全部功能区块；选中自定义城市后内置 pill 无
  高亮残留；刷新按钮对自定义城市生效。
- **Non-goals**：第三布局模式/断点系统（平板=横屏自适应，文档说明）；
  竖屏专属信息架构重排（仅做等比下沉+窄屏横向滚动）；拖拽排序（006）。
- 受影响模块：`src/front/app.at`、`tests/vm_smoke.py`、README、
  `docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-05 全过，vm_smoke 全绿。

## 2. 架构方案

竖屏分支（`if .layout_mode == "portrait"` 内 scroll-y col）按横屏同构
补块；窄屏差异仅两处：生活指数卡改 scroll-x（flex-wrap iced 不支持，
plan412 降级矩阵在册）、其余区块原生 w-full 自适应。`.active_custom`
跟踪当前自定义城市；SelectCity 清零、SelectCustom 设置；Refresh 优先
走 active_custom 的 report_at，否则内置 weather_report。

## 3. 技术栈

AutoLang .at widget/VM merged、vm_smoke MCP（layout 切换断言）。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04"继续 用技能对付计划004"）：沿用 new+work
  两段授权；review/merge 未授权。
- 前置：PLAN-001/002/003 已归档（751bc3/dbfff44/ba88d40）。横屏独占
  区块：搜索（001）、PM/指数/✕（002）、降水/banner（003）——竖屏缺口
  为本计划核心；F-002 系 PLAN-001 复审 info 发现，指派本计划。
- 实证约束：竖屏 pill 行缺失（原 portrait 无城市条）；flex-wrap 在
  iced 降级（日志实证）；Refresh 旧语义仅内置城市（custom 刷新为
  隐性缺口，随 F-002 一并修）。

## 5. 详细设计

### 5.1 前端（src/front/app.at）

- model 增 `var active_custom str = ""`。
- SelectCity：`.active_custom = ""`；SelectCustom：`.city_id = ""` +
  `.active_custom = id`。
- Refresh：`.refreshing = true` → `.updated_at = "--:--"`（可观察哨兵）→
  `active_custom != ""` 分支（索引循环定位 + report_at + 全量覆写）/
  否则内置分支 → `.refreshing = false`。
- 竖屏分支补块（顺序同横屏）：banner（scroll-y col 顶部）→ 搜索行+
  错误行+结果行 → 城市 pill 行（内置 10 if 链 + custom for + ✕）→
  Hero → 指标 3 行 + **PM 行** → **生活指数区（scroll-x 版）** → 预报
  Tab → **24小时降水区** → data_note。

### 5.2 测试（tests/vm_smoke.py）

- 新增 T10（T9 后，横屏态）：① 切竖屏 → `layout_mode`  portrait +
  快照含"生活指数"与"24小时降水"与"✕"（竖屏对齐）；② 切回横屏（防
  后续用例漂移）；③ F-002：state `city_id` 为 ""（T7 选中青岛后）；
  ④ 自定义刷新：点"刷新" → `updated_at` 非 "--:--"（report_at 通道
  生效实证）。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P1-03 标"竖屏功能对齐达成（004）；平板=横屏自适应口径" | 账实对齐 | AC-05 |
| SD-02 | modify | README.md | 布局节：竖屏=手机版全功能；平板说明 | 用户可见能力 | AC-05 |

## 6. 测试设计

- T10（新）见 §5.2；既有 T1–T9 不回归（T4 留竖屏→T8 回横屏→T10 再切）。

## 7. 验收标准

| ID | 标准 | 验证 | 结果 |
|---|---|---|---|
| AC-01 | 竖屏快照含搜索/生活指数/24小时降水/✕（layout_mode=portrait 前提） | T10 ①（portrait + 三区/✕ 快照断言） | **pass**（34/34 ×2） |
| AC-02 | SelectCustom 后 city_id=""（内置 pill 无高亮残留） | T10 ③（city_id "" + city_zh 青岛） | **pass** |
| AC-03 | 自定义城市刷新走 report_at（updated_at 哨兵被回填） | T10 ④（哨兵非 --:-- + Open-Meteo） | **pass** |
| AC-04 | vm_smoke 全绿（31 旧 + T10 新 3 项 = 34）×2 | python tests/vm_smoke.py ×2 | **pass**（口径修正：T10 实装 3 断言，计划草案"4 项"系把无断言的"切回横屏"步骤误计） |
| AC-05 | SD-01/02 落地 | 文件核查（commit 348b9b0） | **pass** |

## 8. 执行步骤

- [x] T-01 前端：竖屏补块（banner/搜索/pill/PM/指数/降水）（AC-01）
  - 验证：T10 ①；竖屏指数区用 scroll-x（flex-wrap iced 不支持）
- [x] T-02 前端：F-002（active_custom/SelectCustom/SelectCity/Refresh）（AC-02/03）
  - 验证：T10 ③④；Refresh 双分支（自定义 report_at/内置 report），
    updated_at 哨兵使通道可观察；赋值顺序与既有块对齐（data_note 在
    pm 之前——Refresh 块历史形态，replace 时按现文件对齐）
- [x] T-03 tests：T10 + 全量回归（AC-04）
  - 验证：vm_smoke 34/34 ×2（w4-smoke2/3）；T10 ③断言曾误查
    active_custom（存 id 不存名），改查 city_zh 后通过——测试学习点
- [x] T-04 docs：SD-01/02（AC-05）
  - 验证：commit 348b9b0 文件核查

依赖：T-01/T-02 可并行；T-03/04 最后。

## 9. 复审记录

- `stage: new | PLAN-004 | rev 1 | outcome: pass | next: work`——授权内
  （"对付计划004"沿用 new+work），任务覆盖 AC-01..05 与 SD-01/02。
- `stage: work | PLAN-004 | rev 1 | outcome: pass | code_commit: 348b9b0
  (on plan-004-dev, base 5bf5bd3) | task_ids: T-01..T-04 | evidence:
  worktree vm_smoke 34/34 ×2（w4-smoke2/3 全绿）| blockers: 无 |
  next: review`——所有任务与 AC 映射已核销，变更已提交，worktree 保留待复审。
- **实施调整记录**：A1 AC-04 口径修正（34 项，见 §7 注）；A2 T10 ③断言
  字段选择（active_custom 存 id / city_zh 存名——model 语义以代码为准）；
  A3 Refresh 赋值顺序按现文件形态对齐（未强行统一块内顺序，避免无收益
  diff）。无技术阻塞项。
- `stage: review | PLAN-004 | rev 1 | outcome: pass |
  reviewed_commit: 348b9b07341839e97e8440c4b8f255ba49eebb06 |
  base_commit: 5bf5bd3（docs(plan): PLAN-004 contract）|
  dependency_revisions: auto-lang 168b56923（仅验证工具链）|
  spec_inputs: docs/specs/weather-app.md（worktree 含 SD-01）、README.md
  （worktree 含 SD-02）——增量已核：描述当前行为与持久决策，未发布 |
  acceptance_results: AC-01 pass（T10 ① portrait+三区/✕）/ AC-02 pass
  （T10 ③ city_id "" + city_zh 青岛）/ AC-03 pass（T10 ④ 哨兵回填 +
  Open-Meteo）/ AC-04 pass（复审复现 34/34 exit 0）/ AC-05 pass
  （SD-01/02 文件核查，commit 348b9b0）|
  findings: F-001 info——竖屏区块与横屏同构复制（~280 行），DSL 无
  组件抽取机制，若后续引入 widget include 可收敛；F-002 info——竖屏
  scroll-y 固定 max-h-[520px]，内容变高后依赖滚动（行为正确）；
  F-003 info——active_custom 存 id 不存名（model 语义），已在计划
  §9 A2 记录。均无阻塞 |
  evidence: 复审复现 /tmp/review4-smoke.out（34/34）；worktree 零脏文件 |
  next: merge`。复审局限声明：复审与实施同会话，结论全部经工件与
  复现命令重建。
- `stage: merge | PLAN-004:r1 | outcome: pass`——归并收据：`prepared`
  基线 r1/pass @ 348b9b0；冻结增量 SD-01/02；交付提交 348b9b0；口径沿用
  PLAN-001 裁定。`landed`：rebase 后 `--ff-only`——v0.6-dev tip =
  f7ed6371cffee5d79916d5795995dc4930c2fde5 = 交付提交（改写映射
  348b9b0→f7ed637；range-diff `=` 等价）；主检出集成冒烟 34/34
  （/tmp/merge4-smoke.out）。`ledger_refreshed`: N/A。`archived`:
  docs/plans/archived/004-mobile-layout.md，delivered。`cleaned`:
  wt-guard.sh 缺失（人工等价核查）；移除前预杀 worktree 派生
  node/esbuild；worktree 与 plan-004-dev 已移除，组目录已删。

## 10. 待澄清事项

- 平板专有问题（如横屏断点细化）随真实平板验证再立计划；竖屏信息
  架构重排（而非等比下沉）属 P1 增强，未排期。
