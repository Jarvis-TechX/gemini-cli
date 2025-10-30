# 🔄 Gemini CLI 完整数据流指南

## 从用户输入到 API 调用的完整路径

---

## 📍 第一层：工程入口点

### **1. 主入口文件**

**文件**: `packages/cli/index.ts`

```typescript
#!/usr/bin/env node

// 第 9 行 - 导入主函数
import './src/gemini.js';
import { main } from './src/gemini.js';

// 第 14 行 - 全局入口点
main().catch((error) => {
  // 错误处理
  process.exit(1);
});
```

**作用**：

- 这是整个应用的**最顶层入口**
- Node.js 通过 `#!/usr/bin/env node` 识别这是一个可执行文件
- 调用 `main()` 函数启动应用

**断点位置**: `packages/cli/index.ts:14` ⭐

---

### **2. main() 函数 - 应用初始化**

**文件**: `packages/cli/src/gemini.tsx`

```typescript
// 第 225 行 - main 函数入口
export async function main() {
  // 🔴 断点：应用启动
  setupUnhandledRejectionHandler();

  // 加载设置
  const settings = loadSettings();

  // 解析命令行参数
  const argv = await parseArguments(settings.merged);

  // 加载配置
  const config = await loadCliConfig({...});

  // 初始化 Gemini Client
  const { geminiClient, ideClient } = await initializeApp({
    config,
    settings,
    argv,
  });

  // 🔴 断点：启动 React UI
  if (argv.nonInteractive) {
    // 非交互模式（命令行模式）
    await runNonInteractive(geminiClient, argv, config, settings);
  } else {
    // 🔴 交互模式（主要流程）
    return gemini(geminiClient, config, settings, initResult);
  }
}
```

**作用**：

1. 加载用户设置和配置
2. 解析命令行参数
3. 初始化 Gemini API Client
4. 决定运行模式（交互/非交互）
5. 启动 React UI

**断点位置**:

- `packages/cli/src/gemini.tsx:225` - main 入口
- `packages/cli/src/gemini.tsx:180` - gemini() 函数（启动 React UI）

---

### **3. gemini() 函数 - 启动 React UI**

**文件**: `packages/cli/src/gemini.tsx`

```typescript
// 第 180 行
export async function gemini(
  geminiClient: GeminiClient,
  config: Config,
  settings: LoadedSettings,
  initResult: InitializationResult,
) {
  // 🔴 断点：渲染 React 应用
  const { instance } = render(
    strictMode ? (
      <React.StrictMode>
        <AppWrapper />
      </React.StrictMode>
    ) : (
      <AppWrapper />
    ),
    {
      exitOnCtrlC: false,
      isScreenReaderEnabled: config.getScreenReader(),
    },
  );
}
```

**作用**：

- 使用 Ink（React for CLI）渲染终端界面
- 启动主应用组件 `AppWrapper`

**断点位置**: `packages/cli/src/gemini.tsx:195` ⭐

---

## 📍 第二层：React UI 层

### **4. AppContainer - 主应用容器**

**文件**: `packages/cli/src/ui/AppContainer.tsx`

这是整个 UI 的核心，管理所有状态和交互。

```typescript
export const AppContainer = ({...}) => {
  // 历史记录管理
  const historyManager = useHistoryManager(config);

  // 用户输入 buffer
  const buffer = useTextBuffer();

  // 消息队列
  const { messageQueue, addMessage } = useMessageQueue({...});

  // 🔴 断点：用户提交处理
  const handleFinalSubmit = useCallback(
    (submittedValue: string) => {
      // 用户按下 Enter 后，输入值被传到这里
      addMessage(submittedValue);
    },
    [addMessage],
  );

  // Gemini Stream 处理器
  const {
    streamingState,
    submitQuery,
    pendingHistoryItems,
    // ...
  } = useGeminiStream(
    geminiClient,
    history,
    historyManager.addItem,
    config,
    settings,
    // ...
  );

  return (
    <UIStateContext.Provider value={uiState}>
      <UIActionsContext.Provider value={uiActions}>
        <MainContent />
      </UIStateContext.Provider>
    </UIStateContext.Provider>
  );
};
```

