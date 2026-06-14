# 架构决策树 (Architectural Decisions)

> 每个决策节点都附带权衡因子和实战推荐。原则：没有银弹，只有场景适配。

---

## 一、渲染模式选择: SPA vs SSR vs SSG vs ISR

```
                    开始: 渲染模式决策
                          │
            ┌─────────────┼─────────────┐
            ▼              ▼              ▼
       需要 SEO?      首屏速度优先?    交互极其复杂?
        │              │              │
    Yes │          Yes │          Yes │
        ▼              ▼              ▼
   SSR/SG/ISR      SSR/SG          SPA
        │              │
        └──────┬───────┘
               │
       内容多久更新一次?
       ├─ 极少更新(博客/文档) ──→ SSG
       ├─ 频繁更新(商品价格/库存) ──→ ISR (定时重新生成)
       ├─ 每次请求都不同(用户仪表盘) ──→ SSR
       └─ 混合(产品页=SSG + 用户部分=CSR) ──→ SSG + CSR Hydration
```

### 决策矩阵

| 维度 | SPA | SSR | SSG | ISR |
|------|-----|-----|-----|-----|
| **SEO** | 差 (需额外预渲染) | 优 | 优 | 优 |
| **首屏速度 (FCP)** | 慢 (JS Bundle → 渲染) | 快 (服务端直出 HTML) | 最快 (CDN 静态 HTML) | 快 (CDN → 过期后 SSR) |
| **TTI** | 快 (无服务端负载) | 取决于水合(JS Hydration) | 快 | 取决于水合 |
| **交互复杂度** | 最高 | 中 (水合期间不可交互) | 中 | 中 |
| **服务器成本** | 最低 (CDN 即可) | 高 (Node 渲染服务) | 最低 (CDN) | 中 (增量渲染) |
| **更新延迟** | 无 (客户端实时) | 无 (每次请求实时) | 有 (需重新构建) | 有 (取决于 revalidate) |
| **部署复杂度** | 低 | 中 (需 Node 环境) | 中 (需构建+CDN) | 中-高 |
| **典型案例** | Figma/canva/后台管理 | 电商首页/社交媒体 | 博客/文档站 | 商品详情页 |

### 实战建议

```
优选 ISR + CSR 混合:
  框架: Next.js / Nuxt
  策略:
    静态部分(布局/导航/公共内容) → SSG/ISR
    动态部分(用户数据/实时内容) → CSR 水合
    超动态部分(消息/通知) → 纯 CSR

避免全量 SSR (除非 SEO 要求极高):
  成本: 每请求一次 Node.js 渲染 = ~10x 纯 CDN 成本
  最佳实践: CDN 边缘 SSR (Vercel Edge / Cloudflare Workers)
```

---

## 二、状态管理选型决策树

```
开始: 选择状态管理方案
      │
      ▼
状态被几个组件使用?
│
├─ 1 个 ──→ useState / ref
│
├─ 2-3 个(父子或兄弟) ──→ Props 提升到最近公共父组件
│
└─ 多个/跨层级/跨路由 ──→
      │
      ▼
状态是否来自服务端?
│
├─ 是 ──→ React Query / SWR / VueUse useFetch
│   (服务端状态不应放入全局状态库)
│
└─ 否 (客户端状态) ──→
      │
      ▼
状态更新频率?
├─ 低频 (点击/提交) ──→ Zustand / Pinia
├─ 高频 (>1次/秒: 实时输入/动画) ──→ Jotai / ref + forceUpdate
│
      ▼
状态是否有派生逻辑?
├─ 有大量派生 ──→ Jotai (原子化派生)
└─ 无派生 ──→ Zustand (简单直接)
      │
      ▼
状态是否有严格的有限状态机?
├─ 是 ──→ XState
└─ 否 ──→ Zustand / Redux Toolkit
      │
      ▼
团队规模?
├─ >10人 + 规范要求 ──→ Redux Toolkit (强约束)
└─ <10人 ──→ Zustand (灵活快速)
```

### 决策要点

