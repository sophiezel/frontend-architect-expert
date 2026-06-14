# 前端可观测性全链路 (Frontend Observability)

> 蒸馏来源: Google Web Vitals 团队 (Addy Osmani / Philip Walton),
> Sujeet Jaiswal (Principal Engineer, RUM 实战),
> 字节跳动埋点平台工程实践, Sentry 错误监控, rrweb 会话回放,
> OpenTelemetry JS SDK, Vercel / Cloudflare Analytics 生态

---

## 一、可观测性三层模型

后端有三支柱: **Logs + Metrics + Traces**。前端映射有本质差异 ——
单点在客户端、网络不可控、设备碎片化。因此前端可观测性重新定义为五维度:

```
┌─────────────────────────────────────────────────────────────────┐
│                    前端可观测性五维度                              │
│                                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │   性能    │  │   错误   │  │   行为   │  │   日志   │       │
│   │ Metrics  │  │  Errors  │  │  Events  │  │   Logs   │       │
│   │ (CWV)    │  │ (Sentry) │  │ (埋点)    │  │          │       │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│        │              │              │              │           │
│        └──────────────┴──────────────┴──────────────┘           │
│                            │                                     │
│                     ┌──────┴──────┐                              │
│                     │  会话回放    │                              │
│                     │  (rrweb)    │                              │
│                     └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

| 维度 | 后端等价物 | 前端特殊挑战 | 核心工具 |
|------|-----------|-------------|---------|
| **性能指标** | Metrics / RED | 设备碎片化、网络抖动、bfcache | web-vitals + PerformanceObserver |
| **错误追踪** | Traces + Sentry | Source Map 管理、跨域脚本错误 | Sentry + Debug IDs |
| **用户行为** | 业务 Metrics | 隐私合规、离线缓冲、批量聚合 | 自定义埋点 SDK |
| **客户端日志** | Logs (stdout) | 存储限制、PII 脱敏、离线重放 | IndexedDB + sendBeacon |
| **会话回放** | — (特有) | DOM 尺寸、隐私遮蔽、存储爆炸 | rrweb / LogRocket |

### 关键认知: Lab Data != Field Data

```
Lab (Lighthouse/WPT):
  - 受控环境: 固定网络/devices/viewport
  - 可复现: 同版本同分数
  - 用途: 开发阶段瓶颈检测、回归卡点

Field (RUM/CrUX):
  - 真实设备/网络/地理位置分布
  - 不可复现: 用户环境千差万别
  - 用途: 用户真实体验度量、CWV 评分、A/B 显著性验证

铁律: 按 CrUX/CWV 评分必须用 Field 数据。Lab 只能辅助调试。
Lab 满分 ≠ 现场满分。永远以 RUM 为准。
```

---

## 二、核心 Web Vitals 采集架构

### 2.1 三指标定义速查

| 指标 | 全称 | 测量什么 | 阈值 Good | 阈值 Poor | 注记 |
|------|------|---------|----------|----------|------|
| **LCP** | Largest Contentful Paint | 最大可见内容渲染完成 | ≤2500ms | >4000ms | 2024年起IMG+文本+背景图+视频海报均计入 |
| **INP** | Interaction to Next Paint | 交互延迟(替代FID) | ≤200ms | >500ms | 2024年3月正式替代FID; 测量整页生命周期 |
| **CLS** | Cumulative Layout Shift | 累计布局偏移 | ≤0.1 | >0.25 | 计算公式: impact × distance; 窗口最大session |

### 2.2 web-vitals 库采集管线

```
┌─────────────────────────────────────────────────────────────────┐
│                    RUM 采集管线 (完整架构)                         │
│                                                                 │
│  Browser                                                         │
│  │                                                               │
│  ├─ PerformanceObserver('largest-contentful-paint')              │
│  ├─ PerformanceObserver('layout-shift')                          │
│  ├─ PerformanceObserver('event') {"durationThreshold": 16}       │
│  │     (INP 需要监听所有点击/按键/触摸 找到最大交互延迟)            │
│  └─ PerformanceObserver('long-animation-frame')  [LoAF 辅助诊断]  │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────┐                                            │
│  │   metric delta  │  ◀── 非 value! CLS 累加值用 delta          │
│  │   + attribution │  ◀── 1.5KB attribution build               │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Session Sampler │  ◀── hash( sessionId ) % 100 < sampleRate │
│  │  + PII Scrubber  │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Buffer (≤40KB) │  ◀── 一批 10-20 条指标                     │
│  │  5s flush timer  │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  visibilitychange / pagehide                                     │
│  └─ sendBeacon( '/v1/rum', protobuf(events) )                   │
│                                                                 │
│  Edge (Worker/Function)                                          │
│  │  ├─ 解密/鉴权                                                 │
│  │  ├─ PII 二次脱敏                                              │
│  │  └─ 转发 → 消息队列 / ClickHouse / TimescaleDB               │
│                                                                 │
│  Backend                                                         │
│  │  ├─ 流式写入 ClickHouse (高吞吐时序)                          │
│  │  ├─ 物化视图: p75 LCP/INP/CLS by route×device×geo×day        │
│  │  └─ 告警: p75 LCP > 4000ms → PagerDuty                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 CLS 的核心陷阱: delta vs value

```typescript
// ❌ 错误 —— 直接用 metric.value 会导致 CLS 虚高
// CLS 在整个页面生命周期内持续累加，value 是累计值
webVitals.onCLS((metric) => {
  report({ cls: metric.value });  // ❌ 第3次回调时value是之前所有layout shift之和
});

// ✅ 正确 —— 使用 metric.delta 获取每次新增的偏移量
// web-vitals 内部已做差值处理，delta 是当前回调相对于上次的增量
webVitals.onCLS((metric) => {
  // metric.value  = 累计 CLS (用于本地调试显示)
  // metric.delta  = 本次增量 (用于上报) —— 这才是你要的
  report({ cls: metric.delta });  // ✅
});

// 原理说明:
// CLS 计算窗口: 5 秒内一组 layout shift 中取最大值作为 session 分数
// 页面生命周期可能有多个 session 窗口
// 最终 CLS score = max( session1, session2, ... )
// web-vitals 内部跟踪: 每次新 session > 旧 session 时触发回调，delta = 新值 - 旧值
```

### 2.4 bfcache (Back/Forward Cache) 处理

```typescript
// bfcache: 浏览器保存整个页面快照，前进/后退时恢复
// 问题: 恢复后所有 observer 丢失，页面看起来"活着"但不产生新的 CWV 指标
// 解决: 监听 pageshow 事件，重新初始化所有 observer

import { onCLS, onINP, onLCP } from 'web-vitals';

function initWebVitals() {
  onCLS(sendToAnalytics, { reportAllChanges: true });
  onLCP(sendToAnalytics);
  onINP(sendToAnalytics);
}

// 首次加载
initWebVitals();

// bfcache 恢复时必须重新初始化
// pageshow.persisted === true → 从 bfcache 恢复
// pageshow.persisted === false → 正常首次加载
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // 1. 重新初始化 PerformanceObserver
    initWebVitals();
    // 2. 可选: 上报 bfcache 恢复事件
    sendToAnalytics({ type: 'bfcache-restore', time: performance.now() });
  }
});

// ⚠️ 在 visibilitychange 中上报时注意:
// bfcache 进入缓存时不触发 unload，但可能触发 visibilitychange(hidden)
// 需要配合 pagehide.persisted 判断是否真的离开还是进 bfcache
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    // 不要在此 flush 所有数据 —— 可能是进 bfcache
    // 放到 pagehide 中判断
  }
});

window.addEventListener('pagehide', (event) => {
  if (!event.persisted) {
    // 真正的页面卸载（不是进 bfcache），flush 所有缓冲数据
    flushBeacon();
  }
});
```

### 2.5 sendBeacon + visibilitychange 上报模式

