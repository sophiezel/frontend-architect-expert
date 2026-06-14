# 架构模式与反模式 (Architecture Patterns & Anti-Patterns)

> 本文档整理经过 12 年一线验证的设计模式与反模式。
> 每个模式都有选择标准、边界条件和失败案例。

---

## 一、状态管理模式

### 决策框架

```
状态作用域决策树:
  单个组件内 ──→ useState / ref / reactive
       │
  父子组件间 ──→ Props (Lifting State Up)
       │
  兄弟/跨层级 ──→ Context + useReducer (小型) / 状态库 (中大型)
       │
  全局/多模块 ──→ Zustand / Jotai / Redux Toolkit / XState
```

### 1. Lifting State Up（状态提升）

```jsx
// 适用: 父子/兄弟组件间共享少量状态
// 边界: 只要超过 3 层传递就应切换方案
function Parent() {
  const [value, setValue] = useState('');
  return (
    <>
      <Input value={value} onChange={setValue} />
      <Display value={value} />
    </>
  );
}
// 选型信号: 没有跨路由共享、没有复杂派生、状态量 < 5 个
```

### 2. Context + useReducer

```jsx
// 适用: 中型应用，状态逻辑复杂但不需要中间件
// 边界: Context value 变化导致所有消费者重渲染 — 需要拆分 Context
const CountContext = createContext();
const CountDispatchContext = createContext();
// 拆分 state 和 dispatch 两个 Context，避免不消费 state 的组件也重渲染

function CountProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <CountContext.Provider value={state}>
      <CountDispatchContext.Provider value={dispatch}>
        {children}
      </CountDispatchContext.Provider>
    </CountContext.Provider>
  );
}
```

### 3. Zustand 切片模式

```javascript
// 适用: 中小型应用、需要 React 之外的访问、简单的全局状态
// 核心优势: 选择器精确订阅、无 Provider 包裹、可脱离 React 使用

const useFishStore = create((set) => ({
  fishes: 0,
  addFish: () => set((s) => ({ fishes: s.fishes + 1 })),
}));

// 精确选择器 — 只在 fishes 变化时重渲染
const fishes = useFishStore((s) => s.fishes);

// 切片 (Slices) — 按领域拆分状态
const createBearSlice = (set) => ({ bears: 0, addBear: () => set(s => ({ bears: s.bears + 1 })) });
const createFishSlice = (set) => ({ fishes: 0, addFish: () => set(s => ({ fishes: s.fishes + 1 })) });
const useStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}));
```

**选择 Zutand 的时候**：
- 不需要时间旅行调试
- 不需要中间件生态
- 团队倾向于简洁 API
- 需要在 React 之外（WebSocket handler、Router guard）访问状态

### 4. Jotai 原子化模式

```javascript
// 适用: 状态之间存在派生关系、需要细粒度更新
// 原子(Atom)是独立的状态单元，可组合

const priceAtom = atom(100);
const quantityAtom = atom(2);
const totalAtom = atom((get) => get(priceAtom) * get(quantityAtom));
// totalAtom 只在 priceAtom 或 quantityAtom 变化时重新计算

// 异步派生
const userAtom = atom(async () => {
  const res = await fetch('/api/user');
  return res.json();
});
```

**选择 Jotai 的时候**：
- 状态之间有大量派生和依赖关系
- 需要细粒度的组件级订阅
- 偏好组合式 API

### 5. XState 状态机

```javascript
// 适用: 多步骤流程、Wizard、支付流程、复杂表单、需要可视化状态图
// 边界: 简单状态管理不要用，学习曲线陡峭

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: { TOGGLE: 'active' }
    },
    active: {
      on: { TOGGLE: 'inactive' }
    }
  }
});
```

**选择 XState 的时候**：
- 状态转换有严格的合法/非法边界
- 需要可视化状态流转给 PM/QA
- 自动生成测试用例

### 状态管理选择总表

