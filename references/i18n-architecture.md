# i18n 国际化架构设计

> 本文档整理前端国际化 (i18n) 的完整架构知识体系，
> 覆盖 ICU Message Format、翻译文件组织、框架集成、RTL 适配及翻译管理平台选型。
> 基于 ICU 规范、react-intl/next-intl/vue-i18n 官方文档、CLDR 数据标准及一线多语言产品经验。

---

## 一、国际化架构设计

### ICU Message Format 深度

ICU (International Components for Unicode) MessageFormat 是 i18n 的工业标准语法，支持复数、性别、选择（select）等复杂语言结构。

```
语法结构:
  {count, plural, =0 {无结果} one {# 条结果} other {# 条结果}}
  {gender, select, male {他} female {她} other {TA}}
  {num, number, ::currency/USD}  // 数字格式化
  {date, date, ::long}           // 日期格式化
```

#### 复数规则（Plural）

```javascript
// ICU 复数类别: zero, one, two, few, many, other
// 不同语言的复数类别完全不同！

// 英语: one / other
// 阿拉伯语: zero / one / two / few / many / other
// 中文: other（仅一种形式）
// 俄语: one / few / many / other
// 日语: other（仅一种形式）

// 翻译键定义
{
  "itemCount": "{count, plural, =0 {No items} one {# item} other {# items}}"
}

// 使用时
<FormattedMessage id="itemCount" values={{ count: 5 }} />
// 渲染: "5 items"
// count=0 时: "No items"
// count=1 时: "1 item"

// ⚠️ 中文陷阱
// 中文只有 other，所有数量使用同一形式
{
  "searchResults": "{count, plural, other {共 # 条结果}}"
}
// 所有 count 值都匹配 other → "共 5 条结果"
```

#### 选择（Select）— 性别/变体

```javascript
// 基于变量的条件选择
{
  "greeting": "{gender, select, male {Welcome, Mr. {name}} female {Welcome, Ms. {name}} other {Welcome, {name}}}",
  "fileStatus": "{status, select, uploading {Uploading...} processing {Processing...} done {Completed} failed {Failed to upload} other {Unknown status}}"
}

// 使用
<FormattedMessage
  id="greeting"
  values={{ gender: 'male', name: 'Smith' }}
/>
// 渲染: "Welcome, Mr. Smith"
```

#### 数字/日期/时间格式化

```javascript
// ICU 数字格式化
{
  "price": "{value, number, ::currency/USD}",
  "percent": "{value, number, ::percent scale/100}",
  "fileSize": "{value, number, ::kilobyte unit-width-short}"
}
// value=42.5, price → "$42.50"
// value=0.855, percent → "85.5%"

// ICU 日期格式化
{
  "publishedAt": "{date, date, ::yyyyMMdd}",
  "eventDate": "{date, date, ::full}",
  "meetingTime": "{time, time, ::short}"
}
// date=new Date('2025-03-15'), eventDate → "Saturday, March 15, 2025"
// ICU 的日期格式与 Intl.DateTimeFormat 共享 CLDR 数据
```

### 翻译文件组织策略

```
┌──────────────────────────────────────────────────────────────┐
│  方案 A: namespaced/key-based（推荐大型应用）                 │
├──────────────────────────────────────────────────────────────┤
│  locales/                                                    │
│  ├── en/                                                     │
│  │   ├── common.json       { "ok": "OK", "cancel": "Cancel" }│
│  │   ├── auth.json         { "login": "Login", ... }         │
│  │   ├── dashboard.json    { "title": "Dashboard", ... }     │
│  │   └── settings.json     { "profile": "Profile", ... }     │
│  ├── zh-CN/                                                  │
│  │   ├── common.json       { "ok": "确定", ... }             │
│  │   └── ...                                                 │
│  └── ar/                  ← RTL 语言                          │
│      └── ...                                                 │
├──────────────────────────────────────────────────────────────┤
│  方案 B: file-based（适合小型/文档翻译）                      │
├──────────────────────────────────────────────────────────────┤
│  locales/                                                    │
│  ├── en.json  { "common.ok": "OK", "auth.login": "Login" }   │
│  ├── zh-CN.json  { "common.ok": "确定", ... }                │
│  └── ar.json                                                  │
├──────────────────────────────────────────────────────────────┤
│  方案 C: 嵌套 JSON（自然分组，无需命名空间配置）              │
├──────────────────────────────────────────────────────────────┤
│  locales/                                                    │
│  ├── en/                                                     │
│  │   └── index.json   { "home": { "title": "Home" }, ... }   │
│  └── zh-CN/                                                  │
│      └── index.json   { "home": { "title": "首页" }, ... }   │
└──────────────────────────────────────────────────────────────┘
```

#### 键命名规范

