# TypeScript 工程化深度指南

> 面向生产环境的 TypeScript 类型系统进阶、类型安全模式与工程化实践。
> 目标读者：有 TypeScript 基础、需要在大型项目中建立类型安全体系的前端架构师。

---

## 一、类型系统进阶

### 1.1 Conditional Types（条件类型）与 infer 关键字

条件类型是 TypeScript 类型系统的“if/else”，配合 `infer` 实现类型提取。

```typescript
// 基础语法：T extends U ? X : Y
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>; // true
type B = IsString<42>;      // false

// infer：在条件类型中声明类型变量，提取局部类型
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type Fn = (a: number) => string;
type Result = ReturnType<Fn>; // string

// 进阶：深层 Promise 解包（递归条件类型）
type Awaited<T> = T extends Promise<infer V> ? Awaited<V> : T;
type DeepPromise = Promise<Promise<number>>;
type Unwrapped = Awaited<DeepPromise>; // number

// 提取数组元素
type ElementOf<T> = T extends (infer E)[] ? E : never;
type ArrEl = ElementOf<number[]>; // number

// 提取 React 组件 Props（条件 + infer 组合）
type ComponentProps<T> = T extends React.ComponentType<infer P> ? P : never;
type Props = ComponentProps<typeof MyComponent>;

// 分发条件类型：T 是联合类型时，条件类型对每个成员分别求值再联合
type ToArray<T> = T extends any ? T[] : never;
type Distributed = ToArray<string | number>; // string[] | number[]

// 阻止分发：用元组包裹
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type NonDist = ToArrayNonDist<string | number>; // (string | number)[]
```

### 1.2 Template Literal Types（模板字面量类型）

TypeScript 4.1 引入，允许在类型层面进行字符串操作。

```typescript
// 基础拼接
type Greeting = `Hello, ${string}`;
type World = `Hello, ${'World' | 'TS'}`; // "Hello, World" | "Hello, TS"

// 实际应用：CSS 属性推导
type CSSProperty = 'margin' | 'padding';
type CSSDirection = 'Top' | 'Right' | 'Bottom' | 'Left';
type CSSShorthand = `${CSSProperty}${CSSDirection}`;
// "marginTop" | "marginRight" | ... | "paddingLeft"

// 事件命名推导
type EventName<T extends string> = `on${Capitalize<T>}`;
// Capitalize, Uncapitalize, Uppercase, Lowercase 是 TS 4.1 内置工具

// 类型安全的路径解析（类似 lodash.get）
type Path<T, K extends keyof T = keyof T> =
  K extends string
    ? T[K] extends object
      ? `${K}.${Path<T[K]>}` | K
      : K
    : never;

type User = { profile: { name: string; age: number } };
type UserPath = Path<User>; // "profile" | "profile.name" | "profile.age"

// 实际应用：类型安全的 EventEmitter
type Events = {
  userLogin: { userId: string; timestamp: number };
  paymentSuccess: { orderId: string; amount: number };
};
type EventKey = keyof Events; // "userLogin" | "paymentSuccess"
type Handler<K extends EventKey> = `on${Capitalize<K>}`;
// "onUserLogin" | "onPaymentSuccess"
```

### 1.3 Mapped Types + Key Remapping（as 子句）

TypeScript 4.1 引入的 key remapping 允许在映射类型中重命名键。

```typescript
// 传统 Mapped Types
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };

// Key Remapping：使用 as 重新映射键名
type Setters<T> = {
  [K in keyof T & string as `set${Capitalize<K>}`]: (val: T[K]) => void;
};

interface Store {
  name: string;
  age: number;
}
type StoreSetters = Setters<Store>;
// { setName: (val: string) => void; setAge: (val: number) => void }

// 过滤键：用 never 排除
type ExcludeKeys<T, U> = {
  [K in keyof T as T[K] extends U ? never : K]: T[K];
};
type NoFunctions = ExcludeKeys<{ a: string; b: () => void }, Function>;
// { a: string }

// 提取特定类型键
type PickByType<T, U> = {
  [K in keyof T as T[K] extends U ? K : never]: T[K];
};
type OnlyStrings = PickByType<{ a: string; b: number; c: string }, string>;
// { a: string; c: string }
```