```typescript
// 传输策略选择矩阵:
// ┌─────────────┬──────────────┬────────────────────┐
// │ API          │ 可靠性       │ 适用场景            │
// ├─────────────┼──────────────┼────────────────────┤
// │ fetch keepalive │ 高 (post-unload) │ 最新浏览器,可设header │
// │ sendBeacon   │ 高 (post-unload) │ 通用,不可设header    │
// │ xhr (sync)   │ 中 (会阻塞)   │ 遗留兼容(不推荐)      │
// │ unload事件   │ 低 (移动端不触发) │ ❌ 禁止使用          │
// │ beforeunload │ 低 (break bfcache) │ ❌ 禁止使用         │
// │ fetchLater() │ 最高 (浏览器管理) │ Chrome 121+ 实验性   │
// └─────────────┴──────────────┴────────────────────┘

class RUMTransporter {
  private buffer: RumEvent[] = [];
  private readonly MAX_SIZE = 20;
  private readonly FLUSH_INTERVAL = 5000;
  private timer: ReturnType<typeof setInterval> | null = null;

  constructor(private endpoint: string, private sampleRate: number = 1) {
    this.startFlushTimer();
    this.registerUnloadListener();
  }

  // 稳定采样: 同 session 始终在或不在样本中
  // 比随机采样好 —— 同一用户的完整行为链可追溯
  private shouldSample(): boolean {
    const sessionId = this.getSessionId();
    const hash = this.djb2(sessionId);
    return (hash % 100) < this.sampleRate;
  }

  private getSessionId(): string {
    let sid = sessionStorage.getItem('__sid');
    if (!sid) {
      sid = `${Date.now()}-${Math.random().toString(36).slice(2)}`;
      sessionStorage.setItem('__sid', sid);
    }
    return sid;
  }

  addEvent(event: Omit<RumEvent, 'timestamp'>) {
    if (!this.shouldSample()) return;

    this.buffer.push({
      ...event,
      timestamp: Date.now(),
      sessionId: this.getSessionId(),
    });

    if (this.buffer.length >= this.MAX_SIZE) {
      this.flush();
    }
  }

  private startFlushTimer() {
    this.timer = setInterval(() => this.flush(), this.FLUSH_INTERVAL);
  }

  private registerUnloadListener() {
    // NEVER use unload or beforeunload —— 会破坏 bfcache
    // visibilitychange + pagehide combo 是最佳实践
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        // 预热: 将数据序列化到 Blob（但不阻塞页面）
        this.flush();
      }
    });

    window.addEventListener('pagehide', (event) => {
      if (event.persisted) {
        // 进入 bfcache，停止定时器，数据留在内存
        this.stopTimer();
      } else {
        // 真正卸载，最后 flush
        this.flush(true); // synchronous
      }
    });

    // bfcache 恢复时重启定时器
    window.addEventListener('pageshow', (event) => {
      if (event.persisted) {
        this.startFlushTimer();
      }
    });
  }

  private flush(sync: boolean = false) {
    if (this.buffer.length === 0) return;

    const payload = this.encode(this.buffer);
    this.buffer = [];

    if (navigator.sendBeacon) {
      // sendBeacon 自动处理 post-unload，浏览器保证投递
      // 限制: data 必须为 Blob/FormData/URLSearchParams/string
      //       不能设置自定义 header
      const blob = new Blob([payload], { type: 'text/plain' });
      const ok = navigator.sendBeacon(this.endpoint, blob);
      if (!ok) {
        // sendBeacon 队列满，降级到 IndexedDB 离线缓冲
        this.saveToOfflineBuffer(payload);
      }
    } else {
      // 降级: fetch keepalive
      fetch(this.endpoint, {
        method: 'POST',
        body: payload,
        keepalive: true,  // 允许 post-unload 继续发送
        headers: { 'Content-Type': 'text/plain' },
      }).catch(() => this.saveToOfflineBuffer(payload));
    }
  }

  // Google fetchLater() 实验性 API (Chrome 121+)
  // 优势: 浏览器完全管理发送时机，不受进程生命周期限制
  private flushWithFetchLater() {
    if ('fetchLater' in window) {
      (window as any).fetchLater(this.endpoint, {
        method: 'POST',
        body: this.encode(this.buffer),
        activateAfter: 0, // 即使页面关闭也发送
      });
    } else {
      this.flush(true);
    }
  }

  private saveToOfflineBuffer(data: string) {
    // IndexedDB 离线缓冲: 下次页面加载时重试
    const req = indexedDB.open('rum-offline', 1);
    req.onsuccess = () => {
      const tx = req.result.transaction('events', 'readwrite');
      tx.objectStore('events').add({ data, timestamp: Date.now(), retries: 0 });
    };
  }

  private encode(events: RumEvent[]): string {
    // 生产环境推荐 Protobuf，比 JSON 体积小 40-60%
    // 字节团队实测: JSON 200KB → Protobuf 80KB
    return JSON.stringify(events);
  }

  private stopTimer() {
    if (this.timer) clearInterval(this.timer);
  }

  private djb2(str: string): number {
    let hash = 5381;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) + hash) + str.charCodeAt(i);
      hash = hash >>> 0; // force unsigned
    }
    return hash;
  }
}

type RumEvent = {
  type: 'lcp' | 'cls' | 'inp' | 'fcp' | 'ttfb';
  value: number;
  attribution?: Record<string, unknown>;
  timestamp: number;
  sessionId: string;
};
```

### 2.6 Attribution Build —— 1.5KB 的回报率

```typescript
// standard build: 2KB brotli, 仅指标值
// attribution build: +1.5KB, 附带根因数据
// Sujeet Jaiswal 的经验: "1.5KB pays for itself on first debug"

import { onLCP, onCLS, onINP } from 'web-vitals/attribution';

onLCP((metric) => {
  // attribution 提供 LCP 元素详情
  const { element, url, resourceLoadDelay, elementRenderDelay } = metric.attribution;
  // resourceLoadDelay: TTFB → 资源开始加载
  // elementRenderDelay: 资源加载完 → 元素渲染
  // → 精确定位: 是CDN慢还是渲染阻塞
  report({
    type: 'lcp',
    value: metric.value,
    rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
    attribution: {
      element: element?.tagName,
      url,
      resourceLoadDelay,
      elementRenderDelay,
    },
  });
});

onCLS((metric) => {
  // attribution 提供最大偏移源
  const { largestShiftTarget, largestShiftTime, largestShiftValue, sources } = metric.attribution;
  report({
    type: 'cls',
    value: metric.delta,  // ← delta!
    attribution: {
      target: largestShiftTarget?.tagName,
      time: largestShiftTime,
      value: largestShiftValue,
    },
  });
});

onINP((metric) => {
  // attribution 提供交互详情
  const { eventTarget, eventType, loadState, inputDelay, processingDuration, presentationDelay } = metric.attribution;
  // inputDelay: 事件触发 → 回调开始执行 (主线程繁忙)
  // processingDuration: 回调执行时间 (JS 计算)
  // presentationDelay: 回调结束 → 浏览器绘制 (渲染排队)
  report({
    type: 'inp',
    value: metric.value,
    attribution: {
      target: eventTarget?.tagName,
      event: eventType,
      loadState,
      phases: { inputDelay, processingDuration, presentationDelay },
    },
  });
});
```

### 2.7 后端聚合: Percentile ≠ Average

```sql
-- 性能指标不服从正态分布，平均值被长尾拖垮
-- 正确做法: 按百分位聚合，CWV 标准使用 p75

-- ClickHouse 物化视图: 按路由+设备+地理按天聚合
CREATE MATERIALIZED VIEW cwv_daily_by_route
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(dt)
ORDER BY (dt, route, device_type, country)
AS SELECT
    toDate(timestamp) AS dt,
    route,
    device_type,
    country,
    -- LCP / INP 使用 p75 分位
    quantile(0.75)(lcp_value) AS p75_lcp,
    quantile(0.75)(inp_value) AS p75_inp,
    -- CLS 同样 p75
    quantile(0.75)(cls_value) AS p75_cls,
    -- p75 合格率 (Good%)
    countIf(lcp_value <= 2500) / count() AS lcp_good_ratio,
    countIf(inp_value <= 200) / count() AS inp_good_ratio,
    countIf(cls_value <= 0.1) AS cls_good_ratio,
    -- 其他分位用于调试
    quantile(0.50)(lcp_value) AS p50_lcp,
    quantile(0.95)(lcp_value) AS p95_lcp,
    count() AS total_samples
FROM rum_events
WHERE lcp_value > 0
GROUP BY dt, route, device_type, country;
```

### 2.8 CrUX 数据特性

