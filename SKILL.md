---
name: trip-map-builder
description: >
  End-to-end trip planning: gather user constraints, build a reference
  itinerary, research locations and dining signals via 大众点评 + 小红书, then
  generate an interactive mobile-first map page (Leaflet + timeline) and
  optionally deploy to Vercel. Use when user asks to plan a trip, create an
  itinerary, research restaurants on 大众点评/小红书, build a trip map page, or
  says "行程规划", "行程地图", "多方案", "住宿推荐", "穿衣建议", "trip map", "plan my
  trip", "做个行程". Covers the full pipeline from scattered inputs (screenshots,
  wishlists) to a deployable reference map with a minute-level timeline, plan
  switcher, legend, labeled markers, navigation links, 小红书 links, lodging
  options, weather-driven packing list, payment info, and reservation buttons.
---

# Trip Map Builder

Three-phase pipeline: **Plan → Research → Build**.

The output is a **reference itinerary**, not a script the traveler must obey.
During the trip, weather, current location, fatigue, and hunger can override
the original plan.

## Shared memory

Before planning or building, read `~/.trip-map-builder/MEMORY.md` if it
exists. Use it only for durable traveler context:

- pace preference
- food and drink preferences
- budget habits
- payment and navigation preferences
- previously generated trip outputs
- recurring constraints and unresolved follow-ups

If the file does not exist, continue normally. Do not block on memory setup.

Do not store raw screenshots, passport data, booking codes, full chat logs, or
other sensitive/private material.

After each completed trip plan, research pass, or map build, update
`~/.trip-map-builder/MEMORY.md` with only durable facts:

```md
# Trip Map Builder Memory

## Traveler Defaults
- Departure city:
- Pace:
- Food preferences:
- Budget habits:
- Payment preference:
- Navigation preference:
- Language preference:

## Past Trips
| Trip | Dates | Destination | Output | Notes |
|------|-------|-------------|--------|-------|

## Reusable Preferences
-

## Open Threads
-
```

## Phase 1: Plan the itinerary

Read `references/trip-planning.md` for the full methodology.

Core sequence:

1. **Extract hard constraints** — dates, flight times, terminals, hotel location, budget
2. **Group user's wishlist** — city-easy / needs-reservation / far-suburbs / pass-through
3. **Cut high-risk items first** — too far, holiday-crowded, weather-dependent. Say what was cut and why.
4. **Arrange by area** — one main area per day, first day light, last day close to airport
5. **Write a minute-level timeline** — every stop needs 几点出门 / 坐哪条线几站 / 几点到 / 几点走
6. **Fill in meals** — daily-area candidates first, 大众点评 + 小红书 signals second, fame last.
7. **Recommend lodging** — 2–3 areas with metro distance, price band, and the budget math
8. **Add weather & clothing** — real current reading + forecast, then a packing list
9. **Offer 2–3 plan variants** — see § 多方案 below
10. **Add tickets & transport** — only critical ones (museum tickets, airport transfer)
11. **Write reference doc** — conclusion first, then daily plan, weather-sensitive spots, meal areas, and what was cut

Key principles:
- Not everything the user listed fits. Delete for them.
- One area per day. One reservation-required spot per day max.
- Itineraries should be smooth, not packed.
- The plan gives coordinates for later adjustment; it does not pretend reality will follow the timeline.
- All user-facing questions follow the **4-beat format**: Re-ground → Simplify → Recommend → Options. See `references/trip-planning.md` § 用户交互 for examples and anti-patterns.

### 多方案（Multiple plans）

同一趟行程默认给 **2–3 个方案**，不是一个。方案之间必须有**结构差异**，不是同一路线换个说法：

| 变体轴 | 典型切法 |
|---|---|
| 距离 | 城市线 / 近郊线 / 远郊线（远郊另起一档，含包车成本） |
| 节奏 | 早出晚归的满档 / 每天能回酒店歇一脚的松档 |
| 主题 | 吃为主的 / 看海为主的 / 玩项目为主的 |

先自己按偏好**拟定** 2–3 个方案并写清各自代价（砍掉了什么、多花多少钱、几点起床），再用 4-beat 问用户要不要：