### 1.4 Branded Types（名义类型/标记类型）

TypeScript 是结构类型系统（structural typing），但有时需要名义类型（nominal typing）防止类型混淆。

```typescript
// 基础 Branded Type
type Brand<T, B extends string> = T & { __brand: B };

type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;

function createUserId(id: string): UserId {
  return id as UserId; // 显式铸造
}

function getUser(id: UserId) { /* ... */ }
function getOrder(id: OrderId) { /* ... */ }

const uid = createUserId('abc');
getUser(uid);    // ✅ OK
// getOrder(uid); // ❌ Error: Type 'UserId' not assignable to 'OrderId'

// 进阶：带验证的 Branded Type
type Email = Brand<string, 'Email'>;
function toEmail(str: string): Email {
  if (!str.includes('@')) throw new Error('Invalid email');
  return str as Email;
}

type PositiveInt = Brand<number, 'PositiveInt'>;
function toPositiveInt(n: number): PositiveInt {
  if (n <= 0 || !Number.isInteger(n)) throw new Error('Not positive integer');
  return n as PositiveInt;
}

// 单元类型 Branded Types（如货币单位）
type USD = Brand<number, 'USD'>;
type EUR = Brand<number, 'EUR'>;
function usdToEur(amount: USD, rate: number): EUR {
  return (amount * rate) as EUR;
}
// usdToEur(100, 0.92); // ❌ 100 不是 USD
usdToEur(100 as USD, 0.92); // ✅ OK
```

### 1.5 Discriminated Unions（可辨识联合）与状态机

Discriminated Union 是 TypeScript 最强大的模式之一，利用字面量类型做编译时穷举。

```typescript
// 状态机建模
type State =
  | { status: 'idle' }
  | { status: 'loading'; requestId: string }
  | { status: 'success'; data: unknown }
  | { status: 'error'; error: Error; retryCount: number };

function render(state: State): string {
  switch (state.status) {
    case 'idle':
      return 'Ready'; // state 类型收窄为 { status: 'idle' }
    case 'loading':
      return `Loading [${state.requestId}]`; // 可访问 requestId
    case 'success':
      return `Got ${String(state.data)}`;
    case 'error':
      return `Failed after ${state.retryCount} retries: ${state.error.message}`;
  }
}

// 高级：嵌套可辨识联合
type Event =
  | { type: 'click'; x: number; y: number }
  | { type: 'keydown'; key: string; ctrl: boolean }
  | { type: 'scroll'; deltaY: number };

function handleEvent(e: Event) {
  if (e.type === 'click') {
    e.x; // ✅ 收窄后可直接访问
  }
}
```

### 1.6 satisfies 运算符（TypeScript 4.9+）

`satisfies` 保留最精确的类型推断，同时检查类型兼容性。

```typescript
// ❌ 问题：使用类型注解会丢失字面量信息
const palette: Record<string, [number, number, number]> = {
  red: [255, 0, 0],
  green: [0, 255, 0],
};
palette.red.at(0); // ❌ TypeScript 不知道 red 存在
palette.green[0];   // ❌ number，不是字面量 0

// ✅ satisfies 保留精确类型
const palette2 = {
  red: [255, 0, 0],
  green: [0, 255, 0],
} satisfies Record<string, [number, number, number]>;

palette2.red;        // [number, number, number] ✅
palette2.red[0];     // 255 ✅ 保留字面量类型
// palette2.blue;     // ❌ 属性不存在

// satisfies + const 断言组合
const routes = {
  home: '/',
  user: '/user/:id',
  settings: '/settings',
} as const satisfies Record<string, `/${string}`>;

// routes.user 类型是 '/user/:id'，而非 string
```

### 1.7 const 断言（as const）

