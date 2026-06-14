# Edge & Serverless 架构

> 本文档整理 Edge Computing / Serverless 在前端架构中的完整知识体系，
> 覆盖 V8 Isolate 运行时对比、Edge 部署模式、边缘数据存储及平台选型决策。
> 基于 Cloudflare Workers、Vercel Edge Functions、Deno Deploy、AWS Lambda@Edge 官方文档及一线生产经验。

---

## 一、Edge 运行时对比

### 运行时架构全景

```
┌─────────────────────────────────────────────────────────────────────┐
│  类型 1: V8 Isolates（Cloudflare Workers / Vercel Edge / Deno Deploy）│
├─────────────────────────────────────────────────────────────────────┤
│  启动过程:                                                          │
│    用户请求到达 → 选择/创建 Isolate → 注入代码 → 执行 → 响应        │
│    代码已预编译为字节码在 CDN 上                                     │
│                                                                     │
│  特点:                                                              │
│    - Cold start: < 5ms (V8 Isolate 创建极快)                        │
│    - 内存上限: 128MB (Workers Free) / 512MB (Paid/Deno)             │
│    - CPU 时间: 10ms (Free) ~ 50ms (Paid) ~ 30s (Deno Deploy)       │
│    - 无文件系统、无 TCP 原始套接字（需 connect() API）              │
│    - 无 Node.js API（require/f文件系统/child_process 均不可用）     │
├─────────────────────────────────────────────────────────────────────┤
│  类型 2: Container (AWS Lambda / GCP Cloud Functions)               │
├─────────────────────────────────────────────────────────────────────┤
│    - Cold start: 50ms ~ 500ms（取决于运行时语言/包大小）            │
│    - 内存: 128MB ~ 10GB                                             │
│    - 最长执行: 15 分钟                                              │
│    - 完整文件系统 (/tmp)                                             │
├─────────────────────────────────────────────────────────────────────┤
│  类型 3: WASM Runtime (Fastly Compute@Edge)                         │
├─────────────────────────────────────────────────────────────────────┤
│    - 代码编译为 WASM，在 WebAssembly 沙箱中运行                     │
│    - Cold start: < 1ms（WASM 实例化极快）                           │
│    - 支持任意编译为 WASM 的语言（Rust/C/Go/Zig/AssemblyScript）     │
│    - 内存上限: 128MB                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 三种运行时详细对比

| 维度 | V8 Isolates (Workers) | Container (Lambda) | WASM (Fastly) |
|------|----------------------|--------------------|---------------|
| **冷启动** | < 5ms | 50-500ms | < 1ms |
| **热启动** | 0ms（Isolate 复用） | 0ms（Container 复用） | 0ms |
| **并发模型** | 每个请求独立 Isolate（隔离性好） | 共享 Container（请求间可能共享全局状态） | 每个请求独立实例 |
| **CPU 架构** | x86_64 (V8 JIT) | x86_64 / ARM64 | x86_64 (WASM 解释/AOT) |
| **内存隔离** | V8 堆隔离（极度安全） | Linux Namespace | WASM 线性内存 |
| **全局状态** | 无跨请求共享（除了 KV/Durable Objects） | 同一 Container 内共享 | 无 |
| **最大执行时间** | 30s (Workers Paid), 30s (Deno Deploy) | 15 分钟 | 60s |
| **初始 Bundle 限制** | 1MB (Workers Free) ~ 10MB (Paid) | 50MB (zipped) ~ 250MB (unzipped) | 100MB |
| **网络 API** | fetch() 原生 (WHATWG) | Node.js http 模块 | fetch() 原生 |
| **Web API 兼容** | Web Crypto / Encoding / Streams | 需 polyfill | Web Crypto / Encoding / Streams |
| **调试体验** | wrangler dev (本地 miniflare) | SAM local / Lambda RIE | Fastly CLI (Viceroy) |

### CPU 时间限制细节

```
Cloudflare Workers:
  Free: 10ms CPU time per request
  Paid: 30s CPU time per request (wall time: 无限制)
  ⚠️ CPU time ≠ wall time: fetch() 等待时间不计入 CPU time

Vercel Edge Functions:
  Free/Hobby: 10ms ~ 100ms (varies)
  Pro: up to 30s
  ⚠️ 对流式响应友好（streaming 不计入执行时间）

Deno Deploy:
  Free: 10ms CPU time
  Pro: 50ms CPU time
  Enterprise: 30s
  ⚠️ 对每个请求的 CPU 时间严格限制

AWS Lambda@Edge:
  5s (Viewer Request/Response triggers)
  30s (Origin Request/Response triggers)
  ⚠️ 内存与 Lambda 相同 (128MB~3GB)