**作用**：

- 管理整个应用的状态（历史、输入、流式响应）
- 提供 Context 给子组件
- 连接用户输入和 Gemini Client

**关键断点**：

- `packages/cli/src/ui/AppContainer.tsx:713` - `handleFinalSubmit` ⭐⭐⭐
- `packages/cli/src/ui/AppContainer.tsx:672` - `addMessage`

---

### **5. InputPrompt - 输入组件**

**文件**: `packages/cli/src/ui/components/InputPrompt.tsx`

```typescript
export const InputPrompt = ({
  onSubmit, // 这就是 handleFinalSubmit
  buffer,
  // ...
}) => {
  // 🔴 断点：处理键盘输入
  const handleInput = useCallback(
    (key: Key) => {
      if (key.paste) {
        buffer.handleInput(key);
        return;
      }

      // Vim 模式处理
      if (vimHandleInput && vimHandleInput(key)) {
        return;
      }

      // Tab 补全
      if (key.tab) {
        handleTabCompletion();
        return;
      }

      // 🔴 Enter 键 - 提交输入
      if (key.return) {
        const text = buffer.getText();
        if (text.trim()) {
          buffer.clear();
          // 调用 onSubmit，即 handleFinalSubmit
          onSubmit(text);
        }
        return;
      }

      // 其他按键
      buffer.handleInput(key);
    },
    [buffer, onSubmit, vimHandleInput]
  );

  // 监听键盘事件
  useKeypress(handleInput);

  return <TextInput buffer={buffer} />;
};
```

**作用**：

- 捕获用户键盘输入
- 处理特殊按键（Tab 补全、Enter 提交、Vim 模式）
- 按下 Enter 后调用 `onSubmit(text)`

**关键断点**：

- `packages/cli/src/ui/components/InputPrompt.tsx:359` - `handleInput` ⭐
- `packages/cli/src/ui/components/InputPrompt.tsx:430+` - Enter 键处理

---

## 📍 第三层：消息处理和路由

### **6. useMessageQueue - 消息队列**

**文件**: `packages/cli/src/ui/hooks/useMessageQueue.ts`

```typescript
export const useMessageQueue = ({...}) => {
  const [messageQueue, setMessageQueue] = useState<string[]>([]);

  // 🔴 断点：消息入队
  const addMessage = useCallback((message: string) => {
    const trimmedMessage = message.trim();
    if (trimmedMessage.length > 0) {
      setMessageQueue((prev) => [...prev, trimmedMessage]);
    }
  }, []);

  // 当不在处理中且有消息时，自动提交
  useEffect(() => {
    if (
      isConfigInitialized &&
      streamingState === StreamingState.Idle &&
      messageQueue.length > 0
    ) {
      const messagesToSubmit = messageQueue.join('\n\n');
      setMessageQueue([]);
      // 🔴 调用 submitQuery
      submitQuery(messagesToSubmit);
    }
  }, [messageQueue, streamingState, isConfigInitialized, submitQuery]);

  return { messageQueue, addMessage };
};
```

**作用**：

- 管理待处理消息队列
- 在合适的时机自动提交消息
- 支持批量消息处理

**关键断点**：

- `packages/cli/src/ui/hooks/useMessageQueue.ts:37` - `addMessage` ⭐
- `packages/cli/src/ui/hooks/useMessageQueue.ts:68` - 自动提交逻辑

---

