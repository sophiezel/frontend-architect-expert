# 前端安全工程指南

> 基于 OWASP Top 10、CSP Level 3、Trusted Types 等标准，面向现代前端架构的纵深防御体系。
> 目标读者：需要建立前端安全体系的前端架构师与安全工程师。

---

## 一、XSS 防御体系

### 1.1 XSS 分类

| 类型 | 攻击向量 | 示例场景 |
|------|---------|---------|
| **Reflected XSS** | 恶意输入通过 URL/表单参数直接回显 | `https://site.com/search?q=<script>alert(1)</script>` |
| **Stored XSS** | 恶意代码持久化存储在数据库/缓存中 | 用户头像字段存储 `<img onerror=stealCookies() src=x>` |
| **DOM-based XSS** | JavaScript 处理用户输入不安全（不经过服务端） | `element.innerHTML = location.hash.slice(1)` |
| **mXSS (Mutation XSS)** | 浏览器解析器在 DOM 重建时改变标签结构 | `innerHTML` 赋值为看似安全的 HTML，浏览器解析后变异 |

### 1.2 框架默认转义与逃逸口

```typescript
// ═══════ React ═══════
// ✅ 自动转义：JSX 花括号默认安全
function Safe({ userInput }: { userInput: string }) {
  return <div>{userInput}</div>; // <script> → &lt;script&gt;
}

// ❌ 逃逸口：dangerouslySetInnerHTML
function Dangerous({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
  // 如果 html = "<img src=x onerror=alert(1)>" → XSS
}

// ❌ 其他逃逸口
function OtherEscapes() {
  return (
    <a href={userInput}>click</a>   // javascript:alert(1) → XSS
    <iframe src={userInput} />     // data:text/html,... → XSS
  );
}

// ✅ 安全实践：净化后使用
import DOMPurify from 'dompurify';
function WithSanitizer({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href'],
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ═══════ Vue ═══════
// ✅ 自动转义：{{ }} 插值默认安全
// ❌ 逃逸口：v-html
// <div v-html="userHtml"></div>  ← XSS 向量

// ═══════ Svelte ═══════
// ✅ 自动转义：{ } 插值默认安全
// ❌ 逃逸口：{@html rawHtml}
```

### 1.3 Sanitizer API（2024 Baseline）

```typescript
// 浏览器原生 Sanitizer API (Chrome 105+, Firefox 83+)
// 无需引入 DOMPurify 等第三方库

// 基础使用
const sanitizer = new Sanitizer();
const clean = element.setHTML(userInput, { sanitizer });
// 等价于 element.innerHTML = x，但经过安全净化

// 自定义配置
const strictSanitizer = new Sanitizer({
  allowElements: ['b', 'i', 'em', 'strong', 'a', 'p'],
  allowAttributes: {
    'href': ['a'],
    'target': ['a'],
  },
  blockElements: ['script', 'style', 'input', 'form'],
  dropElements: ['script', 'style'],
});

// 编程式净化
const cleanDiv = document.createElement('div');
cleanDiv.setHTML(untrustedHTML, { sanitizer: strictSanitizer });

// 字符串净化
const cleanString = sanitizer.sanitizeFor('div', untrustedHTML).innerHTML;

// ⚠️ 兼容性回退
async function sanitize(html: string): Promise<string> {
  if ('Sanitizer' in globalThis) {
    const s = new Sanitizer({ dropElements: ['script', 'style'] });
    const el = document.createElement('div');
    el.setHTML(html, { sanitizer: s });
    return el.innerHTML;
  }
  // Fallback to DOMPurify
  const { default: DOMPurify } = await import('dompurify');
  return DOMPurify.sanitize(html);
}
```

### 1.4 CSP v3 with strict-dynamic + nonce