```
CrUX (Chrome User Experience Report):
  - 数据源: 已登录且已同步的 Chrome 用户
  - 窗口: 28 天滚动 (非日粒度)
  - 分位: p75 (与 CWV 评分标准一致)
  - 覆盖: 百万级起源，按国家/设备/连接类型细分
  - 限制: 仅 Chrome、仅公开可索引页面、有最低流量阈值

Google PSI / Search Console / BigQuery 均可查询 CrUX 数据。
CrUX = 全局视角, RUM = 细粒度视角。两者互补。
```

---

## 三、错误监控体系

### 3.1 Sentry 初始化: 生产级配置

```typescript
// sentry.config.ts — 生产级模板
import * as Sentry from '@sentry/browser';

Sentry.init({
  dsn: '__DSN__',

  // ── 环境与版本 ──
  environment: process.env.DEPLOY_ENV,     // 'production' | 'staging'
  release: process.env.SENTRY_RELEASE,     // GIT_SHA (CI注入)
  // sentry-cli 上传 source map 时使用相同 release 值
  // 构建脚本: SENTRY_RELEASE=$(git rev-parse HEAD) sentry-cli releases files $SENTRY_RELEASE upload-sourcemaps dist/

  // ── 采样 ──
  // tracesSampleRate: 统一采样率 (0.0 ~ 1.0)
  tracesSampleRate: 0.1,    // 10% 性能追踪采样
  // tracesSampler: 按事务类型差异化采样 (优先级更高)
  tracesSampler: (samplingContext) => {
    const name = samplingContext.transactionContext.name;
    if (name?.startsWith('/api/health')) return 0;   // 健康检查不追踪
    if (name?.startsWith('/api/payment')) return 1;  // 支付 100% 追踪
    if (name?.startsWith('/_next')) return 0.01;     // 内部请求 1%
    return 0.10; // 默认 10%
  },

  // ── PII 脱敏 (beforeSend) ──
  beforeSend(event, hint) {
    const error = hint.originalException;

    // 清洗 URL 中的敏感参数
    if (event.request?.url) {
      event.request.url = sanitizeUrl(event.request.url);
    }
    if (event.request?.headers?.Referer) {
      event.request.headers.Referer = sanitizeUrl(event.request.headers.Referer);
    }

    // 清洗错误消息中的 token / 邮箱 / 密码
    if (event.message) {
      event.message = scrubPII(event.message);
    }
    if (event.exception?.values) {
      for (const v of event.exception.values) {
        if (v.value) v.value = scrubPII(v.value);
      }
    }

    // 清洗 breadcrumbs 中的表单输入
    if (event.breadcrumbs) {
      event.breadcrumbs = event.breadcrumbs.map(scrubBreadcrumb);
    }

    // 忽略已知噪声错误
    if (shouldIgnoreError(error)) return null;

    return event;
  },

  // ── 错误过滤 ──
  ignoreErrors: [
    // ResizeObserver 循环限制 — 浏览器行为，非应用错误
    /ResizeObserver loop (limit exceeded|completed with undelivered notifications)/,
    // 非 Error 类型的 Promise rejection (第三方脚本抛字符串)
    'Non-Error promise rejection captured',
    // 浏览器扩展注入的脚本错误
    /^Script error\.?$/,
    // 网络中断导致的 fetch 失败 (非 bug)
    'NetworkError when attempting to fetch resource.',
    // 用户取消导航 (不会影响功能)
    'AbortError: The user aborted a request.',
  ],

  // ── 忽略特定 URL 的错误 (第三方脚本/注入) ──
  denyUrls: [
    /extensions\//i,        // 浏览器扩展
    /^chrome:\/\//i,        // Chrome 内部
    /^moz-extension:\/\//i, // Firefox 扩展
    /cdn\.third-party\.com/,// 已知无问题的第三方CDN
  ],

  // ── Breadcrumbs ──
  // 自动记录: 点击、导航、XHR/fetch、控制台、DOM 变化
  // 用于还原错误发生前的用户操作路径
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.breadcrumbsIntegration({
      console: true,       // console.log/warn/error 自动 breadcrumb
      dom: { serializeAttribute: ['data-testid'] }, // 仅序列化安全属性
      fetch: true,         // fetch 请求自动 breadcrumb
      history: true,       // 路由变化 breadcrumb
      xhr: false,          // XHR 可以关闭(用 fetch)
    }),
  ],

  // ── 自定义 breadcrumb ──
  beforeBreadcrumb(breadcrumb, hint) {
    // 过滤 fetch breadcrumb 中的敏感数据
    if (breadcrumb.category === 'fetch' || breadcrumb.category === 'xhr') {
      if (breadcrumb.data?.url) {
        breadcrumb.data.url = sanitizeUrl(breadcrumb.data.url);
      }
      // 不记录请求正文(可能含密码)
      delete breadcrumb.data?.request_body;
    }
    return breadcrumb;
  },
});
```

### 3.2 PII 脱敏工具函数

```typescript
// 必须脱敏的参数名 (不区分大小写)
const SENSITIVE_PARAMS = [
  'password', 'passwd', 'pwd', 'secret',
  'token', 'access_token', 'refresh_token',
  'credit_card', 'card_number', 'cvv',
  'ssn', 'social_security',
  'email', 'phone', 'mobile',
  'api_key', 'apikey', 'private_key',
];

function sanitizeUrl(url: string): string {
  try {
    const u = new URL(url);
    let changed = false;
    for (const param of SENSITIVE_PARAMS) {
      if (u.searchParams.has(param)) {
        u.searchParams.set(param, '[FILTERED]');
        changed = true;
      }
    }
    return changed ? u.toString() : url;
  } catch {
    return url; // 无效 URL 直接返回
  }
}

function scrubPII(text: string): string {
  let result = text;
  // 邮箱
  result = result.replace(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g, '[EMAIL]');
  // JWT token (三段 base64，点分隔)
  result = result.replace(/eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+/g, '[JWT]');
  // credit card (13-19 位数字，可能有空格/横线)
  result = result.replace(/\b(?:\d[ -]*?){13,19}\b/g, '[CARD]');
  // SSN (xxx-xx-xxxx)
  result = result.replace(/\b\d{3}-\d{2}-\d{4}\b/g, '[SSN]');
  return result;
}

function scrubBreadcrumb(breadcrumb: Sentry.Breadcrumb): Sentry.Breadcrumb {
  // 表单输入: 如果是 password 类型，不记录值
  if (breadcrumb.category === 'ui.input') {
    if ((breadcrumb.data as any)?.inputType === 'password') {
      (breadcrumb.data as any).value = '[FILTERED]';
    }
  }
  return breadcrumb;
}

function shouldIgnoreError(error: unknown): boolean {
  if (!(error instanceof Error)) return false;
  // 第三方 iframe 跨域错误: 消息为字符串但没有 stack
  if (typeof error.message === 'string' && !error.stack) {
    return /^(Script error|Access is denied)/.test(error.message);
  }
  return false;
}
```

### 3.3 Source Map 管理策略演进

```
┌──────────────────────────────────────────────────────────────────┐
│                    Source Map 上传策略对比                          │
│                                                                  │
│  传统: sentry-cli + release + public URL                          │
│  ┌────────────────────┐                                          │
│  │ CI build            │                                         │
│  │ 1. SENTRY_RELEASE=$SHA                                     │  │
│  │ 2. webpack/rollup 生成 .map 文件                               │
│  │ 3. 构建完成后 sentry-cli upload-sourcemaps dist/               │
│  │    --release $SHA --url-prefix "~/"                            │
│  │ 4. 用户报错 → Sentry: "release=$SHA, filename=bundle.js"      │
│  │    → 通过 release+filename 匹配 .map                           │
│  │ 问题: map 不在 bundle 内，文件名匹配脆弱，CDN 缓存              │
│  └────────────────────┘                                          │
│                                                                  │
│  现代: Debug IDs (推荐)                                            │
│  ┌────────────────────┐                                          │
│  │ webpack/rollup      │                                         │
│  │ 1. 构建时: 生成 sourceMappingURL 指向带 Debug ID 的 .map     │  │
│  │ 2. Sentry plugin 自动注入: debugId: "xxxxxxxx-xxxx-..."      │  │
│  │    在 bundle 和 .map 中同时存在                                │
│  │ 3. 上传: 仅上传 .map 文件，Sentry 按 Debug ID 建立索引         │
│  │ 4. 匹配: 错误栈中的 debugId → 精确匹配 .map                   │
│  │    无需 release/url 匹配，完美解决部署顺序和缓存问题            │
│  │ 5. 不部署 .map 到 CDN: .map 仅存 Sentry，bundle 无 sourceURL  │
│  └────────────────────┘                                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```javascript
// vite.config.js — Debug ID 集成
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default {
  build: {
    sourcemap: true, // 必须开启
  },
  plugins: [
    sentryVitePlugin({
      org: 'my-org',
      project: 'frontend',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      // debug: true, // 本地调试用
      // 仅上传 .map 文件
      release: {
        // 可选: 同时创建 release (兼容旧版)
        name: process.env.SENTRY_RELEASE,
      },
      // sourcemaps.filesToDeleteAfterUpload: ['**/*.js.map']  // CI 中删除 map 避免泄露
    }),
  ],
};
```

### 3.4 错误分级与投递策略

```typescript
// 错误优先级矩阵
type ErrorSeverity = 'fatal' | 'error' | 'warning' | 'info';

