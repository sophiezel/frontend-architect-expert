# 微信小程序架构深度参考

> 蒸馏自微信开放文档、司徒正美等专家实践、一线生产项目踩坑经验。
> 覆盖双线程架构原理、性能优化体系、平台差异矩阵、架构决策树。

---

## 一、双线程架构深度

### 1.1 架构模型

```
┌─────────────────────────────────────────────────────┐
│                    微信客户端 (Native)                 │
│  ┌──────────────────┐    ┌──────────────────────┐   │
│  │   渲染层 (WebView) │    │   逻辑层 (JsCore)      │   │
│  │   - WXML 模板编译  │    │   - App/Page/Component │   │
│  │   - WXSS 样式计算  │    │   - API 调用           │   │
│  │   - 事件绑定/捕获  │    │   - 数据处理/网络请求   │   │
│  │   - 动画执行       │    │   - 定时器/task 队列    │   │
│  └────────┬─────────┘    └───────────┬──────────┘   │
│           │          JSBridge        │               │
│           └──────────────────────────┘               │
│  ┌─────────────────────────────────────────────────┐ │
│  │              Native 能力层                        │ │
│  │  - 网络 (wx.request)  - 存储 (wx.setStorage)      │ │
│  │  - 媒体 (wx.chooseImage)  - 设备 (wx.getSystemInfo)│ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

渲染层（WebView）和逻辑层（JsCore）运行在不同的线程/进程，通过 `JSBridge` 进行消息中转。**两个线程之间没有共享内存，所有数据通过 `evaluateJavascript` 字符串传递后被深拷贝**。

### 1.2 setData 通信机制

```js
// setData 的完整数据传输链路：
// Page/Component (逻辑层) → JSBridge → Native 序列化/中转 → WebView (渲染层)

// ❌ 高频 setData 导致阻塞
onPageScroll(e) {
  // 每次 onPageScroll 触发频率约 16ms（60fps）
  // 而 setData 从调用到渲染层接收最少需要 ~2ms（极小数据）~ 50ms（复杂数据）
  // 高频调用会导致调用队列堆积，渲染跟不上 → 页面卡顿
  this.setData({ scrollTop: e.scrollTop }); // 灾难
}

// ✅ 合并调用 + 去重
const scrollThrottled = throttle((e) => {
  this.setData({ scrollTop: e.scrollTop });
}, 100); // 100ms 节流

// ✅ 只传变化的数据
this.setData({
  'list[3].name': 'newName',  // 路径更新，只传变化的字段
});
```

**setData 数据量建议**：
| 数据量 | 耗时 | 建议 |
|--------|------|------|
| < 1KB | < 10ms | 安全 |
| 1KB ~ 10KB | 10ms ~ 50ms | 谨慎，需合并 |
| 10KB ~ 64KB | 50ms ~ 200ms | 危险，需分帧 |
| > 64KB | > 200ms | 致命，需要重构 |

```js
// 分帧 setData —— 大列表场景
function batchSetData(page, dataObj, chunkSize = 10) {
  const keys = Object.keys(dataObj);
  let index = 0;
  
  function nextChunk() {
    const chunk = {};
    const end = Math.min(index + chunkSize, keys.length);
    for (let i = index; i < end; i++) {
      chunk[keys[i]] = dataObj[keys[i]];
    }
    page.setData(chunk, () => {
      index = end;
      if (index < keys.length) {
        setTimeout(nextChunk, 16); // 下一帧
      }
    });
  }
  
  nextChunk();
}

// 使用
batchSetData(this, { /* 大量数据 */ }, 5);
```

### 1.3 自定义组件 Shadow DOM 模型

小程序自定义组件的样式隔离基于简化的 Shadow DOM：

```js
// Component 定义
Component({
  options: {
    styleIsolation: 'isolated',   // 完全隔离（默认）
    // 'apply-shared' — 页面样式影响组件，组件不影响页面
    // 'shared'       — 互相影响
    // 'page-isolated'— 页面隔离（页面不能影响组件）
    multipleSlots: true,          // 多 slot 支持
  },
  
  // 组件的 setData 只影响自身渲染树
  // 不会触发父组件或兄弟组件的重新渲染
  methods: {
    updateSelf() {
      this.setData({ count: this.data.count + 1 });
      // 仅该组件的 WXML 子树重渲染
      // 父/兄弟组件不受影响
    }
  }
});
```

**组件 setData 隔离原理**：每个自定义组件维护独立的 Virtual DOM 子树和渲染队列，`setData` 调用只触发该组件的 diff + patch 流程，不会影响兄弟组件。这对比页面级 `setData` 有巨大性能优势。

### 1.4 Worker 线程

```js
// app.json 配置
{
  "workers": "workers",
  // 可选：开启实验性 Worker（iOS 支持 JIT）
  "useExperimentalWorker": true
}