```typescript
// 数组变为 readonly tuple
const roles = ['admin', 'user', 'guest'] as const;
// type: readonly ["admin", "user", "guest"]
type Role = (typeof roles)[number]; // "admin" | "user" | "guest"

// 对象变为深度 readonly + 字面量类型
const config = {
  api: 'https://api.example.com',
  timeout: 5000,
  retry: { max: 3, delay: 1000 },
} as const;
// config.api 类型是 "https://api.example.com"，不是 string

// 枚举替代方案：as const + 类型提取
const STATUS = {
  PENDING: 'pending',
  ACTIVE: 'active',
  ARCHIVED: 'archived',
} as const;
type Status = (typeof STATUS)[keyof typeof STATUS];
// "pending" | "active" | "archived"
```

---

## 二、类型安全模式

### 2.1 Builder Pattern with Type Safety

```typescript
// 链式调用 + 类型累积
class QueryBuilder<T extends Record<string, unknown> = {}> {
  private _params: T;

  constructor(params: T = {} as T) {
    this._params = params;
  }

  where<K extends string>(key: K, value: unknown): QueryBuilder<T & { [P in K]: unknown }> {
    return new QueryBuilder({ ...this._params, [key]: value } as any);
  }

  orderBy(field: string): QueryBuilder<T & { __orderBy: string }> {
    return new QueryBuilder({ ...this._params, __orderBy: field } as any);
  }

  build(): T {
    return this._params;
  }
}

const query = new QueryBuilder()
  .where('userId', '123')
  .where('status', 'active')
  .orderBy('createdAt')
  .build();
// query 类型: { userId: unknown; status: unknown; __orderBy: string }
```

### 2.2 Type-safe Event Emitter

```typescript
// 使用 Template Literal Types + Mapped Types
class TypedEmitter<Events extends Record<string, unknown[]>> {
  private handlers = new Map<string, Set<(...args: any[]) => void>>();

  on<E extends keyof Events>(event: E, handler: (...args: Events[E]) => void): void {
    const key = String(event);
    if (!this.handlers.has(key)) this.handlers.set(key, new Set());
    this.handlers.get(key)!.add(handler);
  }

  emit<E extends keyof Events>(event: E, ...args: Events[E]): void {
    this.handlers.get(String(event))?.forEach(h => h(...args));
  }

  off<E extends keyof Events>(event: E, handler: (...args: Events[E]) => void): void {
    this.handlers.get(String(event))?.delete(handler);
  }
}

// 使用
interface MyEvents {
  login: [userId: string, timestamp: number];
  logout: [reason: string];
  error: [error: Error];
}

const emitter = new TypedEmitter<MyEvents>();
emitter.on('login', (userId, ts) => {
  // userId: string, ts: number — 类型安全推导
  console.log(userId, ts);
});
// emitter.emit('login', 123); // ❌ 类型不匹配
emitter.emit('login', 'user_1', Date.now()); // ✅
```

### 2.3 API 客户端类型推导

```typescript
// 从路由定义推导请求/响应类型
type ApiRoutes = {
  'GET /users': { params: { limit?: number }; response: { users: { id: string; name: string }[] } };
  'POST /users': { body: { name: string; email: string }; response: { id: string } };
  'GET /users/:id': { params: { id: string }; response: { id: string; name: string } };
};

// 类型安全的 fetcher
function createApiClient<T extends Record<string, any>>(baseUrl: string) {
  return {
    async get<Route extends keyof T & string>(
      route: Route,
      ...args: 'params' extends keyof T[Route] ? [T[Route]['params']] : []
    ): Promise<T[Route]['response']> {
      const res = await fetch(`${baseUrl}${route}`);
      return res.json();
    },
    async post<Route extends keyof T & string>(
      route: Route,
      body: 'body' extends keyof T[Route] ? T[Route]['body'] : never,
    ): Promise<T[Route]['response']> {
      const res = await fetch(`${baseUrl}${route}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
      });
      return res.json();
    },
  };
}

