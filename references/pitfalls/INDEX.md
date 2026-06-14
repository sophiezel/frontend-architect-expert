# 前端架构陷阱索引 (Architecture Pitfalls Index)

> 基于前端开发陷阱升级的架构级索引。每个 pitfall 不仅包含开发层面的诊断和修复，
> 更深度分析架构层面根因、系统级预防策略、跨层影响链和相关架构模式。
>
> 目前已收录 **50 个**架构级陷阱，覆盖数据流、构建部署、CSS、移动端、JavaScript、
> 可观测性与性能工程、隐私合规等领域。

---

## 一、按架构影响维度查找

### 数据流与状态管理
状态分类、副作用边界、派生策略、批处理时序。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-001](pit-001.md) | useEffect 无限循环 | 派生数据存为独立状态、副作用边界缺失、模块边界错误 |
| [pit-002](pit-002.md) | useEffect 闭包陷阱 | 长生命周期资源与短生命周期 props 绑定方式错误、状态时间线建模缺失 |
| [pit-004](pit-004.md) | useRef vs useState 混用 | 状态分类体系缺失、可变值契约未声明、渲染优化策略无系统化 |
| [pit-005](pit-005.md) | setState 异步批处理 | 多步状态转换未原子化、更新策略与读取策略耦合、时序依赖被隐藏 |

### 组件身份与协调
列表渲染的身份映射、key 协议、虚拟列表复用。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-003](pit-003.md) | 列表渲染缺少 key | 数据模型层缺少稳定身份标识、API Schema 未强制 ID、身份传递链路断裂 |
| [pit-014](pit-014.md) | FlatList 缺少 keyExtractor | 跨平台静默失败、RN 身份传输链路在 Bridge 中衰减、未封装平台原生列表 |

### 构建与部署
代码分割、Tree Shaking、chunk 加载、HMR 状态保持、副作用声明。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-011](pit-011.md) | chunk 加载 404 | 部署原子性未建模、资源加载缺少版本感知、CDN 假设过于乐观 |
| [pit-012](pit-012.md) | Tree Shaking 误删 CSS | sideEffects 全局否定而非白名单、CSS 副作用未被架构承认、构建产物健康检查缺失 |
| [pit-013](pit-013.md) | HMR 状态丢失 | 文件按职责混合破坏模块边界、变更频率未作为模块划分维度、开发时架构保障缺失 |

### CSS 布局与渲染
视口单位、层叠上下文、Flexbox/Grid 截断、合成层、scroll-snap。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-006](pit-006.md) | iOS Safari 100vh | 视口语义直接暴露给业务、缺少视口抽象层、降级策略不系统 |
| [pit-007](pit-007.md) | z-index 层叠上下文 | 浮层渲染层级无全局管理、DOM 结构決定 Z 轴层级、层级分配器缺失 |
| [pit-008](pit-008.md) | Flexbox 嵌套塌陷 | 截断策略与布局容器缺少显式契约、缺少"可收缩容器"抽象、布局约束不向下传递 |
| [pit-009](pit-009.md) | backdrop-filter 性能 | 视觉效果未纳入性能预算、缺少设备能力分层、合成层预算无治理 |
| [pit-010](pit-010.md) | scroll-snap iOS 异常 | 原生 CSS 特性直接暴露而无抽象、交互模式缺少降级链、平台兼容矩阵未纳入选型 |

### 移动端适配
键盘交互、WKWebView、平台差异。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-021](pit-021.md) | iOS 键盘遮挡/不恢复 | 移动端平台适配未系统化、键盘作为交互设施未提升为全局关注点、WKWebView 与 Safari 差异未建模 |