```
Re-ground:  行程规划, 方案选择
Simplify:   我按三种走法排了 A/B/C，差别在跑多远、几点起床。
Recommend:  第一次来建议 A（城市+海兼顾）；不想折腾选 C（不出市区）。

Options:
  A) A · 城市+海 — 一天南山、一天盐田，最远 60 分钟地铁
  B) B · 大鹏+欢乐谷 — 要包车，两个整天，门票 +¥690
  C) C · 纯市区 — 最远 30 分钟，09:00 起床，全部免费
```

写进输出时，每个方案都要有：**名称 + 一句话差异 + 预算 + 最早起床时间 + 推荐住宿**。

### 时间轴（Timeline）

每天除了「行程列表」，还要一条**分钟级时间轴**。每行至少包含：

- 起止时间（`09:00–10:10`）
- 动作类型（地铁 / 打车 / 步行 / 吃 / 景点 / 住宿）
- 具体换乘（`1 号线 11 站 → 桃园，换 12 号线 1 站`）
- 花费与时长（`打车 15 分钟，约 ¥25`）

时间轴是给人照着执行的，不是给人读的散文。凡是「去某地」都要写清怎么去、多久、几点到。

### 住宿（Lodging）

用户给预算时，先把口径说清：

- 预算是**整间房**还是**三人总价**？两者差一倍
- 达标区间在哪里？把「拿不到的价位」直接讲明（例：CBD ¥200 拿不到，普遍 400+）
- 给 2–3 个住区，每个写：地铁线 + 站到酒店步行几分钟 + 价格带 + 一个缺点
- 写完说明：换住区后动线怎么变（「路线不变，每天多 15 分钟地铁」）

### 天气与穿衣（Weather）

先取**实况**再谈预期，不要凭季节印象写：

```bash
curl -s "https://wttr.in/<City>?format=j1"            # 实况 + 3 天
curl -s "https://api.open-meteo.com/v1/forecast?..."   # 长预报（境内可能不通）
```

拿不到长预报时，用「实况 + 季节性规律」外推，并**明说是外推**。输出结构：

1. 实况数字（温度、湿度、天气）
2. 出行日期的预期区间（白天 / 夜间）
3. 分场景清单：打底 / 外套 / 海边 / 室内 / 鞋
4. 一句「不用带什么」（例：不用带羽绒服）

### 地图必须真的能用

地图是交付物的一半，底图和标注不成立就等于没做。硬要求：

- **底图选境内可达的源**——CartoDB / OSM 在境内实测直接超时，用高德
- **Retina 要清晰**——请求 512px 瓦片，别让 256px 被放大两倍
- **坐标系要对**——高德是 GCJ-02，WGS-84 点位必须转换，否则整体偏 300–600 米
- **图例 + 常驻名称标签**——用户要能不看点击就知道每个点叫什么
- **标签避让**——密集区域自动隐藏重叠标签，放大后恢复
- **时间轴要有视觉结构**——时间列 + 彩色节点 + 连线，不是一段纯文字

实现细节（含可复制的瓦片 URL、转换函数、图例和避让代码）见 `references/map-quality.md`。

## Phase 2: Research via 大众点评 + 小红书

Read `references/dianping-research.md` for the 大众点评 OpenCLI workflow.
Read `references/xhs-research.md` for the 小红书 OpenCLI + CDP workflow.

For restaurants, use 大众点评 as the main Chinese dining signal for taste,
queue risk, value, and obvious traps. Use 小红书 to supplement atmosphere,
recent experience, photo-worthiness, and soft warnings. Do not bend a whole
day around a famous restaurant unless it is already on the route.

### 大众点评拿不到时：转小红书逐店检索

大众点评的网页端和 m 站都会强制跳登录，拿不到评分。**不要在拿不到评分的情况下凭名气硬写结论**，改为逐店检索小红书：

1. 搜索词用 `店名 + 城市 + 评价` 或 `店名 + 城市 + 测评`，比只搜店名命中率高
2. 看标题判断信号强度：「真实测评」「点单红黑榜」「人均 XX」是好信号；「避雷 XXX」必须点进去看理由
3. 打开 1 篇最相关的详情页，抓 `#detail-desc`，提取：人均、必点菜、缺点、排队规律
4. 每家压成 2–3 句结论，写进地图卡片的 `detail` 字段，并保留 `xhsKeyword` 供用户点开原文