```javascript
// ✅ 推荐：语义化、可上下文理解
"auth.login.title"
"auth.login.passwordPlaceholder"
"auth.login.submitButton"
"dashboard.recentOrders.empty"
"dashboard.recentOrders.itemCount"  // ICU: "{count, plural, ...}"

// ❌ 避免：无意义缩写、数值 key
"btn1.label"
"msg_002"

// ✅ 推荐：kebab-case 或 camelCase（保持一致）
"auth.login.password-placeholder"  // kebab-case
"auth.login.passwordPlaceholder"   // camelCase

// ❌ 避免：混用
"auth.login.password_placeholder"  // snake_case 混合
```

### 编译时 vs 运行时翻译

| 维度 | 编译时 | 运行时 |
|------|-------|--------|
| **性能** | 零运行时开销，翻译直接内联 | 每次渲染查表，有轻微开销 |
| **Bundle** | 每种语言一个 bundle | 可按需加载语言包 |
| **切换语言** | 需重新加载页面 | 即热切换，无需刷新 |
| **支持复杂度** | 简单，适合 ICU | 可处理动态内容和复杂 ICU |
| **典型工具** | LinguiJS, Paraglide | react-intl, next-intl, vue-i18n |

```javascript
// 编译时示例（LinguiJS）
// 源码
<Trans>Hello {name}</Trans>
// 编译后
// en: "Hello {name}"
// zh: "你好{name}"
// 生成的 JS 中直接包含编译后的字符串

// 运行时示例（react-intl）
// 始终携带完整的 ICU 解析器和语言数据
import { IntlProvider, FormattedMessage } from 'react-intl';
<IntlProvider locale={locale} messages={messages}>
  <FormattedMessage id="greeting" values={{ name }} />
</IntlProvider>
```

---

## 二、框架集成

### react-intl / next-intl

```javascript
// React: react-intl（FormatJS 生态）
import { IntlProvider, useIntl, FormattedMessage } from 'react-intl';

// 1. 提供语言环境
function App({ locale, messages }) {
  return (
    <IntlProvider locale={locale} messages={messages}>
      <Router />
    </IntlProvider>
  );
}

// 2. 组件内使用
function Dashboard() {
  const intl = useIntl();
  // 声明式
  return (
    <div>
      <FormattedMessage id="dashboard.title" defaultMessage="Dashboard" />
      <FormattedMessage
        id="dashboard.itemCount"
        defaultMessage="{count, plural, =0 {No items} one {# item} other {# items}}"
        values={{ count: 42 }}
      />
      {/* 命令式 */}
      <button aria-label={intl.formatMessage({ id: 'common.close' })}>
        ×
      </button>
    </div>
  );
}

// Next.js: next-intl（App Router 原生支持）
// messages/en.json
// { "HomePage": { "title": "Welcome", "description": "..." } }

// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

export default async function LocaleLayout({ children, params: { locale } }) {
  const messages = await getMessages();
  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      {children}
    </NextIntlClientProvider>
  );
}

// 组件中使用
import { useTranslations } from 'next-intl';
function HomePage() {
  const t = useTranslations('HomePage');
  return <h1>{t('title')}</h1>;
}

// 服务端组件中使用
import { getTranslations } from 'next-intl/server';
async function ServerComponent() {
  const t = await getTranslations('HomePage');
  return <h1>{t('title')}</h1>;
}
```

### vue-i18n

```javascript
// Vue 3: vue-i18n v9+
import { createI18n } from 'vue-i18n';

const i18n = createI18n({
  legacy: false,           // Composition API 模式
  locale: 'zh-CN',
  fallbackLocale: 'en',   // 缺失时回退到英语
  messages: {
    en: {
      home: { title: 'Home', welcome: 'Welcome, {name}!' },
    },
    'zh-CN': {
      home: { title: '首页', welcome: '欢迎, {name}!' },
    },
  },
});

// SFC 中使用
<script setup>
import { useI18n } from 'vue-i18n';
const { t, locale } = useI18n();
</script>

<template>
  <h1>{{ t('home.title') }}</h1>
  <p>{{ t('home.welcome', { name: userName }) }}</p>
  <select v-model="locale">
    <option value="en">English</option>
    <option value="zh-CN">中文</option>
  </select>
</template>

// 懒加载语言包
const loadLocaleMessages = async (locale) => {
  const messages = await import(`@/locales/${locale}.json`);
  i18n.global.setLocaleMessage(locale, messages.default);
};
```

### 服务端渲染下的语言检测