```nginx
# ═══════ Nginx 配置 ═══════
# strict-dynamic CSP — 现代 XSS 防御的核心

# 每个请求生成唯一 nonce（服务端必须动态生成，不能写死）
# 后端示例（Express）:
# res.locals.nonce = crypto.randomBytes(16).toString('base64');
# 模板: <script nonce="{% nonce %}">

add_header Content-Security-Policy "
  script-src
    'strict-dynamic'
    'nonce-r@nd0m-n0nc3-:base64:'
    'unsafe-inline'
    https:;
  object-src 'none';
  base-uri 'none';
  frame-ancestors 'none';
  require-trusted-types-for 'script';
";

# ⚠️ 关键注意事项：
# 1. nonce 必须每次请求重新生成，不可复用
# 2. strict-dynamic 要求页面所有脚本都带有效 nonce
# 3. 'unsafe-inline' 在 strict-dynamic 下被忽略（浏览器会忽略）
# 4. 被 nonce 脚本动态创建的 <script> 自动继承信任
# 5. 必须配合 Trusted Types 才能真正阻止 DOM XSS
```

### 1.5 Trusted Types（可信类型）

```typescript
// ═══════ 浏览器强制 DOM XSS 预防 ═══════
// 在 CSP 中启用: require-trusted-types-for 'script'

// ❌ 未启用 Trusted Types 时这些代码正常工作
document.getElementById('app')!.innerHTML = userInput; // XSS!

// ✅ 启用后，innerHTML 只接受 TrustedHTML 类型
// 需要先创建 Trusted Types 策略

// 注册默认策略（处理未匹配的赋值）
if (window.trustedTypes) {
  trustedTypes.createPolicy('default', {
    createHTML: (input: string) => {
      // 返回空字符串作为最安全的默认行为
      // 生产环境绝不应该直接返回 input
      return '';
    },
    createScript: (input: string) => input,
    createScriptURL: (input: string) => {
      // 只允许同源和已知 CDN
      const url = new URL(input, document.baseURI);
      const allowed = [location.origin, 'https://cdn.example.com'];
      if (allowed.includes(url.origin)) return input;
      throw new TypeError('Blocked script URL');
    },
  });
}

// 注册专用策略
const dompurifyPolicy = trustedTypes?.createPolicy('dompurify', {
  createHTML: (input: string) => DOMPurify.sanitize(input),
});

// 使用 TrustedHTML
const trusted = dompurifyPolicy!.createHTML(userInput);
document.getElementById('app')!.innerHTML = trusted as unknown as string;

// ⚠️ 危险：绕过的 Trusted Types
// 不要这样做：
// 1. 创建返回原始输入的 createHTML 策略
// 2. 在 default 策略中直接返回 input
// 3. 使用 webpack/rspack 的 string-replace-loader 删除 trustedTypes 调用

// 浏览器兼容性检测
const supportsTrustedTypes = typeof window !== 'undefined' && window.trustedTypes !== undefined;
```

---

## 二、CSP (Content Security Policy) 架构

### 2.1 nonce-based strict CSP vs allowlist CSP

```nginx
# ═══════ 方案A：allowlist CSP（传统，不推荐）═══════
# 问题：allowlist 是"允许列表去防御"的误导
# - 在 JSONP 端点、CDN 重定向等场景下被绕过
# - 维护成本高，有遗漏风险
add_header Content-Security-Policy "
  script-src https://cdn.example.com https://apis.google.com 'self';
";

# ═══════ 方案B：nonce-based strict CSP（推荐）═══════
# 不关心脚本来源，只关心是否持有有效 nonce
add_header Content-Security-Policy "
  script-src 'strict-dynamic' 'nonce-{RANDOM}' https:;
  object-src 'none';
  base-uri 'none';
";

# strict-dynamic 语义：
# 1. 带有效 nonce 的 <script> 允许执行
# 2. 该脚本通过 createElement('script') 动态创建的脚本也允许执行
# 3. 完全忽略 allowlist（https: 只作为 fallback）
```

### 2.2 CSP report-only → enforce 渐进部署

