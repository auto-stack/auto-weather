# auto-weather v2 设计文档（架构与关键决策）

> 配套需求基线：`docs/specs/weather-app.md`。本文记录架构方案与**已实证**的
> 技术约束；计划的任务设计以本文与 specs 为据。

## 1. 总体架构

```
src/front/app.at (widget: model/view/on)
   │  use back.api: weather_report, weather_search, …（类型化调用）
   ▼
src/back/api.at (#[api] 端点，thin 体 + impl_* 实现)
   │  http.get（VM native，出站唯一通道）
   ▼
Provider（可插拔）            持久化
├─ Open-Meteo（现役，免 key）   └─ Env.local_data_dir()/auto-weather/*.json
└─ QWeather（PLAN-005，JWT）      （file.read_text/write_text，双轨一致）
```

- **契约形状**：响应一律扁平标量 + `ok:false` 错误面；列表一律**平行 str/int
  数组**（`ids[]/names[]/…`），由前端 handler 索引循环物化——os-config 探针
  VG12-14 定型的"单跳读"规则。
- **端点纪律**：`#[api]` 体恒 thin（`return impl_*()`）；实现与映射表在普通 fn。

## 2. 双轨现实（Vue / VM）

| 轨 | 后端形态 | 数据面 | 说明 |
|---|---|---|---|
| VM merged（`-r vm` 缺省） | **进程内执行 api.at 全文** | **真实** | 桌面 AutoOS 主面 |
| VM split（`AUTO_VM_MERGE=0`） | AutoVM HTTP server | 真实 | 调试用 |
| Vue（`auto run`） | api_gen 生成的 Rust Axum 桩 | **演示样本（Default 桩）** | 待 a2r http 面 |

`pac.at` **刻意不声明 `api: "rust"`**：该字段在 VM 轨强制 Rust 分离后端
（`rust_ui.rs run_vm_ui`：`backend_impl=="rust"` ⇒ split），而 api_gen 内联
转译面无法翻译 `http.get`/JsonValue 导航（实测 44 个编译错误），且
`primary_type_name_pub`（api.at 首个 pub type）存在时 route A 吸收关闭、
端点落 Default 桩——桩可编译、语义安全降级。Vue 轨真实数据待 a2r-std 补
`http` 面后再开启分离（届时恢复 `api: "rust"` 并在计划内登记偏差）。

## 3. Open-Meteo 契约（现役 provider）

- 实时+预报：`GET https://api.open-meteo.com/v1/forecast?latitude=..&longitude=..&
  current=…,uv_index,visibility&hourly=temperature_2m,weather_code&
  daily=weather_code,temperature_2m_max,temperature_2m_min&forecast_days=5&
  wind_speed_unit=kmh&timezone=auto`
- AQI：`GET https://air-quality-api.open-meteo.com/v1/air-quality?…&current=us_aqi`
  （独立软失败）
- 地理编码（PLAN-001 启用）：`GET https://geocoding-api.open-meteo.com/v1/search?
  name=<q>&count=8&language=zh&format=json`
- 响应为**平行数组**（hourly.time[]/temperature_2m[]/weather_code[]），WMO
  weather_code 映射见 `api.at` 三张表（condition/中文/图标）。

## 4. 数据模型

```
ReportOut  { ok, error, source, condition, cond_zh, icon, temp, feels,
             humidity, wind, wind_dir, pressure, visibility, uv, aqi,
             aqi_level, updated_at, hourly: []HourPoint, daily: []DayPoint }
SearchOut  { ok, error, ids: []str, names: []str, subs: []str,
             lats: []str, lons: []str }          # 平行 str（坐标 2dp 文本）
CityListOut{ ok, error, ids: []str, names: []str, lats: []str, lons: []str }
```

- 坐标传递一律 **2 位小数字符串**（`"39.90"`）：避免 float→str 转换面缺失，
  由 `fmt_latlon()`（全整数运算：`(f+100)*100` 移位取整再拆解）产出。
- 城市 id = `lat_lon`（如 `39.90_116.40`），确定性且天然去重。
- **cities.json 是唯一有序城市表**（PLAN-009）：位置 0 = 主城市（pill ★）；
  文件缺失/损坏时读路径合成北京种子，首次写落盘；`{"cities":[]}` 空表
  （用户删光）是合法持久态；10 内置城表下线（`coords_for` 仅为遗留
  `weather_report` 端点保留，前端统一走 `report_at`）。

## 5. 持久化

`Env.local_data_dir()/auto-weather/cities.json`（唯一有序城市表，位置 0 =
主城市；缺失读路径合成北京种子）。settings.json 5 字段：temp_unit/
wind_unit/lang/startup(main|last)/last_id。后端读-改-写，写前
`quote_json` 转义（os-config 配方）；前端不碰文件。删除城市 = 重写文件；
每次用户选中城市同步 last_id 落盘（Init 启动选路不落盘）。

## 6. 已实证 VM 配方约束（2026-10-04，违反即回归）

1. `json.parse` 结果**只认点访问 / for-in / `""+` 强转**；`.get()/.type()/
   .as_number()` 等方法表不存在（`CALL_SPEC` 实证）。
2. JSON **float 字段**经 `let` 绑定后直接 `as int` 读到装箱垃圾
   （实测 `-858993459`）；须经算术去装箱（`f + 0.0`）或经 typed fn 参数。
3. 平行数组 → 两遍 for-in 预生成**真列表**（`.push`），再 `.get(i)` 对齐
   （parse 结果上禁 `.len()`，F-8 语义缺陷）。
4. `#[api]` 胖体 = api_gen 编译错；必须 thin。
5. a2r 发射器丢 `(a op b) as int` 内层括号 → let 中转变量。
6. 起屏 Init 做真实 HTTP → 首绘可达 10s+，UI 测试首绘重试需 ≥30 次。
7. 前端 handler 保持薄：api 调用 + 单跳标量赋值 + 索引循环物化列表。

## 7. 错误处理与降级矩阵

| 层 | 失败 | 行为 |
|---|---|---|
| back 外呼 | 传输异常 / 非 200 | `try/catch e` + `ok:false` + 错误串 |
| back AQI | 单独失败 | 主报文不受影响，AQI 字段 "—" |
| front | `r.ok == false` | 保留现状（含演示样本），注记不变 |
| 前端渲染 | 列表为空 | for 零迭代，不判 len |

## 8. 计划拆分（roadmap）

| Plan | 目标 | 关键交付 | 依赖 |
|---|---|---|---|
| **PLAN-001** | 城市搜索与多城市管理 | `/api/weather/search`、`/api/weather/cities`（GET/POST 持久化）、`/api/weather/report_at`；前端搜索框+动态 pill+自定义城市切换 | 无 |
| PLAN-002 | 生活指数 + 空气详情 + 城市管理删/排序 | 指数推导 fn、PM 组分端点、城市删除 | 001 |
| PLAN-003 | 降水与预警展示 | 降水概率/量可视化、预警数据结构位 | 001 |
| PLAN-004 | 手机/平板布局 | portrait 打磨、响应式断点 | 001 |
| PLAN-006 | 设置中心 | 单位/主题/语言/默认城市 | 002 |
| PLAN-005 | QWeather provider | JWT 签名接入、预警/分钟级降水 | auto-lang crypto 面或外部 token 服务 |