**必须保留负面信息**。真实的缺点是这份文档可信的来源：人均 132 还要加 10% 服务费、某道菜偏咸、螺有沙粒——这些比「很好吃」有用得多。

如果检索结果推翻了原定主推（评分低、差评集中在同一点），**换店，并在文档里说明为什么换**。

小红书 core sequence:

1. Launch Chrome with `--remote-debugging-port=9223`
2. Connect via OpenCLI's `CDPBridge`
3. Navigate to `xiaohongshu.com/search_result?keyword=<encoded>` (never simulate input box)
4. Intercept `POST /api/sns/web/v1/search/notes` response
5. Pick top 2-3 notes by relevance, open detail pages
6. Extract via DOM: `#detail-title`, `#detail-desc`, `.author-container .username`
7. Compress to one decision-useful sentence per store, write back to local `.md`

Filtering rules:
- Keep: specific store name, address, dish, personal experience, repeated keywords
- Drop: generic area roundups, reposts, pure emotion, "氛围很好" x3
- Output: store name + one representative link + 2-3 sentence verdict

## Phase 3: Build the map page

1. Copy `assets/template.html` → `index.html`
2. Fill `HOTEL` object and `DAYS` array with structured data from Phase 1+2
3. Each location needs: name, lat/lng, type, time, desc; optional: budget, detail, pay, xhs, reserve, gmap
4. Each day needs `steps: [{t, k, v}]` — the visual timeline (see `references/map-quality.md`)
5. Fill `overviewContent()` with trip summary, plan picker, lodging, weather, payment warnings
6. Multiple plans: keep one `PLANS` object keyed by plan, and let the tab bar cycle between them
7. Apply design system — default template uses Apple style, but can switch to any style from awesome-design-md

Ship the map only after checking these in a real browser: 底图瓦片全部加载成功、每个点有名称、图例可见、切换方案后不残留上一个方案的标签。

Location types: `food` | `spot` | `drink` | `hotel` | `transport`

Payment chip values: `1` = confirmed yes (green), `0.5` = maybe (orange), omit = not shown

### Design system (optional)

Default template uses Apple design system (SF Pro, light theme, frosted glass).

To use a different style, grab a `DESIGN.md` from [awesome-design-md](https://github.com/VoltAgent/awesome-design-md):

```bash
# Browse available design systems
# Apple, Vercel, Linear, Stripe, Notion, Airbnb, Nike, Spotify, etc.
curl -O https://raw.githubusercontent.com/VoltAgent/awesome-design-md/main/design-md/<brand>/DESIGN.md
```

Then adjust `template.html`'s `:root` CSS variables (colors, fonts, spacing, border-radius) to match the chosen DESIGN.md tokens.

### Deploy (optional)

```bash
git init && git add . && git commit -m "trip map"
gh repo create REPO --public --source=. --push
# Import from vercel.com/new — auto-deploys on push
```

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| [OpenCLI](https://github.com/jackwener/OpenCLI) | 大众点评 adapter + 小红书调研 | `npm install -g @jackwener/opencli` |
| Chrome/Chromium | 浏览器 + 远程调试 | 已有 |
| [Leaflet.js](https://leafletjs.com) | 地图渲染（CDN 引入，无需安装） | template.html 内置 |
| [gh CLI](https://cli.github.com) | GitHub 仓库创建（可选） | `brew install gh` |

## Resources

- `references/trip-planning.md` — itinerary methodology: 多方案设计、分钟级时间轴、住宿推荐、天气与穿衣、选点原则、常见坑
- `references/map-quality.md` — 底图选型（境内可达 + Retina）、GCJ-02 坐标转换、图例、常驻标签与避让、视觉化时间轴
- `references/dianping-research.md` — 大众点评 workflow，以及登录墙下转小红书逐店检索的替代路径
- `references/xhs-research.md` — OpenCLI installation, Chrome CDP setup, 小红书 search workflow, API details, filtering criteria
- `assets/template.html` — single-file HTML map template (Leaflet + Apple design system)
- [awesome-design-md](https://github.com/VoltAgent/awesome-design-md) — 60+ brand design systems (Apple, Vercel, Stripe, Linear, etc.) for alternative styling