interface ErrorReport {
  severity: ErrorSeverity;
  fingerprint: string;     // 相同 fingerprint → 同一条 issue
  message: string;
  stack?: string;
  breadcrumbs: BreadcrumbData[];
  tags: Record<string, string>;
  // 以下用于频率控制
  rateLimitKey?: string;   // 按 key 计数防刷
  sampleRate?: number;     // 采样率: fatal=1.0, error=0.8, warning=0.1
}

// fingerprint 设计: 同类错误归为一个 issue
function computeFingerprint(error: Error): string {
  // 取 error type + 堆栈前三帧 → 稳定指纹
  const stackLines = (error.stack || '').split('\n').slice(0, 4);
  const cleaned = stackLines
    .map(line => line.replace(/:\d+:\d+/g, '')) // 去掉行列号，防止同文件不同行分裂 issue
    .join('|');
  return `${error.name}:${simpleHash(cleaned)}`;
}
```

---

## 四、数据埋点架构 (字节跳动实践)

### 4.1 埋点全生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    埋点全生命周期 (7 阶段)                         │
│                                                                 │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│  │ 设计  │→│ 注册  │→│ 验证  │→│ 上报  │→│ ETL  │→│ 使用  │→│ 下线  │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
│     │         │         │         │         │         │         │
│     ▼         ▼         ▼         ▼         ▼         ▼         ▼
│  需求文档  元数据注册  验证引擎  SDK聚合   数据清洗  看板/分析  TTL过期
│  埋点方案  JSON Schema 本地+CI   批量上报  入库      告警     清理归档
│                                                                 │
│  核心原则:                                                       │
│  ① 所有埋点必须先注册元数据才允许上报 —— 没有后门                    │
│  ② SDK 内置管控逻辑 —— 终端自主计算采样/丢弃，节省传输成本          │
│  ③ 埋点元数据驱动 ETL 管道 —— 注册表即 Schema，变更即生效          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 埋点 SDK 架构设计

```typescript
// 埋点元数据注册中心
// 所有埋点事件必须在此注册才能上报
// 非注册事件 → SDK 直接丢弃 + 本地 warn

interface TrackMetadata {
  eventName: string;           // 事件名 (全局唯一)
  version: number;             // 事件版本 (变更时递增)
  description: string;         // 业务含义
  owner: string;               // 负责人
  priority: 'P0' | 'P1' | 'P2' | 'P3';  // 分级SLA
  ttl: number;                 // TTL (天), P3 级别 30 天后 pre-discard
  params: Record<string, {
    type: 'string' | 'number' | 'boolean' | 'array';
    required: boolean;
    description: string;
    enum?: unknown[];          // 枚举约束
  }>;
  sampleRate: number;          // 采样率 (0-1)
}

// 注册中心 (从服务端拉取，本地缓存 IndexedDB)
class TrackRegistry {
  private metadata: Map<string, TrackMetadata> = new Map();
  private lastSync = 0;

  async sync(): Promise<void> {
    if (Date.now() - this.lastSync < 600_000) return; // 10分钟缓存

    const resp = await fetch('/api/track-metadata?since=' + this.lastSync);
    const items: TrackMetadata[] = await resp.json();

    for (const meta of items) {
      this.metadata.set(meta.eventName, meta);
    }
    this.lastSync = Date.now();
  }

  has(eventName: string): boolean {
    return this.metadata.has(eventName);
  }

  get(eventName: string): TrackMetadata | undefined {
    return this.metadata.get(eventName);
  }
}

// 策略: 终端计算 + 合规丢弃
// SDK 在客户端完成采样、校验、过滤，减少无效数据传输
class TrackSDK {
  private registry = new TrackRegistry();
  private aggregator = new EventAggregator();

  async track(eventName: string, params: Record<string, unknown>) {
    // ① 合法性检查: 未注册事件直接丢弃
    const meta = this.registry.get(eventName);
    if (!meta) {
      if (process.env.NODE_ENV === 'development') {
        console.warn(`[TrackSDK] Unregistered event: ${eventName}`);
      }
      return; // 非注册事件不上报
    }

    // ② 采样判断 (终端计算，节省带宽)
    if (meta.sampleRate < 1) {
      const hash = this.hashSession(eventName);
      if ((hash % 10000) / 10000 >= meta.sampleRate) return;
    }

    // ③ 参数校验: 类型+必填+枚举
    const validated = this.validateParams(meta, params);
    if (!validated.valid) {
      if (process.env.NODE_ENV === 'development') {
        console.error(`[TrackSDK] Invalid params for ${eventName}:`, validated.errors);
      }
      // 生产环境: 上报质量监控事件 (非原始数据)
      this.reportQualityIssue(eventName, validated.errors);
      return; // 参数不合规 → 不上报
    }

    // ④ 合规丢弃: 检查隐私合规标记
    if (this.isPrivacyProhibited(params)) return;

    // ⑤ 进入聚合器 (批量上报)
    this.aggregator.push({ eventName, params: validated.cleaned, ts: Date.now() });
  }

  private validateParams(meta: TrackMetadata, params: Record<string, unknown>) {
    const errors: string[] = [];
    const cleaned: Record<string, unknown> = {};

    for (const [key, schema] of Object.entries(meta.params)) {
      const value = params[key];
      // 必填检查
      if (schema.required && (value === undefined || value === null)) {
        errors.push(`${key} is required`);
        continue;
      }
      if (value === undefined) continue;

      // 类型检查
      const actualType = Array.isArray(value) ? 'array' : typeof value;
      if (actualType !== schema.type) {
        errors.push(`${key}: expected ${schema.type}, got ${actualType}`);
        continue;
      }

      // 枚举约束
      if (schema.enum && !schema.enum.includes(value)) {
        errors.push(`${key}: ${value} not in enum [${schema.enum}]`);
        continue;
      }

      cleaned[key] = value;
    }

    return { valid: errors.length === 0, errors, cleaned };
  }

  private hashSession(eventName: string): number {
    // 与 session ID 组合 hash，同用户同一事件稳定采样
    const seed = `${this.getSessionId()}_${eventName}`;
    return this.djb2(seed);
  }

  private getSessionId(): string { /* ... */ }
  private djb2(str: string): number { /* ... */ }
  private isPrivacyProhibited(params: Record<string, unknown>): boolean { /* ... */ }
  private reportQualityIssue(eventName: string, errors: string[]): void { /* ... */ }
}
```

### 4.3 事件聚合与传输策略

```typescript
// 埋点聚合核心: 多个单点 Event → Applog 批量包
// 字节团队实践:
//   - JSON 批量上报: 100 events, 200KB → Protobuf: 80KB (体积减少60%)
//   - 批量上报频率: 10 秒 或 20 条 (先到先发)
//   - 消息队列缓冲: SDK 端 IndexedDB → 网关 → Kafka → Flink ETL → ClickHouse

// Protobuf vs JSON 性能对比 (字节团队实测)
// ┌────────┬──────────┬──────────┬──────────┬──────────┐
// │ 事件数  │ JSON     │ Protobuf │ 压缩比    │ 解析速度  │
// ├────────┼──────────┼──────────┼──────────┼──────────┤
// │ 1       │ 2KB      │ 1.2KB    │ 60%      │ 1.8x     │
// │ 10      │ 20KB     │ 8KB      │ 40%      │ 2.5x     │
// │ 100     │ 200KB    │ 80KB     │ 40%      │ 3x       │
// │ 1000    │ 2MB      │ 700KB    │ 35%      │ 4x       │
// └────────┴──────────┴──────────┴──────────┴──────────┘
// 结论: 事件越多，Protobuf 优势越明显。批量上报场景下 Protobuf 是必然选择。

