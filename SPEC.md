# Spec: 香港疫情实时监控大屏 (HK Epidemic Monitoring Dashboard)

## 1. 业务目标 Purpose

为香港疫情数据提供实时可视化监控大屏，集中展示以下核心信息：

- 疫情整体态势（累计确诊、日新增、现存确诊、数据日期范围）
- 每日新增与累计确诊趋势
- 各区累计确诊 Top 10 排名
- 各区最新日新增 Top 12 排名
- 月度新增趋势
- 各区现存确诊地理分布热力图
- 风险等级分布

核心价值：通过统一的 Web 大屏界面，帮助决策者快速掌握疫情全局态势及区域差异。

---

## 2. 上下文与依赖 Context & Dependencies

### 2.1 数据输入

| 来源 | 类型 | 必填 | 说明 |
|---|---|---|---|
| Excel 疫情数据文件 | `.xlsx` | MUST | 固定文件名，位置索引读取，含日期、地区、日新增确诊、累计确诊、现存确诊（可选）、风险等级（可选） |
| GeoJSON 地理边界文件 | `.json` | SHOULD | 香港 18 区行政边界，用于地图渲染；缺失时地图降级为纯文本列表 |
| 环境变量 `PORT` | 整数 | OPTIONAL | 服务监听端口，默认 5000 |

### 2.2 外部依赖

| 依赖 | 用途 | 说明 |
|---|---|---|
| Flask | HTTP 服务 | 提供页面渲染与 API |
| Pandas | 数据聚合 | 读取 Excel、分组、聚合、排序 |
| openpyxl | Excel 读取引擎 | 解析 `.xlsx` 文件 |
| ECharts 5.5.0 | 前端图表渲染 | 浏览器端图表/地图绘制 |

### 2.3 输出

| 去向 | 内容 | 说明 |
|---|---|---|
| 浏览器页面 | HTML + JSON 数据 | 6 个 KPI 卡片 + 6 个图表面板 |
| 独立趋势图文件 | PNG | 仅 `generate_confirmed_trends.py` 独立脚本生成，不参与大屏展示 |

### 2.4 非依赖项

- **无需数据库**：数据直接从 Excel 文件读取。
- **无需认证**：系统面向内部/本地使用，无用户认证机制。

---

## 3. 数据契约 Data Contract

### 3.1 Excel 数据源契约

按列位置索引读取（不依赖列名）：

| 列索引 | 字段名 | 类型 | 必填 | 约束 |
|---|---|---|---|---|
| 0 | 日期 | `datetime` | MUST | 可解析为合法日期，否则丢弃该行 |
| 1 | 地区 | `string` | MUST | 香港 18 区名称，须与 GeoJSON 中 `properties.name` 可匹配 |
| 2 | 日新增确诊 | `integer` | MUST | 非负整数；缺失值按 0 处理 |
| 3 | 累计确诊 | `integer` | MUST | 非负整数；缺失值按 0 处理 |
| 4 | 现存确诊 | `integer` | OPTIONAL | 非负整数；缺失时按 0 处理 |
| 11 | 风险等级 | `string` | OPTIONAL | 枚举值（如"高风险""中风险""低风险"等）；缺失时该项数据为空 |

### 3.2 API 接口契约

#### 3.2.1 `GET /` — 主页面

- **入参**：无
- **出参**：HTML 页面（`dashboard.html`）

#### 3.2.2 `GET /api/overview` — 核心 KPI 概览

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `date_range.start` | `string (YYYY-MM-DD)` | 数据起始日期 |
| `date_range.end` | `string (YYYY-MM-DD)` | 数据截止日期 |
| `latest_date` | `string (YYYY-MM-DD)` | 最新统计日 |
| `latest_daily_new` | `integer` | 最新统计日全港日新增确诊 |
| `latest_cumulative` | `integer` | 最新统计日全港累计确诊 |
| `active_total` | `integer` | 全港现存确诊合计 |
| `active_top_district` | `string` | 现存确诊最高的地区 |
| `active_top_value` | `integer` | 该地区现存确诊数 |
| `cum_top_district` | `string` | 累计确诊最高的地区 |
| `cum_top_value` | `integer` | 该地区累计确诊数 |
| `latest_month` | `string (YYYY-MM)` | 最新统计月份 |
| `latest_month_new` | `integer` | 该月新增确诊数 |
| `risk_main_level` | `string` | 占比最高的风险等级 |
| `risk_main_pct` | `float` | 该风险等级占比（0–100） |