| 方案 | 学习成本 | 打包体积 | 最佳场景 | 避免场景 |
|------|---------|---------|---------|---------|
| useState | 零 | 0 | 组件内状态 | 跨组件共享 |
| Context + useReducer | 低 | 0 | 中小型应用 | 高频更新 (>1次/秒) |
| Zustand | 低 | ~1KB | 通用全局状态 | 时间旅行调试 |
| Jotai | 中 | ~2KB | 派生状态多 | 团队不熟悉原子化 |
| Redux Toolkit | 中 | ~12KB | 大型应用/团队强制规范 | 小型应用 |
| XState | 高 | ~15KB | 多步骤流程/状态机 | 简单 CRUD |

---

## 二、组件组合模式

### 选择标准

```
组件复用/变体需求?
  ├─ 固定变体(少量) ──→ Props 驱动的条件渲染
  ├─ 可插拔内容 ──→ Slots (Vue) / children (React) / Compound Components
  ├─ 逻辑复用 ──→ Custom Hooks (React) / Composables (Vue)
  ├─ 横切关注点(权限/日志/埋点) ──→ HOC(Wrap)/Vue 指令
  └─ 行为注入 ──→ Render Props
```

### Compound Components（复合组件）

```jsx
// 适用: 组件需要隐式共享状态、用户可自由组合子组件顺序和数量
// 代表: <select>/<option>, <Tabs>/<TabPanel>, <Menu>/<MenuItem>

function Tabs({ children }) {
  const [active, setActive] = useState(0);
  // 使用 Context 隐式传递 active 和 onChange
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}
Tabs.Tab = ({ children }) => { /* ... */ };
Tabs.Panel = ({ children }) => { /* ... */ };

// 使用
<Tabs>
  <Tabs.Tab>标签1</Tabs.Tab>
  <Tabs.Tab>标签2</Tabs.Tab>
  <Tabs.Panel>内容1</Tabs.Panel>
  <Tabs.Panel>内容2</Tabs.Panel>
</Tabs>
```

### Render Props vs HOC vs Custom Hooks

```jsx
// Render Props (已过时，仅维护遗留代码时接触)
<DataFetcher render={(data) => <Display data={data} />} />

// HOC (横切关注点：权限、日志)
function withAuth(Component) {
  return (props) => {
    const { user } = useAuth();
    if (!user) return <Redirect to="/login" />;
    return <Component {...props} user={user} />;
  };
}

// Custom Hooks (首选！组合优于继承)
function useAuth() {
  const [user, setUser] = useState(null);
  useEffect(() => { /* fetch user */ }, []);
  return { user, isLoggedIn: !!user };
}
```

---

## 三、数据流模式

### 单向数据流

```
state → view → action → state → view → ...
  │                         │
  └─ 状态驱动视图 ── 视图触发 action ── action 修改状态 ──┘
  这是 React/Vue 的核心理念，保证数据流向可追踪、可预测
```

### CQRS 前端化

```javascript
// 将读写分离模式引入前端状态管理
// 适用: 读频次远大于写频次的场景(消息列表、仪表盘)

// Command (写): 改变状态
function addTodoCommand(todo) {
  const store = useTodoStore.getState();
  store.todos.push(todo);
  // 触发持久化
  syncToBackend(store.todos);
}

// Query (读): 读取派生数据
const useCompletedTodos = () =>
  useTodoStore(s => s.todos.filter(t => t.completed));
```

### Event Bus（危险模式）

```javascript
// ❌ 反模式: 全局 Event Bus
const bus = mitt();
bus.emit('user:login', user);
bus.on('cart:add', handler);

// 问题:
// 1. 无法追踪事件流: 谁 emit 的? 谁在监听?
// 2. 组件卸载时容易忘记解绑
// 3. 时序依赖: 事件顺序决定正确性
// 4. 调试困难: DevTools 看不到事件链

// ✅ 替代: 集中状态管理 + 组件间通过共享状态通信
// 如果确实需要跨组件通信，使用 Context/Zustand 的 getState()
```

