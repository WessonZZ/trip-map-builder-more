# 地图质量：底图、坐标、图例、标签、时间轴

地图是交付物的一半。这一页解决的问题是「点开页面，地图是白的 / 糊的 / 点偏了 / 不知道每个点叫啥」。

---

## 1. 底图：境内可达 + 清晰

**不要用 CartoDB / OSM。** 境内实测 `curl` 直接超时（返回码 `000`），页面上只会剩一句占位提示。

用高德瓦片，无需 Key：

```js
// 高清（512px 瓦片，Retina 下 1:1，不糊）
const TILE_URL = 'https://wprd0{s}.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scl=2&style=7';

// 标准（256px，兜底用）
const TILE_URL_SD = 'https://webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}';
```

关键参数：

| 参数 | 作用 |
|---|---|
| `scl=2` | 返回 512×512。渲染时仍是 256 CSS px，Retina 屏上正好 1:1 映射到物理像素 |
| `style=7` | 路网 + 注记（带中文地名） |
| `style=6` | 卫星图（`webst0{s}` 域名） |
| `subdomains: '1234'` | 对应 `wprd01`–`wprd04` |

加兜底，避免 HD 挂了整页空白：

```js
let hdFailed = 0;
baseLayer.on('tileerror', () => {
  if (++hdFailed === 4) {
    map.removeLayer(baseLayer);
    L.tileLayer(TILE_URL_SD, { subdomains: '1234', maxZoom: 18 }).addTo(map);
  }
});
```

验证方式：`document.querySelectorAll('.leaflet-tile')` 里 `naturalWidth === 512` 且 `complete === true` 才是真的加载了。

---

## 2. 坐标系：必须转换

高德瓦片是 **GCJ-02**，行程数据通常是 **WGS-84**。不转换，整张图偏移 **300–600 米**——餐厅会落到隔壁街区，而且是那种「看起来没错」的错。

```js
function _tfLat(x, y) {
  let r = -100 + 2*x + 3*y + 0.2*y*y + 0.1*x*y + 0.2*Math.sqrt(Math.abs(x));
  r += (20*Math.sin(6*x*Math.PI) + 20*Math.sin(2*x*Math.PI)) * 2/3;
  r += (20*Math.sin(y*Math.PI) + 40*Math.sin(y/3*Math.PI)) * 2/3;
  r += (160*Math.sin(y/12*Math.PI) + 320*Math.sin(y*Math.PI/30)) * 2/3;
  return r;
}
function _tfLng(x, y) {
  let r = 300 + x + 2*y + 0.1*x*x + 0.1*x*y + 0.1*Math.sqrt(Math.abs(x));
  r += (20*Math.sin(6*x*Math.PI) + 20*Math.sin(2*x*Math.PI)) * 2/3;
  r += (20*Math.sin(x*Math.PI) + 40*Math.sin(x/3*Math.PI)) * 2/3;
  r += (150*Math.sin(x/12*Math.PI) + 300*Math.sin(x/30*Math.PI)) * 2/3;
  return r;
}
function toGcj(lat, lng) {
  const a = 6378245.0, ee = 0.00669342162296594323, PI = Math.PI;
  let dLat = _tfLat(lng - 105, lat - 35), dLng = _tfLng(lng - 105, lat - 35);
  const radLat = lat / 180 * PI;
  const magic = 1 - ee * Math.sin(radLat) ** 2;
  const sqrtMagic = Math.sqrt(magic);
  dLat = (dLat * 180) / ((a * (1 - ee)) / (magic * sqrtMagic) * PI);
  dLng = (dLng * 180) / (a / sqrtMagic * Math.cos(radLat) * PI);
  return [lat + dLat, lng + dLng];
}
const P = (lat, lng) => toGcj(lat, lng);
```

**只转换绘制用的坐标，导航链接保持 WGS-84。** 高德 URI 带 `coordinate=wgs84` 会自己转，转两次就偏回去了。