class EventAggregator {
  private buffer: TrackEvent[] = [];
  private readonly MAX_SIZE = 20;
  private readonly FLUSH_MS = 10_000;
  private timer: ReturnType<typeof setInterval>;

  constructor() {
    this.timer = setInterval(() => this.flush(), this.FLUSH_MS);
    // 页面离开时最后 flush
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.flush();
    });
  }

  push(event: TrackEvent) {
    this.buffer.push(event);
    if (this.buffer.length >= this.MAX_SIZE) this.flush();
  }

  private flush() {
    if (this.buffer.length === 0) return;
    const batch = this.buffer.splice(0);

    // Applog 格式: 事件数组 + 公共字段
    const applog = {
      batch_id: crypto.randomUUID ? crypto.randomUUID() : `${Date.now()}-${Math.random()}`,
      session_id: this.getSessionId(),
      page: location.pathname,
      events: batch,
      ts: Date.now(),
    };

    this.send(applog);
  }

  private send(payload: unknown) {
    // 首选 Protobuf 编码 (体积小，性能好)
    // 降级: JSON
    const encoded = this.encodeProto(payload); // 你的 proto 编码函数
    // sendBeacon 不支持自定义 Content-Type，用 Blob
    navigator.sendBeacon(
      '/v1/applog',
      new Blob([encoded], { type: 'application/x-protobuf' })
    );
  }
}
```

### 4.4 埋点质量治理: 三维监控

```
┌─────────────────────────────────────────────────────────────────┐
│                    埋点质量三维监控                                │
│                                                                 │
│  维度 1: 脏数据                                                  │
│    ├─ 参数类型错误 (number 字段传了 string)                       │
│    ├─ 参数值超出枚举范围                                          │
│    ├─ 空值/undefined (required 字段缺失)                          │
│    └─ 异常值 (价格负数、年龄 > 150)                                │
│                                                                 │
│  维度 2: 类型错误                                                 │
│    ├─ 字段名拼写错误 (usr_id vs user_id)                          │
│    ├─ 版本不兼容 (v2 字段传入 v1 协议)                             │
│    ├─ 事件名未注册 → SDK 丢弃但需记录                             │
│    └─ JSON 解析失败 (编码错误)                                    │
│                                                                 │
│  维度 3: 丢失/重复                                                │
│    ├─ 事件丢失率 = (服务端接收 - 应发) / 应发                      │
│    │   (通过校验码/序列号对比)                                    │
│    ├─ 重复率 = (重复事件 / 总事件)                                │
│    │   (batch_id 去重)                                           │
│    └─ 完整率 = 必传参数完整的事件数 / 总事件                       │
│                                                                 │
│  告警规则:                                                        │
│    - 脏数据率 > 5%: P2 告警 → 通知事件 owner                       │
│    - 丢失率 > 1%: P1 告警 → 排查 SDK/网关/消息队列                 │
│    - 某事件完全消失 > 10 分钟: P0 告警 → 立刻排查                   │
│    - 某 P3 事件 30 天无调用: 自动标记预下线 → 通知 owner            │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 分级 SLA 与生命周期管控

```
埋点分级:

  P0 (核心业务):  TTL 永久, 采样 100%, 上下游 SLA 99.9%
    例: 支付成功/失败、登录/注册、核心转化
    丢失率 < 0.1%

  P1 (重要业务):  TTL 90天, 采样 50%, SLA 99%
    例: 商品浏览、搜索、加购

  P2 (一般分析):  TTL 30天, 采样 10%, SLA 95%
    例: 页面停留时长、功能点击

  P3 (实验/调试):  TTL 30天 (pre-discard 30天暂存), 采样 1%, SLA best-effort
    例: A/B 实验中间指标、功能灰度数据
    30天内无调用 → 自动进入 pre-discard 队列
    再 30 天无人恢复 → 永久删除

管控手段:
  ① 注册即管控: 未注册事件 SDK 直接丢弃
  ② 元数据驱动 ETL: 注册的 schema → ETL 自动解析 + 校验
  ③ TTL 自动清理: 过期事件自动归档 → 冷存储 → 删除
  ④ 变更审计: 每次字段变更 → 评审 + Owner 确认
```

### 4.6 动态埋点验证引擎 (字节方案)

```
验证引擎架构:

  规则生成器 (Rule Generator)
    │ 读取埋点元数据 → 自动生成验证规则 (类型/必填/枚举/范围)
    │ 输出: 规则集 (JSON)
    ▼
  规则选择器 (Rule Selector)
    │ 按事件名 → 加载对应规则
    │ 支持灰度: 规则版本与事件版本解耦
    ▼
  埋点验证器 (Event Validator)
    │ 在 SDK 端验证 + 服务端 ETL 二次验证
    │ SDK 侧: 实时拦截 + 本地 warn (dev 模式)
    │ ETL 侧: 聚合统计 + 质量报表
    ▼
  埋点推送器 (Event Pusher)
    │ 验证通过 → 编码 → 聚合 → 批量上报
    │ 验证失败 → 释放 + 记录 (不上报原始数据)

动态实时处理引擎 (advanced):
  SQL/图形化配置
    → Groovy 规则脚本 (业务方可自行编写)
    → 热编译 (GroovyClassLoader 动态加载)
    → 拓扑重构 (重新绑定数据流节点)
  → 无需发版即可修改埋点处理逻辑
```

---

## 五、会话回放 (Session Replay)

### 5.1 rrweb 架构核心

```
┌─────────────────────────────────────────────────────────────────┐
│                    rrweb 录制管线                                 │
│                                                                 │
│  Step 1: Full Snapshot                                          │
│    serializeNodeWithId(node, { doc, mirror })                    │
│    → 遍历整个 DOM 树 → 为每个 node 生成唯一 id                    │
│    → 序列化为 JSON: { type: NodeType, id, tagName, attributes,  │
│                       childNodes[], textContent, cssStyles }     │
│                                                                 │
│  Step 2: Incremental Snapshots (增量快照)                        │
│    MutationObserver({                                            │
│      childList: true,    // 节点增删                             │
│      attributes: true,   // 属性变化                             │
│      characterData: true,// 文本变化                             │
│      subtree: true       // 全子树监听                           │
│    })                                                            │
│    → 每个 mutation 转为增量 event: { type, targetId, payload }   │
│                                                                 │
│  Step 3: 事件序列化                                               │
│    Full Snapshot (type=2) → Incremental Snapshots (type=3)...    │
│    → Meta (type=4): 视口尺寸、设备信息                            │
│    → Custom (type=5): 用户自定义事件                              │
│                                                                 │
│  Step 4: 压缩存储                                                 │
│    LZ-string / gzip 压缩                                         │
│    → 典型 1 分钟录制: 50-200KB (取决于 DOM 复杂度)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 录制初始化 (带隐私保护)

```typescript
import { record } from 'rrweb';

const stopFn = record({
  emit(event) {
    // 每个录制事件在此处理
    // 生产: 批量 + sendBeacon 上报
    replayBuffer.push(event);
    if (replayBuffer.length >= 50) flushReplayBuffer();
  },

  // ── 隐私保护 (必须配置) ──

  // 方式 1: 元素级遮罩 (推荐)
  // 给敏感元素添加 data-rr-mask 属性, rrweb 自动遮蔽文本为 '*'
  // <input data-rr-mask type="password" />
  // <div data-rr-mask>身份证号码: 110101199001011234</div>
  maskTextSelector: '[data-rr-mask]',

  // 方式 2: 全局文本遮罩
  // 所有文本内容转为 '*' (强隐私保护, 但丢失行为上下文)
  // maskAllText: true,

  // 方式 3: 屏蔽所有输入
  maskAllInputs: true,

  // 方式 4: 过滤特定元素 (完全跳过，不进入录制)
  blockSelector: '.no-record',

  // 方式 5: CSS :visited 伪类 —— 不暴露用户浏览历史
  // rrweb 内置忽略 visited 样式

  // ── 隐私过滤钩子 ──
  maskTextFn: (text, element) => {
    // 函数级过滤: 身份证/手机号/银行卡号 格式检测
    if (/\b\d{17}[\dXx]\b/.test(text) ||      // 身份证
        /\b1[3-9]\d{9}\b/.test(text) ||       // 手机号
        /\b\d{16,19}\b/.test(text)) {          // 银行卡
      return text.replace(/\d/g, '*');
    }
    return text;
  },

  // ── 性能优化 ──
  sampling: {
    // 鼠标移动采样: 降低高频事件的数据量
    mousemove: true,          // 录制鼠标移动
    mouseInteraction: true,   // 录制点击/滚动/输入
    scroll: 150,              // 滚动事件节流 (ms)
    // 媒体交互: 播放/暂停
    media: 800,               // 媒体事件节流
    // Canvas / WebGL 录制: 一般关闭 (数据量巨大)
    canvas: 1,                // fps (1=每秒1帧)
  },

  // ── 录制限制 ──
  // 限制检查间隔: 防止长时间录制导致内存泄漏
  checkoutEveryNms: 300_000,  // 每 5 分钟做一次全量快照 (防止增量链过长)
  checkoutEveryNth: 200,      // 每 200 个增量事件做一次全量快照
});

