# PWA 离线架构与 Service Worker 体系

> 本文档整理 PWA（Progressive Web Application）离线架构的完整知识体系，
> 覆盖 Service Worker 生命周期、五种规范缓存策略、离线数据存储及 PWA 选型决策。
> 基于 Chrome DevTools 文档、Workbox 官方指南、Jake Archibald Offline Cookbook 及一线生产经验。

---

## 一、Service Worker 生命周期

### 完整生命周期图

```
                        ┌──────────────┐
                        │  Parsed/Idle  │
                        └──────┬───────┘
                               │ navigator.serviceWorker.register('/sw.js')
                               ▼
                        ┌──────────────┐
                        │  Installing   │  ← install 事件触发
                        │               │    在此阶段预缓存关键资源
                        │  waitUntil()  │    如果失败→Redundant（冗余态）
                        └──────┬───────┘
                               │ 安装完成
                               ▼
                  ┌────────────────────────┐
                  │       Installed        │  ← 等待旧 SW 释放所有页面
                  │   (Waiting to Activate) │    skipWaiting() 可跳过等待
                  └───────┬────────────────┘
                          │ pages controlled by old SW close OR self.skipWaiting()
                          ▼
                  ┌────────────────────────┐
                  │      Activating        │  ← activate 事件触发
                  │                        │    清除旧版本缓存
                  │    cleanup old caches   │    接管无 client 的页面
                  │    clients.claim()      │    clients.claim() 立即接管所有页面
                  └────────┬───────────────┘
                           │ 激活完成
                           ▼
                  ┌────────────────────────┐
                  │       Activated        │  ← 完全控制页面
                  │                        │    处理 fetch/sync/push 事件
                  │   fetch / sync / push   │    空闲时可能被浏览器终止
                  │   message events        │    有新 SW 注册时重新进入 Installing
                  └────────────────────────┘
```

### install 事件 — 预缓存关键资源

```javascript
// sw.js
const CACHE_NAME = 'app-v2'; // 版本化缓存名
const PRECACHE_URLS = [
  '/',
  '/static/css/main.abc123.css',
  '/static/js/main.def456.js',
  '/offline.html',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => {
        console.log('[SW] Precaching app shell');
        return cache.addAll(PRECACHE_URLS);
      })
      .then(() => {
        // 强制新 SW 跳过等待阶段
        // ⚠️ 直接 skipWaiting 可能导致活跃页面使用新 SW 但旧 HTML
        //    最佳实践：通知用户刷新或等待用户自然关闭旧页面
        return self.skipWaiting();
      })
  );
});
```

### activate 事件 — 清理旧缓存 + 接管页面

```javascript
self.addEventListener('activate', (event) => {
  const validCaches = [CACHE_NAME];

  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => !validCaches.includes(name))
          .map((name) => {
            console.log('[SW] Deleting old cache:', name);
            return caches.delete(name);
          })
      );
    }).then(() => {
      // 接管所有未被控制的页面
      // 注意：clients.claim() + skipWaiting() 组合确保更新立即生效
      return self.clients.claim();
    })
  );
});
```

### fetch 事件 — 拦截所有网络请求

```javascript
self.addEventListener('fetch', (event) => {
  // 仅拦截 GET 请求（POST/PUT/DELETE 不应缓存）
  if (event.request.method !== 'GET') return;

  // 仅拦截同源请求（跨域需额外处理）
  const url = new URL(event.request.url);
  if (url.origin !== self.location.origin) return;

  // 不拦截 chrome-extension:// 等非 HTTP(S)
  // 不拦截 CDN 预检请求（除非你有对应的缓存策略）
});
```

### 缓存版本管理策略

```javascript
// 方案 A：语义化版本号
const CACHE_VERSION = '3.2.1';
const CACHE_NAME = `app-static-${CACHE_VERSION}`;

// 方案 B：构建时注入哈希（推荐）
// 由构建工具（webpack/vite 插件）生成
// const CACHE_NAME = 'app-${BUILD_HASH}';

// 方案 C：App Shell + 内容动态缓存分离
const SHELL_CACHE = 'shell-v1';     // 永远不会清理，除非新的 install
const DATA_CACHE = 'data-dynamic';  // 基于 LRU 或过期时间清理
const IMAGE_CACHE = 'images';       // 使用 Workbox ExpirationPlugin
```

### 更新策略终极指南