- **空数据行为**：数据源为空时返回空对象 `{}`。

#### 3.2.3 `GET /api/trend` — 趋势数据

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `dates` | `array<string>` | 日期序列（YYYY-MM-DD），升序 |
| `daily_new` | `array<integer>` | 每日全港新增确诊数，与 dates 一一对应 |
| `cumulative` | `array<integer>` | 每日全港累计确诊数，与 dates 一一对应 |

#### 3.2.4 `GET /api/district_cumulative_top` — 各区累计确诊 Top 10

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `districts` | `array<string>` | 地区名称列表（按累计确诊降序，取前 10） |
| `values` | `array<integer>` | 对应累计确诊数 |

#### 3.2.5 `GET /api/monthly_new` — 月度新增

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `months` | `array<string (YYYY-MM)>` | 月份序列，升序 |
| `values` | `array<integer>` | 对应月份新增确诊合计 |

#### 3.2.6 `GET /api/district_active_distribution` — 各区现存确诊分布

- **入参**：无
- **别名路由**：`/api/district_active_distribution/`、`/api/district_active`
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `stat_date` | `string` | 统计日期（最新日） |
| `districts` | `array<string>` | 地区名称列表（按现存确诊降序） |
| `values` | `array<integer>` | 对应现存确诊数 |

- **数据去重**：同一地区在同一日出现多条记录时，取现存确诊的最大值。

#### 3.2.7 `GET /api/district_daily_latest` — 各区最新日新增 Top 12

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `districts` | `array<string>` | 地区名称列表（按日新增降序，取前 12） |
| `values` | `array<integer>` | 对应日新增数 |

#### 3.2.8 `GET /api/risk_distribution` — 风险等级分布

- **入参**：无
- **出参**：

| 字段 | 类型 | 业务含义 |
|---|---|---|
| `levels` | `array<string>` | 风险等级标签列表 |
| `values` | `array<integer>` | 对应等级的地区数量 |

- **缺失行为**：Excel 无风险等级列时，返回 `{"levels": [], "values": []}`。

#### 3.2.9 `GET /api/hk_districts_geo` — 地理边界数据

- **入参**：无
- **出参**：GeoJSON 格式内容（`application/json`）
- **缺失行为**：文件不存在时返回 HTTP 404，响应体为 `{"error": "geo file missing", "path": <文件路径>}`。

### 3.3 独立脚本契约 — `generate_confirmed_trends.py`

| 项 | 说明 |
|---|---|
| 输入 | 与 `data_loader.py` 相同的 Excel 文件 |
| 输出 | `static/images/confirmed_trends.png` 静态图片 |
| 图表内容 | 全港每日新增确诊（折线）+ 累计确诊（折线，双 Y 轴） |
| 独立性 | 不依赖 Flask 运行，直接读取 Excel 文件生成图片 |

---

## 4. 主流程 Happy Path

### 4.1 系统启动

1. 系统从环境变量读取端口号（默认 5000）。
2. Flask 服务在指定端口启动，绑定 `0.0.0.0`。

### 4.2 页面访问

1. 用户通过浏览器访问 `/`。
2. 系统返回 `dashboard.html` 页面。
3. 页面加载完成后，前端并行发起 8 个 API 请求获取全部数据。
4. 前端收到所有响应后，将 KPI 数据填入卡片，渲染 6 个 ECharts 图表。
5. 页面右上角显示实时时钟。

### 4.3 地理地图渲染

1. 前端请求 `/api/hk_districts_geo`。
2. 若 Flask 返回 GeoJSON，直接使用。
3. 若 Flask 返回 404，尝试从静态文件路径 `static/geo/hk_districts_full.json` 加载。
4. 若仍失败，尝试从外部 CDN（阿里 DataV）加载香港 GeoJSON。
5. 全部失败时，地图区域降级显示地区名称与数值的纯文本列表。

### 4.4 风险分布渲染

1. 前端请求 `/api/risk_distribution`。
2. 若返回非空数据，渲染饼图。
3. 若返回空数组，渲染"暂无风险等级字段数据"占位文本。

### 4.5 独立趋势图生成