### **7. useGeminiStream - 查询路由和处理**

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// 第 805 行 - 核心查询提交函数
const submitQuery = useCallback(
  async (
    query: PartListUnion,
    options?: { isContinuation: boolean },
    prompt_id?: string,
  ) =>
    runInDevTraceSpan(
      { name: 'submitQuery' },
      async ({ metadata: spanMetadata }) => {
        spanMetadata.input = query;
        const queryId = `${Date.now()}-${Math.random()}`;
        activeQueryIdRef.current = queryId;

        // 🔴 断点：判断是 slash 命令还是普通 prompt
        if (isSlashCommand(query)) {
          // Slash 命令处理（如 /help, /chat）
          const result = await handleSlashCommand(query);
          if (result) {
            // 处理命令结果
            return;
          }
        }

        // 🔴 断点：判断是 at 命令（MCP）
        if (isAtCommand(query)) {
          const result = await handleAtCommand(query, config, ...);
          if (result) {
            return;
          }
        }

        // 🔴 断点：普通 prompt - 发送给 Gemini API
        setStreamingState(StreamingState.Responding);

        // 记录用户 prompt
        logUserPrompt(
          UserPromptEvent({
            prompt: typeof query === 'string' ? query : JSON.stringify(query),
            model: config.getModel(),
            sessionId: sessionId,
          }),
        );

        // 🔴 调用 Gemini Client
        const stream = geminiClient.sendMessageStream(
          query,
          abortController.signal,
          prompt_id || '',
        );

        // 🔴 断点：处理流式响应
        for await (const event of stream) {
          if (event.type === ServerGeminiEventType.CONTENT) {
            // 处理文本内容
            handleContentEvent(event);
          } else if (event.type === ServerGeminiEventType.TOOL_CALL) {
            // 处理工具调用
            handleToolCall(event);
          } else if (event.type === ServerGeminiEventType.FINISHED) {
            // 响应完成
            handleFinished(event);
          }
        }
      }
    ),
  [geminiClient, config, handleSlashCommand, ...]
);
```

**作用**：

- **核心路由器**：判断输入是命令还是 prompt
- 处理三种类型：
  1. Slash 命令（`/help`、`/chat`）
  2. At 命令（`@github`、`@mcp-server`）
  3. 普通 prompt（发送给 Gemini API）
- 管理流式响应
- 处理工具调用

**关键断点**：

- `packages/cli/src/ui/hooks/useGeminiStream.ts:805` - `submitQuery` ⭐⭐⭐
- `packages/cli/src/ui/hooks/useGeminiStream.ts:850+` - API 调用
- `packages/cli/src/ui/hooks/useGeminiStream.ts:900+` - 流式响应处理

---

## 📍 第四层：Core 层 - Gemini Client

### **8. GeminiClient.sendMessageStream() - 发送消息**

**文件**: `packages/core/src/core/client.ts`

```typescript
// 第 393 行
async *sendMessageStream(
  request: PartListUnion,
  signal: AbortSignal,
  prompt_id: string,
  turns: number = MAX_TURNS,
): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
  // 🔴 断点：准备发送消息

  // 获取 IDE 上下文（如果有 VSCode 集成）
  const { contextParts, newIdeContext } = await this.getIdeContextDelta();

  // 将用户输入和 IDE 上下文组合
  let enrichedRequest = request;
  if (contextParts.length > 0) {
    enrichedRequest = [
      ...contextParts,
      typeof request === 'string' ? request : ...request,
    ];
  }

  // 🔴 调用底层 GeminiChat
  const stream = this.geminiChat.sendMessageStream(
    this.getEffectiveModel(),
    { message: enrichedRequest },
    prompt_id,
  );

  // 🔴 断点：处理流式响应
  for await (const chunk of stream) {
    if (chunk.type === StreamEventType.CHUNK) {
      // 处理响应块
      yield processChunk(chunk.value);
    } else if (chunk.type === StreamEventType.TOOL_CALL) {
      // 工具调用请求
      yield processToolCall(chunk.value);
    }
  }
}
```

**作用**：

- 添加 IDE 上下文（如 VSCode 打开的文件、光标位置）
- 调用底层的 GeminiChat
- 转换响应格式

**关键断点**：

- `packages/core/src/core/client.ts:393` - `sendMessageStream` ⭐⭐

---

### **9. GeminiChat.sendMessageStream() - 组装 Prompt**

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// 第 226 行 - 发送流式消息
async sendMessageStream(
  model: string,
  params: SendMessageParameters,
  prompt_id: string,
): Promise<AsyncGenerator<StreamEvent>> {
  await this.sendPromise;

  // 🔴 断点：创建用户内容
  const userContent = createUserContent(params.message);

  // 记录到历史
  this.chatRecordingService.recordMessage({
    model,
    type: 'user',
    content: partListUnionToString(toParts(params.message)),
  });

  // 🔴 添加到历史记录
  this.history.push(userContent);

  // 🔴 断点：获取完整的对话历史
  const requestContents = this.getHistory(true);

  // 🔴 调用真正的 API
  const stream = await this.makeApiCallAndProcessStream(
    model,
    requestContents,
    params,
    prompt_id,
  );

  // 🔴 返回流式响应
  for await (const chunk of stream) {
    yield { type: StreamEventType.CHUNK, value: chunk };
  }
}
```

