# 跨端框架架构深度指南 (Cross-Platform Architecture)

> 跨端开发不是"写一套代码跑所有平台"的银弹，而是在复用率与平台体验之间做系统性权衡。
> 本文档覆盖 Taro / uni-app / React Native / Flutter / Electron 五大方案的架构深度，
> 以及从需求到方案的决策路径。

---

## 一、跨端方案全景对比

| 维度 | Taro 3.x | uni-app | React Native | Flutter | Electron |
|------|----------|---------|--------------|---------|----------|
| **渲染方式** | WebView (H5) / 静态编译 (小程序) / RN (App) | WebView / weex Native | Native UI (Bridge → JSI → Fabric) | Skia 自绘引擎 | Chromium 内核 |
| **开发语言** | React / Vue / (实验性 Solid/Svelte) | Vue 2/3 | JavaScript / TypeScript | Dart | JavaScript / TypeScript |
| **性能上限** | 中 (小程序受双线程架构限制) | 中 (weex 接近 Native，WebView 受限) | 高 (新架构接近 Native) | 高 (Skia 直接绘制) | 低 (每个窗口一个完整浏览器实例) |
| **包体大小** | H5: 0KB / 小程序: ~100KB / RN: ~500KB | H5: 0KB / App: ~3MB (weex) | ~2-5MB (Hermes ~1MB) | ~4MB (引擎) + 业务 | ~50-200MB (Chromium) |
| **生态规模** | 大 (微信/支付宝/字节/百度/QQ/京东/快手等小程序 + H5 + RN) | 大 (插件市场、HBuilderX IDE) | 大 (npm 生态、Expo) | 中 (pub.dev，Google 官方维护) | 大 (npm 全生态 + Node.js 原生能力) |
| **学习成本** | 低 (已有 React/Vue 基础) | 低 (Vue 开发者) | 中 (需理解 Native Bridge/JSI) | 中-高 (Dart 语言、Widget 思维) | 低 (Web 开发者) |
| **适用场景** | 小程序为主 + H5 + 部分 App | 快速交付多端 (H5/小程序/App) | 移动端高性能 App | 高性能跨平台 App (含桌面/Web) | 桌面端应用 |
| **热更新** | H5: 天然 | 小程序: 受限 (微信审核) | App: 受限 (Apple 审核) | ✅ CodePush / 自建 | ❌ App Store 禁止 | ✅ 自建/CodePush (需合规) | ✅ (替换 HTML/JS) |
| **原生能力** | 弱 (依赖小程序 API 或 RN 桥) | 中 (插件市场 bridge) | 强 (Native Modules / Turbo Modules) | 强 (Platform Channels / FFI) | 强 (Node.js 直接调用系统 API) |
| **调试体验** | 👍 浏览器 DevTools / 小程序开发者工具 | 👍 HBuilderX + Chrome DevTools | 👍 Flipper / React DevTools | 👍 Flutter DevTools | 👍 Chrome DevTools |
| **团队技能要求** | 前端为主 | 前端为主 | 前端 + 1-2 位 Native (iOS/Android) | 前端 + Dart 学习 | 前端 + Node.js |

---

## 二、Taro 架构深度

### 2.1 编译时 vs 运行时双模式

Taro 3.x 的核心突破在于**编译时(Compile-time)和运行时(Runtime)的分离设计**：

```
┌─────────────────────────────────────────────────────────┐
│                   Taro 3.x 架构                           │
│                                                          │
│  React/Vue 源码                                          │
│      │                                                   │
│      ├─── 编译时 ──→ Babel 插件链 ──→ AST 转换           │
│      │      • JSX → WXML template (小程序)               │
│      │      • 静态样式提取 → WXSS                         │
│      │      • 生命周期映射 (componentDidMount → onLoad)   │
│      │      • 平台条件编译 (#ifdef/#ifndef)               │
│      │                                                   │
│      └─── 运行时 ──→ 虚拟 DOM (Taro DOM Tree)            │
│             • 跨平台 reconcile 算法                       │
│             • 事件系统适配 (onClick → bindtap)            │
│             • 路由适配器                                   │
│             • API 适配器 (Taro.request → wx.request)     │
│                                                          │
│      └─── Platform Adapters ──→ 目标平台                  │
│              • @tarojs/plugin-platform-weapp              │
│              • @tarojs/plugin-platform-alipay             │
│              • @tarojs/plugin-platform-h5                 │
│              • @tarojs/plugin-platform-rn                 │
└─────────────────────────────────────────────────────────┘
```