1. 用户执行 `python generate_confirmed_trends.py`。
2. 脚本读取 Excel 文件，聚合全港每日数据。
3. 生成双轴折线图并保存至 `static/images/confirmed_trends.png`。
4. 在终端中弹窗显示图表。

---

## 5. 异常 & 错误分支 Error Scenarios

| 场景 | 触发条件 | 系统预期行为 | 返回/错误 |
|---|---|---|---|
| E1: Excel 文件缺失 | 指定路径下无 `.xlsx` 文件 | `data_loader` 抛出 `FileNotFoundError`；前端 fetch 失败，显示错误信息 | HTTP 500（未捕获异常） |
| E2: Excel 文件格式错误 | 文件存在但列数不足 4 列 | 读取时产生错误或缺失列 | 取决于 Pandas 行为，通常为异常 |
| E3: Excel 数据为空 | 文件存在但无数据行 | `/api/overview` 返回 `{}`；其他接口返回空数组 | HTTP 200，空数据 |
| E4: GeoJSON 文件缺失 | `static/geo/hk_districts_full.json` 不存在 | `/api/hk_districts_geo` 返回 404；前端尝试备用加载路径 | HTTP 404 + 错误 JSON |
| E5: 风险等级列缺失 | Excel 列数不足 12 | `/api/risk_distribution` 返回 `{"levels": [], "values": []}` | HTTP 200，空数据 |
| E6: 现存确诊列缺失 | Excel 列数不足 5 | 现存确诊按 0 处理；地图和 KPI 中相关值为 0 | HTTP 200，降级数据 |
| E7: 日期解析失败 | 某行日期列无法解析为合法日期 | 丢弃该行 | HTTP 200，部分数据缺失 |
| E8: 前端 API 请求失败 | 网络异常或服务不可用 | 前端 catch 错误，替换整个图表区域为错误提示 | 用户可见错误信息 |
| E9: GeoJSON 全部加载路径失败 | Flask + 静态 + CDN 三路径均失败 | 地图降级为纯文本列表 | 用户可见文本列表 |
| E10: 同名地区同日多条记录 | 数据源中同一 district + date 出现多行 | 现存确诊取最大值 | HTTP 200，去重后数据 |

---

## 6. 边界条件 Boundary Cases

| 场景 | 预期行为 |
|---|---|
| 数据仅含 1 个日期 | 趋势图仅 1 个点；KPI 中 date_range.start == date_range.end |
| 数据仅含 1 个地区 | 地图仅 1 个区域有值；排名类图表仅 1 条数据 |
| 某地区日新增为 0 | 正常展示为 0，不参与 Top 排名 |
| 所有地区日新增均为 0 | 排名图显示全 0；地图无高亮 |
| 最新月份仅部分天数有数据 | 月度新增仅统计有数据的天数 |
| 地区名称与 GeoJSON 不完全匹配 | 匹配失败的地区在地图上无法着色，但不影响其他区域 |
| Excel 行数极大（>10 万行） | 启动时首次加载变慢；`lru_cache` 缓存后 API 响应不受影响 |
| 同一地区在同一天存在多条记录且现存确诊值不同 | 取最大值作为该地区当日的现存确诊 |
| Excel 中日期非单调递增 | `prepared_frames()` 按日期排序后输出，不影响趋势图 |

---

## 7. 非功能约束 Non‑Functional Requirements

| 约束 | 说明 | 级别 |
|---|---|---|
| NFR-1: 数据加载缓存 | Excel 文件仅在首次读取时加载并缓存，后续请求不重复读取 | MUST |
| NFR-2: 无认证机制 | 系统不设用户登录/权限校验 | MUST |
| NFR-3: 响应式布局 | KPI 卡片在 1280px 以下切换为 3 列，720px 以下切换为 2 列；图表在 1100px 以下切换为单列 | MUST |
| NFR-4: 暗色主题 | 大屏使用深色背景 + 青色高亮配色方案 | SHOULD |
| NFR-5: 窗口自适应 | 浏览器窗口尺寸变化时，所有 ECharts 图表自动重绘适配新尺寸 | MUST |
| NFR-6: 数据实时性 | 时钟实时刷新；疫情数据需重启服务方可更新（缓存机制决定） | MUST |
| NFR-7: 前端并行加载 | 页面初始化时 8 个 API 请求并行发起，非串行 | SHOULD |
| NFR-8: 浏览器兼容 | 推荐 Chrome、Firefox、Edge 等现代浏览器 | SHOULD |
| NFR-9: 中文渲染 | 图表标签、KPI 文本使用微软雅黑/黑体等中文字体 | MUST |