```
React Query / SWR 覆盖 80% 的状态:
  服务端数据的缓存、自动重新获取、乐观更新、分页/无限滚动
  团队常犯错误: 将 API 返回的列表存入 Zustand，然后手动管理过期/重试
  正确做法: API 数据用 React Query，仅 UI 状态(模态框/表单)用 Zustand

不要过度集中化:
  对话框状态、表单局部状态、tooltip 状态 → useState
  只有真正跨路由/跨模块的状态才需要全局管理
```

---

## 三、CSS 方案选型

```
开始: CSS 方案决策
      │
      ▼
项目规模?
├─ 小型/个人项目 ──→ Tailwind (最快原型速度)
├─ 中型(3-10万行) ──→ CSS Modules + PostCSS 或 Tailwind
│   (组件隔离 + 零运行时)
└─ 大型(>10万行) / 多团队 ──→
      │
      ▼
是否需要运行时动态样式?
├─ 是(主题切换/用户自定义颜色) ──→ CSS Variables + Vanilla Extract
└─ 否 ──→ CSS Modules 或 Tailwind
      │
      ▼
构建速度要求?
├─ 极致(Vite/HMR <100ms) ──→ Tailwind / CSS Modules
│   (零运行时 CSS-in-JS 也可)
└─ 正常 ──→ 无特殊限制
      │
      ▼
团队偏好?
├─ 设计系统驱动/原子化哲学 ──→ Tailwind
├─ 组件内聚/CSS 文件独立 ──→ CSS Modules
├─ 类型安全/JS 编写样式 ──→ Vanilla Extract / Panda CSS
└─ 设计高度定制/像素级还原 ──→ CSS-in-JS (慎重)
```

### 对比表

| 方案 | 运行时成本 | 类型安全 | 学习曲线 | 打包体积 | 动态样式 | 死亡样式消除 |
|------|----------|---------|---------|---------|---------|-------------|
| **CSS Modules** | 无 | 弱 (*.module.css.d.ts) | 低 | 小 | 弱 (CSS vars) | 天然支持 |
| **Tailwind CSS** | 无 (JIT) | 无 | 中-高 | 极小(仅使用类) | 弱 | 天然支持 |
| **CSS-in-JS (Emotion/Styled)** | **有**(每次渲染) | 优 | 中 | 运行时 ~12KB | 强 | 弱 (需插件) |
| **Vanilla Extract** | **无**(构建时) | 优 | 中 | 小 | 弱 (CSS vars + contract) | 天然支持 |
| **Panda CSS** | **无**(构建时) | 优 | 中 | 小 | 弱 (recipes) | 天然支持 |

### 实战建议

```
2025 年推荐顺序:
  1. Tailwind CSS — 适合大部分场景(设计系统+快速原型)
  2. CSS Modules — 适合组件库开发、需要传统 CSS 的团队
  3. Vanilla Extract / Panda CSS — 适合 TypeScript 重度团队
  4. CSS-in-JS (运行时) — 仅遗留项目维护

关键原则:
  - 零运行时是趋势(构建时生成纯 CSS)
  - 避免在同一个项目中混用 2 种以上 CSS 方案
  - 不管用什么方案，必须设置 Design Token (颜色/间距/字体/圆角/阴影)
```

---

## 四、构建工具选型

```
开始: 构建工具决策
      │
      ▼
新项目 or 已有 Webpack 配置?
├─ 已有 Webpack ──→ 迁移到 Rspack (兼容 webpack 生态，构建速度 5-10x)
└─ 新项目 ──→
      │
      ▼
是否需要兼容 Node.js 旧版本 / CommonJS?
├─ 是 ──→ Webpack / Rspack
└─ 否 ──→ Vite
      │
      ▼
项目规模?
├─ 小型(<100 模块) ──→ Vite (esbuild 足够快)
├─ 中型(100-2000 模块) ──→ Vite (按需编译 = O(变更) 非 O(总))
│   但 build 阶段可选 Rolldown (比 Rollup 更快)
└─ 大型(>2000 模块) / Monorepo ──→
   ├─ 开发: Vite + Rolldown
   └─ 构建: 考虑 Turbopack (如果是 Next.js 生态)
```