**作用**：

- 将用户输入转换为 Gemini API 格式
- **组装完整的对话历史**（包含之前的所有消息）
- 调用实际的 API

**关键断点**：

- `packages/core/src/core/geminiChat.ts:226` - `sendMessageStream` ⭐⭐
- `packages/core/src/core/geminiChat.ts:239` - 创建用户内容
- `packages/core/src/core/geminiChat.ts:256` - 添加到历史
- `packages/core/src/core/geminiChat.ts:257` - 获取历史记录

---

### **10. makeApiCallAndProcessStream() - 实际 API 调用**

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// 第 330 行左右
private async makeApiCallAndProcessStream(
  model: string,
  requestContents: Content[],
  params: SendMessageParameters,
  prompt_id: string,
): Promise<AsyncGenerator<StreamChunk>> {

  // 🔴 断点：准备 API 请求
  const makeApiCall = async () => {
    // 添加系统指令（如 GEMINI.md 内容）
    // 添加工具定义
    // 添加缓存配置

    // 🔴 调用 Google Gemini API
    return this.config.getContentGenerator().generateContentStream(
      {
        model: modelToUse,
        contents: requestContents,  // 完整对话历史
        config: { ...this.generationConfig, ...params.config },
      },
      prompt_id,
    );
  };

  // 处理 429 错误（配额限制）
  const stream = await onPersistent429Callback(makeApiCall);

  // 🔴 返回流式响应
  return stream;
}
```

**作用**：

- 添加系统指令（GEMINI.md）
- 添加工具定义（file operations, shell commands 等）
- 配置缓存策略
- **实际调用 Google Gemini API**

**关键断点**：

- `packages/core/src/core/geminiChat.ts:367` - API 调用 ⭐⭐⭐

---

### **11. ContentGenerator.generateContentStream() - Gemini SDK**

**文件**: `packages/core/src/core/contentGenerator.ts`

```typescript
async *generateContentStream(
  request: GenerateContentRequest,
  prompt_id: string,
): AsyncGenerator<GenerateContentStreamResult> {

  // 🔴 调用 Google Gemini SDK
  const response = await this.model.generateContentStream({
    model: request.model,
    contents: request.contents,
    generationConfig: request.config,
    tools: request.tools,
    systemInstruction: request.systemInstruction,
  });

  // 🔴 断点：流式返回响应
  for await (const chunk of response.stream) {
    yield {
      text: chunk.text(),
      candidates: chunk.candidates,
      usageMetadata: chunk.usageMetadata,
    };
  }
}
```

**作用**：

- 调用 Google 官方的 Gemini SDK (`@google/genai`)
- 处理流式响应
- 返回给上层处理

---

## 🎯 完整数据流图

```
用户在终端输入 "hello world" 并按 Enter
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第一层：工程入口
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
1️⃣ packages/cli/index.ts:14
   main()
           ↓
2️⃣ packages/cli/src/gemini.tsx:225
   main() - 初始化应用
   - 加载设置
   - 解析参数
   - 初始化 GeminiClient
           ↓
3️⃣ packages/cli/src/gemini.tsx:195
   gemini() - 启动 React UI
   render(<AppWrapper />)
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第二层：React UI 层
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
4️⃣ packages/cli/src/ui/AppContainer.tsx
   - 管理状态
   - 提供 Context
           ↓
