# 多维度检查清单 (Multi-Dimensional Checklists)

> 每条检查项背后都有血的教训。请不要跳过任何一条。

---

## 一、提交前检查 (Pre-commit Checklist)

### 代码质量
```
□ TypeScript 编译无错误 (tsc --noEmit)
□ ESLint 无 error 和 warning
□ 无 console.log / debugger 残留
   例外: console.error 用于错误上报 (需配合错误监控平台)
□ 无注释掉的代码块 (用 git history 追溯历史)
□ 无硬编码的 URL / 密钥 / Token
   正确: 环境变量 process.env.NEXT_PUBLIC_API_URL
□ 无 TODO/FIXME 标记 (或已创建对应的 JIRA Issue)
□ 文件命名遵循项目约定 (kebab-case / PascalCase / camelCase)
□ import 路径不使用相对路径回溯过多层
   正确: @/components/Button (alias) 而非 ../../components/Button
```

### 测试
```
□ 单元测试通过 (Jest/Vitest)
□ 新增代码有对应测试 (覆盖主要分支)
□ 快照测试无意外变更 (有意的变更已在 PR 中说明)
□ E2E 关键路径测试通过 (Playwright)
```

### 构建
```
□ 生产构建成功 (npm run build 无 error)
□ Bundle 大小无异常增长 (对比 baseline: +5% 内为正常)
   工具: webpack-bundle-analyzer / rollup-plugin-visualizer
□ 无 chunk 加载失败风险 (公共依赖提取正确)
```

### Git
```
□ commit message 遵循 Conventional Commits
   feat: add user login
   fix: resolve layout shift on mobile
   perf: optimize image loading
   refactor: extract Button component
□ PR 描述包含: 背景/改动/截图(如有 UI 变更)/测试/回滚方案
□ 只包含相关文件变更 (无意外文件)
```

---

## 二、React 组件检查

### 通用
```
□ 列表渲染每个项有稳定的 key (不使用 index)
   正确: key={item.id}
   错误: key={index} (排序/删除时导致错误复用)
   例外: 静态列表且无排序/删除可用 index
□ 所有 props 有 TypeScript 类型定义
□ 复杂 props 使用 interface 并 export (便于复用)
□ 组件文件 ≤ 300 行 (超过则拆分)
```

### Hooks
```
□ useEffect 依赖数组完整且正确
   (运行 eslint-plugin-react-hooks)
□ useEffect 有清理函数 (如果订阅/计时器/事件)
   return () => { clearInterval(timer); unsubscribe(); }
□ 不在条件语句/循环中调用 Hooks
□ useMemo/useCallback 有明确的缓存理由
   不是所有函数都需要 useCallback — 仅在作为 deps 传递时有意义
□ useRef 用于非渲染相关的可变值 (DOM ref/timer ID/前值)
□ 事件处理器使用 useCallback (如果它们作为子组件的 props)
□ 状态更新不可变 (对象/数组使用展开或 immer)
```

### 渲染
```
□ 组件不在渲染函数内部创建组件 (会导致每次重新挂载)
   错误:
   function Parent() {
     function Child() { return <div/>; } // 每次 Parent 渲染重新创建!
     return <Child />;
   }
□ 条件渲染优先使用三元/&&，而非 if-return null
□ React.lazy + Suspense 包裹路由级组件
□ Error Boundary 包裹关键子组件树
□ forwardRef 用于需要暴露 DOM 节点的组件
```

### Server Components (Next.js App Router)
```
□ 'use client' 只在需要交互性时使用
□ 服务端组件不 import 客户端专属库 (如 DOM API)
□ 数据获取在服务端组件中 (避免 client-side waterfall)
□ 客户端组件通过 props 从服务端组件接收数据 (而非内部 fetching)
```

---

## 三、CSS 检查

### 代码规范
```
□ 无 !important (除非覆盖第三方库样式，且必须注释原因)
□ z-index 使用统一的比例尺系统 (文档化)
   推荐: z-index 变量: --z-dropdown: 100; --z-modal: 200; --z-toast: 300;
□ 不使用 ID 选择器作样式 (#id)
□ 选择器嵌套不超过 3 层
□ 不使用 * 全局选择器 (除非重置样式)
□ 颜色使用 CSS 变量 (支持主题切换)
```

### 响应式
```
□ 移动端优先 (min-width Media Query)
□ 测试 320px - 1920px 范围内的布局
□ 使用相对单位 (rem/em/%) 替代固定 px (字体/间距)
□ 避免 vh 单位 (移动端 Safari 100vh 包含地址栏)
   替代: dvh (动态视口高度) 或 svh (小视口高度)
□ 媒体查询用 rem (不受用户缩放影响)
```

