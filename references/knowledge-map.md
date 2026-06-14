# 前端全链路知识域地图 (Knowledge Map)

> 本文档绘制了前端架构师必须精通的核心知识域及其内在联系。
> 每个节点的深度要求：能独立设计、排查、优化该层的系统问题。

---

## 一、核心关系总图

```
                              ┌─────────────────────────┐
                              │     工程化 & 部署        │
                              │  CI/CD | Docker | K8s   │
                              │  CDN | 灰度 | 回滚      │
                              └───────────┬─────────────┘
                                          │ 承载
          ┌───────────────────────────────┼───────────────────────────────┐
          │                               │                               │
    ┌─────▼─────┐  ┌────────────────▼──────────────┐  ┌─────────────▼──────┐
    │ 浏览器运行时│  │        框架 / 库层              │  │  Node.js 服务端    │
    │            │  │  ┌──────────┬──────────┐     │  │                    │
    │ 渲染管线   │  │  │  React   │   Vue     │     │  │  事件循环 (libuv)  │
    │ V8/引擎   │  │  │  Fiber   │  响应式   │     │  │  Stream 流式处理   │
    │ DOM/CSSOM │  │  │  RSC     │  Vapor   │     │  │  Worker Threads    │
    │ GPU合成   │  │  │  Next.js │  Nuxt    │     │  │  Cluster 进程管理  │
    └─────┬─────┘  │  └──────────┴──────────┘     │  └──────────┬─────────┘
          │        └──────────────────────────────┘             │
          │                                                     │
    ┌─────▼──────────────────────────────────────────────────────▼───────┐
    │                          网络 & 协议层                              │
    │  HTTP/1.1 → HTTP/2 (多路复用/Server Push) → HTTP/3 (QUIC/0-RTT)    │
    │  WebSocket (全双工) | SSE (服务端推送) | WebTransport (QUIC 流)     │
    │  DNS 预解析 | Preconnect | CDN 缓存策略 (Cache-Control/ETag)       │
    └────────────────────────────┬────────────────────────────────────────┘
                                 │
                         ┌───────▼───────┐
                         │   构建工具链    │
                         │  Webpack/Rspack│
                         │  Vite/Rolldown│
                         │  esbuild/SWC  │
                         │  Turbopack    │
                         └───────────────┘
```

**依赖流向**：网络 → 构建 → 框架 → 浏览器运行时 → 用户感知
**反向影响**：用户性能感知 驱动 构建优化 驱动 协议选择

---

## 二、浏览器渲染管线 (Browser Rendering Pipeline)

```
HTML ──→ DOM Tree ──┐
                     ├──→ Render Tree ──→ Layout (Reflow) ──→ Paint ──→ Composite
CSS  ──→ CSSOM ─────┘         │                │                  │           │
                               │                │                  │           │
                          display:none    width/height         color/bg        GPU
                          不参与树构建     触发重排             触发重绘        合成层
```

### 各阶段性能瓶颈

| 阶段 | 触发条件 | 代价 | 优化手段 |
|------|---------|------|---------|
| **Parse HTML** | 首次加载、innerHTML | 中等 | 流式传输、预加载 scanner |
| **Parse CSS** | @import、内联样式膨胀 | 低-中 | Critical CSS 内联、移除未使用 CSS |
| **Layout (Reflow)** | 修改几何属性(width/height/left/top/margin/padding/border)、读写 DOM 尺寸、窗口 resize | **极高** | `contain: layout`、读写分离(FastDOM)、`will-change`、transform 动画 |
| **Paint** | color/background/box-shadow/outline 变更 | 高 | `contain: paint`、避免大面积 repaint、GPU 光栅化 |
| **Composite** | transform/opacity 变更 | 低 | 独立合成层、避免层爆炸、隐式合成检测 |

### Layout 触发链（必须了如指掌）

```javascript
// 强制同步布局 (Forced Synchronous Layout) — 经典性能反模式
element.classList.add('wider');   // 标记样式失效
const width = element.offsetWidth; // 强制 Layout 同步计算！
element.classList.add('taller');   // 再次标记样式失效
const height = element.offsetHeight; // 再次强制 Layout！

// 正确做法：读写分离
const width = element.offsetWidth;  // 一次性读取所有需要的值
const height = element.offsetHeight;
element.classList.add('wider', 'taller'); // 统一写入
// 下一帧统一布局
```