| 场景 | skipWaiting | clients.claim | 效果 |
|------|------------|---------------|------|
| **内容网站（新闻/博客）** | 建议 | 建议 | 用户立即看到最新内容，页面重载即可生效 |
| **编辑器/表单应用** | 禁止 | 禁止 | 避免用户在填写表单时被强制更新导致数据丢失 |
| **金融/交易应用** | 禁止 | 建议(激活时) | 新页面使用新 SW，当前页面保持不变 |
| **实时通信应用** | 延迟(用户确认后) | 延迟 | 通知用户有新版本可用，由用户决定何时更新 |

```javascript
// 最佳实践：通知用户更新，而非强制
self.addEventListener('install', (event) => {
  // 不要在这里 skipWaiting()
});

self.addEventListener('message', (event) => {
  if (event.data === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});

// 页面端代码
let refreshing = false;
navigator.serviceWorker.addEventListener('controllerchange', () => {
  if (!refreshing) {
    refreshing = true;
    window.location.reload();
  }
});

// 检测新 SW 等待中
const registration = await navigator.serviceWorker.getRegistration();
registration.addEventListener('updatefound', () => {
  const newWorker = registration.installing;
  newWorker.addEventListener('statechange', () => {
    if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
      // 新 SW 已安装但等待激活 → 提示用户
      showUpdateBanner();
    }
  });
});
```

---

## 二、五种规范缓存策略

### 策略总览

```
┌─────────────────────────────────────────────────────────────────┐
│  策略                    适用资源               用户体验          │
├─────────────────────────────────────────────────────────────────┤
│  Cache First          hashed 静态资源 (JS/CSS/字体)   最快      │
│  Network First        HTML 文档 / API 响应          最新      │
│  Stale-While-Revalid. 图片 / 非关键 API             平衡      │
│  Cache Only           precache 确定不变的资源         离线      │
│  Network Only         实时数据（交易/验证码）         最可靠    │
└─────────────────────────────────────────────────────────────────┘
```

### 1. Cache First（缓存优先）

适用：带哈希的不可变资源（`main.abc123.js`、`style.def456.css`、字体文件）

```javascript
// 裸 Service Worker 实现
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // 仅匹配带哈希的静态资源
  if (url.pathname.match(/\.(js|css|woff2|wasm)$/) && url.pathname.match(/[a-f0-9]{8,}/)) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        return cached || fetch(event.request).then((response) => {
          return caches.open('static-v1').then((cache) => {
            cache.put(event.request, response.clone());
            return response;
          });
        });
      })
    );
  }
});

// Workbox 等价写法
import { registerRoute } from 'workbox-routing';
import { CacheFirst } from 'workbox-strategies';

registerRoute(
  ({ request }) => request.destination === 'script' ||
                   request.destination === 'style' ||
                   request.destination === 'font',
  new CacheFirst({
    cacheName: 'static-resources',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
      }),
    ],
  })
);
```

### 2. Network First（网络优先）

适用：HTML 文档、需要实时性的 API 响应

```javascript
// 裸 SW 实现
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // HTML 请求
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          const cloned = response.clone();
          caches.open('html-v1').then((cache) => cache.put(event.request, cloned));
          return response;
        })
        .catch(() => {
          // 网络失败 → 返回缓存的 HTML 或离线页面
          return caches.match(event.request).then((cached) => {
            return cached || caches.match('/offline.html');
          });
        })
    );
  }
});

// Workbox 写法
import { NetworkFirst } from 'workbox-strategies';

registerRoute(
  ({ request }) => request.mode === 'navigate',
  new NetworkFirst({
    cacheName: 'html-pages',
    networkTimeoutSeconds: 3, // 3 秒后回退到缓存
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 分钟
      }),
    ],
  })
);
```

### 3. Stale-While-Revalidate（缓存立即返回 + 后台更新）

适用：图片、非关键 API（用户头像、配置数据）

```javascript
// 裸 SW 实现
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/api/config') ||
      event.request.destination === 'image') {

    event.respondWith(
      caches.open('swr-v1').then((cache) => {
        return cache.match(event.request).then((cached) => {
          const fetchPromise = fetch(event.request).then((networkResponse) => {
            cache.put(event.request, networkResponse.clone());
            return networkResponse;
          }).catch(() => {
            // 网络更新失败，下次请求继续使用旧缓存
          });

          // 立即返回缓存（如果有），同时执行网络更新（不阻塞响应）
          return cached || fetchPromise;
        });
      })
    );
  }
});

// Workbox 写法
import { StaleWhileRevalidate } from 'workbox-strategies';

registerRoute(
  ({ request }) => request.destination === 'image',
  new StaleWhileRevalidate({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 200,
        maxAgeSeconds: 7 * 24 * 60 * 60, // 7 days
      }),
    ],
  })
);
```