**编译时策略**：将 React/Vue 组件编译为小程序 WXML 模板。静态部分直接转译，动态部分生成数据绑定 `{{ }}` 模板表达式。这避免了运行时虚拟 DOM 在小程序双线程架构下的性能瓶颈。

**运行时策略**：当编译时无法静态化时（如动态组件、高阶组件），回退到运行时虚拟 DOM reconcile。Taro 维护自己的 DOM 树 (`TaroElement`)，通过 `setData` 将 diff 结果同步到小程序渲染层。

### 2.2 关键编译产物差异 (易出错点)

```jsx
// 源码 (React)
function UserCard({ user }) {
  return (
    <View className="card">
      <Text>{user.name}</Text>
      {user.vip && <Text className="vip-badge">VIP</Text>}
    </View>
  );
}

// 编译产物 (微信小程序 WXML)
// <template name="tmpl_0">
//   <view class="card">
//     <text>{{i.cn[0].v}}</text>
//     <block wx:if="{{i.cn[1].v}}">
//       <text class="vip-badge">VIP</text>
//     </block>
//   </view>
// </template>
//
// DATA: { cn: [{ v: user.name }, { v: user.vip }] }
//
// ⚠️ 隐式约束:
// 1. WXML 模板不支持完整的 JS 表达式 → 编译时可能静默降级
// 2. 小程序 setData 有 256KB 单次限制 → 大数据列表可能截断
// 3. WXS (WeiXin Script) 不兼容 CommonJS → require 引入的库无法在 WXS 中使用
```

### 2.3 适配器模式 (Adapter Pattern)

Taro 3.x 使用**适配器模式**抽象平台差异：

```typescript
// @tarojs/runtime 中的适配器接口 (简化)
interface PlatformAdapter {
  // 渲染适配器
  render(child: TaroNode, container: TaroElement): void;
  // 事件适配器
  patchEvent(event: string): string; // onClick → bindtap
  // API 适配器
  request(options: RequestOptions): Promise<Response>;
  // 路由适配器
  navigateTo(options: NavigateOptions): Promise<void>;
  // 存储适配器
  setStorage(options: StorageOptions): Promise<void>;
}

// 微信小程序适配器
const WeappAdapter: PlatformAdapter = {
  render: (child, container) => {
    // 将 Taro Virtual DOM 编译为 WXML data
    // 通过 this.setData({ cn: serialize(child) }) 同步到视图
  },
  patchEvent: (event) => event.replace(/^on/, 'bind').toLowerCase(),
  request: (options) => wx.request(options),
  // ...
};
```

**适配器模式的架构价值**：
- 新增平台只需实现 `PlatformAdapter` 接口，不修改核心运行时
- 但每个适配器的质量取决于平台 API 的完整度 — 未覆盖的 API 将静默失败

### 2.4 跨端组件库设计模式

```typescript
// ❌ 反模式：直接使用平台特定属性
import { View } from '@tarojs/components';
<View onLongPress={handler} /> // 支付宝小程序不支持 onLongPress → 静默失败

// ✅ 正确模式：通过平台能力探测 + 降级
import Taro from '@tarojs/taro';

function LongPressable({ children, onLongPress, onTap }) {
  const [timer, setTimer] = useState(null);

  const handleTouchStart = () => {
    const t = setTimeout(() => {
      onLongPress?.();
    }, 500);
    setTimer(t);
  };

  const handleTouchEnd = () => {
    if (timer) { clearTimeout(timer); onTap?.(); }
  };

  // Taro.getEnv() 可在 H5 使用，小程序端编译时常量
  if (process.env.TARO_ENV === 'h5') {
    return <View onTouchStart={handleTouchStart} onTouchEnd={handleTouchEnd}>{children}</View>;
  }

  return <View onLongPress={onLongPress} onTap={handleTap}>{children}</View>;
}
```

---

## 三、uni-app 架构

### 3.1 渲染模式：WebView vs weex Native

uni-app 在不同平台使用不同的渲染引擎：

