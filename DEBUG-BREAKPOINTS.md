# 🎯 Gemini CLI 调试断点位置指南

## 📍 用户输入和命令处理的关键断点

### 1️⃣ 用户输入入口点（最重要！）

#### **A. 用户提交 Prompt 的入口**

**文件**: `packages/cli/src/ui/AppContainer.tsx`

```typescript
// 第 713 行 - 用户按下 Enter 后的第一个处理点
const handleFinalSubmit = useCallback(
  (submittedValue: string) => {
    // 🔴 在这里设置断点！查看用户输入的原始内容
    addMessage(submittedValue);
  },
  [addMessage],
);
```

**为什么重要**：

- 这是用户输入的**最顶层入口**
- `submittedValue`
  包含用户刚刚输入的完整内容（可能是 prompt、slash 命令或 shell 命令）

**调试场景**：

- 查看用户输入了什么
- 调试输入验证问题

---

#### **B. 消息队列处理**

**文件**: `packages/cli/src/ui/hooks/useMessageQueue.ts`

```typescript
// 第 37 行 - 消息被加入队列
const addMessage = useCallback((message: string) => {
  // 🔴 在这里设置断点！查看消息如何被排队
  const trimmedMessage = message.trim();
  if (trimmedMessage.length > 0) {
    setMessageQueue((prev) => [...prev, trimmedMessage]);
  }
}, []);
```

**为什么重要**：

- 处理消息排队逻辑
- 多个消息可能被排队等待处理

---

### 2️⃣ Slash 命令处理流程

#### **A. 命令分发器（判断是 Slash 命令还是普通 Prompt）**

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// 第 805 行 - 查询提交的核心函数
const submitQuery = useCallback(
  async (
    query: PartListUnion,
    options?: { isContinuation: boolean },
    prompt_id?: string,
  ) =>
    runInDevTraceSpan(
      { name: 'submitQuery' },
      async ({ metadata: spanMetadata }) => {
        // 🔴 在这里设置断点！查看完整的查询对象
        spanMetadata.input = query;
        const queryId = `${Date.now()}-${Math.random()}`;
        activeQueryIdRef.current = queryId;

        // ... 这里会判断是 slash 命令还是 Gemini 查询
      },
    ),
  // ...
);
```

**为什么重要**：

- 这是所有查询（slash 命令、普通 prompt、at 命令）的**统一入口**
- 在这里可以看到查询被如何路由

---

#### **B. Slash 命令识别和处理**

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// 搜索 "isSlashCommand" 或 "handleSlashCommand"

// 第 49 行左右 - 判断是否是 slash 命令
import { isAtCommand, isSlashCommand } from '../utils/commandUtils.js';

// 在 submitQuery 函数内部：
if (isSlashCommand(query)) {
  // 🔴 在这里设置断点！查看 slash 命令处理逻辑
  const result = await handleSlashCommand(query);
  // ...
}
```

**文件**: `packages/cli/src/ui/hooks/slashCommandProcessor.ts`

```typescript
// 第 200 行左右 - 实际执行 slash 命令
const handleSlashCommand = useCallback(
  async (cmd: PartListUnion): Promise<SlashCommandProcessorResult | false> => {
    // 🔴 在这里设置断点！查看 slash 命令的解析和执行
    const parsedCommand = parseSlashCommand(cmd);
    if (!parsedCommand) {
      return false;
    }

    // 查找对应的命令处理器
    const command = commands?.find((c) => c.name === parsedCommand.command);
    if (!command) {
      return false;
    }

    // 🔴 重要！执行命令
    const result = await command.handler(parsedCommand.args, commandContext);
    // ...
  },
  [commands, commandContext],
);
```

**为什么重要**：

- 可以看到 slash 命令如何被解析（如 `/help`、`/chat`、`/model`）
- 可以单步进入具体的命令处理器

---

#### **C. 具体的 Slash 命令实现**

**文件**: `packages/cli/src/ui/commands/` 目录下的各个命令文件

例如 `/help` 命令：