```

### 什么能做什么不能做

```javascript
// ✅ Edge Function 可以做的
export default {
  async fetch(request, env, ctx) {
    // 1. 请求改写/重定向
    const url = new URL(request.url);
    if (url.pathname === '/old-path') {
      return Response.redirect('https://example.com/new-path', 301);
    }

    // 2. A/B 测试路由
    const cookie = request.headers.get('cookie') || '';
    const variant = cookie.includes('variant=b') ? 'b' : 'a';
    if (!cookie.includes('variant')) {
      const chosen = Math.random() < 0.5 ? 'a' : 'b';
      url.pathname = `/experiments/${chosen}${url.pathname}`;
    }

    // 3. 认证/速率限制
    const authHeader = request.headers.get('Authorization');
    if (!authHeader) return new Response('Unauthorized', { status: 401 });

    const { success } = await env.RATE_LIMITER.limit({ key: getIP(request) });
    if (!success) return new Response('429 Too Many Requests', { status: 429 });

    // 4. 响应修改（注入 headers、修改 HTML）
    const response = await fetch(request);
    const modified = new HTMLRewriter()
      .on('head', new HeadRewriter())
      .transform(response);
    return modified;

    // 5. 流式处理
    const { readable, writable } = new TransformStream();
    ctx.waitUntil(processStream(readable));
    return new Response(readable);
  }
}

// ❌ Edge Function 不能做的
// - 长连接 (WebSocket 服务端有限制，需 Durable Objects)
// - 大量数据库操作（应在 Origin 处理）
// - 文件系统操作（仅 KV/Blob 存储）
// - GPU 计算 / 视频转码
// - Node.js native modules (bcrypt/ imagemagick/ puppeteer)
```

---

## 二、Edge 部署模式

### 模式 1: Request Routing/Rewriting（请求路由）

```javascript
// Cloudflare Workers — 动态路由
export default {
  async fetch(request) {
    const url = new URL(request.url);

    // 按路径路由
    const routes = {
      '/api': fetchOrigin,
      '/admin': authGuard(fetchOrigin),
      '/blog/*': redirectToNewCMS,
      '/assets/*': fetchCDN,    // 静态资源回源 CDN
      '/_next/*': fetchOrigin,  // Next.js 资源
    };

    const handler = matchRoute(url.pathname, routes);
    return handler ? handler(request) : fetch(request);
  }
};

// 按地理位置路由
function getNearestOrigin(request) {
  const country = request.cf?.country; // CF 免费提供
  if (country === 'CN') return 'https://origin-cn.example.com';
  if (['JP', 'KR', 'SG'].includes(country)) return 'https://origin-asia.example.com';
  return 'https://origin-us.example.com';
}
```

### 模式 2: SSR at Edge

```javascript
// Next.js Edge Runtime
// next.config.js
const nextConfig = {
  experimental: {
    runtime: 'experimental-edge', // 或 App Router 中按文件指定
  },
};

// app/page.tsx
export const runtime = 'edge'; // ⭐ 指定 Edge 运行时

export default async function Page() {
  // ✅ Edge 中可用的 Next.js 特性
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 },
  }).then(r => r.json());

  return <div>{data.title}</div>;
}