### 可观测性与性能工程
RUM 数据采集、错误监控降噪、埋点治理、会话回放隐私、性能预算门禁。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-044](pit-044.md) | 前端监控盲区 | 可观测性未作为架构质量属性、前端 I/O 模型错误分类缺失、告警策略非设计而是追加 |
| [pit-046](pit-046.md) | RUM CLS value/delta 误用 | 采集层与聚合层语义契约缺失、缺少 Metrics Semantics Validator、可观测性数据建模缺失 |
| [pit-047](pit-047.md) | Sentry 上报风暴 | 信号提取管道缺失、错误可操作性(Actionability)建模缺失、缺少多层噪音过滤体系 |
| [pit-048](pit-048.md) | 埋点数据污染 | 数据契约(Data Contract)在前端缺失、埋点被当作副作用而非数据产品、缺少全生命周期治理 |
| [pit-049](pit-049.md) | 会话回放隐私泄露 | Privacy by Design 原则缺失、DOM 数据分类标记缺失、缺少采集/传输/存储三层防护 |
| [pit-050](pit-050.md) | 性能预算形同虚设 | 非功能性需求可测试性缺失、性能所有权模糊、Lab-RUM 脱节、预算无强制闭环 |

### JavaScript 语言陷阱
浮点精度、Promise 错误处理、异步迭代模式、日期时区、数组操作。

| Pitfall | 标题 | 架构根因关键词 |
|---------|------|--------------|
| [pit-015](pit-015.md) | 浮点数精度 | 数值类型策略缺失、精度边界在序列化中丢失、全局数值策略未声明 |
| [pit-016](pit-016.md) | Promise 静默吞掉 | 错误处理策略未全局化/分层、错误被视为实现细节、异步错误传播链断裂 |
| [pit-017](pit-017.md) | forEach 中 async 无效 | 同步/异步迭代语义未区分、forEach 过度使用、缺少批量执行引擎 |
| [pit-018](pit-018.md) | Date 解析时区陷阱 | 存储/传输/展示时区三态未分离、时区信息在序列化边界丢失、全局时间策略缺失 |
| [pit-019](pit-019.md) | map 遗忘 return | TypeScript 严格模式未启用、代码风格未提升为架构约束、缺少安全 map 抽象 |
| [pit-020](pit-020.md) | map 遗漏 return (分支) | map 回调分支完整性未在类型系统表达、map 语义被弱化为副作用 |

---

## 二、按架构模式关联查找

> 每个 pitfall 的"架构视角"小节都关联了 `patterns.md` 和 `decisions.md` 中的对应模式。

### 涉及模式: 状态管理模式 (patterns.md 第一节)

| Pitfall | 关联模式 |
|---------|---------|
| [pit-001](pit-001.md) | CQRS 前端化、反模式: useEffect 地狱 |
| [pit-002](pit-002.md) | useRef 逃生舱、CQRS 前端化 |
| [pit-004](pit-004.md) | 状态管理决策框架、反模式: 巨型组件 |
| [pit-005](pit-005.md) | 单向数据流、XState 状态机 |

### 涉及模式: 组件组合模式 (patterns.md 第二节)

| Pitfall | 关联模式 |
|---------|---------|
| [pit-003](pit-003.md) | Compound Components、身份映射协议 |
| [pit-007](pit-007.md) | Compound Components (Dropdown/Tooltip)、Portal 策略 |
| [pit-010](pit-010.md) | Compound Components (Carousel)、交互模式封装 |

### 涉及模式: 渲染优化模式 (patterns.md 第四节)

| Pitfall | 关联模式 |
|---------|---------|
| [pit-009](pit-009.md) | 性能预算、Skeleton/SSR Streaming、设备能力分层 |
| [pit-014](pit-014.md) | 虚拟列表 (Virtual List) |

### 涉及模式: 反模式 (patterns.md 第六节)

| Pitfall | 关联模式 |
|---------|---------|
| [pit-001](pit-001.md) | useEffect 地狱 |
| [pit-004](pit-004.md) | 巨型组件 |
| [pit-016](pit-016.md) | Event Bus (同构: 事件无法追踪) |
| [pit-017](pit-017.md) | Event Bus (同构: 操作无法追踪完成状态) |
| [pit-019](pit-019.md) | 过早抽象 |

### 涉及决策: 构建工具选型 (decisions.md 第四节)

| Pitfall | 关联决策 |
|---------|---------|
| [pit-011](pit-011.md) | Webpack vs Vite (chunk 加载机制差异) |
| [pit-012](pit-012.md) | Tree shaking 机制差异 (Rollup vs Webpack) |
| [pit-013](pit-013.md) | Vite 内置 HMR vs Webpack HMR |

### 涉及决策: CSS 方案选型 (decisions.md 第三节)

