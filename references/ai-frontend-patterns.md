# AI 前端应用落地模式 (AI Frontend Patterns)

> 面向 LLM 驱动应用的前端架构模式。涵盖流式响应、AI Chat 组件设计、Token 管理和错误处理。

---

## 一、核心架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                       AI 前端应用架构                            │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Chat UI  │  │ 流式解析 │  │ Token    │  │ 错误恢复      │  │
│  │ 组件层   │  │ 中间层   │  │ 管理层   │  │ 中间层        │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬───────┘  │
│       │              │              │                │          │
│  ┌────┴──────────────┴──────────────┴────────────────┴──────┐  │
│  │                   AI 请求适配层                            │  │
│  │  Vercel AI SDK / 自研 fetch wrapper / LangChain 前端       │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │                   传输层                                   │  │
│  │  SSE (Server-Sent Events) / WebSocket / HTTP Streaming    │  │
│  │  ReadableStream + TextDecoder                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、Vercel AI SDK 集成模式

### 核心 Hook

```typescript
// useChat — 完整的聊天管理
import { useChat } from 'ai/react';

function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, stop, error, reload } =
    useChat({
      api: '/api/chat',
      // 请求配置
      body: { model: 'gpt-4o' },
      // 错误处理
      onError: (error) => {
        console.error('Chat error:', error);
        toast.error(error.message);
      },
      // 完成回调
      onFinish: (message) => {
        analytics.track('chat_completed', { messageId: message.id });
      },
    });

  return (
    <div>
      {messages.map(m => (
        <Message key={m.id} role={m.role} content={m.content} />
      ))}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} disabled={isLoading} />
        {isLoading && <button type="button" onClick={stop}>停止</button>}
      </form>
    </div>
  );
}
```

### useCompletion — 文本补全

```typescript
// 适用: 写作助手、代码补全、翻译
import { useCompletion } from 'ai/react';

function WritingAssistant() {
  const { completion, complete, isLoading, stop } = useCompletion({
    api: '/api/completion',
  });

  return (
    <div>
      <textarea onChange={(e) => complete(e.target.value)} />
      <div>{completion}</div>
    </div>
  );
}
```

### 底层 Stream 处理（不依赖 SDK）

```typescript
// 手动处理 SSE 流 — 当你需要完全控制时
async function* streamChat(messages: Message[]): AsyncGenerator<string> {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop() || ''; // 保留不完整的行

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') return;
        const parsed = JSON.parse(data);
        const delta = parsed.choices?.[0]?.delta?.content;
        if (delta) yield delta;
      }
    }
  }
}
```

---

## 三、AI Chat 组件架构

### 组件树设计

```
<ChatProvider>               ← 全局 Chat 状态 (Zustand context)
  <ChatLayout>
    <Sidebar>                ← 对话列表 (独立 scroll)
      <ConversationList>
        <ConversationItem />
      </ConversationList>
    </Sidebar>
    <MainPanel>
      <MessageList>          ← 虚拟列表 + 自动滚动
        <Message>
          <MessageContent>   ← Markdown 渲染 (代码高亮/数学/图表)
          <MessageActions /> ← 复制/重新生成/编辑
        </Message>
      </MessageList>
      <Composer>             ← 输入框 + 附件 + 发送
        <Textarea />
        <AttachmentPreview />
        <SendButton />
      </Composer>
    </MainPanel>
  </ChatLayout>
</ChatProvider>
```

### 消息列表：虚拟列表 + 自动滚动

