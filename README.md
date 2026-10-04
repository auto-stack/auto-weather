# auto-weather

天气，使用 AutoLang / AutoUI 开发的独立应用。

本仓是首批产品源码基线；当前能力以导入版本为准，仓库描述中的产品方向不表示全部已实现。

## 运行

安装对应版本的 `auto` CLI 后，从本仓根执行：

```sh
auto run
auto run -r vm
```

- 前端端口：`17816`；后端端口：`17817`（v0.6-dev 起，`pac.at` 声明 `api: "rust"`）。
- 后端为纯 `.at` 代理（`src/back/api.at`）：前端只打本地 `/api/weather/report`，
  由后端出网取数，VM merged 轨进程内执行，Vue 轨由 `api_gen` 生成服务。

## 数据

- **实时数据**：默认走 [Open-Meteo](https://open-meteo.com/)（免注册免 key，
  非商用免费档），覆盖 10 个内置城市的实时天气 + 24 小时 + 5 日预报 + AQI。
- **降级**：后端请求失败（断网/超时/API 异常）时 `ok:false`，前端全场保留
  `weather_data.at` 演示样本，界面注记保持「演示数据 · 非实时气象」。
- **城市搜索**（PLAN-001）：横屏布局顶部输入任意城市（中文/拼音）→ 点击
  结果添加为自定义城市 pill，点击 pill 查看真实天气；列表经后端持久化到
  `local_data_dir`，跨启动保留。
- **生活指数与空气详情**（PLAN-002）："生活指数"区展示穿衣/洗车/运动/感冒/
  紫外线（本地推导）；当前详情含 PM2.5/PM10/O3 组分；自定义城市 pill 带
  ✕ 可删除。
- **降水与预警**（PLAN-003）："24小时降水"条形区（逐小时概率%/量mm，条高按
  量分档、颜色按概率分档）；"天气预警"banner 为条件渲染展示位（有预警内容
  时显示，接入待 QWeather）。
- **QWeather**：自定义 API Host 已实证只认新平台 JWT
  （`Authorization: Bearer <JWT>`），Key ID 本身不是 token；纯 `.at` 侧
  暂无 Ed25519/ES256 签名原语，待 auto-lang crypto 面或外部 token 服务
  落地后作为第二 provider 接入（私钥经 env 注入，严禁入库）。

## 验证

```sh
python tests/vm_smoke.py   # 需 D:/autostack/auto-lang/target/debug/auto.exe
```


## 来源与组合

来源提交、路径与文件 hash 见 `SOURCE-IMPORT.json`。首次导入提交保留在 `source-sync` 分支；完整 v0.5 恢复后从该基线导入差异，再与产品开发线合并。

AutoOS 通过 [`apps/014-weather`](https://github.com/auto-stack/auto-os/tree/v0.6-dev/apps/014-weather) submodule 固定本仓版本；教学 Demo 保留在来源仓。

已有测试随源导入；端口与平台相关测试需要按本仓配置准备运行环境。安装/启动与双端完整功能验收是不同检查项。
