# auto-weather

天气，使用 AutoLang / AutoUI 开发的独立应用。

本仓是首批产品源码基线；当前能力以导入版本为准，仓库描述中的产品方向不表示全部已实现。

## 运行

安装对应版本的 `auto` CLI 后，从本仓根执行：

```sh
auto run
auto run -r vm
```

- 前端端口：`17816`；后端端口：`17817`（`pac.at` **刻意不声明** `api: "rust"`：VM 轨进程内真数据，Vue 轨走 api_gen 桩——见设计文档 §2）。
- 后端为纯 `.at` 代理（`src/back/api.at`）：前端只打本地 `/api/weather/report`，
  由后端出网取数，VM merged 轨进程内执行，Vue 轨由 `api_gen` 生成服务。

## 数据

- **实时数据**：默认走 [Open-Meteo](https://open-meteo.com/)（免注册免 key，
  非商用免费档），覆盖 10 个内置城市的实时天气 + 24 小时 + 5 日预报 + AQI。
- **降级**：后端请求失败（断网/超时/API 异常）时 `ok:false`，前端全场保留
  `weather_data.at` 演示样本，界面注记保持「演示数据 · 非实时气象」。
- **城市列表**（PLAN-009）：单一有序城市表——首屏只有主城市（缺省北京种子，
  未来接入定位后默认为定位城市，归位 0）；其他城市经搜索（横/竖屏共享顶部
  搜索框，中文/拼音）添加；设置卡 ↑↓ 重新排序（移至顶即主城市，pill 带 ★）；
  整表持久化到 `local_data_dir`，跨启动保留；启动落点可选主城市/上次选中。
- **生活指数与空气详情**（PLAN-002）："生活指数"区展示穿衣/洗车/运动/感冒/
  紫外线（本地推导）；当前详情含 PM2.5/PM10/O3 组分；自定义城市 pill 带
  ✕ 可删除。
- **降水与预警**（PLAN-003）："24小时降水"条形区（逐小时概率%/量mm，条高按
  量分档、颜色按概率分档）；"天气预警"banner 为条件渲染展示位（有预警内容
  时显示，接入待 QWeather）。
- **双布局**（PLAN-004）：横屏（网页/桌面/平板自适应）与竖屏（手机版）
  功能完全对齐——搜索/城市管理/生活指数/24小时降水/预警位双端可用；
  "刷新"对自定义城市同样生效。
- **设置中心**（PLAN-006/009）：齿轮开合设置卡——温度单位 °C/°F、风速单位
  km/h/m/s（即时生效并持久化）、启动偏好（主城市/上次选中，启动生效）、
  城市管理 ↑↓ 排序与删除。设置存于 `local_data_dir/settings.json`（5 字段）。
- **i18n**（PLAN-007）：设置卡切 中文/English——界面文案即时切换；报文文案
  （星期/生活指数/AQI 等级/风向/天气现象）与搜索结果随下次取数生效；内置
  城市英名表；QWeather 三端点 `lang` 参数直连（probe 实证，无需本地映射表）。
  语言存 `settings.json` 的 `lang`（zh/en）。
- **QWeather**：双 provider 之一（PLAN-005）。设 `QWEATHER_KEY` env 时实况/
  24h/7 日/空气优先走 QWeather（`?key=` 查询参数认证——Bearer 头 401 实测），
  任一段失败自动保留 Open-Meteo 基线；不设 key 则纯 Open-Meteo。预警与
  分钟级降水待账号开通对应 API 套餐后接入（端点 404 实测，alert 契约
  已就位）。key 严禁入库——本机开发凭据存 `~/.qweather/`（600）。

## 验证

```sh
python tests/vm_smoke.py   # 需 D:/autostack/auto-lang/target/debug/auto.exe
```


## 来源与组合

来源提交、路径与文件 hash 见 `SOURCE-IMPORT.json`。首次导入提交保留在 `source-sync` 分支；完整 v0.5 恢复后从该基线导入差异，再与产品开发线合并。

AutoOS 通过 [`apps/014-weather`](https://github.com/auto-stack/auto-os/tree/v0.6-dev/apps/014-weather) submodule 固定本仓版本；教学 Demo 保留在来源仓。

已有测试随源导入；端口与平台相关测试需要按本仓配置准备运行环境。安装/启动与双端完整功能验收是不同检查项。