### 4. Cache Only（仅缓存）

适用：通过 `install` 事件预缓存的不可变资源（App Shell）

```javascript
// 用于 precache 的资源，网络不参与
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  // 仅对明确预缓存的路径使用
  if (PRECACHED_PATHS.has(url.pathname)) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        return cached || new Response('Offline', { status: 503 });
      })
    );
  }
});
```

### 5. Network Only（仅网络）

适用：实时交易、支付确认、验证码、WebSocket 握手

```javascript
// 对特定 API 路径完全跳过 SW 缓存
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/api/payment') ||
      url.pathname.startsWith('/api/verification')) {
    // 直接透传，不匹配也不缓存
    event.respondWith(fetch(event.request));
  }
});

// Workbox: 不注册 route 即可让请求直通网络
// 或者显式设置
import { NetworkOnly } from 'workbox-strategies';
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/payment'),
  new NetworkOnly()
);
```

### Workbox ExpirationPlugin 详细配置

```javascript
import { ExpirationPlugin } from 'workbox-expiration';

// 基于条数的过期
new ExpirationPlugin({
  maxEntries: 100,        // 最多缓存 100 条
  maxAgeSeconds: 86400,   // 单条最大 24 小时
  purgeOnQuotaError: true, // 存储空间不足时自动清理
});

// 基于时间的过期（适合图片/视频缩略图）
new ExpirationPlugin({
  maxAgeSeconds: 7 * 24 * 60 * 60, // 7 天
  matchOptions: { ignoreVary: true }, // 忽略 Vary 头部
});
```

---

## 三、离线架构深度设计

### App Shell 模式

```
┌──────────────────────────────────────────────────────────────┐
│  App Shell (预缓存)                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  <header>  导航栏 / Logo / 用户头像                       │ │
│  │  <main>    内容区域（从 IndexedDB/API 动态填充）          │ │
│  │  <footer>  页脚 / 版权                                    │ │
│  │  <spinner> 骨架加载状态                                   │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                              │
│  App Shell 即时渲染 (最快 100ms)                              │
│      ↓                                                       │
│  动态内容异步填充 (API → IndexedDB → UI)                     │
└──────────────────────────────────────────────────────────────┘
```

```javascript
// sw.js — precache app shell
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('app-shell-v3').then((cache) => {
      return cache.addAll([
        '/',
        '/static/css/shell.css',
        '/static/js/shell.js',
        '/static/icons/logo.svg',
        // 不缓存数据，只缓存骨架 UI
      ]);
    })
  );
});

// 页面端 — shell 渲染后异步填充内容
// main.js
async function initApp() {
  // 1. 渲染 Shell（即时，从缓存）
  renderShell();

  // 2. 尝试从 IDB 读取缓存数据（毫秒级）
  const cachedData = await idb.get('feed', 'latest');
  if (cachedData) {
    renderContent(cachedData);
  }

  // 3. 网络请求最新数据
  try {
    const freshData = await fetch('/api/feed');
    await idb.put('feed', freshData, 'latest');
    renderContent(freshData);
  } catch {
    // 网络失败 → 继续使用缓存数据
    console.log('[App] Offline — using cached data');
  }
}
```

### IndexedDB 离线数据层

```javascript
// 使用 idb-keyval（轻量）或 idb（完整事务支持）
import { openDB } from 'idb';

const db = await openDB('AppDB', 1, {
  upgrade(db) {
    // 文章存储（按 url 索引）
    const articles = db.createObjectStore('articles', { keyPath: 'url' });
    articles.createIndex('by-date', 'publishedAt');
    articles.createIndex('by-tag', 'tags', { multiEntry: true });

    // 用户草稿
    db.createObjectStore('drafts', { keyPath: 'id', autoIncrement: true });

    // 离线队列（待同步的操作）
    const queue = db.createObjectStore('offline-queue', {
      keyPath: 'id',
      autoIncrement: true,
    });
    queue.createIndex('by-timestamp', 'timestamp');
  },
});

// 离线写入 → 后台同步
async function saveArticleOffline(article) {
  await db.add('offline-queue', {
    type: 'CREATE_ARTICLE',
    payload: article,
    timestamp: Date.now(),
  });
  // 如果支持 Background Sync，注册
  if ('serviceWorker' in navigator && 'SyncManager' in window) {
    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register('sync-articles');
  }
}
```