### Composite（合成层）提升条件

浏览器会将满足以下条件的元素提升为独立合成层（GraphicsLayer）：
1. 3D transform: `transform: translate3d(0,0,0)` / `translateZ(0)`
2. `<video>` / `<canvas>` / `<iframe>` 元素
3. `will-change: transform` 或 `will-change: opacity`
4. CSS filter / backdrop-filter
5. CSS animation / transition 作用在 transform/opacity 上
6. 对 opacity/transform 应用了 animation/transition 的元素（即便没有 will-change）

**层爆炸警告**：过度提升合成层会导致 GPU 内存暴涨。iOS Safari 尤其敏感——低端设备合成层超过~50个开始掉帧。

---

## 三、JavaScript 运行时深层

### V8 引擎编译管线

```
源代码 ──→ [Parser] ──→ AST ──→ [Ignition: 解释器] ──→ 字节码执行
                                    │
                          ┌─────────┼─────────┐
                          ▼         ▼         ▼
                    [Sparkplug] [Maglev] [Turbofan]
                     快速编译    中级优化    深度优化
                     非优化代码  中等优化    高度优化机器码
```

| 编译器 | 定位 | 特点 |
|--------|------|------|
| **Ignition** | 基准解释器 | 所有代码的入口，生成字节码，收集类型反馈(Type Feedback) |
| **Sparkplug** | 快速非优化编译器 | 从字节码直接生成机器码，无重优化开销，适合短生命周期代码 |
| **Maglev** | 中级优化编译器 (2023+) | 介于 Sparkplug 和 Turbofan 之间，编译速度是 Turbofan 的 10 倍 |
| **Turbofan** | 深度优化编译器 | 基于类型反馈做激进优化(内联缓存/逃逸分析/标量替换)，如果不命中则去优化(Deoptimization) |

### Hidden Class（隐藏类）与 Inline Cache（内联缓存）

V8 优化对象属性访问的核心机制：

```javascript
// ✅ 好的：始终按相同顺序初始化，共享 Hidden Class
function Point(x, y) {
  this.x = x;  // HC0 → HC1 (新增 x)
  this.y = y;  // HC1 → HC2 (新增 y)
}
const p1 = new Point(1, 2);
const p2 = new Point(3, 4);
// p1 和 p2 共享 HC2，属性访问被 Turbofan 优化为单态内联缓存 (Monomorphic IC)

// ❌ 坏的：动态添加属性，破坏 Hidden Class 链
const p3 = new Point(5, 6);
p3.z = 7;  // p3 离开共享 HC，创建新的 HC3
// 属性访问退化为多态 (Polymorphic IC) → 更慢

// ❌ 更坏的：delete 彻底破坏 Hidden Class
delete p3.x; // 退化到字典模式 (Dictionary Mode) → 极慢
```

**实战规则**：
- 构造函数中初始化所有实例属性（包括 `null`）
- 不动态添加/删除属性
- 不混用不同顺序的初始化
- 性能敏感代码避免 `delete`，改用 `obj.x = null`

### 事件循环精确控制

```
┌─────────────────────────────────────────────────────────────┐
│ 宏观: 一轮事件循环                                          │
│                                                             │
│ 宏任务(Macrotask) ─→ 微任务(Microtask) 全部清空 ─→ rAF ─→  │
│ render ─→ rIC ─→ 下一个宏任务 ─→ ...                       │
│                                                             │
│ 执行顺序:                                                   │
│ 1. 取出一个宏任务执行 (script/setTimeout/setInterval/I/O)   │
│ 2. 清空所有微任务队列 (Promise.then/MutationObserver/       │
│    queueMicrotask) — 注意微任务中产生的新微任务也会在此阶段  │
│    清空，直到队列为空                                       │
│ 3. 执行 requestAnimationFrame 回调                          │
│ 4. 浏览器计算样式 → Layout → Paint → Composite(按需)        │
│ 5. 执行 requestIdleCallback 回调 (如果有剩余时间)           │
│ 6. 回到步骤 1                                               │
└─────────────────────────────────────────────────────────────┘
```