```javascript
// Next.js: 通过中间件实现
// middleware.ts
import { NextRequest } from 'next/server';
import createIntlMiddleware from 'next-intl/middleware';

export default createIntlMiddleware({
  locales: ['en', 'zh-CN', 'ar', 'ja'],
  defaultLocale: 'en',
  // 语言检测策略:
  // 1. URL 前缀: /zh-CN/products（推荐，SEO 友好）
  // 2. Cookie: NEXT_LOCALE=zh-CN（无 URL 前缀时回退）
  // 3. Accept-Language header: 'zh-CN,zh;q=0.9,en;q=0.8'
  localeDetection: true,
});

// 自定义语言检测
export function middleware(request: NextRequest) {
  const acceptLanguage = request.headers.get('accept-language');
  const cookieLocale = request.cookies.get('NEXT_LOCALE')?.value;
  const urlLocale = request.nextUrl.pathname.split('/')[1];

  const detectedLocale =
    urlLocale ||
    cookieLocale ||
    parseAcceptLanguage(acceptLanguage) ||
    'en';

  // 重定向到带语言前缀的 URL
  if (!urlLocale) {
    return NextResponse.redirect(
      new URL(`/${detectedLocale}${request.nextUrl.pathname}`, request.url)
    );
  }
}
```

---

## 三、RTL (Right-to-Left) 适配

### CSS Logical Properties（逻辑属性）

```css
/* ❌ 物理属性（只适应 LTR） */
.box {
  margin-left: 20px;
  padding-right: 10px;
  border-left: 1px solid #ccc;
  text-align: left;
}

/* ✅ 逻辑属性（自动适应 LTR/RTL） */
.box {
  margin-inline-start: 20px; /* LTR=left, RTL=right */
  padding-inline-end: 10px;  /* LTR=right, RTL=left */
  border-inline-start: 1px solid #ccc;
  text-align: start;         /* LTR=left, RTL=right */
}

/* 逻辑属性映射速查
   block-start   → top (竖排时 top, 横排时根据书写模式)
   block-end     → bottom
   inline-start  → LTR 时为 left,  RTL 时为 right
   inline-end    → LTR 时为 right, RTL 时为 left
*/
```

### 布局反转策略

```scss
// 策略 1: 使用 CSS 变量注入方向敏感值
:root {
  --direction: ltr;
  --start: left;
  --end: right;
  --padding-start: 20px;
  --padding-end: 10px;
  --transform-direction: 1;
}

:root[dir="rtl"] {
  --direction: rtl;
  --start: right;
  --end: left;
  --padding-start: 10px;    // 值可能需要互换
  --padding-end: 20px;
  --transform-direction: -1;
}

// 策略 2: 使用 transform 做方向翻转
.icon-double-arrow {
  transform: scaleX(var(--transform-direction));
  // LTR: scaleX(1)  → 正常
  // RTL: scaleX(-1) → 水平翻转
}

// 策略 3: Flexbox/Grid 自动处理
.container {
  display: flex;
  // direction 属性自动控制 flex 方向
  // LTR: 从左到右
  // RTL: 从右到左
}

// ⚠️ 需要手动处理的属性
.arabic-text {
  font-family: 'Noto Naskh Arabic', sans-serif;
  direction: rtl;          // 必须显式设置
  text-align: right;       // 或 start
  letter-spacing: 0;       // 阿拉伯语连字需正常间距
}

// ❌ 禁止使用的物理属性（在 RTL 场景中）
.avoid-these {
  /* 使用 margin-inline-* 替代 margin-left/right */
  /* 使用 padding-inline-* 替代 padding-left/right */
  /* 使用 text-align: start/end 替代 left/right */
  /* 使用 border-inline-* 替代 border-left/right */
  /* 使用 inset-inline-* 替代 left/right */
}
```

### 常见 RTL 陷阱

```css
/* 陷阱 1: absolute 定位使用 left/right */
.badge {
  position: absolute;
  /* ❌ */
  right: -8px;
  /* ✅ */
  inset-inline-end: -8px;
}

/* 陷阱 2: 箭头图标旋转 */
.next-arrow::before {
  content: '→';
  /* ❌ LTR 专用 */
  /* ✅ 使用逻辑属性或条件渲染 */
}

/* 陷阱 3: box-shadow 偏移 */
.card {
  /* ❌ */
  box-shadow: 5px 0 10px rgba(0,0,0,0.1);
  /* ✅ */
  box-shadow: calc(var(--transform-direction) * 5px) 0 10px rgba(0,0,0,0.1);
}

/* 陷阱 4: transform 平移 */
.slide-in {
  /* ❌ */
  transform: translateX(-100%);
  /* ✅ */
  transform: translateX(calc(var(--transform-direction) * -100%));
}

/* 陷阱 5: clip-path 方向敏感 */
.progress-bar {
  /* ❌ */
  clip-path: inset(0 0 0 50%);  /* 物理方向 */
  /* ✅ */
  --clip-amount: 50%;
  /* 通过 JS 动态设置 */
}
```

### React 中的 RTL 处理