```
uni-app 源码 (.vue)
      │
      ├─── H5 ──→ uni-app 运行时 ──→ WebView DOM 渲染
      │
      ├─── 小程序 (微信/支付宝/字节/...) ──→ 编译为对应平台 WXML + JS
      │     • 与 Taro 编译时策略类似
      │     • 组件映射: <view> → <view>, <text> → <text>
      │
      ├─── App (Android) ──→ weex 渲染引擎 (Native)
      │     • .nvue 文件 → weex 组件 → Native View
      │     • app-vue 页面 → 内置 WebView
      │     • 同一 App 内可混用: Tab 页用 nvue (高性能), 普通页用 vue (高兼容)
      │
      └─── App (iOS) ──→ WKWebView (vue 页面) / weex (nvue 页面)
```

**weex vs WebView 选择**：
- **nvue (weex)**：纯原生渲染，性能接近 Native，但 CSS 支持受限 (仅 flex 布局, 不支持 grid)
- **vue (WebView)**：完整 CSS/CSS3 支持，开发效率高，但性能上限受限
- **关键架构陷阱**：nvue 和 vue 页面间的通信需要额外桥接层级

### 3.2 条件编译策略 (`#ifdef`)

uni-app 的条件编译是**编译时**指令，在构建阶段决定哪些代码保留/删除：

```vue
<template>
  <view>
    <!-- #ifdef H5 -->
    <web-socket-component />
    <!-- #endif -->

    <!-- #ifdef MP-WEIXIN -->
    <button open-type="getUserInfo">微信授权</button>
    <!-- #endif -->

    <!-- #ifndef APP-PLUS -->
    <web-only-feature />
    <!-- #endif -->

    <!-- #ifdef APP-PLUS || H5 -->
    <cross-platform-feature />
    <!-- #endif -->
  </view>
</template>

<script>
export default {
  methods: {
    share() {
      // #ifdef MP-WEIXIN
      wx.shareAppMessage({ title: '分享' })
      // #endif
      // #ifdef H5
      navigator.share?.({ title: '分享', url: location.href })
      // #endif
    }
  }
}
</script>

<style>
/* #ifdef MP-WEIXIN */
.button { color: #07c160; } /* 微信绿 */
/* #endif */
/* #ifdef H5 */
.button { color: #1677ff; } /* H5 蓝 */
/* #endif */
</style>
```

**条件编译架构限制**：
1. 只能判断**平台常量** (`MP-WEIXIN`, `H5`, `APP-PLUS`)，不能做运行时判断
2. 不支持 `#ifdef` 嵌套，复杂条件需要拆分
3. 样式中的 `#ifdef` 仅支持 `/* 注释形式 */`，不支持 `//` 和 HTML 注释
4. pages.json 中的路由配置也使用 `#ifdef`，但语法有细微差异

### 3.3 插件市场生态

uni-app 的插件市场 (ext.dcloud.net.cn) 是其核心竞争优势，但需要注意：

- **质量参差**：插件缺乏 CI 验证，跨平台兼容性声明可能不准确
- **版本锁定风险**：插件开发者可能停止维护，依赖其插件会导致技术负债
- **安全审计缺失**：插件可访问任意 Native API，需在集成前审查

---

## 四、React Native 工程化

### 4.1 Bridge → JSI → Fabric：通信架构演进