```javascript
// 经典面试题级陷阱 — 理解微任务递归
function loop() {
  Promise.resolve().then(loop); // 微任务递归！
}
loop();
// 页面永远无法渲染 — 微任务清空前不会进行 render

// 正确做法：让出渲染机会
function loop() {
  requestAnimationFrame(loop); // 每帧执行一次
}
```

### 内存管理（分代 GC）

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
新生代 (Young Generation / New Space)
  大小: 1-8MB
  算法: Scavenger (Cheney 半空间复制)
  存活两次晋升到老生代
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↓ 晋升 (Promotion)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
老生代 (Old Generation)
  算法: Mark-Compact + 增量标记
  并发标记(Concurrent Marking)
  并发清除(Concurrent Sweeping)
  惰性清除(Lazy Sweeping)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**泄漏检测实战**：
```javascript
// Chrome DevTools Memory 面板三步法
// 1. Take Heap Snapshot (Baseline)
// 2. 执行可疑操作, Take Heap Snapshot 2
// 3. 使用 Comparison 视图，按 Delta 排序查看新增对象
// 目标: 找到脱离 DOM 但仍在 JS 堆中被引用的对象

// 常见原因:
// - 忘记 clearInterval / removeEventListener
// - 闭包持有 DOM 引用
// - 全局变量 / window 上挂载
// - Detached DOM tree (JS 引用已移除的 DOM)
```

### Promise/A+ 规范细节

```javascript
// Promise 状态不可逆
const p = new Promise((resolve, reject) => {
  resolve('first');
  reject('second');  // 无效果! 状态已锁定
  resolve('third');  // 无效果!
});

// .then 始终返回新 Promise (链式调用)
Promise.resolve(1)
  .then(v => v + 1)   // 返回 2
  .then(v => { throw new Error('oops') }) // 返回 rejected
  .then(v => v + 3, err => 'caught')      // err handler返回 'caught' → resolved
  .then(v => console.log(v));             // 输出 'caught'

// Promise 构造函数中的错误需要 reject 或 catch 才能被外部感知
new Promise(() => { throw new Error('silent death'); });
// ⚠️ 无人捕获 — 但如果环境支持 unhandledRejection 事件，会被触发
```

---

## 四、React 内核

### Fiber 架构核心

```
┌────────────────────────────────────────────┐
│ Fiber 树结构 (双缓冲)                       │
│                                            │
│   current tree (屏幕上)  ←──→  workInProgress tree (内存中构建) │
│                                            │
│   渲染阶段 (Render Phase)                   │
│   ├─ 可中断 (Concurrent Mode)              │
│   ├─ beginWork: 向下遍历, diff 子节点      │
│   ├─ completeWork: 向上回溯, 处理副作用     │
│   └─ 产出 effectList (副作用链表)          │
│                                            │
│   提交阶段 (Commit Phase)                   │
│   ├─ 不可中断                               │
│   ├─ Before Mutation: getSnapshotBeforeUpdate │
│   ├─ Mutation: DOM 变更, 删除, 插入         │
│   └─ Layout: useLayoutEffect, componentDidMount │
│                                            │
│   优先级 (Lane 模型)                         │
│   SyncLane > InputContinuousLane > DefaultLane > IdleLane │
└────────────────────────────────────────────┘
```

### Hooks 实现原理与陷阱

```javascript
// Hooks 基于链表存储 — 调用顺序决定一切
// 每个 Fiber 节点维护一个 Hook 链表:
// fiber.memoizedState → hook1 (state) → hook2 (effect) → hook3 (ref) → null

// 闭包陷阱 (Stale Closure) — 最常见的 Hooks 问题
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const timer = setInterval(() => {
      // ❌ 闭包捕获的是首次渲染的 count (0)
      console.log(count);        // 永远是 0
      setCount(count + 1);      // 永远是 0 + 1 = 1
    }, 1000);
    return () => clearInterval(timer);
  }, []); // 空依赖 → effect 只在 mount 时执行

  // ✅ 解决方案1: 函数式更新
  // setCount(c => c + 1);

  // ✅ 解决方案2: useRef 保持最新引用
  // const countRef = useRef(count);
  // countRef.current = count;
  // 在 effect 中读 countRef.current
}
```