```typescript
// packages/cli/src/ui/commands/helpCommand.ts

export const helpCommand: SlashCommand = {
  name: 'help',
  description: 'Show available commands',
  handler: async (args, context) => {
    // 🔴 在这里设置断点！查看具体命令的执行逻辑
    return {
      type: MessageType.HELP,
      timestamp: Date.now(),
    };
  },
};
```

**常用命令文件**：

- `helpCommand.ts` - `/help`
- `chatCommand.ts` - `/chat`
- `modelCommand.ts` - `/model`
- `clearCommand.ts` - `/clear`
- `quitCommand.ts` - `/quit`

---

### 3️⃣ 普通 Prompt 处理（发送给 Gemini API）

#### **A. Gemini API 调用准备**

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// 在 submitQuery 函数内部，大约第 850+ 行
// 构建发送给 Gemini API 的请求

try {
  setStreamingState(StreamingState.Responding);

  // 🔴 在这里设置断点！查看发送给 API 的完整请求
  const stream = geminiClient.sendMessage({
    message: query,
    history: history,
    // ... 其他配置
  });

  // 🔴 在这里设置断点！开始处理流式响应
  for await (const event of stream) {
    // 处理 API 返回的事件
    if (event.type === ServerGeminiEventType.CONTENT) {
      // 🔴 在这里设置断点！查看 Gemini 返回的内容
      handleContentEvent(event);
    } else if (event.type === ServerGeminiEventType.TOOL_CALL) {
      // 🔴 在这里设置断点！查看工具调用请求
      handleToolCallEvent(event);
    }
    // ...
  }
} catch (error) {
  // 🔴 在这里设置断点！捕获 API 错误
  console.error('API Error:', error);
}
```

---

#### **B. Gemini Client 实现（Core 包）**

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// 大约第 100+ 行
async sendMessage(options: SendMessageOptions) {
  // 🔴 在这里设置断点！查看实际的 API 请求构建
  const { message, history, config } = options;

  // 构建 API 请求
  const request = this.buildRequest(message, history);

  // 🔴 在这里设置断点！即将发送 API 请求
  const response = await this.client.generateContent(request);

  return response;
}
```

---

### 4️⃣ Shell 命令处理（Shell Mode）

**文件**: `packages/cli/src/ui/hooks/shellCommandProcessor.ts`

```typescript
// Shell 模式下的命令处理
export const useShellCommandProcessor = (...) => {
  const handleShellCommand = useCallback(
    async (command: string) => {
      // 🔴 在这里设置断点！查看 shell 命令执行
      const result = await executeShellCommand(command);
      return result;
    },
    []
  );

  return { handleShellCommand };
};
```

---

### 5️⃣ At 命令处理（MCP 和自定义命令）

**文件**: `packages/cli/src/ui/hooks/atCommandProcessor.ts`

```typescript
export async function handleAtCommand(
  cmd: PartListUnion,
  config: Config,
  // ...
) {
  // 🔴 在这里设置断点！查看 @server 命令处理
  const parsed = parseAtCommand(cmd);

  if (parsed.serverName) {
    // 处理 MCP server 命令，如 @github list repos
  }

  return result;
}
```

---

### 6️⃣ 工具调用处理（Tool Execution）

#### **A. 工具调度器（React 层）**

**文件**: `packages/cli/src/ui/hooks/useReactToolScheduler.ts`

```typescript
// 工具调用的 React 层面处理
export const useReactToolScheduler = (...) => {
  const handleToolCall = useCallback(
    async (toolCall: ToolCallInfo) => {
      // 🔴 在这里设置断点！查看工具调用的 UI 处理
      // 如：文件读写、Shell 执行等
    },
    []
  );
};
```

---

#### **B. 核心工具调度器（Core 层）**

**文件**: `packages/core/src/core/coreToolScheduler.ts`