```typescript
// 核心: 结合虚拟列表和智能自动滚动
function MessageList({ messages }: { messages: Message[] }) {
  const listRef = useRef<VirtualListRef>(null);
  const [userScrolledUp, setUserScrolledUp] = useState(false);

  // 检测用户是否主动向上滚动
  useEffect(() => {
    const container = listRef.current?.scrollElement;
    if (!container) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        // 如果底部哨兵不可见，说明用户在上方
        setUserScrolledUp(!entry.isIntersecting);
      },
      { root: container, threshold: 0 }
    );

    const sentinel = container.querySelector('.scroll-sentinel');
    if (sentinel) observer.observe(sentinel);
    return () => observer.disconnect();
  }, []);

  // 新消息到达时，仅在用户没主动上翻时自动滚动
  useEffect(() => {
    if (!userScrolledUp && messages.length > 0) {
      listRef.current?.scrollToIndex(messages.length - 1, { align: 'end' });
    }
  }, [messages, userScrolledUp]);

  return (
    <VirtualList
      ref={listRef}
      items={messages}
      itemSize={getMessageHeight}
      overscan={5}
    >
      {({ index, data }) => <Message message={data} />}
    </VirtualList>
  );
}
```

### 输入框 IME 兼容

```typescript
function Composer({ onSend }: { onSend: (text: string) => void }) {
  const [isComposing, setIsComposing] = useState(false);
  const [value, setValue] = useState('');

  const handleKeyDown = (e: React.KeyboardEvent) => {
    // 中文/日文/韩文输入法组合输入时，不触发发送
    if (e.key === 'Enter' && !e.shiftKey && !isComposing) {
      e.preventDefault();
      onSend(value);
      setValue('');
    }
  };

  return (
    <textarea
      value={value}
      onChange={(e) => setValue(e.target.value)}
      onCompositionStart={() => setIsComposing(true)}
      onCompositionEnd={() => setIsComposing(false)}
      onKeyDown={handleKeyDown}
      rows={1}
      // 自动扩展高度
      style={{ height: 'auto' }}
      onInput={(e) => {
        const target = e.target as HTMLTextAreaElement;
        target.style.height = 'auto';
        target.style.height = `${Math.min(target.scrollHeight, 200)}px`;
      }}
    />
  );
}
```

### Markdown 渲染

```typescript
import ReactMarkdown from 'react-markdown';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
import remarkGfm from 'remark-gfm';
import remarkMath from 'remark-math';
import rehypeKatex from 'rehype-katex';

function MessageContent({ content }: { content: string }) {
  return (
    <ReactMarkdown
      remarkPlugins={[remarkGfm, remarkMath]}
      rehypePlugins={[rehypeKatex]}
      components={{
        // 代码块自定义渲染
        code({ node, className, children, ...props }) {
          const match = /language-(\w+)/.exec(className || '');
          if (match && !props.inline) {
            return (
              <CodeBlock
                language={match[1]}
                code={String(children).replace(/\n$/, '')}
              />
            );
          }
          return <code className={className} {...props}>{children}</code>;
        },
        // Mermaid 图表 (安全性考虑: 仅渲染静态 SVG)
        pre({ children }) {
          // Mermaid 在服务端预渲染为 SVG，避免前端执行 JS
          return <div className="mermaid-wrapper">{children}</div>;
        },
      }}
    />
  );
}
```

---

## 四、Streaming UX 模式

### 逐字显示 + 光标闪烁

```typescript
function StreamingMessage({ content }: { content: string }) {
  const [displayed, setDisplayed] = useState('');
  const frameRef = useRef<number>(0);
  const lastUpdateRef = useRef<number>(0);
  const charsPerFrame = 3; // 每帧渲染 3 个字符 (平衡流畅与性能)

  useEffect(() => {
    // 使用 rAF 节流渲染，避免每收到一个 delta 就触发一次 React 渲染
    const animate = (timestamp: number) => {
      if (timestamp - lastUpdateRef.current < 16) {
        frameRef.current = requestAnimationFrame(animate);
        return;
      }
      lastUpdateRef.current = timestamp;

      const target = content.length;
      const current = displayed.length;
      if (current < target) {
        const next = Math.min(current + charsPerFrame, target);
        setDisplayed(content.slice(0, next));
      }

      frameRef.current = requestAnimationFrame(animate);
    };

    frameRef.current = requestAnimationFrame(animate);
    return () => {
      if (frameRef.current) cancelAnimationFrame(frameRef.current);
    };
  }, [content]);

  return (
    <span>
      {displayed}
      {displayed.length < content.length && (
        <span className="animate-pulse">|</span> // 闪烁光标
      )}
    </span>
  );
}
```