const api = createApiClient<ApiRoutes>('/api');
// 类型自动推导
const users = await api.get('GET /users');                // { users: { id, name }[] }
const newUser = await api.post('POST /users', { name: 'Alice', email: 'a@b.com' });
// api.get('POST /users'); // ❌ 参数类型不匹配
```

### 2.4 Exhaustiveness Checking with never

```typescript
// 编译时穷举检查：确保处理了所有联合成员
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${String(x)}`);
}

type Animal = { kind: 'dog'; bark: () => void } | { kind: 'cat'; meow: () => void };

function handleAnimal(a: Animal) {
  switch (a.kind) {
    case 'dog':
      return a.bark();
    case 'cat':
      return a.meow();
    default:
      // 如果 Animal 新增成员但此处未处理 → 编译错误
      assertNever(a); // ❌ 如果 a 不是 never 类型，则此处编译报错
  }
}
```

### 2.5 Recursive Types and Compiler Limits

```typescript
// 递归类型（TypeScript 4.5+ 对尾递归优化）
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function ? T[K] : DeepReadonly<T[K]>
    : T[K];
};

// JSON 类型（递归定义）
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// ⚠️ 编译器限制：
// - 递归深度默认限制 ~50 层
// - 条件类型分发过多可能触发 "Type instantiation is excessively deep"
// - 预防策略：拆解复杂类型、使用接口代替内联类型、避免深层联合分发
```

### 2.6 TypeScript Project References（复合工程）

```jsonc
// tsconfig.json (root)
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true
  },
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/client" },
    { "path": "./packages/server" }
  ]
}

// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

```bash
# 增量构建：每次仅构建变更的 project
tsc --build --incremental

# 监视模式：同时监视所有引用
tsc --build --watch

# 清理所有构建产物
tsc --build --clean
```

---

## 三、工程化实践

### 3.1 tsconfig strict 全家桶

```jsonc
// 推荐的生产级 tsconfig
{
  "compilerOptions": {
    /* 严格模式 — 必须全部开启 */
    "strict": true,                    // 一键开启以下所有严格检查
    // strict 等价于:
    //   strictNullChecks: true,
    //   noImplicitAny: true,
    //   strictFunctionTypes: true,
    //   strictBindCallApply: true,
    //   strictPropertyInitialization: true,
    //   noImplicitThis: true,
    //   alwaysStrict: true,

    /* 额外的类型安全开关 */
    "noUncheckedIndexedAccess": true,  // 索引访问包含 undefined
    "noImplicitReturns": true,         // 函数所有分支必须有返回值
    "noFallthroughCasesInSwitch": true,// switch 禁止落空
    "noUnusedLocals": true,            // 禁止未使用局部变量
    "noUnusedParameters": true,        // 禁止未使用参数
    "exactOptionalPropertyTypes": true,// 可选属性禁止赋 undefined

    /* 模块 */
    "moduleResolution": "bundler",     // 现代 bundler 解析策略
    "isolatedModules": true,           // 每个文件独立编译（transpile-only 兼容）
    "resolveJsonModule": true,
    "esModuleInterop": true,

    /* 输出 */
    "declaration": true,               // 生成 .d.ts
    "declarationMap": true,            // .d.ts.map（IDE 跳转到源码）
    "sourceMap": true,
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}
```

### 3.2 类型声明文件管理

```typescript
// 全局类型扩展：global.d.ts
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiBaseUrl: string;
      env: 'dev' | 'staging' | 'prod';
    };
  }
}

// 模块声明：为无类型的 npm 包添加类型
// types/modules.d.ts
declare module 'legacy-lib' {
  export function doThing(x: string): number;
}

declare module '*.svg' {
  const content: React.FunctionComponent<React.SVGProps<SVGSVGElement>>;
  export default content;
}

declare module '*.module.css' {
  const classes: { readonly [key: string]: string };
  export default classes;
}

// 环境变量类型
// env.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_API_URL: string;
    DATABASE_URL: string;
    SESSION_SECRET: string;
  }
}
```

### 3.3 泛型约束与类型收窄

```typescript
// extends 约束泛型
function getLength<T extends { length: number }>(arg: T): number {
  return arg.length;
}

// 类型收窄（Type Narrowing）
function process(val: string | number | Date) {
  if (typeof val === 'string') {
    val.toUpperCase(); // val: string
  } else if (typeof val === 'number') {
    val.toFixed(2);    // val: number
  } else {
    val.getTime();     // val: Date
  }
}

// 自定义类型守卫
function isError(val: unknown): val is Error {
  return val instanceof Error;
}