### 布局
```
□ 使用 Flexbox/Grid 而非 float/position 布局
□ Flexbox gap 有兼容性降级 (Safari <14.1 不支持 Flex gap)
□ Grid 布局有明确的 grid-template-columns/rows
□ 滚动容器设置 overflow-y: auto 和 -webkit-overflow-scrolling: touch
□ 使用 aspect-ratio 控制媒体宽高比 (防止 CLS)
□ 图片/视频有明确的 width/height 属性
```

### 性能
```
□ 使用 contain 属性隔离子树重排重绘
□ will-change 在动画开始前设置、结束后移除
□ 动画使用 transform + opacity (纯合成层，无重排)
□ @supports 包裹新特性降级
□ 无渲染阻塞的 @import (改用 <link>)
```

---

## 四、性能检查 (Core Web Vitals)

### LCP (Largest Contentful Paint) — 目标 < 2.5s
```
□ 首屏关键图片使用 eager + fetchpriority="high"
□ LCP 元素预加载 (<link rel="preload">)
□ 文字使用 font-display: swap 或 optional
□ 首屏无 render-blocking JS/CSS
□ 服务端响应 TTFB < 800ms
□ LCP 不依赖客户端 JS 渲染 (SSR/SSG 直出)
□ 未使用动态注入首屏内容
```

### CLS (Cumulative Layout Shift) — 目标 < 0.1
```
□ 所有 <img> 有 width/height 或 aspect-ratio
□ 广告/嵌入内容有预留空间 (min-height)
□ 动态注入内容插入在现有内容下方 (非上方)
□ 字体加载期间使用 size-adjust 减少偏移
□ 无无尺寸的 iframe
□ 动画使用 transform 而非布局属性 (top/left)
□ 用户交互后 500ms 内的布局变更不计入 CLS
   (交互后的变更仍应避免)
```

### INP (Interaction to Next Paint) — 目标 < 200ms
```
□ 事件处理器不阻塞主线程 > 50ms
□ 长任务拆分 (yield to main thread / scheduler.yield)
□ 不强制同步布局 (读写分离)
□ 使用 Web Worker 处理重计算
□ debounce/throttle 高频事件 (scroll/resize/input)
□ useDeferredValue / startTransition 包裹非紧急更新
□ 合成层不爆炸 (<50 层)
```

---

## 五、兼容性检查

### iOS Safari
```
□ 测试 iOS Safari 最近 2 个大版本
□ 100vh 问题: 使用 dvh/svh 或 JS 计算
□ Safe Area: env(safe-area-inset-*) 适配
□ 弹性滚动: -webkit-overflow-scrolling: touch (旧版) / overflow-y: auto (现代)
□ 触摸事件: 无 300ms 延迟 (meta viewport + touch-action: manipulation)
□ backdrop-filter 性能 (低端设备考虑降级)
□ position: fixed 在软键盘弹出时的表现
□ 横竖屏切换重绘
```

### Android
```
□ 测试 Android Chrome (最近 2 个大版本)
□ 测试微信内置浏览器 (X5 WebView)
□ 软键盘弹出/收起导致视口变化 (window.visualViewport)
□ 系统字体缩放影响 (rem 相对 html font-size)
□ WebView postMessage 通信
□ 后退手势与路由冲突
```

### WKWebView (iOS App 内嵌)
```
□ IndexedDB 存储限制 (~50MB 总量)
□ WebSocket 断连重连 (WKWebView 不保活)
□ Cookie 同步 (WKHTTPCookieStore)
□ window.open 限制
□ Service Worker 不支持 (iOS < 16.4)
□ postMessage JSON 序列化性能
□ scroll bounce 效果控制
```

### 微信内置浏览器
```
□ WeixinJSBridge 可用性检测
□ 分享接口 (wx.config + wx.onMenuShareTimeline)
□ 缓存策略激进 (URL 加时间戳或 hash)
□ 长按菜单控制
□ 文件上传限制
```

---

## 六、可访问性检查 (A11y)

### WCAG 2.1 AA
```
□ 语义化 HTML: h1-h6 / nav / main / aside / article / section / header / footer
□ 所有交互元素可键盘访问 (Tab/Enter/Escape/Arrow Keys)
□ 焦点可见 :focus-visible 样式
□ 焦点顺序逻辑合理 (tabindex 仅在必要时使用)
□ 非文本内容有 alt 文本 (图片/图标)
□ 表单有 label (htmlFor 或 aria-label)
□ 色彩对比度 ≥ 4.5:1 (正文) / ≥ 3:1 (大字)
□ 不纯靠颜色传达信息
□ 错误提示有 aria-describedby 关联
□ 模态框打开时焦点锁定在框内
□ 模态框关闭后焦点返回触发元素
□ 屏幕朗读器可正确读出动态内容 (aria-live)
□ 页面有跳过导航的链接 (skip-to-content)
```