### 中断 + 重试 + 继续生成

```typescript
function useStreamingChat() {
  const [status, setStatus] = useState<'idle' | 'streaming' | 'aborted' | 'error'>('idle');
  const abortRef = useRef<AbortController | null>(null);

  const abort = useCallback(() => {
    abortRef.current?.abort();
    setStatus('aborted');
  }, []);

  const send = useCallback(async (messages: Message[]) => {
    abortRef.current = new AbortController();
    setStatus('streaming');

    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          messages,
          continue_from: lastMessageId, // 继续生成的上下文标记
        }),
        signal: abortRef.current.signal,
      });
      // ... 处理流
    } catch (err) {
      if (err.name === 'AbortError') {
        setStatus('aborted');
      } else {
        setStatus('error');
      }
    }
  }, []);

  const retry = useCallback(() => {
    /* 使用相同参数重试 */
  }, []);

  const continueGeneration = useCallback(() => {
    /* 发送 continue_from 标记，让 LLM 从断点继续 */
  }, []);

  return { status, send, abort, retry, continueGeneration };
}
```

---

## 五、Token 感知管理

### Tiktoken 估算

```typescript
import { encodingForModel } from 'js-tiktoken';

const enc = encodingForModel('gpt-4o');

function estimateTokens(text: string): number {
  return enc.encode(text).length;
}

function estimateMessagesTokens(messages: Message[]): number {
  // 每条消息有固定开销: role 约 3-4 token
  const OVERHEAD = 4;
  return messages.reduce(
    (sum, m) => sum + estimateTokens(m.content) + OVERHEAD,
    0
  );
}
```

### 上下文窗口管理

```typescript
interface ContextWindowConfig {
  modelMaxTokens: number;    // 模型上下文上限 (如 gpt-4o: 128k)
  responseReserve: number;   // 预留给回复的 token (如 4096)
  systemPromptTokens: number; // 系统提示 token 数
}

function trimConversation(
  messages: Message[],
  config: ContextWindowConfig
): Message[] {
  const available = config.modelMaxTokens - config.responseReserve - config.systemPromptTokens;

  // 策略: 从旧到新保留，直到超出可用 token
  const result: Message[] = [];
  let used = 0;

  // 从最新消息向前遍历
  for (let i = messages.length - 1; i >= 0; i--) {
    const tokens = estimateTokens(messages[i].content) + 4;
    if (used + tokens > available) break;
    result.unshift(messages[i]);
    used += tokens;
  }

  // 如果第一条消息是 system，确保它被包含
  if (messages[0]?.role === 'system' && !result.some(m => m.role === 'system')) {
    result.unshift(messages[0]);
  }

  return result;
}
```

### Web Worker 中的 Token 计算

```typescript
// token-worker.ts
import { encodingForModel } from 'js-tiktoken';

const enc = encodingForModel('gpt-4o');

self.onmessage = (e: MessageEvent<{ messages: Message[] }>) => {
  const { messages } = e.data;
  let total = 0;
  const perMessage: number[] = [];

  for (const msg of messages) {
    const tokens = enc.encode(msg.content).length + 4; // +4 for role overhead
    total += tokens;
    perMessage.push(tokens);
  }

  self.postMessage({ total, perMessage });
};

// 使用
const tokenWorker = new Worker(new URL('./token-worker.ts', import.meta.url));

function estimateInWorker(messages: Message[]): Promise<number> {
  return new Promise((resolve) => {
    tokenWorker.onmessage = (e) => resolve(e.data.total);
    tokenWorker.postMessage({ messages });
  });
}
```