// in 操作符收窄
function handle(e: MouseEvent | KeyboardEvent) {
  if ('key' in e) {
    e.key; // KeyboardEvent
  } else {
    e.clientX; // MouseEvent
  }
}

// 断言函数（Assertion Functions, TS 3.7+）
function assert(condition: unknown, msg?: string): asserts condition {
  if (!condition) throw new Error(msg ?? 'Assertion failed');
}
const x: string | undefined = getValue();
assert(x !== undefined);
x.toUpperCase(); // x: string（收窄后）
```

### 3.4 declaration + sourceMap 发布策略

```jsonc
// package.json 最佳实践
{
  "name": "@myorg/utils",
  "main": "./dist/index.js",          // CJS 入口
  "module": "./dist/index.mjs",       // ESM 入口
  "types": "./dist/index.d.ts",       // 类型入口
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist"],
  // typescript 源码随发布（可选，方便调试跳转）
  "typesVersions": {
    "*": { "*": ["./dist/*", "./src/*"] }
  },
  "sideEffects": false               // 允许 tree-shaking
}
```

```bash
# 构建脚本
tsc --declaration --declarationMap --sourceMap
# 输出:
# dist/index.js
# dist/index.d.ts      ← types 入口
# dist/index.js.map    ← source map
# dist/index.d.ts.map  ← 声明 map（IDE 跳转到 .ts 源码）
```

### 3.5 类型性能优化

```typescript
// ❌ 避免：深层递归类型导致编译器 OOM 或卡死
type DeepFlatten<T> = T extends (infer U)[] ? DeepFlatten<U> : T;
// 对深层嵌套数组可能触发 "Type instantiation is excessively deep"

// ✅ 优化策略1：尾递归改写（TS 4.5+ 优化尾递归）
type DeepFlattenTail<T, Acc extends unknown[] = []> =
  T extends [infer Head, ...infer Tail]
    ? Head extends unknown[]
      ? DeepFlattenTail<[...Head, ...Tail], Acc>
      : DeepFlattenTail<Tail, [...Acc, Head]>
    : Acc;

// ✅ 优化策略2：拆解复杂类型为多个原子类型
// ❌ 巨型内联类型
type Complex = {
  a: { b: { c: { d: string[] }[] }[] };
  // ...
};

// ✅ 分层定义
interface Level4 { d: string[] }
interface Level3 { c: Level4[] }
interface Level2 { b: Level3[] }
interface Level1 { a: Level2[] }

// ✅ 优化策略3：使用接口替代交叉类型
// ❌ 交叉类型在大型联合中计算成本高
type Intersection<T, U> = T & U;
// ✅ 接口 extends
interface Extended extends T, U {}

// ✅ 优化策略4：避免不必要的大联合类型分发
// ❌ 分发所有 Props
type AllProps<T extends Component> = T extends any ? Props<T> : never;
// ✅ 直接限定
type AllProps<T extends Component> = Props<T>;
```

---

## 四、TypeScript 与框架

### 4.1 React 类型

```typescript
// ComponentProps：提取原生元素或组件 Props
import { ComponentProps, ComponentRef, forwardRef } from 'react';

type ButtonProps = ComponentProps<'button'>;
// onClick, disabled, type, ref, etc.

// 扩展原生元素
interface MyButtonProps extends ComponentProps<'button'> {
  variant: 'primary' | 'secondary';
  loading?: boolean;
}

// forwardRef 类型
const Input = forwardRef<HTMLInputElement, { label: string; error?: string }>(
  (props, ref) => (
    <div>
      <label>{props.label}</label>
      <input ref={ref} aria-invalid={!!props.error} />
      {props.error && <span>{props.error}</span>}
    </div>
  )
);

// ComponentRef：提取 ref 类型
type InputRef = ComponentRef<typeof Input>; // HTMLInputElement

// ReactNode vs ReactElement vs JSX.Element
// ReactNode   — 所有可渲染内容（最宽泛）
// ReactElement — createElement 返回值（有 type/props/key）
// JSX.Element — ReactElement 的别名