```
┌─────────────────────────────────────────────────────────────┐
│                  React Native 通信架构演进                     │
│                                                              │
│  旧架构 (Bridge):                                            │
│  ┌──────────┐  JSON序列化   ┌───────────┐  JSON反序列化 ┌─────┐
│  │ JS Thread │ ──────────→  │   Bridge   │ ──────────→ │Native│
│  │           │ ←────────── │  (异步队列) │ ←────────── │     │
│  └──────────┘  JSON反序列化 └───────────┘  JSON序列化  └─────┘
│  ⚠️ 三大瓶颈:                                                │
│  1. 所有通信异步 → 高频操作 (滚动/动画) 掉帧                  │
│  2. JSON 序列化开销 → 大数据 (图片/列表) 卡顿                 │
│  3. 单 Bridge 通道 → 不同模块竞争带宽                         │
│                                                              │
│  新架构 (JSI + Fabric + Turbo Modules):                      │
│  ┌──────────┐  JSI (C++ 直接调用)   ┌─────────────────┐      │
│  │ JS Thread │ ←──────────────────→ │ Native (C++)     │      │
│  │  (Hermes) │  无序列化/同步调用    │  Turbo Modules   │      │
│  └──────────┘                       │  Fabric Renderer │      │
│                                      └─────────────────┘      │
│  ✅ 优势:                                                     │
│  1. 同步/异步可选 → 动画/手势零延迟                            │
│  2. C++ 直接访问 JS 对象 → 无 JSON 序列化                     │
│  3. Turbo Modules 按需加载 → 启动时间优化                      │
│  4. Fabric 在 UI 线程直接渲染 → 滚动/动画流畅                  │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Hermes 引擎 vs JSC

| 维度 | Hermes | JavaScriptCore (JSC) |
|------|--------|---------------------|
| **启动时间** | 快 (预编译字节码) | 慢 (JIT 编译延后) |
| **内存占用** | 低 (~1MB, 无 JIT 开销) | 高 (~5MB+，JIT 需额外内存) |
| **执行峰值** | 中 (解释器为主) | 高 (JIT 优化热点代码) |
| **包体** | +~1MB (引擎) | 0 (系统自带) |
| **iOS 支持** | ✅ 0.64+ | ✅ 默认 |
| **Android 支持** | ✅ (默认从 0.70+) | ⚠️ (部分设备版本过旧) |
| **调试** | Chrome DevTools 协议 | Safari Web Inspector |

**选型建议**：
- 大部分场景用 Hermes (React Native 0.70+ 默认)
- 如果应用中有极端计算密集型操作 (如实时视频处理)，JSC JIT 可能更优
- JSC 在 iOS 上是唯一受 App Store 允许的 JS 引擎 (除了 Hermes 0.64+)

### 4.3 CodePush 热更新架构

```
┌──────────────────────────────────────────────────────┐
│                CodePush 热更新流程                      │
│                                                       │
│  1. 开发者发布更新                                     │
│     appcenter codepush release-react -d Production    │
│                                                       │
│  2. 客户端检查更新                                     │
│     codePush.sync({                                   │
│       installMode: codePush.InstallMode.IMMEDIATE,    │
│       mandatoryInstallMode:                           │
│         codePush.InstallMode.IMMEDIATE,               │
│     });                                               │
│                                                       │
│  3. 下载 → 解压 → 替换 JS Bundle                      │
│     ⚠️ iOS App Store 规则:                             │
│     • 只能更新 JS/资源，不能新增 Native Module          │
│     • 不能有"应用商店绕过"暗示                          │
│     • 热更新内容的功能不能超过原审核时的描述             │
│                                                       │
│  4. 关键架构防护点:                                    │
│     • 版本兼容检查: 新 bundle 引用的 Native Module     │
│       必须在当前 Native 版本中存在                      │
│     • 回滚机制: 保留上一版本 bundle，启动失败自动回退   │
│     • 签名校验: 防止中间人攻击篡改 bundle               │
└──────────────────────────────────────────────────────┘
```

### 4.4 FlatList 性能陷阱 (速查)

| 陷阱 | 现象 | 修复 |
|------|------|------|
| 缺 `keyExtractor` | 滚动闪烁/数据错位 | 必须提供稳定的 key (详见 pit-014) |
| `getItemLayout` 未提供 | 滚动卡顿/跳变 | 固定高度 item 必须声明 |
| 图片未使用 `resizeMode` | 列表首次渲染慢 | 预设 `cover`/`contain` |
| `onEndReached` 重复触发 | 请求风暴 | 加 `onEndReachedThreshold={0.5}` + 防抖 |
| `windowSize` 过大 | 内存暴增 | 默认 21 过大，调至 10 以下 |
| 嵌套 `ScrollView` | 滚动冲突 | 使用 `SectionList` 替代 |

---

## 五、Flutter Web 前端实践

### 5.1 三种渲染器对比

Flutter Web 提供三种渲染器，选择直接影响性能、兼容性和体感：

| 渲染器 | 原理 | 包体 (gzip) | 性能 | 兼容性 | 适用场景 |
|--------|------|------------|------|--------|---------|
| **CanvasKit** | Skia 编译为 WebAssembly，Canvas 2D 自绘 | ~1.5MB (wasm + js) | 高 (接近 GPU) | 现代浏览器 (Chrome/Edge/Firefox/Safari 15+) | 桌面端/高性能 Web 应用 |
| **HTML renderer** | 用 HTML + CSS + Canvas 组合模拟 Flutter 渲染 | ~500KB | 中 (DOM 操作开销) | 所有浏览器 | 移动端 Web / 低端设备 / 需要 SEO |
| **auto (默认)** | 移动端用 HTML，桌面端用 CanvasKit | 取决于自动选择 | 取决于自动选择 | — | 通用场景 |

```dart
// 渲染器选择
// 运行时指定:
// flutter run -d chrome --web-renderer html
// flutter build web --web-renderer canvaskit