// SvelteKit — adapter-cloudflare
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';
export default { kit: { adapter: adapter() } };
```

### 模式 3: API Gateway at Edge

```javascript
// 边缘层的统一 API Gateway
export default {
  async fetch(request, env) {
    // 1. 认证 — 极快（JWT verify 在 V8 中 < 1ms）
    const token = request.headers.get('Authorization')?.replace('Bearer ', '');
    if (!token) return json({ error: 'No token' }, 401);

    try {
      const payload = await verifyJWT(token, env.JWT_SECRET);
      // 2. 速率限制
      const { success } = await env.RATE_LIMITER.limit({
        key: `${payload.sub}:${new URL(request.url).pathname}`
      });
      if (!success) return json({ error: 'Rate limited' }, 429);

      // 3. 日志
      ctx.waitUntil(env.ANALYTICS.writeDataPoint({
        blobs: [payload.sub, request.method, new URL(request.url).pathname],
      }));

      // 4. 转发到 Origin
      return fetch(request);
    } catch {
      return json({ error: 'Invalid token' }, 401);
    }
  }
};
```

### 模式 4: A/B 测试 at Edge

```javascript
// 边缘层做流量分割 — 用户无感知
export default {
  async fetch(request, env) {
    const cookie = parseCookies(request.headers.get('cookie'));
    let variant = cookie['ab_experiment'];

    if (!variant) {
      // 基于用户 ID 的确定性哈希（同一用户始终在同一组）
      const userId = cookie['user_id'] || crypto.randomUUID();
      const hash = await sha256(userId + env.EXPERIMENT_SALT);
      variant = parseInt(hash.slice(0, 8), 16) % 100 < 50 ? 'control' : 'variant';
    }

    const upstreamUrl = variant === 'variant'
      ? 'https://variant-backend.example.com'
      : 'https://control-backend.example.com';

    const response = await fetch(new Request(upstreamUrl + request.url, request));

    // Set cookie so user stays in same bucket
    response.headers.set('Set-Cookie', `ab_experiment=${variant}; Path=/; Max-Age=86400`);
    return response;
  }
};
```

---

## 三、Edge 数据存储

### 存储产品矩阵

```
┌────────────────────────────────────────────────────────────────────┐
│  类型                 产品              一致性       典型延迟       │
├────────────────────────────────────────────────────────────────────┤
│  最终一致 KV          CF Workers KV    最终一致     <1ms(热)/<100ms│
│                       Deno KV          强一致       <1ms          │
│                       Vercel KV        最终一致     <10ms         │
├────────────────────────────────────────────────────────────────────┤
│  强一致对象           CF Durable Obj.  强一致(单实例) <1ms        │
├────────────────────────────────────────────────────────────────────┤
│  SQLite 边缘          CF D1            强一致       <10ms         │
│                       Turso (libSQL)   强一致(主)   <10ms(边缘副本)│
├────────────────────────────────────────────────────────────────────┤
│  对象存储             CF R2            S3 兼容      取决于对象大小  │
│                       Vercel Blob      最强一致     <50ms         │
└────────────────────────────────────────────────────────────────────┘
```

### KV (最终一致) vs Durable Objects (强一致)

```javascript
// KV — 最终一致性场景
// 适合: 配置、功能开关、静态数据缓存
const value = await env.CONFIG.get('featureFlags', 'json');
// ⚠️ 写入后可能 60s 内读不到最新数据（CF Workers KV 保证最终一致）
// 策略: 本地缓存 + 短 TTL
await env.CONFIG.put('key', JSON.stringify(data), {
  expirationTtl: 3600, // 1 hour TTL
});

// Durable Objects — 强一致性场景
// 适合: 计数器、聊天室、实时协作、购物车
export class Counter {
  constructor(state, env) { this.state = state; }
  async fetch(request) {
    let value = (await this.state.storage.get('value')) || 0;
    value += 1;
    await this.state.storage.put('value', value);
    return new Response(value.toString());
  }
}
// 每个 DO 实例是单线程、顺序执行 → 天然无竞态

// D1 — SQLite at Edge
// 适合: 需要 SQL 查询的结构化数据、轻量关系数据
const { results } = await env.DB.prepare(
  'SELECT * FROM users WHERE email = ?'
).bind(email).all();
```

### 何时用 Edge DB vs Origin DB

```
                    开始: 数据存储选型
                          │
            ┌─────────────┼─────────────┐
            ▼              ▼              ▼
      全局低延迟访问?  强事务性/复杂查询?  大量数据?
            │              │              │
        Yes │          Yes │          Yes │
            ▼              ▼              ▼
       Edge DB          Origin DB       Origin DB
       (D1/Turso/KV)   (PostgreSQL)    (PostgreSQL
            │              │              + Redis 缓存)
            │              │
            └──────┬───────┘
                   │
            是否需要实时协作?
            Yes → Durable Objects (单线程保证顺序)
             No → D1 (SQLite) + KV 缓存层
```

```javascript
// 混合架构: Edge + Origin
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // 快速读取 → Edge (D1 读副本或 KV)
    if (request.method === 'GET' && url.pathname.startsWith('/api/posts/')) {
      const cached = await env.KV.get(url.pathname, 'json');
      if (cached) return json(cached);
      // Cache miss → fallback to origin
    }

    // 写入操作 → 始终 Origin（保证一致性）
    if (request.method === 'POST' || request.method === 'PUT') {
      return fetch('https://origin-api.example.com' + url.pathname, {
        method: request.method,
        headers: request.headers,
        body: request.body,
      });
    }

    return fetch(request);
  }
};
```

---

## 四、平台选型决策

### Cloudflare Workers vs Vercel Edge vs Deno Deploy

| 维度 | Cloudflare Workers | Vercel Edge | Deno Deploy |
|------|-------------------|-------------|-------------|
| **网络覆盖** | 全球 310+ 城市（最大） | 全球 ~100 个节点 | 全球 35 个区域 |
| **冷启动** | < 5ms | < 50ms | < 5ms |
| **免费额度** | 10 万次/天, 10ms CPU | 100 万次/月 (hobby) | 100 万次/月 |
| **运行时** | V8 Isolate + 自定义 API | V8 Isolate + Web API | V8 Isolate + Deno API |
| **框架集成** | wrangler + all frameworks | Next.js 原生 | Fresh / 通用 |
| **存储** | KV, D1, R2, Durable Obj, Queues | KV, Blob, Postgres | Deno KV, Queues |
| **配套生态** | Workers for Platforms, Pages, Stream, Images, AI | Analytics, Speed Insights, Cron Jobs | Subhosting, Playgrounds |
| **npm 支持** | Node.js compat (大部分) | Edge-compatible npm | 通过 npm: 前缀直接使用 |
| **WebSocket** | 需 Durable Objects | 不支持 (建议专用服务) | 原生 WebSocket |
| **最佳场景** | 全球大规模、重存储、实时应用 | Next.js 项目、全栈部署 | Deno 生态、WebSocket 应用 |

### 决策树

```
                    开始: Edge 平台选型
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
       使用 Next.js?              需要全球最大覆盖?
              │                       │
          Yes │                   Yes │
              ▼                       ▼
         Vercel                  Cloudflare Workers
              │                       │
              ├─ 不需要复杂存储        ├─ 需要 SQL? → D1
              │  → Vercel Functions   ├─ 需要实时同步? → DO
              │                       └─ 需要存储文件? → R2
              ├─ 需要边缘渲染
              │  → Vercel Edge
              │
              └─ 需要 Postgres
                 → Vercel Postgres + ISR

              ┌───────────┴───────────┐
              ▼                       ▼
       需要 WebSocket 原生支持?   需要 Rust/Go 高性能?
              │                       │
          Yes │                   Yes │
              ▼                       ▼
       Deno Deploy                Fastly Compute@Edge
       (Fresh app)                (WASM runtime)