### React Compiler (React Forget)

```javascript
// React Compiler 自动处理 memo 和依赖
// 开发者无需手动 useMemo / useCallback / React.memo

// 之前需要手动优化:
function ExpensiveList({ items, filter }) {
  const filtered = useMemo(() =>
    items.filter(i => i.name.includes(filter)),
    [items, filter]
  );
  return filtered.map(i => <Item key={i.id} data={i} />);
}

// React Compiler 之后，直接写：
function ExpensiveList({ items, filter }) {
  const filtered = items.filter(i => i.name.includes(filter));
  return filtered.map(i => <Item key={i.id} data={i} />);
}
// 编译器自动推断 memo 边界和依赖
// 规则: 只能在组件/Hook 顶层使用，需要遵守 Rules of React
```

---

## 五、Vue 内核

### 响应式系统演进

```javascript
// Vue 2: defineProperty (有局限性)
// - 无法检测属性添加/删除 (需要 Vue.set/Vue.delete)
// - 无法检测数组索引和 length 修改
// - 初始化时递归遍历所有属性 (性能开销)

// Vue 3: Proxy
const state = reactive({
  nested: { deep: true },
  arr: [1, 2, 3]
});
// ✅ Proxy 可以拦截:
// - 属性添加: state.newProp = 'works'
// - 属性删除: delete state.nested
// - 数组索引: state.arr[0] = 99
// - has: 'key' in state
// - 懒代理: nested 对象只在被访问时才转为 proxy

// Effect Scope (Vue 3.2+)
// 精确控制响应式副作用生命周期
const scope = effectScope();
scope.run(() => {
  const doubled = computed(() => count.value * 2);
  watch(doubled, (v) => console.log(v));
  watchEffect(() => document.title = `${count.value}`);
});
// scope.stop() — 一次性清理所有内部 effect
```

### 编译优化

```
模板:
<div>
  <span>{{ static }}</span>
  <span>{{ dynamic }}</span>
</div>

编译产物 (简化):
createElementVNode("div", null, [
  // 静态节点提升到 render 函数外部，复用 VNode
  _hoisted_1,  // <span>static content</span>

  // 动态节点: 创建时直接标记需要 patch 的类型
  createElementVNode("span", null, dynamic, PatchFlags.TEXT)
  // PatchFlags = 1 (TEXT) → 只对比 textContent
])
```

### Vapor Mode（无 VDOM 模式）

```
Vapor Mode 核心理念: 编译时将模板静态分析，直接生成操作 DOM 的命令式代码
不创建 VNode，无 Diff 开销，极致运行时性能

当前状态(2025): 实验阶段，基于 Vue 3.6+
适用场景: 性能极度敏感的 H5 活动页、大量列表渲染
限制: 不支持 Transition/KeepAlive/Teleport 等特性
```

---

## 六、CSS 渲染引擎深度

### 层叠上下文 (Stacking Context) 完整规则

创建层叠上下文的全部条件：
```
1. 根层叠上下文: <html>
2. z-index != auto + 定位元素 (position: relative/absolute/fixed/sticky)
3. z-index != auto + flex/grid 子元素
4. opacity < 1
5. transform != none
6. filter != none / backdrop-filter != none
7. perspective != none
8. clip-path / mask / mask-image
9. isolation: isolate
10. will-change: 任意会创建层叠上下文的值
11. contain: layout | paint | strict | content
12. position: fixed (即使 z-index: auto)
```

**关键影响**：每个层叠上下文内是一个独立的 z-index 作用域。子元素无法通过 z-index 逃逸。

### Formatting Context 决策指南

| FC 类型 | 触发方式 | 行为特征 |
|---------|---------|---------|
| **BFC** | `overflow:hidden/auto`、`display:flow-root`、`float`、`position:absolute/fixed` | 独立布局区域、清除浮动、margin 不折叠 |
| **IFC** | `display:inline/inline-block`、文本节点 | 行内元素在行框(Line Box)中水平排列 |
| **Flex** | `display:flex/inline-flex` | 一维弹性布局、主轴/交叉轴、flex-basis 增长/收缩算法 |
| **Grid** | `display:grid/inline-grid` | 二维网格布局、显式/隐式网格、fr 单位分配算法 |

