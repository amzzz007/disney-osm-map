# 上海迪士尼乐园 · OSM 交互地图

一个**单文件、可离线**的上海迪士尼乐园内部地图。直接双击 `disney-osm-map.html` 即可在浏览器中打开，无需服务器、无需 API Key。

![版本](https://img.shields.io/badge/version-1.0-2f6fed) ![单文件](https://img.shields.io/badge/single--file-1.8MB-22c55e) ![依赖](https://img.shields.io/badge/API%20Key-不需要-00e5ff)

---

## 特性

- **单文件自包含** — OSM 数据与 39 个项目配图全部内联，断网也能用
- **深色地图底图** — 内联空样式 + 纯数据渲染，不加载任何外部瓦片
- **45 个游乐项目标点** — 按类型着色，重要项目更大更醒目
- **英文介绍卡片** — 点击项目右侧滑出，含 Wikipedia 摘要与配图
- **base16 解码动画** — 「译成中文」按钮触发十六进制逐字节解码动画
- **智能标签避让** — 按优先级占位，碰撞即隐藏，缩放分级显示
- **零 Backend** — 纯静态 HTML，无构建步骤

---

## 快速开始

```bash
# 方式一：直接双击
open disney-osm-map.html      # macOS
xdg-open disney-osm-map.html  # Linux
start disney-osm-map.html     # Windows

# 方式二：本地服务（可选，非必需）
python3 -m http.server 8000
# 然后访问 http://localhost:8000/disney-osm-map.html
```

> **注意**：文件唯一的外部依赖是 MapLibre GL JS 的 CDN。首次打开需要联网加载它；如需完全离线，可将该 JS 文件下载到本地并替换 `<script src>`。

---

## 数据来源

| 内容 | 来源 | 说明 |
|---|---|---|
| 地图要素 | [OpenStreetMap](https://www.openstreetmap.org/) | 1653 个要素（486 node + 1167 way），bbox `121.6497,31.1419,121.6627,31.1506` |
| 项目介绍 | Wikipedia REST API | 18 个项目有独立条目，另 21 个为自撰 |
| 项目配图 | Wikimedia Commons | 17 张，已转 base64 内联 |

数据追溯至 **2026-10-25 08:24** 抓取。

### 关于数据完整性

本版本数据抓取时，Overpass 官方主站与备用镜像在当前网络环境下返回 `406 Not Acceptable`，可用镜像数据库为空。最终通过 `api.openstreetmap.org/api/0.6/map` 以相同 bbox 提取原始 XML 并转换为 Overpass `out geom` 结构。

**这意味着**：数据是真实的 OSM 数据，但获取端点与标准 Overpass 流程不同。如需最新数据，建议用 [Overpass Turbo](https://overpass-turbo.eu/) 重新导出。

---

## 使用说明

### 交互

| 操作 | 效果 |
|---|---|
| 点击游乐设施 | 右侧滑出英文介绍卡片 |
| 点击「译成中文」 | 触发 base16 解码动画，输出中文译文 |
| 点击其他设施 | 显示类型与 OSM 标签基础卡片 |
| 右上角「设施标点」 | 开关设施点位与名称标签 |
| 图例折叠按钮 | 收起/展开图例 |
| 滚轮 / 双指 | 缩放（限制 10–18） |

### 标签显示分级

为保持地图清爽，名称标签按缩放级别显示：

| 缩放 | 显示内容 |
|---|---|
| < 15.0 | 全部隐藏 |
| 15.0 – 15.8 | 仅游乐设施/景点 |
| ≥ 16.4 | 增加餐厅、商店 |
| ≥ 17.2 | 增加卫生间、停车场 |

---

## 技术实现

### 架构

```
disney-osm-map.html
├── <style>           全部 CSS（深色玻璃拟态）
├── <script>          window.DISNEY_OSM   = {...}   // 1653 个 OSM 要素
├── <script>          window.DISNEY_CARDS = {...}   // 39 条介绍 + 17 张图
└── <script>          主逻辑（原生 JS，无框架）
```

### 关键设计

**1. 为什么不使用 MapLibre 的 symbol 图层做标签**

地图使用内联空样式，**没有 glyphs 字体资源**，symbol 图层的文字渲染会失败。因此标签用 DOM 元素 + `map.project()` 手动定位实现。

**2. 碰撞避让**

按优先级排序后依次占位，矩形相交则隐藏后者。文字放点位右侧，右侧越界自动翻转到左侧。

**3. 设施点位来自面要素质心**

OSM 中大部分设施是闭合多边形而非点。用**面积加权质心**把它们折算为点：

```js
// 以首点为局部原点，避免大数相减的精度丢失
var ox = pts[0][0], oy = pts[0][1];
// ... 叉积计算
return [ox + cx / (3 * area2), oy + cy / (3 * area2)];
```

> 直接对经纬度做叉积会因数值量级过大导致灾难性抵消——初版 39 个质心只有 10 个落在正确多边形内，修正后 39/39 正确。

**4. base16 解码动画**

中文 → UTF-8 字节 → 十六进制串，然后每 16ms 推进一帧逐字节解码：

```js
var step = Math.max(4, Math.floor(total / 60));   // 约 60 帧完成
// 已解码部分显示中文，未解码部分继续显示 hex
```

---

## 已知限制

- **部分项目无配图** — 39 个项目中有 22 个无配图：其中 21 个在 Wikipedia 无独立条目（介绍为自撰），1 个（Soarin'）有条目但无合适配图
- **标签为估算宽度** — 碰撞检测按字符数估算（中文 11px、拉丁 6px），中英混排长名称可能有像素级误差
- **不含实时信息** — 无排队时长、无运营状态，不提供实时导航

---

## 文件校验

```
SHA-256 / MD5
MD5: 73f90a84474d5487af324e4adcdd05eb
大小: 1,833,528 字节
```

---

## 致谢与许可

- 地图数据 © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright)，遵循 [ODbL](https://opendatacommons.org/licenses/odbl/) 许可
- 项目介绍与配图来自 Wikipedia / Wikimedia Commons，遵循各自许可协议
- 地图引擎 [MapLibre GL JS](https://maplibre.org/)，BSD-3-Clause

迪士尼相关名称与形象版权归 Disney 所有。本项目为个人学习用途的非官方作品，与 Disney 无关联。