// 技巧: 录制控制在用户行为触发后
// 不要页面加载就录制所有用户 —— 浪费存储
// 策略: 错误发生后录制 (Sentry 集成) / 关键转化页面录制 / 按比例采样
```

### 5.3 rrweb 回放引擎

```typescript
// rrweb 回放使用 requestAnimationFrame 高精度计时
// 而非 setTimeout —— 后者最小精度 ~4ms，积累误差明显

import { Replayer } from 'rrweb';

const replayer = new Replayer(events, {
  root: document.getElementById('replay-container')!,

  // ── 速度控制 ──
  speed: 1,      // 1x, 2x, 4x, 0.5x

  // ── 跳过静默期 ──
  skipInactive: true,  // 自动跳过无操作时段

  // ── 鼠标轨迹 ──
  showWarning: true,   // 警告框提示用户此为回放
  mouseTail: {
    duration: 500,     // 鼠标轨迹持续时间 (ms)
    strokeStyle: '#3B82F6',
  },

  // ── 隐私回放 ──
  // 回放时也可以后置脱敏 (录制时未脱敏的场景)
  maskTextClass: 'rr-mask',
  blockClass: 'rr-block',

  // ── 插件 ──
  plugins: [
    // rrweb/plugins/console: 回放控制台日志
  ],
});

// 回放控制
replayer.play();
replayer.pause();
replayer.goto(5000);  // 跳到 5 秒位置

// 性能: rrweb 回放延迟 ~44ms (网络边缘场景), 一般可忽略
// 大规模并发回放时建议 workers 渲染 + 截图缓存
```

### 5.4 大规模存储优化

```typescript
// 百万级会话的存储策略

// 1. 分层存储
//    - 热数据 (近7天): Redis/SSD, 即时回放
//    - 温数据 (7-30天): S3/对象存储, 异步取回
//    - 冷数据 (>30天): 归档/压缩/删除
//
// 2. 去重引擎
//    - 相同 DOM 结构的录制片段 → hash → 只存一份 + 引用计数
//    - 典型场景: 列表页无限滚动, 大量结构相同的 item
//    - 压缩比: 50-80% (取决于页面重复度)
//
// 3. 索引策略
//    - session_id → 录制文件定位
//    - user_id + timestamp → session_id (反向索引)
//    - error_fingerprint → [session_ids] (Sentry 集成)
//
// 4. 存储成本公式
//    日均录制时长 (小时) × 平均速率 (KB/h) × 保留天数
//    = 100,000 sessions × 30s avg × 200KB/min × 30 days
//    = 100,000 × 0.5 × 200 × 30
//    ≈ 300 GB/mo (原始) → ~90 GB/mo (压缩后)
```

---

## 六、前端日志系统

### 6.1 分级策略

```typescript
// 日志等级矩阵
// ┌──────────┬──────────┬────────────────────────────┐
// │ Level    │ 场景      │ 采样率 / 操作               │
// ├──────────┼──────────┼────────────────────────────┤
// │ debug    │ 开发调试  │ 0% (生产不记录)             │
// │ info     │ 关键流程  │ 5-10% 采样                  │
// │ warn     │ 潜在问题  │ 80% 采样                    │
// │ error    │ 可恢复错误│ 100% 采样 + Sentry 关联     │
// │ fatal    │ 不可恢复  │ 100% 采样 + Sentry + 告警   │
// └──────────┴──────────┴────────────────────────────┘

type LogLevel = 'debug' | 'info' | 'warn' | 'error' | 'fatal';

interface StructuredLog {
  level: LogLevel;
  message: string;
  timestamp: number;
  sessionId: string;
  pageUrl: string;
  // 结构化字段 (不是字符串拼接!)
  context?: Record<string, unknown>;
  // 与 Error 关联
  errorFingerprint?: string;
  // 与 Sentry Trace 关联
  traceId?: string;
  // 与业务操作关联
  operationId?: string;
}

class FrontendLogger {
  private buffer: StructuredLog[] = [];
  private transporter = new RUMTransporter('/v1/logs', 1);

  // 采样率配置: 按 level 分级
  private sampleRates: Record<LogLevel, number> = {
    debug: 0,
    info: 0.05,
    warn: 0.8,
    error: 1,
    fatal: 1,
  };

  log(level: LogLevel, message: string, context?: Record<string, unknown>) {
    // 采样
    if (Math.random() > this.sampleRates[level]) return;

    // 结构化日志 (非 fprintf 风格拼接)
    const entry: StructuredLog = {
      level,
      message,
      timestamp: Date.now(),
      sessionId: this.getSessionId(),
      pageUrl: location.href,
      context,
    };

    // fatal/error 同步上报, 其余批量
    if (level === 'fatal') {
      this.transporter.flush(); // 先 flush 历史
      navigator.sendBeacon('/v1/logs', new Blob([JSON.stringify([entry])]));
    } else {
      this.buffer.push(entry);
      if (this.buffer.length >= 30 || level === 'error') {
        this.flush();
      }
    }
  }

  debug(msg: string, ctx?: Record<string, unknown>) { this.log('debug', msg, ctx); }
  info(msg: string, ctx?: Record<string, unknown>) { this.log('info', msg, ctx); }
  warn(msg: string, ctx?: Record<string, unknown>) { this.log('warn', msg, ctx); }
  error(msg: string, error?: Error, ctx?: Record<string, unknown>) {
    this.log('error', msg, {
      ...ctx,
      errorName: error?.name,
      errorMessage: error?.message,
      errorStack: error?.stack?.split('\n').slice(0, 5).join('\n'),
      errorFingerprint: error ? computeFingerprint(error) : undefined,
    });
    // error 级别同步送 Sentry
    if (error) Sentry.captureException(error, { extra: ctx });
  }
  fatal(msg: string, error?: Error, ctx?: Record<string, unknown>) {
    if (error) ctx = { ...ctx, fatalError: true };
    this.log('fatal', msg, ctx);
    if (error) Sentry.captureException(error, { level: 'fatal', extra: ctx });
  }

  private flush() { /* sendBeacon batch */ }
  private getSessionId(): string { /* ... */ }
}
```

### 6.2 敏感信息脱敏 (beforeSend for Logs)

```typescript
// 日志脱敏规则: 与错误监控共享 sanitizeUrl / scrubPII 函数
// 但日志脱敏范围更大 —— 涵盖 localStorage / cookie / input

function sanitizeLogContext(ctx: Record<string, unknown>): Record<string, unknown> {
  const sanitized: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(ctx)) {
    // ① 黑名单 key: 直接丢弃值
    if (/password|token|secret|apikey|authorization/i.test(key)) {
      sanitized[key] = '[REDACTED]';
      continue;
    }

    // ② URL 类: 清洗 query params
    if (typeof value === 'string' && /^https?:\/\//.test(value)) {
      sanitized[key] = sanitizeUrl(value);
      continue;
    }

    // ③ 深度字符串: PII 正则替换
    if (typeof value === 'string') {
      sanitized[key] = scrubPII(value);
      continue;
    }

    // ④ 嵌套对象: 递归
    if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
      sanitized[key] = sanitizeLogContext(value as Record<string, unknown>);
      continue;
    }

    sanitized[key] = value;
  }

  return sanitized;
}