### 对比表

| 工具 | 开发服务器 | 构建 | HMR | 生态 | 适用 |
|------|----------|------|-----|------|------|
| **Webpack** | 慢 (全量打包) | 中 | 中 | 最大 | 遗留项目 |
| **Vite** | 极快 (按需编译) | 中(Rollup)/快(Rolldown) | 极快 (<50ms) | 快速增长 | 新项目首选 |
| **Turbopack** | 快 (增量) | 快 | 快 | Next.js 专属 | Next.js 项目 |
| **Rspack** | 快 | 快 | 快 | 兼容 Webpack | Webpack 迁移 |
| **Rolldown** | 极快 (Vite 集成) | 快 | 极快 | 中(兼容 Rollup) | Vite 构建加速 |

### Rolldown (2025+)

```
Rolldown 是 Vite 下一阶段的默认打包器 (Rust 实现):
  - 兼容 Rollup 插件 API (大部分)
  - 构建速度: 比 Rollup 快 10-30x
  - Tree Shaking: 与 Rollup 相同精度
  - 未来会同时支持 transform hook (替代 esbuild 的部分职能)
```

---

## 五、微前端决策阈值

```
开始: 是否需要微前端?
      │
      ▼
团队数量 > 3 个独立前端团队?
├─ 否 ──→ 不要微前端 (Monorepo 即可)
│
└─ 是 ──→
      │
      ▼
满足以下至少 2 项?
  □ 各团队需要独立部署/发布
  □ 技术栈异构(React + Vue + 原生)
  □ 渐进式迁移(逐步替换遗留系统)
  □ 独立灰度策略/AB 测试
│
├─ 否 ──→ Monorepo + Module Federation 共享模块
│
└─ 是 ──→
      │
      ▼
技术栈是否统一?
├─ 统一(全 React/Vue) ──→ Module Federation
├─ 异构 ──→
│   ├─ Web 环境 ──→ wujie (组件级隔离)
│   └─ 小程序环境(每个分包独立) ──→ 框架原生分包能力
│
      ▼
通信需求?
├─ 低频(路由同步/用户信息) ──→ URL + shared state
└─ 高频(实时协作) ──→ postMessage + 事件协议 + 类型定义共享
```

### 阈值量化

```
推荐微前端的条件 (AND 逻辑):
  1. 独立前端团队 >= 3
  2. 至少一个团队需要周级独立发布
  3. 代码量 > 50万行 或 monorepo CI 时间 > 10 分钟
  4. 至少两个子应用在不同技术栈

不推荐的场景:
  - 同一个应用内的不同路由/页面 (用代码分割即可)
  - 仅为了"解耦" (模块边界清晰 + Monorepo 已足够)
  - 追求"技术先进性" (微前端的运维成本是单体的 3-5x)
```

---

## 六、Monorepo vs Multi-Repo

```
开始: 仓库策略
      │
      ▼
共享代码量?
├─ >30% 代码共享 ──→ Monorepo (共享类型/工具/组件)
├─ <10% 代码共享 ──→ Multi-Repo (独立构建/部署)
└─ 10%-30% ──→ Monorepo + 严格 package 边界
      │
      ▼
团队耦合度?
├─ 频繁跨包修改 ──→ Monorepo (一个 PR 跨多个包)
├─ 极少跨包修改 ──→ Multi-Repo
      │
      ▼
构建工具链?
├─ 使用 Turborepo/Nx ──→ Monorepo 的增量构建优势
└─ 无法统一构建工具 ──→ Multi-Repo
```

### Monorepo 工具对比

| 工具 | 缓存 | 并行 | 远程缓存 | 依赖图 | 适合 |
|------|------|------|---------|--------|------|
| **Turborepo** | 本地+远程 | ✓ | ✓ (Vercel) | 自动 | 通用首选 |
| **Nx** | 本地+远程 | ✓ | ✓ (Nx Cloud) | 自动+插件 | 大型企业 |
| **pnpm workspace** | 无 | 手动 | 无 | 手动 | 小型 Monorepo |
| **Rush** | 本地+远程 | ✓ | ✓ | 自动 | 微软生态 |

