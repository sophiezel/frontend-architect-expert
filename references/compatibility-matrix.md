# 多终端兼容性矩阵 (Compatibility Matrix)

> 数据来源: caniuse-lite 数据 + 12 年实战踩坑记录。
> 兼容性标注: ✅ 完全支持 | ⚠️ 有条件支持/已知Bug | ❌ 不支持 | 🔸 前缀/部分支持

---

## 一、预设目标终端

| 终端 | 版本范围 | 内核 | 备注 |
|------|---------|------|------|
| **iOS Safari** | 16.0+ (最近2大版本) | WebKit | 实际用户分布 15.x ~ 18.x |
| **Android Chrome** | 100+ (最近2大版本) | Chromium (Blink) | |
| **Android 系统 WebView** | Chromium 90+ | Blink (不同版本) | 版本由系统决定，碎片化严重 |
| **WKWebView (iOS App 内嵌)** | iOS 16.0+ | WebKit | iOS Safari 同版内核但有限制 |
| **微信内置浏览器** | 最新版 | X5 (Android) / WebKit (iOS) | 实质是 WebView wrapper |
| **微信小程序** | 基础库 3.0+ | 双线程架构 (WebView + JSCore) | 非标准浏览器环境 |
| **Electron** | 28+ (Chromium 120+) | Chromium + Node.js | 版本可控 |
| **桌面浏览器** | Chrome/Firefox/Safari/Edge 最近2大版本 | — | 兼容性基准线 |

---

## 二、CSS 特性兼容性矩阵

| 特性 | iOS Safari | Android Chrome | WKWebView | 微信浏览器 | 微信小程序 | Electron | 降级方案 |
|------|-----------|----------------|-----------|-----------|-----------|----------|---------|
| **CSS Grid** | ✅ 10.3+ | ✅ | ✅ | ✅ (X5≥Blink57) | ⚠️ 部分支持 | ✅ | Flexbox fallback |
| **Flexbox gap** | ✅ 14.1+ | ✅ | ✅ 14.1+ | ✅ | ⚠️ 不支持 | ✅ | margin hack / CSS `gap` 在旧版用 `> * + *` 选择器模拟 |
| **position: sticky** | ✅ + -webkit- | ✅ | ✅ + -webkit- | ✅ | ⚠️ 仅垂直方向 | ✅ | 用 JS Intersection Observer + fixed 模拟 |
| **backdrop-filter** | ✅ 9.0+ | ✅ | ✅ 9.0+ | ⚠️ X5 性能差 | ❌ | ✅ | `@supports` 检测, 降级为 opacity 背景 |
| **scroll-snap** | ✅ 9.0+ | ✅ | ✅ 9.0+ | ⚠️ | ⚠️ | ✅ | JS 滚动库 (e.g., swiper) |
| **aspect-ratio** | ✅ 15.0+ | ✅ | ✅ 15.0+ | ⚠️ 旧版本不支持 | ⚠️ | ✅ | padding-top % hack |
| **:has() selector** | ✅ 15.4+ | ✅ 105+ | ✅ 15.4+ | ⚠️ 旧 X5 不支持 | ❌ | ✅ | JS class toggle |
| **Container Queries** | ✅ 16.0+ | ✅ 105+ | ✅ 16.0+ | ⚠️ | ❌ | ✅ | Media Query fallback |
| **View Transitions API** | ❌ | ✅ 111+ | ❌ | ❌ | ❌ | ✅ 111+ | CSS animations / Framer Motion |
| **CSS Nesting** | ✅ 16.4+ | ✅ 120+ | ⚠️ 16.4+ | ⚠️ | ❌ | ✅ 120+ | PostCSS nesting 预编译 |
| **text-wrap: balance** | ❌ | ✅ 114+ | ❌ | ❌ | ❌ | ✅ | 无 (progressive enhancement) |
| **scroll-driven animations** | ❌ | ✅ 115+ | ❌ | ❌ | ❌ | ✅ | Intersection Observer + CSS animation |
| **color-mix()** | ✅ 16.2+ | ✅ 111+ | ✅ 16.2+ | ⚠️ | ❌ | ✅ | PostCSS / Sass 预编译 |
| **@property** | ✅ 16.4+ | ✅ 85+ | ✅ 16.4+ | ⚠️ | ❌ | ✅ | 无 (progressive enhancement) |
| **subgrid** | ✅ 16.0+ | ✅ 117+ | ✅ 16.0+ | ⚠️ | ❌ | ✅ | 嵌套 Grid |