// 代码内动态选择 (Flutter 3.10+):
import 'package:flutter/foundation.dart';
void main() {
  // 移动端浏览器默认 HTML，桌面端 CanvasKit
  runApp(MyApp());
}
```

### 5.2 Canvaskit 加载优化

CanvasKit 的最大问题是首屏加载：1.5MB 的 wasm 文件阻塞渲染。

```
优化策略:

1. 延迟加载 CanvasKit
   flutter build web --web-renderer canvaskit --dart2js-optimization=O3
   // 这会生成 canvaskit.js 而非内联，浏览器可缓存

2. 自定义加载指示器
   // index.html:
   <script>
     window.addEventListener('load', function() {
       // 在 canvaskit 加载前显示骨架屏
       _flutter.loader.loadEntrypoint({
         onEntrypointLoaded: async function(engineInitializer) {
           let appRunner = await engineInitializer.initializeEngine({
             useColorEmoji: false, // 减少 canvaskit wasm 大小 ~100KB
           });
           appRunner.runApp();
         }
       });
     });
   </script>

3. CDN 缓存策略
   • canvaskit.wasm: Cache-Control: public, max-age=31536000, immutable
   • main.dart.js: Cache-Control: public, max-age=3600 (业务代码，允许短期更新)

4. 使用 HTML renderer 作为降级
   // 检测 WebAssembly 是否可用
   if (typeof WebAssembly === 'undefined') {
     // 降级到 HTML renderer
   }
```

### 5.3 与现有 Web 项目共存

```
策略 1: iframe 嵌入 (简单但隔离性强)
  ┌──────────────────────┐
  │ Web App (React/Vue)  │
  │  ┌────────────────┐  │
  │  │ <iframe>       │  │
  │  │  Flutter Web   │  │
  │  │  App           │  │
  │  └────────────────┘  │
  └──────────────────────┘
  ⚠️ 通信依赖 postMessage

策略 2: Flutter Web 作为主框架 + WebView 嵌入 H5 (推荐)
  flutter_inappwebview 插件在 Flutter Web 中嵌入 WebView
  ⚠️ Web 端 flutter_inappwebview 不支持，需要平台判断

策略 3: 微前端混合 (高复杂度)
  使用 Module Federation 将 Flutter Web 和 H5 应用组合到同一页面
  ⚠️ 需要自定义通信层，且 Flutter 的 js-interop 有限制
```

---

## 六、选型决策树

```
启动项目
  │
  ├─── 目标平台?
  │     │
  │     ├── 仅小程序 + H5
  │     │     │
  │     │     ├── 团队 React ──→ Taro (React)   [推荐]
  │     │     ├── 团队 Vue ────→ uni-app         [推荐]
  │     │     └── 仅微信小程序 ──→ 原生 + 框架无关组件
  │     │
  │     ├── 仅移动端 (iOS + Android)
  │     │     │
  │     │     ├── 高性能需求 (动画/视频/游戏)
  │     │     │     ├── 团队愿意学 Dart ──→ Flutter  [推荐]
  │     │     │     └── 坚持 JS/TS ──→ React Native + 新架构 + Hermes
  │     │     │
  │     │     ├── 业务型 App (表单/列表/CRUD)
  │     │     │     ├── 现有 React 代码 ──→ React Native
  │     │     │     ├── 现有 Vue 代码 ────→ uni-app (nvue) / 考虑转 React
  │     │     │     └── 新项目 ──→ React Native (生态大) 或 Flutter (性能好)
  │     │     │
  │     │     └── 已有小程序代码复用 ──→ Taro (RN 模式) 或 uni-app (App 模式)
  │     │
  │     ├── 仅桌面端
  │     │     │
  │     │     ├── Web 技术栈 ──→ Electron  [推荐，生态大]
  │     │     ├── 需要轻量 ────→ Tauri (Rust, ~10MB)  ⚠️ 前端团队有学习成本
  │     │     └── 跨桌面+移动 ─→ Flutter (Windows/macOS/Linux/iOS/Android)
  │     │
  │     └── 全平台 (小程序 + H5 + App + 桌面)
  │           │
  │           ├── 可行但不推荐单一方案 (维护成本过高)
  │           ├── 推荐混合策略:
  │           │   • 小程序 + H5 ──→ Taro
  │           │   • 移动端 App ──→ React Native / Flutter
  │           │   • 桌面端 ──→ Electron / Tauri
  │           │   • 共享: 业务逻辑层 (纯 JS/TS, 无 UI 依赖)
  │           └── 共享逻辑层策略:
  │               ┌─────────────────┐
  │               │ @shared/business │ (纯 TypeScript)
  │               │ - models/types   │
  │               │ - utils/helpers  │
  │               │ - api-client     │
  │               │ - state-machine  │
  │               └────────┬────────┘
  │            ┌───────────┼───────────┐
  │       Taro(RN)     RN/Flutter   Electron
  │       (小程序+H5)   (移动端)     (桌面端)
  │
  └─── 性能要求?
        │
        ├── 60fps 滚动/动画 ──→ Flutter / RN 新架构(JS I+Fabric) / nvue
        ├── 一般流畅即可 ──→ Taro / uni-app (vue 模式) / Electron
        └── 极端性能 (游戏/AR) ──→ 原生开发