```nginx
# 阶段1：仅报告，不阻断（观察1-4周）
add_header Content-Security-Policy-Report-Only "
  script-src 'strict-dynamic' 'nonce-{NONCE}';
  object-src 'none';
  base-uri 'none';
  report-uri /api/csp-report;
";

# 阶段2：分组灰度发布
# 通过 nginx split_clients 或 CDN 路由实现
split_clients "${remote_addr}" $csp_mode {
  10%  enforce;   # 10% 流量开启强制模式
  *    report_only;
}

# 阶段3：全量开启
add_header Content-Security-Policy "
  script-src 'strict-dynamic' 'nonce-{NONCE}';
  object-src 'none';
  base-uri 'none';
  report-uri /api/csp-report;
";
```

### 2.3 CSP violation reporting endpoint

```typescript
// ═══════ 后端：Express CSP 报告收集端点 ═══════
// POST /api/csp-report
interface CSPReport {
  'csp-report': {
    'document-uri': string;
    'violated-directive': string;
    'blocked-uri': string;
    'source-file': string;
    'line-number': number;
    'column-number': number;
    'script-sample'?: string;  // CSP v3: 被阻止脚本的前 40 字符
    'effective-directive': string;
    'original-policy': string;
  };
}

// 处理报告
app.post('/api/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  const report = req.body['csp-report'];
  console.warn('CSP Violation:', report['violated-directive'], report['blocked-uri']);

  // 生产环境建议：
  // 1. 采样（防止 DDoS）— 相同 violation 每分钟只记录一次
  // 2. 聚合到监控系统（Datadog/Grafana/Sentry）
  // 3. 设置告警阈值（新 violation 类型 5 分钟内出现 > 10 次 → 告警）
  // 4. 过滤已知误报（浏览器插件注入）

  res.status(204).end();
});

// ═══════ 前端：检测 CSP 是否生效 ═══════
// CSP 报告中的 script-sample 对调试至关重要
// 前端调试代码:
if (process.env.NODE_ENV === 'development') {
  document.addEventListener('securitypolicyviolation', (e) => {
    console.group('⚠️ CSP Violation');
    console.log('Directive:', e.violatedDirective);
    console.log('Blocked URI:', e.blockedURI);
    console.log('Source File:', e.sourceFile);
    console.log('Sample:', e.sample);
    console.log('Original Policy:', e.originalPolicy);
    console.groupEnd();
  });
}
```

### 2.4 与第三方脚本的共存策略

```typescript
// ═══════ 问题：第三方脚本（GA, Sentry, Intercom）如何与 strict CSP 共存 ═══════

// 方案1：构建时注入 nonce（推荐）
// 在 HTML 模板中为第三方脚本添加 nonce 属性
// <script nonce="{% NONCE %}" src="https://www.googletagmanager.com/gtag/js"></script>

// 方案2：预定义 Trusted Types 策略
// 在第三方脚本加载前注册策略
trustedTypes?.createPolicy('ga-script', {
  createScriptURL: (url: string) => {
    if (url.startsWith('https://www.googletagmanager.com/')) {
      return url;
    }
    throw new TypeError('Blocked');
  },
});

// 方案3：使用 Partytown 隔离第三方脚本到 Web Worker
// 第三方脚本运行在 worker 中，主线程通过 proxy 通信
// 但 Partytown 需要 eval（与 strict CSP 冲突），需要额外处理

// 方案4：Server-side 加载（GA4 server-side tagging）
// 将第三方标记逻辑搬到服务端，前端只发送轻量事件

// ⚠️ 不可用的方案：
// 1. 放弃 strict-dynamic 退回 allowlist — 安全降级
// 2. 使用 'unsafe-eval' — 彻底破坏 CSP 安全性
// 3. 在 CSP 中允许 data: 或 blob: 作为 script-src
```

---

## 三、供应链安全

### 3.1 npm 依赖安全

