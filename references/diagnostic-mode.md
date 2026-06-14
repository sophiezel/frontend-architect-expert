# 分层诊断模式 (Diagnostic Mode)

> 当常规陷阱排查(参考 pitfalls/INDEX.md)全部耗尽后，进入此文件定义的分层深度诊断协议。
> 这是一个系统化的诊断框架，不是灵机一动的猜测。每一步都有可验证的假设和测量手段。

---

## 目录

1. [五层诊断框架](#一五层诊断框架)
2. [逐层诊断命令](#二逐层诊断命令)
3. [升级阶段 (Escalation Phases)](#三升级阶段-escalation-phases)
4. [架构级深度诊断](#四架构级深度诊断)
5. [诊断日志模板](#五诊断日志模板)

---

## 一、五层诊断框架

```
┌─────────────────────────────────────────────────────────────────┐
│                    问题输入 (Symptom + Context)                   │
│                      例如: 首屏白屏 3s / 点击无响应 / 内存持续增长  │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Layer 1      │   │  Layer 2      │   │  Layer 3      │
│  浏览器/运行时  │   │  框架/库      │   │  构建/部署     │
│  ──────────── │   │  ──────────── │   │  ──────────── │
│  渲染管线     │   │  组件模型     │   │  打包策略     │
│  事件循环     │   │  状态管理     │   │  代码分割     │
│  内存/GC      │   │  数据流       │   │  CDN/缓存     │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                ┌───────────┼───────────┐
                ▼                       ▼
        ┌───────────────┐       ┌───────────────┐
        │  Layer 4      │       │  Layer 5      │
        │  网络/协议     │       │  工程/团队     │
        │  ──────────── │       │  ──────────── │
        │  HTTP/2       │       │  模块边界     │
        │  流式传输     │       │  契约测试     │
        │  WebSocket    │       │  灰度策略     │
        │  SSE          │       │  监控告警     │
        └───────────────┘       └───────────────┘
```

**排查顺序**：从用户最近端开始(Layer 1)，逐层往下，直到定位根因。
**每次只怀疑一层**，排除后再进入下一层。不跳跃、不猜测。

### 层间关联关系

| 上层症状 | 常见下层根因 |
|---------|------------|
| L1: 首屏渲染慢 | L3: chunk 过大未拆分 / L4: CDN 回源慢 |
| L1: 交互卡顿 | L2: 不必要的 re-render / L1: 强制同步布局 |
| L2: 状态丢失 | L5: 灰度 release 新旧版本不兼容 |
| L3: 构建产物异常 | L5: 依赖版本锁冲突 |
| L4: 请求排队 | L2: 组件挂载时瀑布式请求 |
| L2: SSR 白屏 | L1: 水合失败 / L4: streaming 中断 |

---

## 二、逐层诊断命令

### Layer 1: 浏览器/运行时

#### 1.1 渲染管线诊断

```
目标: 确认帧率、Layout/Paint/Composite 瓶颈、强制同步布局
工具: Chrome DevTools Performance + Rendering 面板
```

**Performance 录制协议**：

```bash
# 使用 Chrome DevTools Protocol (CDP) 通过 Puppeteer 自动化 Performance 录制
node -e "
const puppeteer = require('puppeteer');
(async () => {
  const browser = await puppeteer.launch({ headless: 'new' });
  const page = await browser.newPage();
  // 启用 Performance 域
  const client = await page.target().createCDPSession();
  await client.send('Performance.enable', { timeDomain: 'threadTicks' });
  // 开始录制
  await page.tracing.start({ categories: ['devtools.timeline', 'disabled-by-default-devtools.timeline'] });
  await page.goto('http://localhost:3000/problem-page');
  await page.waitForTimeout(5000);
  const trace = await page.tracing.stop();
  require('fs').writeFileSync('trace.json', JSON.stringify(trace));
  console.log('Trace saved: trace.json (load in chrome://tracing or DevTools > Performance)');
  await browser.close();
})();
"
```

**强制同步布局检测** (Forced Synchronous Layout)：

```bash
# 在 Chrome DevTools Console 中运行以下检测脚本
# 应用 performance.measureUserAgentSpecificMemory() 前先打标签

# 1. 监控 Layout 抖动
let layoutThrashingDetected = false;
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name === 'layout-shift' && entry.value > 0.1) {
      console.warn('[Layout Shift]', entry);
      layoutThrashingDetected = true;
    }
  }
});
observer.observe({ type: 'layout-shift', buffered: true });

# 2. Long Task 检测 (阻塞主线程超过 50ms)
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`[Long Task] duration=${entry.duration}ms`, entry);
  }
});
longTaskObserver.observe({ type: 'longtask', buffered: true });
```

**DevTools Rendering 面板检查清单**：

```
在 Chrome DevTools → More tools → Rendering 中逐项开启：
□ Paint flashing      — 绿色闪烁区=重绘区域，越多越严重
□ Layout Shift Regions — 蓝色框=CLS 区域，页面加载后不应出现
□ Layer borders       — 橙色边框=独立合成层，过多说明层爆炸
□ FPS meter           — 实时帧率，保持 60fps，跌至 30 以下需排查
□ Core Web Vitals overlay — LCP/FID(INP)/CLS 实时值
```

#### 1.2 事件循环诊断

```
目标: 确认主线程是否被阻塞、微任务队列是否过长、requestAnimationFrame 是否正常调度
```

```javascript
// 事件循环阻塞检测 —— 在页面 Console 中注入
(function detectEventLoopBlockage() {
  let lastCheck = performance.now();
  let blockageCount = 0;
  
  function check() {
    const now = performance.now();
    const elapsed = now - lastCheck;
    // 如果两次检查间隔超过 100ms (正常应为 ~100ms±5ms)，说明主线程被阻塞
    if (elapsed > 150) {
      blockageCount++;
      console.error(`[Event Loop Blocked] gap=${elapsed.toFixed(2)}ms, count=${blockageCount}`);
      
      // 堆栈采样：当前正在执行什么
      console.trace('Blockage trace:');
      
      // 查询长任务 API
      performance.getEntriesByType('longtask').forEach(entry => {
        console.table({
          duration: entry.duration,
          startTime: entry.startTime,
          attribution: JSON.stringify(entry.attribution || 'N/A')
        });
      });
    }
    lastCheck = now;
    setTimeout(check, 100);
  }
  setTimeout(check, 100);
  console.log('[Event Loop Monitor] Started. Check Console for blockage alerts.');
})();
```

#### 1.3 内存与 GC 诊断

```
目标: 检测内存泄漏、确认 GC 回收是否正常、获取堆快照进行对比
```

```javascript
// 内存采样 —— 在 DevTools Console 中运行
// 步骤: 先执行 baseline，再执行可疑操作 ← target snapshot，对比两者

// Step 1: 记录基线
// DevTools > Memory > Take heap snapshot (手动)

// Step 2: 模拟用户操作 N 次 (如 10 次页面跳转/弹窗打开关闭)
// 然后再次 Take heap snapshot

// Step 3: 对比两个 snapshot
// 在第二个 snapshot 上选择 Comparison 视图，
// 按 Delta 降序排列，查找对象的 # 列持续增长

// 自动采样脚本:
let samples = [];
const sampleInterval = setInterval(() => {
  if (performance.memory) {
    samples.push({
      time: performance.now(),
      usedJSHeapSize: performance.memory.usedJSHeapSize,
      totalJSHeapSize: performance.memory.totalJSHeapSize,
      jsHeapSizeLimit: performance.memory.jsHeapSizeLimit
    });
    console.log(`[Memory] used=${(performance.memory.usedJSHeapSize/1048576).toFixed(1)}MB`);
  }
}, 1000);

// 停止采样并输出 CSV
// clearInterval(sampleInterval);
// console.log(samples.map(s => `${s.time},${s.usedJSHeapSize}`).join('\n'));
```

**Chrome DevTools Memory 三件套使用指南**：

| 工具 | 用途 | 何时使用 | 关键操作 |
|------|------|---------|---------|
| **Heap Snapshot** | 查看当前 JS 堆中所有对象 | 怀疑闭包泄漏、DOM 分离节点、事件监听器残留 | 取 baseline → 操作 → 取 target → Comparison 视图 |
| **Allocation instrumentation on timeline** | 记录对象分配的时间线 | 怀疑内存持续增长无法 GC | 录制操作过程，看蓝色柱(新分配)是否在操作结束后归零 |
| **Allocation sampling** | 按函数采样内存分配 | 定位哪个函数分配了最多内存 | 按分配大小排序，找到内存大户函数 |

---

### Layer 2: 框架/库

#### 2.1 React 组件诊断

```
目标: 确认不必要的 re-render、context 过度更新、memo 失效
```

```bash
# React DevTools Profiler 录制
# 1. 安装 React DevTools 浏览器扩展
# 2. 打开 DevTools → Profiler 标签
# 3. 点击录制按钮 (●)
# 4. 执行可疑交互
# 5. 停止录制
# 6. 查看 Flamegraph: 灰色组件=未渲染, 黄色/绿色=已渲染
#    关注 "Render duration" 和 "Why did this render?"
```

```javascript
// 运行时 re-render 检测 —— 注入到代码中
// 用于开发环境追踪哪些组件渲染次数异常

import { useEffect, useRef } from 'react';

export function useRenderCount(componentName) {
  const renderCount = useRef(0);
  useEffect(() => {
    renderCount.current += 1;
    if (renderCount.current > 10) {
      console.warn(
        `%c[Re-render Alert] %c${componentName} %crendered ${renderCount.current} times`,
        'color: red; font-weight: bold',
        'color: orange',
        'color: gray'
      );
    }
  });
  // 不打印每次渲染，避免日志刷屏
}

export function useWhyDidYouRender(componentName, props) {
  const prevProps = useRef(props);
  useEffect(() => {
    const changed = {};
    Object.keys(props).forEach(key => {
      if (prevProps.current[key] !== props[key]) {
        changed[key] = { prev: prevProps.current[key], next: props[key] };
      }
    });
    if (Object.keys(changed).length > 0) {
      console.group(`[Why render] ${componentName}`);
      console.table(changed);
      console.groupEnd();
    }
    prevProps.current = props;
  });
}
```

**React Context 性能排查**：

```javascript
// 检测 context value 是否每次渲染都创建新对象
// 这是 context 消费者过度渲染的主要原因

function ParentComponent() {
  // ❌ 错误: 每次渲染都创建新对象，所有消费者都重渲染
  // const value = { user, theme, permissions };

  // ✅ 正确: 使用 useMemo 稳定引用
  const value = useMemo(() => ({ user, theme, permissions }), [user, theme, permissions]);

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}
```

#### 2.2 Vue 组件诊断

```
目标: 确认响应式过度追踪、计算属性失效、组件更新范围不预期
```

```javascript
// Vue 3 响应式追踪 —— 在组件 setup 中注入
// 使用 onRenderTracked / onRenderTriggered 调试钩子

import { onRenderTracked, onRenderTriggered } from 'vue';

// 开发环境启用
if (import.meta.env.DEV) {
  onRenderTracked((e) => {
    console.log('[Vue Render Tracked]', {
      effect: e.effect,
      // 这个 effect 追踪了哪些响应式依赖
      // 依赖越多，越容易不必要地触发更新
    });
  });

  onRenderTriggered((e) => {
    console.log('[Vue Render Triggered]', {
      key: e.key,     // 哪个依赖变化触发了渲染
      oldValue: e.oldValue,
      newValue: e.newValue,
    });
  });
}
```

```bash
# Vue DevTools 使用
# 1. Timeline 标签 → 录制 → 操作 → 停止
#    看 Component render 事件: 哪些组件在渲染，耗时多少
# 2. 组件树中选择组件 → 右侧面板 "Dependencies"
#    可看到该组件追踪了哪些响应式数据
```

#### 2.3 状态管理层诊断

```
目标: 确认 store 更新频率、selector 粒度、immer 冻结开销
```

```javascript
// Redux store 更新监控
// 注入 Redux middleware 记录每次 dispatch 的耗时

const diagnosticsMiddleware = (store) => (next) => (action) => {
  const start = performance.now();
  const result = next(action);
  const end = performance.now();
  const duration = end - start;

  if (duration > 16) {
    // 超过一帧 (16ms) 标记为慢更新
    console.warn(
      `[Redux Slow Dispatch] action="${action.type}" duration=${duration.toFixed(2)}ms`
    );
  }
  return result;
};

// Zustand 选择器粒度检查
// ❌ 不好: 整个 store 对象变化触发更新
// const user = useStore(state => state);

// ✅ 好: 只订阅需要的字段
// const name = useStore(state => state.user.name);
```

---

### Layer 3: 构建/部署

#### 3.1 Bundle 分析诊断

```
目标: 确认 chunk 大小、重复依赖、tree-shaking 有效性
```

```bash
# Webpack Bundle Analyzer
npx webpack-bundle-analyzer dist/stats.json

# 生成 stats.json
npm run build -- --stats=detailed 2>&1 | tee build.log
# 或在 webpack.config.js 中启用:
# stats: 'verbose'

# Vite / Rollup
npx rollup-plugin-visualizer dist/stats.html

# 在 vite.config.js 中配置:
# import { visualizer } from 'rollup-plugin-visualizer';
# plugins: [visualizer({ open: true, gzipSize: true })]
```

**Bundle 大小回归检查**：

```bash
#!/bin/bash
# 对比两次构建的 bundle 大小变化
# 用法: bash compare-bundle.sh baseline-stats.json current-stats.json

BASELINE=$1
CURRENT=$2

echo "=== Bundle Size Comparison ==="
echo ""

# 解析 JSON 对比各 chunk (使用 jq)
jq -r '.assets[] | "\(.name)\t\(.size)"' "$BASELINE" | sort > /tmp/baseline.txt
jq -r '.assets[] | "\(.name)\t\(.size)"' "$CURRENT" | sort > /tmp/current.txt

# 显示新增/删除/大小变化
diff /tmp/baseline.txt /tmp/current.txt | while read line; do
  case "$line" in
    ">"*) echo "  [+] NEW: ${line#> }" ;;
    "<"*) echo "  [-] REMOVED: ${line#< }" ;;
  esac
done

# 显示大小变化前 10
echo ""
echo "Top 10 size changes:"
awk 'NR==FNR{a[$1]=$2;next}{diff=$2-a[$1]; if(diff!=0) printf "%+d  %s\n", diff, $1}' /tmp/baseline.txt /tmp/current.txt | sort -rn | head -10
```

**重复依赖检测**：

```bash
# 使用 depcheck 查找重复的依赖版本
npx depcheck

# 使用 npm ls 查看依赖树
npm ls some-package --depth=10

# 使用 yarn 查看为什么安装了某个包
yarn why some-package

# Webpack 重复模块检测 —— 在 webpack.config.js 中
# const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
# 查看 analyzer 报告中是否同一库出现多个版本

# 强制统一版本 —— package.json
# "overrides" (npm) 或 "resolutions" (yarn):
{
  "overrides": {
    "lodash": "4.17.21",
    "moment": "2.29.4"
  }
}
```

#### 3.2 Source Map 分析

```bash
# 使用 source-map-explorer 可视化 bundle 中各模块占比
npx source-map-explorer dist/**/*.js --html dist/source-map-report.html

# 针对单个文件分析
npx source-map-explorer dist/assets/index-abc123.js --html dist/report.html

# 无 source map 时的间接分析: 搜索 bundle 中的特征字符串
grep -o '\/\*[^*]*\*\/' dist/assets/index-abc123.js | sort | uniq -c | sort -rn | head -20
```

#### 3.3 CDN / 缓存诊断

```bash
# 检查 CDN 缓存命中情况
curl -sI https://cdn.example.com/assets/main-abc123.js | grep -E '^(cache|etag|age|x-cache|x-amz)'

# 期望结果:
# cache-control: public, max-age=31536000, immutable  ← 带 hash 的资源应为永久缓存
# etag: "abc123..."
# x-cache: Hit from cloudfront  ← CDN 命中
# age: 3600  ← 缓存已存活时间

# 快速检查所有关键资源的缓存头
URLS=(
  "https://cdn.example.com/assets/main-abc123.js"
  "https://cdn.example.com/assets/vendor-def456.js"
)
for url in "${URLS[@]}"; do
  echo "=== $url ==="
  curl -sI "$url" | grep -iE 'cache|etag|age' || echo "  No cache headers found!"
done
```

---

### Layer 4: 网络/协议

#### 4.1 HTTP/2 诊断

```bash
# 检查 HTTP/2 是否生效
curl -sI --http2 https://example.com | head -5
# HTTP/2 200  ← 确认返回 HTTP/2

# Chrome DevTools Network 面板
# 右键列头 → 勾选 Protocol
# 期望看到: h2 (HTTP/2) 或 h3 (HTTP/3)
# 如果出现 http/1.1，说明 HTTP/2 降级，检查:
#   - 是否使用了 HTTPS (HTTP/2 通常要求 TLS)
#   - 服务器配置是否正确

# 查看 TCP 连接数 (HTTP/2 多路复用应只有 1-2 个连接)
# DevTools > Network > 右键 > "Connection ID"
# 所有资源应共享同一 Connection ID
```

#### 4.2 请求瀑布分析

```bash
# 在 Chrome DevTools Network 面板中
# 1. 录制页面加载
# 2. 观察 waterfall 图:
#    - 大量灰色间隙 (Queueing) = 浏览器并发限制或 HTTP/1.1 连接数限制
#    - 长绿色条 (Waiting/TTFB) = 服务端处理慢
#    - 浅蓝长条 (Content Download) = 资源过大或网络慢
# 3. 导出 HAR 文件用于命令行分析

# HAR 分析脚本
node -e "
const har = require('./network.har');
const entries = har.log.entries;
const slowRequests = entries
  .map(e => ({
    url: e.request.url,
    ttfb: e.timings.wait,
    download: e.timings.receive,
    total: e.time,
    status: e.response.status
  }))
  .filter(e => e.total > 1000 || e.ttfb > 500)
  .sort((a, b) => b.total - a.total);
console.table(slowRequests.slice(0, 20));
"
```

#### 4.3 WebSocket / SSE 连接诊断

```javascript
// WebSocket 心跳监控
const ws = new WebSocket('wss://example.com/ws');
let lastPong = Date.now();

ws.addEventListener('message', (e) => {
  if (e.data === 'pong') {
    const latency = Date.now() - lastPong - 5000; // 心跳间隔 5s
    console.log(`[WS] Latency: ${latency}ms`);
    if (latency > 3000) {
      console.error('[WS] High latency detected, consider reconnect');
    }
  }
});

// 每 5s 发送心跳
setInterval(() => {
  lastPong = Date.now();
  if (ws.readyState === WebSocket.OPEN) {
    ws.send('ping');
  } else {
    console.warn(`[WS] Connection state: ${ws.readyState} (0=CONNECTING,1=OPEN,2=CLOSING,3=CLOSED)`);
  }
}, 5000);

// SSE 连接中断检测
const eventSource = new EventSource('/api/stream');
eventSource.addEventListener('error', (e) => {
  console.error('[SSE] Connection error', e);
  // readyState: 0=CONNECTING, 1=OPEN, 2=CLOSED
  console.log(`[SSE] readyState=${eventSource.readyState}`);
  // 浏览器会自动重连，但需要监控重连频率
});

let sseReconnectCount = 0;
eventSource.addEventListener('open', () => {
  sseReconnectCount++;
  if (sseReconnectCount > 3) {
    console.error(`[SSE] Reconnected ${sseReconnectCount} times, connection unstable`);
  }
});
```

---

### Layer 5: 工程/团队

#### 5.1 模块边界与契约

```
目标: 确认跨模块/跨服务调用的契约是否被打破
```

```bash
# 检查 API 契约变更
# 使用 OpenAPI/Swagger diff 工具
npx openapi-diff https://staging-api.example.com/openapi.json https://prod-api.example.com/openapi.json

# TypeScript 接口变更检测 (monorepo)
# 使用 @microsoft/api-extractor 或自定义脚本
npx api-extractor run

# 检查跨包的类型兼容性
npx tsc --project tsconfig.json --noEmit 2>&1 | grep "error TS"
```

#### 5.2 灰度发布与监控

```bash
# 灰度发布回滚检查清单
# 1. 确认当前灰度比例
curl -s https://your-monitoring-api/experiments/current | jq '.experiments[] | {name, percentage, version}'

# 2. 对比灰度组与对照组的 Core Web Vitals
curl -s https://your-monitoring-api/metrics?group=canary | jq '{lcp: .LCP.p75, inp: .INP.p75, cls: .CLS.p75}'
curl -s https://your-monitoring-api/metrics?group=baseline | jq '{lcp: .LCP.p75, inp: .INP.p75, cls: .CLS.p75}'

# 3. 查看错误率
curl -s https://your-monitoring-api/errors?group=canary&window=15m | jq '.errorRate'
```

---

## 三、升级阶段 (Escalation Phases)

当常规陷阱矩阵 (pitfalls/INDEX.md) 无法定位问题时，按以下阶段升级：

### Phase 1: 扩大搜索半径

```
目标: 将搜索范围从已知陷阱扩展到开源社区
```

**操作顺序**：

1. **GitHub Issues 关键词搜索**：

```
搜索模板:
  repo:facebook/react  "error message keyword"
  repo:vuejs/core      "symptom description"
  repo:vercel/next.js  "build failure pattern"
  repo:webpack/webpack "chunk loading error"

过滤条件:
  - sort:comments-desc  (最多讨论的)
  - sort:updated-desc   (最近活跃的)
  - is:issue is:open
  - label:bug,bug-report,needs-triage
```

```bash
# 使用 gh CLI 批量搜索
gh search issues --repo facebook/react "useEffect infinite loop" --limit 20 --json title,url,state,comments | jq '.[] | "\(.title)\n  \(.url) (\(.comments) comments)\n"'
gh search issues --repo webpack/webpack "code splitting chunk load failed" --limit 20 --json title,url,state,comments
gh search issues --repo vercel/next.js "hydration mismatch" --limit 20 --json title,url,state,comments
```

2. **StackOverflow 搜索**：

```
英文关键词策略:
  - 剥离业务词汇，保留技术核心词
  - 使用错误消息原文 (不加引号)
  - [reactjs] + "specific error"  ← 用方括号限定标签

示例:
  [reactjs] useState stale closure
  [next.js] middleware not running on production
  [webpack] dynamic import chunk failed
  [vite] HMR not working after file change
  [css] z-index stacking context not working

搜索技巧:
  - is:answered (只看有已采纳答案的)
  - score:>=3  (高质量问答)
  - created:>=2023-01-01 (最近的问题，框架版本匹配)
```

### Phase 2: 通用修复尝试

```
目标: 执行低风险、快速可回退的通用修复
注意: 这些不是猜测，而是基于排除法的系统性尝试
```

```bash
# ─── 步骤 1: 清除所有缓存 ───
rm -rf node_modules
rm -rf .next dist build out
rm -rf .cache .turbo .parcel-cache
npm cache clean --force
# 或 yarn cache clean
# 或 pnpm store prune

# 重新安装 (使用 lockfile 精确版本)
npm ci
# 或 yarn install --frozen-lockfile
# 或 pnpm install --frozen-lockfile

# 重新构建
npm run build

# ─── 步骤 2: 依赖版本回滚 ───
# 如果 build 失败发生在依赖升级后
git log --oneline -20 package.json yarn.lock
# 找到最近一次成功的构建 commit
npm install some-suspect-package@last-known-good-version

# 锁定所有间接依赖版本
# 检查 lockfile 是否被破坏
git diff yarn.lock  # 查看意外变更

# ─── 步骤 3: Node.js 版本检查 ───
node -v
npm -v
# 与 CI/生产环境对比，确认 Node 版本一致
# 使用 nvm 切换版本验证
nvm use 18   # 切换到 LTS 测试
npm run build
nvm use 20   # 切换到当前 LTS 测试
npm run build
```

### Phase 3: 环境差异矩阵

```
目标: 通过系统化的环境对比，定位环境特异性的问题
```

**环境差异矩阵模板** (请填入实际值):

| 维度 | 开发环境 (正常) | 问题环境 | 差异 |
|------|---------------|---------|------|
| **Node.js 版本** | `node -v` → | `node -v` → | |
| **包管理器版本** | `npm -v` → | `npm -v` → | |
| **操作系统** | `uname -a` → | `uname -a` → | |
| **浏览器** | Chrome 版本 → | Chrome 版本 → | |
| **关键依赖版本** | React/Vue/Next.js 版本 | ← 对比 | |
| **环境变量** | `env \| sort` | `env \| sort` | diff |
| **磁盘空间** | `df -h` → | `df -h` → | |
| **内存** | `free -m` → | `free -m` → | |
| **网络环境** | 公司/VPN/家 | 公司/VPN/家 | |
| **DNS 解析** | `nslookup` → | `nslookup` → | |
| **构建缓存** | `.cache` 存在? | `.cache` 存在? | |
| **node_modules** | `ls node_modules \| wc -l` → | 对比 → | |

```bash
#!/bin/bash
# 生成环境快照报告
# 用法: bash env-snapshot.sh > env-report.txt

echo "=== Environment Snapshot ==="
echo "Timestamp: $(date -Iseconds)"
echo ""

echo "--- System ---"
echo "OS: $(uname -a)"
echo "Shell: $SHELL"
echo ""

echo "--- Runtime ---"
echo "Node: $(node -v 2>/dev/null || echo 'NOT INSTALLED')"
echo "npm:  $(npm -v 2>/dev/null || echo 'NOT INSTALLED')"
echo "yarn: $(yarn -v 2>/dev/null || echo 'NOT INSTALLED')"
echo "pnpm: $(pnpm -v 2>/dev/null || echo 'NOT INSTALLED')"
echo ""

echo "--- Project Dependencies (from package.json) ---"
node -e "
const pkg = require('./package.json');
['dependencies','devDependencies','peerDependencies'].forEach(field => {
  if(pkg[field]) {
    console.log('[' + field + ']');
    Object.entries(pkg[field]).forEach(([k,v]) => console.log('  ' + k + ': ' + v));
  }
});
" 2>/dev/null || echo "No package.json found"

echo ""
echo "--- Lockfile ---"
echo "npm lock: $( [ -f package-lock.json ] && echo 'YES' || echo 'NO' )"
echo "yarn lock: $( [ -f yarn.lock ] && echo 'YES' || echo 'NO' )"
echo "pnpm lock: $( [ -f pnpm-lock.yaml ] && echo 'YES' || echo 'NO' )"

echo ""
echo "--- Disk ---"
df -h . | tail -1
echo ""
echo "--- Git ---"
git log --oneline -1 2>/dev/null || echo "Not a git repo"
echo "Branch: $(git branch --show-current 2>/dev/null || echo 'N/A')"
echo "Dirty: $(git status --porcelain 2>/dev/null | wc -l | tr -d ' ') files modified"
```

### Phase 4: Bug Report 生成

```
目标: 生成结构化的 bug report，可直接提交给框架维护者或团队
```

**Bug Report 模板**：

```markdown
## Bug Report: [简要描述]

### 环境信息
<!-- 粘贴 Phase 3 的环境快照 -->

### 复现步骤
1. 
2. 
3. 

### 预期行为


### 实际行为


### 最小复现仓库
<!-- 链接到 CodeSandbox / StackBlitz / GitHub 最小复现 -->

### 相关日志 / 截图
<!-- ```
粘贴终端输出、浏览器 Console、Network 面板截图
``` -->

### 诊断数据
<!-- 附加以下诊断结果 -->

**Layer 1 - 浏览器/运行时**:
- Performance 录制: [链接/截图]
- 内存采样: 基线=___MB → 操作后=___MB (Δ___MB)
- Long Task: [有/无]，最长阻塞 ___ms

**Layer 2 - 框架/库**:
- Re-render 计数: 组件 ___ 渲染了 ___ 次
- Warning 数量: ___ 条

**Layer 3 - 构建/部署**:
- Bundle 大小: 基线 ___KB → 当前 ___KB (Δ___KB)
- Chunk 数量: 基线 ___ → 当前 ___ (Δ___)

**Layer 4 - 网络/协议**:
- TTFB: ___ms
- LCP: ___ms
- 关键请求耗时: ___

**Layer 5 - 工程/团队**:
- 灰度比例: ___%
- 灰度组 vs 对照组差异: ___

### 已尝试的修复
<!-- 按顺序列出 Phase 2 中已执行的通用修复 -->
- [ ] 清除所有缓存 (`rm -rf node_modules .next` + `npm ci`)
- [ ] 依赖版本回滚
- [ ] Node.js 版本切换
- [ ] 浏览器无痕模式测试
- [ ] 禁用浏览器扩展测试

### 相关 Issues
<!-- GitHub Issues / StackOverflow 链接 -->
```

---

## 四、架构级深度诊断

### 4.1 性能预算诊断 (Performance Budget)

```
目标: 量化性能目标，用数据判断问题严重程度
```

**Core Web Vitals 阈值**：

| 指标 | Good (绿色) | Needs Improvement (橙色) | Poor (红色) |
|------|------------|------------------------|------------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |
| **TTFB** (Time to First Byte) | ≤ 800ms | ≤ 1800ms | > 1800ms |
| **FCP** (First Contentful Paint) | ≤ 1.8s | ≤ 3.0s | > 3.0s |

**性能预算设置 (Lighthouse / webpack)**：

```javascript
// webpack 性能预算配置 (webpack.config.js)
module.exports = {
  performance: {
    maxAssetSize: 250 * 1024,      // 单个资源最大 250KB
    maxEntrypointSize: 500 * 1024, // 入口点最大 500KB
    hints: 'error',                // 超出时构建失败
    assetFilter: (assetFilename) => {
      // 排除 .map 和 .css 文件
      return !(/\.map$/.test(assetFilename)) && !(/\.css$/.test(assetFilename));
    },
  },
};
```

**Lighthouse CI 性能预算** (lighthouserc.js)：

```javascript
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/'],
      numberOfRuns: 3,
    },
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3500 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'first-contentful-paint': ['error', { maxNumericValue: 1800 }],
        'speed-index': ['error', { maxNumericValue: 3400 }],
      },
    },
  },
};
```

**命令行性能测量**：

```bash
# Lighthouse CLI
npx lighthouse http://localhost:3000 --preset=desktop --output=html --output-path=./lighthouse-report.html

# 批量测量多个页面
for url in "/" "/products" "/about" "/contact"; do
  echo "Measuring: $url"
  npx lighthouse "http://localhost:3000${url}" --output=json --output-path="./lh-$(echo $url | tr '/' '-').json" --chrome-flags="--headless" --only-categories=performance
done

# 使用 sitespeed.io 进行深度分析
npx sitespeed.io http://localhost:3000 --browsertime.iterations 3

# Chrome DevTools Protocol 性能追踪 (编程式)
# 使用 Puppeteer 录制 trace 并用 Lighthouse 分析
node -e "
const puppeteer = require('puppeteer');
(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  // 启用性能观察者收集 Core Web Vitals
  await page.evaluateOnNewDocument(() => {
    // 注册 LCP 观察者
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      window.__LCP = lastEntry.startTime;
      console.log('[LCP]', lastEntry.startTime, 'ms');
    }).observe({ type: 'largest-contentful-paint', buffered: true });
    
    // 注册 CLS 观察者
    let clsValue = 0;
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!entry.hadRecentInput) clsValue += entry.value;
      }
      window.__CLS = clsValue;
      console.log('[CLS]', clsValue);
    }).observe({ type: 'layout-shift', buffered: true });
    
    // 注册 INP 观察者 (First Input Delay 的替代)
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        console.log('[INP candidate]', entry.duration, 'ms, type=', entry.name);
      }
    }).observe({ type: 'first-input', buffered: true });
  });
  
  await page.goto('http://localhost:3000');
  await page.waitForTimeout(5000);
  
  const metrics = await page.evaluate(() => ({
    LCP: window.__LCP,
    CLS: window.__CLS,
    FCP: performance.getEntriesByType('paint').find(e => e.name === 'first-contentful-paint')?.startTime,
  }));
  
  console.log('=== Core Web Vitals ===');
  console.table(metrics);
  
  await browser.close();
})();
"
```

### 4.2 内存泄漏诊断 (Memory Leak)

```
目标: 确认是否存在内存泄漏，定位泄漏源
```

**堆快照对比流程**：

```
步骤:
  1. 打开 DevTools > Memory > 选择 "Heap snapshot"
  2. 初始状态 → Take snapshot #1 (baseline)
  3. 执行可疑操作 N 次 (如: 打开关闭 Modal 20 次)
  4. 操作完成 → Take snapshot #2 (after actions)
  5. 手动触发 GC (DevTools > Memory > 垃圾桶图标)
  6. 等待 3 秒 → Take snapshot #3 (after GC)
  7. 在 snapshot #3 上选择 Comparison 视图，对比 #1

关键指标:
  - snapshot #3 与 #1 的 Δ 应为 0 或接近 0
  - 如果 #3 的 size delta > 5MB，存在泄漏
  - 按 Delta 降序，查看哪些对象/构造函数持续增长
  - 重点关注: Detached DOM tree, closure, EventListener
```

**分配时间线 (Allocation Timeline)**：

```javascript
// 使用 Puppeteer 自动化内存泄漏检测
// 适用于 CI 环境

const puppeteer = require('puppeteer');

async function detectMemoryLeak(url, scenario, iterations = 20) {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  const client = await page.target().createCDPSession();

  await client.send('HeapProfiler.enable');
  const baselineMetrics = await page.metrics();

  const memorySamples = [];
  
  for (let i = 0; i < iterations; i++) {
    await scenario(page); // 执行用户传入的操作
    const metrics = await page.metrics();
    memorySamples.push({
      iteration: i,
      jsHeapUsedSize: metrics.JSHeapUsedSize,
      jsHeapTotalSize: metrics.JSHeapTotalSize,
    });
    console.log(`[Iteration ${i}] JSHeapUsed=${(metrics.JSHeapUsedSize / 1048576).toFixed(1)}MB`);
    
    // 每 5 次迭代触发一次 GC
    if (i % 5 === 0) {
      await client.send('HeapProfiler.collectGarbage');
      await page.waitForTimeout(1000);
    }
  }

  // 分析增长趋势
  const firstSample = memorySamples[0].jsHeapUsedSize;
  const lastSample = memorySamples[memorySamples.length - 1].jsHeapUsedSize;
  const growth = (lastSample - firstSample) / 1048576;

  const leakDetected = memorySamples.length >= 10 &&
    memorySamples.slice(-5).every((s, i, arr) => {
      if (i === 0) return true;
      return s.jsHeapUsedSize >= arr[i - 1].jsHeapUsedSize;
    });

  console.log(`\n=== Memory Leak Report ===`);
  console.log(`Growth over ${iterations} iterations: ${growth.toFixed(2)}MB`);
  console.log(`Leak detected: ${leakDetected ? 'YES' : 'NO'}`);

  await browser.close();
  return { leakDetected, growth, memorySamples };
}
```

**常见泄漏模式快速检测**：

```javascript
// 模式 1: 全局事件监听器未清理
// DevTools Console → getEventListeners(document)

// 模式 2: setInterval 未清理
// 检查活跃的定时器
const timerCount = () => {
  // 劫持 setInterval 来追踪
  let count = 0;
  const orig = window.setInterval;
  window.setInterval = (...args) => {
    count++;
    console.warn(`[Timer] Active intervals: ${count}`);
    const id = orig(...args);
    const origClear = window.clearInterval;
    window.clearInterval = (id) => { count--; return origClear(id); };
    return id;
  };
};
// timerCount();

// 模式 3: 闭包持有大对象引用
// 在 Heap Snapshot Comparison 中搜索 "closure"
// 查看 Retained Size 是否异常大

// 模式 4: Detached DOM 节点
// 在 Heap Snapshot 的 Class filter 中搜索 "Detached"
// 或:
// document.querySelectorAll('*') 对比 snapshot 中的 DOM 节点数
```

### 4.3 Bundle 分析诊断 (Bundle Analysis)

```
目标: 精确分析每个 chunk 的内容，找出体积膨胀的根因
```

```bash
# === webpack-bundle-analyzer 深度分析 ===
# 1. 生成带 source map 的 stats
npx webpack --profile --json > stats.json
# 2. 可视化
npx webpack-bundle-analyzer stats.json

# 3. 命令行快速分析 (无需 UI)
npx webpack-bundle-analyzer stats.json --mode static --report dist/report.html

# === source-map-explorer (适合任何有 source map 的 bundle) ===
npx source-map-explorer dist/assets/*.js --html dist/source-map-report.html --only-mapped

# === 自定义分析脚本 ===
node -e "
const stats = require('./stats.json');

// 按大小排序的模块列表
const modules = stats.chunks
  .flatMap(chunk => chunk.modules || [])
  .map(m => ({ name: m.name, size: m.size, chunks: m.chunks }))
  .sort((a, b) => b.size - a.size);

console.log('=== Top 20 largest modules ===');
modules.slice(0, 20).forEach((m, i) => {
  console.log(\`\${i+1}. \${m.name} - \${(m.size/1024).toFixed(1)}KB [\${m.chunks.join(',')}]\`);
});

// 重复依赖检测
const moduleNames = modules.map(m => {
  const match = m.name && m.name.match(/node_modules\/(@[^\/]+\/[^\/]+|[^\/]+)/);
  return match ? match[1] : m.name;
});
const dupes = moduleNames.filter((name, i, arr) => arr.indexOf(name) !== i);
console.log('\n=== Potential duplicates (same package, possibly different versions) ===');
[...new Set(dupes)].forEach(name => console.log('  ' + name));

// Chunk 组成分析
console.log('\n=== Chunk composition ===');
stats.chunks.forEach(chunk => {
  const totalSize = (chunk.modules || []).reduce((sum, m) => sum + (m.size || 0), 0);
  console.log(\`  \${chunk.names[0] || chunk.id}: \${(totalSize/1024).toFixed(1)}KB (\${(chunk.modules||[]).length} modules)\`);
});
"
```

**Tree Shaking 有效性验证**：

```bash
# 1. 确认 webpack mode 为 production (production 模式自动启用)
# 2. 检查 package.json 的 sideEffects 字段
#    "sideEffects": false  ← 最佳，表示所有模块都可安全移除

# 3. 使用 webpack 的 usedExports 分析
# webpack.config.js:
# optimization: { usedExports: true, sideEffects: true }

# 4. 在 bundle 中搜索未使用的函数名
grep -o 'unusedFunction' dist/assets/main-*.js | wc -l

# 5. 检查 barrel export (index.js 重导出) 是否引入过多
# 这种模式会破坏 tree shaking:
# export * from './moduleA';  ← 暴力导出
# 改为具名导出:
# export { ComponentA, ComponentB } from './moduleA';
```

**代码分割诊断**：

```javascript
// 检查动态导入是否生效
// 在 Network 面板中搜索 .js chunk
// 期望: 初始加载只包含必要 chunk，其余按需加载

// 检查 chunk 命名是否清晰
// ✅ 好
const LazyModal = lazy(() => import(/* webpackChunkName: "modal" */ './Modal'));

// ❌ 差: 自动生成的无意义数字 chunk id
// lazy(() => import('./Modal'))  →  789.js

// 检查 common chunk 提取
// webpack.config.js 的 splitChunks 配置审查
{
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },
}
```

### 4.4 SSR / Streaming 诊断

```
目标: 诊断服务端渲染和流式传输的特有问题
```

**水合 (Hydration) 不匹配诊断**：

```
水合不匹配的根本原因:
  React 服务端与客户端渲染结果不一致，
  通常由以下原因引起:

  (a) 使用浏览器特有 API (window/document/localStorage) 在服务端
  (b) 使用随机值/时间戳 (Date.now(), Math.random())
  (c) 根据浏览器特性条件渲染 (navigator.userAgent)
  (d) 第三方脚本在服务端注入不同内容
  (e) HTML 格式错误导致浏览器解析树与服务端不同
```

```javascript
// 水合不匹配检测辅助组件
// 用于 Next.js App Router 开发环境

export function HydrationGuard({ children }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  // 服务端不渲染可能不匹配的内容
  if (!mounted) {
    return (
      <div suppressHydrationWarning style={{ visibility: 'hidden' }}>
        {children}
      </div>
    );
  }

  return children;
}

// 使用 suppressHydrationWarning 进行安全抑制
// 仅当内容确实会在客户端不同时使用:
// <time suppressHydrationWarning>{formattedDate}</time>
```

**流式 (Streaming) 中断诊断**：

```bash
# 检测 Next.js App Router 流式响应是否正常
curl -N -H "Accept: text/html" http://localhost:3000/page-with-suspense 2>&1 | head -50

# 观察输出: 应该分多次写入，先返回 Shell HTML，然后逐步注入 Suspense 内容
# 正常流式输出模式:
#   <!DOCTYPE html><html>... ← Shell (立即)
#   <!--$?--><template id="B:0"></template>... ← Suspense fallback (立即)
#   <!--/$-->
#   ... hidden div with data ... ← 流式注入的 Suspense 内容
#   <script>替换 Suspense fallback</script> ← 水合用脚本

# 如果一次性返回完整 HTML → 流式传输未生效

# 检查 Transfer-Encoding
curl -sI -H "Accept: text/html" http://localhost:3000/ | grep -i transfer
# Transfer-Encoding: chunked  ← 应出现此头

# 模拟网络中断测试
curl -N -H "Accept: text/html" http://localhost:3000/ &
PID=$!
sleep 2
kill $PID  # 2 秒后断连
echo "--- Streaming connection killed after 2s ---"
# 检查页面在此情况下是否优雅降级
```

**SSR 性能诊断**：

```javascript
// Next.js 服务端渲染性能监控
// instrumentation.ts (或 instrumentation.js)

export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const { performance, PerformanceObserver } = require('perf_hooks');
    
    const obs = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name.includes('Next.js') || entry.name.includes('render')) {
          console.log(`[SSR Perf] ${entry.name}: ${entry.duration.toFixed(2)}ms`);
        }
      }
    });
    obs.observe({ type: 'measure', buffered: true });
  }
}

// 或在组件中测量
// app/layout.tsx
export default function RootLayout({ children }) {
  // 仅在服务端测量
  if (typeof window === 'undefined') {
    const start = performance.now();
    // ... render
    const end = performance.now();
    console.log(`[SSR] RootLayout render: ${(end - start).toFixed(2)}ms`);
  }
  return <html>...</html>;
}
```

**React Server Components (RSC) 负载分析**：

```bash
# 查看 RSC payload 大小 (Next.js App Router)
# Network 面板 → 搜索 "_rsc" 或 RSC 请求
# 查看响应大小和加载时间

# 命令行测量
curl -s -o /dev/null -w "RSC Payload Size: %{size_download} bytes\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -H "RSC: 1" \
  http://localhost:3000/

# RSC payload 应保持小巧 (一般 < 50KB 为健康)
# 如果超过 200KB，考虑:
#   - 是否将太多客户端组件误标记为服务端组件
#   - 是否传递了大量 props 数据
#   - 是否使用了 'use client' 过多导致 RSC 边界过于靠外
```

---

## 五、诊断日志模板

```
目标: 每次诊断结束后生成结构化日志，便于代码审查和技术复盘
```

### 诊断记录模板

```markdown
## 诊断记录: [日期] - [问题简述]

### 初始症状
- 用户描述:
- 可复现环境:
- 影响范围: [用户数 / 页面数 / 设备数]

### 诊断路径

| 层级 | 诊断操作 | 发现 | 排除/确认 |
|------|---------|------|----------|
| L1 浏览器 | Performance 录制 | Frame duration: ___ms | |
| L1 浏览器 | 内存采样 | Heap: ___MB → ___MB | |
| L1 浏览器 | Long Task 检测 | ___ 个 > 50ms | |
| L2 框架 | Re-render 检测 | ___ 次不必要渲染 | |
| L2 框架 | Context 更新追踪 | ___ 个消费者更新 | |
| L3 构建 | Bundle 分析 | 最大 chunk: ___KB | |
| L3 构建 | 重复依赖检测 | ___ 个重复包 | |
| L4 网络 | 瀑布分析 | TTFB: ___ms | |
| L4 网络 | 协议检查 | Protocol: ___ | |
| L5 工程 | 灰度对比 | 差异: ___ | |
| L5 工程 | API 契约检查 | [有/无] 变化 | |

### 根因分析
[Layer n] 层发现:
- 直接原因:
- 根本原因:
- 影响的架构决策/假设:

### 修复方案
- 最小修复:
- 回退方案:
- 验证方案:
- 回归测试:

### 架构复盘
- 此问题是否揭示了设计缺陷? [是/否]
- 如果是，建议的架构调整:
- 是否需要更新 pitfalls/INDEX.md? [是/否]
- 是否需要更新 skill 的检查清单? [是/否]

### 时间线
- 发现时间:
- 诊断耗时:
- 修复耗时:
- 验证耗时:
- 总计:
```

---

## 附录 A: 快速诊断命令集

```bash
# ─── 一键环境快照 ───
echo "Node $(node -v) | npm $(npm -v) | OS $(uname -s) $(uname -m)" && \
ls -la node_modules/.package-lock.json 2>/dev/null && echo "Lockfile: OK"

# ─── 一键性能基线 ───
npx lighthouse http://localhost:3000 --output=json --only-categories=performance 2>/dev/null | \
  node -e "const d=require('fs').readFileSync('/dev/stdin','utf8');const j=JSON.parse(d);console.table({LCP:j.audits['largest-contentful-paint'].numericValue/1000+'s',TBT:j.audits['total-blocking-time'].numericValue+'ms',CLS:j.audits['cumulative-layout-shift'].numericValue,SI:j.audits['speed-index'].numericValue/1000+'s'})"

# ─── 一键 Bundle 大小 ───
ls -lhS dist/assets/*.js 2>/dev/null | head -10 && \
  echo "Total JS: $(du -sh dist/assets/ 2>/dev/null | cut -f1)" && \
  echo "Total CSS: $(du -sh dist/assets/*.css 2>/dev/null | cut -f1)"

# ─── 一键网络诊断 ───
curl -sI http://localhost:3000 | grep -iE 'server|content-type|cache|etag|x-powered'
```

## 附录 B: 诊断决策树

```
问题: 白屏 / 不渲染
├── HTML 响应为空?
│   ├── 是 → Layer 2: SSR 错误 → 检查服务端日志
│   └── 否 → 继续
├── JavaScript 报错?
│   ├── 是 → Console 查看错误堆栈
│   │   ├── "is not a function" → Layer 2: 版本不兼容 / 导入错误
│   │   ├── "Cannot read property of undefined" → Layer 2: 异步时序
│   │   ├── "ChunkLoadError" → Layer 3: chunk 404 / CDN 问题
│   │   └── "Minified React error #" → 查阅 React 错误码表
│   └── 否 → 继续
├── DOM 为空?
│   ├── React DevTools 显示组件? → Layer 1: CSS 隐藏 (opacity/visibility/display)
│   └── React DevTools 空白? → Layer 2: React 未挂载 / 根节点选择错误
└── 仅在特定设备? → Layer 1: 兼容性 / Layer 4: CDN 边缘节点

问题: 交互卡顿
├── 点击无响应?
│   ├── 事件处理器被覆盖? → Layer 2: z-index / pointer-events / 事件冒泡
│   ├── 主线程阻塞? → Layer 1: Performance 录制查看 Long Task
│   └── 内存过高导致 GC 频繁? → Layer 1: Memory 面板查看锯齿状内存图
├── 滚动卡顿?
│   ├── scroll 事件未节流? → Layer 1: passive event listener
│   ├── 滚动区域有大量 DOM? → Layer 1: 虚拟滚动 (windowing)
│   └── 滚动时持续重绘? → Layer 1: Paint flashing 检测
└── 动画掉帧?
    ├── 使用 left/top 而非 transform? → Layer 1: 重排 vs 合成
    ├── 合成层爆炸? → Layer 1: Layer borders 检测
    └── requestAnimationFrame 内执行耗时计算? → Layer 1: 拆分帧预算

问题: 内存持续增长
├── 页面切换时内存不释放?
│   ├── SPA 路由切换? → Layer 2: 组件卸载时未清理副作用
│   └── iframe 未销毁? → Layer 1: iframe 内存泄漏
├── 弹窗/Modal 打开关闭?
│   └── 事件监听器未移除? → Layer 1: getEventListeners()
├── 定时器/轮询?
│   └── setInterval 未 clear? → Layer 2: useEffect 清理函数
└── WebSocket/SSE?
    └── 断连后消息队列堆积? → Layer 4: 连接管理
```

---

> **版本**: 1.0.0
> **最后更新**: 2026-06-14
> **维护者**: frontend-architect-expert skill
> **关联文档**: pitfalls/INDEX.md / knowledge-map.md / patterns.md / checklist.md