| Pitfall | 关联决策 |
|---------|---------|
| [pit-006](pit-006.md) | CSS Variables 作为 Design Token |
| [pit-008](pit-008.md) | 设计系统中的截断原语 (Truncate utility) |
| [pit-009](pit-009.md) | 视觉效果性能等级定义 |
| [pit-010](pit-010.md) | @supports 特性检测与渐进增强 |

### 涉及决策: 渲染模式选型 (decisions.md 第一节)

| Pitfall | 关联决策 |
|---------|---------|
| [pit-011](pit-011.md) | ISR + CSR 混合的 chunk 加载策略 |
| [pit-018](pit-018.md) | SSR 日期格式化水合不匹配 |

### 涉及决策: 状态管理选型 (decisions.md 第二节)

| Pitfall | 关联决策 |
|---------|---------|
| [pit-002](pit-002.md) | Zustand getState() 脱离 React 访问 |
| [pit-005](pit-005.md) | useReducer vs 多个 useState |

---

## 三、按架构预防策略查找

### 需要静态防护 (ESLint / TypeScript / stylelint)

| Pitfall | 关键静态规则 |
|---------|-------------|
| [pit-001](pit-001.md) | `react-hooks/exhaustive-deps: error`, `no-restricted-syntax` (依赖数组字面量) |
| [pit-003](pit-003.md) | `react/jsx-key: error`, 禁止 `Math.random()` 为 key |
| [pit-006](pit-006.md) | stylelint: `declaration-property-value-disallowed-list` (禁止裸 100vh) |
| [pit-007](pit-007.md) | stylelint: 禁止裸 z-index 数值，强制使用 CSS 变量 |
| [pit-010](pit-010.md) | stylelint: 禁止 `-webkit-overflow-scrolling` 与 `scroll-snap` 同时出现 |
| [pit-012](pit-012.md) | `package.json` sideEffects 白名单策略 |
| [pit-015](pit-015.md) | `no-floating-decimal`, 自定义: 禁止浮点等值比较 |
| [pit-016](pit-016.md) | `@typescript-eslint/no-floating-promises`, `promise/catch-or-return` |
| [pit-017](pit-017.md) | `@typescript-eslint/no-misused-promises`, 禁止 `forEach` + `async` |
| [pit-018](pit-018.md) | 禁止 `new Date(string)` 无时区 |
| [pit-019](pit-019.md) | `array-callback-return: error`, `noImplicitReturns: true` |
| [pit-020](pit-020.md) | 同上 + 禁止 map 中块语句无 return |

### 需要 CI 防护

| Pitfall | CI 检测方法 |
|---------|------------|
| [pit-001](pit-001.md) | 渲染预算测试 (Jest + Profiler)、dependency-cruiser 模块边界检查 |
| [pit-002](pit-002.md) | 渲染一致性测试、lint-staged hooks |
| [pit-006](pit-006.md) | CSS 变量使用检查、Playwright 多设备视口测试 |
| [pit-009](pit-009.md) | Lighthouse CI 性能预算、backdrop-filter 数量/面积阈值 |
| [pit-011](pit-011.md) | chunk 可访问性验证、HTML-chunk 一致性检查 |
| [pit-012](pit-012.md) | CSS 产物完整性验证、选择器数量对比、视觉回归测试 |
| [pit-015](pit-015.md) | 精度测试套件、金额一致性验证 |
| [pit-018](pit-018.md) | 多时区并行测试 (TZ=UTC/Asia/Shanghai/America/New_York) |

### 需要架构防护 (模块边界/API 契约/设计系统)