### 已知 CSS Bug 速查

```
Bug 1: iOS Safari position: fixed + 软键盘
  描述: 软键盘弹出时，fixed 元素不跟随视口上移 (iOS < 16)
  修复: 检测 visualViewport offset，用 JS 调整元素位置

Bug 2: iOS Safari 100vh 包含底部工具栏
  描述: 100vh 不等于可视区域高度
  修复: dvh (动态视口) / svh (小视口) 单位，或 JS 计算 window.innerHeight

Bug 3: Android Chrome backdrop-filter + border-radius
  描述: 某些版本子元素超出圆角
  修复: 父元素加 overflow: hidden; transform: translateZ(0);

Bug 4: WKWebView CSS animation 无限旋转时丢帧
  描述: 页面不可见 (进入后台) 时动画停止，回来时不自动恢复
  修复: visibilitychange 事件中重置 animation

Bug 5: 微信 X5 WebView flex 子元素 min-height 不生效
  描述: flex 容器的 min-height 被子元素忽略
  修复: 在 flex 子元素上加 height: 100% 或使用绝对定位替代
```

---

## 三、JavaScript API 兼容性矩阵

| API | iOS Safari | Android Chrome | WKWebView | 微信浏览器 | 微信小程序 | Electron | 降级/Polyfill |
|-----|-----------|----------------|-----------|-----------|-----------|----------|---------------|
| **IntersectionObserver** | ✅ 12.1+ | ✅ | ✅ 12.1+ | ✅ | ✅ | ✅ | scroll event fallback |
| **ResizeObserver** | ✅ 13.4+ | ✅ | ✅ 13.4+ | ✅ | ⚠️ 部分版本 | ✅ | ResizeObserver polyfill |
| **MutationObserver** | ✅ 6.0+ | ✅ | ✅ | ✅ | ✅ | ✅ | 几乎不需要 polyfill |
| **WebSocket** | ✅ | ✅ | ⚠️ (不保活) | ✅ | ✅ | ✅ | 心跳+重连机制 |
| **WebRTC** | ✅ 11.0+ | ✅ | ⚠️ (getUserMedia 可能弹出权限) | ⚠️ | ❌ | ✅ | |
| **Service Worker** | ✅ 11.3+ | ✅ | ⚠️ (iOS < 16.4 不支持) | ⚠️ | ❌ | ✅ | APP Cache (已淘汰) / 无 |
| **IndexedDB** | ✅ | ✅ | ⚠️ 存储上限 ~50MB | ✅ | ❌ (有 wx.setStorage) | ✅ | localStorage (仅小量数据) |
| **Web Worker** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | 主线程同步执行 |
| **SharedArrayBuffer** | ⚠️ (需 COOP/COEP) | ⚠️ (需 COOP/COEP) | ⚠️ | ⚠️ | ❌ | ✅ | postMessage + 结构化克隆 |
| **Broadcast Channel** | ✅ 15.4+ | ✅ | ✅ 15.4+ | ⚠️ | ❌ | ✅ | localStorage events hack |
| **AbortController** | ✅ 12.1+ | ✅ | ✅ 12.1+ | ✅ | ⚠️ | ✅ | |
| **Intl (国际化)** | ✅ 10.0+ | ✅ | ✅ 10.0+ | ✅ | ⚠️ (仅部分 locale) | ✅ | |
| **requestIdleCallback** | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | setTimeout fallback |
| **structuredClone** | ✅ 15.4+ | ✅ 98+ | ✅ 15.4+ | ⚠️ | ❌ | ✅ | JSON.parse(JSON.stringify()) / lodash cloneDeep |
| **URLPattern** | ⚠️ 16.4+ | ✅ 95+ | ⚠️ 16.4+ | ⚠️ | ❌ | ✅ | regex 手动解析 |