### Background Sync（后台同步）

```javascript
// sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-articles') {
    event.waitUntil(syncOfflineArticles());
  }
});

async function syncOfflineArticles() {
  // 从 IDB 中取出所有待同步操作
  const db = await openDB('AppDB', 1);
  const queue = await db.getAll('offline-queue');
  const sorted = queue.sort((a, b) => a.timestamp - b.timestamp);

  for (const item of sorted) {
    try {
      await fetch('/api/articles', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(item.payload),
      });
      await db.delete('offline-queue', item.id);
    } catch (err) {
      // 单条失败不影响后续
      console.error('[Sync] Failed:', item.id, err);
    }
  }
}
```

### Periodic Background Sync（定期后台同步）

```javascript
// 仅适用于 PWA 安装后，且浏览器按系统条件调度
// Chrome 限制：最低间隔以分钟计，且取决于站点参与度评分
if ('periodicSync' in registration) {
  const status = await navigator.permissions.query({
    name: 'periodic-background-sync',
  });
  if (status.state === 'granted') {
    await registration.periodicSync.register('content-sync', {
      minInterval: 60 * 60 * 1000, // 最小 1 小时
    });
  }
}

// sw.js
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'content-sync') {
    event.waitUntil(fetchAndCacheLatestContent());
  }
});
```

### Push Notification（VAPID）

```javascript
// 订阅
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array('BEl62...'), // VAPID 公钥
  });
  // 发送 subscription 对象到后端
  await fetch('/api/push/subscribe', {
    method: 'POST',
    body: JSON.stringify(subscription),
  });
}

// sw.js — 接收 push 事件
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};
  const { title, body, icon, badge, url } = data;

  event.waitUntil(
    self.registration.showNotification(title, {
      body,
      icon: icon || '/icon-192.png',
      badge: badge || '/badge-72.png',
      data: { url }, // 点击后的跳转 URL
      actions: data.actions || [],
      vibrate: [200, 100, 200],
      tag: data.tag, // 同一 tag 会折叠旧通知
    })
  );
});

// sw.js — 点击通知
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  const url = event.notification.data?.url || '/';
  event.waitUntil(
    clients.openWindow(url)
  );
});
```

---

## 四、PWA 架构决策

### 决策树：何时需要 PWA

```
                    开始: PWA 决策
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
      用户需要离线访问?           需要推送通知?
              │                       │
          Yes │                   Yes │
              ▼                       ▼
           PWA ✓                   PWA ✓
              │
              ▼
      是否需要安装到主屏幕?
              │
          Yes ├─ 市场/品牌需求 ─→ PWA (standalone/fullscreen)
              │
           No └─ 不需要安装 ─→ 仅使用 SW 缓存策略（无需 manifest）
              │
              ▼
      是否需要硬件 API 访问?
      (Camera/Bluetooth/NFC/Geolocation)
              │
          Yes ──→ 考虑 Native（PWA 的硬件 API 支持仍有限）
           No ──→ PWA 适合
```

### PWA vs Native App 差异

| 维度 | PWA | Native App |
|------|-----|------------|
| **安装** | 浏览器内一键、无需审核 | App Store / 各厂商应用商店、审核周期 |
| **更新** | 自动静默更新（SW 生命周期） | 用户手动更新、允许自动更新 |
| **存储上限** | ~60% 磁盘可用（每个 Origin） | 系统不限制（但用户可清除） |
| **后台同步** | Periodic Sync（不可靠、由浏览器调度） | WorkManager (Android) / BGTaskScheduler (iOS) 可靠 |
| **硬件访问** | Web API（受限）：Camera/Geolocation/Bluetooth(Web) | 完整硬件 API: NFC/SPP/BLE central/HealthKit |
| **推送通知** | ✅ VAPID（但 iOS 限制 strict） | ✅ 完整支持（FCM/APNs） |
| **离线能力** | SW + IndexedDB（丰富） | 原生 SQLite + 文件系统 |
| **用户体验** | 浏览器约束（地址栏/安全区域） | 完全系统级集成（小组件/Siri/快捷指令） |
| **开发成本** | 1 套代码（PWA），需适配不同浏览器 SW 实现 | 2 套代码（iOS + Android） |
| **分发渠道** | 仅 HTTPS、可通过自定义安装提示 | App Store / 第三方市场 |

### 何时不适合 PWA