```javascript
// 1. 通过 dir 属性转发
function App({ locale }) {
  const isRTL = ['ar', 'fa', 'he', 'ur'].includes(locale);
  return (
    <html dir={isRTL ? 'rtl' : 'ltr'} lang={locale}>
      <body>
        <IntlProvider locale={locale} messages={messages}>
          {/* 组件自动适配 */}
        </IntlProvider>
      </body>
    </html>
  );
}

// 2. 条件渲染（万不得已时使用）
function ArrowIcon({ direction }) {
  const { locale } = useIntl();
  const isRTL = locale === 'ar' || locale === 'he';
  return isRTL ? <ArrowLeft /> : <ArrowRight />;
}

// 3. 动态样式
function Sidebar() {
  const { locale } = useIntl();
  const isRTL = locale === 'ar' || locale === 'he';

  return (
    <aside style={{
      [isRTL ? 'right' : 'left']: 0,
      transform: `translateX(${isRTL ? '' : '-'}100%)`
    }}>
      {/* sidebar content */}
    </aside>
  );
}
```

---

## 四、i18n 决策

### 翻译管理平台对比

| 平台 | 适用规模 | 核心优势 | 劣势 |
|------|---------|---------|------|
| **Lokalise** | 中-大型 | 强大协作、OTA 推送更新 (无需发版更新翻译) | 费用较高、企业版定价 |
| **Crowdin** | 中-大型 | GitHub/GitLab 深度集成、Machine Translation 引擎丰富 | UI 复杂、学习曲线陡 |
| **Phrase** | 中-大型 | 完备的翻译记忆 (TM)、术语库、翻译质量保证 (QA) | 配置繁琐 |
| **Tolgee** | 小型 | 开源、自带 in-context 编辑（点击页面直接修改翻译） | 规模化后性能 |
| **i18next** + 自建 | 定制需求 | 自托管、完全控制、零额外费用 | 需自行维护翻译工作流 |

### 翻译工作流

```
┌────────────────────────────────────────────────────────────┐
│  开发流程                                                   │
│  1. Developer 在代码中定义 key → 提取到 en.json            │
│  2. Push 到翻译平台 (Lokalise/Crowdin)                     │
│  3. Translator 翻译为 target locale                        │
│  4. 翻译平台自动创建 PR → CI 检查 → Merge                  │
│  5. 部署 → 用户看到新翻译                                   │
├────────────────────────────────────────────────────────────┤
│  紧急修复（OTA）                                            │
│  1. 翻译平台直接修改翻译文本                                │
│  2. Publish → CDN 推送                                     │
│  3. 客户端下载增量翻译包 → 即时生效，无需发版                │
│  ⚠️ 仅适用于运行时翻译方案，编译时方案需重新构建            │
└────────────────────────────────────────────────────────────┘
```

### 关键决策 Checklist

```
□ 是否需要 SEO 多语言? → 是 → 每个 locale 独立 URL (/en/ /zh/) 而非 Query String
□ 是否需要频繁更新翻译? → 是 → 运行时 + OTA，否则编译时更优
□ 支持语言数量? → ≤ 5 种 → 自维护 / > 5 种 → 翻译平台
□ 是否存在 RTL 语言? → 是 → 必须使用 CSS Logical Properties
□ 是否需要动态内容翻译? → 是 → 评估 Translation API (DeepL/Google Translate) 集成
□ 是否使用 Next.js App Router? → 是 → next-intl（原生支持 RSC）
□ 是否使用微前端? → 是 → 每个微前端独立翻译包 + 共享 common 语言包
```

### 语言检测优先级

```javascript
// 推荐优先级链
function detectLocale(request, cookies, userProfile) {
  // 1. URL 路径前缀: /zh-CN/products（显式、SEO 最优）
  const urlLocale = extractFromPath(request.url);

  // 2. 用户偏好: 登录用户在设置中指定的语言
  const userPref = userProfile?.preferredLocale;

  // 3. 会话 Cookie: 非登录用户在语言切换器中选择的
  const cookieLocale = cookies.get('lang')?.value;

  // 4. Accept-Language header: 浏览器默认语言
  const browserLocale = parseAcceptLanguage(
    request.headers.get('accept-language')
  );

  // 5. 默认语言
  const defaultLocale = 'en';

  return urlLocale || userPref || cookieLocale || browserLocale || defaultLocale;
}
```

---

## 相关参考

- ICU MessageFormat 规范: https://unicode-org.github.io/icu/userguide/format_parse/messages/
- CLDR 语言数据: https://cldr.unicode.org/
- FormatJS (react-intl): https://formatjs.io/docs/react-intl
- next-intl: https://next-intl-docs.vercel.app/
- vue-i18n: https://vue-i18n.intlify.dev/
- RTL 适配: 参考 `pitfalls/pit-073.md` (RTL 布局破坏)
- 翻译键缺失: 参考 `pitfalls/pit-072.md` (翻译键缺失)