---

## 8. 验收准则 Acceptance Criteria

### AC-1: 核心 KPI 卡片渲染

**Given** Excel 文件存在且含有效数据
**When** 用户访问 `/` 页面
**Then** 页面显示 6 个 KPI 卡片，包含：累计确诊、最新日新增、各区现存合计、重点地区排名、月度新增、风险等级占比

*MUST*

### AC-2: 趋势图正确展示

**Given** `/api/trend` 返回包含 dates、daily_new、cumulative 的数据
**When** 前端加载完成
**Then** `#chart-trend` 中渲染双轴折线图，左轴为日新增、右轴为累计

*MUST*

### AC-3: 地区累计 Top 10

**Given** 数据含多个地区的累计确诊数据
**When** 前端请求 `/api/district_cumulative_top`
**Then** 返回按累计确诊降序排列的前 10 个地区及对应数值，图表中正确渲染为水平条形图

*MUST*

### AC-4: 月度新增图表

**Given** 数据跨越多个自然月
**When** 前端请求 `/api/monthly_new`
**Then** 返回按月升序排列的月份及对应新增合计，图表中渲染为垂直条形图

*MUST*

### AC-5: 地理地图渲染

**Given** GeoJSON 文件存在且地区名称与数据可匹配
**When** 前端加载地图数据
**Then** `#chart-active-dist` 中渲染香港地图，各区按现存确诊数值着色，数值越高颜色越深

*MUST*

### AC-6: 地图降级行为

**Given** GeoJSON 文件不存在且外部 CDN 不可达
**When** 前端尝试加载地图
**Then** 地图区域显示地区名称与数值的纯文本列表，不抛出前端错误

*SHOULD*

### AC-7: 风险分布饼图

**Given** Excel 含风险等级列且数据非空
**When** 前端请求 `/api/risk_distribution`
**Then** 返回各风险等级及其地区数量，图表中渲染为环形饼图

*MUST*

### AC-8: 风险分布空数据降级

**Given** Excel 不含风险等级列
**When** 前端请求 `/api/risk_distribution`
**Then** 返回 `{"levels": [], "values": []}`，图表区域显示"暂无风险等级字段数据"

*MUST*

### AC-9: 各区最新日新增 Top 12

**Given** 最新统计日有多个地区的日新增数据
**When** 前端请求 `/api/district_daily_latest`
**Then** 返回按日新增降序排列的前 12 个地区及对应数值，图表中渲染为垂直条形图

*MUST*

### AC-10: 空数据概览降级

**Given** Excel 文件存在但无数据行
**When** 前端请求 `/api/overview`
**Then** 返回空对象 `{}`，前端 KPI 卡片显示为空或 0

*MUST*

### AC-11: 窗口缩放自适应

**Given** 页面已加载且图表已渲染
**When** 用户调整浏览器窗口大小
**Then** 所有 ECharts 图表自动重绘并适配新窗口尺寸，不出现图表溢出或截断

*MUST*

### AC-12: 独立趋势图生成

**Given** Excel 文件存在且含有效数据
**When** 用户执行 `python generate_confirmed_trends.py`
**Then** 生成 `static/images/confirmed_trends.png`，包含全港每日新增和累计确诊双轴折线图

*SHOULD*

### AC-13: GeoJSON 接口缺失响应

**Given** `static/geo/hk_districts_full.json` 文件不存在
**When** 前端请求 `/api/hk_districts_geo`
**Then** 返回 HTTP 404，响应体包含 `{"error": "geo file missing"}` 及文件路径

*MUST*

### AC-14: 同名同日数据去重

**Given** 数据中同一地区在同一日期存在多条记录
**When** 前端请求 `/api/district_active_distribution`
**Then** 该地区在结果中仅出现一次，现存确诊值为多条记录中的最大值

*MUST*

### AC-15: 响应式布局

**Given** 用户在 1280px 宽度以下查看页面
**When** 页面加载完成
**Then** KPI 卡片以 3 列网格布局展示；在 720px 以下以 2 列布局展示；图表在 1100px 以下以单列布局展示

*SHOULD*