### GPU 合成层提升条件详解

```css
/* 显式提升：开发者明确告知浏览器 */
.will-composite {
  will-change: transform;    /* 最明确的信号 */
  transform: translateZ(0);  /* 3D hack，强制提升 */
}

/* 隐式提升：浏览器自动提升(可能意外发生) */
.implicit-composite {
  /* 如果此元素与一个在 compositing 层上元素重叠，可能被隐式提升 */
  /* 诊断方法: DevTools → Layers 面板 → 查看 Compositing Reasons */
}

/* ⚠️ backdrop-filter 的性能陷阱 */
.backdrop-blur {
  backdrop-filter: blur(10px);
  /* iOS Safari: 每个像素都要对下层做 GPU 采样 → 严重掉帧 */
  /* 降级: @supports not (backdrop-filter: blur()) { ... } */
}
```

### CSS Containment（隔离优化）

```css
.long-list-item {
  contain: layout style paint; /* content 是最激进的 */
  /* layout: 内部布局不影响外部，外部不影响内部 */
  /* style: counter/quote 等样式不影响外部 */
  /* paint: 裁剪到边框盒，内部绘制不会溢出 */
  /* size: 元素尺寸不依赖子元素 (需要指定尺寸) */
  /* content: layout + style + paint + size 的别名 */
}
```

---

## 七、构建工具链

### Webpack 完整构建流程

```
entry ─→ resolve (解析模块路径) ─→ load (loader 转换)
  ─→ parse (生成 AST) ─→ analyze (依赖分析)
  ─→ module graph (模块依赖图) ─→ chunk (代码块分割)
  ─→ optimize (Tree Shaking/压缩/作用域提升)
  ─→ hash (内容哈希) ─→ emit (输出文件)
```

**关键环节**：

1. **Resolve**: `resolve.alias` 和 `resolve.extensions` 影响模块查找速度
2. **Loader 管线**: `use: ['style-loader', 'css-loader', 'sass-loader']` 从右往左执行
3. **Module Federation**: 共享依赖、运行时加载远程模块

### Module Federation 原理

```
ContainerPlugin (暴露):
  生成 remoteEntry.js → 包含 shared scope 和暴露模块的异步加载逻辑

ContainerReferencePlugin (消费):
  在构建时标记远程模块引用 → 运行时通过 remoteEntry.js 加载

共享依赖 (Shared):
  shared: { react: { singleton: true, requiredVersion: '^18' } }
  如果版本匹配，共享同一个实例 → 避免 React 重复加载
  如果版本不匹配，fallback: 各自加载自己的版本
```

### Vite 内部机制

```
开发模式:
  1. 依赖预构建 (esbuild): node_modules → ESM → .vite/deps/
     原因: CommonJS → ESM 转换、合并碎片模块减少请求数
  2. 源码按需编译: 浏览器请求 /src/App.tsx 时才编译
     中间件拦截 → esbuild/SWC 转换 → 返回 ESM
  3. HMR: 精确到模块级别，通过 WebSocket 推送更新边界

构建模式 (Vite 5+ 默认 Rolldown, 旧版 Rollup):
  - Rollup/Rolldown 处理打包、Tree Shaking、代码分割
  - esbuild 处理压缩 (minify)
  - 不在开发模式中使用的插件需适配构建钩子
```

### Tree Shaking 原理

```javascript
// Webpack 条件:
// 1. ESM 语法 (import/export) — CJS 不可 tree shake
// 2. usedExports: true (optimization.usedExports)
// 3. sideEffects: false (package.json 或 module.rule.sideEffects)

// sideEffects 声明: 告知打包器哪些文件无副作用
// package.json:
{
  "sideEffects": [
    "*.css",       // CSS 有副作用(插入样式)
    "*.scss",
    "./src/polyfills.js"
  ]
}
// 其余文件可安全 Tree Shaking
```

---

## 八、Node.js 服务端