---

## 四、渲染优化模式

### Virtual List（虚拟列表）

```jsx
// 原理: 只渲染可视区域 + buffer 的元素
// 关键指标: 可视区域高度 / 预估行高 = 可见数量 (通常 20-50 个)

// react-window 示例
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  width="100%"
  itemCount={100000}
  itemSize={40}  // 每行高度固定
>
  {({ index, style }) => <div style={style}>Row {index}</div>}
</FixedSizeList>

// 动态高度: VariableSizeList (需要 getItemSize 预估 + 实测)
// Vue: vue-virtual-scroller <RecycleScroller> / <DynamicScroller>
```

### Optimistic UI（乐观更新）

```javascript
// 适用: 写操作 > 95% 成功率、用户感知即时响应优先
async function toggleLike(postId) {
  // 1. 立即更新 UI (乐观)
  const store = useStore.getState();
  const previous = store.posts[postId].liked;
  store.setLike(postId, !previous);

  try {
    // 2. 后台发送真实请求
    await api.like(postId, !previous);
  } catch {
    // 3. 失败回滚
    store.setLike(postId, previous);
    toast.error('操作失败，请重试');
  }
}
```

### Skeleton / SSR Streaming

```jsx
// SSR Streaming: 先发送 Shell，再流式填充动态内容
// Next.js App Router 示例
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <Header /> {/* 静态 Shell */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductList /> {/* 流式注入 */}
      </Suspense>
    </div>
  );
}
```

---

## 五、构建架构模式

### Monorepo 管理

```yaml
# pnpm workspace + Turborepo
packages:
  - "apps/*"        # 应用
  - "packages/*"    # 共享包
  - "tooling/*"     # 工具链

# Turborepo 核心能力:
# 1. 并行执行: 无依赖的 task 同时跑
# 2. 缓存: 输入不变则跳过，CI 和本地共享缓存
# 3. 依赖图: 只构建变更影响的包
```

### 微前端选型

| 方案 | 隔离级别 | 设计理念 | 适用场景 | 核心限制 |
|------|---------|---------|---------|---------|
| **Module Federation** | 运行时模块共享 | 不隔离 DOM，共享运行时 | 同一框架、有共享依赖 | 版本冲突风险 |
| **qiankun** | JS/CSS 沙箱 | 基于 single-spa | 遗留系统迁移 | Performance 开销 |
| **wujie** | WebComponent + iframe | 组件级隔离 | 技术栈完全异构 | iframe 通信代价 |
| **micro-app** | CustomElement 模拟沙箱 | 类 iframe 隔离 | 无侵入接入 | 样式隔离不够完美 |

---

## 六、反模式 (Anti-Patterns)

### 1. useEffect 地狱

```jsx
// ❌ 反模式: 连锁 effect 触发
function Component({ userId }) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState({});

  useEffect(() => { fetchUser(userId).then(setUser); }, [userId]);
  useEffect(() => { if (user) fetchPosts(user.id).then(setPosts); }, [user]);
  useEffect(() => { if (posts.length) fetchComments(posts).then(setComments); }, [posts]);

  // 问题: 3 次串行渲染(waterfall)、无法取消、竞态条件
}

// ✅ 修复: 合并为单一数据流
function Component({ userId }) {
  const { user, posts, comments } = useUserData(userId);
  // 在 Hook 内使用 AbortController 或 SWR 的 key 机制处理竞态
}
```

### 2. Props Drilling 无节制

```jsx
// ❌ 反模式: 逐层传递不使用的 props
<Page config={config}>
  <Layout config={config}>  {/* Layout 不需要 config */}
    <Header config={config}> {/* Header 也不需要 */}
      <UserMenu config={config} /> {/* 只有这里需要！ */}
    </Header>
  </Layout>
</Page>

// ✅ 修复: Context / 状态库 / 组件组合
// 状态库直接访问: const config = useStore(s => s.config);
// 或: 将需要 config 的部分作为单独的子树
```