// workers/request/index.js — Worker 线程
worker.onMessage((msg) => {
  // 在独立 JsCore 线程执行
  const result = heavyComputation(msg.data);
  worker.postMessage({ result });
});

// 主线程（逻辑层）
const worker = wx.createWorker('workers/request/index.js');
worker.onMessage((msg) => {
  console.log('Worker result:', msg.result);
});
worker.postMessage({ data: largeArray });
```

**Worker 通信开销公式**：
```
总耗时 = 主线程序列化数据 + JSBridge 传输 + Worker 反序列化 + 计算 + Worker 序列化结果 + JSBridge 传输 + 主线程反序列化

// 实测数据 (iPhone 12, 阵乘 1000×1000):
// 不使用 Worker: 420ms
// 使用 Worker (无 JIT):  序列化 15ms + 传输 8ms + 计算 680ms + 反序列化 15ms = 718ms  ← 更慢!
// 使用 Worker (有 JIT):  序列化 15ms + 传输 8ms + 计算 210ms + 反序列化 15ms = 248ms  ← 快了
```

**关键限制**：
- 最多同时运行 1 个 Worker
- Worker 内无法调用 `wx.*` API
- 无 JIT 时纯计算在 Worker 中可能比主线程还慢
- 传输大数据时的序列化开销可能抵消所有计算收益

---

## 二、性能优化体系

### 2.1 启动优化

#### 分包加载

```json
// app.json
{
  "pages": [
    "pages/index/index",       // 主包页面
    "pages/logs/logs"
  ],
  "subPackages": [
    {
      "root": "packageA",
      "pages": ["pages/cat/cat", "pages/dog/dog"],
      // 独立分包：不依赖主包，可独立下载运行
      "independent": true
    },
    {
      "root": "packageB",
      "pages": ["pages/apple/apple"],
      // 普通分包：依赖主包
    }
  ],
  // 分包预下载配置
  "preloadRule": {
    "pages/index/index": {
      "network": "all",         // "wifi" | "all"
      "packages": ["packageB"]  // 进入 index 页时预下载 packageB
    }
  }
}
```

**分包大小限制**（单个分包限制）：
- 主包：≤ 2MB
- 单个分包：≤ 2MB
- 所有分包总计：≤ 20MB
- 独立分包：≤ 2MB（不计算在主包内）

#### 分包异步化

```json
// 组件级分包异步化 — 基础库 2.24.3+
{
  "usingComponents": {
    "heavy-component": "/packageB/components/heavy/index"
  },
  "componentPlaceholder": {
    "heavy-component": "loading-view"  // 占位组件
  }
}
```

```js
// 文件级分包异步化
// pages/index/index.js
Page({
  onLoad() {
    // 按需加载分包中的模块
    require.async('/packageB/utils/heavy-lib.js').then((mod) => {
      mod.process();
    });
  }
});
```

### 2.2 代码包优化

```json
// project.config.json — packOptions 忽略无用文件
{
  "packOptions": {
    "ignore": [
      { "type": "folder", "value": "docs" },
      { "type": "folder", "value": "scripts" },
      { "type": "file", "value": ".gitignore" },
      { "type": "glob", "value": "**/*.test.js" },
      { "type": "glob", "value": "**/*.spec.js" }
    ]
  }
}
```

```json
// app.json — 全局自定义组件按需注入（基础库 2.11.1+）
{
  "lazyCodeLoading": "requiredComponents"
  // "requiredComponents": 仅在使用时才注入自定义组件代码
  // 减少主包大小和非必需代码的执行
}
```

```js
// 静态依赖分析 — 避免这种模式
// ❌ 静态 import 把整个库打包
import _ from 'lodash';
_.debounce(fn, 300);

// ✅ 按需引入（如果库支持）
import debounce from 'lodash/debounce';

// ✅ 使用原生 API 替代
const debouncedFn = debounceByTimer(fn, 300);
```

### 2.3 渲染优化

```xml
<!-- ❌ 过多节点 -->
<view wx:for="{{list}}" wx:key="id">
  <view class="wrapper">
    <view class="inner">
      <view class="content">{{item.text}}</view>
    </view>
  </view>
</view>

<!-- ✅ 精简节点 -->
<view wx:for="{{list}}" wx:key="id" class="item">
  {{item.text}}
</view>
```

```js
// ❌ setData 传全量数据
this.setData({ form: this.data.form }); // 整个 form 对象重新传输

// ✅ 只传变化数据
this.setData({
  'form.name': 'newName',        // 路径更新
});

// ❌ 在频繁回调中 setData
onPageScroll(e) {
  this.setData({ scrollTop: e.scrollTop }); // 16ms 一次，灾难
}