```js
// 绘制
L.marker(P(l.lat, l.lng), ...)
map.fitBounds(coords.map(c => P(...c)))
// 导航——传原始值
`https://uri.amap.com/marker?position=${lng},${lat}&coordinate=wgs84`
```

坐标系对不对，用位移量自查：深圳一带应该在 **lat −0.003、lng +0.005** 左右（约 330 米 / 510 米）。差一个数量级说明公式写错了。

---

## 3. 图例与名称标签

用户第一句抱怨通常是「我不知道地图上哪个点是什么」。两个都要做。

### 图例（左下角，紧凑两行）

上排是每天的路线颜色，下排是类型图标：

```js
const LegendControl = L.Control.extend({
  options: { position: 'bottomleft' },
  onAdd() { const d = L.DomUtil.create('div', 'map-legend'); L.DomEvent.disableClickPropagation(d); return d; }
});
new LegendControl().addTo(map);

function updateLegend() {
  const el = document.querySelector('.map-legend');
  if (!el) return;
  el.innerHTML =
    `<div class="lg-row">${DAYS.slice(1).map(d =>
      `<span class="lg-chip"><i style="background:${d.color}"></i>${d.label}</span>`).join('')}</div>` +
    `<div class="lg-row"><span class="lg-chip"><b>🍽️</b>吃</span><span class="lg-chip"><b>📍</b>景点</span>` +
    `<span class="lg-chip"><b>🏨</b>住宿</span><span class="lg-chip"><b>🚃</b>车站</span></div>`;
}
```

切换方案后要重新调用 `updateLegend()`，否则图例还停在旧方案上。

### 常驻标签

```css
.map-label {
  background: rgba(255,255,255,.94) !important; border: none !important;
  border-radius: 6px !important; box-shadow: 0 1px 3px rgba(0,0,0,.18) !important;
  padding: 1px 5px !important; font-size: 10px !important;
  font-weight: 600 !important; white-space: nowrap;
}
.map-label::before { display: none !important; }   /* 去掉 Leaflet 的小箭头 */
```

```js
m.bindTooltip(l.name.replace(/（.*?）|[（(].*?[）)]/g, ''), {
  permanent: true, direction: 'right', offset: [12, 0],
  className: 'map-label', opacity: 1
});
```

去掉括号里的补充说明（`蘩楼 · 早茶（振华路店）` → `蘩楼 · 早茶`），否则标签太长互相压。

标签是绝对定位的，靠近地图边缘时会溢出容器，把整页撑出横向滚动。给地图容器加裁剪：

```css
#map { overflow: hidden; }
```

同理，页面级再加一条 `body { overflow-x: hidden; }` 兜底。手机上出现「整页能左右拖」基本都是这个原因。

### 标签避让

密集区域（一个商圈 3–4 个点）标签会叠成一坨。渲染后跑一次碰撞检测，重叠的隐藏，放大后恢复：

```js
function declutterLabels() {
  const kept = [];
  [...document.querySelectorAll('.map-label')].forEach(el => {
    el.style.display = '';
    const r = el.getBoundingClientRect();
    if (!r.width) return;
    const hit = kept.some(b => !(r.right < b.left || r.left > b.right || r.bottom < b.top || r.top > b.bottom));
    if (hit) el.style.display = 'none'; else kept.push(r);
  });
}
map.on('zoomend moveend', () => { if (cur !== 0) setTimeout(declutterLabels, 30); });
```

### 切换方案后的残留标签

`map.removeLayer(marker)` 不一定会带走 permanent tooltip，切换方案后旧标签会留在图上。三件事一起做：

```js
let markerTooltips = [], hotelMarker = null;

m.on('add', () => {
  const t = m.getTooltip();
  if (t && !markerTooltips.includes(t)) markerTooltips.push(t);
});

function clearMap() {
  markerTooltips.forEach(t => { try { map.removeLayer(t); } catch(e){} });
  markers.forEach(m => map.removeLayer(m));
  polylines.forEach(p => map.removeLayer(p));
  markerTooltips = []; markers = []; polylines = [];
  // 兜底：清掉残留 DOM，但保留酒店标签
  document.querySelectorAll('.map-label:not(.map-label-hotel)').forEach(e => e.remove());
  if (hotelMarker) setTimeout(() => { try { hotelMarker.openTooltip(); } catch(e){} }, 0);
}
```

酒店标签加独立 class `map-label-hotel`，否则会被一起清掉。

---

## 4. 时间轴：结构化数据，不是一段文字

数据用数组，不要塞一个多行字符串——后者没法做视觉结构。

```js
{ id: 2, label: 'D2 南山', date: '12/3 周四', color: '#34c759',
  steps: [
    { t:'08:50', k:'metro', v:'1 号线 2 站 → 华强路（约 6 分钟）' },
    { t:'09:00', k:'eat',   v:'蘩楼早茶，约 70 分钟' },
    { t:'14:05', k:'taxi',  v:'打车 12 分钟，约 ¥20 → 深圳湾公园' }
  ],
  locations: [ /* 卡片数据 */ ] }