```typescript
// 大约第 100+ 行
export class CoreToolScheduler {
  async executeTool(toolCall: ToolCallRequest) {
    // 🔴 在这里设置断点！查看工具的实际执行
    const { name, args } = toolCall;

    // 根据工具名称分发
    switch (name) {
      case 'readFile':
        // 🔴 在这里设置断点！文件读取
        return await this.readFileTool(args);
      case 'writeFile':
        // 🔴 在这里设置断点！文件写入
        return await this.writeFileTool(args);
      case 'executeCommand':
        // 🔴 在这里设置断点！命令执行
        return await this.executeCommandTool(args);
      // ...
    }
  }
}
```

---

## 🚀 推荐的调试工作流

### 场景 1：调试用户输入处理

**断点设置顺序**：

1. `packages/cli/src/ui/AppContainer.tsx:713` - `handleFinalSubmit`
   - 查看用户输入的原始值

2. `packages/cli/src/ui/hooks/useMessageQueue.ts:37` - `addMessage`
   - 查看消息如何被排队

3. `packages/cli/src/ui/hooks/useGeminiStream.ts:805` - `submitQuery`
   - 查看查询如何被提交

**操作步骤**：

```bash
# 1. 构建项目
npm run build

# 2. 在上述位置设置断点

# 3. 按 F5 启动调试

# 4. 在 CLI 中输入内容并按 Enter

# 5. 观察断点如何依次被触发
```

---

### 场景 2：调试 Slash 命令

**断点设置顺序**：

1. `packages/cli/src/ui/hooks/useGeminiStream.ts` - 搜索 `isSlashCommand`
   - 查看是否识别为 slash 命令

2. `packages/cli/src/ui/hooks/slashCommandProcessor.ts:200+` -
   `handleSlashCommand`
   - 查看命令解析

3. `packages/cli/src/ui/commands/helpCommand.ts` (或其他命令文件)
   - 查看具体命令执行

**操作步骤**：

```bash
# 1. 设置断点

# 2. 启动调试（F5）

# 3. 输入 slash 命令，如：/help

# 4. 单步跟踪命令处理流程
```

---

### 场景 3：调试 Gemini API 调用

**断点设置顺序**：

1. `packages/cli/src/ui/hooks/useGeminiStream.ts:805` - `submitQuery`
   - 查看查询内容

2. `packages/core/src/core/geminiChat.ts` - `sendMessage` 函数
   - 查看 API 请求构建

3. `packages/cli/src/ui/hooks/useGeminiStream.ts` -
   `for await (const event of stream)`
   - 查看 API 响应流

**操作步骤**：

```bash
# 1. 设置断点

# 2. 启动调试（F5）

# 3. 输入普通 prompt（非 slash 命令）

# 4. 观察 API 调用过程
```

---

### 场景 4：调试工具调用

**断点设置顺序**：

1. `packages/cli/src/ui/hooks/useGeminiStream.ts` - 搜索 `TOOL_CALL`
   - 查看工具调用事件

2. `packages/cli/src/ui/hooks/useReactToolScheduler.ts` - `handleToolCall`
   - 查看 UI 层工具处理

3. `packages/core/src/core/coreToolScheduler.ts` - `executeTool`
   - 查看实际工具执行

**操作步骤**：

```bash
# 1. 设置断点

# 2. 启动调试（F5）

# 3. 输入一个会触发工具的 prompt，如：
#    "读取 package.json 文件的内容"

# 4. 观察工具调用流程
```

---

## 📊 完整的执行流程图

```
用户输入
    ↓
📍 handleFinalSubmit (AppContainer.tsx:713)
    ↓
📍 addMessage (useMessageQueue.ts:37)
    ↓
📍 submitQuery (useGeminiStream.ts:805)
    ↓
    ├─→ isSlashCommand? ────→ Yes ──→ 📍 handleSlashCommand (slashCommandProcessor.ts)
    │                                      ↓
    │                                 📍 具体命令处理器 (ui/commands/*.ts)
    │
    ├─→ isAtCommand? ──────→ Yes ──→ 📍 handleAtCommand (atCommandProcessor.ts)
    │
    └─→ 普通 Prompt ────────→ 📍 geminiClient.sendMessage (geminiChat.ts)
                                     ↓
                              发送到 Gemini API
                                     ↓
                              处理流式响应
                                     ↓
                            ┌────────┴────────┐
                            ↓                 ↓
                      CONTENT 事件      TOOL_CALL 事件
                      (文本响应)        (工具调用)
                                             ↓
                                   📍 useReactToolScheduler
                                             ↓
                                   📍 coreToolScheduler.executeTool
                                             ↓
                                        执行具体工具
```