```bash
# ═══════ package-lock.json 完整性哈希 ═══════
# npm v7+ 自动记录 integrity 字段（sha512）
# 验证：确保安装的包与 lockfile 一致
npm ci        # 严格按 lockfile 安装（比 npm install 更安全）
npm audit     # 扫描已知 CVE
npm audit fix # 自动修复兼容的漏洞

# ═══════ Snyk 深度扫描 ═══════
npx snyk test          # 本地扫描
npx snyk monitor       # 持续监控（上传到 Snyk 平台）
npx snyk code test     # 静态代码扫描

# ═══════ Socket.dev 行为分析 ═══════
# Socket 分析包的 install scripts、网络请求、文件系统访问等行为
# 不依赖 CVE 数据库，可发现未知恶意包
npx @socketsecurity/cli scan package.json
npx @socketsecurity/cli scan package-lock.json

# ═══════ 依赖策略：lockfile-lint ═══════
{
  "path": "./package-lock.json",
  "type": "npm",
  "validate-https": true,
  "validate-integrity": true,
  "allowed-hosts": ["registry.npmjs.org"],
  "allowed-schemes": ["https:"]
}
# 在 CI 中添加:
# npx lockfile-lint --path package-lock.json --validate-https --validate-integrity
```

### 3.2 SRI (Subresource Integrity)

```html
<!-- ═══════ SRI on CDN scripts ═══════ -->
<!-- ❌ 无 SRI：CDN 被攻击后脚本可被替换 -->
<script src="https://cdn.example.com/jquery.min.js"></script>

<!-- ✅ 带 SRI：浏览器验证文件哈希 -->
<script
  src="https://cdn.example.com/jquery@3.7.1.min.js"
  integrity="sha384-abcdef1234567890abcdef1234567890"
  crossorigin="anonymous"
></script>

<!-- 生成 SRI 哈希 -->
<!-- 方式1：openssl -->
# openssl dgst -sha384 -binary jquery.js | openssl base64 -A

<!-- 方式2：使用 sri-toolbox -->
# npx sri-toolbox generate jquery.js

<!-- 方式3：在线工具 -->
# https://www.srihash.org/

<!-- ⚠️ 注意事项 -->
<!-- 1. crossorigin="anonymous" 必须设置（即使同源也需要） -->
<!-- 2. 使用 sha384，不要用 sha256（碰撞风险） -->
<!-- 3. integrity 不止用于 script，也用于 link[rel=stylesheet] -->
<link
  rel="stylesheet"
  href="https://cdn.example.com/styles.css"
  integrity="sha384-..."
  crossorigin="anonymous"
/>
```

### 3.3 SBOM + SLSA attestation

```bash
# ═══════ SBOM (Software Bill of Materials) ═══════
# 生成项目的物料清单（所有依赖及其依赖的依赖）

# 使用 CycloneDX
npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# 使用 spdx-sbom-generator
npx @spdx/sbom-generator -o sbom.spdx.json

# SLSA (Supply-chain Levels for Software Artifacts)
# 验证构建来源和完整性
# 对大型项目：要求 dependency 的 GitHub Actions 使用 SLSA Level 3+
# https://slsa.dev/

# ═══════ npm provenance ═══════
# npm publish 时生成签名来源证明（SLSA Level 3）
# package.json
{
  "publishConfig": {
    "provenance": true
  }
}
# GitHub Actions 中：
# npm publish --provenance
```

### 3.4 依赖锁定与版本管理

```jsonc
// package.json 版本策略
{
  "dependencies": {
    // ✅ 精确版本：防止小版本意外升级
    "react": "18.2.0",
    "react-dom": "18.2.0",

    // ⚠️ 波浪号：允许 patch 升级（~1.2.3 = >=1.2.3 <1.3.0）
    "lodash": "~4.17.21",

    // ❌ 脱字符：允许 minor 升级（^1.2.3 = >=1.2.3 <2.0.0）
    // 仅在确认依赖遵守 semver 时使用
    // "express": "^4.18.0"
  },
  "overrides": {
    // 强制传递依赖使用安全版本
    "braces": "3.0.3"  // CVE-2024-4068 修复版本
  },
  "scripts": {
    "audit:ci": "npm audit --omit=dev --audit-level=high",
    "depcheck": "npx depcheck"
  }
}
```