```

`k` 的取值决定节点颜色和图标：

```js
const KICON = {
  walk:  { e: '🚶', c: '#af52de' },
  metro: { e: '🚇', c: '#007aff' },
  taxi:  { e: '🚕', c: '#ff9500' },
  eat:   { e: '🍜', c: '#ff3b30' },
  spot:  { e: '📍', c: '#34c759' },
  hotel: { e: '🏨', c: '#8e8e93' }
};
```

渲染成三列：**时间列（等宽数字右对齐）+ 节点列（圆点 + 竖向连线）+ 内容列**。

```js
d.steps.map((s, i) => {
  const k = KICON[s.k];
  return `<div class="rt-item">
    <div class="rt-time">${s.t}</div>
    <div class="rt-rail">
      <div class="rt-dot" style="background:${k.c}">${k.e}</div>
      ${i < d.steps.length - 1 ? '<div class="rt-line"></div>' : ''}
    </div>
    <div class="rt-body">${s.v}</div>
  </div>`;
}).join('')
```

```css
.rt-item { display: flex; gap: 10px; }
.rt-time { flex: 0 0 46px; font-size: 12px; font-weight: 700; color: var(--text-secondary);
           font-variant-numeric: tabular-nums; text-align: right; padding-top: 5px; }
.rt-rail { flex: 0 0 24px; display: flex; flex-direction: column; align-items: center; }
.rt-dot  { width: 24px; height: 24px; border-radius: 50%; display: flex;
           align-items: center; justify-content: center; font-size: 12px; }
.rt-line { flex: 1; width: 2px; background: var(--separator); }
.rt-body { flex: 1; font-size: 14px; line-height: 1.5; padding: 4px 0 18px; }
```

一行写一件事。凡是移动都要带「怎么走 + 多久 + 几点到」。

---

## 5. 票务标识

只写「需预约」不够——要在三个地方同时出现，用户才不会漏：

1. **卡片徽章**：名称前一个彩色小标签
2. **地图弹窗**：点开点位能看到
3. **总览清单**：按天列出所有需要提前办的事

```js
const BOOK = {
  reserve: { label: '需预约',     c: '#ff9500', act: '免费但要预约，提前 1–3 天在官方渠道约（周一闭馆）' },
  ticket:  { label: '需购票',     c: '#007aff', act: '提前在官方小程序买票，通常更便宜也省排队' },
  call:    { label: '需提前联系', c: '#af52de', act: '提前一天电话确认档期' },
  maybe:   { label: '旺季需预约', c: '#8e8e93', act: '12 月一般不用，出发前在官方公众号确认一次' },
  queue:   { label: '现场排队',   c: '#ff3b30', act: '不接受预约，只能现场取号，尽量开门就到' }
};
```

时间轴对应那一步也加一个小 chip（`{t, k, v, b:'ticket'}`），扫一眼就知道哪几步要提前动手。

总览里的清单直接从 `DAYS` 生成，不要手写——手写会随行程改动而失效：

```js
const rows = DAYS.slice(1).flatMap(d => d.locations.filter(l => l.booking).map(l => ({ d, l })));
```

## 5. 交付前自查

在真浏览器里逐条确认，不要只靠读代码：

| 检查项 | 怎么查 |
|---|---|
| 瓦片加载 | `.leaflet-tile` 里 `complete && naturalWidth === 512` 的数量等于总数 |
| 点位不偏 | 挑一个已知地标，标记应落在它上面 |
| 名称可见 | `.map-label` 数量 ≥ 当天点位数（减去被避让的） |
| 图例正确 | 切换方案后图例里的日期跟着变 |
| 无残留 | 切方案后 `.map-label` 不含上一个方案的点位 |
| 缩放恢复 | 放大一级后，原本被隐藏的标签重新出现 |