```

### 成本估算模型

```javascript
// 请求量级 vs 成本 (2025 年参考, 以美元计, USD)

// 场景 A: 1M req/month, 轻计算
// CF Workers:   $0 (Free tier)
// Vercel Edge:  $0 (Hobby)
// Deno Deploy:  $0 (Free tier)
// AWS Lambda@E: $0.60 + CloudFront

// 场景 B: 100M req/month, 中等计算 (10ms avg)
// CF Workers:   $0.30/Million requests + $0.01/Million CPU-ms
//               ≈ $30 + $100 = $130/month
// Vercel Edge:  Pro $100/month (1TB bandwidth incl.)
//               + $60/100GB extra bandwidth
// Deno Deploy:  $2/Million requests ≈ $200/month
// AWS Lambda@E: $0.60/Million + $0.00005/128MB-ms
//               = $60 + compute cost

// 场景 C: 高计算需求 (> 50ms CPU per request)
// → 不适合 Edge，应使用 Origin + 传统 Server
```

---

## 五、关键演练

### 调试 Edge Function

```bash
# Cloudflare Workers
npx wrangler dev                    # 本地模拟 (miniflare)
npx wrangler tail                   # 实时日志流
npx wrangler deploy --dry-run       # 校验部署

# Vercel Edge
vercel dev                          # 本地开发
# 日志: Vercel Dashboard → Functions → 实时日志
# ⚠️ Edge 日志延迟较大 (5-30s)

# Deno Deploy
deployctl run --project=my-project  # 本地运行
deployctl logs --project=my-project # 日志流
```

### 常见故障模式

```javascript
// 故障 1: CPU Timeout — 超过限制被强制终止
// 症状: 520/502 错误，response body 不完整
// 诊断: wrangler tail 中查找 "CPU time exceeded"
// 修复:
// - 减少同步操作
// - 将重计算移到 Origin
// - 使用 ctx.waitUntil() 延迟非关键工作

// 故障 2: 内存溢出
// 症状: 500 错误，"out of memory" 日志
// Workers 默认 128MB
// 诊断: 检查是否有大对象未释放、无限增长的缓存
// 修复: 使用 WeakRef、限制缓存大小、分批处理

// 故障 3: KV 读到过期数据
// 症状: 写入后立即读取返回旧值
// 根因: CF KV 最终一致性，全局传播延迟可达 60s
// 修复: 关键状态使用 Durable Objects 或 D1
// 非关键数据: 接受陈旧读取或缓存层二次校验

// 故障 4: 依赖不可用的 Node.js API
// 症状: "require is not defined" / "process is not defined"
// Edge Runtime 非完整 Node.js
// 修复:
// - 使用 Web Crypto 替代 crypto
// - 使用 fetch() 替代 node-fetch/axios
// - 兼容 Node.js API 列表: https://developers.cloudflare.com/workers/runtime-apis/nodejs/
```

---

## 相关参考

- Cloudflare Workers 文档: https://developers.cloudflare.com/workers/
- Vercel Edge Functions: https://vercel.com/docs/functions/edge-functions
- Deno Deploy: https://deno.com/deploy/docs
- Fastly Compute@Edge: https://docs.fastly.com/products/compute
- 相关陷阱: `pitfalls/pit-074.md` (Edge CPU 超时), `pit-075.md` (KV 最终一致性)