// 生产场景: 日志的 URL 必须删除 query 参数中的 token
// /api/user?token=abc123&page=2 → /api/user?token=[FILTERED]&page=2
```

### 6.3 传输策略: 离线优先

```typescript
// 前端日志面临的核心问题: 网络不可靠
// 解决方案: IndexedDB 离线缓冲 + 指数退避重试

class OfflineLogBuffer {
  private db: IDBDatabase | null = null;

  async init() {
    const req = indexedDB.open('log-offline', 1);
    req.onupgradeneeded = () => {
      req.result.createObjectStore('logs', { keyPath: 'id', autoIncrement: true });
    };
    this.db = await new Promise<IDBDatabase>((resolve) => { req.onsuccess = () => resolve(req.result); });
  }

  async enqueue(entry: StructuredLog) {
    if (!this.db) await this.init();
    const tx = this.db!.transaction('logs', 'readwrite');
    tx.objectStore('logs').add({
      ...entry,
      retries: 0,
      nextRetry: Date.now(),
    });
  }

  async flush() {
    if (!this.db) return;
    const tx = this.db.transaction('logs', 'readwrite');
    const store = tx.objectStore('logs');
    const all = await new Promise<any[]>((resolve) => {
      const req = store.getAll();
      req.onsuccess = () => resolve(req.result);
    });

    const chunk = all.slice(0, 50); // 每次最多 50 条
    if (chunk.length === 0) return;

    try {
      await fetch('/v1/logs', {
        method: 'POST',
        body: JSON.stringify(chunk),
        headers: { 'Content-Type': 'application/json' },
      });
      // 成功 → 删除已发送
      const delTx = this.db.transaction('logs', 'readwrite');
      for (const item of chunk) {
        delTx.objectStore('logs').delete(item.id);
      }
    } catch {
      // 失败 → 更新重试次数和下次重试时间 (指数退避)
      const updateTx = this.db.transaction('logs', 'readwrite');
      for (const item of chunk) {
        const retries = item.retries + 1;
        if (retries > 3) {
          // 超过 3 次 → 丢弃 (防止僵尸数据占用)
          updateTx.objectStore('logs').delete(item.id);
        } else {
          updateTx.objectStore('logs').put({
            ...item,
            retries,
            nextRetry: Date.now() + Math.pow(2, retries) * 60_000, // 1min, 2min, 4min
          });
        }
      }
    }
  }

  // 在线时定时 flush (页面加载时自动重试)
  scheduleFlush(intervalMs: number = 60_000) {
    setInterval(() => {
      if (navigator.onLine) this.flush();
    }, intervalMs);
  }
}
```

---

## 七、可观测性架构决策树

### 7.1 错误监控: Sentry vs DataDog vs 自建

```
问题: 错误监控选什么?

                开始
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
  团队<5人    团队5-20人   团队>20+
      │           │           │
      ▼           ▼           ▼
  Sentry      Sentry      Sentry/DataDog
  (免费层      (Team        (Enterprise
   足够)       Plan)       选型)

Sentry 优势:
  - JS / React / Vue / Next.js 一等公民
  - Source Map 自动化 (Debug IDs)
  - Session Replay 集成
  - 免费层: 5000 errors/月 (小项目足够)
  - Release tracking + GitHub/GitLab integration
  - 缺点: 仅错误 (不覆盖全量 APM)

DataDog RUM 优势:
  - 全链路: 错误 + 性能 + 日志 + 基础设施 统一看板
  - 强大的自定义仪表盘和告警
  - 支持 OpenTelemetry 标准
  - 缺点: 贵 (按15天数据保留 + host计费)
        前端 RUM 不是其核心优势领域

自建什么时候合理:
  ✓ 法律要求数据不出境/自控 (金融/政务/医疗)
  ✓ 日志量巨大 (500M+ events/day)，SaaS 费用不可接受
  ✓ 需要与内部系统深度整合 (埋点平台统一)
  ✗ 团队 < 10 人 (运维成本 > SaaS 费用)
```

### 7.2 RUM 服务商对比

| 服务 | 价格 | 核心优势 | 核心劣势 | 适用场景 |
|------|------|---------|---------|---------|
| **Vercel Analytics** | $0 (免费) / Pro $20 | 零配置; 隐私优先(无cookie); Web Vitals + 访问量 | 仅 Vercel 部署; 无错误追踪 | Next.js/Vercel 生态 |
| **Cloudflare Web Analytics** | $0 (免费) / Pro $10 | 边缘计算; 零配置; 隐私合规(GDPR) | 仅 Cloudflare 部署; 无错误追踪 | Cloudflare 用户 |
| **Google Analytics 4** | $0 (免费) | 免费; 强大受众; 与广告联动 | 隐私争议(第三方cookie); 非 CWV 原生 | 需要受众分析/广告 |
| **Sentry RUM** | $26/mo 起 | Source Map; Error + Perf 统一; Session Replay | 成本随事件量线性增长 | 以错误追踪为主的团队 |
| **DataDog RUM** | $1.50/1000 sessions | 全栈观测; 与后端 APM 联动 | 贵; 复杂度高; 数据保留短 | 已有 DataDog 采购的大团队 |
| **SpeedCurve** | $300/mo 起 | 专业合成监控; 视觉回归; 性能预算 | 仅合成 + RUM，无错误 | 性能专项团队 |
| **自建 (web-vitals + ClickHouse)** | 基础设施 ~$200-500/mo | 完全数据控制; 无上限; 定制化 | 需维护; 需开发看板 | 大规模/合规强要求 |

### 7.3 会话回放: 自建 vs 商用

```
会话回放决策:

  Sentry Session Replay:
    ✓ 与错误自动关联 (点击错误 → 直接看回放, 无手动查找)
    ✓ 隐私遮罩开箱即用
    ✓ 60分钟留存已覆盖大量场景
    ✗ 无功能标记 (Feature Flag 关联)
    ✗ 高级搜索 (按事件/文本搜索) 需升级
    $ Team plan 含 50k replays

  LogRocket:
    ✓ 功能标记 + A/B 测试集成
    ✓ UI 体验最佳
    ¥ 按 session 量收费, 中大型贵

  FullStory:
    ✓ 全量录制 (无采样丢弃)
    ✓ "愤怒点击" / "死亡滚动" 自动检测
    ✓ 用户会话搜索 (按操作路径)
    ¥ 价格高 (未公开), 通常 Enterprise

  PostHog (开源):
    ✓ 开源自建 (Apache 2.0)
    ✓ 事件分析 + Feature Flag + Session Replay 统一
    ✓ Copilot 定价透明
    ✗ 自建运维成本
    ✗ 回放 UX 不如专用产品

  rrweb 自建:
    ✓ 完全控制数据
    ✓ 零 License 费
    ✓ 深度定制 (特定组件录制 / 自定义播放器)
    ✗ 需要建设存储/索引/回放界面/安全管控
    ✗ 团队需投入 1-2 人持续迭代

决策框架:
  ┌─ 团队 > 20 人 + 数据合规强需求 ──→ rrweb 自建 + ClickHouse/S3
  ├─ 已有 Sentry ──→ Sentry Session Replay (零额外集成)
  ├─ 需要 Feature Flag + 事件分析 ──→ PostHog
  ├─ UX 研究/产品分析为主 ──→ FullStory / LogRocket
  └─ 预算敏感 + 小团队 ──→ Sentry Session Replay > PostHog Cloud
```

### 7.4 采样策略汇总

```typescript
// 不同信号类型的最佳采样策略
const SAMPLING_CONFIG = {
  // Core Web Vitals: 100% 采集 (体积小，数据无偏见)
  cwv: { sampleRate: 1.0, reason: 'CWV 要求 p75，采样导致偏差' },

  // 性能追踪 (traces): 10% 采样 (体积中，代表性足够)
  traces: { sampleRate: 0.10, reason: '10% traces 已有统计显著性' },

  // 错误: 100% 采集 + fingerprint 频率限制防止刷屏
  errors: {
    sampleRate: 1.0,
    rateLimitPerFingerprint: 100,  // 每种错误每分钟最多 100 条
    reason: '错误不应采样，但需要防刷',
  },

  // 用户行为埋点 (P0/P1 核心): 全量
  coreEvents: { sampleRate: 1.0 },

  // 用户行为埋点 (P2/P3 分析): 采样
  analyticsEvents: { sampleRate: 0.10 },

  // session replay: 按错误触发 或 按比例采样
  sessionReplay: {
    errorTriggered: 1.0,       // 有错误 → 100% 录制
    randomSample: 0.05,        // 无错误 → 5% 随机
    maxDuration: 300_000,      // 最多 5 分钟
  },

  // 日志: 分级采样
  logs: {
    debug: 0,
    info: 0.05,
    warn: 0.80,
    error: 1.0,
    fatal: 1.0,
  },
};