---

## 六、错误处理与恢复

### 网络断连 + 重试退避

```typescript
function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelay?: number;
    maxDelay?: number;
  } = {}
): Promise<T> {
  const { maxRetries = 3, baseDelay = 1000, maxDelay = 30000 } = options;

  return new Promise((resolve, reject) => {
    let attempt = 0;

    const tryRequest = async () => {
      try {
        const result = await fn();
        resolve(result);
      } catch (error) {
        attempt++;
        if (attempt > maxRetries) {
          reject(error);
          return;
        }

        // 指数退避 + 随机抖动
        const delay = Math.min(
          baseDelay * Math.pow(2, attempt) + Math.random() * 1000,
          maxDelay
        );

        // 如果是 429 Rate Limit，使用 Retry-After header
        if (error instanceof Response && error.status === 429) {
          const retryAfter = error.headers.get('Retry-After');
          const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : delay;
          setTimeout(tryRequest, waitTime);
        } else {
          setTimeout(tryRequest, delay);
        }
      }
    };

    tryRequest();
  });
}
```

### 流式中断的错误恢复

```typescript
// 流式传输中断后，需要恢复上下文并继续
async function resumeStream(
  conversationId: string,
  lastReceivedContent: string
): Promise<ReadableStream> {
  // 方案 A: 服务端记录已发送内容，客户端请求续传
  const response = await fetch(`/api/chat/resume/${conversationId}`, {
    method: 'POST',
    body: JSON.stringify({
      last_received: lastReceivedContent,
    }),
  });

  if (!response.ok) {
    // 方案 B: 降级 — 重新发送完整对话
    return fallbackRetry(conversationId);
  }

  return response.body!;
}
```

### 错误状态 UI

```typescript
function MessageError({ error, onRetry }: {
  error: ChatError;
  onRetry: () => void;
}) {
  const message = useMemo(() => {
    switch (error.type) {
      case 'NETWORK':
        return '网络连接失败，请检查网络后重试';
      case 'RATE_LIMIT':
        return '请求过于频繁，请稍后重试';
      case 'TIMEOUT':
        return '响应超时，AI 可能正在处理复杂问题';
      case 'CONTENT_FILTER':
        return '内容被安全策略拦截';
      case 'TOKEN_LIMIT':
        return '对话已达到长度限制，请开启新对话';
      case 'SERVER_ERROR':
        return '服务暂时不可用，请稍后重试';
      default:
        return `未知错误: ${error.message}`;
    }
  }, [error]);

  return (
    <div className="error-message" role="alert">
      <p>{message}</p>
      <button onClick={onRetry}>重试</button>
    </div>
  );
}
```

---

## 七、性能优化

### 渲染节流

```typescript
// 流式内容更新节流: 每帧最多触发一次 React 渲染
function useThrottledStream<T>(stream: AsyncIterable<T>, batchSize = 3) {
  const [items, setItems] = useState<T[]>([]);
  const bufferRef = useRef<T[]>([]);
  const rafRef = useRef<number>(0);

  const flush = useCallback(() => {
    if (bufferRef.current.length > 0) {
      setItems(prev => [...prev, ...bufferRef.current]);
      bufferRef.current = [];
    }
  }, []);

  useEffect(() => {
    const consume = async () => {
      for await (const item of stream) {
        bufferRef.current.push(item);
        if (bufferRef.current.length >= batchSize) {
          if (!rafRef.current) {
            rafRef.current = requestAnimationFrame(() => {
              flush();
              rafRef.current = 0;
            });
          }
        }
      }
      // 流结束，清空剩余缓冲
      flush();
    };
    consume();
    return () => {
      if (rafRef.current) cancelAnimationFrame(rafRef.current);
    };
  }, [stream]);

  return items;
}
```

### 消息列表 Memoization