// ✅ 用 WXS 在渲染层直接处理
// index.wxs
var getScrollClass = function(scrollTop) {
  return scrollTop > 100 ? 'fixed' : '';
};
module.exports = { getScrollClass: getScrollClass };
```

```xml
<!-- index.wxml — WXS 在渲染层执行，不经过 setData -->
<wxs module="util" src="./index.wxs"></wxs>
<view class="{{util.getScrollClass(scrollTop)}}">内容</view>
```

### 2.4 内存优化

| 策略 | 效果 | 代价 |
|------|------|------|
| 增加自定义组件 | 减少 setData 影响范围，提升渲染性能 | 每个组件约 200~500KB 内存 |
| 减少自定义组件 | 降低内存占用 | setData 需传更多数据，渲染更慢 |
| 页面 onUnload 清理 | 避免内存泄漏 | 需要开发规范保障 |

```js
// 页面卸载时清理
Page({
  onUnload() {
    // 清理定时器
    clearInterval(this._timer);
    clearTimeout(this._timer2);
    // 清理事件监听
    wx.offLocationChange(this._locationHandler);
    // 清理大对象引用
    this._largeData = null;
  }
});
```

---

## 三、小程序 vs Web 差异

### 3.1 WXML/WXSS 限制矩阵

| 特性 | Web (HTML/CSS) | 小程序 (WXML/WXSS) |
|------|---------------|---------------------|
| DOM API | `document.querySelector` 等 | 完全不可用 |
| window/document | 全局对象 | 不存在（报错 `Can't find variable: window`） |
| CSS 标签选择器 | 支持所有 | 不支持 `*`、部分属性选择器 |
| CSS `overflow:scroll` | 支持 | **不支持**，需用 `scroll-view` 组件 |
| `position: fixed` | 相对于视口 | 受 `page` 容器影响，需在真机验证 |
| `vh` 单位 | 支持 | 支持，但 iOS 底部安全区需额外处理 |
| `<canvas>` | HTML 元素 | `<canvas>` 组件，API 不同 |
| `<video>` | HTML 元素 | `<video>` 组件，行为有差异 |

### 3.2 网络限制

```js
// 小程序网络请求强制 HTTPS
wx.request({
  url: 'https://api.example.com/data', // 必须 https://
  // ❌ http:// 会直接失败，request:fail url not in domain list
});

// WebSocket 限制
wx.connectSocket({
  url: 'wss://ws.example.com',  // 必须 wss://
  // 最大并发连接数: 5 (基础库 2.23.1+)
  // 超时: 默认 60s，不可修改
});
```

### 3.3 JS 执行环境差异

```js
// ❌ 这些在小程序中不存在或行为不同
new Function('return this')();  // 不可用（非严格模式禁用）
eval('1 + 1');                   // 不可用
// 动态执行代码被禁止

// ❌ 部分 ES6+ API 需要基础库支持
// Promise.finally — 基础库 2.10.0+
// Array.prototype.flat — 基础库 2.13.0+
// 可选链 ?. — 基础库 2.23.0+
// 调用前检查基础库版本
if (wx.canIUse('promise.finally')) {
  // safe
}
```

---

## 四、跨平台小程序差异对比

### 4.1 API 差异矩阵

| 能力 | 微信 | 支付宝 | 字节(抖音) |
|------|------|--------|-----------|
| 网络请求 | `wx.request` | `my.request` | `tt.request` |
| 用户信息 | `wx.getUserProfile` | `my.getOpenUserInfo` | `tt.getUserInfo` |
| 支付 | `wx.requestPayment` | `my.tradePay` | `tt.pay` |
| 地图组件 | `<map>` | `<map>` (属性差异) | `<map>` 不支持部分属性 |
| 动画 | `wx.createAnimation` | `my.createAnimation` | 基本一致 |
| 蓝牙 | 完整 BLE 支持 | 完整 BLE 支持 | 部分支持 |
| 云开发 | 完整支持 | 不支持 | 不支持 |
| 插件 | 完整生态 | 有限 | 有限 |

### 4.2 组件差异

| 组件 | 微信 | 支付宝 | 字节(抖音) |
|------|------|--------|-----------|
| `scroll-view` | 完整支持 | 支持，属性名略不同 | 支持 |
| `rich-text` | 支持 | 支持，nodes 格式不同 | 支持 |
| `web-view` | 支持，需业务域名 | 不支持部分场景 | 有限支持 |
| `canvas` | 旧版 + 2D API | API 有差异 | 2D API 不完全 |

### 4.3 条件编译方案