5️⃣ packages/cli/src/ui/components/InputPrompt.tsx:359
   handleInput(key) - 捕获键盘输入
   - 检测到 Enter 键
   - 获取输入文本: "hello world"
           ↓
   onSubmit("hello world")
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第三层：消息处理和路由
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
6️⃣ packages/cli/src/ui/AppContainer.tsx:713
   handleFinalSubmit("hello world")
           ↓
   addMessage("hello world")
           ↓
7️⃣ packages/cli/src/ui/hooks/useMessageQueue.ts:37
   addMessage("hello world")
   - 添加到队列
   - 触发自动提交
           ↓
8️⃣ packages/cli/src/ui/hooks/useGeminiStream.ts:805
   submitQuery("hello world")
   - 判断：不是 /command
   - 判断：不是 @command
   - 是普通 prompt！
   - 记录用户输入日志
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第四层：Core 层 - Gemini Client
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
9️⃣ packages/core/src/core/client.ts:393
   GeminiClient.sendMessageStream("hello world")
   - 获取 IDE 上下文（如果有）
   - 组合：["IDE上下文", "hello world"]
           ↓
🔟 packages/core/src/core/geminiChat.ts:226
   GeminiChat.sendMessageStream(...)
   - 创建用户内容对象
   - 添加到历史记录: this.history.push(...)
   - 获取完整历史: this.getHistory()
           ↓
   历史记录格式（示例）：
   [
     { role: "user", parts: [{ text: "之前的问题" }] },
     { role: "model", parts: [{ text: "之前的回答" }] },
     { role: "user", parts: [{ text: "hello world" }] }  // 当前输入
   ]
           ↓
1️⃣1️⃣ packages/core/src/core/geminiChat.ts:367
   makeApiCallAndProcessStream(...)
   - 添加系统指令（GEMINI.md）
   - 添加工具定义（file ops, shell, etc.）
   - 配置缓存
           ↓
   最终 API 请求格式：
   {
     model: "gemini-2.0-flash",
     contents: [完整对话历史],
     systemInstruction: "GEMINI.md 内容",
     tools: [文件工具, Shell工具, ...],
     generationConfig: {...}
   }
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第五层：Google Gemini API
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
1️⃣2️⃣ packages/core/src/core/contentGenerator.ts
   ContentGenerator.generateContentStream(...)
   - 调用 @google/genai SDK
   - POST https://generativelanguage.googleapis.com/v1beta/...
           ↓
   ┌─────────────────────────────┐
   │   Google Gemini API 云端     │
   │   - 接收请求                 │
   │   - 处理 prompt              │
   │   - 执行推理                 │
   │   - 流式返回响应             │
   └─────────────────────────────┘
           ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
响应返回（流式）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           ↓
   响应流逐层返回：
   ContentGenerator → GeminiChat → GeminiClient
                    → useGeminiStream → UI 组件
           ↓
   用户在终端看到流式输出：
   "Hello! How can I help you today?"
```

---

## 🎓 关键概念

### **1. 对话历史管理**

整个对话历史存储在 `GeminiChat.history` 中：

```typescript
// packages/core/src/core/geminiChat.ts

private history: Content[] = [];

// 每次用户输入时：
this.history.push({
  role: "user",
  parts: [{ text: "hello world" }]
});

// 每次 API 响应时：
this.history.push({
  role: "model",
  parts: [{ text: "AI 的回答" }]
});