// 成本优化公式:
// 日均成本 = events/day × avg_size × cost_per_GB
//
// 1000 万 events / day × 2KB × $0.50/GB = ~$10/day
// 加入采样: 100 万 events / day × 2KB × $0.50/GB = ~$1/day
// 效果: 10x 成本降低, 且对于趋势分析误差 < 3%
```

---

## 八、关键反模式 (Production Incidents)

### 反模式 1: 使用 `unload` / `beforeunload` 发送数据

```
❌ NEVER:
  window.addEventListener('unload', () => {
    navigator.sendBeacon('/analytics', data);  // iOS Safari 不触发 unload
  });
  window.addEventListener('beforeunload', () => {
    fetch('/analytics', { method: 'POST', body: data });  // 破坏 bfcache
  });

✅ ALWAYS:
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') flush();
  });
  window.addEventListener('pagehide', (e) => {
    if (!e.persisted) flush(); // 非 bfcache 才 flush
  });

后果: unload → iOS 丢失 30-50% 数据; beforeunload → bfcache 命中率归零
```

### 反模式 2: CLS 上报使用 `metric.value` 而非 `metric.delta`

```
❌ NEVER:
  onCLS(metric => send({ cls: metric.value }));
  // 每次回调 value 都是累计值 → CLS 虚高 3-10x

✅ ALWAYS:
  onCLS(metric => send({ cls: metric.delta }));
  // delta = 当期增量，正确反映单次布局偏移

后果: CLS 虚高 → 告警误报 → 性能团队无效排查 → 狼来了效应
```

### 反模式 3: 日志用字符串拼接而非结构化 JSON

```
❌ NEVER:
  console.log(`[${Date.now()}] User ${userId} clicked ${buttonName}`)
  // 难以解析、搜索、聚合

✅ ALWAYS:
  logger.info('button_click', { userId, buttonName, pageUrl, componentId });
  // 结构化: 按字段搜索/聚合/告警; 支持自动索引

后果: 故障排查时 grep 日志 30 分钟 → 改结构化后 30 秒
```

### 反模式 4: 无 PII 脱敏上报

```
❌ NEVER:
  Sentry.captureException(new Error(`Payment failed for ${email}`));
  // 或请求 URL 包含 access_token 直接上报

✅ ALWAYS:
  Sentry.captureException(new Error('Payment failed'), {
    extra: { email: scrubPII(email), url: sanitizeUrl(url) }
  });
  // 或: Sentry.init({ beforeSend: scrubEvent })

后果: 用户邮箱 / token 出现在 Sentry issue → GDPR 违规 → 法律风险
```

### 反模式 5: 每次事件单独发 HTTP 请求

```
❌ NEVER:
  track('click', { btn: 'submit' });  // 内部: fetch('/track', body)
  track('view',  { page: 'home' });   // 内部: fetch('/track', body)
  // 每个埋点一个 HTTP 请求 → 连接数爆炸

✅ ALWAYS:
  tracker.push('click', { btn: 'submit' });  // 进入队列
  tracker.push('view', { page: 'home' });    // 进入队列
  // 10秒 或 20条 → 一次 sendBeacon 批量发送

后果: 密集操作页面 (滑动/游戏) → 瞬时 200+ 并发请求 → 浏览器限制 → 数据丢失
```

### 反模式 6: PerformanceObserver 未处理 `buffered` 选项

```
❌ NEVER:
  const po = new PerformanceObserver((list) => {
    // 只拿到 observer 创建之后的 entry
    list.getEntries().forEach(report);
  });
  po.observe({ type: 'largest-contentful-paint' });
  // LCP 可能在 observer 创建前就触发了 → 丢失

✅ ALWAYS:
  const po = new PerformanceObserver((list) => {
    // 首次回调包含 buffer 中的历史 entry
    list.getEntries().forEach(report);
  });
  po.observe({ type: 'largest-contentful-paint', buffered: true });
  // ↑ buffered: true 确保拿到创建前的指标

后果: LCP 丢失率 15-40% (取决于脚本加载时机) → RUM 数据严重偏低
```

### 反模式 7: 采样使用 `Math.random()` 而非 session 稳定哈希

```
❌ NEVER:
  if (Math.random() < 0.1) track(event);
  // 同一 session 内, 事件 A 入选 但事件 B 落选
  // → 用户行为链断裂, 无法做漏斗分析

✅ ALWAYS:
  const hash = djb2(sessionId);
  if ((hash % 100) < 10) track(event);
  // 同一 session 始终入选或落选 → 用户行为链完整

后果: 转化漏斗数据无意义 (进入 1000 人, 但购买事件只采到 300 人 → 转化率 = ?)
```

### 反模式 8: 生产环境暴露 Source Map

```
❌ NEVER:
  // webpack output: /static/js/main.abc123.js
  //          + /static/js/main.abc123.js.map  ← 部署到 CDN
  // 任何人 DevTools 都能还原源码

✅ ALWAYS:
  // 构建: 生成 .map 文件
  // 部署: 仅部署 .js 文件到 CDN
  // 上传: .map 文件仅上传到 Sentry (非公开)
  // 代码最后不带 sourceMappingURL 注释
  // 匹配: 通过 Debug ID 或 release + artifact 绑定

后果: 攻击者拿到源码 → 发现 API 端点/签名算法 → 安全漏洞
```

---

## 附录 A: OpenTelemetry 前端集成简介

```typescript
// OTel JS 前端: 处于早期阶段，生产级推荐 Sentry
// 但趋势明确: OTel 是未来标准，Sentry 也在向其靠拢

import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { ZoneContextManager } from '@opentelemetry/context-zone';

const provider = new WebTracerProvider({
  resource: { 'service.name': 'frontend-app' },
});

// 关键安全实践: 不要直接暴露 OTLP endpoint 给浏览器
// 通过 Edge Function Worker 代理
provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: '/api/otel/traces',  // 同源代理，非直接 OTLP
    })
  )
);

provider.register({ contextManager: new ZoneContextManager() });
```

---

## 附录 B: 能力检查清单 (Observability Maturity Model)

```
Level 1 (基础):
  □ Google Analytics 4 已部署
  □ console.error ← 线上唯一"错误监控"
  □ 无 Source Map, 线上报错只看到 minified 代码
  □ Core Web Vitals: "听说过，没采集"

Level 2 (入门):
  □ Sentry 已接入, release tracking 已配置
  □ Source Map 通过 Debug IDs 自动关联
  □ web-vitals: LCP/CLS 已采集并上报
  □ 已知关键页面的 p75 LCP

Level 3 (成熟):
  □ 完整的 CWV 监控: LCP/INP/CLS + attribution
  □ 错误被 fingerprint 聚合, 同类不刷屏
  □ PII 脱敏规则覆盖日志 + breadcrumbs
  □ Session Replay 在错误发生时自动录制
  □ Lighthouse CI 在 PR 时检查性能回归

Level 4 (先进):
  □ RUM 仪表盘: p75 by route/device/geo
  □ 错误与 CWV 关联: 错误率提升 1% → LCP p75 +200ms?
  □ 埋点平台: 全生命周期管理, 注册→上报→ETL→下线
  □ 埋点质量自动监控 (脏数据/丢失/类型错误)
  □ 动态规则引擎 (热编译 Groovy → 无发版改埋点逻辑)
  □ 成本优化: 冷热分层存储, 采样策略精细化

Level 5 (极致):
  □ 自建 RUM 管道: web-vitals → ClickHouse → Grafana
  □ AI 辅助异常检测: 指标异常自动归因
  □ Session Replay + 错误 自动关联, 一键定位
  □ OpenTelemetry 标准统一前后端链路
  □ 性能预算从 CWV 扩展到交互体验预算 (响应时间/动画帧率)
```

---

> 文档版本: v1.0
> 适用场景: 场景 B (架构设计), 场景 F (代码审查) 可观测性部分
> 关联文档: `references/decisions.md` (技术选型), `references/checklist.md` (检查清单)
> 蒸馏时间: 2026-06