---

## 四、认证与授权

### 4.1 JWT 前端存储策略

```typescript
// ═══════ 方案对比 ═══════
// 方案A：httpOnly cookie（推荐）
// 优势：JavaScript 无法访问，免疫 XSS 窃取
// 劣势：受 CSRF 影响（需配合 SameSite + CSRF token）
// Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Strict; Path=/api

// 方案B：内存存储
// 优势：XSS 也无法窃取（仅存在于 JS 闭包）
// 劣势：刷新页面丢失，需刷新 token 机制
class TokenStore {
  private token: string | null = null;
  set(t: string) { this.token = t; }
  get(): string | null { return this.token; }
  clear() { this.token = null; }
}
// 配合 Service Worker 可持久化到闭包（但复杂度高）

// 方案C：localStorage（❌ 不安全）
// 任何 XSS 都可以读取 localStorage.getItem('token')
// 永远不要将 access token 存储在 localStorage

// ═══════ 混合方案（推荐）═══════
// access token: 内存中，短期有效（15分钟）
// refresh token: httpOnly Secure SameSite=Strict Cookie
// 页面刷新时，用 refresh token 换取新 access token
```

### 4.2 OAuth 2.0 + PKCE 流程

```typescript
// ═══════ SPA OAuth 2.0 PKCE 流程 ═══════

// 1. 生成 code_verifier 和 code_challenge
function generatePKCE() {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  const verifier = btoa(String.fromCharCode(...array))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');

  // 用 SHA-256 生成 challenge
  const encoder = new TextEncoder();
  return crypto.subtle.digest('SHA-256', encoder.encode(verifier))
    .then(hash => {
      const challenge = btoa(String.fromCharCode(...new Uint8Array(hash)))
        .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
      return { verifier, challenge };
    });
}

// 2. 重定向到授权端点
async function redirectToAuth() {
  const { verifier, challenge } = await generatePKCE();
  sessionStorage.setItem('code_verifier', verifier);

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: 'your-client-id',
    redirect_uri: 'https://yourapp.com/callback',
    code_challenge: challenge,
    code_challenge_method: 'S256',
    scope: 'openid profile email',
    state: crypto.randomUUID(), // 防 CSRF
  });

  location.href = `https://auth.example.com/authorize?${params}`;
}

// 3. 回调处理：用 code 换取 token
async function handleCallback() {
  const params = new URLSearchParams(location.search);
  const code = params.get('code');
  const state = params.get('state');
  const verifier = sessionStorage.getItem('code_verifier');

  const res = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code: code!,
      code_verifier: verifier!,
      redirect_uri: 'https://yourapp.com/callback',
      client_id: 'your-client-id',
    }),
  });

  const tokens = await res.json();
  sessionStorage.removeItem('code_verifier');
  // tokens.access_token → 存内存
  // tokens.refresh_token → 后端设置 httpOnly cookie
}
```

### 4.3 Refresh Token Rotation

```typescript
// ═══════ 刷新令牌轮换机制 ═══════
// 每次刷新都被消耗，颁发新 refresh token
// 检测 refresh token 重用 → 撤销整个 token 家族（防攻击）

// 服务端逻辑（Node.js + Redis）
async function rotateRefreshToken(oldToken: string, userId: string) {
  const family = await redis.get(`refresh_family:${oldToken}`);

  if (!family) {
    // Token 不在数据库中 → 可能是重放攻击
    // 撤销整个家族的所有 token
    await redis.del(`refresh_family:all:${family}`);
    throw new Error('Token reuse detected — all sessions revoked');
  }

  // 正常轮换：生成新 refresh token
  const newToken = crypto.randomBytes(48).toString('base64');
  await redis.set(`refresh_family:${newToken}`, family, 'EX', 7 * 24 * 3600);
  await redis.del(`refresh_family:${oldToken}`); // 消耗旧 token

  return newToken;
}