### ARIA 使用原则
```
规则 1: 尽可能用原生 HTML 代替 ARIA
规则 2: 不要用 ARIA 改变语义 (除非必须)
规则 3: 所有交互式 ARIA 控件必须可键盘操作
规则 4: 不要对可聚焦元素使用 role="presentation" / aria-hidden 清理
规则 5: 交互元素必须有可访问名称 (accessible name)
```

---

## 七、安全检查 (前端)

### XSS 防御
```
□ 用户输入永不使用 dangerouslySetInnerHTML / v-html
   如确实需要: 使用 DOMPurify 净化
   import DOMPurify from 'dompurify';
   const clean = DOMPurify.sanitize(dirty);
□ URL 使用 https:// (无 javascript: 协议)
□ 避免 eval / new Function / setTimeout(string)
```

### CSP (Content Security Policy)
```
□ 配置 Content-Security-Policy header
   起步配置:
   default-src 'self';
   script-src 'self';
   style-src 'self' 'unsafe-inline';
   img-src 'self' https: data:;
   connect-src 'self' https://api.example.com;
□ 尽量不要用 'unsafe-inline' 和 'unsafe-eval'
```

### 其他
```
□ 第三方脚本使用 SRI (Subresource Integrity)
   <script src="https://cdn.example.com/lib.js"
           integrity="sha384-..." crossorigin="anonymous"></script>
□ 敏感信息不暴露在前端代码中 (密钥/Token)
□ 环境变量使用 NEXT_PUBLIC_ 前缀明确标记 (Next.js)
□ 第三方依赖定期审计 (npm audit / Snyk)
□ CORS 配置严格 (Access-Control-Allow-Origin 不使用 *)
□ 登录/支付关键操作页面无第三方不可信脚本
```

---

## 八、AI 前端应用检查

### 流式响应
```
□ 使用 ReadableStream + SSE 或 WebSocket 处理流式 AI 响应
□ 解析 delta 增量 (OpenAI 格式: choices[0].delta.content)
□ 中断请求 (AbortController) 在组件卸载或用户取消时调用
□ 重试机制: 指数退避 (1s → 2s → 4s → 8s, max 30s)
□ 流式响应超时处理 (30s 无数据 → 提示用户重试)
□ 网络断连检测 (navigator.onLine + online/offline 事件)
```

### Token 感知
```
□ 使用 tiktoken 或等效库估算 token 数
□ 消息列表截断策略:
   选择: 保留系统提示 + 最近 N 轮对话 (不在中间截断)
   工具: 按 token 阈值自动裁剪, 保留语义完整性
□ Token 消耗展示 (用户可见的消耗条)
□ 接近上下文上限时提示用户 (如 "对话即将达到上限，建议开启新对话")
□ 工具调用结果压缩 (超过阈值的工具结果截断 + 摘要)
```

### UI/UX
```
□ 逐字/逐块显示流式内容 (打字机效果)
□ 生成中显示光标闪烁动画
□ 中断按钮始终可见且响应 < 200ms
□ 错误信息用户可理解 (非技术错误码)
□ 长响应分页或可折叠展示
□ 代码块正确高亮 (Prism/highlight.js/Shiki)
□ Markdown 安全渲染 (XSS 防护)
□ 数学公式正确渲染 (KaTeX/MathJax)
□ Mermaid 图表正确渲染
```

### 性能
```
□ 流式内容渲染节流: 每 16ms (一帧) 只更新一次 DOM
   实现: 使用 requestAnimationFrame 批量更新
□ 大消息列表使用虚拟列表 (react-window / vue-virtual-scroller)
□ 自动滚动到底部: IntersectionObserver 检测用户是否在底部
   如果用户在向上翻阅历史，不自动滚动
□ Token 计算放在 Web Worker 中 (避免阻塞主线程)
□ 对话历史使用 IndexedDB 本地持久化 (减少网络请求)
```

---

## 九、检查清单使用指南

### 何时执行
```
Pre-commit:  第一~三项 (每次提交前)
PR Review:   全部九项 (审查时逐项确认或豁免)
Release:     第四~七项 (发布前强制)
Architecture Review: 全部九项 + decisions.md (架构评审时)
```

### 豁免规则
```
每项检查允许豁免，但必须:
  1. 在 PR 中明确标记豁免项
  2. 说明豁免原因
  3. 设定豁免截止日期或条件
  4. 至少一位 Senior+ 审查者确认

示例:
  □ 无 console.log — 豁免: 错误上报需要 console.error 输出到监控平台
    截止: 接入 Sentry 后移除此条豁免
```