| Pitfall | 架构级防护措施 |
|---------|---------------|
| [pit-001](pit-001.md) | 副作用声明机制 (useSyncExternalStore/RTK Query)、模块边界规则 |
| [pit-002](pit-002.md) | 长生命周期资源管理器模式、依赖注入 WebSocket 连接 |
| [pit-003](pit-003.md) | API Schema 强制 ID 字段、数据层归一化 (createEntityAdapter) |
| [pit-005](pit-005.md) | useReducer 原子更新策略、状态机 (XState) |
| [pit-006](pit-006.md) | 视口适配中间件、全屏组件封装 `<FullScreen>` |
| [pit-007](pit-007.md) | Portal 渲染策略、全局层级管理器 (Stacking Manager) |
| [pit-008](pit-008.md) | 布局原语库 (`<Truncate>` / `<FlexCell shrink>`)、CSS reset min-width: 0 |
| [pit-009](pit-009.md) | 视觉效果委员会、合成层预算、设备能力探测 |
| [pit-010](pit-010.md) | 交互模式库 (@app/interaction-patterns)、降级链策略 |
| [pit-011](pit-011.md) | 非覆盖式部署 (版本目录)、版本协商协议 |
| [pit-012](pit-012.md) | sideEffects 分类体系、构建产物完整性测试 |
| [pit-014](pit-014.md) | 平台原生列表封装 (`<AppFlatList>`)、数据归一化中间层 |
| [pit-015](pit-015.md) | 数值类型体系 (Money/Percentage)、API 金额字段字符串传输 |
| [pit-016](pit-016.md) | 全局错误处理管道、数据获取层封装、错误优先级分级 |
| [pit-017](pit-017.md) | 批量操作引擎 (Serial/Parallel/FailSafe/RateLimited)、禁止 forEach 规范 |
| [pit-018](pit-018.md) | 日期三态模型 (存储/传输/展示)、API Schema 强制时区 |
| [pit-021](pit-021.md) | 移动端视口管理层 (与 pit-006 共享)、键盘行为声明 |

---

## 四、按症状快速查找

### 白屏
- [pit-011](pit-011.md) — chunk 加载 404
- [pit-012](pit-012.md) — Tree Shaking 误删 CSS
- [pit-016](pit-016.md) — Promise 静默吞掉（数据加载失败白屏）

### 性能问题 (CPU/GPU/FPS)
- [pit-001](pit-001.md) — useEffect 无限循环 (CPU 100%)
- [pit-009](pit-009.md) — backdrop-filter 性能灾难 (GPU)
- [pit-005](pit-005.md) — setState 竞态请求

### 样式异常
- [pit-006](pit-006.md) — iOS Safari 100vh 滚动条
- [pit-007](pit-007.md) — z-index 层叠上下文
- [pit-008](pit-008.md) — Flexbox 嵌套塌陷
- [pit-010](pit-010.md) — scroll-snap iOS 异常
- [pit-012](pit-012.md) — Tree Shaking 误删 CSS

### 构建失败
- [pit-011](pit-011.md) — chunk 加载 404
- [pit-012](pit-012.md) — Tree Shaking 误删 CSS
- [pit-013](pit-013.md) — HMR 状态丢失

### 数据异常
- [pit-015](pit-015.md) — 浮点数精度
- [pit-016](pit-016.md) — Promise 静默吞掉
- [pit-017](pit-017.md) — forEach + async 无效
- [pit-018](pit-018.md) — Date 时区陷阱
- [pit-019](pit-019.md) — map 遗忘 return
- [pit-020](pit-020.md) — map 分支 return 遗漏

### 崩溃/卡死
- [pit-001](pit-001.md) — useEffect 无限循环
- [pit-003](pit-003.md) — 列表 key 错误导致状态串位
- [pit-014](pit-014.md) — FlatList keyExtractor 缺失

### 移动端专项
- [pit-021](pit-021.md) — iOS 键盘遮挡/不恢复
- [pit-006](pit-006.md) — iOS Safari 100vh
- [pit-010](pit-010.md) — scroll-snap iOS 异常

### 可观测性与监控
- [pit-044](pit-044.md) — 前端监控盲区（错误/性能/行为/告警四维盲区）
- [pit-046](pit-046.md) — RUM CLS 数据采集失真（value vs delta 误用）
- [pit-047](pit-047.md) — Sentry 上报风暴（配额耗尽/噪音过滤缺失）
- [pit-048](pit-048.md) — 埋点数据污染（脏数据/类型错误/重复丢失）
- [pit-049](pit-049.md) — 会话回放隐私泄露（rrweb 未脱敏/GDPR 合规）
- [pit-050](pit-050.md) — 性能预算形同虚设（CI 门禁配置了但从未阻止回归）

---

## 五、按标签查找

### #architecture
所有 21 个 pitfall 均包含 `#architecture` 标签，并包含完整的"架构视角"小节。