```typescript
// 只重渲染变化的消息，而非整个列表
const MemoizedMessage = memo(
  function Message({ message, isStreaming }: MessageProps) {
    return (
      <div className={`message message-${message.role}`}>
        <MessageContent content={message.content} />
        {isStreaming && <Cursor />}
      </div>
    );
  },
  (prev, next) =>
    prev.message.id === next.message.id &&
    prev.message.content === next.message.content &&
    prev.isStreaming === next.isStreaming
);
```

### 对话持久化

```typescript
// 使用 IndexedDB 持久化对话历史
import { openDB, DBSchema } from 'idb';

interface ChatDB extends DBSchema {
  conversations: {
    key: string;
    value: {
      id: string;
      title: string;
      messages: Message[];
      createdAt: number;
      updatedAt: number;
    };
  };
}

const db = await openDB<ChatDB>('chat-db', 1, {
  upgrade(db) {
    db.createObjectStore('conversations', { keyPath: 'id' });
  },
});

// 自动保存 (防抖)
const saveDebounced = useMemo(
  () => debounce(async (conversation: Conversation) => {
    await db.put('conversations', {
      ...conversation,
      updatedAt: Date.now(),
    });
  }, 1000),
  []
);
```

---

## 八、安全性

### 用户输入净化

```typescript
// AI 应用中，用户输入会出现在 prompt 或消息列表中
// 必须在显示时进行 XSS 净化
import DOMPurify from 'dompurify';

function SafeContent({ content }: { content: string }) {
  // Markdown → HTML → 净化
  const html = markdownToHtml(content);
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: [
      'p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li',
      'code', 'pre', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
      'blockquote', 'table', 'thead', 'tbody', 'tr', 'th', 'td',
      'img', 'span', 'div',
    ],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'class', 'target'],
  });

  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### Token 消耗审计

```typescript
// 在生产环境上报 Token 消耗
function useTokenAudit() {
  useEffect(() => {
    return useChatStore.subscribe((state, prev) => {
      if (state.messages.length > prev.messages.length) {
        const newMessage = state.messages[state.messages.length - 1];
        if (newMessage.role === 'assistant' && newMessage.usage) {
          // 上报到监控平台
          analytics.track('ai_token_usage', {
            promptTokens: newMessage.usage.prompt_tokens,
            completionTokens: newMessage.usage.completion_tokens,
            model: newMessage.model,
            conversationId: state.conversationId,
          });
        }
      }
    });
  }, []);
}
```

---

## 九、测试策略

```typescript
// AI Chat 组件的测试层次

// 1. 单元测试: 工具函数
describe('trimConversation', () => {
  it('should trim oldest messages when exceeding token limit', () => {
    const messages = generateMessages(50);
    const trimmed = trimConversation(messages, {
      modelMaxTokens: 8000,
      responseReserve: 2048,
      systemPromptTokens: 100,
    });
    expect(estimateMessagesTokens(trimmed)).toBeLessThan(8000 - 2048 - 100);
  });
});

// 2. 流式解析测试
describe('parseSSEStream', () => {
  it('should handle partial chunks', async () => {
    const stream = createMockSSEStream(['Hello', ' World', ' [DONE]']);
    const results = [];
    for await (const delta of parseSSEStream(stream)) {
      results.push(delta);
    }
    expect(results.join('')).toBe('Hello World');
  });
});

// 3. E2E: 模拟完整对话流程
test('user can send message and receive streaming response', async ({ page }) => {
  await page.route('**/api/chat', (route) => {
    // 模拟 SSE 响应
    route.fulfill({
      contentType: 'text/event-stream',
      body: [
        'data: {"choices":[{"delta":{"content":"Hello"}}]}',
        'data: [DONE]',
      ].join('\n\n'),
    });
  });

  await page.goto('/chat');
  await page.fill('[data-testid="composer"]', 'Hi');
  await page.click('[data-testid="send"]');

  // 等待流式内容出现
  await expect(page.locator('[data-testid="message"]'))
    .toContainText('Hello');
});
```