// 前端处理 refresh token rotation
class AuthClient {
  private accessToken: string | null = null;
  private refreshPromise: Promise<string> | null = null;

  async fetchWithAuth(url: string, options: RequestInit = {}): Promise<Response> {
    if (!this.accessToken) await this.refresh();

    const res = await fetch(url, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${this.accessToken}` },
    });

    if (res.status === 401) {
      await this.refresh();
      return this.fetchWithAuth(url, options);
    }
    return res;
  }

  private async refresh(): Promise<string> {
    // 防并发刷新：多个请求同时 401 时只刷新一次
    if (!this.refreshPromise) {
      this.refreshPromise = fetch('/api/refresh', {
        method: 'POST',
        credentials: 'include', // 携带 httpOnly cookie
      })
        .then(r => r.json())
        .then(data => {
          this.accessToken = data.accessToken;
          this.refreshPromise = null;
          return data.accessToken;
        })
        .catch(err => {
          this.refreshPromise = null;
          throw err;
        });
    }
    return this.refreshPromise;
  }
}
```

### 4.4 Passkeys (WebAuthn)

```typescript
// ═══════ Passkeys 基础流程 ═══════
// 浏览器内建的无密码认证，基于 FIDO2/WebAuthn 标准
// 使用设备生物识别（指纹/面部）或 PIN

// 注册 Passkey
async function registerPasskey() {
  // 1. 服务端生成注册挑战
  const res = await fetch('/api/webauthn/register/start', { method: 'POST' });
  const options = await res.json();

  // 2. 调用 WebAuthn API 创建密钥对
  const credential = await navigator.credentials.create({
    publicKey: {
      challenge: Uint8Array.from(atob(options.challenge), c => c.charCodeAt(0)),
      rp: { name: 'Your App', id: 'yourapp.com' },
      user: {
        id: Uint8Array.from(options.userId, c => c.charCodeAt(0)),
        name: options.email,
        displayName: options.name,
      },
      pubKeyCredParams: [{ type: 'public-key', alg: -7 }, { type: 'public-key', alg: -257 }],
      authenticatorSelection: {
        authenticatorAttachment: 'platform',  // 平台内置（Touch ID/Face ID）
        userVerification: 'required',
      },
    },
  });

  // 3. 将凭证发送给服务端验证并存储公钥
  await fetch('/api/webauthn/register/finish', {
    method: 'POST',
    body: JSON.stringify(credential),
  });
}

// 登录（使用 Passkey）
async function loginWithPasskey() {
  const res = await fetch('/api/webauthn/login/start', { method: 'POST' });
  const options = await res.json();

  const assertion = await navigator.credentials.get({
    publicKey: {
      challenge: Uint8Array.from(atob(options.challenge), c => c.charCodeAt(0)),
      allowCredentials: options.allowCredentials.map((c: any) => ({
        ...c,
        id: Uint8Array.from(atob(c.id), ch => ch.charCodeAt(0)),
      })),
      userVerification: 'required',
    },
  });

  await fetch('/api/webauthn/login/finish', {
    method: 'POST',
    body: JSON.stringify(assertion),
  });

  // 注意：Passkey 的 private key 永远不离开设备
  // 服务端只存储 public key
}
```

---

## 五、安全 Header

### 5.1 核心安全响应头

```nginx
# ═══════ 生产环境 Nginx 安全头配置 ═══════

# 1. HSTS — 强制 HTTPS
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# 2. X-Content-Type-Options — 禁止 MIME 嗅探
add_header X-Content-Type-Options "nosniff" always;

# 3. X-Frame-Options — 防点击劫持（CSP frame-ancestors 更现代）
add_header X-Frame-Options "DENY" always;
# 或在 CSP 中: frame-ancestors 'none';

# 4. Referrer-Policy — 控制 Referer 信息泄漏
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# 5. Permissions-Policy — 控制浏览器 API 访问
add_header Permissions-Policy "
  camera=(),                    # 禁用摄像头
  microphone=(),                # 禁用麦克风
  geolocation=(self),           # 仅同源可用
  payment=(),                   # 禁用 Payment API
  usb=(),                       # 禁用 WebUSB
  magnetometer=(),              # 禁用传感器
  autoplay=(self),              # 仅同源自动播放
  display-capture=(),           # 禁用屏幕捕获
" always;

# 6. Cross-Origin 隔离头
# COOP — Cross-Origin Opener Policy
add_header Cross-Origin-Opener-Policy "same-origin" always;

# COEP — Cross-Origin Embedder Policy（需要 COOP）
# ⚠️ 开启 COEP 要求所有跨域资源带 CORP 头或 CORS
# add_header Cross-Origin-Embedder-Policy "require-corp" always;

# CORP — Cross-Origin Resource Policy
add_header Cross-Origin-Resource-Policy "same-origin" always;

# 7. Cache-Control（防止敏感数据缓存到磁盘）
# 仅对包含认证信息的响应
if ($http_authorization) {
  add_header Cache-Control "no-store" always;
}
```

### 5.2 安全头最佳实践矩阵

| 安全头 | 推荐值 | 风险等级 | 适用场景 |
|--------|--------|---------|---------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` | 低风险 | 所有 HTTPS 站点 |
| `Content-Security-Policy` | `script-src 'strict-dynamic' 'nonce-...'` | 高风险 | 所有动态站点 |
| `X-Content-Type-Options` | `nosniff` | 低风险 | 所有站点 |
| `X-Frame-Options` | `DENY` 或 `SAMEORIGIN` | 中风险 | 所有站点（或 CSP `frame-ancestors`） |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 低风险 | 所有站点 |
| `Permissions-Policy` | 按需最小化 | 低风险 | 所有站点 |
| `Cross-Origin-Opener-Policy` | `same-origin` | 中风险 | 需要 `SharedArrayBuffer` 或隔离 |
| `Cross-Origin-Resource-Policy` | `same-origin` | 中风险 | API 响应、静态资源 |
| `Cross-Origin-Embedder-Policy` | `require-corp` | 高风险 | 需要 `SharedArrayBuffer`（谨慎） |
| `Cache-Control` | `no-store`（认证页面） | 低风险 | 认证后的页面/API |

### 5.3 安全头验证工具

```bash
# ═══════ 在线检测 ═══════
# 1. https://securityheaders.com/
# 2. https://observatory.mozilla.org/
# 3. https://csp-evaluator.withgoogle.com/

# ═══════ 命令行检测 ═══════
# curl 检查返回头
curl -I https://yourapp.com

# nuclei 模板扫描
nuclei -u https://yourapp.com -t exposures/configs/security-headers.yaml

# ═══════ CI 中自动化检测 ═══════
# 使用 httpx + 自定义检查
cat sites.txt | httpx -silent -include-response-header \
  | grep -L 'Content-Security-Policy'
```

---

## 六、安全审计清单

### 6.1 12 项生产安全检查

```markdown
## 生产部署前安全检查清单

### 1. CSP 头配置
- [ ] CSP 头已设置（非 Report-Only 模式）
- [ ] script-src 使用 strict-dynamic + nonce（非 allowlist）
- [ ] 已排除 'unsafe-eval' 和 'unsafe-inline'
- [ ] 已设置 report-uri 端点
- [ ] 使用 https://csp-evaluator.withgoogle.com/ 验证

### 2. XSS 防护
- [ ] 代码审查：所有 dangerouslySetInnerHTML/v-html 有净化
- [ ] 已引入 DOMPurify 或使用 Sanitizer API
- [ ] Trusted Types 在生产环境启用（require-trusted-types-for 'script'）
- [ ] 所有用户输入进入 HTML 前经过转义

### 3. 依赖安全
- [ ] npm audit 无 High/Critical 漏洞
- [ ] lockfile 完整性已验证（npm ci 通过）
- [ ] 使用 Socket.dev 或类似工具扫描行为异常
- [ ] 关键第三方 CDN 脚本使用 SRI

### 4. 认证安全
- [ ] JWT access token 不存储在 localStorage
- [ ] 使用 httpOnly + Secure + SameSite=Strict cookie 存储 refresh token
- [ ] OAuth 使用 PKCE 流程
- [ ] 实现 refresh token rotation

### 5. 安全头
- [ ] HSTS 正确配置（max-age ≥ 1 年）
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options: DENY 或 CSP frame-ancestors
- [ ] Referrer-Policy 已设置
- [ ] Permissions-Policy 已最小化

### 6. CSRF 防护
- [ ] 状态变更请求使用 POST/PUT/DELETE（非 GET）
- [ ] SameSite cookie 设置
- [ ] 自定义请求头验证（如 X-Requested-With）
- [ ] 或使用 double-submit cookie 模式

### 7. HTTPS/TLS
- [ ] 全站 HTTPS（HSTS preload 候选）
- [ ] TLS 1.2+, 禁用 SSL/TLS 1.0/1.1
- [ ] 证书有效期 ≤ 90 天（自动续期）

### 8. 敏感数据
- [ ] 密码/Token/密钥不提交到 Git
- [ ] .env 文件不在仓库中（.gitignore）
- [ ] 前端代码中不出现 API Key/Secret
- [ ] 使用 git-secrets 或 truffleHog 扫描

### 9. 第三方安全
- [ ] iframe 嵌入的第三方内容使用 sandbox 属性
- [ ] 外部链接使用 rel="noopener noreferrer"
- [ ] 第三方脚本的 nonce 正确设置
- [ ] CSP 策略覆盖所有第三方资源域

### 10. 数据保护
- [ ] 输入验证（前端+后端双层）
- [ ] 输出编码（按上下文：HTML/JS/URL/CSS）
- [ ] 文件上传类型白名单（非黑名单）
- [ ] 敏感数据不在 URL query string 中传输

### 11. 日志与监控
- [ ] CSP violation 报告已接入监控系统
- [ ] 异常登录模式告警
- [ ] 关键操作审计日志

### 12. 安全流程
- [ ] 安全漏洞披露政策（SECURITY.md）
- [ ] 依赖自动更新/Dependabot 配置
- [ ] 定期安全演练时间表
```

### 6.2 自动化安全扫描脚本

```bash
#!/bin/bash
# security-audit.sh — 前端安全自动化扫描

echo "=== 1. npm 审计 ==="
npm audit --omit=dev --audit-level=high

echo "=== 2. lockfile 完整性检查 ==="
npx lockfile-lint --path package-lock.json --validate-https --validate-integrity

echo "=== 3. 密钥泄漏扫描 ==="
npx trufflehog filesystem . --no-update

echo "=== 4. 依赖行为分析 ==="
npx @socketsecurity/cli scan package.json

echo "=== 5. HTTP 安全头检查 ==="
curl -sI https://yourapp.com | grep -E "^Strict-Transport|^Content-Security|^X-Content-Type|^X-Frame|^Referrer-Policy|^Cross-Origin" || echo "请替换为实际 URL"

echo "=== 审计完成 ==="
```

---

**参考文献**:
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CSP Level 3 Specification](https://www.w3.org/TR/CSP3/)
- [Trusted Types Specification](https://w3c.github.io/trusted-types/dist/spec/)
- [MDN Web Security](https://developer.mozilla.org/en-US/docs/Web/Security)
- [Steve Kinney — Web Security](https://stevekinney.net/courses/web-security)
- [OAuth 2.0 for Browser-Based Apps (BCP)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
- [W3C Web Authentication](https://www.w3.org/TR/webauthn-2/)
