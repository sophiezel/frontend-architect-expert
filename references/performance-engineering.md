# 运行时性能工程 (Runtime Performance Engineering)

> 蒸馏来源: Philip Walton (Google), Addy Osmani (Google),
> Google Chrome DevTools team, web.dev performance guides,
> Web Vitals RFCs, W3C Performance Working Group

---

## 目录

- [一、Core Web Vitals 深度解析](#一core-web-vitals-深度解析)
  - [1.1 LCP — 最大内容绘制](#11-lcp--最大内容绘制)
  - [1.2 CLS — 累积布局偏移](#12-cls--累积布局偏移)
  - [1.3 INP — 与下一次绘制的交互](#13-inp--与下一次绘制的交互)
- [二、Performance APIs 全景](#二performance-apis-全景)
- [三、RUM 性能监控架构](#三rum-性能监控架构)
- [四、性能预算与 CI 门禁](#四性能预算与-ci-门禁)
- [五、内存工程](#五内存工程)
- [六、帧率与 Jank 诊断](#六帧率与-jank-诊断)
- [七、Bundle 与加载性能](#七bundle-与加载性能)
- [八、性能反模式清单](#八性能反模式清单)

---

## 一、Core Web Vitals 深度解析

### 1.1 LCP — 最大内容绘制

**阈值**: Good < 2.5s / Needs Improvement < 4.0s / Poor >= 4.0s
**测量对象**: 视口内最大可见内容元素（图片、视频、包含文本的块级元素、带 `url()` 的背景图）
**采集 API**: `PerformanceObserver` 观测 `type: 'largest-contentful-paint'`
**CWV p75**: 按 75 分位聚合，非平均值

```
LCP 时间线（attribution 分解）:
  ┌────────────┬─────────────────────┬──────────────────────┬─────────────────────┐
  │  TTFB      │ Resource Load Delay │ Resource Load        │ Element Render      │
  │            │ (发现→开始加载)       │ Duration (下载耗时)    │ Delay (渲染延迟)      │
  └────────────┴─────────────────────┴──────────────────────┴─────────────────────┘
       ↓                  ↓                     ↓                      ↓
  startTime →       发现资源时间          responseEnd              renderTime
  navigationStart                        - requestStart           - loadEnd
```

#### LCP Attribution 深入

```typescript
// 使用 web-vitals 库的 attribution 功能 (v3+)
import {onLCP, type LCPAttribution} from 'web-vitals/attribution'

onLCP((metric) => {
  const attr = metric.attribution as LCPAttribution

  const diagnostics = {
    // 可归因为具体元素
    element: attr.element,      // HTMLImageElement | HTMLVideoElement | HTMLElement
    url: attr.url,              // 图片 URL（如果是 <img>）

    // 子阶段分解（毫秒）
    timeToFirstByte: attr.timeToFirstByte,
    resourceLoadDelay: attr.resourceLoadDelay,
    resourceLoadDuration: attr.resourceLoadDuration,
    elementRenderDelay: attr.elementRenderDelay,

    // 资源发现相关的 Resource Timing 条目
    resourceEntry: attr.navigationEntry,

    // 元素类型
    elementType: attr.elementType,  // 'image' | 'video' | 'text'
  }
}, {reportAllChanges: false}) // 只报告最终的 LCP
```

#### LCP 优化决策树

```
LCP > 2.5s 时：
1. TTFB 高 (>800ms) → 检查 SSR/CDN/边缘缓存/后端响应时间
2. Resource Load Delay 高 → 图片发现晚于 scan/preload scanner，考虑 preload
   - <img> 出现在 JS 渲染后？→ 考虑 SSR/静态导出 / 用 <link rel="preload">
   - CSS background-image？→ 非 LCP 候选，改用 <img> 或 preload
3. Resource Load Duration 高 → 图片太大 / CDN 慢 / 未使用现代格式
   - 用 AVIF/WebP + srcset + sizes
   - 放到同域名 CDN 减少连接建立
4. Element Render Delay 高 → 渲染被 JS/CSS 阻塞
   - render-blocking CSS 延迟了首次渲染
   - 长任务阻塞了主线程，阻止渲染管线推进
```

#### LCP 常见陷阱

```typescript
// ❌ 错误：用 load 事件测量 LCP — load 可能延迟数秒
window.addEventListener('load', () => {
  const lcp = performance.getEntriesByType('largest-contentful-paint').pop()
  // 可能已经错过后续更大的 LCP 候选
})

// ✅ 正确：用 PerformanceObserver
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries()
  const lastEntry = entries[entries.length - 1] // 只取最后一个
  console.log('LCP:', lastEntry.startTime)
})
observer.observe({type: 'largest-contentful-paint', buffered: true})

// ❌ 异步加载的图片不触发新 LCP？
// LCP 只考虑视口内元素，且元素一旦离开视口被移除，不会再触发新 LCP
// 但用户滚动后新进入视口的图片不会成为 LCP 候选
```

#### LCP 优化实操

```html
<!-- 1. Preload LCP 图片 -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">

<!-- 2. fetchpriority 提高优先级 -->
<img src="/hero.webp" fetchpriority="high" alt="hero"
     width="1200" height="675" decoding="async">

<!-- 3. AVIF + WebP 回退链 -->
<picture>
  <source srcset="/hero.avif" type="image/avif">
  <source srcset="/hero.webp" type="image/webp">
  <img src="/hero.jpg" alt="hero" width="1200" height="675">
</picture>
```

```css
/* 4. 将 LCP 文本的 web font 改为内联 font-display: block 很短 */
@font-face {
  font-family: 'Heading';
  src: url('/heading.woff2') format('woff2');
  font-display: swap; /* 或 optional，避免 FOIT 拖慢 LCP */
}
```

---

### 1.2 CLS — 累积布局偏移

**阈值**: Good < 0.1 / Needs Improvement < 0.25 / Poor >= 0.25
**测量公式**: `CLS = Σ(impact_fraction * distance_fraction)`
**Session Window 机制**: 1 秒间隔 / 最大 5 秒窗口 / 取窗口内所有 shift 之和的最大值

```
Session Window 示意图:

|shift1: 0.04|  --300ms-- |shift2: 0.03| --1200ms gap-- [新窗口]
                                                          |shift3: 0.15| --200ms-- |shift4: 0.02|

Window 1: shift1 + shift2 = 0.07  (300ms gap < 1s → 合并)
Window 2: shift3 + shift4 = 0.17  (200ms gap < 1s → 合并)
最终 CLS = max(0.07, 0.17) = 0.17  (Not Good)
```

#### CLS 来源分类与修复

```typescript
// 来源 1: 无尺寸的图片
// ❌ 浏览器不知道图片高度，加载后撑开布局
<img src="/banner.jpg" alt="">

// ✅ 始终设置 width/height 或 aspect-ratio
<img src="/banner.jpg" alt="" width="800" height="400">
<img src="/banner.jpg" alt="" style="aspect-ratio: 2/1">

// 来源 2: 动态注入内容（广告、cookie banner）
// ✅ 预留空间
<div class="ad-slot" style="min-height: 250px">
  <!-- 广告 JS 注入 -->
</div>

// 来源 3: Web Font 导致 FOIT/FOUT
// ✅ font-display: swap + 与 fallback 一致的 metrics
@font-face {
  font-family: 'Body';
  src: url('/body.woff2') format('woff2');
  font-display: swap;
  /* 可选: size-adjust 让 fallback 和主字体占用相同宽度 */
  size-adjust: 98%;
  ascent-override: 92%;
}

// 来源 4: 无动画过渡的 DOM 插入
// ❌ 向列表顶部插入元素
list.insertBefore(newItem, list.firstChild)

// ✅ 保留滚动位置
const scrollTop = list.scrollTop
list.insertBefore(newItem, list.firstChild)
list.scrollTop = scrollTop + newItem.offsetHeight
```

#### CLS Attribution

```typescript
import {onCLS, type CLSAttribution} from 'web-vitals/attribution'

onCLS((metric) => {
  const attr = metric.attribution as CLSAttribution
  for (const entry of metric.entries) {
    // 每个 LayoutShift 条目包含：
    console.log({
      value: entry.value,                      // 本次偏移值
      sources: entry.sources?.map(s => ({
        node: s.node,                           // 导致偏移的 DOM 节点
        previousRect: s.previousRect,           // 偏移前的矩形
        currentRect: s.currentRect,             // 偏移后的矩形
      })),
    })
  }
  // attribution 聚合信息:
  console.log({
    largestShiftTarget: attr.largestShiftTarget,  // 最大偏移的节点
    largestShiftValue: attr.largestShiftValue,
    // 时间线聚合
    largestShiftTime: attr.largestShiftTime,
  })
}, {reportAllChanges: false}) // 只上报聚合后的最终 CLS
```

#### CLS 特别注意

```typescript
// ⚠️ 关键：发送 delta 而非 value
// web-vitals v3+ delta = 当前 metric value - 上次上报的 value
// 如果用 value 会导致重复上报累加值

onCLS(({delta, value, id}) => {
  // body 是 Analytics 事件体
  analytics.send({
    name: 'CLS',
    delta,   // ← 发送这个
    id,      // 同一页面内唯一 metric ID
  })
})

// CLS 是持续更新到页面生命周期结束
// 不在 load 事件时作为最终值，要持续到 pagehide
```

---

### 1.3 INP — 与下一次绘制的交互

**阈值**: Good < 200ms / Needs Improvement < 500ms / Poor >= 500ms
**替代 FID**: 2024 年 3 月正式替代 FID 成为 CWV 核心指标
**测量内容**: 页面生命周期内最慢的一次交互（而非 FID 只看第一次）

```
INP 分解:
┌───────────────┬───────────────────┬──────────────────────┐
│  Input Delay  │  Processing Time  │  Presentation Delay  │
│  输入延迟       │  处理时间           │  呈现延迟              │
│  (等待其他任务)  │  (事件处理器执行)    │  (等待下一次渲染帧)     │
└───────────────┴───────────────────┴──────────────────────┘
       ↓                 ↓                      ↓
事件入列时间        nextPaint 之前的       从处理完毕到
→ 处理器开始        同步+微任务执行        实际渲染帧提交
```

#### INP 测量

```typescript
import {onINP, type INPAttribution} from 'web-vitals/attribution'

onINP((metric) => {
  const attr = metric.attribution as INPAttribution

  console.log({
    // INP 归因到具体交互
    interactionType: attr.interactionType,     // 'pointer' | 'keyboard' | 'tap'
    eventTarget: attr.eventTarget,              // 交互的 DOM 元素
    eventType: attr.eventType,                  // 具体事件类型 (click, keydown 等)

    // 子阶段分解
    inputDelay: attr.inputDelay,
    processingDuration: attr.processingDuration,
    presentationDelay: attr.presentationDelay,

    // 长动画帧归因
    longAnimationFrameEntries: attr.longAnimationFrameEntries,
  })
})
```

#### INP 优化方向

```typescript
// 方向 1: 减少 Input Delay — 拆分长任务
// ❌ 单次处理 5000 个列表项
function processAll(items: Item[]) {
  items.forEach(item => expensiveTransform(item))
}

// ✅ 用 scheduler.postTask 或 requestIdleCallback
async function processInChunks(items: Item[], chunkSize = 50) {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize)
    // scheduler.yield() — 或在 polyfill 中用 setTimeout(0)
    await new Promise(r => (window as any).scheduler?.postTask
      ? (window as any).scheduler.postTask(() => {
          chunk.forEach(item => expensiveTransform(item))
        }, {priority: 'user-blocking'})
      : setTimeout(r, 0)
    )
  }
}

// 方向 2: 减少 Processing Time — 去抖/节流 + web worker
// ❌ 每次 input 触发重计算
inputEl.addEventListener('input', (e) => {
  doHeavySearch(e.target.value)
})

// ✅ 去抖 + 主线程只处理轻量操作
import {debounce} from 'lodash-es'  // 或自己实现

const debouncedSearch = debounce((value) => {
  // 如果确实需要主线程 API，才传回主线程
  requestAnimationFrame(() => updateResults(results))
}, 150)

inputEl.addEventListener('input', (e) => {
  scheduleIdleTask(() => {
    const results = searchWorker.postMessage({query: e.target.value})
  })
})

// 方向 3: 减少 Presentation Delay
// 确保交互后不触发同步 layout
function handleClick() {
  // ❌ 先读取布局属性触发强制同步布局
  const width = el.offsetWidth
  el.style.width = width + 10 + 'px'

  // ✅ 读取和写入分离，或全部用 transform
  el.style.transform = 'scale(1.1)'
}
```

---

## 二、Performance APIs 全景

### 2.1 PerformanceObserver — 统一观测接口

```typescript
// 通用模式：始终用 buffered: true 捕获注册前的条目
function observePerformance<T extends PerformanceEntry>(
  type: string,
  callback: (entries: T[]) => void,
): () => void {
  const observer = new PerformanceObserver((list) => {
    callback(list.getEntries() as T[])
  })
  observer.observe({type, buffered: true})
  return () => observer.disconnect()
}

// 使用示例 — 观测所有支持的 entry type
const disposers: Array<() => void> = []

// 1. LCP 观测
disposers.push(observePerformance('largest-contentful-paint', (entries) => {
  const lcp = entries[entries.length - 1]
  console.log(`LCP: ${lcp.startTime}ms, element:`, (lcp as any).element)
}))

// 2. CLS 观测 (LayoutShift)
disposers.push(observePerformance('layout-shift', (entries) => {
  for (const entry of entries) {
    if (!(entry as any).hadRecentInput) {
      console.log(`Layout Shift: ${entry.value}, startTime: ${entry.startTime}`)
    }
  }
}))

// 3. Long Tasks (>50ms)
disposers.push(observePerformance('longtask', (entries) => {
  for (const entry of entries) {
    console.log(`Long Task: ${entry.duration}ms`, (entry as any).attribution)
  }
}))

// 4. Long Animation Frames (Chrome 123+)
disposers.push(observePerformance('long-animation-frame', (entries) => {
  for (const entry of entries) {
    const lf = entry as any
    console.log({
      duration: lf.duration,
      blockingDuration: lf.blockingDuration,
      scripts: lf.scripts?.map((s: any) => ({
        name: s.name,
        duration: s.duration,
        invoker: s.invoker,
      })),
    })
  }
}))

// 5. Element Timing — LCP 候选的每个元素
disposers.push(observePerformance('element', (entries) => {
  for (const entry of entries) {
    console.log(`Element: ${entry.identifier}`, {
      renderTime: entry.startTime,
      loadTime: (entry as any).loadTime,
      element: (entry as any).element,
    })
  }
}))
```

### 2.2 Navigation Timing API

```typescript
// Navigation Timing Level 2 — 完整的时间分解
const [navEntry] = performance.getEntriesByType(
  'navigation'
) as PerformanceNavigationTiming[]

const timingBreakdown = {
  // 阶段分解（ms）
  dns: navEntry.domainLookupEnd - navEntry.domainLookupStart,
  tcp: navEntry.connectEnd - navEntry.connectStart,
  tls: navEntry.secureConnectionStart > 0
    ? navEntry.connectEnd - navEntry.secureConnectionStart
    : 0,
  request: navEntry.responseStart - navEntry.requestStart,
  response: navEntry.responseEnd - navEntry.responseStart,
  total: navEntry.responseEnd - navEntry.startTime,

  // 高级指标
  ttfb: navEntry.responseStart - navEntry.startTime,
  fcp: 0, // 需要从 paint entries 获取
  domInteractive: navEntry.domInteractive - navEntry.startTime,
  domComplete: navEntry.domComplete - navEntry.startTime,
  loadEventEnd: navEntry.loadEventEnd - navEntry.startTime,

  // 移交大小
  transferSize: navEntry.transferSize,
  encodedBodySize: navEntry.encodedBodySize,
  decodedBodySize: navEntry.decodedBodySize,

  // 服务端时间分解 (Server-Timing header)
  serverTiming: navEntry.serverTiming?.map(st => ({
    name: st.name,
    duration: st.duration,
    description: st.description,
  })),
}

// 判断 bfcache 恢复
if (navEntry.type === 'back_forward') {
  console.log('Page restored from bfcache — no full navigation')
}
```

### 2.3 Resource Timing API

```typescript
// 获取所有资源加载时间
const resources = performance.getEntriesByType('resource') as PerformanceResourceTiming[]

// 按资源类型聚合
const byType = resources.reduce((acc, entry) => {
  const type = entry.initiatorType // 'img'|'script'|'css'|'fetch'|'link'|...
  acc[type] = acc[type] || []
  acc[type].push({
    name: entry.name,
    duration: entry.duration,
    transferSize: entry.transferSize,
    // 分阶段
    dns: entry.domainLookupEnd - entry.domainLookupStart,
    tcp: entry.connectEnd - entry.connectStart,
    ttfb: entry.responseStart - entry.requestStart,
    download: entry.responseEnd - entry.responseStart,
    // 缓存判断 — transferSize 为 0 表示缓存命中
    fromCache: entry.transferSize === 0 && entry.decodedBodySize > 0,
  })
  return acc
}, {} as Record<string, Array<object>>)

// 发现有问题的资源
const slowResources = resources
  .filter(e => e.duration > 1000) // > 1s
  .map(e => ({
    url: e.name,
    duration: e.duration,
    type: e.initiatorType,
  }))
  .sort((a, b) => b.duration - a.duration)
```

### 2.4 User Timing API

```typescript
// 自定义性能标记
performance.mark('hydrate-start')

// 水合完成
performance.mark('hydrate-end')
performance.measure('hydration', 'hydrate-start', 'hydrate-end')

// 测量 React Suspense resolve 时间
performance.mark('suspense-start')
// ...resolve...
performance.mark('suspense-end')
performance.measure('suspense-resolution', 'suspense-start', 'suspense-end')

// 读取自定义测量
const measures = performance.getEntriesByType('measure')
for (const m of measures) {
  console.log(`${m.name}: ${m.duration.toFixed(2)}ms`)
}

// 用 PerformanceObserver 实时监听
const measureObserver = new PerformanceObserver((list) => {
  list.getEntriesByType('measure').forEach((entry) => {
    console.log(`[User Timing] ${entry.name}: ${entry.duration}ms`)
  })
})
measureObserver.observe({type: 'measure', buffered: true})
```

### 2.5 web-vitals 库 vs 原生 API 权衡

```typescript
// === 用 web-vitals 的优势 ===
// 1. 自动处理 bfcache 恢复
// 2. 自动去重（同一 metric ID 只报一次）
// 3. 正确的 CLS session window 聚合
// 4. INP attribution 内置
// 5. 5KB gzipped
import {onLCP, onCLS, onINP, onFCP, onTTFB} from 'web-vitals'

// === 自己用原生 API 的场景 ===
// 1. 需要自定义聚合逻辑（非 CWV 标准）
// 2. 不想引入外部依赖
// 3. 需要非标准的 entry type（如 long-animation-frame）

// === 推荐：业务代码用 web-vitals，底层库/框架插件用原生 API ===
```

---

## 三、RUM 性能监控架构

### 3.1 完整数据管线

```
┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────────┐    ┌─────────────┐
│ Browser │───→│ Collector│───→│ Transport │───→│ Aggregation  │───→│ Dashboard   │
│ (web-   │    │ (batch)  │    │ (beacon)  │    │ (ClickHouse │    │ (Grafana/   │
│  vitals)│    │          │    │           │    │  /BigQuery)  │    │  Datadog)   │
└─────────┘    └──────────┘    └───────────┘    └──────────────┘    └─────────────┘
```

### 3.2 Collector — 客户端采集层

```typescript
// collector.ts — 前端 RUM 采集器
import {onLCP, onCLS, onINP, onFCP, onTTFB, type Metric} from 'web-vitals'

interface RUMPayload {
  name: string
  metricType: 'web-vital' | 'custom'
  delta: number
  id: string
  value: number
  rating: 'good' | 'needs-improvement' | 'poor'
  page: string
  navigationType: string
  timestamp: number
  // SPA 相关
  softNavId?: string
  softNavName?: string
}

const BATCH_SIZE = 10
const BATCH_INTERVAL = 5000 // 5s
const MAX_BATCH_AGE = 30000  // 30s
let batch: RUMPayload[] = []
let batchTimer: ReturnType<typeof setTimeout> | null = null

// 获取当前路由（支持 SPA）
function getCurrentPage(): string {
  // React Router
  if (window.__REACT_ROUTER_STATE__) {
    return window.__REACT_ROUTER_STATE__.location.pathname
  }
  // Next.js App Router
  if ((window as any).__NEXT_DATA__) {
    return (window as any).__NEXT_DATA__.page
  }
  return window.location.pathname
}

// 获取软导航标识
let softNavId = ''
let softNavCount = 0

export function markSoftNavigation(name: string) {
  softNavId = `soft-nav-${++softNavCount}-${name}`
  // 提示 web-vitals 重新开始采集（web-vitals v4+）
}

// 构建 payload
function createPayload(metric: Metric, extra: Partial<RUMPayload> = {}): RUMPayload {
  return {
    name: metric.name,
    metricType: 'web-vital',
    delta: metric.delta,
    id: metric.id,
    value: metric.value,
    rating: metric.rating,
    page: getCurrentPage(),
    navigationType: metric.navigationType,
    timestamp: Date.now(),
    softNavId: softNavId || undefined,
    softNavName: undefined,
    ...extra,
  }
}

// 核心：CWV 回调
function reportWebVital(metric: Metric) {
  const payload = createPayload(metric)
  addToBatch(payload)
}

// 注册 web-vitals 采集
onCLS(reportWebVital)
onINP(reportWebVital)
onLCP(reportWebVital)
onFCP(reportWebVital)
onTTFB(reportWebVital)

// 自定义指标示例
export function reportCustomTiming(name: string, durationMs: number) {
  addToBatch({
    name,
    metricType: 'custom',
    delta: durationMs,
    id: `custom-${Date.now()}-${Math.random()}`,
    value: durationMs,
    rating: 'good',
    page: getCurrentPage(),
    navigationType: 'navigate',
    timestamp: Date.now(),
  })
}
```

### 3.3 Transport — 传输层

```typescript
// transport.ts — 用 sendBeacon 可靠传输
function addToBatch(payload: RUMPayload) {
  batch.push(payload)

  if (batch.length >= BATCH_SIZE) {
    flush()
  } else if (!batchTimer) {
    batchTimer = setTimeout(flush, BATCH_INTERVAL)
  }
}

function flush() {
  if (batch.length === 0) return
  const snapshot = batch
  batch = []
  if (batchTimer) {
    clearTimeout(batchTimer)
    batchTimer = null
  }
  sendBatch(snapshot)
}

function sendBatch(payloads: RUMPayload[]) {
  // PII 清洗 — 移除 URL 中的敏感参数
  for (const p of payloads) {
    p.page = scrubPII(p.page)
  }

  const blob = new Blob([JSON.stringify({events: payloads})], {
    type: 'application/json',
  })

  // sendBeacon 不阻塞页面卸载，在 bfcache 和 pagehide 下都能工作
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/rum/collect', blob)
  } else {
    // 降级：fetch + keepalive
    fetch('/api/rum/collect', {
      method: 'POST',
      body: blob,
      keepalive: true,
      headers: {'Content-Type': 'application/json'},
    }).catch(() => {
      // 静默失败 — 不要干扰用户体验
    })
  }
}

// PII 清洗
function scrubPII(url: string): string {
  try {
    const u = new URL(url, window.location.origin)
    // 移除常见 PII 参数
    const sensitive = ['token', 'access_token', 'auth', 'session', 'password',
                       'email', 'phone', 'ssn', 'api_key', 'key']
    for (const key of sensitive) {
      u.searchParams.delete(key)
    }
    // 截断过长的路径（防意外泄露在 URL query 中编码的 PII）
    const path = u.pathname.length > 200
      ? u.pathname.slice(0, 200) + '...'
      : u.pathname
    return path + u.search
  } catch {
    return url.length > 200 ? url.slice(0, 200) + '...' : url
  }
}

// 页面卸载时刷出剩余数据
// ⚠️ 不用 beforeunload/unload — 会破坏 bfcache
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    flush()
  }
})

// pagehide 作为兜底
window.addEventListener('pagehide', () => {
  flush()
})
```

### 3.4 SPA 软导航处理

```typescript
// spa-rum.ts — SPA 中的 CWV 重新测量
// web-vitals v4+ 支持 onFCP/onLCP/onCLS/onINP/onTTFB 的软导航重设

// React Router v6 示例
import {useEffect, useRef} from 'react'
import {useLocation} from 'react-router-dom'

function useRumNavigationTracking() {
  const location = useLocation()
  const prevPathname = useRef(location.pathname)

  useEffect(() => {
    if (prevPathname.current !== location.pathname) {
      // 1. 标记软导航
      markSoftNavigation(location.pathname)

      // 2. 清空前一个路由的 web-vitals 实例
      // (web-vitals v4+ 自动处理，低于 v4 需要手动 reset)

      // 3. 自定义 User Timing 测量
      performance.mark(`nav-to-${location.pathname}`)

      prevPathname.current = location.pathname
    }
  }, [location.pathname])
}

// Next.js App Router 示例
'use client'
import {usePathname, useSearchParams} from 'next/navigation'
import {useEffect, useRef} from 'react'

function useNextJSRumTracking() {
  const pathname = usePathname()
  const searchParams = useSearchParams()
  const prevKey = useRef('')

  useEffect(() => {
    const key = pathname + searchParams.toString()
    if (prevKey.current !== key) {
      markSoftNavigation(pathname)
      prevKey.current = key
    }
  }, [pathname, searchParams])
}
```

### 3.5 采样策略

```typescript
// sampling.ts — 按指标类型分级采样
const SAMPLING_RATES: Record<string, number> = {
  // LCP/INP/CLS: 100% — CWV 核心指标，全量采集
  LCP: 1.0,
  INP: 1.0,
  CLS: 1.0,
  // FCP/TTFB: 10% — 辅助指标，采样即可
  FCP: 0.1,
  TTFB: 0.1,
  // Custom: 1% — 自定义指标
  custom: 0.01,
  // Attribution: 10% — 归因数据量大
  attribution: 0.1,
}

function shouldSample(metricName: string): boolean {
  const rate = SAMPLING_RATES[metricName] ?? 0.1
  return Math.random() < rate
}

// 在 collector 中集成
function shouldReport(metric: Metric): boolean {
  if (!shouldSample(metric.name)) return false

  // 归因数据单独采样（数据量大的场景）
  if (metric.attribution && !shouldSample('attribution')) {
    // 仍然上报 metric 值，但移除 attribution
    const {attribution, ...rest} = metric
    return true // 继续上报
  }

  return true
}
```

### 3.6 聚合与告警

```sql
-- ClickHouse / BigQuery 查询: p75 LCP 按路由 + 部署版本
-- ClickHouse SQL
SELECT
    page,
    deploy_version,
    quantile(0.75)(value) AS p75_lcp,
    quantile(0.75)(value) AS p75_inp,
    quantile(0.75)(value) AS p75_cls,
    count() AS sample_count
FROM rum_events
WHERE
    name IN ('LCP', 'INP', 'CLS')
    AND timestamp >= now() - INTERVAL 1 HOUR
    AND page = '/checkout'
GROUP BY page, deploy_version
ORDER BY deploy_version DESC
```

```typescript
// 前端告警规则 (Datadog / Grafana 配置参考)
const alertRules = [
  {
    // LCP 回归告警
    metric: 'p75_lcp',
    condition: 'current_p75 > baseline_p75 * 1.2', // 20% 回归
    threshold: 2500, // 超过 Good 阈值
    window: '5m',
    action: 'rollback-webhook',
  },
  {
    // CLS 突破阈值
    metric: 'p75_cls',
    condition: 'p75_cls > 0.1',
    window: '5m',
    action: 'alert-slack',
  },
  {
    // 样本量过低（采集器故障检测）
    metric: 'sample_count',
    condition: 'sum(sample_count) < expected_count * 0.5',
    window: '15m',
    action: 'alert-pagerduty',
  },
]
```

---

## 四、性能预算与 CI 门禁

### 4.1 预算分级体系

```
预算三色分层:
┌────────────────────────────────────────────────────────────┐
│ 🟢 Good (通过)    → 合并允许                                │
│ 🟡 Warning        → 允许合并, 创建 JIRA ticket 追踪         │
│ 🔴 Critical       → 阻塞合并, 必须优化                       │
└────────────────────────────────────────────────────────────┘
```

### 4.2 Lighthouse CI 配置

```javascript
// lighthouserc.js — Lighthouse CI 断言式预算
module.exports = {
  ci: {
    collect: {
      staticDistDir: './dist',
      numberOfRuns: 3, // 多次运行取中位数，减少 CI 抖动
    },
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        // CWV 预算
        'largest-contentful-paint': ['error', {maxNumericValue: 2500}],
        'cumulative-layout-shift': ['error', {maxNumericValue: 0.1}],
        'total-blocking-time': ['error', {maxNumericValue: 200}],
        'interactive': ['warn', {maxNumericValue: 3500}],

        // Bundle 预算
        'dom-size': ['error', {maxNumericValue: 1500}],
        'total-byte-weight': ['error', {maxNumericValue: 2000000}], // 2MB

        // 最佳实践
        'unused-javascript': ['warn', {maxNumericValue: 200000}],
        'unused-css-rules': ['warn', {maxNumericValue: 50000}],
        'uses-responsive-images': 'warn',
        'uses-optimized-images': 'error',
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
}
```

### 4.3 Bundle Size Guard

```javascript
// bundlesize.config.js — Bundle Size 预算
// 使用 bundlesize 或 size-limit 包
module.exports = {
  files: [
    // 入口 JS
    {
      path: './dist/assets/index-*.js',
      maxSize: '150 KB',
      compression: 'brotli',
    },
    // Vendor chunk
    {
      path: './dist/assets/vendor-*.js',
      maxSize: '120 KB',
      compression: 'brotli',
    },
    // 所有 CSS
    {
      path: './dist/assets/*.css',
      maxSize: '50 KB',
      compression: 'brotli',
    },
    // 总 JS 体积
    {
      path: './dist/assets/*.js',
      maxSize: '350 KB',
      compression: 'brotli',
    },
  ],
}
```

```javascript
// webpack.config.js — webpack 内置性能提示
module.exports = {
  performance: {
    hints: 'error',
    maxAssetSize: 200 * 1024,      // 200KB 单个资源
    maxEntrypointSize: 350 * 1024,  // 350KB 入口点
    assetFilter(assetFilename) {
      // 只检查 JS 和 CSS
      return /\.(js|css)$/.test(assetFilename)
    },
  },
}
```

### 4.4 预算 JSON 格式（自定义格式，跨工具共享）

```json
{
  "$schema": "./performance-budget.schema.json",
  "version": "1.0",
  "budgets": [
    {
      "id": "lcp-p75",
      "type": "rum",
      "metric": "LCP",
      "aggregation": "p75",
      "thresholds": {
        "good": 2000,
        "warning": 3500,
        "critical": 4000
      },
      "scope": "per-route",
      "routes": ["/", "/product/*", "/checkout"]
    },
    {
      "id": "cls-p75",
      "type": "rum",
      "metric": "CLS",
      "aggregation": "p75",
      "thresholds": {
        "good": 0.05,
        "warning": 0.15,
        "critical": 0.25
      },
      "scope": "per-route"
    },
    {
      "id": "inp-p75",
      "type": "rum",
      "metric": "INP",
      "aggregation": "p75",
      "thresholds": {
        "good": 150,
        "warning": 350,
        "critical": 500
      },
      "scope": "per-route"
    },
    {
      "id": "total-js-brotli",
      "type": "bundle",
      "metric": "total-size",
      "unit": "KB",
      "compression": "brotli",
      "thresholds": {
        "good": 200,
        "warning": 400,
        "critical": 600
      },
      "scope": "global",
      "glob": "dist/assets/*.js"
    },
    {
      "id": "total-css-brotli",
      "type": "bundle",
      "metric": "total-size",
      "unit": "KB",
      "compression": "brotli",
      "thresholds": {
        "good": 40,
        "warning": 80,
        "critical": 120
      },
      "scope": "global",
      "glob": "dist/assets/*.css"
    }
  ]
}
```

### 4.5 RUM 回归自动回滚

```typescript
// rum-regression-guard.ts — 部署后 RUM 回归检测
// 在 CI/CD pipeline 中作为 post-deploy step 执行

interface RegressionCheckConfig {
  rumEndpoint: string
  apiKey: string
  lookbackWindow: '5m' | '15m' | '1h'
  regressionThreshold: number   // e.g., 0.20 = 20%
  metrics: string[]             // ['LCP', 'INP', 'CLS']
}

async function checkRegression(config: RegressionCheckConfig): Promise<{
  passed: boolean
  regressions: Array<{metric: string; pre: number; post: number; delta: number}>
}> {
  const results = []

  for (const metric of config.metrics) {
    // 查询部署前 baseline
    const baselineQuery = `
      SELECT quantile(0.75)(value) AS p75
      FROM rum_events
      WHERE name = '${metric}'
        AND timestamp < deploy_time
        AND timestamp >= deploy_time - INTERVAL 24 HOUR
    `
    const postDeployQuery = `
      SELECT quantile(0.75)(value) AS p75
      FROM rum_events
      WHERE name = '${metric}'
        AND timestamp >= deploy_time
        AND timestamp >= now() - INTERVAL ${config.lookbackWindow}
    `

    const [pre, post] = await Promise.all([
      queryRUM(baselineQuery, config),
      queryRUM(postDeployQuery, config),
    ])

    const delta = (post - pre) / pre
    if (delta > config.regressionThreshold && post > getThreshold(metric)) {
      results.push({metric, pre, post, delta})
    }
  }

  return {
    passed: results.length === 0,
    regressions: results,
  }
}

function getThreshold(metric: string): number {
  const thresholds: Record<string, number> = {
    LCP: 2500,
    INP: 200,
    CLS: 0.1,
  }
  return thresholds[metric] ?? Infinity
}

// CI Pipeline 集成 (GitHub Actions 示例)
// .github/workflows/deploy.yml
/*
  - name: Deploy
    run: npm run deploy

  - name: RUM Regression Check
    run: |
      npx ts-node scripts/rum-regression-guard.ts \
        --deploy-time "${{ github.event.head_commit.timestamp }}" \
        --lookback 15m \
        --threshold 0.20

  - name: Rollback on Regression
    if: failure()
    run: |
      echo "Performance regression detected — rolling back"
      npm run rollback
*/
```

---

## 五、内存工程

### 5.1 Heap Snapshot 工作流

```typescript
// 在 Chrome DevTools 中：
// 1. Memory 面板 → Heap Snapshot → Take snapshot (初始快照)
// 2. 执行可疑操作（路由切换、列表滚动、对话框打开/关闭）
// 3. Take snapshot (操作后快照)
// 4. 选择 "Comparison" 视图，按 "# Delta" 排序
// 5. 查找重复出现不合理的对象（闭包、事件监听、DOM 引用）

// 可编程式快照对比（Puppeteer）
// scripts/memory-leak-detect.ts
import puppeteer from 'puppeteer'

async function detectMemoryLeak(url: string, iterations = 10) {
  const browser = await puppeteer.launch()
  const page = await browser.newPage()

  await page.goto(url)
  const metrics: number[] = []

  for (let i = 0; i < iterations; i++) {
    // 执行会引发内存泄漏的操作序列
    await page.click('#open-modal')
    await page.waitForTimeout(500)
    await page.click('#close-modal')
    await page.waitForTimeout(500)

    // 手动触发 GC（需要 --js-flags="--expose-gc"）
    await page.evaluate(() => (window as any).gc?.())

    const jsHeap = await page.evaluate(
      () => (performance as any).memory?.usedJSHeapSize
    )
    metrics.push(jsHeap)
    console.log(`Iteration ${i + 1}: ${(jsHeap / 1024 / 1024).toFixed(2)} MB`)
  }

  // 检查内存趋势
  const trend = metrics[metrics.length - 1] - metrics[0]
  if (trend > 5 * 1024 * 1024) { // 增长 > 5MB
    console.error(`⚠️  Possible memory leak detected: +${(trend/1024/1024).toFixed(2)}MB`)
  }

  await browser.close()
}
```

### 5.2 常见内存泄漏检测模式

```typescript
// 泄漏模式 1: Detached DOM Nodes
// 症状: JS 持有已从 DOM 移除的节点引用
class Dropdown {
  private el: HTMLElement

  mount(parent: HTMLElement) {
    this.el = document.createElement('div')
    parent.appendChild(this.el)
  }

  // ❌ 卸载时未清除 el 引用
  unmount() {
    this.el.remove() // 从 DOM 移除，但 this.el 仍持有引用
    // 在 Heap Snapshot 中表现为 Detached HTMLElement
  }

  // ✅ 正确实现
  unmountFixed() {
    this.el.remove()
    this.el = null! // 或 undefined
  }
}

// 泄漏模式 2: 事件监听未清理
class Chart {
  private resizeHandler: () => void

  init() {
    this.resizeHandler = this.onResize.bind(this)
    // ❌ addEventListener 但从不 removeEventListener
    window.addEventListener('resize', this.resizeHandler)
  }

  // ✅ 提供 destroy
  destroy() {
    window.removeEventListener('resize', this.resizeHandler)
  }
}

// 泄漏模式 3: 定时器 / setInterval 未清除
class Carousel {
  private intervalId: number | null = null

  start() {
    // ❌ 没有检查是否已有 interval
    this.intervalId = window.setInterval(() => this.next(), 3000)
  }

  // ✅ 加防护
  startFixed() {
    if (this.intervalId !== null) return
    this.intervalId = window.setInterval(() => this.next(), 3000)
  }

  destroy() {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId)
      this.intervalId = null
    }
  }
}

// 泄漏模式 4: useEffect 中的异步操作未取消
// ❌ React 组件卸载后仍然 setState
useEffect(() => {
  let cancelled = false

  fetchData().then(data => {
    if (!cancelled) {
      setData(data)
    }
  })

  return () => {
    cancelled = true
  }
}, [])

// 泄漏模式 5: WebSocket / EventSource 未关闭
useEffect(() => {
  const ws = new WebSocket('wss://example.com')

  ws.onmessage = (event) => {
    setMessages(prev => [...prev, JSON.parse(event.data)])
  }

  return () => {
    ws.close() // 清理 WebSocket
  }
}, [])
```

### 5.3 SPA 生命周期内存管理

```typescript
// spa-memory-manager.ts — SPA 路由切换时的内存清理
class SPAMemoryManager {
  private routeComponents = new Map<string, {
    instance: any
    cleanup: Array<() => void>
  }>()

  // 注册路由组件
  register(routeId: string, instance: any, cleanupFns: Array<() => void>) {
    this.routeComponents.set(routeId, {instance, cleanup: cleanupFns})
  }

  // 路由切换时调用
  onRouteLeave(routeId: string) {
    const route = this.routeComponents.get(routeId)
    if (!route) return

    // 1. 执行所有清理函数
    for (const cleanup of route.cleanup) {
      try { cleanup() } catch {}
    }

    // 2. 断开所有引用
    route.instance = null
    route.cleanup = []

    // 3. 从 Map 移除
    this.routeComponents.delete(routeId)
  }

  // 在 visibilitychange 时建议 GC（低优先级，空闲执行）
  setupMemoryOptimization() {
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.scheduleCleanup()
      }
    })
  }

  private scheduleCleanup() {
    // scheduler.postTask 或 requestIdleCallback
    const scheduler = 'scheduler' in window
      ? (window as any).scheduler
      : null

    const runGC = () => {
      // 清除过期的 WeakRef 引用
      // 对于支持 gc 的环境，可尝试触发
      if (typeof (window as any).gc === 'function') {
        ;(window as any).gc()
      }
    }

    if (scheduler?.postTask) {
      scheduler.postTask(runGC, {priority: 'background'})
    } else {
      requestIdleCallback(runGC, {timeout: 5000})
    }
  }
}
```

### 5.4 WeakRef 泄漏检测

```typescript
// weakref-leak-detector.ts — 用 WeakRef 检测对象是否正确释放
class LeakDetector {
  private registry = new FinalizationRegistry<string>((label) => {
    console.log(`[LeakDetector] Object '${label}' was garbage collected ✓`)
  })

  private tracked = new Map<string, WeakRef<object>>()

  // 注册要跟踪的对象
  track<T extends object>(label: string, obj: T): T {
    this.registry.register(obj, label)
    this.tracked.set(label, new WeakRef(obj))
    return obj
  }

  // 手动检查对象是否存活（仅在特定场景使用，如测试）
  checkAlive(label: string): boolean {
    const ref = this.tracked.get(label)
    if (!ref) return false
    return ref.deref() !== undefined
  }

  // 检查所有跟踪对象的状态
  report(): string[] {
    const alive: string[] = []
    for (const [label, ref] of this.tracked) {
      if (ref.deref()) {
        alive.push(label)
      }
    }
    return alive
  }
}

// 使用示例
const detector = new LeakDetector()

// 在组件创建时
const heavyData = detector.track('search-results-cache', new Map())

// TTL 缓存 + 泄漏检测
class ManagedCache<K, V> {
  private cache = new Map<K, {value: V; expiry: number}>()
  private ref = detector.track('managed-cache', this.cache)

  set(key: K, value: V, ttlMs: number) {
    this.cache.set(key, {value, expiry: Date.now() + ttlMs})
  }

  cleanup() {
    const now = Date.now()
    for (const [key, entry] of this.cache) {
      if (entry.expiry < now) {
        this.cache.delete(key)
      }
    }
  }

  destroy() {
    this.cache.clear()
  }
}
```

### 5.5 useMemo/useCallback 成本收益

```typescript
// ⚠️ useMemo/useCallback 不是免费的：
// 1. 每次渲染都要执行依赖数组比较（浅比较 O(n)）
// 2. 额外的内存分配（memoized 值保存在 fiber 上）
// 3. 代码复杂度增加

// 决策树: 是否需要 useMemo？
// ├── 值被用作 useEffect 依赖？ → 需要（避免无限循环）
// ├── 传递给 React.memo 子组件？ → 需要（避免子组件无效渲染）
// ├── 计算开销 > 1ms？ → 需要
// └── 以上都不是 → 不需要（直接重新计算更便宜）

// ❌ 过度使用 useMemo
function BadComponent({items}: {items: number[]}) {
  // 简单算术不需要 memo — 数组创建 + 比较成本 > 计算成本
  const count = useMemo(() => items.length, [items])
  const first = useMemo(() => items[0], [items])

  return <div>{count} / {first}</div>
}

// ✅ 合理使用
function GoodComponent({items, threshold}: {items: number[]; threshold: number}) {
  // O(n) 过滤 + 排序 — 值得缓存
  const filtered = useMemo(
    () => items.filter(i => i > threshold).sort((a, b) => a - b),
    [items, threshold]
  )

  // 传递给 memo 子组件的回调 — 需要稳定引用
  const handleSelect = useCallback((id: number) => {
    doSomething(id)
  }, [])

  return <ItemList items={filtered} onSelect={handleSelect} />
}

// 更优: 如果数据不变，可以把计算提取到外部或使用 Web Worker
```

---

## 六、帧率与 Jank 诊断

### 6.1 rAF FPS 计数器

```typescript
// fps-monitor.ts — 基于 rAF 的 FPS 监控
class FPSMonitor {
  private frames = 0
  private lastTime = performance.now()
  private rafId: number | null = null
  private samples: number[] = []
  private onLowFPS: ((fps: number) => void) | null = null

  start(onLowFPS?: (fps: number) => void) {
    this.onLowFPS = onLowFPS ?? null
    this.lastTime = performance.now()
    this.frames = 0
    this.loop()
  }

  private loop = () => {
    this.frames++
    const now = performance.now()
    const elapsed = now - this.lastTime

    if (elapsed >= 1000) { // 每秒统计一次
      const fps = Math.round((this.frames * 1000) / elapsed)
      this.samples.push(fps)

      // 保留最近 60 个样本
      if (this.samples.length > 60) {
        this.samples.shift()
      }

      // 触发低帧率回调
      if (fps < 30 && this.onLowFPS) {
        this.onLowFPS(fps)
      }

      this.frames = 0
      this.lastTime = now
    }

    this.rafId = requestAnimationFrame(this.loop)
  }

  stop() {
    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId)
      this.rafId = null
    }
  }

  getStats() {
    if (this.samples.length === 0) return {avg: 0, min: 0, max: 0, p1: 0}
    const sorted = [...this.samples].sort((a, b) => a - b)
    const avg = sorted.reduce((s, v) => s + v, 0) / sorted.length
    const p1 = sorted[Math.floor(sorted.length * 0.01)] // 1 分位 — 最差帧率
    return {
      avg: Math.round(avg),
      min: sorted[0],
      max: sorted[sorted.length - 1],
      p1,
      current: sorted[sorted.length - 1],
    }
  }
}

// 使用
const monitor = new FPSMonitor()
monitor.start((fps) => {
  console.warn(`[FPS] Low frame rate detected: ${fps} FPS`)
})

// 在性能回归测试中集成
// 滚动页面并监控 FPS
async function scrollTest(page: any) {
  const fpsData = await page.evaluate(() => {
    const m = new (window as any).FPSMonitor()
    m.start()

    return new Promise((resolve) => {
      // 自动滚动
      let scrollY = 0
      const interval = setInterval(() => {
        scrollY += 100
        window.scrollTo(0, scrollY)
        if (scrollY >= document.body.scrollHeight) {
          clearInterval(interval)
          m.stop()
          resolve(m.getStats())
        }
      }, 100)
    })
  })

  console.log('Scroll FPS:', fpsData)
}
```

### 6.2 Long Tasks 归因

```typescript
// long-task-attribution.ts — 细粒度归因长任务
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const duration = entry.duration
    const attribution = (entry as any).attribution?.[0]

    if (!attribution) continue

    console.warn(`[Long Task] ${duration.toFixed(1)}ms`, {
      // 容器类型: 'iframe' | 'window' | 'same-origin-ancestor' 等
      containerType: attribution.containerType,
      containerName: attribution.containerName,
      containerId: attribution.containerId,

      // 脚本归因
      scriptURL: attribution.containerSrc,
      scriptName: attribution.containerName,

      // 具体的长任务来源
      // ("unknown" 表示无法归因到具体脚本)
    })
  }
})

longTaskObserver.observe({type: 'longtask', buffered: true})
```

### 6.3 Long Animation Frames（新一代诊断）

```typescript
// long-animation-frame-observer.ts — LoAF API (Chrome 123+)
// 比 Long Tasks 更精确：不仅检测长任务，还检测整个渲染帧
if ('PerformanceObserver' in window) {
  try {
    const loafObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const loaf = entry as any
        if (loaf.duration <= 50) continue // 只关注 > 50ms 的帧

        console.group(`[LoAF] ${loaf.duration.toFixed(1)}ms frame`)

        // 帧内所有脚本执行
        if (loaf.scripts?.length > 0) {
          console.table(loaf.scripts.map((s: any) => ({
            script: s.invoker || s.sourceURL || '(inline)',
            duration: `${s.duration.toFixed(1)}ms`,
            type: s.invokerType,
            // 以下字段指示脚本的调用来源:
            // invokerType: 'user-callback' | 'event-listener' | 'resolve-promise'
            //   | 'classic-script' | 'module-script' 等
          })))
        }

        // 长时间阻塞的渲染部分
        console.log('Blocking duration:', loaf.blockingDuration, 'ms')

        console.groupEnd()
      }
    })

    loafObserver.observe({type: 'long-animation-frame', buffered: true})
  } catch {
    // LoAF 不被当前浏览器支持时降级为 Long Tasks
    console.warn('Long Animation Frame API not available')
  }
}
```

### 6.4 Chrome Performance Panel 工作流

```
性能诊断决策流程:

1. Performance 面板录制 (3-5s)
   ├── 发现 Long Tasks → 用 Bottom-Up 视图 → 找 Self Time 最大的函数
   ├── 发现 Layout Shift → 切换到 Experience 轨道 → 点击红色 shift 标记
   ├── 发现渲染问题 → 切换 Frames 视图 → 找到卡顿帧 → 检查 GPU/Paint/Composite 耗时
   └── 发现大量 Recalculate Style → 找触发强制同步布局的代码

2. Bottom-Up (自底向上): Self Time 排序 → 定位瓶颈函数
   含义: Self Time = 函数自身执行时间，Total Time = 自身 + 所有子调用

3. Call Tree (调用树): 按调用路径自上而下 → 理解执行上下文

4. 火焰图: 视觉化堆栈，宽度 = 执行时间
   颜色: 黄色=脚本, 紫色=渲染, 绿色=绘制, 蓝色=加载, 灰色=空闲

5. Summary (饼图): 宏观耗时分布 — Loading / Scripting / Rendering / Painting
```

### 6.5 CSS 触发的性能开销

```typescript
// CSS Triggers 速查（完整的性能成本分级）
// 变更一个属性时，浏览器需重做的步骤:

const CSS_TRIGGER_COST: Record<string, {
  layout: boolean    // 是否需要重新布局 (reflow)
  paint: boolean     // 是否需要重新绘制 (repaint)
  composite: boolean // 是否需要重新合成
}> = {
  // === 最昂贵: Layout → Paint → Composite ===
  width:            {layout: true, paint: true, composite: true},
  height:           {layout: true, paint: true, composite: true},
  top:              {layout: true, paint: true, composite: true},
  left:             {layout: true, paint: true, composite: true},
  margin:           {layout: true, paint: true, composite: true},
  padding:          {layout: true, paint: true, composite: true},
  border:           {layout: true, paint: true, composite: true},
  fontSize:         {layout: true, paint: true, composite: true},
  fontFamily:       {layout: true, paint: true, composite: true},
  display:          {layout: true, paint: true, composite: true},
  position:         {layout: true, paint: true, composite: true},

  // === 中等: Paint → Composite（跳过 Layout）===
  color:            {layout: false, paint: true, composite: true},
  backgroundColor:  {layout: false, paint: true, composite: true},
  boxShadow:         {layout: false, paint: true, composite: true},
  outline:          {layout: false, paint: true, composite: true},

  // === 最便宜: 仅 Composite（GPU 加速）===
  transform:        {layout: false, paint: false, composite: true},
  opacity:          {layout: false, paint: false, composite: true},

  // === 零成本: 不影响渲染 ===
  cursor:           {layout: false, paint: false, composite: false},
}

// 强制同步布局 (Forced Synchronous Layout) 检测
// ⚠️ 在 DevTools Performance 面板中标记为紫色线条（Layout 轨道）
function detectFSL() {
  // 典型的 FSL 模式: 写→读→写在同一个 frame
  // 写
  element.style.width = '100px'
  // 读 — 触发布局重算
  const height = element.offsetHeight
  // 再写
  element.style.height = height + 10 + 'px'

  // ✅ 修复: 先批量读，再批量写
  // 读
  const heights = elements.map(el => el.offsetHeight)
  // 写
  elements.forEach((el, i) => {
    el.style.height = heights[i] + 10 + 'px'
  })
}

// FastDOM 风格的批处理
class LayoutBatcher {
  private reads: Array<() => void> = []
  private writes: Array<() => void> = []

  measure(fn: () => void) {
    this.reads.push(fn)
    this.scheduleFlush()
  }

  mutate(fn: () => void) {
    this.writes.push(fn)
    this.scheduleFlush()
  }

  private scheduled = false
  private scheduleFlush() {
    if (this.scheduled) return
    this.scheduled = true
    requestAnimationFrame(() => {
      // 先读
      for (const read of this.reads) read()
      this.reads = []
      // 后写
      for (const write of this.writes) write()
      this.writes = []
      this.scheduled = false
    })
  }
}
```

---

## 七、Bundle 与加载性能

### 7.1 Bundle 分析工具链

```bash
# === webpack-bundle-analyzer ===
# 生成可视化 report
npm install --save-dev webpack-bundle-analyzer

# webpack.config.js
const BundleAnalyzerPlugin =
  require('webpack-bundle-analyzer').BundleAnalyzerPlugin

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',      # 生成 HTML report
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json',
    }),
  ],
}
```

```bash
# === source-map-explorer ===
# 适合查看具体的行级来源
npm install --save-dev source-map-explorer

# package.json scripts
{
  "analyze": "source-map-explorer dist/assets/*.js --html dist/sme-report.html",
  "analyze:gzip": "source-map-explorer dist/assets/*.js --gzip"
}
```

### 7.2 代码分割策略

```typescript
// === 路由级分割 (React.lazy) ===
import {lazy, Suspense} from 'react'
import {createBrowserRouter} from 'react-router-dom'

// 每个路由独立 chunk
const HomePage = lazy(() => import(/* webpackChunkName: "home" */ './pages/Home'))
const ProductPage = lazy(() => import(/* webpackChunkName: "product" */ './pages/Product'))
const CheckoutPage = lazy(() => import(/* webpackChunkName: "checkout" */ './pages/Checkout'))

// 路由预加载 — 在链接 hover 时预加载
function LinkWithPrefetch({to, children}: {to: string; children: React.ReactNode}) {
  const preload = () => {
    const chunkMap: Record<string, () => Promise<any>> = {
      '/': () => import('./pages/Home'),
      '/product': () => import('./pages/Product'),
      '/checkout': () => import('./pages/Checkout'),
    }
    chunkMap[to]?.()
  }

  return (
    <a href={to} onMouseEnter={preload} onFocus={preload}>
      {children}
    </a>
  )
}

// === 组件级分割 ===
// 对于大型组件（富文本编辑器、图表库、代码编辑器等）
const CodeEditor = lazy(() => import('./components/CodeEditor'))

// or 使用 @loadable/component 支持 SSR
// import loadable from '@loadable/component'
// const CodeEditor = loadable(() => import('./components/CodeEditor'))
```

```javascript
// webpack.config.js — Vendor 分割
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // React & ReactDOM 独立 chunk（长期缓存）
        framework: {
          test: /[\\/]node_modules[\\/](react|react-dom|scheduler)[\\/]/,
          name: 'framework',
          priority: 40,
          reuseExistingChunk: true,
        },
        // 其他 node_modules 库
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
          minChunks: 2, // 至少被 2 个 chunk 引用
          reuseExistingChunk: true,
        },
        // 公共组件
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
          name: 'common',
        },
      },
    },
    // Runtime chunk 独立（频繁变化的 webpack runtime）
    runtimeChunk: 'single',
  },
}
```

### 7.3 Tree Shaking 验证

```javascript
// === 检查未使用的导出 ===
// webpack.config.js — 输出未使用导出报告
const {WebpackBundleSizeAnalyzerPlugin} = require('webpack-bundle-size-analyzer')

module.exports = {
  // webpack 5 默认支持 tree shaking（需要 ES Module）
  mode: 'production', // 必须 production mode

  // 验证: 构建后检查 unused exports
  plugins: [
    {
      apply(compiler) {
        compiler.hooks.done.tap('UnusedExportsCheck', (stats) => {
          const json = stats.toJson({
            usedExports: true,
            providedExports: true,
          })

          for (const mod of json.modules || []) {
            if (!mod.usedExports || !mod.providedExports) continue
            const unused = mod.providedExports.filter(
              (exp) => !mod.usedExports!.includes(exp)
            )
            if (unused.length > 0) {
              console.log(`⚠️  Unused exports in ${mod.name}:`, unused)
            }
          }
        })
      },
    },
  ],
}

// === 确保 package.json sideEffects 正确 ===
// package.json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
  // 如果所有文件都没有副作用，可以设为 false
  // "sideEffects": false
}

// === 常见导致 Tree Shake 失败的原因 ===
// 1. 使用 CommonJS (require/module.exports)
// 2. 副作用导入: import './styles.css' 未在 sideEffects 声明
// 3. 动态属性访问: obj[dynamicKey]
// 4. 类的静态属性引用
```

### 7.4 Import Cost 优化

```typescript
// === 使用 VS Code Import Cost 插件实时查看包大小 ===

// === 替代大型库 ===
// ❌ moment.js ~ 72KB gzipped (不可 tree-shake)
// ✅ date-fns ~ 函数级 tree-shaking
import {format, addDays} from 'date-fns'
//   ↑ 只打包用到的函数

// ❌ lodash ~ 72KB gzipped (全量)
// ✅ lodash-es ~ 函数级 tree-shaking + 更小的替代方案
import debounce from 'lodash-es/debounce'  // 部分 tree-shakable
// ✅ 或直接使用原生（现代浏览器支持）
// Array.prototype.flatMap, Object.fromEntries 等已原生

// ❌ axios ~ 13KB gzipped
// ✅ ky ~ 3KB gzipped (基于 fetch)
// ✅ 或直接使用 fetch (现代浏览器已原生)
```

```typescript
// === 按需加载大型图表库 ===
// ❌ import Chart from 'chart.js' — 全量 200KB+
// ✅ 按需注册
import {
  Chart,
  BarController,
  BarElement,
  CategoryScale,
  LinearScale,
} from 'chart.js'

Chart.register(BarController, BarElement, CategoryScale, LinearScale)

// 或者用 dynamic import 延迟加载
const ChartComponent = lazy(async () => {
  const {Chart, registerables} = await import('chart.js')
  Chart.register(...registerables)
  return {default: ActualChartComponent}
})
```

### 7.5 Modern JS 差异化分发

```html
<!-- 现代浏览器用 <script type="module"> -->
<!-- 旧浏览器用 <script nomodule> -->

<!-- 方案一: webpack 双构建 -->
<!-- modern build: ES2017+ 语法，更小更快 -->
<script type="module" src="/assets/app.modern.js"></script>
<!-- legacy build: ES5 语法，带 polyfill -->
<script nomodule src="/assets/app.legacy.js"></script>

<!-- 方案二: Vite 自动处理 -->
<!-- Vite 的 @vitejs/plugin-legacy 自动生成双版本 -->
```

```javascript
// vite.config.js — 差异化分发
import legacy from '@vitejs/plugin-legacy'

export default {
  plugins: [
    legacy({
      targets: ['defaults', 'not IE 11'], // 现代目标
      // 自动生成 legacy chunk
      modernPolyfills: true,
    }),
  ],
  build: {
    // 现代构建使用 ES module
    target: 'es2015',
    // 但也需要产出 legacy
  },
}
```

```javascript
// webpack 差异化构建配置
// webpack.modern.js
module.exports = {
  output: {
    filename: '[name].modern.[contenthash].js',
  },
  target: ['web', 'es2017'], // webpack 5 支持 esXXXX target
  module: {
    rules: [{
      test: /\.js$/,
      exclude: /node_modules/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: [['@babel/preset-env', {
            targets: {esmodules: true}, // 仅支持 ES Module 的浏览器
            modules: false,              // 保留 ES Module 语法
          }]],
        },
      },
    }],
  },
}

// 检测是否需要 polyfill
const needsPolyfill = !('Promise' in window && 'fetch' in window)
if (needsPolyfill) {
  import('./polyfills.js')
}
```

---

## 八、性能反模式清单

> 以下每条均为生产环境验证的 NEVER 规则，违反后将面临可量化的问题。

### 8.1 测量反模式

| # | 规则 | 后果 | 正确做法 |
|---|------|------|---------|
| 1 | NEVER 仅在 Lab (Lighthouse) 中测量性能 | Lab 数据不代表真实用户设备/网络/地理位置 | Lab + RUM 双轨测量，Lab 做趋势检测，RUM 做 p75 决策 |
| 2 | NEVER 用 median/avg 替代 p75 | CWV 考核 p75，平均值掩饰长尾问题 | 所有聚合用 percentiles(0.75) |
| 3 | NEVER 用 `load` 事件作为测量终点 | LCP 可能远早于 load；CLS 在 load 后仍会累积 | 用 PerformanceObserver 及 web-vitals 库 |
| 4 | NEVER 忽略 bfcache | bfcache 恢复时 observer 不重新触发，丢失数据 | 在 pageshow 事件检查 `event.persisted`，重新注册 observer |
| 5 | NEVER 多次调用 web-vitals 函数 | 每次调用创建新的 PerformanceObserver 实例 → 内存泄漏 | 全局单例调用一次 |

### 8.2 数据发送反模式

| # | 规则 | 后果 | 正确做法 |
|---|------|------|---------|
| 6 | NEVER 发送 CLS value 而非 delta | 多次上报导致聚合值膨胀 | 始终发送 `delta` |
| 7 | NEVER 按事件采样而非按 session 采样 | 丢失 session 级关联，无法计算 p75 session 维度的 INP | 按 session/pageview 维度采样 |
| 8 | NEVER 使用 `unload`/`beforeunload` 发送 beacon | 破坏 bfcache，且浏览器可能不发送请求 | 用 `visibilitychange` + `pagehide` + `sendBeacon` |
| 9 | NEVER 不清洗 PII 就发送 performance 数据 | URL 可能含 token/email/phone 等 | 发送前 scrub URL 参数和过长路径 |

### 8.3 架构反模式

| # | 规则 | 后果 | 正确做法 |
|---|------|------|---------|
| 10 | NEVER 忽略图片的 width/height | CLS 的直接来源 | 始终设置 width/height 或 aspect-ratio |
| 11 | NEVER 用 `100vh` 做全屏高度 | iOS Safari 工具栏导致溢出 | 用 `100dvh` 或 `-webkit-fill-available` |
| 12 | NEVER 在 useEffect 依赖中放引用类型字面量 | 每次渲染新引用 → 死循环 | useMemo/useCallback 或提取到组件外 |
| 13 | NEVER 无限制使用 backdrop-filter | 每个创建独立 GPU 合成层，移动端 3 个以上帧率崩塌 | 限制使用次数，或在移动端用纯色替代 |

### 8.4 渲染反模式

| # | 规则 | 后果 | 正确做法 |
|---|------|------|---------|
| 14 | NEVER 在 requestAnimationFrame 中做同步布局读取 | 计数器直觉：rAF 在布局前执行，读取触发强制重算 | 读操作放在 rAF 之前或使用 LayoutBatcher |
| 15 | NEVER 在 scroll 事件处理器中读取 offsetHeight | 高频事件 × 强制布局 = 严重 jank | throttle + 分离读写阶段 |
| 16 | NEVER 使用 CSS `@import` | 串行阻塞加载，每个 @import 增加 RTT | 统一用 `<link>` 或打包到主 CSS |

### 8.5 防呆检查清单

```typescript
// deployment-checklist.ts — 每次部署前自检
const PRE_DEPLOY_CHECKLIST = [
  // 1. LCP 防御
  'LCP 元素（hero image / heading）是否在 HTML 中静态存在？',
  'LCP 图片是否有 fetchpriority="high"？',
  'LCP 图片是否有明确 width/height？',

  // 2. CLS 防御
  '所有 <img> 是否有 width/height？',
  '动态注入内容（广告、banner）是否有预留空间？',
  'Web Font 是否使用 font-display: swap/optional？',

  // 3. INP 防御
  '是否有 >50ms 的同步操作？考虑拆分或移到 worker',
  '高频事件（scroll, input, mousemove）是否有 throttle/debounce？',
  '交互处理器中是否有强制同步布局？',

  // 4. Bundle 防御
  'Bundle size 是否 < 预算？',
  '是否有未使用的大型依赖可以删除？',
  '是否启用了 compression (gzip/brotli)？',

  // 5. Memory 防御
  '组件是否有 proper cleanup（destroy/unmount）?',
  'WebSocket/EventSource/定时器是否有 teardown？',
  '是否有未解除的 addEventListener？',

  // 6. Observer 防御
  'web-vitals 是否全局单例调用？',
  '是否处理了 bfcache 恢复？',
  '是否在 visibilitychange/pagehide 发送了剩余数据？',
]

function runChecklist() {
  const results = PRE_DEPLOY_CHECKLIST.map((item, i) => ({
    id: i + 1,
    item,
    status: '⚠️  MANUAL CHECK REQUIRED',
  }))

  console.table(results)
  return results
}
```

### 8.6 上线后监控反模式

```typescript
// ❌ 错误监控：只看 p50（中位数）
// ✅ 正确监控：p50(趋势) + p75(CWV阈值) + p95(长尾)

const DASHBOARD_METRICS = {
  realtime: ['p75 LCP', 'p75 INP', 'p75 CLS'],
  trend: ['p50 LCP 7d moving avg', 'p95 LCP', 'sample coverage %'],
  regression: ['deploy-baseline delta %', 'regression count / deploy'],
}

// ❌ "性能没问题" — 仅看了本地开发环境
// ✅ "性能 OK" = {
//   Lab: Lighthouse CI 通过，
//   RUM: p75 CWV 全部 Good，
//   Bundle: 预算内，
//   Memory: 无增长趋势，
//   FPS: 30+ FPS (移动端) / 55+ FPS (桌面端)
// }
```

---

## 参考资源

- [web.dev/vitals](https://web.dev/vitals/) — CWV 官方文档
- [web-vitals NPM](https://www.npmjs.com/package/web-vitals) — Google 官方测量库
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) — 性能预算 CI 工具
- [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance) — 性能面板文档
- [webpack Bundle Analysis](https://webpack.js.org/guides/code-splitting/) — 代码分割指南
- [W3C Performance Timeline](https://www.w3.org/TR/performance-timeline/) — Performance APIs 规范
- [Long Animation Frames API](https://developer.chrome.com/docs/web-platform/long-animation-frames) — LoAF API 详解
- [CSS Triggers](https://csstriggers.com/) — CSS 属性性能成本速查
