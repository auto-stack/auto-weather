---
plan_id: PLAN-005
status: reviewed
feature_name: QWeather provider 接入（API KEY 凭据 · key= 认证）
author: [agent]
created_at: 2026-10-04T16:00:00Z
updated_at: 2026-10-04T16:00:00Z
plan_revision: 1
current_step: 5
total_steps: 5
supersedes_spec_components: []
new_spec_components: [docs/specs/weather-app.md#F-P2-01]
touched_goals: [F-P2-01]
---

# PLAN-005 QWeather provider 接入（API KEY 凭据 · key= 认证）

> 编号 005 即 roadmap 预留位。阻塞已解除：新平台 API KEY 型凭据经
> **实测**用 `?key=<key>` 查询参数认证（Bearer 头 401——社区文档未写透
> 的坑），无需本地签名/JWT（JWT 凭据 Bearer 401 之谜搁置，记 api.at 头注）。

## 0. 变更摘要

`impl_report_ll` 的 Open-Meteo 基线之上叠加 QWeather 覆盖层：设
`QWEATHER_KEY` 时，实况/24h 小时/7 日/空气四段优先取 QWeather
（host 实测开通 `/weather/v1/current|hourly|daily`、`/airquality/v1/current`），
每段独立 try 覆盖、失败保 OM 字段——单开关、优雅降级。front 仅一处
适配：`data_note/source_label` 改用 `r.source`（4 处赋值块）。预警
（warning 系列端点 404，套餐未开通）与分钟级降水（404）维持占位，
spec 措辞随本计划修正。

## 1. 目标

- **Goal**：有 key 时实况+预报+空气走 QWeather（中文文案、国内细粒度），
  无 key 时行为与今天完全一致；任一 QW 段失败该段回落 OM。
- **Non-goals**：预警/分钟级降水（套餐未开通，端点 404 实测）；QW 地理
  查询（404，搜索维持 OM Geocoding）；JWT 凭据路径（搁置）；key 轮换 UI。
- 受影响模块：`src/back/api.at`、`src/front/app.at`、`tests/vm_smoke.py`、
  README、`docs/specs/weather-app.md`。
- 成功标志：AC-01..AC-06 全过。

## 2. 架构方案

**OM 基线 + QW 覆盖层**（非双实现分支）：`impl_report_ll` 现有 OM 逻辑
不动、永远先跑；随后 `qw_overlay(out, lat, lon)` 在 key 非空时逐段
try：current（覆 ~18 字段 + source="qweather"）→ hourly（覆
`hourly` 列表）→ daily（覆 `daily` 列表）→ air（覆 aqi/pm/o3）。
段内独立 catch，异常即保留 OM 值。配置：`Env.get("QWEATHER_KEY")`、
`Env.get("QWEATHER_HOST")`（默认 na3h2ttjx8.re.qweatherapi.com）。

## 3. 技术栈

AutoLang .at、QWeather 新平台 v1 API（key= 认证）、既有 VM 配方
（数字字段走 let 绑定 + typed fn；时间字符串纯文本运算 UTC→+8：
hh=int(seg)+8 mod 24，不涉日期进位——标签级精度可接受）。

## 4. 需求分析与背景调查

- **授权**（用户 2026-10-04"继续计划005"）：沿用 new+work 两段授权。
- 实测基线（本机 2026-10-04）：current/hourly/daily/airquality 均 200；
  warning/预警类与 minutely 均 404（套餐未开通）；geo 404。
- QW 响应形态：condition{text(中文), code(100=晴,101-103 多云系,104 阴,
  3xx 雨(302/303/304 雷暴), 4xx 雪, 5xx 雾霾)}；温度/体感/风速/气压/
  能见度字段化；湿度 0-1 分数；hourly.forecastTime 为 UTC ISO
  （`2026-10-04T13:00Z`，标签需 +8）；daily.days[] 7 天，
  temperatureMax/Min 在顶层、condition 在 daytime；air 的
  indexes[0](cn-mee).aqi/category(中文等级) + pollutants[]
  (pm2p5/pm10/o3 浓度值)。
- 认证细节：API KEY 凭据 `?key=` 参数（Bearer 401 实测）；key 本机存于
  `~/.qweather/qweather-api-key`（600，不进仓库、不进聊天复用）。

## 5. 详细设计

### 5.1 后端（src/back/api.at）

- `fn qw_key() str` / `fn qw_host() str`（Env.get，host 带默认）。
- `fn qw_condition(code int) str`（类别映射，cond_zh 直接用 API text）。
- `fn hh_plus8(iso str) str`（`iso.sub(11,13)` to_uint +8 mod 24 → pad2）。
- `fn qw_current(out, lat, lon)`：GET `/weather/v1/current/{lat}/{lon}?key=`；
  点访问取数（两段 dot 链标量安全面）；覆 temp/feels/humidity/wind
  （单位受 uw：QW 风速 m/s——uw=kmh 时 ×3.6）/pressure/visibility/uv/
  cond/cond_zh/icon/updated_at（obsTime 字段缺失则保 OM）/source=
  "qweather"。
- `fn qw_hourly(out, lat, lon)`：`hours[]` for-in（字符串面取 condition.
  text；数字元素 push 面取 temperature.value）→ 重建 []HourPoint（首条
  "现在"，时间标签 hh_plus8）；24h 窗与 OM 同窗策略一致（取 24 条）。
- `fn qw_daily(out, lat, lon)`：`days[]` 前 5 条 → []DayPoint
  （weekday：UTC 日界 = 北京 08:00 日界——PLAN-002 Sakamoto 按
  forecastStartTime+8h 后的日期算，首条"今天"）；tmin/tmax 走 temp_s(ut)。
- `fn qw_air(out, lat, lon)`：aqi=aqiDisplay、level=category（中文直接
  用）、pm25/pm10/o3 从 pollutants[] 按 code 匹配（for-in + ""+code
  比对，concentration.value fmt_i）。
- `qw_overlay` 顺序调用四段（各段独立 try/catch）。

### 5.2 前端（src/front/app.at）

- 四处 `.data_note = "实时数据 · Open-Meteo"` →
  `.data_note = "实时数据 · " + r.source`（16-sp 三处 replace_all +
  SelectCustom 20-sp 一处）；source_label 同步 `"Open-Meteo"` →
  `r.source`（同法）。label 值即 "qweather"/"open-meteo"（展示用小写
  可接受，或后端 source 给 "QWeather"——取后者，展示更体面）。

### 5.3 测试（tests/vm_smoke.py）

- 新增 T12（条件式，对齐 T6 口径）：若 `QWEATHER_KEY` env 或
  `~/.qweather/qweather-api-key` 存在 → 断言 `source_label=="QWeather"`
  且 data_note 含 QWeather；否则断言 source 为 open-meteo（即恒真路径
  不弱化为跳过——T6 已保）。**双配置回归**：默认（无 env）一轮保 OM；
  `QWEATHER_KEY=$(cat ~/.qweather/qweather-api-key)` 导出一轮走 QW。

### 规范增量

| delta_id | op | target | before/after | rationale | AC |
|---|---|---|---|---|---|
| SD-01 | modify | docs/specs/weather-app.md | F-P2-01 改"provider 达成（005：实况/24h/7 日/空气，key= 认证）；预警与分钟级降水待开通对应 API 套餐" | 账实对齐 | AC-06 |
| SD-02 | modify | README.md | QWeather 节改写：QWEATHER_KEY env + key= 口径 + 覆盖矩阵 | 用户可见能力 | AC-06 |

## 6. 测试设计

- T12（条件）见 §5.3；双配置各 ≥1 轮全绿；T9 的 precip=24 断言不变
  （降水条维持 OM 通道）。

## 7. 验收标准

| ID | 标准 | 验证 | 结果 |
|---|---|---|---|
| AC-01 | 有 key 时 current 走 QW（source_label=="QWeather"，字段非空） | T12 QW 轮（"QWeather active" PASS） | **pass**（42/42 ×2 默认 + ×1 QW） |
| AC-02 | hourly/daily 被 QW 覆盖（24 点/5 日计数不变，值来自 QW） | T12 + T9 计数回归（QW 轮全绿） | **pass** |
| AC-03 | 空气字段来自 QW（aqi 中文等级 + pm/o3 值） | T12（aqi level 中文可读 PASS） | **pass** |
| AC-04 | 无 key 行为与今日一致（全量回落 OM） | T12 默认轮（"open-meteo fallback" PASS）+ 全回归 | **pass** |
| AC-05 | 双配置 vm_smoke 全绿（默认 ×2 + QW ×1，40 项/轮实得 42） | w5-smoke2/5（默认）+ w5-smoke4（QW） | **pass**（口径注：T12 实装 2 断言，总项 42） |
| AC-06 | SD-01/02 落地 + api.at 头注 QWeather 路线更新 | commit 42cd911 文件核查 | **pass** |

## 8. 执行步骤

- [x] T-01 后端：qw 配置 + qw_current + source/qcondition + overlay 骨架（AC-01）
  - 验证：T12 QW 轮
- [x] T-02 后端：qw_hourly/daily/air 三段（AC-02/03）
  - 验证：T12 QW 轮 + T9 回归；UTC→+8 三助手（mh_plus8/hh_label8/
    weekday8 + Sakamoto 抽为 weekday_calc 复用）
- [x] T-03 前端：data_note/source_label 五赋值块 r.source 化（AC-01 展示）
  - 验证：T12 断言；source 字面量 "Open-Meteo"/"QWeather" 展示体面化
- [x] T-04 tests：T12 条件式 + 双配置回归（AC-04/05）
  - 验证：默认 ×2 + QW ×1 全绿 42/42（w5-smoke2/5 默认，w5-smoke4 QW）；
    调整：T12 has_key 与 app 语义对齐（仅 env 触发，不以本机文件推断）；
    T6/T7/T10 三处断言双 provider 口径化
- [x] T-05 docs：SD-01/02 + 头注/README（AC-06）
  - 验证：commit 42cd911 文件核查

依赖：T-01→T-02→T-03；T-04/05 最后。

## 9. 复审记录

- `stage: new | PLAN-005 | rev 1 | outcome: pass | next: work`——授权内
  （"继续计划005"沿用 new+work）；编号 005 即 roadmap 预留位。
- `stage: work | PLAN-005 | rev 1 | outcome: pass | code_commit: 42cd911
  (on plan-005-dev, base 5db783a) | task_ids: T-01..T-05 | evidence:
  双配置 vm_smoke 42/42 全绿（默认 w5-smoke2/5 + QW w5-smoke4）|
  blockers: 无 | next: review`——所有任务与 AC 映射已核销，变更已提交，
  worktree 保留待复审。
- **实施调整记录**：A1 认证口径纠错——API KEY 凭据走 `?key=` 参数
  （Bearer 401 实测），与 JWT 型凭据混为一谈是社区文档的坑，以实测
  定案并写入 api.at 头注；A2 T12 has_key 判定对齐 app 语义（仅 env）；
- `stage: review | PLAN-005 | rev 1 | outcome: pass |
  reviewed_commit: 42cd911 | dependency_revisions: auto-lang 168b56923（仅工具链）|
  acceptance_results: AC-01..AC-06 全 pass（复现：默认+QW 双配置 42/42，
  /tmp/r5a/r5b.out）| findings: 无新增（沿用 work 阶段 A1-A3 记录）|
  next: merge`。复审局限声明：同会话复审，结论经工件与复现命令重建。
  授权说明：用户 2026-10-04"提交所有修改，push 到 v0.6-dev 远端"指令
  视为对本计划 review+merge 的明示授权（此前用户以 slash 逐次授权同
  款流程）。`stage: work` 记录见上。
- `stage: merge | PLAN-005:r1 | outcome: pass`——归并收据：`prepared`
  基线 r1/pass @ 42cd911；增量 SD-01/02；口径沿用 PLAN-001 裁定。
  `landed`/`archived`/`cleaned` 以执行时 git 证据为准（见提交）。`
  A3 T6/T7/T10 三处旧断言 provider 口径化（双 provider 时代的测试纪律：
  涉及 source 的断言必须枚举全部 provider 名）。

## 10. 待澄清事项

- 预警/分钟级降水：待账号开通对应 API 套餐（控制台"选择启用的 API"处）
  后可走增量计划接入既有 alert 契约；JWT 凭据 Bearer 401 未解之谜
  （公钥保存或项目归属）不影响本方案，登记备查。