```js
// Taro 跨平台编译
if (process.env.TARO_ENV === 'weapp') {
  // 微信小程序
} else if (process.env.TARO_ENV === 'alipay') {
  // 支付宝小程序
}

// uni-app 条件编译
// #ifdef MP-WEIXIN
console.log('微信');
// #endif
// #ifdef MP-ALIPAY
console.log('支付宝');
// #endif

// 原生小程序判断平台（通过环境变量/构建配置注入）
const platform = typeof wx !== 'undefined' ? 'wechat'
  : typeof my !== 'undefined' ? 'alipay'
  : typeof tt !== 'undefined' ? 'bytedance'
  : 'unknown';
```

---

## 五、架构决策

### 5.1 原生小程序 vs Taro/uni-app 决策树

```
需要同时发布到 3+ 个小程序平台？
  ├─ 是 → Taro 或 uni-app（跨平台成本 < 多套代码）
  │    ├─ React 技术栈 → Taro
  │    ├─ Vue 技术栈 → uni-app
  │    └─ 独立团队各自开发 → 原生
  │
  └─ 否 → 仅微信小程序？
       ├─ 是 → 原生小程序（性能最优，无框架开销）
       │    ├─ 团队有 React 背景 → 可考虑 Taro + React
       │    └─ 团队小程序经验丰富 → 原生
       │
       └─ 否 → 需同时支持 H5 + 小程序
            ├─ 组件复用需求高 → Taro
            └─ 各自独立维护可行 → H5 + 原生小程序
```

**选型对比**：

| 维度 | 原生小程序 | Taro | uni-app |
|------|-----------|------|---------|
| 性能 | ★★★★★ | ★★★★ | ★★★★ |
| 学习成本 | ★★★ | ★★★★（需学 React/Vue） | ★★★★ |
| 微信生态能力 | 即时支持 | 滞后 0.5~2 个月 | 滞后 0.5~2 个月 |
| 跨平台能力 | 无 | 微信/支付宝/字节/H5(RN) | 所有小程序 + App + H5 |
| 调试体验 | ★★★★★ | ★★★★ | ★★★★ |
| 包体积增量 | 0 | +30~80KB | +30~80KB |

### 5.2 分包策略决策树

```
项目复杂度？
  ├─ 简单（5 页以内）→ 不分包，保持单包简洁
  │
  ├─ 中等（5~20 页）
  │    ├─ 首页内容多 → 独立分包（首页快速启动）
  │    ├─ 首页轻量 → 普通分包（按功能模块拆分）
  │    └─ 有低频功能（如设置/关于）→ 独立分包
  │
  └─ 复杂（20+ 页）
       ├─ 首页必须极速 → 独立分包 + 预下载
       ├─ 核心流程 → 主包或普通分包 + 预下载
       ├─ 低频功能 → 独立分包
       └─ 大型第三方库 → 分包异步化按需加载
```

### 5.3 Worker 使用决策

```
耗时任务计算时长？
  ├─ < 50ms → 不要用 Worker（通信开销 > 计算收益）
  ├─ 50ms ~ 200ms → 视数据量决定
  │    ├─ 传输数据 < 1KB → 可以尝试 Worker
  │    └─ 传输数据 > 10KB → 不建议 Worker
  ├─ > 200ms
  │    ├─ 需要 JIT 环境 → useExperimentalWorker: true
  │    ├─ 不依赖 JIT → Worker 明确有益
  │    └─ Android 优先 → Worker 有 JIT，收益明显
  └─ > 500ms → 强烈建议 Worker 或迁移至服务端计算
```

```js
// Worker 使用决策检查清单
function shouldUseWorker(computation, dataSize) {
  // 1. 预估算
  const estimatedComputeTime = benchmark(computation);
  const serializationCost = dataSize * 0.002;  // 约 2μs/KB
  
  // 2. 决定
  if (estimatedComputeTime < 50) return false;  // 太短
  if (serializationCost > estimatedComputeTime * 0.5) return false; // 通信占主导
  
  // 3. 检查环境
  const systemInfo = wx.getSystemInfoSync();
  if (systemInfo.platform === 'ios' && !isExperimentalWorkerEnabled) {
    // iOS 无 JIT，Worker 内的 JS 执行可能比主线程慢 3~5x
    return estimatedComputeTime > 500;  // 仅超大计算才推荐
  }
  
  return true;
}
```

---

## 架构关键原则

1. **数据最小化原则**：每次 setData 只传变化字段，数据量控制在 1KB 以内
2. **组件化隔离原则**：用自定义组件天然隔离 setData 影响范围，避免页面级 setData
3. **WXS 前置原则**：渲染层能处理的计算（格式化、条件判断）尽量用 WXS，不经过逻辑层
4. **分包懒加载原则**：主包只放启动路径的必要代码，其余按功能分包
5. **平台渐进增强原则**：核心功能在所有平台一致，平台特有能力做渐增强
6. **Worker 谨慎原则**：仅在计算量 > 通信成本 × 2 时使用 Worker