// 事件处理类型
function handler(e: React.MouseEvent<HTMLButtonElement>) {
  e.currentTarget; // HTMLButtonElement
  e.preventDefault();
}

// 泛型组件（TS 4.7+）
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
}
function List<T>({ items, renderItem }: ListProps<T>) {
  return <>{items.map(renderItem)}</>;
}

// Context 类型安全
const ThemeCtx = React.createContext<{ theme: 'light' | 'dark' }>({ theme: 'light' });
// React 19: use() hook 类型
```

### 4.2 Vue 3 类型

```typescript
// defineProps with TypeScript
interface Props {
  title: string;
  count?: number;
  items: string[];
}

const props = defineProps<Props>();

// defineEmits with 类型
const emit = defineEmits<{
  (e: 'update', value: string): void;
  (e: 'delete', id: number): void;
}>();

// defineExpose
defineExpose<{ focus: () => void }>({
  focus() { inputRef.value?.focus(); },
});

// 泛型组件 (Vue 3.3+)
// <script setup lang="ts" generic="T">
// defineProps<{ items: T[]; selectedKey: keyof T }>()
// </script>

// 组合式函数类型
function useFetch<T>(url: string) {
  const data = ref<T | null>(null);
  const error = ref<Error | null>(null);
  return { data, error };
}
```

### 4.3 Node.js 后端类型

```typescript
// Express typed middleware
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user: { id: string; role: 'admin' | 'user' };
}

function authMiddleware(req: Request, res: Response, next: NextFunction): void {
  // 验证后挂载 user
  (req as AuthenticatedRequest).user = { id: '1', role: 'admin' };
  next();
}

// 泛型中间件工厂
function requireRole(role: 'admin' | 'user') {
  return (req: AuthenticatedRequest, _res: Response, next: NextFunction): void => {
    if (req.user.role !== role) {
      _res.status(403).json({ error: 'Forbidden' });
      return;
    }
    next();
  };
}

// Koa typed context
import { Context, Middleware } from 'koa';
const mw: Middleware = async (ctx: Context, next) => {
  ctx.state.user = { id: '1' };
  await next();
};
```

---

## 五、常见类型体操速查

| 场景 | 工具类型 |
|------|---------|
| 排除联合类型 | `Exclude<T, U>` |
| 提取联合类型 | `Extract<T, U>` |
| 提取参数类型 | `Parameters<typeof fn>` |
| 提取返回值 | `ReturnType<typeof fn>` |
| 提取实例类型 | `InstanceType<typeof Class>` |
| 属性非空 | `NonNullable<T>` |
| 深层只读 | `DeepReadonly<T>`（自定义递归） |
| 深层可选 | `DeepPartial<T>`（自定义递归） |
| Promise 解包 | `Awaited<T>`（TS 4.5+） |
| 可索引类型键 | `keyof T & string` |
| 函数重载选择 | 最后一个重载定义 |

---

## 六、TypeScript 版本迁移要点

| 版本 | 关键特性 | 迁移注意 |
|------|---------|---------|
| 4.9 | `satisfies` 运算符 | 替换 `as` 断言，保留精确类型 |
| 5.0 | `const` 类型参数、装饰器 | decorator 仍需 `experimentalDecorators` |
| 5.1 | 函数返回值 `undefined` 宽松化 | `noUncheckedIndexedAccess` 行为微调 |
| 5.2 | `using` 声明（Explicit Resource Management） | 需要 `lib: "esnext.disposable"` |
| 5.3 | Import Attributes、tighter narrowing | `switch(true)` 收窄 |
| 5.4 | NoInfer 工具类型、`Map.groupBy` | `NoInfer<T>` 阻止泛型推导 |
| 5.5 | 推断类型谓词、正则语法检查 | 自动推断 `x is T` 返回类型 |
| 5.6 | Iterator Helper methods | 需要 `lib: "esnext"` |
| 5.7 | 未初始化变量检查增强 | `strictNullChecks` 下更严格 |

---

**参考文献**:
- [TypeScript Handbook v5](https://www.typescriptlang.org/docs/handbook/)
- [type-challenges](https://github.com/type-challenges/type-challenges)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