### 已知 JS Bug 速查

```
Bug 1: iOS Safari Date 解析
  描述: new Date('2024-01-01') 在 iOS Safari 中返回 Invalid Date
  原因: iOS Safari 不支持 ISO 8601 短格式 ('YYYY-MM-DD')
  修复: new Date('2024/01/01') 或使用 date-fns/dayjs

Bug 2: iOS Safari IndexedDB 存储空间用尽时静默失败
  描述: 超过 50MB 上限时事务不抛错，静默回滚
  修复: 写入后立即读取验证；大文件用文件系统 API 或服务端存储

Bug 3: WKWebView window.open 不触发
  描述: WKWebView 默认阻止 window.open
  修复: WKUIDelegate 的 createWebViewWithConfiguration 协议方法

Bug 4: Android WebView 软键盘 resize 视口
  描述: 系统 WebView 软键盘弹出可能挤压视口或遮盖内容
  修复: window.visualViewport API 动态调整布局

Bug 5: 微信 X5 WebView 的 Promise finally 在极旧版本不支持
  描述: 微信 Android < 7.0.15 使用的 X5 内核可能缺少 Promise.finally
  修复: core-js polyfill 或 ponyfill
```

---

## 四、图片/媒体格式兼容性

| 格式 | iOS Safari | Android Chrome | WKWebView | 微信浏览器 | 微信小程序 | Electron |
|------|-----------|----------------|-----------|-----------|-----------|----------|
| **WebP** | ✅ 14.0+ | ✅ | ✅ 14.0+ | ✅ | ✅ | ✅ |
| **AVIF** | ✅ 16.0+ | ✅ 85+ | ✅ 16.0+ | ⚠️ 依赖 X5 版本 | ⚠️ | ✅ |
| **SVG** | ✅ | ✅ | ✅ | ✅ | ⚠️ (仅部分 SVG 标签) | ✅ |
| **WebM** | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **HEVC/H.265** | ✅ | ⚠️ (取决于硬件) | ✅ | ❌ | ❌ | ⚠️ |
| **Lottie (JSON 动画)** | ✅ | ✅ | ✅ | ✅ | ⚠️ (需 lottie-miniprogram) | ✅ |
| **APNG** | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ✅ |

### 图片格式最佳实践

```html
<!-- 响应式图片 + 渐进式格式选择 -->
<picture>
  <!-- 优先 AVIF -->
  <source srcset="hero.avif" type="image/avif" />
  <!-- 降级 WebP -->
  <source srcset="hero.webp" type="image/webp" />
  <!-- 最终降级 JPEG/PNG -->
  <img
    src="hero.jpg"
    alt="Hero banner"
    width="1200"
    height="600"
    loading="lazy"
    decoding="async"
  />
</picture>
```

---

## 五、微信小程序特殊兼容性

### 架构差异

```
Web (标准):
  JS 线程 + 渲染线程 (同一进程, DOM 操作)
  └→ 可以直接操作 DOM

微信小程序 (双线程):
  逻辑层 (JSCore) ←→ setData (序列化) ←→ 渲染层 (WebView)
  └→ 不能操作 DOM
  └→ setData 有大小限制 (单次 < 256KB, 秒级总量 < 1MB)
  └→ 组件通信: triggerEvent / selectComponent
```

### 小程序特性矩阵