```
1. 强依赖硬件访问
   - 蓝牙 SPP（串口）通信
   - NFC 读卡（非 NDEF 格式）
   - TouchID/FaceID 生物认证（WebAuthn 可部分替代但体验差距大）
   - ARKit/ARCore 级别的 AR 体验

2. 需要系统级集成
   - 系统分享目标（接收其他 App 发来的内容）
   - 系统键盘扩展 / CallKit 级别功能
   - 系统后台守护进程（常驻后台）

3. iOS 的 PWA 限制（截至 iOS 18）
   - 无 Web Push（iOS 16.4+ 已支持但覆盖率取决于用户版本）
   - 无 Background Sync / Periodic Sync
   - 存储上限更激进（使用超过 500MB 会被清理）
   - 必须 HTTPS

4. 首屏性能极致要求
   - SW 首次安装无缓存（首次访问无加速）
   - App Shell 模式增加架构复杂度
```

### 关键决策 Checklist

```
□ 是否需要离线核心功能? → 需要 → 部署 SW + App Shell
□ 目标用户中 iOS 占比 > 40%? → 是 → PWA 功能受限，考虑 Capacitor 打包
□ 是否需要频繁的后台操作? → 是 → Native
□ 团队是否有 SW 调试经验? → 否 → 从 Workbox 开始，渐进采用
□ 是否已有 Web 应用? → 是 → PWA 是渐进增强的最优路径
□ 需要从零构建? → 评估 TWA (Trusted Web Activity) 作为中间方案
```

---

## 五、调试与监控

### Chrome DevTools Service Worker 面板

```bash
# 1. 打开 Application > Service Workers
#    - 查看 SW 状态、版本、注册时间
#    - 手动 Update / Unregister / Skip Waiting
#    - "Bypass for network" 复选框用于临时跳过缓存

# 2. Application > Cache Storage
#    - 查看所有缓存数据库及其内容
#    - 预览缓存的响应（Response 内容可见）
#    - 手动删除

# 3. Application > IndexedDB
#    - 查看数据库 → Object Store → 数据记录
#    - 手动添加/编辑/删除记录（调试离线场景）

# 4. Network 面板
#    - Size 列: 显示 "(ServiceWorker)" 表示来自缓存
#    - "from ServiceWorker" 在 Headers 中可见
```

### 常见错误与排查

```javascript
// 错误 1: SW 注册失败
// 原因: 非 HTTPS（localhost 除外）、作用域错误、文件 404
if (!navigator.serviceWorker) {
  console.error('Service Worker not supported');
  return;
}
try {
  await navigator.serviceWorker.register('/sw.js', {
    scope: '/', // 必须是 sw.js 所在目录或子目录
  });
} catch (err) {
  console.error('SW registration failed:', err);
  // Chrome 通常不返回详细错误，检查:
  // - sw.js 是否为 404
  // - sw.js 的 MIME type 是否为 text/javascript
  // - SSL 证书是否有效
}

// 错误 2: Cache Storage 配额耗尽
// 检查: navigator.storage.estimate()
const { usage, quota } = await navigator.storage.estimate();
console.log(`Used: ${(usage / 1024 / 1024).toFixed(2)}MB / ${(quota / 1024 / 1024).toFixed(2)}MB`);
// 合理使用率: < 50%，超过 70% 应考虑清理策略

// 错误 3: 带哈希资源缓存失效
// 使用了 Cache First 但因 URL 变化缓存未命中
// 方案: 在 install 事件中更新 PRECACHE_URLS 列表
```

### 监控指标

```javascript
// 用 Performance API 监控 SW 命中率
let swHits = 0;
let totalRequests = 0;

// 在 fetch handler 中
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      totalRequests++;
      if (cached) swHits++;

      // 定期上报（通过 Message Channel 发回页面）
      if (totalRequests % 100 === 0) {
        self.clients.matchAll().then((clients) => {
          clients.forEach((client) => {
            client.postMessage({
              type: 'SW_METRICS',
              hitRate: swHits / totalRequests,
              totals: totalRequests,
            });
          });
        });
      }

      return cached || fetch(event.request);
    })
  );
});
```

---

## 相关参考

- Workbox 官方文档: https://developer.chrome.com/docs/workbox
- Jake Archibald's Offline Cookbook: https://web.dev/offline-cookbook/
- Web Vitals 与 PWA: 参考 `performance-engineering.md`
- PWA 相关陷阱: `pitfalls/pit-070.md` (SW 缓存过期), `pit-071.md` (Workbox 策略选错)