### 事件循环详细阶段（libuv）

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval 到期回调
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  系统操作(I/O)的延迟回调
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           poll            │  ⭐ 核心: 执行 I/O 回调/等待新 I/O
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           check           │  setImmediate 回调
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │      close callbacks      │  socket.on('close', ...)
│  └─────────────┬─────────────┘
│                │
└────────────────┘
  其中，process.nextTick 和微任务在每个阶段之间执行
```

### Stream 体系

```
Readable ──→ Transform ──→ Writable
   │              │              │
   ├─ flowing    ├─ Transform  ├─ 背压(backpressure)
   │  mode       │  Stream      │
   ├─ paused     ├─ PassThrough ├─ cork/uncork
   │  mode       │              │
   └─ read()     └─ pipeline()  └─ drain 事件
```

### Worker Threads vs Cluster

| 特性 | Worker Threads | Cluster (child_process) |
|------|---------------|------------------------|
| 内存 | 共享 SharedArrayBuffer | 独立内存空间 (IPC 通信) |
| 适合 | CPU 密集计算 | 多核网络服务 |
| 开销 | 较低 (线程) | 较高 (进程) |
| 通信 | postMessage + 结构化克隆 | IPC Channel |
| 典型场景 | 图片压缩、Token 计算 | HTTP 服务多核利用 |

---

## 九、网络协议

### HTTP 版本对比

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 (QUIC) |
|------|---------|--------|---------------|
| 传输层 | TCP | TCP | UDP (QUIC) |
| 多路复用 | 否(串行, 队头阻塞) | 是(流式) | 是(无队头阻塞) |
| 连接建立 | TCP 3次握手 + TLS 4次 | TCP + TLS 4次 | 0-RTT / 1-RTT |
| 头部压缩 | 无 | HPACK | QPACK |
| 服务端推送 | 无 | Server Push | 扩展中 |

### CDN 缓存策略

```nginx
# 静态资源缓存策略
location /static/ {
  # 强缓存: 带 hash 的资源永久缓存
  add_header Cache-Control "public, max-age=31536000, immutable";
}
location /index.html {
  # 协商缓存: 入口文件始终验证
  add_header Cache-Control "public, max-age=0, must-revalidate";
}

# ETag vs Last-Modified
# ETag: 内容哈希，更精确
# Last-Modified: 时间戳，秒级精度，CDN 边缘节点不一致时出问题
```

---

## 十、多终端特有行为

| 平台 | 关键限制 | 对策 |
|------|---------|------|
| **iOS Safari** | `100vh` 包含底部工具栏、弹性滚动(rubber band)、Safe Area | `dvh`/`svh`/`lvh` 单位、`env(safe-area-inset-bottom)`、`-webkit-overflow-scrolling` |
| **Android WebView** | 版本碎片(Chrome/系统WebView/厂商定制)、软键盘影响视口 | 兼容至 Chrome 69+、`window.visualViewport` API |
| **WKWebView** | WKUserContentController 消息通信、Cookie 同步、`window.open` 限制 | `webkit.messageHandlers` bridge、WKWebViewConfiguration、JSBridge |
| **微信内置浏览器** | JSAPI 限制、缓存策略、长按菜单、分享接口 | WeixinJSBridge、绕过缓存(url+timestamp) |
| **小程序** | 双线程架构(逻辑层/渲染层)、setData 大小限制 | 数据分片传输、Skyline 渲染引擎 |

---

## 十一、依赖关系速查

```
性能问题排障路径:
  用户反馈"页面卡"
  ├─ 渲染卡? → 渲染管线: Layout Thrashing? 合成层爆炸? 长任务?
  ├─ 交互卡? → 事件循环: 微任务阻塞? rAF 堆积?
  ├─ 数据卡? → 框架: 不必要的重渲染? 深层状态变更?
  └─ 加载卡? → 构建/网络: Bundle 大小? CDN 命中率? DNS 延迟?

内存泄漏排障路径:
  Heap Snapshot Comparison
  ├─ 闭包? → 检查 effect 清理函数
  ├─ 事件? → 检查 addEventListener/removeEventListener 配对
  ├─ DOM? → 检查 detached DOM tree
  └─ 缓存? → 检查 Map/Set/WeakMap 无界增长
```