### #react
[pit-001](pit-001.md) · [pit-002](pit-002.md) · [pit-003](pit-003.md) · [pit-004](pit-004.md) · [pit-005](pit-005.md) · [pit-013](pit-013.md)

### #css
[pit-006](pit-006.md) · [pit-007](pit-007.md) · [pit-008](pit-008.md) · [pit-009](pit-009.md) · [pit-010](pit-010.md)

### #javascript
[pit-015](pit-015.md) · [pit-016](pit-016.md) · [pit-017](pit-017.md) · [pit-018](pit-018.md) · [pit-019](pit-019.md) · [pit-020](pit-020.md)

### #webpack / #build
[pit-011](pit-011.md) · [pit-012](pit-012.md) · [pit-013](pit-013.md)

### #react-native
[pit-014](pit-014.md)

### #observability / #monitoring
[pit-044](pit-044.md) · [pit-046](pit-046.md) · [pit-047](pit-047.md) · [pit-048](pit-048.md) · [pit-050](pit-050.md)

### #privacy / #compliance
[pit-049](pit-049.md)

### #performance-budget
[pit-050](pit-050.md)

### #rum / #web-vitals
[pit-044](pit-044.md) · [pit-046](pit-046.md)

### #sentry
[pit-044](pit-044.md) · [pit-047](pit-047.md)

### #tracking / #analytics
[pit-048](pit-048.md)

### #ios-safari / #mobile
[pit-006](pit-006.md) · [pit-010](pit-010.md) · [pit-021](pit-021.md)

---

## 六、架构模式交叉引用矩阵

| Pitfall | 模式引用 | 决策引用 |
|---------|---------|---------|
| pit-001 | CQRS前端化, useEffect地狱反模式 | 状态管理选型 |
| pit-002 | useRef逃生舱, CQRS前端化 | 状态管理选型(Zustand) |
| pit-003 | Compound Components, Virtual List | API Schema契约 |
| pit-004 | 状态管理决策框架, 巨型组件反模式 | 状态管理选型(更新频率) |
| pit-005 | 单向数据流, XState状态机 | 状态管理选型(useReducer) |
| pit-006 | CSS Variables / Design Token | CSS方案选型, 图片策略 |
| pit-007 | Compound Components, 事件总线反模式 | 微前端决策 |
| pit-008 | CSS 截断原语, Virtual List | CSS方案选型 |
| pit-009 | 渲染优化模式, CSS-in-JS反模式 | CSS方案选型, 性能预算 |
| pit-010 | 组件组合模式 | 构建工具选型, CSS方案选型 |
| pit-011 | 构建架构 (Webpack/Vite), Monorepo | 渲染模式选型, 部署架构 |
| pit-012 | Monorepo sideEffects, 构建架构 | CSS方案选型, 构建工具选型 |
| pit-013 | Compound Components, 过早抽象反模式 | Monorepo管理 |
| pit-014 | Virtual List, 状态管理模式 | pit-003同源 |
| pit-015 | CQRS前端化, 品牌类型 | API Schema契约 |
| pit-016 | Event Bus反模式, 组件库决策 | Node.js版本策略 |
| pit-017 | 乐观更新, CQRS前端化, Event Bus反模式 | — |
| pit-018 | — | 渲染模式选型(SSR), CSS方案选型 |
| pit-019 | 过早抽象反模式, 组件组合模式 | TypeScript严格模式 |
| pit-020 | 与pit-019相同 + TDD | — |
| pit-021 | pit-006同源视口管理 | CSS方案选型, RN WebView |
| pit-044 | Telemetry as Code, Circuit Breaker(前端) | — |
| pit-046 | Metrics Contract Pattern, Telemetry Pipeline | Data Quality SLA |
| pit-047 | Signal Extraction Pipeline, Error Budget | Client-Side Circuit Breaker |
| pit-048 | Data Contract Pattern, Registry-Driven Development | SDK as Gatekeeper |
| pit-049 | Privacy by Design, Data Classification for Frontend, Layered Privacy Protection | — |
| pit-050 | Performance Budget as Code, RAIL模型, Progressive Performance Governance, Lab-RUM Bridge | — |