| 特性 | 支持情况 | 注意事项 |
|------|---------|---------|
| **CSS Grid** | ⚠️ 基础支持 | gap/auto-placement 可能异常 |
| **CSS Flexbox** | ✅ | 布局模式主要依赖 Flexbox |
| **rpx 单位** | ✅ | 1rpx = 屏幕宽度/750 |
| **@keyframes** | ✅ | 性能不如 transition |
| **position: fixed** | ⚠️ | 原生组件(如 video/map)层级更高，无法被覆盖 |
| **WebSocket** | ✅ | wx.connectSocket API |
| **WXS (WeiXin Script)** | ✅ | 在渲染层执行，用于响应式数据处理 |
| **Skyline 渲染引擎** | ⚠️ 新版支持 | 替代 WebView 的新渲染引擎，CSS 支持度更高 |
| **自定义 tabBar** | ✅ | 但覆盖原生的特殊处理 |
| **异步分包加载** | ✅ | require.async / 分包预下载 |

---

## 六、Electron 特定注意事项

### 与纯 Web 的主要区别

```
优势:
  □ 可控制 Chromium 版本 (不依赖用户浏览器)
  □ 可访问 Node.js API (fs/os/path/child_process)
  □ 可访问系统级 API (托盘/通知/全局快捷键)
  □ CSP 可完全控制
  □ 无跨域限制 (可通过主进程代理)

陷阱:
  □ 内存管理: 每个 BrowserWindow 独立进程, 内存占用是 Web 的 2-3x
  □ preload 脚本的 Context Bridge: contextBridge 暴露的 API 是副本, 非引用
  □ 安全: nodeIntegration: false (强制), contextIsolation: true (强制)
  □ 更新: 使用 electron-updater, 需处理各种平台的差分更新
  □ 签名: macOS 需要 Apple Developer 签名, Windows 需要代码签名证书
```

---

## 七、渐进式增强策略

```
分层加载策略:

第一层 (基础): 语义 HTML + 基本 CSS (所有终端支持)
  └→ 不依赖任何高级特性

第二层 (增强): CSS Grid/Flexbox/自定义属性 (95% 终端支持)
  └→ @supports 检测, 旧终端按需降级

第三层 (最优): Container Queries/View Transitions/子网格 (85% 终端支持)
  └→ 可选体验增强, 不影响核心功能

检测模式:
if ('CSS' in window && CSS.supports('display', 'grid')) {
  // 使用 Grid
} else {
  // 降级为 Flexbox/Float
}
```

### 降级方案原则

1. **核心功能绝不能依赖可选特性**：支付、下单、登录等关键路径不使用任何低于 95% 覆盖率的特性
2. **降级不是"完全不可用"**：降级方案保留核心功能，仅损失视觉增强
3. **使用 `@supports` 进行 CSS 特性检测**，而非 UA 嗅探
4. **使用 polyfill.io (或自建 polyfill 服务) 按需加载 polyfill**
5. **在 CI 中使用 caniuse-lite + browserslist 自动检查兼容性**
   ```javascript
   // postcss.config.js 中配置 browserslist
   // 自动添加 vendor prefix，但不会帮助检测 API 缺失
   // 建议: CI 中配合 eslint-plugin-compat
   ```

### 测试终端矩阵（推荐优先级）

```
P0 (每次发布必测):
  - iOS Safari (最新版)
  - Android Chrome (最新版)
  - 微信内置浏览器 (最新版)

P1 (每次发布必测, 可自动化):
  - iOS Safari (上一个大版本)
  - 桌面 Chrome/Firefox/Safari (最新版)

P2 (重大特性 + Release 前):
  - WKWebView
  - 微信小程序
  - Android 系统 WebView (低端设备)
  - iPadOS Safari

P3 (按需):
  - Electron (如果产品有桌面版)
  - 旧版本微信浏览器
  - 国产浏览器 (UC/QQ/百度)
```