### 3. 过早抽象 (Premature Abstraction)

```jsx
// ❌ 反模式: 只有一个使用场景就抽成通用组件
function UserAvatar({ user, size, shape, border, onClick }) {
  // 40 个 props，覆盖所有"可能"的变体
}
// 问题: 没有足够多的使用场景验证抽象是否正确

// ✅ 原则: WET 优于错误的 DRY
// 等有 3 个使用场景后再抽象
// 或者像 React 文档建议的: "写两次，第三次抽象"
```

### 4. 巨型组件 (God Component)

```jsx
// ❌ 反模式: 单一组件 500+ 行
function Dashboard() {
  // 30+ useState
  // 15+ useEffect
  // 混合 UI 逻辑和业务逻辑
}

// ✅ 修复: 从下沉状态开始拆分
function Dashboard() {
  return (
    <DashboardLayout>
      <Header />
      <Sidebar />   {/* 各自管理自己的状态 */}
      <MainContent />
      <ActivityPanel />
    </DashboardLayout>
  );
}
// 每个子组件单独可测试
```

### 5. CSS-in-JS 运行时性能陷阱

```javascript
// ❌ 反模式: 渲染时动态生成样式对象
function Item({ color }) {
  return <div css={{ color }}> // 每次渲染生成新的样式对象 → 重新注入 CSS
}

// ✅ 修复: 使用 CSS Variables 或静态样式
function Item({ color }) {
  return <div style={{ '--item-color': color }} css={staticStyles} />
}
```

### 6. 不设图片宽高导致 CLS

```html
<!-- ❌ 反模式: 图片加载完成后撑开布局 -->
<img src="hero.jpg" alt="banner" />
<!-- 文本因图片加载而上下跳动 -->

<!-- ✅ 修复: 显式宽高，或使用 aspect-ratio -->
<img src="hero.jpg" alt="banner" width="1200" height="600" />
<!-- 浏览器自动计算 aspect-ratio，预留空间 -->
<div style="aspect-ratio: 2/1; background: #eee;">
  <img src="hero.jpg" alt="banner" />
</div>
```

### 7. 缺少 Suspense 边界导致全页挂起

```jsx
// ❌ 反模式
function Page() {
  return (
    <div>
      <ProductList />  {/* 如果需要 Suspense 的数据依赖，整个 Page 挂起 */}
    </div>
  );
}

// ✅ 修复: 精细化 Suspense 边界
function Page() {
  return (
    <div>
      <Header /> {/* 即时渲染 */}
      <Suspense fallback={<ProductListSkeleton />}>
        <ProductList /> {/* 仅这个区域等待 */}
      </Suspense>
    </div>
  );
}
```

---

## 七、模式组合指南

```
常见场景 → 推荐模式组合:

用户仪表盘 (读 > 写 100:1):
  SSR (首屏) + Suspense Streaming (动态内容)
  Zustand (全局状态) + React Query/SWR (服务端状态)
  Skeleton + Optimistic UI (操作反馈)

电商商品页 (SEO + 交互):
  SSG (商品页) + ISR (库存/价格不可变的部分用 SSG, 价格/库存增量再生)
  Jotai (购物车原子化) + Virtual List (评论列表)
  图片用 next/image + WebP

实时消息/聊天:
  WebSocket + Zustand (同步状态)
  Virtual List (消息列表) + Optimistic UI (发送消息)
  Web Worker 处理大量消息排序

多步骤表单:
  XState (状态机) + Context (步骤间共享数据)
  每步单独组件 + Suspense lazy 加载下一步

遗留系统迁移:
  wujie / qiankun (微前端容器)
  Module Federation (如果是同一框架升级)
  Strangler Fig 模式: 逐步替换，新旧共存
```