---

## 🎯 快速参考表

| 调试目标             | 文件路径                                             | 行号/函数 | 说明                 |
| -------------------- | ---------------------------------------------------- | --------- | -------------------- |
| **用户输入入口**     | `packages/cli/src/ui/AppContainer.tsx`               | 713       | `handleFinalSubmit`  |
| **消息排队**         | `packages/cli/src/ui/hooks/useMessageQueue.ts`       | 37        | `addMessage`         |
| **查询提交**         | `packages/cli/src/ui/hooks/useGeminiStream.ts`       | 805       | `submitQuery`        |
| **Slash 命令处理**   | `packages/cli/src/ui/hooks/slashCommandProcessor.ts` | ~200      | `handleSlashCommand` |
| **命令解析**         | `packages/cli/src/utils/commands.ts`                 | -         | `parseSlashCommand`  |
| **Gemini API 调用**  | `packages/core/src/core/geminiChat.ts`               | ~100      | `sendMessage`        |
| **工具调度（UI）**   | `packages/cli/src/ui/hooks/useReactToolScheduler.ts` | -         | `handleToolCall`     |
| **工具调度（Core）** | `packages/core/src/core/coreToolScheduler.ts`        | -         | `executeTool`        |
| **Shell 命令**       | `packages/cli/src/ui/hooks/shellCommandProcessor.ts` | -         | `handleShellCommand` |
| **At 命令**          | `packages/cli/src/ui/hooks/atCommandProcessor.ts`    | -         | `handleAtCommand`    |

---

## 💡 调试技巧

### 1. 使用条件断点

在 `handleFinalSubmit` 设置条件断点，只在特定输入时暂停：

```javascript
// 条件表达式
submittedValue.includes('/help');
```

### 2. 使用日志点

在关键位置添加日志点而不暂停执行：

```javascript
// 日志表达式
User input: {submittedValue}
```

### 3. 监视关键变量

在"监视"面板添加：

```javascript
submittedValue;
query;
streamingState;
toolCalls;
```

### 4. 调用堆栈导航

当在深层函数中暂停时，使用调用堆栈向上追溯到
`handleFinalSubmit`，查看完整的执行路径。

---

## 🎓 实战练习

### 练习 1：追踪 `/help` 命令

1. 在以下位置设置断点：
   - `handleFinalSubmit`
   - `handleSlashCommand`
   - `packages/cli/src/ui/commands/helpCommand.ts`

2. 启动调试，输入 `/help`

3. 使用 F11 单步进入，观察完整流程

### 练习 2：追踪普通 Prompt 到 API

1. 在以下位置设置断点：
   - `submitQuery`
   - `geminiChat.ts` 的 `sendMessage`

2. 启动调试，输入 "Hello"

3. 观察请求如何被构建和发送

### 练习 3：追踪工具调用

1. 在以下位置设置断点：
   - `submitQuery`
   - `coreToolScheduler.executeTool`

2. 启动调试，输入 "读取 README.md"

3. 观察工具调用的完整生命周期

---

## 📌 注意事项

1. **Source Maps**：确保已构建项目 (`npm run build`)，否则断点可能不准确

2. **异步代码**：使用 F11 进入异步函数时，可能需要等待 Promise 解析

3. **React 渲染**：某些函数可能被多次调用（React 重新渲染），注意区分

4. **工具执行**：某些工具可能需要用户确认，断点可能在确认对话框显示时暂停

---

现在你已经知道所有关键的断点位置了！开始调试吧 🚀