### 实战经验值

```
Monorepo 成功的关键:
  1. 统一 TypeScript 版本 (root tsconfig → extends)
  2. 统一 lint 规则 (eslint-config 共享包)
  3. 锁定依赖版本 (pnpm-lock.yaml)
  4. 包间依赖版本使用 workspace:*
  5. CI 只构建变更影响的包 (Turborepo --filter=[HEAD^1])

Multi-Repo 成功的关键:
  1. 共享代码发布为 npm 包 (语义化版本)
  2. 类型定义随包发布
  3. 跨仓库的 CI 触发用 webhook
  4. 统一的 lint/prettier 配置 (可远程引用)
```

---

## 七、图片策略

```
开始: 图片优化决策
      │
      ▼
图片类型?
├─ 照片/截图 ──→ WebP (主) + JPEG (兼容) + AVIF (仅有现代浏览器)
├─ 图标/Logo ──→ SVG (矢量) 或 PNG (带 alpha)
├─ 插画/动画 ──→ SVG (静态) 或 Lottie (动画)
└─ 透明背景图 ──→ WebP (有损/无损) 或 PNG (兼容)
      │
      ▼
响应式需求?
├─ 是 ──→ srcset + sizes (或 <picture> + source type)
└─ 否 ──→ 提供 2x 版本 (Retina 适配)
      │
      ▼
加载策略?
├─ 首屏关键图片(LCP候选) ──→ eager + fetchpriority="high" + preload
├─ 折叠线以下 ──→ lazy + decoding="async"
└─ 不确定 ──→ 默认 lazy
      │
      ▼
生成策略?
├─ Next.js ──→ next/image (自动 WebP/AVIF + srcset + 懒加载)
├─ 其他框架 ──→ 构建时生成多尺寸 + CDN 动态裁剪(如 imgix/Cloudinary)
└─ 无后端 ──→ sharp 构建时生成 + 多格式输出
```

### 格式选择决策

| 用例 | 首选 | 降级 | 不推荐 |
|------|------|------|--------|
| 产品照片 (色彩丰富) | AVIF (体积最小) | WebP → JPEG | PNG |
| 截图/UI (文字边缘) | WebP 无损 或 PNG | — | AVIF (文字模糊) |
| 图标 | SVG | PNG @2x | — |
| 背景装饰 | WebP | JPEG | — |
| 透明产品图 | WebP (支持 alpha) | PNG | — |

---

## 八、组件库: 自建 vs 第三方

```
开始: 组件库决策
      │
      ▼
设计稿定制化程度?
├─ 高度定制(自研设计系统) ──→ 自建 (基于 Radix/Headless/Aria)
└─ 中等/可接受限制 ──→
      │
      ▼
团队能力和时间?
├─ 团队 >= 4人 + 有组件库经验 + 6个月以上周期 ──→ 自建
├─ 团队 < 4人 或 时间 < 3个月 ──→ 第三方 + 主题覆写
      │
      ▼
维护预算?
├─ 持续投入预算 (至少 1人/年) ──→ 自建
└─ 无维护预算 ──→ 第三方 (把维护交给社区)
      │
      ▼
可访问性要求?
├─ WCAG AA 强制 ──→ Radix UI / Headless UI 基础 + 自建样式
└─ 无强制要求 ──→ 第三方完整组件库
```

### 推荐方案

| 场景 | 推荐组合 |
|------|---------|
| React + 高度定制 | Radix UI (行为) + Tailwind (样式) |
| React + 快速开发 | Ant Design / Mantine |
| Vue + 高度定制 | Radix Vue (或 Headless) + UnoCSS/Tailwind |
| Vue + 快速开发 | Element Plus / Naive UI |
| 跨平台 (RN + Web) | Tamagui / 自建 shared primitives |
| 移动端 (仅 RN) | React Native Paper / NativeBase |