// 发送 API 请求时，发送完整历史：
const requestContents = this.history; // 包含所有对话
```

---

### **2. Prompt 组装过程**

最终发送给 API 的完整 Prompt 包含：

```typescript
{
  model: "gemini-2.0-flash",

  // 1️⃣ 系统指令（GEMINI.md 文件内容）
  systemInstruction: "You are a helpful coding assistant...",

  // 2️⃣ 完整对话历史
  contents: [
    { role: "user", parts: [{ text: "之前的问题1" }] },
    { role: "model", parts: [{ text: "之前的回答1" }] },
    { role: "user", parts: [{ text: "之前的问题2" }] },
    { role: "model", parts: [{ text: "之前的回答2" }] },
    // IDE 上下文（如果有 VSCode 集成）
    { role: "user", parts: [{ text: "IDE Context: 打开的文件..." }] },
    // 当前用户输入
    { role: "user", parts: [{ text: "hello world" }] },
  ],

  // 3️⃣ 可用工具定义
  tools: [
    {
      name: "readFile",
      description: "Read a file from the filesystem",
      parameters: {...}
    },
    {
      name: "executeCommand",
      description: "Execute a shell command",
      parameters: {...}
    },
    // ... 更多工具
  ],

  // 4️⃣ 生成配置
  generationConfig: {
    temperature: 1.0,
    topP: 0.95,
    topK: 40,
    maxOutputTokens: 8192,
  }
}
```

---

### **3. 三种输入类型的路由**

在 `useGeminiStream.ts:805` 的 `submitQuery` 中：

```typescript
submitQuery(query) {
  // 类型 1：Slash 命令
  if (query.startsWith('/')) {
    // → handleSlashCommand()
    // → 执行内置命令（/help, /chat, /clear, etc.）
    // → 不发送给 API
  }

  // 类型 2：At 命令（MCP）
  else if (query.startsWith('@')) {
    // → handleAtCommand()
    // → 调用 MCP Server
    // → 可能发送给 API（取决于命令）
  }

  // 类型 3：普通 Prompt
  else {
    // → geminiClient.sendMessageStream()
    // → 发送给 Gemini API
  }
}
```

---

## 📌 关键断点总结

| 位置            | 文件                          | 行号 | 说明                         |
| --------------- | ----------------------------- | ---- | ---------------------------- |
| **入口**        | `packages/cli/index.ts`       | 14   | 应用启动                     |
| **初始化**      | `packages/cli/src/gemini.tsx` | 225  | main()                       |
| **UI 启动**     | `packages/cli/src/gemini.tsx` | 195  | React 渲染                   |
| **键盘输入**    | `InputPrompt.tsx`             | 359  | handleInput                  |
| **Enter 提交**  | `InputPrompt.tsx`             | 430+ | onSubmit                     |
| **用户提交**    | `AppContainer.tsx`            | 713  | handleFinalSubmit ⭐⭐⭐     |
| **消息入队**    | `useMessageQueue.ts`          | 37   | addMessage                   |
| **查询路由**    | `useGeminiStream.ts`          | 805  | submitQuery ⭐⭐⭐           |
| **Client 发送** | `client.ts`                   | 393  | sendMessageStream ⭐⭐       |
| **组装历史**    | `geminiChat.ts`               | 226  | sendMessageStream ⭐⭐       |
| **API 调用**    | `geminiChat.ts`               | 367  | generateContentStream ⭐⭐⭐ |

---

## 🚀 实战调试建议

### **场景 1：追踪用户输入到 API**

设置以下断点（按顺序）：

1. `InputPrompt.tsx:430` - 看到用户按 Enter
2. `AppContainer.tsx:713` - 看到 `handleFinalSubmit("hello world")`
3. `useGeminiStream.ts:805` - 看到 `submitQuery("hello world")`
4. `geminiChat.ts:367` - 看到实际 API 调用

然后逐个 F5 继续，观察数据如何流动。

---

### **场景 2：查看完整的 API 请求**

在 `geminiChat.ts:367` 设置断点：

```typescript
// 断点在这里
return this.config.getContentGenerator().generateContentStream({
  model: modelToUse,
  contents: requestContents, // 👈 在这里查看完整历史
  config: { ...this.generationConfig, ...params.config },
});
```

在变量面板查看 `requestContents`，你会看到完整的对话历史。

---

### **场景 3：理解 Slash 命令如何被拦截**

在 `useGeminiStream.ts:805` 设置断点：

```typescript
submitQuery(query) {
  // 断点在这里
  if (isSlashCommand(query)) {
    // 👈 查看 query 值，如 "/help"
    return handleSlashCommand(query);
  }
  // 如果不是命令，继续往下执行
}
```

输入 `/help`，观察它如何被拦截而不发送给 API。

---

现在你应该完全理解整个数据流了！从键盘输入到云端 API，每一步都清晰可见。🎉