关键提醒:
• 跨端框架的"一套代码多端运行"是理论上限，实际项目中 10-30% 代码需要平台适配
• 不要同时使用两个跨端框架 (如 Taro + Flutter 覆盖不同场景) — 维护成本呈指数增长
• 共享逻辑层是跨端架构中最被低估的投资 — 它比 UI 复用带来的价值更高
• 小程序生态有强平台锁定效应：Taro/uni-app 的策略是"编译到各平台"，但微信小程序的
  独有 API (如 open-type、开放能力) 可能在其他平台不可用，需要降级处理
```

---

## 附录 A：关键术语对照

| 术语 | 说明 |
|------|------|
| **JSI** (JavaScript Interface) | RN 新架构中替代 Bridge 的 C++ 层，提供同步/异步调用能力 |
| **Fabric** | RN 新架构的渲染引擎，直接在 UI 线程渲染，替代旧 UIManager |
| **Turbo Modules** | RN 新架构的 Native Module 系统，按需加载，替代旧 Native Modules |
| **Hermes** | Meta 开发的轻量 JS 引擎，专为移动端优化 |
| **Skia** | Google 开源 2D 图形库，Flutter 底层渲染引擎 |
| **CanvasKit** | Skia 编译为 WebAssembly 的 Web 版本 |
| **weex** | Apache 开源跨平台移动框架，uni-app 在 App 端使用其 Native 渲染 |
| **WXML** | 微信小程序标记语言，类似 HTML |
| **WXS** | 微信小程序脚本语言，运行在渲染层，类似简化版 JS |
| **双线程架构** | 微信小程序架构：逻辑层 (JSCore) + 渲染层 (WebView)，通过 Native 桥通信 |
| **nvue** | uni-app 的原生渲染页面格式，使用 weex 渲染引擎 |
| **app-vue** | uni-app 的 WebView 渲染页面格式 |

## 附录 B：各方案真实包体积参考 (非压缩)

| 方案 | Hello World 包体 | 中型应用包体 | 主要组成 |
|------|-----------------|------------|---------|
| Taro (H5) | ~50KB (gzip) | ~200-500KB | React/Vue runtime + Taro runtime |
| Taro (小程序) | ~100KB | ~300-800KB | 编译产物 + Taro runtime |
| uni-app (H5) | ~100KB (gzip) | ~300-500KB | Vue runtime + uni-app runtime |
| uni-app (App-vue) | ~3MB | ~8-15MB | WebView + uni-app 引擎 |
| React Native (Hermes) | ~2.5MB (Android) | ~10-30MB | Hermes + RN 框架 + JS bundle |
| Flutter | ~4MB (引擎) | ~15-40MB (Android) | Flutter 引擎 + Dart AOT |
| Electron | ~50MB | ~120-200MB | Chromium (~50MB min) + Node.js + 业务代码 |

---

## 附录 C：调试工具速查

| 方案 | 工具 | 特点 |
|------|------|------|
| Taro | 微信开发者工具 CLI + Chrome DevTools | H5 用 DevTools；小程序用开发者工具 |
| uni-app | HBuilderX + Chrome DevTools | IDE 内置调试，支持真机同步 |
| React Native | Flipper / React DevTools / Metro | Flipper 可查看 Native 层日志、网络、布局 |
| Flutter | Flutter DevTools / Dart DevTools | Widget Inspector、性能分析、内存分析 |
| Electron | Chrome DevTools (BrowserWindow) | 完整 DevTools，包括 Node.js 调试 |
